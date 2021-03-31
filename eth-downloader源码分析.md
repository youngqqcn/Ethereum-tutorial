downloader主要负责区块链最开始的同步工作，当前的同步有两种模式，一种是传统的fullmode,这种模式通过下载区块头，和区块体来构建区块链，同步的过程就和普通的区块插入的过程一样，包括区块头的验证，交易的验证，交易执行，账户状态的改变等操作，这其实是一个比较消耗CPU和磁盘的一个过程。 另一种模式就是 快速同步的fast sync模式， 这种模式有专门的文档来描述。请参考fast sync的文档。简单的说 fast sync的模式会下载区块头，区块体和收据， 插入的过程不会执行交易，然后在一个区块高度(最高的区块高度 - 1024)的时候同步所有的账户状态，后面的1024个区块会采用fullmode的方式来构建。 这种模式会加区块的插入时间，同时不会产生大量的历史的账户信息。会相对节约磁盘， 但是对于网络的消耗会更高。 因为需要下载收据和状态。

## downloader 数据结构


```go
type Downloader struct {
	// WARNING: The `rttEstimate` and `rttConfidence` fields are accessed atomically.
	// On 32 bit platforms, only 64-bit aligned fields can be atomic. The struct is
	// guaranteed to be so aligned, so take advantage of that. For more information,
	// see https://golang.org/pkg/sync/atomic/#pkg-note-BUG.
	rttEstimate   uint64 // Round trip time to target for download requests 以下载请求为目标的往返时间 
	rttConfidence uint64 // Confidence in the estimated RTT (unit: millionths to allow atomic ops) 对估算的RTT的置信度（单位：百万分之一，以允许原子操作）

	mode uint32         // Synchronisation mode defining the strategy used (per sync cycle), use d.getMode() to get the SyncMode
	mux  *event.TypeMux // Event multiplexer to announce sync operation events

	checkpoint uint64   // Checkpoint block number to enforce head against (e.g. fast sync)
	genesis    uint64   // Genesis block number to limit sync to (e.g. light client CHT)
	queue      *queue   // Scheduler for selecting the hashes to download
	peers      *peerSet // Set of active peers from which download can proceed

	stateDB    ethdb.Database  // Database to state sync into (and deduplicate via)
	stateBloom *trie.SyncBloom // Bloom filter for fast trie node and contract code existence checks

	// Statistics
	syncStatsChainOrigin uint64 // Origin block number where syncing started at
	syncStatsChainHeight uint64 // Highest block number known when syncing started
	syncStatsState       stateSyncStats
	syncStatsLock        sync.RWMutex // Lock protecting the sync stats fields

	lightchain LightChain
	blockchain BlockChain

	// Callbacks
	dropPeer peerDropFn // Drops a peer for misbehaving

	// Status
	synchroniseMock func(id string, hash common.Hash) error // Replacement for synchronise during testing
	synchronising   int32
	notified        int32
	committed       int32
	ancientLimit    uint64 // The maximum block number which can be regarded as ancient data.

	// Channels
	headerCh      chan dataPack        // Channel receiving inbound block headers 区块头
	bodyCh        chan dataPack        // Channel receiving inbound block bodies  区块体
	receiptCh     chan dataPack        // Channel receiving inbound receipts   收据
	bodyWakeCh    chan bool            // Channel to signal the block body fetcher of new tasks  
	receiptWakeCh chan bool            // Channel to signal the receipt fetcher of new tasks
	headerProcCh  chan []*types.Header // Channel to feed the header processor new tasks

	// State sync
    // 状态同步
	pivotHeader *types.Header // Pivot block header to dynamically push the syncing state root
	pivotLock   sync.RWMutex  // Lock protecting pivot header reads from updates

    // 快照同步
	snapSync       bool         // Whether to run state sync over the snap protocol
	SnapSyncer     *snap.Syncer // TODO(karalabe): make private! hack for now
	stateSyncStart chan *stateSync
	trackStateReq  chan *stateReq
	stateCh        chan dataPack // Channel receiving inbound node state data

    // 取消&终止
	// Cancellation and termination
	cancelPeer string         // Identifier of the peer currently being used as the master (cancel on drop)
	cancelCh   chan struct{}  // Channel to cancel mid-flight syncs
	cancelLock sync.RWMutex   // Lock to protect the cancel channel and peer in delivers
	cancelWg   sync.WaitGroup // Make sure all fetcher goroutines have exited.

	quitCh   chan struct{} // Quit channel to signal termination
	quitLock sync.Mutex    // Lock to prevent double closes

	// Testing hooks
	syncInitHook     func(uint64, uint64)  // Method to call upon initiating a new sync run
	bodyFetchHook    func([]*types.Header) // Method to call upon starting a block body fetch
	receiptFetchHook func([]*types.Header) // Method to call upon starting a receipt fetch
	chainInsertHook  func([]*fetchResult)  // Method to call upon inserting a chain of blocks (possibly in multiple invocations)
}
```



## 同步下载
Synchronise试图和一个peer来同步，如果同步过程中遇到一些错误，那么会删除掉Peer。然后会被重试。

```go

// Synchronise tries to sync up our local block chain with a remote peer, both
// adding various sanity checks as well as wrapping it with various log entries.
func (d *Downloader) Synchronise(id string, head common.Hash, td *big.Int, mode SyncMode) error {
	err := d.synchronise(id, head, td, mode) // 同步区块

	switch err {
	case nil, errBusy, errCanceled:
		return err
	}

	if errors.Is(err, errInvalidChain) || errors.Is(err, errBadPeer) || errors.Is(err, errTimeout) ||
		errors.Is(err, errStallingPeer) || errors.Is(err, errUnsyncedPeer) || errors.Is(err, errEmptyHeaderSet) ||
		errors.Is(err, errPeersUnavailable) || errors.Is(err, errTooOld) || errors.Is(err, errInvalidAncestor) {
		log.Warn("Synchronisation failed, dropping peer", "peer", id, "err", err)
		if d.dropPeer == nil {
			// The dropPeer method is nil when `--copydb` is used for a local copy.
			// Timeouts can occur if e.g. compaction hits at the wrong time, and can be ignored
			log.Warn("Downloader wants to drop peer, but peerdrop-function is not set", "peer", id)
		} else {
			d.dropPeer(id)
		}
		return err
	}
	log.Warn("Synchronisation failed, retrying", "err", err)
	return err
}


// synchronise will select the peer and use it for synchronising. If an empty string is given
// it will use the best peer possible and synchronize if its TD is higher than our own. If any of the
// checks fail an error will be returned. This method is synchronous
func (d *Downloader) synchronise(id string, hash common.Hash, td *big.Int, mode SyncMode) error {
	// Mock out the synchronisation if testing
	if d.synchroniseMock != nil {
		return d.synchroniseMock(id, hash)
	}
	// Make sure only one goroutine is ever allowed past this point at once
	if !atomic.CompareAndSwapInt32(&d.synchronising, 0, 1) {
		return errBusy
	}
	defer atomic.StoreInt32(&d.synchronising, 0)

	// Post a user notification of the sync (only once per session)
	if atomic.CompareAndSwapInt32(&d.notified, 0, 1) {
		log.Info("Block synchronisation started")
	}
	// If we are already full syncing, but have a fast-sync bloom filter laying
	// around, make sure it doesn't use memory any more. This is a special case
	// when the user attempts to fast sync a new empty network.
	if mode == FullSync && d.stateBloom != nil {
		d.stateBloom.Close()
	}
	// If snap sync was requested, create the snap scheduler and switch to fast
	// sync mode. Long term we could drop fast sync or merge the two together,
	// but until snap becomes prevalent, we should support both. TODO(karalabe).
	if mode == SnapSync { // 快照
		if !d.snapSync {
			log.Warn("Enabling snapshot sync prototype")
			d.snapSync = true
		}
		mode = FastSync
	}
	// Reset the queue, peer set and wake channels to clean any internal leftover state
	d.queue.Reset(blockCacheMaxItems, blockCacheInitialItems)
	d.peers.Reset()

	for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
		select {
		case <-ch:
		default:
		}
	}
	for _, ch := range []chan dataPack{d.headerCh, d.bodyCh, d.receiptCh} {
		for empty := false; !empty; {
			select {
			case <-ch:
			default:
				empty = true
			}
		}
	}
	for empty := false; !empty; {
		select {
		case <-d.headerProcCh:
		default:
			empty = true
		}
	}
	// Create cancel channel for aborting mid-flight and mark the master peer
	d.cancelLock.Lock()
	d.cancelCh = make(chan struct{})
	d.cancelPeer = id
	d.cancelLock.Unlock()

	defer d.Cancel() // No matter what, we can't leave the cancel channel open

	// Atomically set the requested sync mode
	atomic.StoreUint32(&d.mode, uint32(mode))

	// Retrieve the origin peer and initiate the downloading process
	p := d.peers.Peer(id)
	if p == nil {
		return errUnknownPeer
	}
	return d.syncWithPeer(p, hash, td)
}
```


syncWithPeer

```go

// 区块同步
// syncWithPeer starts a block synchronization based on the hash chain from the
// specified peer and head hash.
func (d *Downloader) syncWithPeer(p *peerConnection, hash common.Hash, td *big.Int) (err error) {
	d.mux.Post(StartEvent{})
	defer func() {
		// reset on error
		if err != nil {
			d.mux.Post(FailedEvent{err})
		} else {
			latest := d.lightchain.CurrentHeader()
			d.mux.Post(DoneEvent{latest})
		}
	}()
	if p.version < 64 {
		return fmt.Errorf("%w: advertized %d < required %d", errTooOld, p.version, 64)
	}
	mode := d.getMode()

	log.Debug("Synchronising with the network", "peer", p.id, "eth", p.version, "head", hash, "td", td, "mode", mode)
	defer func(start time.Time) {
		log.Debug("Synchronisation terminated", "elapsed", common.PrettyDuration(time.Since(start)))
	}(time.Now())

	// Look up the sync boundaries: the common ancestor and the target block
	latest, pivot, err := d.fetchHead(p)
	if err != nil {
		return err
	}
	if mode == FastSync && pivot == nil {
		// If no pivot block was returned, the head is below the min full block
		// threshold (i.e. new chian). In that case we won't really fast sync
		// anyway, but still need a valid pivot block to avoid some code hitting
		// nil panics on an access.
		pivot = d.blockchain.CurrentBlock().Header()
	}
	height := latest.Number.Uint64()

	origin, err := d.findAncestor(p, latest)
	if err != nil {
		return err
	}
	d.syncStatsLock.Lock()
	if d.syncStatsChainHeight <= origin || d.syncStatsChainOrigin > origin {
		d.syncStatsChainOrigin = origin
	}
	d.syncStatsChainHeight = height
	d.syncStatsLock.Unlock()

	// Ensure our origin point is below any fast sync pivot point
	if mode == FastSync {
		if height <= uint64(fsMinFullBlocks) {
			origin = 0
		} else {
			pivotNumber := pivot.Number.Uint64()
			if pivotNumber <= origin {
				origin = pivotNumber - 1
			}
			// Write out the pivot into the database so a rollback beyond it will
			// reenable fast sync
			rawdb.WriteLastPivotNumber(d.stateDB, pivotNumber)
		}
	}
	d.committed = 1
	if mode == FastSync && pivot.Number.Uint64() != 0 {
		d.committed = 0
	}
	if mode == FastSync {
		// Set the ancient data limitation.
		// If we are running fast sync, all block data older than ancientLimit will be
		// written to the ancient store. More recent data will be written to the active
		// database and will wait for the freezer to migrate.
		//
		// If there is a checkpoint available, then calculate the ancientLimit through
		// that. Otherwise calculate the ancient limit through the advertised height
		// of the remote peer.
		//
		// The reason for picking checkpoint first is that a malicious peer can give us
		// a fake (very high) height, forcing the ancient limit to also be very high.
		// The peer would start to feed us valid blocks until head, resulting in all of
		// the blocks might be written into the ancient store. A following mini-reorg
		// could cause issues.
		if d.checkpoint != 0 && d.checkpoint > fullMaxForkAncestry+1 {
			d.ancientLimit = d.checkpoint
		} else if height > fullMaxForkAncestry+1 {
			d.ancientLimit = height - fullMaxForkAncestry - 1
		} else {
			d.ancientLimit = 0
		}
		frozen, _ := d.stateDB.Ancients() // Ignore the error here since light client can also hit here.

		// If a part of blockchain data has already been written into active store,
		// disable the ancient style insertion explicitly.
		if origin >= frozen && frozen != 0 {
			d.ancientLimit = 0
			log.Info("Disabling direct-ancient mode", "origin", origin, "ancient", frozen-1)
		} else if d.ancientLimit > 0 {
			log.Debug("Enabling direct-ancient mode", "ancient", d.ancientLimit)
		}
		// Rewind the ancient store and blockchain if reorg happens.
		if origin+1 < frozen {
			if err := d.lightchain.SetHead(origin + 1); err != nil {
				return err
			}
		}
	}
	// Initiate the sync using a concurrent header and content retrieval algorithm
	d.queue.Prepare(origin+1, mode)
	if d.syncInitHook != nil {
		d.syncInitHook(origin, height)
	}
	fetchers := []func() error{
		func() error { return d.fetchHeaders(p, origin+1) }, // Headers are always retrieved
		func() error { return d.fetchBodies(origin + 1) },   // Bodies are retrieved during normal and fast sync
		func() error { return d.fetchReceipts(origin + 1) }, // Receipts are retrieved during fast sync
		func() error { return d.processHeaders(origin+1, td) },
	}
	if mode == FastSync {
		d.pivotLock.Lock()
		d.pivotHeader = pivot
		d.pivotLock.Unlock()

		fetchers = append(fetchers, func() error { return d.processFastSyncContent() })
	} else if mode == FullSync {
		fetchers = append(fetchers, d.processFullSyncContent)
	}
	return d.spawnSync(fetchers)
}
```




spawnSync给每个fetcher启动一个goroutine, 然后阻塞的等待fetcher出错。 

```go


// spawnSync runs d.process and all given fetcher functions to completion in
// separate goroutines, returning the first error that appears.
func (d *Downloader) spawnSync(fetchers []func() error) error {
	errc := make(chan error, len(fetchers))
	d.cancelWg.Add(len(fetchers))
	for _, fn := range fetchers {
		fn := fn
		go func() { defer d.cancelWg.Done(); errc <- fn() }()
	}
	// Wait for the first error, then terminate the others.
	var err error
	for i := 0; i < len(fetchers); i++ {
		if i == len(fetchers)-1 {
			// Close the queue when all fetchers have exited.
			// This will cause the block processor to end when
			// it has processed the queue.
			d.queue.Close()
		}
		if err = <-errc; err != nil && err != errCanceled {
			break
		}
	}
	d.queue.Close()
	d.Cancel()
	return err
}
```

## headers的处理

fetchHeaders方法用来获取header。 然后根据获取的header去获取body和receipt等信息。

```go
//fetchHeaders不断从数字中同时检索标头
//请求，直到不再返回，可能会在途中节流。到
//促进并发，但仍可防止恶意节点发送错误消息
//标头，我们使用“原始”对等体构造标头链骨架
//与他人同步，并使用其他任何人填写缺少的标头。标头来自
//仅当其他对等方干净地映射到框架时，才会被接受。如果没有人
//可以填充骨骼-甚至不能填充原始对等体-假定其无效，并且
//原点已删除。
// fetchHeaders keeps retrieving headers concurrently from the number
// requested, until no more are returned, potentially throttling on the way. To
// facilitate concurrency but still protect against malicious nodes sending bad
// headers, we construct a header chain skeleton using the "origin" peer we are
// syncing with, and fill in the missing headers using anyone else. Headers from
// other peers are only accepted if they map cleanly to the skeleton. If no one
// can fill in the skeleton - not even the origin peer - it's assumed invalid and
// the origin is dropped.
func (d *Downloader) fetchHeaders(p *peerConnection, from uint64) error {
	p.log.Debug("Directing header downloads", "origin", from)
	defer p.log.Debug("Header download terminated")

	// Create a timeout timer, and the associated header fetcher
	skeleton := true            // Skeleton assembly phase or finishing up
	pivoting := false           // Whether the next request is pivot verification
	request := time.Now()       // time of the last skeleton fetch request
	timeout := time.NewTimer(0) // timer to dump a non-responsive active peer
	<-timeout.C                 // timeout channel should be initially empty
	defer timeout.Stop()

	var ttl time.Duration
	getHeaders := func(from uint64) {
		request = time.Now()

		ttl = d.requestTTL()
		timeout.Reset(ttl)

		if skeleton {
			p.log.Trace("Fetching skeleton headers", "count", MaxHeaderFetch, "from", from)
			go p.peer.RequestHeadersByNumber(from+uint64(MaxHeaderFetch)-1, MaxSkeletonSize, MaxHeaderFetch-1, false)
		} else {
			p.log.Trace("Fetching full headers", "count", MaxHeaderFetch, "from", from)
			go p.peer.RequestHeadersByNumber(from, MaxHeaderFetch, 0, false)
		}
	}
	getNextPivot := func() {
		pivoting = true
		request = time.Now()

		ttl = d.requestTTL()
		timeout.Reset(ttl)

		d.pivotLock.RLock()
		pivot := d.pivotHeader.Number.Uint64()
		d.pivotLock.RUnlock()

		p.log.Trace("Fetching next pivot header", "number", pivot+uint64(fsMinFullBlocks))
		go p.peer.RequestHeadersByNumber(pivot+uint64(fsMinFullBlocks), 2, fsMinFullBlocks-9, false) // move +64 when it's 2x64-8 deep
	}
	// Start pulling the header chain skeleton until all is done
	ancestor := from
	getHeaders(from)

	mode := d.getMode()
	for {
		select {
		case <-d.cancelCh:
			return errCanceled

		case packet := <-d.headerCh:
			// Make sure the active peer is giving us the skeleton headers
			if packet.PeerId() != p.id {
				log.Debug("Received skeleton from incorrect peer", "peer", packet.PeerId())
				break
			}
			headerReqTimer.UpdateSince(request)
			timeout.Stop()

			// If the pivot is being checked, move if it became stale and run the real retrieval
			var pivot uint64

			d.pivotLock.RLock()
			if d.pivotHeader != nil {
				pivot = d.pivotHeader.Number.Uint64()
			}
			d.pivotLock.RUnlock()

			if pivoting {
				if packet.Items() == 2 {
					// Retrieve the headers and do some sanity checks, just in case
					headers := packet.(*headerPack).headers

					if have, want := headers[0].Number.Uint64(), pivot+uint64(fsMinFullBlocks); have != want {
						log.Warn("Peer sent invalid next pivot", "have", have, "want", want)
						return fmt.Errorf("%w: next pivot number %d != requested %d", errInvalidChain, have, want)
					}
					if have, want := headers[1].Number.Uint64(), pivot+2*uint64(fsMinFullBlocks)-8; have != want {
						log.Warn("Peer sent invalid pivot confirmer", "have", have, "want", want)
						return fmt.Errorf("%w: next pivot confirmer number %d != requested %d", errInvalidChain, have, want)
					}
					log.Warn("Pivot seemingly stale, moving", "old", pivot, "new", headers[0].Number)
					pivot = headers[0].Number.Uint64()

					d.pivotLock.Lock()
					d.pivotHeader = headers[0]
					d.pivotLock.Unlock()

					// Write out the pivot into the database so a rollback beyond
					// it will reenable fast sync and update the state root that
					// the state syncer will be downloading.
					rawdb.WriteLastPivotNumber(d.stateDB, pivot)
				}
				pivoting = false
				getHeaders(from)
				continue
			}
			// If the skeleton's finished, pull any remaining head headers directly from the origin
			if skeleton && packet.Items() == 0 {
				skeleton = false
				getHeaders(from)
				continue
			}
			// If no more headers are inbound, notify the content fetchers and return
			if packet.Items() == 0 {
				// Don't abort header fetches while the pivot is downloading
				if atomic.LoadInt32(&d.committed) == 0 && pivot <= from {
					p.log.Debug("No headers, waiting for pivot commit")
					select {
					case <-time.After(fsHeaderContCheck):
						getHeaders(from)
						continue
					case <-d.cancelCh:
						return errCanceled
					}
				}
				// Pivot done (or not in fast sync) and no more headers, terminate the process
				p.log.Debug("No more headers available")
				select {
				case d.headerProcCh <- nil:
					return nil
				case <-d.cancelCh:
					return errCanceled
				}
			}
			headers := packet.(*headerPack).headers

			// If we received a skeleton batch, resolve internals concurrently
			if skeleton {
				filled, proced, err := d.fillHeaderSkeleton(from, headers)
				if err != nil {
					p.log.Debug("Skeleton chain invalid", "err", err)
					return fmt.Errorf("%w: %v", errInvalidChain, err)
				}
				headers = filled[proced:]
				from += uint64(proced)
			} else {
				// If we're closing in on the chain head, but haven't yet reached it, delay
				// the last few headers so mini reorgs on the head don't cause invalid hash
				// chain errors.
				if n := len(headers); n > 0 {
					// Retrieve the current head we're at
					var head uint64
					if mode == LightSync {
						head = d.lightchain.CurrentHeader().Number.Uint64()
					} else {
						head = d.blockchain.CurrentFastBlock().NumberU64()
						if full := d.blockchain.CurrentBlock().NumberU64(); head < full {
							head = full
						}
					}
					// If the head is below the common ancestor, we're actually deduplicating
					// already existing chain segments, so use the ancestor as the fake head.
					// Otherwise we might end up delaying header deliveries pointlessly.
					if head < ancestor {
						head = ancestor
					}
					// If the head is way older than this batch, delay the last few headers
					if head+uint64(reorgProtThreshold) < headers[n-1].Number.Uint64() {
						delay := reorgProtHeaderDelay
						if delay > n {
							delay = n
						}
						headers = headers[:n-delay]
					}
				}
			}
			// Insert all the new headers and fetch the next batch
			if len(headers) > 0 {
				p.log.Trace("Scheduling new headers", "count", len(headers), "from", from)
				select {
				case d.headerProcCh <- headers:
				case <-d.cancelCh:
					return errCanceled
				}
				from += uint64(len(headers))

				// If we're still skeleton filling fast sync, check pivot staleness
				// before continuing to the next skeleton filling
				if skeleton && pivot > 0 {
					getNextPivot()
				} else {
					getHeaders(from)
				}
			} else {
				// No headers delivered, or all of them being delayed, sleep a bit and retry
				p.log.Trace("All headers delayed, waiting")
				select {
				case <-time.After(fsHeaderContCheck):
					getHeaders(from)
					continue
				case <-d.cancelCh:
					return errCanceled
				}
			}

		case <-timeout.C:
			if d.dropPeer == nil {
				// The dropPeer method is nil when `--copydb` is used for a local copy.
				// Timeouts can occur if e.g. compaction hits at the wrong time, and can be ignored
				p.log.Warn("Downloader wants to drop peer, but peerdrop-function is not set", "peer", p.id)
				break
			}
			// Header retrieval timed out, consider the peer bad and drop
			p.log.Debug("Header request timed out", "elapsed", ttl)
			headerTimeoutMeter.Mark(1)
			d.dropPeer(p.id)

			// Finish the sync gracefully instead of dumping the gathered data though
			for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
				select {
				case ch <- false:
				case <-d.cancelCh:
				}
			}
			select {
			case d.headerProcCh <- nil:
			case <-d.cancelCh:
			}
			return fmt.Errorf("%w: header request timed out", errBadPeer)
		}
	}
}

```

processHeaders方法，这个方法从headerProcCh通道来获取header。并把获取到的header丢入到queue来进行调度，这样body fetcher或者是receipt fetcher就可以领取到fetch任务。


```go
// processHeaders takes batches of retrieved headers from an input channel and
// keeps processing and scheduling them into the header chain and downloader's
// queue until the stream ends or a failure occurs.
func (d *Downloader) processHeaders(origin uint64, td *big.Int) error {
	// Keep a count of uncertain headers to roll back
	var (
		rollback    uint64 // Zero means no rollback (fine as you can't unroll the genesis)
		rollbackErr error
		mode        = d.getMode()
	)
	defer func() {
		if rollback > 0 {
			lastHeader, lastFastBlock, lastBlock := d.lightchain.CurrentHeader().Number, common.Big0, common.Big0
			if mode != LightSync {
				lastFastBlock = d.blockchain.CurrentFastBlock().Number()
				lastBlock = d.blockchain.CurrentBlock().Number()
			}
			if err := d.lightchain.SetHead(rollback - 1); err != nil { // -1 to target the parent of the first uncertain block
				// We're already unwinding the stack, only print the error to make it more visible
				log.Error("Failed to roll back chain segment", "head", rollback-1, "err", err)
			}
			curFastBlock, curBlock := common.Big0, common.Big0
			if mode != LightSync {
				curFastBlock = d.blockchain.CurrentFastBlock().Number()
				curBlock = d.blockchain.CurrentBlock().Number()
			}
			log.Warn("Rolled back chain segment",
				"header", fmt.Sprintf("%d->%d", lastHeader, d.lightchain.CurrentHeader().Number),
				"fast", fmt.Sprintf("%d->%d", lastFastBlock, curFastBlock),
				"block", fmt.Sprintf("%d->%d", lastBlock, curBlock), "reason", rollbackErr)
		}
	}()
	// Wait for batches of headers to process
	gotHeaders := false

	for {
		select {
		case <-d.cancelCh:
			rollbackErr = errCanceled
			return errCanceled

		case headers := <-d.headerProcCh:
			// Terminate header processing if we synced up
			if len(headers) == 0 {
				// Notify everyone that headers are fully processed
				for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
					select {
					case ch <- false:
					case <-d.cancelCh:
					}
				}
				// If no headers were retrieved at all, the peer violated its TD promise that it had a
				// better chain compared to ours. The only exception is if its promised blocks were
				// already imported by other means (e.g. fetcher):
				//
				// R <remote peer>, L <local node>: Both at block 10
				// R: Mine block 11, and propagate it to L
				// L: Queue block 11 for import
				// L: Notice that R's head and TD increased compared to ours, start sync
				// L: Import of block 11 finishes
				// L: Sync begins, and finds common ancestor at 11
				// L: Request new headers up from 11 (R's TD was higher, it must have something)
				// R: Nothing to give
				if mode != LightSync {
					head := d.blockchain.CurrentBlock()
					if !gotHeaders && td.Cmp(d.blockchain.GetTd(head.Hash(), head.NumberU64())) > 0 {
						return errStallingPeer
					}
				}
				// If fast or light syncing, ensure promised headers are indeed delivered. This is
				// needed to detect scenarios where an attacker feeds a bad pivot and then bails out
				// of delivering the post-pivot blocks that would flag the invalid content.
				//
				// This check cannot be executed "as is" for full imports, since blocks may still be
				// queued for processing when the header download completes. However, as long as the
				// peer gave us something useful, we're already happy/progressed (above check).
				if mode == FastSync || mode == LightSync {
					head := d.lightchain.CurrentHeader()
					if td.Cmp(d.lightchain.GetTd(head.Hash(), head.Number.Uint64())) > 0 {
						return errStallingPeer
					}
				}
				// Disable any rollback and return
				rollback = 0
				return nil
			}
			// Otherwise split the chunk of headers into batches and process them
			gotHeaders = true
			for len(headers) > 0 {
				// Terminate if something failed in between processing chunks
				select {
				case <-d.cancelCh:
					rollbackErr = errCanceled
					return errCanceled
				default:
				}
				// Select the next chunk of headers to import
				limit := maxHeadersProcess
				if limit > len(headers) {
					limit = len(headers)
				}
				chunk := headers[:limit]

				// In case of header only syncing, validate the chunk immediately
				if mode == FastSync || mode == LightSync {
					// If we're importing pure headers, verify based on their recentness
					var pivot uint64

					d.pivotLock.RLock()
					if d.pivotHeader != nil {
						pivot = d.pivotHeader.Number.Uint64()
					}
					d.pivotLock.RUnlock()

					frequency := fsHeaderCheckFrequency
					if chunk[len(chunk)-1].Number.Uint64()+uint64(fsHeaderForceVerify) > pivot {
						frequency = 1
					}
					if n, err := d.lightchain.InsertHeaderChain(chunk, frequency); err != nil {
						rollbackErr = err

						// If some headers were inserted, track them as uncertain
						if (mode == FastSync || frequency > 1) && n > 0 && rollback == 0 {
							rollback = chunk[0].Number.Uint64()
						}
						log.Warn("Invalid header encountered", "number", chunk[n].Number, "hash", chunk[n].Hash(), "parent", chunk[n].ParentHash, "err", err)
						return fmt.Errorf("%w: %v", errInvalidChain, err)
					}
					// All verifications passed, track all headers within the alloted limits
					if mode == FastSync {
						head := chunk[len(chunk)-1].Number.Uint64()
						if head-rollback > uint64(fsHeaderSafetyNet) {
							rollback = head - uint64(fsHeaderSafetyNet)
						} else {
							rollback = 1
						}
					}
				}
				// Unless we're doing light chains, schedule the headers for associated content retrieval
				if mode == FullSync || mode == FastSync {
					// If we've reached the allowed number of pending headers, stall a bit
					for d.queue.PendingBlocks() >= maxQueuedHeaders || d.queue.PendingReceipts() >= maxQueuedHeaders {
						select {
						case <-d.cancelCh:
							rollbackErr = errCanceled
							return errCanceled
						case <-time.After(time.Second):
						}
					}
					// Otherwise insert the headers for content retrieval
					inserts := d.queue.Schedule(chunk, origin)
					if len(inserts) != len(chunk) {
						rollbackErr = fmt.Errorf("stale headers: len inserts %v len(chunk) %v", len(inserts), len(chunk))
						return fmt.Errorf("%w: stale headers", errBadPeer)
					}
				}
				headers = headers[limit:]
				origin += uint64(limit)
			}
			// Update the highest block number we know if a higher one is found.
			d.syncStatsLock.Lock()
			if d.syncStatsChainHeight < origin {
				d.syncStatsChainHeight = origin - 1
			}
			d.syncStatsLock.Unlock()

			// Signal the content downloaders of the availablility of new tasks
			for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
				select {
				case ch <- true:
				default:
				}
			}
		}
	}
}
```


## bodies处理
fetchBodies函数定义了一些闭包函数，然后调用了fetchParts函数

```go

// 区块体
// fetchBodies iteratively downloads the scheduled block bodies, taking any
// available peers, reserving a chunk of blocks for each, waiting for delivery
// and also periodically checking for timeouts.
func (d *Downloader) fetchBodies(from uint64) error {
	log.Debug("Downloading block bodies", "origin", from)

	var (
		deliver = func(packet dataPack) (int, error) {
			pack := packet.(*bodyPack)
			return d.queue.DeliverBodies(pack.peerID, pack.transactions, pack.uncles)
		}
		expire   = func() map[string]int { return d.queue.ExpireBodies(d.requestTTL()) }
		fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchBodies(req) }
		capacity = func(p *peerConnection) int { return p.BlockCapacity(d.requestRTT()) }
		setIdle  = func(p *peerConnection, accepted int, deliveryTime time.Time) { p.SetBodiesIdle(accepted, deliveryTime) }
	)
	err := d.fetchParts(d.bodyCh, deliver, d.bodyWakeCh, expire,
		d.queue.PendingBlocks, d.queue.InFlightBlocks, d.queue.ReserveBodies,
		d.bodyFetchHook, fetch, d.queue.CancelBodies, capacity, d.peers.BodyIdlePeers, setIdle, "bodies")

	log.Debug("Block body download terminated", "err", err)
	return err
}

// 交易收据
// fetchReceipts iteratively downloads the scheduled block receipts, taking any
// available peers, reserving a chunk of receipts for each, waiting for delivery
// and also periodically checking for timeouts.
func (d *Downloader) fetchReceipts(from uint64) error {
	log.Debug("Downloading transaction receipts", "origin", from)

	var (
		deliver = func(packet dataPack) (int, error) {
			pack := packet.(*receiptPack)
			return d.queue.DeliverReceipts(pack.peerID, pack.receipts)
		}
		expire   = func() map[string]int { return d.queue.ExpireReceipts(d.requestTTL()) }
		fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchReceipts(req) }
		capacity = func(p *peerConnection) int { return p.ReceiptCapacity(d.requestRTT()) }
		setIdle  = func(p *peerConnection, accepted int, deliveryTime time.Time) {
			p.SetReceiptsIdle(accepted, deliveryTime)
		}
	)
	err := d.fetchParts(d.receiptCh, deliver, d.receiptWakeCh, expire,
		d.queue.PendingReceipts, d.queue.InFlightReceipts, d.queue.ReserveReceipts,
		d.receiptFetchHook, fetch, d.queue.CancelReceipts, capacity, d.peers.ReceiptIdlePeers, setIdle, "receipts")

	log.Debug("Transaction receipt download terminated", "err", err)
	return err
}

// fetchParts iteratively downloads scheduled block parts, taking any available
// peers, reserving a chunk of fetch requests for each, waiting for delivery and
// also periodically checking for timeouts.
//
// As the scheduling/timeout logic mostly is the same for all downloaded data
// types, this method is used by each for data gathering and is instrumented with
// various callbacks to handle the slight differences between processing them.
//
// The instrumentation parameters:
//  - errCancel:   error type to return if the fetch operation is cancelled (mostly makes logging nicer)
//  - deliveryCh:  channel from which to retrieve downloaded data packets (merged from all concurrent peers)
//  - deliver:     processing callback to deliver data packets into type specific download queues (usually within `queue`)
//  - wakeCh:      notification channel for waking the fetcher when new tasks are available (or sync completed)
//  - expire:      task callback method to abort requests that took too long and return the faulty peers (traffic shaping)
//  - pending:     task callback for the number of requests still needing download (detect completion/non-completability)
//  - inFlight:    task callback for the number of in-progress requests (wait for all active downloads to finish)
//  - throttle:    task callback to check if the processing queue is full and activate throttling (bound memory use)
//  - reserve:     task callback to reserve new download tasks to a particular peer (also signals partial completions)
//  - fetchHook:   tester callback to notify of new tasks being initiated (allows testing the scheduling logic)
//  - fetch:       network callback to actually send a particular download request to a physical remote peer
//  - cancel:      task callback to abort an in-flight download request and allow rescheduling it (in case of lost peer)
//  - capacity:    network callback to retrieve the estimated type-specific bandwidth capacity of a peer (traffic shaping)
//  - idle:        network callback to retrieve the currently (type specific) idle peers that can be assigned tasks
//  - setIdle:     network callback to set a peer back to idle and update its estimated capacity (traffic shaping)
//  - kind:        textual label of the type being downloaded to display in log messages
func (d *Downloader) fetchParts(deliveryCh chan dataPack, deliver func(dataPack) (int, error), wakeCh chan bool,
	expire func() map[string]int, pending func() int, inFlight func() bool, reserve func(*peerConnection, int) (*fetchRequest, bool, bool),
	fetchHook func([]*types.Header), fetch func(*peerConnection, *fetchRequest) error, cancel func(*fetchRequest), capacity func(*peerConnection) int,
	idle func() ([]*peerConnection, int), setIdle func(*peerConnection, int, time.Time), kind string) error {

	// Create a ticker to detect expired retrieval tasks
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	update := make(chan struct{}, 1)

	// Prepare the queue and fetch block parts until the block header fetcher's done
	finished := false
	for {
		select {
		case <-d.cancelCh:
			return errCanceled

		case packet := <-deliveryCh:
			deliveryTime := time.Now()
			// If the peer was previously banned and failed to deliver its pack
			// in a reasonable time frame, ignore its message.
			if peer := d.peers.Peer(packet.PeerId()); peer != nil {
				// Deliver the received chunk of data and check chain validity
				accepted, err := deliver(packet)
				if errors.Is(err, errInvalidChain) {
					return err
				}
				// Unless a peer delivered something completely else than requested (usually
				// caused by a timed out request which came through in the end), set it to
				// idle. If the delivery's stale, the peer should have already been idled.
				if !errors.Is(err, errStaleDelivery) {
					setIdle(peer, accepted, deliveryTime)
				}
				// Issue a log to the user to see what's going on
				switch {
				case err == nil && packet.Items() == 0:
					peer.log.Trace("Requested data not delivered", "type", kind)
				case err == nil:
					peer.log.Trace("Delivered new batch of data", "type", kind, "count", packet.Stats())
				default:
					peer.log.Trace("Failed to deliver retrieved data", "type", kind, "err", err)
				}
			}
			// Blocks assembled, try to update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case cont := <-wakeCh:
			// The header fetcher sent a continuation flag, check if it's done
			if !cont {
				finished = true
			}
			// Headers arrive, try to update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case <-ticker.C:
			// Sanity check update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case <-update:
			// Short circuit if we lost all our peers
			if d.peers.Len() == 0 {
				return errNoPeers
			}
			// Check for fetch request timeouts and demote the responsible peers
			for pid, fails := range expire() {
				if peer := d.peers.Peer(pid); peer != nil {
					// If a lot of retrieval elements expired, we might have overestimated the remote peer or perhaps
					// ourselves. Only reset to minimal throughput but don't drop just yet. If even the minimal times
					// out that sync wise we need to get rid of the peer.
					//
					// The reason the minimum threshold is 2 is because the downloader tries to estimate the bandwidth
					// and latency of a peer separately, which requires pushing the measures capacity a bit and seeing
					// how response times reacts, to it always requests one more than the minimum (i.e. min 2).
					if fails > 2 {
						peer.log.Trace("Data delivery timed out", "type", kind)
						setIdle(peer, 0, time.Now())
					} else {
						peer.log.Debug("Stalling delivery, dropping", "type", kind)

						if d.dropPeer == nil {
							// The dropPeer method is nil when `--copydb` is used for a local copy.
							// Timeouts can occur if e.g. compaction hits at the wrong time, and can be ignored
							peer.log.Warn("Downloader wants to drop peer, but peerdrop-function is not set", "peer", pid)
						} else {
							d.dropPeer(pid)

							// If this peer was the master peer, abort sync immediately
							d.cancelLock.RLock()
							master := pid == d.cancelPeer
							d.cancelLock.RUnlock()

							if master {
								d.cancel()
								return errTimeout
							}
						}
					}
				}
			}
			// If there's nothing more to fetch, wait or terminate
			if pending() == 0 {
				if !inFlight() && finished {
					log.Debug("Data fetching completed", "type", kind)
					return nil
				}
				break
			}
			// Send a download request to all idle peers, until throttled
			progressed, throttled, running := false, false, inFlight()
			idles, total := idle()
			pendCount := pending()
			for _, peer := range idles {
				// Short circuit if throttling activated
				if throttled {
					break
				}
				// Short circuit if there is no more available task.
				if pendCount = pending(); pendCount == 0 {
					break
				}
				// Reserve a chunk of fetches for a peer. A nil can mean either that
				// no more headers are available, or that the peer is known not to
				// have them.
				request, progress, throttle := reserve(peer, capacity(peer))
				if progress {
					progressed = true
				}
				if throttle {
					throttled = true
					throttleCounter.Inc(1)
				}
				if request == nil {
					continue
				}
				if request.From > 0 {
					peer.log.Trace("Requesting new batch of data", "type", kind, "from", request.From)
				} else {
					peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Headers), "from", request.Headers[0].Number)
				}
				// Fetch the chunk and make sure any errors return the hashes to the queue
				if fetchHook != nil {
					fetchHook(request.Headers)
				}
				if err := fetch(peer, request); err != nil {
					// Although we could try and make an attempt to fix this, this error really
					// means that we've double allocated a fetch task to a peer. If that is the
					// case, the internal state of the downloader and the queue is very wrong so
					// better hard crash and note the error instead of silently accumulating into
					// a much bigger issue.
					panic(fmt.Sprintf("%v: %s fetch assignment failed", peer, kind))
				}
				running = true
			}
			// Make sure that we have peers available for fetching. If all peers have been tried
			// and all failed throw an error
			if !progressed && !throttled && !running && len(idles) == total && pendCount > 0 {
				return errPeersUnavailable
			}
		}
	}
}
```





## processFastSyncContent 和 processFullSyncContent



```go
// processFastSyncContent从队列中获取结果并将其写入数据库。它还控制枢轴块状态节点的同步。
// processFastSyncContent takes fetch results from the queue and writes them to the
// database. It also controls the synchronisation of state nodes of the pivot block.
func (d *Downloader) processFastSyncContent() error {
	// Start syncing state of the reported head block. This should get us most of
	// the state of the pivot block.
	d.pivotLock.RLock()
	sync := d.syncState(d.pivotHeader.Root)
	d.pivotLock.RUnlock()

	defer func() {
		// The `sync` object is replaced every time the pivot moves. We need to
		// defer close the very last active one, hence the lazy evaluation vs.
		// calling defer sync.Cancel() !!!
		sync.Cancel()
	}()

	closeOnErr := func(s *stateSync) {
		if err := s.Wait(); err != nil && err != errCancelStateFetch && err != errCanceled {
			d.queue.Close() // wake up Results
		}
	}
	go closeOnErr(sync)

	// To cater for moving pivot points, track the pivot block and subsequently
	// accumulated download results separately.
	var (
		oldPivot *fetchResult   // Locked in pivot block, might change eventually
		oldTail  []*fetchResult // Downloaded content after the pivot
	)

	for {
		// Wait for the next batch of downloaded data to be available, and if the pivot
		// block became stale, move the goalpost
		results := d.queue.Results(oldPivot == nil) // Block if we're not monitoring pivot staleness
		if len(results) == 0 {
			// If pivot sync is done, stop
			if oldPivot == nil {
				return sync.Cancel()
			}
			// If sync failed, stop
			select {
			case <-d.cancelCh:
				sync.Cancel()
				return errCanceled
			default:
			}
		}
		if d.chainInsertHook != nil {
			d.chainInsertHook(results)
		}
		// If we haven't downloaded the pivot block yet, check pivot staleness
		// notifications from the header downloader
		d.pivotLock.RLock()
		pivot := d.pivotHeader
		d.pivotLock.RUnlock()

		if oldPivot == nil {
			if pivot.Root != sync.root {
				sync.Cancel()
				sync = d.syncState(pivot.Root)

				go closeOnErr(sync)
			}
		} else {
			results = append(append([]*fetchResult{oldPivot}, oldTail...), results...)
		}
		// Split around the pivot block and process the two sides via fast/full sync
		if atomic.LoadInt32(&d.committed) == 0 {
			latest := results[len(results)-1].Header
			// If the height is above the pivot block by 2 sets, it means the pivot
			// become stale in the network and it was garbage collected, move to a
			// new pivot.
			//
			// Note, we have `reorgProtHeaderDelay` number of blocks withheld, Those
			// need to be taken into account, otherwise we're detecting the pivot move
			// late and will drop peers due to unavailable state!!!
			if height := latest.Number.Uint64(); height >= pivot.Number.Uint64()+2*uint64(fsMinFullBlocks)-uint64(reorgProtHeaderDelay) {
				log.Warn("Pivot became stale, moving", "old", pivot.Number.Uint64(), "new", height-uint64(fsMinFullBlocks)+uint64(reorgProtHeaderDelay))
				pivot = results[len(results)-1-fsMinFullBlocks+reorgProtHeaderDelay].Header // must exist as lower old pivot is uncommitted

				d.pivotLock.Lock()
				d.pivotHeader = pivot
				d.pivotLock.Unlock()

				// Write out the pivot into the database so a rollback beyond it will
				// reenable fast sync
				rawdb.WriteLastPivotNumber(d.stateDB, pivot.Number.Uint64())
			}
		}
		P, beforeP, afterP := splitAroundPivot(pivot.Number.Uint64(), results)
		if err := d.commitFastSyncData(beforeP, sync); err != nil {
			return err
		}
		if P != nil {
			// If new pivot block found, cancel old state retrieval and restart
			if oldPivot != P {
				sync.Cancel()
				sync = d.syncState(P.Header.Root)

				go closeOnErr(sync)
				oldPivot = P
			}
			// Wait for completion, occasionally checking for pivot staleness
			select {
			case <-sync.done:
				if sync.err != nil {
					return sync.err
				}
				if err := d.commitPivotBlock(P); err != nil {
					return err
				}
				oldPivot = nil

			case <-time.After(time.Second):
				oldTail = afterP
				continue
			}
		}
		// Fast sync done, pivot commit done, full import
		if err := d.importBlockResults(afterP); err != nil {
			return err
		}
	}
}

```



```go
//processFullSyncContent从队列中获取结果，并将其导入到链中。
// processFullSyncContent takes fetch results from the queue and imports them into the chain.
func (d *Downloader) processFullSyncContent() error {
	for {
		results := d.queue.Results(true)
		if len(results) == 0 {
			return nil
		}
		if d.chainInsertHook != nil {
			d.chainInsertHook(results)
		}
		if err := d.importBlockResults(results); err != nil {
			return err
		}
	}
}
```