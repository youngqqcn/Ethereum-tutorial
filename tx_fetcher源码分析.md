
# tx_fetcher 源码分析

暂时不研究细节, 大致理解tx_fetcher的功能和流程即可, 具体的细节后面再研究


```go
// TxFetcher负责根据公告检索新交易。
//
// 提取程序分为三个阶段：
//  -新发现的交易将移入等待列表。
//  -经过约500毫秒后，等待清单中尚未进行的交易整个广播都移到了排队区。
//  -当连接的对等方没有进行中的检索请求时，任何排队的交易（并由对等方宣布）分配给
//   查看并进入获取状态，直到实现或失败为止。
//
// 提取程序的不变量为：
// -每个跟踪的交易（哈希）只能存在于三个阶段。这样可以确保提取程序的操作类似于有限
//  设置自动机状态，并且会发生数据泄漏。
// -每个宣布交易的对等方都可以按计划进行检索，但是只有一个并发。这样可以确保我们可以立即知道是什么
//  缺少回复，然后重新安排时间。
//
// TxFetcher is responsible for retrieving new transaction based on announcements.
//
// The fetcher operates in 3 stages:
//   - Transactions that are newly discovered are moved into a wait list.
//   - After ~500ms passes, transactions from the wait list that have not been
//     broadcast to us in whole are moved into a queueing area.
//   - When a connected peer doesn't have in-flight retrieval requests, any
//     transaction queued up (and announced by the peer) are allocated to the
//     peer and moved into a fetching status until it's fulfilled or fails.
//
// The invariants of the fetcher are:
//   - Each tracked transaction (hash) must only be present in one of the
//     three stages. This ensures that the fetcher operates akin to a finite
//     state automata and there's do data leak.
//   - Each peer that announced transactions may be scheduled retrievals, but
//     only ever one concurrently. This ensures we can immediately know what is
//     missing from a reply and reschedule it.
type TxFetcher struct {
	notify  chan *txAnnounce
	cleanup chan *txDelivery
	drop    chan *txDrop
	quit    chan struct{}

	underpriced mapset.Set // Transactions discarded as too cheap (don't re-fetch)

    //阶段1：等待新发现的交易的等待清单广播，而无需明确的请求/回复往返。
	// Stage 1: Waiting lists for newly discovered transactions that might be
	// broadcast without needing explicit request/reply round trips.
	waitlist  map[common.Hash]map[string]struct{} // Transactions waiting for an potential broadcast
	waittime  map[common.Hash]mclock.AbsTime      // Timestamps when transactions were added to the waitlist
	waitslots map[string]map[common.Hash]struct{} // Waiting announcement sgroupped by peer (DoS protection)

    // 阶段2：等待分配给某个对等方的事务队列直接获取。
	// Stage 2: Queue of transactions that waiting to be allocated to some peer
	// to be retrieved directly.
	announces map[string]map[common.Hash]struct{} // Set of announced transactions, grouped by origin peer
	announced map[common.Hash]map[string]struct{} // Set of download locations, grouped by transaction hash

    // 第3阶段：当前正在检索的交易集，其中一些可能是完成，并且重新安排了一些时间。
    // 请注意，此步骤与上一阶段共享`announces`以避免重复（需要进行DoS检查）。
	// Stage 3: Set of transactions currently being retrieved, some which may be
	// fulfilled and some rescheduled. Note, this step shares 'announces' from the
	// previous stage to avoid having to duplicate (need it for DoS checks).
	fetching   map[common.Hash]string              // Transaction set currently being retrieved
	requests   map[string]*txRequest               // In-flight transaction retrievals
	alternates map[common.Hash]map[string]struct{} // In-flight transaction alternate origins if retrieval fails

	// Callbacks
	hasTx    func(common.Hash) bool             // Retrieves a tx from the local txpool
	addTxs   func([]*types.Transaction) []error // Insert a batch of transactions into local txpool
	fetchTxs func(string, []common.Hash) error  // Retrieves a set of txs from a remote peer

	step  chan struct{} // Notification channel when the fetcher loop iterates
	clock mclock.Clock  // Time wrapper to simulate in tests
	rand  *mrand.Rand   // Randomizer to use in tests instead of map range loops (soft-random)
}


// 交易通知,网络上有一批新的交易可用
// txAnnounce is the notification of the availability of a batch
// of new transactions in the network.
type txAnnounce struct {
	origin string        // Identifier of the peer originating the notification
	hashes []common.Hash // Batch of transaction hashes being announced
}


// 交易请求,从对端节点检索交易
// txRequest represents an in-flight transaction retrieval request destined to
// a specific peers.
type txRequest struct {
	hashes []common.Hash            // Transactions having been requested
	stolen map[common.Hash]struct{} // Deliveries by someone else (don't re-request)
	time   mclock.AbsTime           // Timestamp of the request
}


// 交易已经加入交易池
// txDelivery is the notification that a batch of transactions have been added
// to the pool and should be untracked.
type txDelivery struct {
	origin string        // Identifier of the peer originating the notification
	hashes []common.Hash // Batch of transaction hashes having been delivered
	direct bool          // Whether this is a direct reply or a broadcast
}

// 交易被删除
// txDrop is the notiication that a peer has disconnected.
type txDrop struct {
	peer string
}

```




```go
// Notify announces the fetcher of the potential availability of a new batch of
// transactions in the network.
func (f *TxFetcher) Notify(peer string, hashes []common.Hash) error {
	// Keep track of all the announced transactions
	txAnnounceInMeter.Mark(int64(len(hashes)))

	// Skip any transaction announcements that we already know of, or that we've
	// previously marked as cheap and discarded. This check is of course racey,
	// because multiple concurrent notifies will still manage to pass it, but it's
	// still valuable to check here because it runs concurrent  to the internal
	// loop, so anything caught here is time saved internally.
	var (
		unknowns               = make([]common.Hash, 0, len(hashes))
		duplicate, underpriced int64
	)
	for _, hash := range hashes {
		switch {
		case f.hasTx(hash):
			duplicate++

		case f.underpriced.Contains(hash):
			underpriced++

		default:
			unknowns = append(unknowns, hash)
		}
	}
	txAnnounceKnownMeter.Mark(duplicate)
	txAnnounceUnderpricedMeter.Mark(underpriced)

	// If anything's left to announce, push it into the internal loop
	if len(unknowns) == 0 {
		return nil
	}
	announce := &txAnnounce{
		origin: peer,
		hashes: unknowns,
	}
	select {
	case f.notify <- announce:
		return nil
	case <-f.quit:
		return errTerminated
	}
}



//Enqueue将一批接收到的事务导入到事务池中
//和提取程序。交易广播和直接请求回复。差异很重要，因此提取程序可以
// 尽快重新处理丢失的交易。
// Enqueue imports a batch of received transaction into the transaction pool
// and the fetcher. This method may be called by both transaction broadcasts and
// direct request replies. The differentiation is important so the fetcher can
// re-shedule missing transactions as soon as possible.
func (f *TxFetcher) Enqueue(peer string, txs []*types.Transaction, direct bool) error {
	// Keep track of all the propagated transactions
	if direct {
		txReplyInMeter.Mark(int64(len(txs)))
	} else {
		txBroadcastInMeter.Mark(int64(len(txs)))
	}
	// Push all the transactions into the pool, tracking underpriced ones to avoid
	// re-requesting them and dropping the peer in case of malicious transfers.
	var (
		added       = make([]common.Hash, 0, len(txs))
		duplicate   int64
		underpriced int64
		otherreject int64
	)
	errs := f.addTxs(txs)
	for i, err := range errs {
		if err != nil {
			// Track the transaction hash if the price is too low for us.
			// Avoid re-request this transaction when we receive another
			// announcement.
			if err == core.ErrUnderpriced || err == core.ErrReplaceUnderpriced {
				for f.underpriced.Cardinality() >= maxTxUnderpricedSetSize {
					f.underpriced.Pop()
				}
				f.underpriced.Add(txs[i].Hash())
			}
			// Track a few interesting failure types
			switch err {
			case nil: // Noop, but need to handle to not count these

			case core.ErrAlreadyKnown:
				duplicate++

			case core.ErrUnderpriced, core.ErrReplaceUnderpriced:
				underpriced++

			default:
				otherreject++
			}
		}
		added = append(added, txs[i].Hash())
	}
	if direct {
		txReplyKnownMeter.Mark(duplicate)
		txReplyUnderpricedMeter.Mark(underpriced)
		txReplyOtherRejectMeter.Mark(otherreject)
	} else {
		txBroadcastKnownMeter.Mark(duplicate)
		txBroadcastUnderpricedMeter.Mark(underpriced)
		txBroadcastOtherRejectMeter.Mark(otherreject)
	}

	select {
	case f.cleanup <- &txDelivery{origin: peer, hashes: added, direct: direct}:
		return nil
	case <-f.quit:
		return errTerminated
	}
}


// 删除对端
// Drop should be called when a peer disconnects. It cleans up all the internal
// data structures of the given node.
func (f *TxFetcher) Drop(peer string) error {
	select {
	case f.drop <- &txDrop{peer: peer}:
		return nil
	case <-f.quit:
		return errTerminated
	}
}


// rescheduleTimeout iterates over all the transactions currently in flight and
// schedules a cleanup run when the first would trigger.
//
// The method has a granularity of 'gatherSlack', since there's not much point in
// spinning over all the transactions just to maybe find one that should trigger
// a few ms earlier.
//
// This method is a bit "flaky" "by design". In theory the timeout timer only ever
// should be rescheduled if some request is pending. In practice, a timeout will
// cause the timer to be rescheduled every 5 secs (until the peer comes through or
// disconnects). This is a limitation of the fetcher code because we don't trac
// pending requests and timed out requests separatey. Without double tracking, if
// we simply didn't reschedule the timer on all-timeout then the timer would never
// be set again since len(request) > 0 => something's running.
func (f *TxFetcher) rescheduleTimeout(timer *mclock.Timer, trigger chan struct{}) {
	if *timer != nil {
		(*timer).Stop()
	}
	now := f.clock.Now()

	earliest := now
	for _, req := range f.requests {
		// If this request already timed out, skip it altogether
		if req.hashes == nil {
			continue
		}
		if earliest > req.time {
			earliest = req.time
			if txFetchTimeout-time.Duration(now-earliest) < gatherSlack {
				break
			}
		}
	}
	*timer = f.clock.AfterFunc(txFetchTimeout-time.Duration(now-earliest), func() {
		trigger <- struct{}{}
	})
}
```