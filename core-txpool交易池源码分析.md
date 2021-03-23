# tx pool
txpool主要用来存放当前提交的等待写入区块的交易，有远端和本地的。

txpool里面的交易分为两种，
1. 提交但是还不能执行的，放在queue里面等待能够执行(比如说nonce太高)。
2. 等待执行的，放在pending里面等待执行。

从txpool的测试案例来看，txpool主要功能有下面几点。

1. 交易验证的功能，包括余额不足，Gas不足，Nonce太低, value值是合法的，不能为负数
2. 能够缓存Nonce比当前本地账号状态高的交易。 存放在queue字段。 如果是能够执行的交易存放在pending字段
3. 相同用户的相同Nonce的交易只会保留一个GasPrice最大的那个。 其他的插入不成功
4. 如果账号没有钱了，那么queue和pending中对应账号的交易会被删除
5. 如果账号的余额小于一些交易的额度，那么对应的交易会被删除，同时有效的交易会从pending移动到queue里面。防止被广播
6. txPool支持一些限制PriceLimit(remove的最低GasPrice限制)，PriceBump(替换相同Nonce的交易的价格的百分比) AccountSlots(每个账户的pending的槽位的最小值) GlobalSlots(全局pending队列的最大值)AccountQueue(每个账户的queueing的槽位的最小值) GlobalQueue(全局queueing的最大值) Lifetime(在queue队列的最长等待时间)
7. 有限的资源情况下按照GasPrice的优先级进行替换
8. 本地的交易会使用journal的功能存放在磁盘上，重启之后会重新导入。 远程的交易不会




数据结构

```go
//TxPool包含所有当前已知的事务。交易次数
//从网络接收或提交池时进入池
//本地。当它们包含在区块链中时，它们退出池。
//
//池分隔可处理的交易（可将其应用于
//当前状态）和将来的交易。交易在那些之间移动
//随着时间的流逝，两个状态会被接收和处理。
// TxPool contains all currently known transactions. Transactions
// enter the pool when they are received from the network or submitted
// locally. They exit the pool when they are included in the blockchain.
//
// The pool separates processable transactions (which can be applied to the
// current state) and future transactions. Transactions move between those
// two states over time as they are received and processed.
type TxPool struct {
	config      TxPoolConfig
	chainconfig *params.ChainConfig
	chain       blockChain
	gasPrice    *big.Int
	txFeed      event.Feed
	scope       event.SubscriptionScope
	signer      types.Signer
	mu          sync.RWMutex

	istanbul bool // Fork indicator whether we are in the istanbul stage.

	currentState  *state.StateDB // Current state in the blockchain head
	pendingNonces *txNoncer      // Pending state tracking virtual nonces
	currentMaxGas uint64         // Current gas limit for transaction caps

	locals  *accountSet // Set of local transaction to exempt from eviction rules
	journal *txJournal  // Journal of local transaction to back up to disk

    // pending 和  queue是txpool的两个最重要的字段
	pending map[common.Address]*txList   // All currently processable transactions
	queue   map[common.Address]*txList   // Queued but non-processable transactions
    
    beats   map[common.Address]time.Time // Last heartbeat from each known account
	all     *txLookup                    // All transactions to allow lookups
	priced  *txPricedList                // All transactions sorted by price

	chainHeadCh     chan ChainHeadEvent
	chainHeadSub    event.Subscription
	reqResetCh      chan *txpoolResetRequest
	reqPromoteCh    chan *accountSet
	queueTxEventCh  chan *types.Transaction
	reorgDoneCh     chan chan struct{}
	reorgShutdownCh chan struct{}  // requests shutdown of scheduleReorgLoop
	wg              sync.WaitGroup // tracks loop, scheduleReorgLoop
}
```



|字段|	描述|
|---|---|
|config	|TxPoolConfig 类型，包含了交易池的配置信息，如 PriceLimit，移除交易的最低 GasPrice 限制；PriceBump，替换相同 Nonce 的交易的价格的百分比；AccountSlots，每个账户 pending 的槽位的最小值；GlobalSlots，全局 pending 队列的最大值；AccountQueue，每个账户的 queueing 的槽位的最小值；GlobalQueue，全局 queueing 的最大值；Lifetime，在队列的最长等待时间|
|chainconfig|	区块链的配置|
|gasPrice	|最低的 GasPrice 限制|
|txFeed	|可以通过 txFeed 来订阅 TxPool 的消息|
|chainHeadCh	|可以通过 chainHeadCh 订阅区块头的消息|
|signer	|封装了事务签名处理|


初始化pool
在 NewTxPool 方法里，如果本地可以发起交易，并且配置的 Journal 目录不为空，那么从指定的目录加载交易日志。NewTxPool 方法的最后会用一个 goroutine 调用

```go
// NewTxPool creates a new transaction pool to gather, sort and filter inbound
// transactions from the network.
func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
	// Sanitize the input to ensure no vulnerable gas prices are set
	config = (&config).sanitize()

	// Create the transaction pool with its initial settings
	pool := &TxPool{
		config:          config,
		chainconfig:     chainconfig,
		chain:           chain,
		signer:          types.NewEIP155Signer(chainconfig.ChainID),
		pending:         make(map[common.Address]*txList),
		queue:           make(map[common.Address]*txList),
		beats:           make(map[common.Address]time.Time),
		all:             newTxLookup(),
		chainHeadCh:     make(chan ChainHeadEvent, chainHeadChanSize),
		reqResetCh:      make(chan *txpoolResetRequest),
		reqPromoteCh:    make(chan *accountSet),
		queueTxEventCh:  make(chan *types.Transaction),
		reorgDoneCh:     make(chan chan struct{}),
		reorgShutdownCh: make(chan struct{}),
		gasPrice:        new(big.Int).SetUint64(config.PriceLimit),
	}
	pool.locals = newAccountSet(pool.signer)
	for _, addr := range config.Locals {
		log.Info("Setting new local account", "address", addr)
		pool.locals.add(addr)
	}
	pool.priced = newTxPricedList(pool.all)

    // 重置
	pool.reset(nil, chain.CurrentBlock().Header())

    // 启动一个 reorg循环, 为了它可以处理 从日志加载交易时发来的 reorg请求
	// Start the reorg loop early so it can handle requests generated during journal loading.
	pool.wg.Add(1)
	go pool.scheduleReorgLoop()

    // 从本地磁盘日志中加载交易
	// If local transactions and journaling is enabled, load from disk
	if !config.NoLocals && config.Journal != "" {
		pool.journal = newTxJournal(config.Journal)

		if err := pool.journal.load(pool.AddLocals); err != nil {
			log.Warn("Failed to load transaction journal", "err", err)
		}
		if err := pool.journal.rotate(pool.local()); err != nil {
			log.Warn("Failed to rotate transaction journal", "err", err)
		}
	}

    // 订阅事件
	// Subscribe events from blockchain and start the main event loop.
	pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)
	pool.wg.Add(1)
	go pool.loop()

	return pool
}

```


reset方法检索区块链的当前状态并且确保事务池的内容关于当前的区块链状态是有效的。主要功能包括：

1. 因为更换了区块头，所以原有的区块中有一些交易因为区块头的更换而作废，这部分交易需要重新加入到txPool里面等待插入新的区块
2. 生成新的currentState和pendingState
3. 因为状态的改变。将pending中的部分交易移到queue里面
4. 因为状态的改变，将queue里面的交易移入到pending里面。

reset代码

```go
//reset检索区块链的当前状态并确保内容
//交易池的链状态有效。
// reset retrieves the current state of the blockchain and ensures the content
// of the transaction pool is valid with regard to the chain state.
func (pool *TxPool) reset(oldHead, newHead *types.Header) {
    // If we're reorging an old state, reinject all dropped transactions
    var reinject types.Transactions

    if oldHead != nil && oldHead.Hash() != newHead.ParentHash {
        // If the reorg is too deep, avoid doing it (will happen during fast sync)
        oldNum := oldHead.Number.Uint64()
        newNum := newHead.Number.Uint64()

        if depth := uint64(math.Abs(float64(oldNum) - float64(newNum))); depth > 64 { //如果老的头和新的头差距太远, 那么取消重建
            log.Warn("Skipping deep transaction reorg", "depth", depth)
        } else {
            // Reorg seems shallow enough to pull in all transactions into memory
            var discarded, included types.Transactions

            var (
                rem = pool.chain.GetBlock(oldHead.Hash(), oldHead.Number.Uint64())
                add = pool.chain.GetBlock(newHead.Hash(), newHead.Number.Uint64())
            )
            // 如果老的高度大于新的.那么需要把多的全部删除.
            for rem.NumberU64() > add.NumberU64() {
                discarded = append(discarded, rem.Transactions()...)
                if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
                    log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
                    return
                }
            }
            // 如果新的高度大于老的, 那么需要增加.
            for add.NumberU64() > rem.NumberU64() {
                included = append(included, add.Transactions()...)
                if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
                    log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
                    return
                }
            }
            // 高度相同了.如果hash不同,那么需要往后找,一直找到他们相同hash根的节点.
            for rem.Hash() != add.Hash() {
                discarded = append(discarded, rem.Transactions()...)
                if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
                    log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
                    return
                }
                included = append(included, add.Transactions()...)
                if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
                    log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
                    return
                }
            }
            // 找出所有存在discard里面,但是不在included里面的值.
            // 需要等下把这些交易重新插入到pool里面。
            reinject = types.TxDifference(discarded, included)
        }
    }
    // Initialize the internal state to the current head
    if newHead == nil {
        newHead = pool.chain.CurrentBlock().Header() // Special case during testing
    }
    statedb, err := pool.chain.StateAt(newHead.Root)
    if err != nil {
        log.Error("Failed to reset txpool state", "err", err)
        return
    }
    pool.currentState = statedb
    pool.pendingState = state.ManageState(statedb)
    pool.currentMaxGas = newHead.GasLimit

    // Inject any transactions discarded due to reorgs
    log.Debug("Reinjecting stale transactions", "count", len(reinject))
    pool.addTxsLocked(reinject, false)

    // validate the pool of pending transactions, this will remove
    // any transactions that have been included in the block or
    // have been invalidated because of another transaction (e.g.
    // higher gas price)
    // 验证pending transaction池里面的交易， 会移除所有已经存在区块链里面的交易，或者是因为其他交易导致不可用的交易(比如有一个更高的gasPrice)
    // demote 降级 将pending中的一些交易降级到queue里面。
    pool.demoteUnexecutables()

    // Update all accounts to the latest known pending nonce
    // 根据pending队列的nonce更新所有账号的nonce
    for addr, list := range pool.pending {
        txs := list.Flatten() // Heavy but will be cached and is needed by the miner anyway
        pool.pendingState.SetNonce(addr, txs[len(txs)-1].Nonce()+1)
    }
    // Check the queue and move transactions over to the pending if possible
    // or remove those that have become invalid
    // 检查队列并尽可能地将事务移到pending，或删除那些已经失效的事务
    // promote 升级 
    pool.promoteExecutables(nil)
}
```


addTx 

addTx 将交易放入交易池中，pool.add(tx, local) 会返回一个 bool 类型，如果为 true，则表明这笔交易合法并且交易之前不存在于交易池，这时候调用 promoteExecutables，可以将可处理的交易变成待处理。所以说，交易池的交易大致分为两种，一种是提交了但还不能执行的，放在 queue 里等待能够被执行（比如 nonce 太高），还有就是等待执行的，放在 pending 里面等待执行。

```go
// addTx enqueues a single transaction into the pool if it is valid.
func (pool *TxPool) addTx(tx *types.Transaction, local bool) error {
    pool.mu.Lock()
    defer pool.mu.Unlock()

    // Try to inject the transaction and update any state
    replace, err := pool.add(tx, local)
    if err != nil {
        return err
    }
    // If we added a new transaction, run promotion checks and return
    if !replace {
        from, _ := types.Sender(pool.signer, tx) // already validated
        pool.promoteExecutables([]common.Address{from})
    }
    return nil
}
```


addTxsLocked


```go
// addTxsLocked attempts to queue a batch of transactions if they are valid,
// whilst assuming the transaction pool lock is already held.
// addTxsLocked尝试把有效的交易放入queue队列，调用这个函数的时候假设已经获取到锁
func (pool *TxPool) addTxsLocked(txs []*types.Transaction, local bool) error {
    // Add the batch of transaction, tracking the accepted ones
    dirty := make(map[common.Address]struct{})
    for _, tx := range txs {
        if replace, err := pool.add(tx, local); err == nil {
            if !replace { // replace 是替换的意思， 如果不是替换，那么就说明状态有更新，有可以下一步处理的可能。
                from, _ := types.Sender(pool.signer, tx) // already validated
                dirty[from] = struct{}{}
            }
        }
    }
    // Only reprocess the internal state if something was actually added
    if len(dirty) > 0 {
        addrs := make([]common.Address, 0, len(dirty))
        for addr, _ := range dirty {
            addrs = append(addrs, addr)
        }	
        // 传入了被修改的地址，
        pool.promoteExecutables(addrs)
    }
    return nil
}
```

demoteUnexecutables 从pending删除无效的或者是已经处理过的交易，其他的不可执行的交易会被移动到future queue中。


```go
// demoteUnexecutables removes invalid and processed transactions from the pools
// executable/pending queue and any subsequent transactions that become unexecutable
// are moved back into the future queue.
func (pool *TxPool) demoteUnexecutables() {
    // Iterate over all accounts and demote any non-executable transactions
    for addr, list := range pool.pending {
        nonce := pool.currentState.GetNonce(addr)

        // Drop all transactions that are deemed too old (low nonce)
        // 删除所有小于当前地址的nonce的交易，并从pool.all删除。
        for _, tx := range list.Forward(nonce) {
            hash := tx.Hash()
            log.Trace("Removed old pending transaction", "hash", hash)
            delete(pool.all, hash)
            pool.priced.Removed()
        }
        // Drop all transactions that are too costly (low balance or out of gas), and queue any invalids back for later
        // 删除所有的太昂贵的交易。 用户的balance可能不够用。或者是out of gas
        drops, invalids := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
        for _, tx := range drops {
            hash := tx.Hash()
            log.Trace("Removed unpayable pending transaction", "hash", hash)
            delete(pool.all, hash)
            pool.priced.Removed()
            pendingNofundsCounter.Inc(1)
        }
        for _, tx := range invalids {
            hash := tx.Hash()
            log.Trace("Demoting pending transaction", "hash", hash)
            pool.enqueueTx(hash, tx)
        }
        // If there's a gap in front, warn (should never happen) and postpone all transactions
        // 如果存在一个空洞(nonce空洞)， 那么需要把所有的交易都放入future queue。
        // 这一步确实应该不可能发生，因为Filter已经把 invalids的都处理了。 应该不存在invalids的交易，也就是不存在空洞的。
        if list.Len() > 0 && list.txs.Get(nonce) == nil {
            for _, tx := range list.Cap(0) {
                hash := tx.Hash()
                log.Error("Demoting invalidated transaction", "hash", hash)
                pool.enqueueTx(hash, tx)
            }
        }
        // Delete the entire queue entry if it became empty.
        if list.Empty() { 
            delete(pool.pending, addr)
            delete(pool.beats, addr)
        }
    }
}
```

enqueueTx 把一个新的交易插入到future queue。 这个方法假设已经获取了池的锁。


```go
// enqueueTx inserts a new transaction into the non-executable transaction queue.
//
// Note, this method assumes the pool lock is held!
func (pool *TxPool) enqueueTx(hash common.Hash, tx *types.Transaction) (bool, error) {
    // Try to insert the transaction into the future queue
    from, _ := types.Sender(pool.signer, tx) // already validated
    if pool.queue[from] == nil {
        pool.queue[from] = newTxList(false)
    }
    inserted, old := pool.queue[from].Add(tx, pool.config.PriceBump)
    if !inserted {
        // An older transaction was better, discard this
        queuedDiscardCounter.Inc(1)
        return false, ErrReplaceUnderpriced
    }
    // Discard any previous transaction and mark this
    if old != nil {
        delete(pool.all, old.Hash())
        pool.priced.Removed()
        queuedReplaceCounter.Inc(1)
    }
    pool.all[hash] = tx
    pool.priced.Put(tx)
    return old != nil, nil
}
```


promoteExecutables方法把 已经变得可以执行的交易从future queue 插入到pending queue。通过这个处理过程，所有的无效的交易(nonce太低，余额不足)会被删除。

```go
// promoteExecutables moves transactions that have become processable from the
// future queue to the set of pending transactions. During this process, all
// invalidated transactions (low nonce, low balance) are deleted.
func (pool *TxPool) promoteExecutables(accounts []common.Address) []*types.Transaction {
	// Track the promoted transactions to broadcast them at once
	var promoted []*types.Transaction

	// Iterate over all accounts and promote any executable transactions
	for _, addr := range accounts {
		list := pool.queue[addr]
		if list == nil {
			continue // Just in case someone calls with a non existing account
		}

        // 删除nonce太小的交易
		// Drop all transactions that are deemed too old (low nonce)
		forwards := list.Forward(pool.currentState.GetNonce(addr))
		for _, tx := range forwards {
			hash := tx.Hash()
			pool.all.Remove(hash)
		}
		log.Trace("Removed old queued transactions", "count", len(forwards))


        // 删除钱不够的或者gas超出的交易
		// Drop all transactions that are too costly (low balance or out of gas)
		drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			pool.all.Remove(hash)
		}
		log.Trace("Removed unpayable queued transactions", "count", len(drops))
		queuedNofundsMeter.Mark(int64(len(drops)))

        // 收集所有可执行的交易, 并且提升它们
		// Gather all executable transactions and promote them
		readies := list.Ready(pool.pendingNonces.get(addr))
		for _, tx := range readies {
			hash := tx.Hash()
			if pool.promoteTx(addr, hash, tx) {
				promoted = append(promoted, tx)
			}
		}
		log.Trace("Promoted queued transactions", "count", len(promoted))
		queuedGauge.Dec(int64(len(readies)))

        // 删除超出限制的交易
		// Drop all transactions over the allowed limit
		var caps types.Transactions
		if !pool.locals.contains(addr) {
			caps = list.Cap(int(pool.config.AccountQueue))
			for _, tx := range caps {
				hash := tx.Hash()
				pool.all.Remove(hash)
				log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
			}
			queuedRateLimitMeter.Mark(int64(len(caps)))
		}

        // 将所以移除的交易标记为删除
		// Mark all the items dropped as removed
		pool.priced.Removed(len(forwards) + len(drops) + len(caps))
		queuedGauge.Dec(int64(len(forwards) + len(drops) + len(caps)))
		if pool.locals.contains(addr) {
			localGauge.Dec(int64(len(forwards) + len(drops) + len(caps)))
		}


        // 删除
		// Delete the entire queue entry if it became empty.
		if list.Empty() {
			delete(pool.queue, addr)
			delete(pool.beats, addr)
		}
	}
	return promoted
}

```


该方法首先遍历所有当前账户交易，通过 list.Forward 方法迭代当前账户，检查 nonce，如果 nonce 太低，删除该交易。接着通过 list.Filter 方法检查余额不足或 gas 不足的交易，删除不满足的交易。这时得到的所有可执行的交易，通过调用 promoteTx 加入到 pending 里，接着移除超过了限制的交易。对于已经加入到 promoted 的交易，调用 pool.txFeed.Send 将消息发给订阅者，在 eth 协议里，这个交易会被广播出去。

经过上面的处理，pending 的数量可能会超过系统配置的数量，这时需要进行一些处理，移除一些交易。

pending 处理完后，继续处理 future queue，队列里的数量也可能会超过 GlobalQueue 里的数量，根据心跳时间排列所有交易，移除最旧的交易。


promoteTx把某个交易加入到pending 队列. 这个方法假设已经获取到了锁.


```go
// promoteTx adds a transaction to the pending (processable) list of transactions
// and returns whether it was inserted or an older was better.
//
// Note, this method assumes the pool lock is held!
func (pool *TxPool) promoteTx(addr common.Address, hash common.Hash, tx *types.Transaction) bool {
	// Try to insert the transaction into the pending queue
	if pool.pending[addr] == nil {
		pool.pending[addr] = newTxList(true)
	}
	list := pool.pending[addr]

	inserted, old := list.Add(tx, pool.config.PriceBump)
	if !inserted {
		// An older transaction was better, discard this
		pool.all.Remove(hash)
		pool.priced.Removed(1)
		pendingDiscardMeter.Mark(1)
		return false
	}
	// Otherwise discard any previous transaction and mark this
	if old != nil {
		pool.all.Remove(old.Hash())
		pool.priced.Removed(1)
		pendingReplaceMeter.Mark(1)
	} else {
		// Nothing was replaced, bump the pending counter
		pendingGauge.Inc(1)
	}
	// Set the potentially new pending nonce and notify any subsystems of the new tx
	pool.pendingNonces.set(addr, tx.Nonce()+1)

	// Successful promotion, bump the heartbeat
	pool.beats[addr] = time.Now()
	return true
}
```


removeTx，删除某个交易， 并把所有后续的交易移动到future queue

```go

// removeTx removes a single transaction from the queue, moving all subsequent
// transactions back to the future queue.
func (pool *TxPool) removeTx(hash common.Hash, outofbound bool) {
	// Fetch the transaction we wish to delete
	tx := pool.all.Get(hash)
	if tx == nil {
		return
	}
	addr, _ := types.Sender(pool.signer, tx) // already validated during insertion

    // 从已知的交易列表中删除
	// Remove it from the list of known transactions
	pool.all.Remove(hash)
	if outofbound {
		pool.priced.Removed(1)
	}
	if pool.locals.contains(addr) {
		localGauge.Dec(1)
	}

    // 从 pending队列中删除删除交易, 并且重置账户的nonce
	// Remove the transaction from the pending lists and reset the account nonce
	if pending := pool.pending[addr]; pending != nil {
		if removed, invalids := pending.Remove(tx); removed {
			// If no more pending transactions are left, remove the list
			if pending.Empty() {
				delete(pool.pending, addr)
			}
			// Postpone any invalidated transactions
			for _, tx := range invalids {
				// Internal shuffle shouldn't touch the lookup set.
				pool.enqueueTx(tx.Hash(), tx, false, false)
			}
			// Update the account nonce if needed
			pool.pendingNonces.setIfLower(addr, tx.Nonce())
			// Reduce the pending counter
			pendingGauge.Dec(int64(1 + len(invalids)))
			return
		}
	}

    // 交易在 future队列中
	// Transaction is in the future queue
	if future := pool.queue[addr]; future != nil {
		if removed, _ := future.Remove(tx); removed {
			// Reduce the queued counter
			queuedGauge.Dec(1)
		}
		if future.Empty() {
			delete(pool.queue, addr)
			delete(pool.beats, addr)
		}
	}
}

```


loop是txPool的一个goroutine.也是主要的事件循环.等待和响应外部区块链事件以及各种报告和交易驱逐事件。


```go
// loop is the transaction pool's main event loop, waiting for and reacting to
// outside blockchain events as well as for various reporting and transaction
// eviction events.
func (pool *TxPool) loop() {
	defer pool.wg.Done()

	var (
		prevPending, prevQueued, prevStales int
		// Start the stats reporting and transaction eviction tickers
		report  = time.NewTicker(statsReportInterval)
		evict   = time.NewTicker(evictionInterval)
		journal = time.NewTicker(pool.config.Rejournal)
		// Track the previous head headers for transaction reorgs
		head = pool.chain.CurrentBlock()
	)
	defer report.Stop()
	defer evict.Stop()
	defer journal.Stop()

	for {
		select {

        // 处理区块头事件
		// Handle ChainHeadEvent
		case ev := <-pool.chainHeadCh:
			if ev.Block != nil {
				pool.requestReset(head.Header(), ev.Block.Header())
				head = ev.Block
			}

        // 系统关闭
		// System shutdown.
		case <-pool.chainHeadSub.Err():
			close(pool.reorgShutdownCh)
			return

        // 状态定时报告
		// Handle stats reporting ticks
		case <-report.C:
			pool.mu.RLock()
			pending, queued := pool.stats()
			stales := pool.priced.stales
			pool.mu.RUnlock()

			if pending != prevPending || queued != prevQueued || stales != prevStales {
				log.Debug("Transaction pool status report", "executable", pending, "queued", queued, "stales", stales)
				prevPending, prevQueued, prevStales = pending, queued, stales
			}

        // 处理不活跃交易
		// Handle inactive account transaction eviction
		case <-evict.C:
			pool.mu.Lock()
			for addr := range pool.queue {
				// Skip local transactions from the eviction mechanism
				if pool.locals.contains(addr) {
					continue
				}
				// Any non-locals old enough should be removed
				if time.Since(pool.beats[addr]) > pool.config.Lifetime {
					list := pool.queue[addr].Flatten()
					for _, tx := range list {
						pool.removeTx(tx.Hash(), true)
					}
					queuedEvictionMeter.Mark(int64(len(list)))
				}
			}
			pool.mu.Unlock()

        // 处理本地(日志)交易
		// Handle local transaction journal rotation
		case <-journal.C:
			if pool.journal != nil {
				pool.mu.Lock()
				if err := pool.journal.rotate(pool.local()); err != nil {
					log.Warn("Failed to rotate local tx journal", "err", err)
				}
				pool.mu.Unlock()
			}
		}
	}
}
```

add 方法, 验证交易并将其插入到future queue. 如果这个交易是替换了当前存在的某个交易,那么会返回之前的那个交易,这样外部就不用调用promote方法. 如果某个新增加的交易被标记为local, 那么它的发送账户会进入白名单,这个账户的关联的交易将不会因为价格的限制或者其他的一些限制被删除.


```go
//add验证交易并将其插入到非可执行队列中，以备后用
//等待升级和执行。如果交易是已经的交易的替代
//待处理或排队的交易，如果价格较高，它将覆盖前一个交易。
//
//如果新添加的交易标记为本地交易，则其发送帐户为
//列入白名单，以防止任何关联的事务从池中退出
//由于价格限制。
// add validates a transaction and inserts it into the non-executable queue for later
// pending promotion and execution. If the transaction is a replacement for an already
// pending or queued one, it overwrites the previous transaction if its price is higher.
//
// If a newly added transaction is marked as local, its sending account will be
// whitelisted, preventing any associated transaction from being dropped out of the pool
// due to pricing constraints.
func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error) {
	// If the transaction is already known, discard it
	hash := tx.Hash()
	if pool.all.Get(hash) != nil {
		log.Trace("Discarding already known transaction", "hash", hash)
		knownTxMeter.Mark(1)
		return false, ErrAlreadyKnown
	}
	// Make the local flag. If it's from local source or it's from the network but
	// the sender is marked as local previously, treat it as the local transaction.
	isLocal := local || pool.locals.containsTx(tx)

	// If the transaction fails basic validation, discard it
	if err := pool.validateTx(tx, isLocal); err != nil {
		log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
		invalidTxMeter.Mark(1)
		return false, err
	}
	// If the transaction pool is full, discard underpriced transactions
	if uint64(pool.all.Count()+numSlots(tx)) > pool.config.GlobalSlots+pool.config.GlobalQueue {
		// If the new transaction is underpriced, don't accept it
		if !isLocal && pool.priced.Underpriced(tx) {
			log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
			underpricedTxMeter.Mark(1)
			return false, ErrUnderpriced
		}
		// New transaction is better than our worse ones, make room for it.
		// If it's a local transaction, forcibly discard all available transactions.
		// Otherwise if we can't make enough room for new one, abort the operation.
		drop, success := pool.priced.Discard(pool.all.Slots()-int(pool.config.GlobalSlots+pool.config.GlobalQueue)+numSlots(tx), isLocal)

		// Special case, we still can't make the room for the new remote one.
		if !isLocal && !success {
			log.Trace("Discarding overflown transaction", "hash", hash)
			overflowedTxMeter.Mark(1)
			return false, ErrTxPoolOverflow
		}
		// Kick out the underpriced remote transactions.
		for _, tx := range drop {
			log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
			underpricedTxMeter.Mark(1)
			pool.removeTx(tx.Hash(), false)
		}
	}
	// Try to replace an existing transaction in the pending pool
	from, _ := types.Sender(pool.signer, tx) // already validated
	if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
		// Nonce already pending, check if required price bump is met
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted {
			pendingDiscardMeter.Mark(1)
			return false, ErrReplaceUnderpriced
		}
		// New transaction is better, replace old one
		if old != nil {
			pool.all.Remove(old.Hash())
			pool.priced.Removed(1)
			pendingReplaceMeter.Mark(1)
		}
		pool.all.Add(tx, isLocal)
		pool.priced.Put(tx, isLocal)
		pool.journalTx(from, tx)
		pool.queueTxEvent(tx)
		log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())

		// Successful promotion, bump the heartbeat
		pool.beats[from] = time.Now()
		return old != nil, nil
	}
	// New transaction isn't replacing a pending one, push into queue
	replaced, err = pool.enqueueTx(hash, tx, isLocal, true)
	if err != nil {
		return false, err
	}
	// Mark local addresses and journal local transactions
	if local && !pool.locals.contains(from) {
		log.Info("Setting new local account", "address", from)
		pool.locals.add(from)
		pool.priced.Removed(pool.all.RemoteToLocals(pool.locals)) // Migrate the remotes if it's marked as local first time.
	}
	if isLocal {
		localGauge.Inc(1)
	}
	pool.journalTx(from, tx)

	log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
	return replaced, nil
}

```

validateTx 使用一致性规则来检查一个交易是否有效,并采用本地节点的一些启发式的限制.

```go
// validateTx checks whether a transaction is valid according to the consensus
// rules and adheres to some heuristic limits of the local node (price and size).
func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
    // Heuristic limit, reject transactions over 32KB to prevent DOS attacks
    if tx.Size() > 32*1024 {
        return ErrOversizedData
    }
    // Transactions can't be negative. This may never happen using RLP decoded
    // transactions but may occur if you create a transaction using the RPC.
    if tx.Value().Sign() < 0 {
        return ErrNegativeValue
    }
    // Ensure the transaction doesn't exceed the current block limit gas.
    if pool.currentMaxGas.Cmp(tx.Gas()) < 0 {
        return ErrGasLimit
    }
    // Make sure the transaction is signed properly
    // 确保交易被正确签名.
    from, err := types.Sender(pool.signer, tx)
    if err != nil {
        return ErrInvalidSender
    }
    // Drop non-local transactions under our own minimal accepted gas price
    local = local || pool.locals.contains(from) // account may be local even if the transaction arrived from the network
    // 如果不是本地的交易,并且GasPrice低于我们的设置,那么也不会接收.
    if !local && pool.gasPrice.Cmp(tx.GasPrice()) > 0 {
        return ErrUnderpriced
    }
    // Ensure the transaction adheres to nonce ordering
    // 确保交易遵守了Nonce的顺序
    if pool.currentState.GetNonce(from) > tx.Nonce() {
        return ErrNonceTooLow
    }
    // Transactor should have enough funds to cover the costs
    // cost == V + GP * GL
    // 确保用户有足够的余额来支付.
    if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
        return ErrInsufficientFunds
    }
    intrGas := IntrinsicGas(tx.Data(), tx.To() == nil, pool.homestead)
    // 如果交易是一个合约创建或者调用. 那么看看是否有足够的 初始Gas.
    if tx.Gas().Cmp(intrGas) < 0 {
        return ErrIntrinsicGas
    }
    return nil
}

```


validateTx 有很多使用 if 语句的条件判断，大致会有如下判断：

- 拒绝大于 32kb 的交易，防止 DDoS 攻击
- 拒绝转账金额小于0的交易
- 拒绝 gas 超过交易池 gas 上限的交易
- 验证这笔交易的签名是否合法
- 如果交易不是来自本地的，并且 gas 小于当前交易池中的 gas，拒绝这笔交易
- 当前用户 nonce 如果大于这笔交易的 nonce，拒绝这笔交易
- 当前账户余额不足，拒绝这笔交易，queue 和 pending 对应账户的交易会被删除
- 拒绝当前交易固有花费小于交易池 gas 的交易
判断交易合法后，回到 add 方法，接着判断交易池的容量，如果交易池超过容量了，并且这笔交易的费用低于当前交易池中列表的最小值，拒绝这笔交易；如果这笔交易费用比当前交易池列表最小值高，那么从交易池中移除交易费用最低的交易，为这笔新交易腾出空间，也就是说按照 GasPrice 排出优先级。接着通过调用 Overlaps 通过检查这笔交易的 Nonce 值确认该用户是否已经存在这笔交易，如果已经存在，删除之前的交易，将该交易放入交易池，返回；如果不存在，调用 enqueueTx 将交易放入交易池，如果交易是本地发出的，将发送者保存在交易池的 local 中。注意到 add 方法最后会调用 pool.journalTx(from, tx)。

accountSet

accountSet 就是一个账号的集合和一个处理签名的对象.


```go
// accountSet is simply a set of addresses to check for existence, and a signer
// capable of deriving addresses from transactions.
type accountSet struct {
    accounts map[common.Address]struct{}
    signer   types.Signer
}

// newAccountSet creates a new address set with an associated signer for sender
// derivations.
func newAccountSet(signer types.Signer) *accountSet {
    return &accountSet{
        accounts: make(map[common.Address]struct{}),
        signer:   signer,
    }
}

// contains checks if a given address is contained within the set.
func (as *accountSet) contains(addr common.Address) bool {
    _, exist := as.accounts[addr]
    return exist
}

// containsTx checks if the sender of a given tx is within the set. If the sender
// cannot be derived, this method returns false.
// containsTx检查给定tx的发送者是否在集合内。 如果发件人无法被计算出，则此方法返回false。
func (as *accountSet) containsTx(tx *types.Transaction) bool {
    if addr, err := types.Sender(as.signer, tx); err == nil {
        return as.contains(addr)
    }
    return false
}

// add inserts a new address into the set to track.
func (as *accountSet) add(addr common.Address) {
    as.accounts[addr] = struct{}{}
}
```