fetcher包含基于块通知的同步。当我们接收到NewBlockHashesMsg消息得时候，我们只收到了很多Block的hash值。 需要通过hash值来同步区块，然后更新本地区块链。 fetcher就提供了这样的功能。


数据结构

```go
// blockAnnounce is the hash notification of the availability of a new block in the
// network.
type blockAnnounce struct {
	hash   common.Hash   // Hash of the block being announced
	number uint64        // Number of the block being announced (0 = unknown | old protocol)
	header *types.Header // Header of the block partially reassembled (new protocol)
	time   time.Time     // Timestamp of the announcement

	origin string // Identifier of the peer originating the notification

	fetchHeader headerRequesterFn // Fetcher function to retrieve the header of an announced block
	fetchBodies bodyRequesterFn   // Fetcher function to retrieve the body of an announced block
}

// headerFilterTask represents a batch of headers needing fetcher filtering.
type headerFilterTask struct {
	peer    string          // The source peer of block headers
	headers []*types.Header // Collection of headers to filter
	time    time.Time       // Arrival time of the headers
}

// bodyFilterTask represents a batch of block bodies (transactions and uncles)
// needing fetcher filtering.
type bodyFilterTask struct {
	peer         string                 // The source peer of block bodies
	transactions [][]*types.Transaction // Collection of transactions per block bodies
	uncles       [][]*types.Header      // Collection of uncles per block bodies
	time         time.Time              // Arrival time of the blocks' contents
}

// blockOrHeaderInject represents a schedules import operation.
type blockOrHeaderInject struct {
	origin string

	header *types.Header // Used for light mode fetcher which only cares about header.
	block  *types.Block  // Used for normal mode fetcher which imports full block.
}

// BlockFetcher负责累积来自各个对等方的区块公告,并安排它们进行检索。 
// BlockFetcher is responsible for accumulating block announcements from various peers
// and scheduling them for retrieval.
type BlockFetcher struct {
	light bool // The indicator whether it's a light fetcher or normal one.

	// Various event channels
	notify chan *blockAnnounce
	inject chan *blockOrHeaderInject

	headerFilter chan chan *headerFilterTask // 区块头过滤
	bodyFilter   chan chan *bodyFilterTask // 过滤

	done chan common.Hash // 完成
	quit chan struct{} // 退出

	// Announce states
	announces  map[string]int                   // Per peer blockAnnounce counts to prevent memory exhaustion
	announced  map[common.Hash][]*blockAnnounce // Announced blocks, scheduled for fetching
	fetching   map[common.Hash]*blockAnnounce   // Announced blocks, currently fetching
	fetched    map[common.Hash][]*blockAnnounce // Blocks with headers fetched, scheduled for body retrieval
	completing map[common.Hash]*blockAnnounce   // Blocks with headers, currently body-completing

	// Block cache
	queue  *prque.Prque                         // Queue containing the import operations (block number sorted)
	queues map[string]int                       // Per peer block counts to prevent memory exhaustion
	queued map[common.Hash]*blockOrHeaderInject // Set of already queued blocks (to dedup imports)

	// Callbacks
	getHeader      HeaderRetrievalFn  // Retrieves a header from the local chain
	getBlock       blockRetrievalFn   // Retrieves a block from the local chain
	verifyHeader   headerVerifierFn   // Checks if a block's headers have a valid proof of work
	broadcastBlock blockBroadcasterFn // Broadcasts a block to connected peers
	chainHeight    chainHeightFn      // Retrieves the current chain's height
	insertHeaders  headersInsertFn    // Injects a batch of headers into the chain
	insertChain    chainInsertFn      // Injects a batch of blocks into the chain
	dropPeer       peerDropFn         // Drops a peer for misbehaving

	// Testing hooks
	announceChangeHook func(common.Hash, bool)           // Method to call upon adding or deleting a hash from the blockAnnounce list
	queueChangeHook    func(common.Hash, bool)           // Method to call upon adding or deleting a block from the import queue
	fetchingHook       func([]common.Hash)               // Method to call upon starting a block (eth/61) or header (eth/62) fetch
	completingHook     func([]common.Hash)               // Method to call upon starting a block body fetch (eth/62)
	importedHook       func(*types.Header, *types.Block) // Method to call upon successful header or block import (both eth/61 and eth/62)
}

```


```go
// 启动一个同步器, 接收并处理hash通知以及区块获取
// Start boots up the announcement based synchroniser, accepting and processing
// hash notifications and block fetches until termination requested.
func (f *BlockFetcher) Start() {
	go f.loop()
}

// 停止
// Stop terminates the announcement based synchroniser, canceling all pending
// operations.
func (f *BlockFetcher) Stop() {
	close(f.quit)
}
```





loop函数函数太长。 我先帖一个省略版本的出来。fetcher通过四个map(announced,fetching,fetched,completing )记录了announce的状态(等待fetch,正在fetch,fetch完头等待fetch body, fetch完成)。 loop其实通过定时器和各种消息来对各种map里面的announce进行状态转换。

```go
// Loop is the main fetcher loop, checking and processing various notification
// events.
func (f *BlockFetcher) loop() {

    // 两个定时器
	// Iterate the block fetching until a quit is requested
	fetchTimer := time.NewTimer(0)
	completeTimer := time.NewTimer(0)
	defer fetchTimer.Stop()
	defer completeTimer.Stop()

	for {
		 // 清理所有过期的fetch
		// Clean up any expired block fetches
		for hash, announce := range f.fetching {
			if time.Since(announce.time) > fetchTimeout {
				f.forgetHash(hash)
			}
		}

		// 导入区块
		// Import any queued blocks that could potentially fit
		height := f.chainHeight() // 当前区块
		for !f.queue.Empty() {
			op := f.queue.PopItem().(*blockOrHeaderInject)
			hash := op.hash()
			if f.queueChangeHook != nil {
				f.queueChangeHook(hash, false)
			}
			// If too high up the chain or phase, continue later
			number := op.number()
			if number > height+1 {
				f.queue.Push(op, -int64(number)) // 区块高度的相反数, queue是一个大根堆
				if f.queueChangeHook != nil {
					f.queueChangeHook(hash, true)
				}
				break
			}
			// Otherwise if fresh and still unknown, try and import
			if (number+maxUncleDist < height) || (f.light && f.getHeader(hash) != nil) || (!f.light && f.getBlock(hash) != nil) {
				f.forgetBlock(hash)
				continue
			}
			if f.light { // 轻节点只导入区块头
				f.importHeaders(op.origin, op.header)
			} else { // 导入区块
				f.importBlocks(op.origin, op.block)
			}
		}

		// 等待外部事件发生
		// Wait for an outside event to occur
		select {
		case <-f.quit: // 结束
			// BlockFetcher terminating, abort all operations
			return

		case notification := <-f.notify:  // 区块通知
			// A block was announced, make sure the peer isn't DOSing us
			blockAnnounceInMeter.Mark(1)

			count := f.announces[notification.origin] + 1
			if count > hashLimit {
				log.Debug("Peer exceeded outstanding announces", "peer", notification.origin, "limit", hashLimit)
				blockAnnounceDOSMeter.Mark(1)
				break
			}

			// 如果我们有一个有效的块号，请检查它是否可能有用
			// If we have a valid block number, check that it's potentially useful
			if notification.number > 0 {
				if dist := int64(notification.number) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
					log.Debug("Peer discarded announcement", "peer", notification.origin, "number", notification.number, "hash", notification.hash, "distance", dist)
					blockAnnounceDropMeter.Mark(1)
					break
				}
			}

			// 一切正常，如果尚未下载块，请安排发布
			// All is well, schedule the announce if block's not yet downloading
			if _, ok := f.fetching[notification.hash]; ok {
				break
			}
			if _, ok := f.completing[notification.hash]; ok {
				break
			}
			f.announces[notification.origin] = count
			f.announced[notification.hash] = append(f.announced[notification.hash], notification)
			if f.announceChangeHook != nil && len(f.announced[notification.hash]) == 1 {
				f.announceChangeHook(notification.hash, true)
			}
			if len(f.announced) == 1 {
				f.rescheduleFetch(fetchTimer)
			}

		case op := <-f.inject: 
			// 请求直接插入块，尝试填补所有未决的空白 
			// A direct block insertion was requested, try and fill any pending gaps
			blockBroadcastInMeter.Mark(1)

			// 轻节点仅允许直接块注入，如果我们收到,默默地删除标头注入
			// Now only direct block injection is allowed, drop the header injection
			// here silently if we receive.
			if f.light {
				continue
			}
			f.enqueue(op.origin, nil, op.block)

		case hash := <-f.done: 
			// 待完成的导入已完成，请删除该通知的所有痕迹 
			// A pending import finished, remove all traces of the notification
			f.forgetHash(hash)
			f.forgetBlock(hash)

		case <-fetchTimer.C:

			// 至少一个块的计时器用完了，检查是否需要检索 
			// At least one block's timer ran out, check for needing retrieval
			request := make(map[string][]common.Hash)

			for hash, announces := range f.announced {
				// In current LES protocol(les2/les3), only header announce is
				// available, no need to wait too much time for header broadcast.
				timeout := arriveTimeout - gatherSlack
				if f.light {
					timeout = 0
				}
				if time.Since(announces[0].time) > timeout {
					// Pick a random peer to retrieve from, reset all others
					announce := announces[rand.Intn(len(announces))]
					f.forgetHash(hash)

					// If the block still didn't arrive, queue for fetching
					if (f.light && f.getHeader(hash) == nil) || (!f.light && f.getBlock(hash) == nil) {
						request[announce.origin] = append(request[announce.origin], hash)
						f.fetching[hash] = announce
					}
				}
			}
			// Send out all block header requests
			for peer, hashes := range request {
				log.Trace("Fetching scheduled headers", "peer", peer, "list", hashes)

				// Create a closure of the fetch and schedule in on a new thread
				fetchHeader, hashes := f.fetching[hashes[0]].fetchHeader, hashes
				go func() {
					if f.fetchingHook != nil {
						f.fetchingHook(hashes)
					}
					for _, hash := range hashes {
						headerFetchMeter.Mark(1)
						fetchHeader(hash) // Suboptimal, but protocol doesn't allow batch header retrievals
					}
				}()
			}
			// Schedule the next fetch if blocks are still pending
			f.rescheduleFetch(fetchTimer)

		case <-completeTimer.C: // 至少有一个标头的计时器用尽，检索所有内容
			// At least one header's timer ran out, retrieve everything
			request := make(map[string][]common.Hash)

			for hash, announces := range f.fetched {
				// Pick a random peer to retrieve from, reset all others
				announce := announces[rand.Intn(len(announces))]
				f.forgetHash(hash)

				// If the block still didn't arrive, queue for completion
				if f.getBlock(hash) == nil {
					request[announce.origin] = append(request[announce.origin], hash)
					f.completing[hash] = announce
				}
			}
			// Send out all block body requests
			for peer, hashes := range request {
				log.Trace("Fetching scheduled bodies", "peer", peer, "list", hashes)

				// Create a closure of the fetch and schedule in on a new thread
				if f.completingHook != nil {
					f.completingHook(hashes)
				}
				bodyFetchMeter.Mark(int64(len(hashes)))
				go f.completing[hashes[0]].fetchBodies(hashes)
			}
			// Schedule the next fetch if blocks are still pending
			f.rescheduleComplete(completeTimer)

		case filter := <-f.headerFilter:
            /// 区块头
			// Headers arrived from a remote peer. Extract those that were explicitly
			// requested by the fetcher, and return everything else so it's delivered
			// to other parts of the system.
			var task *headerFilterTask
			select {
			case task = <-filter:
			case <-f.quit:
				return
			}
			headerFilterInMeter.Mark(int64(len(task.headers)))

			// Split the batch of headers into unknown ones (to return to the caller),
			// known incomplete ones (requiring body retrievals) and completed blocks.
			unknown, incomplete, complete, lightHeaders := []*types.Header{}, []*blockAnnounce{}, []*types.Block{}, []*blockAnnounce{}
			for _, header := range task.headers {
				hash := header.Hash()

				// Filter fetcher-requested headers from other synchronisation algorithms
				if announce := f.fetching[hash]; announce != nil && announce.origin == task.peer && f.fetched[hash] == nil && f.completing[hash] == nil && f.queued[hash] == nil {
					// If the delivered header does not match the promised number, drop the announcer
					if header.Number.Uint64() != announce.number {
						log.Trace("Invalid block number fetched", "peer", announce.origin, "hash", header.Hash(), "announced", announce.number, "provided", header.Number)
						f.dropPeer(announce.origin)
						f.forgetHash(hash)
						continue
					}
					// Collect all headers only if we are running in light
					// mode and the headers are not imported by other means.
					if f.light {
						if f.getHeader(hash) == nil {
							announce.header = header
							lightHeaders = append(lightHeaders, announce)
						}
						f.forgetHash(hash)
						continue
					}
					// Only keep if not imported by other means
					if f.getBlock(hash) == nil {
						announce.header = header
						announce.time = task.time

						// If the block is empty (header only), short circuit into the final import queue
						if header.TxHash == types.EmptyRootHash && header.UncleHash == types.EmptyUncleHash {
							log.Trace("Block empty, skipping body retrieval", "peer", announce.origin, "number", header.Number, "hash", header.Hash())

							block := types.NewBlockWithHeader(header)
							block.ReceivedAt = task.time

							complete = append(complete, block)
							f.completing[hash] = announce
							continue
						}
						// Otherwise add to the list of blocks needing completion
						incomplete = append(incomplete, announce)
					} else {
						log.Trace("Block already imported, discarding header", "peer", announce.origin, "number", header.Number, "hash", header.Hash())
						f.forgetHash(hash)
					}
				} else {
					// BlockFetcher doesn't know about it, add to the return list
					unknown = append(unknown, header)
				}
			}
			headerFilterOutMeter.Mark(int64(len(unknown)))
			select {
			case filter <- &headerFilterTask{headers: unknown, time: task.time}:
			case <-f.quit:
				return
			}
			// Schedule the retrieved headers for body completion
			for _, announce := range incomplete {
				hash := announce.header.Hash()
				if _, ok := f.completing[hash]; ok {
					continue
				}
				f.fetched[hash] = append(f.fetched[hash], announce)
				if len(f.fetched) == 1 {
					f.rescheduleComplete(completeTimer)
				}
			}
			// Schedule the header for light fetcher import
			for _, announce := range lightHeaders {
				f.enqueue(announce.origin, announce.header, nil)
			}
			// Schedule the header-only blocks for import
			for _, block := range complete {
				if announce := f.completing[block.Hash()]; announce != nil {
					f.enqueue(announce.origin, nil, block)
				}
			}

		case filter := <-f.bodyFilter: // 区块体
			// Block bodies arrived, extract any explicitly requested blocks, return the rest
			var task *bodyFilterTask
			select {
			case task = <-filter:
			case <-f.quit:
				return
			}
			bodyFilterInMeter.Mark(int64(len(task.transactions)))
			blocks := []*types.Block{}
			// abort early if there's nothing explicitly requested
			if len(f.completing) > 0 {
				for i := 0; i < len(task.transactions) && i < len(task.uncles); i++ {
					// Match up a body to any possible completion request
					var (
						matched   = false
						uncleHash common.Hash // calculated lazily and reused
						txnHash   common.Hash // calculated lazily and reused
					)
					for hash, announce := range f.completing {
						if f.queued[hash] != nil || announce.origin != task.peer {
							continue
						}
						if uncleHash == (common.Hash{}) {
							uncleHash = types.CalcUncleHash(task.uncles[i])
						}
						if uncleHash != announce.header.UncleHash {
							continue
						}
						if txnHash == (common.Hash{}) {
							txnHash = types.DeriveSha(types.Transactions(task.transactions[i]), new(trie.Trie))
						}
						if txnHash != announce.header.TxHash {
							continue
						}
						// Mark the body matched, reassemble if still unknown
						matched = true
						if f.getBlock(hash) == nil {
							block := types.NewBlockWithHeader(announce.header).WithBody(task.transactions[i], task.uncles[i])
							block.ReceivedAt = task.time
							blocks = append(blocks, block)
						} else {
							f.forgetHash(hash)
						}

					}
					if matched {
						task.transactions = append(task.transactions[:i], task.transactions[i+1:]...)
						task.uncles = append(task.uncles[:i], task.uncles[i+1:]...)
						i--
						continue
					}
				}
			}
			bodyFilterOutMeter.Mark(int64(len(task.transactions)))
			select {
			case filter <- task:
			case <-f.quit:
				return
			}
			// Schedule the retrieved blocks for ordered import
			for _, block := range blocks {
				if announce := f.completing[block.Hash()]; announce != nil {
					f.enqueue(announce.origin, nil, block)
				}
			}
		}
	}
}
```



### 区块头的过滤流程
#### FilterHeaders请求
FilterHeaders方法在接收到BlockHeadersMsg的时候被调用。这个方法首先投递了一个channel filter到headerFilter。 然后往filter投递了一个headerFilterTask的任务。然后阻塞等待filter队列返回消息。

```go
//FilterHeaders提取提取程序明确请求的所有标头，
//返回应以不同方式处理的内容。 
// FilterHeaders extracts all the headers that were explicitly requested by the fetcher,
// returning those that should be handled differently.
func (f *BlockFetcher) FilterHeaders(peer string, headers []*types.Header, time time.Time) []*types.Header {
	log.Trace("Filtering headers", "peer", peer, "headers", len(headers))

	// Send the filter channel to the fetcher
	filter := make(chan *headerFilterTask)

	select {
	case f.headerFilter <- filter:
	case <-f.quit:
		return nil
	}
	// Request the filtering of the header list
	select {
	case filter <- &headerFilterTask{peer: peer, headers: headers, time: time}:
	case <-f.quit:
		return nil
	}
	// Retrieve the headers remaining after filtering
	select {
	case task := <-filter:
		return task.headers
	case <-f.quit:
		return nil
	}
}


// FilterBodies提取由显式请求的所有块体, 获取程序，返回应以不同方式处理的内容。
// FilterBodies extracts all the block bodies that were explicitly requested by
// the fetcher, returning those that should be handled differently.
func (f *BlockFetcher) FilterBodies(peer string, transactions [][]*types.Transaction, uncles [][]*types.Header, time time.Time) ([][]*types.Transaction, [][]*types.Header) {
	log.Trace("Filtering bodies", "peer", peer, "txs", len(transactions), "uncles", len(uncles))

	// Send the filter channel to the fetcher
	filter := make(chan *bodyFilterTask)

	select {
	case f.bodyFilter <- filter:
	case <-f.quit:
		return nil, nil
	}
	// Request the filtering of the body list
	select {
	case filter <- &bodyFilterTask{peer: peer, transactions: transactions, uncles: uncles, time: time}:
	case <-f.quit:
		return nil, nil
	}
	// Retrieve the bodies remaining after filtering
	select {
	case task := <-filter:
		return task.transactions, task.uncles
	case <-f.quit:
		return nil, nil
	}
}
```





#### notification的处理
在接收到NewBlockHashesMsg的时候，对于本地区块链还没有的区块的hash值会调用fetcher的Notify方法发送到notify通道。


```go

```


在loop中的处理，主要是检查一下然后加入了announced这个容器等待定时处理。


#### Enqueue处理
在接收到NewBlockMsg的时候会调用fetcher的Enqueue方法，这个方法会把当前接收到的区块发送到inject通道。 可以看到这个方法生成了一个inject对象然后发送到inject通道


```go
// Enqueue tries to fill gaps the fetcher's future import queue.
func (f *BlockFetcher) Enqueue(peer string, block *types.Block) error {
	op := &blockOrHeaderInject{
		origin: peer,
		block:  block,
	}
	select {
	case f.inject <- op:
		return nil
	case <-f.quit:
		return errTerminated
	}
}

// enqueue schedules a new header or block import operation, if the component
// to be imported has not yet been seen.
func (f *BlockFetcher) enqueue(peer string, header *types.Header, block *types.Block) {
	var (
		hash   common.Hash
		number uint64
	)
	if header != nil {
		hash, number = header.Hash(), header.Number.Uint64()
	} else {
		hash, number = block.Hash(), block.NumberU64()
	}
	// Ensure the peer isn't DOSing us
	count := f.queues[peer] + 1
	if count > blockLimit {
		log.Debug("Discarded delivered header or block, exceeded allowance", "peer", peer, "number", number, "hash", hash, "limit", blockLimit)
		blockBroadcastDOSMeter.Mark(1)
		f.forgetHash(hash)
		return
	}
	// Discard any past or too distant blocks
	if dist := int64(number) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
		log.Debug("Discarded delivered header or block, too far away", "peer", peer, "number", number, "hash", hash, "distance", dist)
		blockBroadcastDropMeter.Mark(1)
		f.forgetHash(hash)
		return
	}
	// Schedule the block for future importing
	if _, ok := f.queued[hash]; !ok {
		op := &blockOrHeaderInject{origin: peer}
		if header != nil {
			op.header = header
		} else {
			op.block = block
		}
		f.queues[peer] = count
		f.queued[hash] = op
		f.queue.Push(op, -int64(number))
		if f.queueChangeHook != nil {
			f.queueChangeHook(hash, true)
		}
		log.Debug("Queued delivered header or block", "peer", peer, "number", number, "hash", hash, "queued", f.queue.Size())
	}
}
```



fetcher importBlocks方法

```go
//importBlocks产生一个新的goroutine，以将块插入到链中。如果
//区块的编号与当前导入阶段的高度相同，它会更新
//相应的阶段状态。
// importBlocks spawns a new goroutine to run a block insertion into the chain. If the
// block's number is at the same height as the current import phase, it updates
// the phase states accordingly.
func (f *BlockFetcher) importBlocks(peer string, block *types.Block) {
	hash := block.Hash()

	// Run the import on a new thread
	log.Debug("Importing propagated block", "peer", peer, "number", block.Number(), "hash", hash)
	go func() {
		defer func() { f.done <- hash }()

		// If the parent's unknown, abort insertion
		parent := f.getBlock(block.ParentHash())
		if parent == nil {
			log.Debug("Unknown parent of propagated block", "peer", peer, "number", block.Number(), "hash", hash, "parent", block.ParentHash())
			return
		}
		// Quickly validate the header and propagate the block if it passes
		switch err := f.verifyHeader(block.Header()); err {
		case nil:
			// All ok, quickly propagate to our peers
			blockBroadcastOutTimer.UpdateSince(block.ReceivedAt)
			go f.broadcastBlock(block, true)

		case consensus.ErrFutureBlock:
			// Weird future block, don't fail, but neither propagate

		default:
			// Something went very wrong, drop the peer
			log.Debug("Propagated block verification failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
			f.dropPeer(peer)
			return
		}
		// Run the actual import and log any issues
		if _, err := f.insertChain(types.Blocks{block}); err != nil {
			log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
			return
		}
		// If import succeeded, broadcast the block
		blockAnnounceOutTimer.UpdateSince(block.ReceivedAt)
		go f.broadcastBlock(block, false)

		// Invoke the testing hook if needed
		if f.importedHook != nil {
			f.importedHook(nil, block)
		}
	}()
}
```


#### 定时器的处理

一共存在两个定时器。fetchTimer和completeTimer，分别负责获取区块头和获取区块body。

状态转换: notify-->  announced  --fetchTimer(fetch header)---> fetching  --(headerFilter)--> fetched --completeTimer(fetch body)-->completing --(bodyFilter)--> enqueue --task.done--> forgetHash

> TODO: 发现一个问题。 completing的容器有可能泄露。如果发送了一个hash的body请求。 但是请求失败，对方并没有返回。 这个时候completing容器没有清理。 是否有可能导致问题。


