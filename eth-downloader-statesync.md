statesync 用来获取pivot point所指定的区块的所有的state 的trie树，也就是所有的账号的信息，包括普通账号和合约账户。

## 数据结构
stateSync调度下载由给定state root所定义的特定state trie的请求。


```go
// stateSync schedules requests for downloading a particular state trie defined
// by a given state root.
type stateSync struct {
	d *Downloader // Downloader instance to access and manage current peerset

	root   common.Hash        // State root currently being synced
	sched  *trie.Sync         // State trie sync scheduler defining the tasks
	keccak crypto.KeccakState // Keccak256 hasher to verify deliveries with

	trieTasks map[common.Hash]*trieTask // Set of trie node tasks currently queued for retrieval
	codeTasks map[common.Hash]*codeTask // Set of byte code tasks currently queued for retrieval

	numUncommitted   int
	bytesUncommitted int

	started chan struct{} // Started is signalled once the sync loop starts

	deliver    chan *stateReq // Delivery channel multiplexing peer responses
	cancel     chan struct{}  // Channel to signal a termination request
	cancelOnce sync.Once      // Ensures cancel only ever gets called once
	done       chan struct{}  // Channel to signal termination completion
	err        error          // Any error hit during sync (set before completion)
}
```






```go
// 主循环
// stateFetcher manages the active state sync and accepts requests
// on its behalf.
func (d *Downloader) stateFetcher() {
	for {
		select {
		case s := <-d.stateSyncStart:
			for next := s; next != nil; {
				next = d.runStateSync(next)
			}
		case <-d.stateCh:
			// Ignore state responses while no sync is running.
		case <-d.quitCh:
			return
		}
	}
}

// runStateSync runs a state synchronisation until it completes or another root
// hash is requested to be switched over to.
func (d *Downloader) runStateSync(s *stateSync) *stateSync {
	var (
		active   = make(map[string]*stateReq) // Currently in-flight requests
		finished []*stateReq                  // Completed or failed requests
		timeout  = make(chan *stateReq)       // Timed out active requests
	)
	log.Trace("State sync starting", "root", s.root)

	defer func() {
		// Cancel active request timers on exit. Also set peers to idle so they're
		// available for the next sync.
		for _, req := range active {
			req.timer.Stop()
			req.peer.SetNodeDataIdle(int(req.nItems), time.Now())
		}
	}()
	go s.run() // 启动run
	defer s.Cancel()

	// Listen for peer departure events to cancel assigned tasks
	peerDrop := make(chan *peerConnection, 1024)
	peerSub := s.d.peers.SubscribePeerDrops(peerDrop)
	defer peerSub.Unsubscribe()

	for {
		// Enable sending of the first buffered element if there is one.
		var (
			deliverReq   *stateReq
			deliverReqCh chan *stateReq
		)
		if len(finished) > 0 {
			deliverReq = finished[0]
			deliverReqCh = s.deliver
		}

		select {
		// The stateSync lifecycle:
		case next := <-d.stateSyncStart: // 启动
			d.spindownStateSync(active, finished, timeout, peerDrop)
			return next

		case <-s.done: 
			d.spindownStateSync(active, finished, timeout, peerDrop)
			return nil

		// Send the next finished request to the current sync:
		case deliverReqCh <- deliverReq:
			// Shift out the first request, but also set the emptied slot to nil for GC
			copy(finished, finished[1:])
			finished[len(finished)-1] = nil
			finished = finished[:len(finished)-1]

		// Handle incoming state packs:
		case pack := <-d.stateCh:
			// Discard any data not requested (or previously timed out)
			req := active[pack.PeerId()]
			if req == nil {
				log.Debug("Unrequested node data", "peer", pack.PeerId(), "len", pack.Items())
				continue
			}
			// Finalize the request and queue up for processing
			req.timer.Stop()
			req.response = pack.(*statePack).states
			req.delivered = time.Now()

			finished = append(finished, req)
			delete(active, pack.PeerId())

		// Handle dropped peer connections:
		case p := <-peerDrop:
			// Skip if no request is currently pending
			req := active[p.id]
			if req == nil {
				continue
			}
			// Finalize the request and queue up for processing
			req.timer.Stop()
			req.dropped = true
			req.delivered = time.Now()

			finished = append(finished, req)
			delete(active, p.id)

		// Handle timed-out requests:
		case req := <-timeout:
			// If the peer is already requesting something else, ignore the stale timeout.
			// This can happen when the timeout and the delivery happens simultaneously,
			// causing both pathways to trigger.
			if active[req.peer.id] != req {
				continue
			}
			req.delivered = time.Now()
			// Move the timed out data back into the download queue
			finished = append(finished, req)
			delete(active, req.peer.id)

		// Track outgoing state requests:
		case req := <-d.trackStateReq:
			// If an active request already exists for this peer, we have a problem. In
			// theory the trie node schedule must never assign two requests to the same
			// peer. In practice however, a peer might receive a request, disconnect and
			// immediately reconnect before the previous times out. In this case the first
			// request is never honored, alas we must not silently overwrite it, as that
			// causes valid requests to go missing and sync to get stuck.
			if old := active[req.peer.id]; old != nil {
				log.Warn("Busy peer assigned new state fetch", "peer", old.peer.id)
				// Move the previous request to the finished set
				old.timer.Stop()
				old.dropped = true
				old.delivered = time.Now()
				finished = append(finished, old)
			}
			// Start a timer to notify the sync loop if the peer stalled.
			req.timer = time.AfterFunc(req.timeout, func() {
				timeout <- req
			})
			active[req.peer.id] = req
		}
	}
}


// run starts the task assignment and response processing loop, blocking until
// it finishes, and finally notifying any goroutines waiting for the loop to
// finish.
func (s *stateSync) run() {
	close(s.started)
	if s.d.snapSync {
		s.err = s.d.SnapSyncer.Sync(s.root, s.cancel)
	} else {
		s.err = s.loop()
	}
	close(s.done)
}


// loop is the main event loop of a state trie sync. It it responsible for the
// assignment of new tasks to peers (including sending it to them) as well as
// for the processing of inbound data. Note, that the loop does not directly
// receive data from peers, rather those are buffered up in the downloader and
// pushed here async. The reason is to decouple processing from data receipt
// and timeouts.
func (s *stateSync) loop() (err error) {
	// Listen for new peer events to assign tasks to them
	newPeer := make(chan *peerConnection, 1024)
	peerSub := s.d.peers.SubscribeNewPeers(newPeer)
	defer peerSub.Unsubscribe()
	defer func() {
		cerr := s.commit(true)
		if err == nil {
			err = cerr
		}
	}()

	// Keep assigning new tasks until the sync completes or aborts
	for s.sched.Pending() > 0 {
		if err = s.commit(false); err != nil {
			return err
		}
		s.assignTasks()
		// Tasks assigned, wait for something to happen
		select {
		case <-newPeer:
			// New peer arrived, try to assign it download tasks

		case <-s.cancel:
			return errCancelStateFetch

		case <-s.d.cancelCh:
			return errCanceled

		case req := <-s.deliver:
			// Response, disconnect or timeout triggered, drop the peer if stalling
			log.Trace("Received node data response", "peer", req.peer.id, "count", len(req.response), "dropped", req.dropped, "timeout", !req.dropped && req.timedOut())
			if req.nItems <= 2 && !req.dropped && req.timedOut() {
				// 2 items are the minimum requested, if even that times out, we've no use of
				// this peer at the moment.
				log.Warn("Stalling state sync, dropping peer", "peer", req.peer.id)
				if s.d.dropPeer == nil {
					// The dropPeer method is nil when `--copydb` is used for a local copy.
					// Timeouts can occur if e.g. compaction hits at the wrong time, and can be ignored
					req.peer.log.Warn("Downloader wants to drop peer, but peerdrop-function is not set", "peer", req.peer.id)
				} else {
					s.d.dropPeer(req.peer.id)

					// If this peer was the master peer, abort sync immediately
					s.d.cancelLock.RLock()
					master := req.peer.id == s.d.cancelPeer
					s.d.cancelLock.RUnlock()

					if master {
						s.d.cancel()
						return errTimeout
					}
				}
			}
			// Process all the received blobs and check for stale delivery
			delivered, err := s.process(req)
			req.peer.SetNodeDataIdle(delivered, req.delivered)
			if err != nil {
				log.Warn("Node data write error", "err", err)
				return err
			}
		}
	}
	return nil
}
```