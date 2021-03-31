queue给downloader提供了调度功能和限流的功能。 通过调用Schedule/ScheduleSkeleton来申请对任务进行调度，然后调用ReserveXXX方法来领取调度完成的任务，并在downloader里面的线程来执行，调用DeliverXXX方法把下载完的数据给queue。 最后通过WaitResults来获取已经完成的任务。中间还有一些对任务的额外控制，ExpireXXX用来控制任务是否超时， CancelXXX用来取消任务。


队列代表需要获取或正在获取的哈希 
```go
// queue represents hashes that are either need fetching or are being fetched
type queue struct {
	mode SyncMode // Synchronisation mode to decide on the block parts to schedule for fetching

	// Headers are "special", they download in batches, supported by a skeleton chain
	headerHead      common.Hash                    // Hash of the last queued header to verify order
	headerTaskPool  map[uint64]*types.Header       // Pending header retrieval tasks, mapping starting indexes to skeleton headers
	headerTaskQueue *prque.Prque                   // Priority queue of the skeleton indexes to fetch the filling headers for
	headerPeerMiss  map[string]map[uint64]struct{} // Set of per-peer header batches known to be unavailable
	headerPendPool  map[string]*fetchRequest       // Currently pending header retrieval operations
	headerResults   []*types.Header                // Result cache accumulating the completed headers
	headerProced    int                            // Number of headers already processed from the results
	headerOffset    uint64                         // Number of the first header in the result cache
	headerContCh    chan bool                      // Channel to notify when header download finishes

    // 以下所有数据检索均基于已组装的标头链
	// All data retrievals below are based on an already assembles header chain
	blockTaskPool  map[common.Hash]*types.Header // Pending block (body) retrieval tasks, mapping hashes to headers
	blockTaskQueue *prque.Prque                  // Priority queue of the headers to fetch the blocks (bodies) for
	blockPendPool  map[string]*fetchRequest      // Currently pending block (body) retrieval operations

	receiptTaskPool  map[common.Hash]*types.Header // Pending receipt retrieval tasks, mapping hashes to headers
	receiptTaskQueue *prque.Prque                  // Priority queue of the headers to fetch the receipts for
	receiptPendPool  map[string]*fetchRequest      // Currently pending receipt retrieval operations

	resultCache *resultStore       // Downloaded but not yet delivered fetch results
	resultSize  common.StorageSize // Approximate size of a block (exponential moving average)

	lock   *sync.RWMutex
	active *sync.Cond
	closed bool

	lastStatLog time.Time
}
```


## Schedule方法

Schedule调用申请对一些区块头进行下载调度。可以看到做了一些合法性检查之后，把任务插入了blockTaskPool，receiptTaskPool，receiptTaskQueue，receiptTaskPool。
TaskPool是Map，用来记录header的hash是否存在。 TaskQueue是优先级队列，优先级是区块的高度的负数， 这样区块高度越小的优先级越高，就实现了首先调度小的任务的功能。

```go
// Schedule为下载队列添加了一组标题以进行调度，返回遇到的新标头。
// Schedule adds a set of headers for the download queue for scheduling, returning
// the new headers encountered.
func (q *queue) Schedule(headers []*types.Header, from uint64) []*types.Header {
	q.lock.Lock()
	defer q.lock.Unlock()

	// Insert all the headers prioritised by the contained block number
	inserts := make([]*types.Header, 0, len(headers))
	for _, header := range headers {
		// Make sure chain order is honoured and preserved throughout
		hash := header.Hash()
		if header.Number == nil || header.Number.Uint64() != from {
			log.Warn("Header broke chain ordering", "number", header.Number, "hash", hash, "expected", from)
			break
		}
		if q.headerHead != (common.Hash{}) && q.headerHead != header.ParentHash {
			log.Warn("Header broke chain ancestry", "number", header.Number, "hash", hash)
			break
		}
		// Make sure no duplicate requests are executed
		// We cannot skip this, even if the block is empty, since this is
		// what triggers the fetchResult creation.
		if _, ok := q.blockTaskPool[hash]; ok {
			log.Warn("Header already scheduled for block fetch", "number", header.Number, "hash", hash)
		} else {
			q.blockTaskPool[hash] = header
			q.blockTaskQueue.Push(header, -int64(header.Number.Uint64()))
		}
		// Queue for receipt retrieval
		if q.mode == FastSync && !header.EmptyReceipts() {
			if _, ok := q.receiptTaskPool[hash]; ok {
				log.Warn("Header already scheduled for receipt fetch", "number", header.Number, "hash", hash)
			} else {
				q.receiptTaskPool[hash] = header
				q.receiptTaskQueue.Push(header, -int64(header.Number.Uint64()))
			}
		}
		inserts = append(inserts, header)
		q.headerHead = hash
		from++
	}
	return inserts
}
```



## ReserveXXX
ReserveXXX方法用来从queue里面领取一些任务来执行。downloader里面的goroutine会调用这个方法来领取一些任务来执行。 这个方法直接调用了reserveHeaders方法。 所有的ReserveXXX方法都会调用reserveHeaders方法，除了传入的参数有一些区别。

```go
//ReserveHeaders为给定的对等端保留一组头，跳过任何先前失败的批次。
// ReserveHeaders reserves a set of headers for the given peer, skipping any
// previously failed batches.
func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
	q.lock.Lock()
	defer q.lock.Unlock()

	// Short circuit if the peer's already downloading something (sanity check to
	// not corrupt state)
	if _, ok := q.headerPendPool[p.id]; ok {
		return nil
	}
	// Retrieve a batch of hashes, skipping previously failed ones
	send, skip := uint64(0), []uint64{}
	for send == 0 && !q.headerTaskQueue.Empty() {
		from, _ := q.headerTaskQueue.Pop()
		if q.headerPeerMiss[p.id] != nil {
			if _, ok := q.headerPeerMiss[p.id][from.(uint64)]; ok {
				skip = append(skip, from.(uint64))
				continue
			}
		}
		send = from.(uint64)
	}
	// Merge all the skipped batches back
	for _, from := range skip {
		q.headerTaskQueue.Push(from, -int64(from))
	}
	// Assemble and return the block download request
	if send == 0 {
		return nil
	}
	request := &fetchRequest{
		Peer: p,
		From: send,
		Time: time.Now(),
	}
	q.headerPendPool[p.id] = request
	return request
}


// ReserveBodies reserves a set of body fetches for the given peer, skipping any
// previously failed downloads. Beside the next batch of needed fetches, it also
// returns a flag whether empty blocks were queued requiring processing.
func (q *queue) ReserveBodies(p *peerConnection, count int) (*fetchRequest, bool, bool) {
	q.lock.Lock()
	defer q.lock.Unlock()

	return q.reserveHeaders(p, count, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool, bodyType)
}


// ReserveReceipts reserves a set of receipt fetches for the given peer, skipping
// any previously failed downloads. Beside the next batch of needed fetches, it
// also returns a flag whether empty receipts were queued requiring importing.
func (q *queue) ReserveReceipts(p *peerConnection, count int) (*fetchRequest, bool, bool) {
	q.lock.Lock()
	defer q.lock.Unlock()

	return q.reserveHeaders(p, count, q.receiptTaskPool, q.receiptTaskQueue, q.receiptPendPool, receiptType)
}   



// reserveHeaders reserves a set of data download operations for a given peer,
// skipping any previously failed ones. This method is a generic version used
// by the individual special reservation functions.
//
// Note, this method expects the queue lock to be already held for writing. The
// reason the lock is not obtained in here is because the parameters already need
// to access the queue, so they already need a lock anyway.
//
// Returns:
//   item     - the fetchRequest
//   progress - whether any progress was made
//   throttle - if the caller should throttle for a while
func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, kind uint) (*fetchRequest, bool, bool) {
	// Short circuit if the pool has been depleted, or if the peer's already
	// downloading something (sanity check not to corrupt state)
	if taskQueue.Empty() {
		return nil, false, true
	}
	if _, ok := pendPool[p.id]; ok {
		return nil, false, false
	}
	// Retrieve a batch of tasks, skipping previously failed ones
	send := make([]*types.Header, 0, count)
	skip := make([]*types.Header, 0)
	progress := false
	throttled := false
	for proc := 0; len(send) < count && !taskQueue.Empty(); proc++ {
		// the task queue will pop items in order, so the highest prio block
		// is also the lowest block number.
		h, _ := taskQueue.Peek()
		header := h.(*types.Header)
		// we can ask the resultcache if this header is within the
		// "prioritized" segment of blocks. If it is not, we need to throttle

		stale, throttle, item, err := q.resultCache.AddFetch(header, q.mode == FastSync)
		if stale {
			// Don't put back in the task queue, this item has already been
			// delivered upstream
			taskQueue.PopItem()
			progress = true
			delete(taskPool, header.Hash())
			proc = proc - 1
			log.Error("Fetch reservation already delivered", "number", header.Number.Uint64())
			continue
		}
		if throttle {
			// There are no resultslots available. Leave it in the task queue
			// However, if there are any left as 'skipped', we should not tell
			// the caller to throttle, since we still want some other
			// peer to fetch those for us
			throttled = len(skip) == 0
			break
		}
		if err != nil {
			// this most definitely should _not_ happen
			log.Warn("Failed to reserve headers", "err", err)
			// There are no resultslots available. Leave it in the task queue
			break
		}
		if item.Done(kind) {
			// If it's a noop, we can skip this task
			delete(taskPool, header.Hash())
			taskQueue.PopItem()
			proc = proc - 1
			progress = true
			continue
		}
		// Remove it from the task queue
		taskQueue.PopItem()
		// Otherwise unless the peer is known not to have the data, add to the retrieve list
		if p.Lacks(header.Hash()) {
			skip = append(skip, header)
		} else {
			send = append(send, header)
		}
	}
	// Merge all the skipped headers back
	for _, header := range skip {
		taskQueue.Push(header, -int64(header.Number.Uint64()))
	}
	if q.resultCache.HasCompletedItems() {
		// Wake Results, resultCache was modified
		q.active.Signal()
	}
	// Assemble and return the block download request
	if len(send) == 0 {
		return nil, progress, throttled
	}
	request := &fetchRequest{
		Peer:    p,
		Headers: send,
		Time:    time.Now(),
	}
	pendPool[p.id] = request
	return request, progress, throttled
}
```




## DeliverXXX
Deliver方法在数据下载完之后会被调用。

```go

//DeliverHeaders将标头检索响应注入标头结果
//缓存。此方法要么接受它收到的所有标头，要么不接受
//如果它们未正确映射到骨架。
//
//如果标头被接受，则该方法将尝试传递集合
//到处理器的就绪标头，以保持管道满载。但是它将
//请勿阻止，以防止其他未完成的交货停滞。
// DeliverHeaders injects a header retrieval response into the header results
// cache. This method either accepts all headers it received, or none of them
// if they do not map correctly to the skeleton.
//
// If the headers are accepted, the method makes an attempt to deliver the set
// of ready headers to the processor to keep the pipeline full. However it will
// not block to prevent stalling other pending deliveries.
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
	q.lock.Lock()
	defer q.lock.Unlock()

	var logger log.Logger
	if len(id) < 16 {
		// Tests use short IDs, don't choke on them
		logger = log.New("peer", id)
	} else {
		logger = log.New("peer", id[:16])
	}
	// Short circuit if the data was never requested
	request := q.headerPendPool[id]
	if request == nil {
		return 0, errNoFetchesPending
	}
	headerReqTimer.UpdateSince(request.Time)
	delete(q.headerPendPool, id)

	// Ensure headers can be mapped onto the skeleton chain
	target := q.headerTaskPool[request.From].Hash()

	accepted := len(headers) == MaxHeaderFetch
	if accepted {
		if headers[0].Number.Uint64() != request.From {
			logger.Trace("First header broke chain ordering", "number", headers[0].Number, "hash", headers[0].Hash(), "expected", request.From)
			accepted = false
		} else if headers[len(headers)-1].Hash() != target {
			logger.Trace("Last header broke skeleton structure ", "number", headers[len(headers)-1].Number, "hash", headers[len(headers)-1].Hash(), "expected", target)
			accepted = false
		}
	}
	if accepted {
		parentHash := headers[0].Hash()
		for i, header := range headers[1:] {
			hash := header.Hash()
			if want := request.From + 1 + uint64(i); header.Number.Uint64() != want {
				logger.Warn("Header broke chain ordering", "number", header.Number, "hash", hash, "expected", want)
				accepted = false
				break
			}
			if parentHash != header.ParentHash {
				logger.Warn("Header broke chain ancestry", "number", header.Number, "hash", hash)
				accepted = false
				break
			}
			// Set-up parent hash for next round
			parentHash = hash
		}
	}
	// If the batch of headers wasn't accepted, mark as unavailable
	if !accepted {
		logger.Trace("Skeleton filling not accepted", "from", request.From)

		miss := q.headerPeerMiss[id]
		if miss == nil {
			q.headerPeerMiss[id] = make(map[uint64]struct{})
			miss = q.headerPeerMiss[id]
		}
		miss[request.From] = struct{}{}

		q.headerTaskQueue.Push(request.From, -int64(request.From))
		return 0, errors.New("delivery not accepted")
	}
	// Clean up a successful fetch and try to deliver any sub-results
	copy(q.headerResults[request.From-q.headerOffset:], headers)
	delete(q.headerTaskPool, request.From)

	ready := 0
	for q.headerProced+ready < len(q.headerResults) && q.headerResults[q.headerProced+ready] != nil {
		ready += MaxHeaderFetch
	}
	if ready > 0 {
		// Headers are ready for delivery, gather them and push forward (non blocking)
		process := make([]*types.Header, ready)
		copy(process, q.headerResults[q.headerProced:q.headerProced+ready])

		select {
		case headerProcCh <- process:
			logger.Trace("Pre-scheduled new headers", "count", len(process), "from", process[0].Number)
			q.headerProced += len(process)
		default:
		}
	}
	// Check for termination and return
	if len(q.headerTaskPool) == 0 {
		q.headerContCh <- false
	}
	return len(headers), nil
}

//DeliverBodies将块体检索响应注入到结果队列中。
//该方法返回交付中接受的块体的数量，
//还唤醒所有等待数据传递的线程。
// DeliverBodies injects a block body retrieval response into the results queue.
// The method returns the number of blocks bodies accepted from the delivery and
// also wakes any threads waiting for data delivery.
func (q *queue) DeliverBodies(id string, txLists [][]*types.Transaction, uncleLists [][]*types.Header) (int, error) {
	q.lock.Lock()
	defer q.lock.Unlock()
	validate := func(index int, header *types.Header) error {
		if types.DeriveSha(types.Transactions(txLists[index]), trie.NewStackTrie(nil)) != header.TxHash {
			return errInvalidBody
		}
		if types.CalcUncleHash(uncleLists[index]) != header.UncleHash {
			return errInvalidBody
		}
		return nil
	}

	reconstruct := func(index int, result *fetchResult) {
		result.Transactions = txLists[index]
		result.Uncles = uncleLists[index]
		result.SetBodyDone()
	}
	return q.deliver(id, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool,
		bodyReqTimer, len(txLists), validate, reconstruct)
}


//DeliverReceipts将收据检索响应注入到结果队列中。
//该方法返回从交货中接受的交易收据的数量
//并唤醒所有等待数据传递的线程。 
// DeliverReceipts injects a receipt retrieval response into the results queue.
// The method returns the number of transaction receipts accepted from the delivery
// and also wakes any threads waiting for data delivery.
func (q *queue) DeliverReceipts(id string, receiptList [][]*types.Receipt) (int, error) {
	q.lock.Lock()
	defer q.lock.Unlock()
	validate := func(index int, header *types.Header) error {
		if types.DeriveSha(types.Receipts(receiptList[index]), trie.NewStackTrie(nil)) != header.ReceiptHash {
			return errInvalidReceipt
		}
		return nil
	}
	reconstruct := func(index int, result *fetchResult) {
		result.Receipts = receiptList[index]
		result.SetReceiptsDone()
	}
	return q.deliver(id, q.receiptTaskPool, q.receiptTaskQueue, q.receiptPendPool,
		receiptReqTimer, len(receiptList), validate, reconstruct)
}

// deliver injects a data retrieval response into the results queue.
//
// Note, this method expects the queue lock to be already held for writing. The
// reason this lock is not obtained in here is because the parameters already need
// to access the queue, so they already need a lock anyway.
func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header,
	taskQueue *prque.Prque, pendPool map[string]*fetchRequest, reqTimer metrics.Timer,
	results int, validate func(index int, header *types.Header) error,
	reconstruct func(index int, result *fetchResult)) (int, error) {

	// Short circuit if the data was never requested
	request := pendPool[id]
	if request == nil {
		return 0, errNoFetchesPending
	}
	reqTimer.UpdateSince(request.Time)
	delete(pendPool, id)

	// If no data items were retrieved, mark them as unavailable for the origin peer
	if results == 0 {
		for _, header := range request.Headers {
			request.Peer.MarkLacking(header.Hash())
		}
	}
	// Assemble each of the results with their headers and retrieved data parts
	var (
		accepted int
		failure  error
		i        int
		hashes   []common.Hash
	)
	for _, header := range request.Headers {
		// Short circuit assembly if no more fetch results are found
		if i >= results {
			break
		}
		// Validate the fields
		if err := validate(i, header); err != nil {
			failure = err
			break
		}
		hashes = append(hashes, header.Hash())
		i++
	}

	for _, header := range request.Headers[:i] {
		if res, stale, err := q.resultCache.GetDeliverySlot(header.Number.Uint64()); err == nil {
			reconstruct(accepted, res)
		} else {
			// else: betweeen here and above, some other peer filled this result,
			// or it was indeed a no-op. This should not happen, but if it does it's
			// not something to panic about
			log.Error("Delivery stale", "stale", stale, "number", header.Number.Uint64(), "err", err)
			failure = errStaleDelivery
		}
		// Clean up a successful fetch
		delete(taskPool, hashes[accepted])
		accepted++
	}
	// Return all failed or missing fetches to the queue
	for _, header := range request.Headers[accepted:] {
		taskQueue.Push(header, -int64(header.Number.Uint64()))
	}
	// Wake up Results
	if accepted > 0 {
		q.active.Signal()
	}
	if failure == nil {
		return accepted, nil
	}
	// If none of the data was good, it's a stale delivery
	if accepted > 0 {
		return accepted, fmt.Errorf("partial failure: %v", failure)
	}
	return accepted, fmt.Errorf("%w: %v", failure, errStaleDelivery)
}
```


## ExpireXXX and CancelXXX
### ExpireXXX
ExpireBodies函数获取了锁，然后直接调用了expire函数。 

	// ExpireBodies checks for in flight block body requests that exceeded a timeout
	// allowance, canceling them and returning the responsible peers for penalisation.
	func (q *queue) ExpireBodies(timeout time.Duration) map[string]int {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		return q.expire(timeout, q.blockPendPool, q.blockTaskQueue, bodyTimeoutMeter)
	}

expire函数，

	// expire is the generic check that move expired tasks from a pending pool back
	// into a task pool, returning all entities caught with expired tasks.
	// expire是通用检查，将过期任务从待处理池移回任务池，返回所有捕获已到期任务的实体。

	func (q *queue) expire(timeout time.Duration, pendPool map[string]*fetchRequest, taskQueue *prque.Prque, timeoutMeter metrics.Meter) map[string]int {
		// Iterate over the expired requests and return each to the queue
		expiries := make(map[string]int)
		for id, request := range pendPool {
			if time.Since(request.Time) > timeout {
				// Update the metrics with the timeout
				timeoutMeter.Mark(1)
	
				// Return any non satisfied requests to the pool
				if request.From > 0 {
					taskQueue.Push(request.From, -float32(request.From))
				}
				for hash, index := range request.Hashes {
					taskQueue.Push(hash, float32(index))
				}
				for _, header := range request.Headers {
					taskQueue.Push(header, -float32(header.Number.Uint64()))
				}
				// Add the peer to the expiry report along the the number of failed requests
				expirations := len(request.Hashes)
				if expirations < len(request.Headers) {
					expirations = len(request.Headers)
				}
				expiries[id] = expirations
			}
		}
		// Remove the expired requests from the pending pool
		for id := range expiries {
			delete(pendPool, id)
		}
		return expiries
	}


### CancelXXX
Cancle函数取消已经分配的任务， 把任务重新加入到任务池。

	// CancelBodies aborts a body fetch request, returning all pending headers to the
	// task queue.
	func (q *queue) CancelBodies(request *fetchRequest) {
		q.cancel(request, q.blockTaskQueue, q.blockPendPool)
	}

	// Cancel aborts a fetch request, returning all pending hashes to the task queue.
	func (q *queue) cancel(request *fetchRequest, taskQueue *prque.Prque, pendPool map[string]*fetchRequest) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		if request.From > 0 {
			taskQueue.Push(request.From, -float32(request.From))
		}
		for hash, index := range request.Hashes {
			taskQueue.Push(hash, float32(index))
		}
		for _, header := range request.Headers {
			taskQueue.Push(header, -float32(header.Number.Uint64()))
		}
		delete(pendPool, request.Peer.id)
	}

## ScheduleSkeleton
Schedule方法传入的是已经fetch好的header。Schedule(headers []*types.Header, from uint64)。而ScheduleSkeleton函数的参数是一个骨架， 然后请求对骨架进行填充。所谓的骨架是指我首先每隔192个区块请求一个区块头，然后把返回的header传入ScheduleSkeleton。 在Schedule函数中只需要queue调度区块体和回执的下载，而在ScheduleSkeleton函数中，还需要调度那些缺失的区块头的下载。

	// ScheduleSkeleton adds a batch of header retrieval tasks to the queue to fill
	// up an already retrieved header skeleton.
	func (q *queue) ScheduleSkeleton(from uint64, skeleton []*types.Header) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// No skeleton retrieval can be in progress, fail hard if so (huge implementation bug)
		if q.headerResults != nil {
			panic("skeleton assembly already in progress")
		}
		// Shedule all the header retrieval tasks for the skeleton assembly
		// 因为这个方法在skeleton为false的时候不会调用。 所以一些初始化工作放在这里执行。
		q.headerTaskPool = make(map[uint64]*types.Header)
		q.headerTaskQueue = prque.New()
		q.headerPeerMiss = make(map[string]map[uint64]struct{}) // Reset availability to correct invalid chains
		q.headerResults = make([]*types.Header, len(skeleton)*MaxHeaderFetch)
		q.headerProced = 0
		q.headerOffset = from
		q.headerContCh = make(chan bool, 1)
	
		for i, header := range skeleton {
			index := from + uint64(i*MaxHeaderFetch)
			// 每隔MaxHeaderFetch这么远有一个header
			q.headerTaskPool[index] = header
			q.headerTaskQueue.Push(index, -float32(index))
		}
	}

### ReserveHeaders
这个方法只skeleton的模式下才会被调用。 用来给peer保留fetch 区块头的任务。
	
	// ReserveHeaders reserves a set of headers for the given peer, skipping any
	// previously failed batches.
	func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// Short circuit if the peer's already downloading something (sanity check to
		// not corrupt state)
		if _, ok := q.headerPendPool[p.id]; ok {
			return nil
		}
		// Retrieve a batch of hashes, skipping previously failed ones
		// 从队列中获取一个，跳过之前失败过的节点。
		send, skip := uint64(0), []uint64{}
		for send == 0 && !q.headerTaskQueue.Empty() {
			from, _ := q.headerTaskQueue.Pop()
			if q.headerPeerMiss[p.id] != nil {
				if _, ok := q.headerPeerMiss[p.id][from.(uint64)]; ok {
					skip = append(skip, from.(uint64))
					continue
				}
			}
			send = from.(uint64)
		}
		// Merge all the skipped batches back
		for _, from := range skip {
			q.headerTaskQueue.Push(from, -float32(from))
		}
		// Assemble and return the block download request
		if send == 0 {
			return nil
		}
		request := &fetchRequest{
			Peer: p,
			From: send,
			Time: time.Now(),
		}
		q.headerPendPool[p.id] = request
		return request
	}


### DeliverHeaders

	
	// DeliverHeaders injects a header retrieval response into the header results
	// cache. This method either accepts all headers it received, or none of them
	// if they do not map correctly to the skeleton.
	// 这个方法对于所有的区块头，要么全部接收，要么全部拒绝(如果不能映射到一个skeleton上面)
	// If the headers are accepted, the method makes an attempt to deliver the set
	// of ready headers to the processor to keep the pipeline full. However it will
	// not block to prevent stalling other pending deliveries.
	// 如果区块头被接收，这个方法会试图把他们投递到headerProcCh管道上面。 不过这个方法不会阻塞式的投递。而是尝试投递，如果不能投递就返回。
	func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// Short circuit if the data was never requested
		request := q.headerPendPool[id]
		if request == nil {
			return 0, errNoFetchesPending
		}
		headerReqTimer.UpdateSince(request.Time)
		delete(q.headerPendPool, id)
	
		// Ensure headers can be mapped onto the skeleton chain
		target := q.headerTaskPool[request.From].Hash()
	
		accepted := len(headers) == MaxHeaderFetch
		if accepted { //首先长度需要匹配， 然后检查区块号和最后一块区块的Hash值是否能够对应上。
			if headers[0].Number.Uint64() != request.From {
				log.Trace("First header broke chain ordering", "peer", id, "number", headers[0].Number, "hash", headers[0].Hash(), request.From)
				accepted = false
			} else if headers[len(headers)-1].Hash() != target {
				log.Trace("Last header broke skeleton structure ", "peer", id, "number", headers[len(headers)-1].Number, "hash", headers[len(headers)-1].Hash(), "expected", target)
				accepted = false
			}
		}
		if accepted {// 依次检查每一块区块的区块号， 以及链接是否正确。
			for i, header := range headers[1:] {
				hash := header.Hash()
				if want := request.From + 1 + uint64(i); header.Number.Uint64() != want {
					log.Warn("Header broke chain ordering", "peer", id, "number", header.Number, "hash", hash, "expected", want)
					accepted = false
					break
				}
				if headers[i].Hash() != header.ParentHash {
					log.Warn("Header broke chain ancestry", "peer", id, "number", header.Number, "hash", hash)
					accepted = false
					break
				}
			}
		}
		// If the batch of headers wasn't accepted, mark as unavailable
		if !accepted { // 如果不被接收，那么标记这个peer在这个任务上的失败。下次请求就不会投递给这个peer
			log.Trace("Skeleton filling not accepted", "peer", id, "from", request.From)
	
			miss := q.headerPeerMiss[id]
			if miss == nil {
				q.headerPeerMiss[id] = make(map[uint64]struct{})
				miss = q.headerPeerMiss[id]
			}
			miss[request.From] = struct{}{}
	
			q.headerTaskQueue.Push(request.From, -float32(request.From))
			return 0, errors.New("delivery not accepted")
		}
		// Clean up a successful fetch and try to deliver any sub-results
		copy(q.headerResults[request.From-q.headerOffset:], headers)
		delete(q.headerTaskPool, request.From)
	
		ready := 0
		for q.headerProced+ready < len(q.headerResults) && q.headerResults[q.headerProced+ready] != nil {//计算这次到来的header可以让headerResults有多少数据可以投递了。
			ready += MaxHeaderFetch
		}
		if ready > 0 {
			// Headers are ready for delivery, gather them and push forward (non blocking)
			process := make([]*types.Header, ready)
			copy(process, q.headerResults[q.headerProced:q.headerProced+ready])
			// 尝试投递
			select {
			case headerProcCh <- process:
				log.Trace("Pre-scheduled new headers", "peer", id, "count", len(process), "from", process[0].Number)
				q.headerProced += len(process)
			default:
			}
		}
		// Check for termination and return
		if len(q.headerTaskPool) == 0 {
			// 这个通道比较重要， 如果这个通道接收到数据，说明所有的header任务已经完成。
			q.headerContCh <- false
		}
		return len(headers), nil
	}

RetrieveHeaders，ScheduleSkeleton函数在上次调度还没有做完的情况下是不会调用的。 所以上次调用完成之后，会使用这个方法来获取结果，重置状态。
	
	// RetrieveHeaders retrieves the header chain assemble based on the scheduled
	// skeleton.
	func (q *queue) RetrieveHeaders() ([]*types.Header, int) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		headers, proced := q.headerResults, q.headerProced
		q.headerResults, q.headerProced = nil, 0
	
		return headers, proced
	}