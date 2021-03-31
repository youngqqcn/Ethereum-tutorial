
node中的服务的定义， eth其实就是实现了一个服务。

	type Service interface {
		// Protocols retrieves the P2P protocols the service wishes to start.
		Protocols() []p2p.Protocol
	
		// APIs retrieves the list of RPC descriptors the service provides
		APIs() []rpc.API
	
		// Start is called after all services have been constructed and the networking
		// layer was also initialized to spawn any goroutines required by the service.
		Start(server *p2p.Server) error
	
		// Stop terminates all goroutines belonging to the service, blocking until they
		// are all terminated.
		Stop() error
	}

go ethereum 的eth目录是以太坊服务的实现。 以太坊协议是通过node的Register方法注入的。


	// RegisterEthService adds an Ethereum client to the stack.
	func RegisterEthService(stack *node.Node, cfg *eth.Config) {
		var err error
		if cfg.SyncMode == downloader.LightSync {
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				return les.New(ctx, cfg)
			})
		} else {
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				fullNode, err := eth.New(ctx, cfg)
				if fullNode != nil && cfg.LightServ > 0 {
					ls, _ := les.NewLesServer(fullNode, cfg)
					fullNode.AddLesServer(ls)
				}
				return fullNode, err
			})
		}
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
	}

以太坊协议的数据结构


```go
// Ethereum 实现了以太坊全节点服务
// Ethereum implements the Ethereum full node service.
type Ethereum struct {
	config *Config

	// Handlers
	txPool             *core.TxPool         // 交易池,
	blockchain         *core.BlockChain     // 区块链功能,如区块验证,区块插入,区块查询等
	handler            *handler             // 处理
	ethDialCandidates  enode.Iterator
	snapDialCandidates enode.Iterator

	// DB interfaces
	chainDb ethdb.Database              // Block chain database  数据库

	eventMux       *event.TypeMux       
	engine         consensus.Engine     // 共识
	accountManager *accounts.Manager    // 账户管理

    // 布隆过滤器相关
	bloomRequests     chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests
	bloomIndexer      *core.ChainIndexer             // Bloom indexer operating during block imports
	closeBloomHandler chan struct{}

	APIBackend *EthAPIBackend    // 提供给RPC服务使用的API后端

	miner     *miner.Miner      // 挖矿
	gasPrice  *big.Int          // gas价格
	etherbase common.Address    // 矿工地址

	networkID     uint64
	netRPCService *ethapi.PublicNetAPI

	p2pServer *p2p.Server       // p2p服务

	lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
}

```


以太坊协议的创建New. 暂时先不涉及core的内容。 只是大概介绍一下。 core里面的内容后续会分析。

```go
	// New creates a new Ethereum object (including the
	// initialisation of the common Ethereum object)
	func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
		if config.SyncMode == downloader.LightSync {
			return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
		}
		if !config.SyncMode.IsValid() {
			return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
		}
		// 创建leveldb。 打开或者新建 chaindata目录
		chainDb, err := CreateDB(ctx, config, "chaindata")
		if err != nil {
			return nil, err
		}
		// 数据库格式升级
		stopDbUpgrade := upgradeDeduplicateData(chainDb)
		// 设置创世区块。 如果数据库里面已经有创世区块那么从数据库里面取出(私链)。或者是从代码里面获取默认值。
		chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
		if _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil && !ok {
			return nil, genesisErr
		}
		log.Info("Initialised chain configuration", "config", chainConfig)
	
		eth := &Ethereum{
			config:         config,
			chainDb:        chainDb,
			chainConfig:    chainConfig,
			eventMux:       ctx.EventMux,
			accountManager: ctx.AccountManager,
			engine:         CreateConsensusEngine(ctx, config, chainConfig, chainDb), // 一致性引擎。 这里我理解是Pow
			shutdownChan:   make(chan bool),
			stopDbUpgrade:  stopDbUpgrade,
			networkId:      config.NetworkId,  // 网络ID用来区别网路。 测试网络是0.主网是1
			gasPrice:       config.GasPrice,   // 可以通过配置 --gasprice 客户端接纳的交易的gasprice最小值。如果小于这个值那么会被节点丢弃。 
			etherbase:      config.Etherbase,  //挖矿的受益者
			bloomRequests:  make(chan chan *bloombits.Retrieval),  //bloom的请求
			bloomIndexer:   NewBloomIndexer(chainDb, params.BloomBitsBlocks),
		}
	
		log.Info("Initialising Ethereum protocol", "versions", ProtocolVersions, "network", config.NetworkId)
	
		if !config.SkipBcVersionCheck { // 检查数据库里面存储的BlockChainVersion和客户端的BlockChainVersion的版本是否一致
			bcVersion := core.GetBlockChainVersion(chainDb)
			if bcVersion != core.BlockChainVersion && bcVersion != 0 {
				return nil, fmt.Errorf("Blockchain DB version mismatch (%d / %d). Run geth upgradedb.\n", bcVersion, core.BlockChainVersion)
			}
			core.WriteBlockChainVersion(chainDb, core.BlockChainVersion)
		}
	
		vmConfig := vm.Config{EnablePreimageRecording: config.EnablePreimageRecording}
		// 使用数据库创建了区块链
		eth.blockchain, err = core.NewBlockChain(chainDb, eth.chainConfig, eth.engine, vmConfig)
		if err != nil {
			return nil, err
		}
		// Rewind the chain in case of an incompatible config upgrade.
		if compat, ok := genesisErr.(*params.ConfigCompatError); ok {
			log.Warn("Rewinding chain to upgrade configuration", "err", compat)
			eth.blockchain.SetHead(compat.RewindTo)
			core.WriteChainConfig(chainDb, genesisHash, chainConfig)
		}
		// bloomIndexer 暂时不知道是什么东西 这里面涉及得也不是很多。 暂时先不管了
		eth.bloomIndexer.Start(eth.blockchain.CurrentHeader(), eth.blockchain.SubscribeChainEvent)
	
		if config.TxPool.Journal != "" {
			config.TxPool.Journal = ctx.ResolvePath(config.TxPool.Journal)
		}
		// 创建交易池。 用来存储本地或者在网络上接收到的交易。
		eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
		// 创建协议管理器
		if eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb); err != nil {
			return nil, err
		}
		// 创建矿工
		eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine)
		eth.miner.SetExtra(makeExtraData(config.ExtraData))
		// ApiBackend 用于给RPC调用提供后端支持
		eth.ApiBackend = &EthApiBackend{eth, nil}
		// gpoParams GPO Gas Price Oracle 的缩写。 GasPrice预测。 通过最近的交易来预测当前的GasPrice的值。这个值可以作为之后发送交易的费用的参考。
		gpoParams := config.GPO
		if gpoParams.Default == nil {
			gpoParams.Default = config.GasPrice
		}
		eth.ApiBackend.gpo = gasprice.NewOracle(eth.ApiBackend, gpoParams)
	
		return eth, nil
	}

// New creates a new Ethereum object (including the
// initialisation of the common Ethereum object)
func New(stack *node.Node, config *Config) (*Ethereum, error) {

    //=============== 验证配置是否正确和兼容 =====================
	// Ensure configuration values are compatible and sane
	if config.SyncMode == downloader.LightSync { 
        // eth.Ethereum这是全节点实现, 要使用轻节点,需要使用les.LightEthereum
		return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
	}
	if !config.SyncMode.IsValid() {
		return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
	}

    // 验证gasprice是否大于0
	if config.Miner.GasPrice == nil || config.Miner.GasPrice.Cmp(common.Big0) <= 0 {
		log.Warn("Sanitizing invalid miner gas price", "provided", config.Miner.GasPrice, "updated", DefaultConfig.Miner.GasPrice)
		config.Miner.GasPrice = new(big.Int).Set(DefaultConfig.Miner.GasPrice)
	}
    // 是否裁减
	if config.NoPruning && config.TrieDirtyCache > 0 {
		if config.SnapshotCache > 0 {
			config.TrieCleanCache += config.TrieDirtyCache * 3 / 5
			config.SnapshotCache += config.TrieDirtyCache * 2 / 5
		} else {
			config.TrieCleanCache += config.TrieDirtyCache
		}
		config.TrieDirtyCache = 0
	}
	log.Info("Allocated trie memory caches", "clean", common.StorageSize(config.TrieCleanCache)*1024*1024, "dirty", common.StorageSize(config.TrieDirtyCache)*1024*1024)

    // =================== 创建 Ethereum对象 ================================
	// Assemble the Ethereum object

    // 创建leveldb。 打开或者新建chaindata目录
	chainDb, err := stack.OpenDatabaseWithFreezer("chaindata", config.DatabaseCache, config.DatabaseHandles, config.DatabaseFreezer, "eth/db/chaindata/")
	if err != nil {
		return nil, err
	}

    // 创世区块
	chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
	if _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil && !ok {
		return nil, genesisErr
	}
	log.Info("Initialised chain configuration", "config", chainConfig)

    // 构造
	eth := &Ethereum{
		config:            config,  // 外部可配的配置
		chainDb:           chainDb,  // 数据库
		eventMux:          stack.EventMux(), 
		accountManager:    stack.AccountManager(), // 账户管理
		engine:            CreateConsensusEngine(stack, chainConfig, &config.Ethash, config.Miner.Notify, config.Miner.Noverify, chainDb), // 共识默认是(PoW , 即 Ethash)
		closeBloomHandler: make(chan struct{}), 
		networkID:         config.NetworkId, // 网络id
		gasPrice:          config.Miner.GasPrice, // 最小gasPrice, 如果交易低于此值则不打包
		etherbase:         config.Miner.Etherbase, // 矿工的地址
		bloomRequests:     make(chan chan *bloombits.Retrieval), // bloomFilter 相关
		bloomIndexer:      NewBloomIndexer(chainDb, params.BloomBitsBlocks, params.BloomConfirms), // bloomFilter 相关
		p2pServer:         stack.Server(), // P2P服务
	}

	bcVersion := rawdb.ReadDatabaseVersion(chainDb) // 数据库版本
	var dbVer = "<nil>"
	if bcVersion != nil {
		dbVer = fmt.Sprintf("%d", *bcVersion)
	}
	log.Info("Initialising Ethereum protocol", "network", config.NetworkId, "dbversion", dbVer)

    // 是否跳过区块链版本检查
	if !config.SkipBcVersionCheck {
		if bcVersion != nil && *bcVersion > core.BlockChainVersion {
			return nil, fmt.Errorf("database version is v%d, Geth %s only supports v%d", *bcVersion, params.VersionWithMeta, core.BlockChainVersion)
		} else if bcVersion == nil || *bcVersion < core.BlockChainVersion {
			log.Warn("Upgrade blockchain database version", "from", dbVer, "to", core.BlockChainVersion)
			rawdb.WriteDatabaseVersion(chainDb, core.BlockChainVersion)
		}
	}

	var (
		vmConfig = vm.Config{
			EnablePreimageRecording: config.EnablePreimageRecording,
			EWASMInterpreter:        config.EWASMInterpreter,
			EVMInterpreter:          config.EVMInterpreter,
		}
		cacheConfig = &core.CacheConfig{
			TrieCleanLimit:      config.TrieCleanCache,
			TrieCleanJournal:    stack.ResolvePath(config.TrieCleanCacheJournal),
			TrieCleanRejournal:  config.TrieCleanCacheRejournal,
			TrieCleanNoPrefetch: config.NoPrefetch,
			TrieDirtyLimit:      config.TrieDirtyCache,
			TrieDirtyDisabled:   config.NoPruning,
			TrieTimeLimit:       config.TrieTimeout,
			SnapshotLimit:       config.SnapshotCache,
			Preimages:           config.Preimages,
		}
	)

    // 区块链相关(即区块相关)
	eth.blockchain, err = core.NewBlockChain(chainDb, cacheConfig, chainConfig, eth.engine, vmConfig, eth.shouldPreserve, &config.TxLookupLimit)
	if err != nil {
		return nil, err
	}
	// Rewind the chain in case of an incompatible config upgrade.
	if compat, ok := genesisErr.(*params.ConfigCompatError); ok {
		log.Warn("Rewinding chain to upgrade configuration", "err", compat)
		eth.blockchain.SetHead(compat.RewindTo)
		rawdb.WriteChainConfig(chainDb, genesisHash, chainConfig)
	}
	eth.bloomIndexer.Start(eth.blockchain) // bloom过滤索引器

    // 是否将交易池中的交易写入磁盘(节点重启之后可以恢复txPool, 这个配置对于交易所或者钱包的全节点很重要, 可以防止交易丢失)
	if config.TxPool.Journal != "" {
		config.TxPool.Journal = stack.ResolvePath(config.TxPool.Journal)
	}
    // 交易池
	eth.txPool = core.NewTxPool(config.TxPool, chainConfig, eth.blockchain)

    // 允许下载程序在快速同步期间使用trie缓存余量
	// Permit the downloader to use the trie cache allowance during fast sync
	cacheLimit := cacheConfig.TrieCleanLimit + cacheConfig.TrieDirtyLimit + cacheConfig.SnapshotLimit
	checkpoint := config.Checkpoint
	if checkpoint == nil {
		checkpoint = params.TrustedCheckpoints[genesisHash]
	}
    // 创建handler,  这个handler主要是广播交易,广播(挖出的)区块,管理P2P相关
	if eth.handler, err = newHandler(&handlerConfig{
		Database:   chainDb,
		Chain:      eth.blockchain,
		TxPool:     eth.txPool,
		Network:    config.NetworkId,
		Sync:       config.SyncMode,
		BloomCache: uint64(cacheLimit),
		EventMux:   eth.eventMux,
		Checkpoint: checkpoint,
		Whitelist:  config.Whitelist,
	}); err != nil {
		return nil, err
	}

    // 创建矿工
	eth.miner = miner.New(eth, &config.Miner, chainConfig, eth.EventMux(), eth.engine, eth.isLocalBlock)
	eth.miner.SetExtra(makeExtraData(config.Miner.ExtraData))

    // RPC服务后端实现
	eth.APIBackend = &EthAPIBackend{stack.Config().ExtRPCEnabled(), eth, nil}
	gpoParams := config.GPO
	if gpoParams.Default == nil {
		gpoParams.Default = config.Miner.GasPrice
	}
    // gasPrice相关
	eth.APIBackend.gpo = gasprice.NewOracle(eth.APIBackend, gpoParams)

    // 发现服务相关
	eth.ethDialCandidates, err = setupDiscovery(eth.config.EthDiscoveryURLs)
	if err != nil {
		return nil, err
	}
    // 快照发现服务相关
	eth.snapDialCandidates, err = setupDiscovery(eth.config.SnapDiscoveryURLs)
	if err != nil {
		return nil, err
	}

    // RPC服务
	// Start the RPC service
	eth.netRPCService = ethapi.NewPublicNetAPI(eth.p2pServer, config.NetworkId)

	// Register the backend on the node
	stack.RegisterAPIs(eth.APIs()) // 注册几个服务后端 eth, miner, downloader,filter,admin,debug,net
	stack.RegisterProtocols(eth.Protocols())  // P2P协议
	stack.RegisterLifecycle(eth) // 注册 eth, 加入lifecycles, node会对lifecycles进行管理
	// Check for unclean shutdown
	if uncleanShutdowns, discards, err := rawdb.PushUncleanShutdownMarker(chainDb); err != nil {
		log.Error("Could not update unclean-shutdown-marker list", "error", err)
	} else {
		if discards > 0 {
			log.Warn("Old unclean shutdowns found", "count", discards)
		}
		for _, tstamp := range uncleanShutdowns {
			t := time.Unix(int64(tstamp), 0)
			log.Warn("Unclean shutdown detected", "booted", t,
				"age", common.PrettyAge(t))
		}
	}
	return eth, nil
}
```



ApiBackend 定义在 api_backend.go文件中。 封装了一些函数。

```go
// EthAPIBackend implements ethapi.Backend for full nodes
type EthAPIBackend struct {
	extRPCEnabled bool
	eth           *Ethereum
	gpo           *gasprice.Oracle
}
```



```go

// Start implements node.Lifecycle, starting all internal goroutines needed by the
// Ethereum protocol implementation.
func (s *Ethereum) Start() error {
	eth.StartENRUpdater(s.blockchain, s.p2pServer.LocalNode())

	// Start the bloom bits servicing goroutines
	s.startBloomHandlers(params.BloomBitsBlocks)

	// Figure out a max peers count based on the server limits
	maxPeers := s.p2pServer.MaxPeers
	if s.config.LightServ > 0 {
		if s.config.LightPeers >= s.p2pServer.MaxPeers {
			return fmt.Errorf("invalid peer config: light peer count (%d) >= total peer count (%d)", s.config.LightPeers, s.p2pServer.MaxPeers)
		}
		maxPeers -= s.config.LightPeers
	}
	// Start the networking layer and the light server if requested
	s.handler.Start(maxPeers)
	return nil
}
```

handler

```go

type handler struct {
	networkID  uint64
	forkFilter forkid.Filter // Fork ID filter, constant across the lifetime of the node

	fastSync  uint32 // Flag whether fast sync is enabled (gets disabled if we already have blocks)
	snapSync  uint32 // Flag whether fast sync should operate on top of the snap protocol
	acceptTxs uint32 // Flag whether we're considered synchronised (enables transaction processing)

	checkpointNumber uint64      // Block number for the sync progress validator to cross reference
	checkpointHash   common.Hash // Block hash for the sync progress validator to cross reference

	database ethdb.Database
	txpool   txPool
	chain    *core.BlockChain
	maxPeers int

	downloader   *downloader.Downloader
	stateBloom   *trie.SyncBloom
	blockFetcher *fetcher.BlockFetcher
	txFetcher    *fetcher.TxFetcher
	peers        *peerSet

	eventMux      *event.TypeMux
	txsCh         chan core.NewTxsEvent
	txsSub        event.Subscription
	minedBlockSub *event.TypeMuxSubscription

	whitelist map[uint64]common.Hash

	// channels for fetcher, syncer, txsyncLoop
	txsyncCh chan *txsync
	quitSync chan struct{}

	chainSync *chainSyncer
	wg        sync.WaitGroup
	peerWG    sync.WaitGroup
}


// newHandler returns a handler for all Ethereum chain management protocol.
func newHandler(config *handlerConfig) (*handler, error) {
	// Create the protocol manager with the base fields
	if config.EventMux == nil {
		config.EventMux = new(event.TypeMux) // Nicety initialization for tests
	}
	h := &handler{
		networkID:  config.Network,
		forkFilter: forkid.NewFilter(config.Chain),
		eventMux:   config.EventMux,
		database:   config.Database,
		txpool:     config.TxPool,
		chain:      config.Chain,
		peers:      newPeerSet(),
		whitelist:  config.Whitelist,
		txsyncCh:   make(chan *txsync),
		quitSync:   make(chan struct{}),
	}
	if config.Sync == downloader.FullSync {
		// The database seems empty as the current block is the genesis. Yet the fast
		// block is ahead, so fast sync was enabled for this node at a certain point.
		// The scenarios where this can happen is
		// * if the user manually (or via a bad block) rolled back a fast sync node
		//   below the sync point.
		// * the last fast sync is not finished while user specifies a full sync this
		//   time. But we don't have any recent state for full sync.
		// In these cases however it's safe to reenable fast sync.
		fullBlock, fastBlock := h.chain.CurrentBlock(), h.chain.CurrentFastBlock()
		if fullBlock.NumberU64() == 0 && fastBlock.NumberU64() > 0 {
			h.fastSync = uint32(1)
			log.Warn("Switch sync mode from full sync to fast sync")
		}
	} else {
		if h.chain.CurrentBlock().NumberU64() > 0 {
			// Print warning log if database is not empty to run fast sync.
			log.Warn("Switch sync mode from fast sync to full sync")
		} else {
			// If fast sync was requested and our database is empty, grant it
			h.fastSync = uint32(1)
			if config.Sync == downloader.SnapSync {
				h.snapSync = uint32(1)
			}
		}
	}
	// If we have trusted checkpoints, enforce them on the chain
	if config.Checkpoint != nil {
		h.checkpointNumber = (config.Checkpoint.SectionIndex+1)*params.CHTFrequency - 1
		h.checkpointHash = config.Checkpoint.SectionHead
	}
	// Construct the downloader (long sync) and its backing state bloom if fast
	// sync is requested. The downloader is responsible for deallocating the state
	// bloom when it's done.
	if atomic.LoadUint32(&h.fastSync) == 1 {
		h.stateBloom = trie.NewSyncBloom(config.BloomCache, config.Database)
	}
	h.downloader = downloader.New(h.checkpointNumber, config.Database, h.stateBloom, h.eventMux, h.chain, nil, h.removePeer)

	// Construct the fetcher (short sync)
	validator := func(header *types.Header) error {
		return h.chain.Engine().VerifyHeader(h.chain, header, true)
	}
	heighter := func() uint64 {
		return h.chain.CurrentBlock().NumberU64()
	}
	inserter := func(blocks types.Blocks) (int, error) {
		// If sync hasn't reached the checkpoint yet, deny importing weird blocks.
		//
		// Ideally we would also compare the head block's timestamp and similarly reject
		// the propagated block if the head is too old. Unfortunately there is a corner
		// case when starting new networks, where the genesis might be ancient (0 unix)
		// which would prevent full nodes from accepting it.
		if h.chain.CurrentBlock().NumberU64() < h.checkpointNumber {
			log.Warn("Unsynced yet, discarded propagated block", "number", blocks[0].Number(), "hash", blocks[0].Hash())
			return 0, nil
		}
		// If fast sync is running, deny importing weird blocks. This is a problematic
		// clause when starting up a new network, because fast-syncing miners might not
		// accept each others' blocks until a restart. Unfortunately we haven't figured
		// out a way yet where nodes can decide unilaterally whether the network is new
		// or not. This should be fixed if we figure out a solution.
		if atomic.LoadUint32(&h.fastSync) == 1 {
			log.Warn("Fast syncing, discarded propagated block", "number", blocks[0].Number(), "hash", blocks[0].Hash())
			return 0, nil
		}
		n, err := h.chain.InsertChain(blocks)
		if err == nil {
			atomic.StoreUint32(&h.acceptTxs, 1) // Mark initial sync done on any fetcher import
		}
		return n, err
	}
	h.blockFetcher = fetcher.NewBlockFetcher(false, nil, h.chain.GetBlockByHash, validator, h.BroadcastBlock, heighter, nil, inserter, h.removePeer)

	fetchTx := func(peer string, hashes []common.Hash) error {
		p := h.peers.ethPeer(peer)
		if p == nil {
			return errors.New("unknown peer")
		}
		return p.RequestTxs(hashes)
	}
	h.txFetcher = fetcher.NewTxFetcher(h.txpool.Has, h.txpool.AddRemotes, fetchTx)
	h.chainSync = newChainSyncer(h)
	return h, nil
}
```



协议管理器的Start方法。这个方法里面启动了大量的goroutine用来处理各种事务，可以推测，这个类应该是以太坊服务的主要实现类。


```go
func (h *handler) Start(maxPeers int) {
	h.maxPeers = maxPeers

	// broadcast transactions
	h.wg.Add(1)

    // 订阅新交易事件
	h.txsCh = make(chan core.NewTxsEvent, txChanSize)
	h.txsSub = h.txpool.SubscribeNewTxsEvent(h.txsCh) 
	go h.txBroadcastLoop()

	// broadcast mined blocks
	h.wg.Add(1)
    // 订阅挖矿消息。当新的Block被挖出来的时候会产生消息
	h.minedBlockSub = h.eventMux.Subscribe(core.NewMinedBlockEvent{})
    // 挖矿广播 goroutine 当挖出来的时候需要尽快的广播到网络上面去。
	go h.minedBroadcastLoop()

	// start sync handlers
	h.wg.Add(2)
	go h.chainSync.loop()  // 同步
	go h.txsyncLoop64() // TODO(karalabe): Legacy initial tx echange, drop with eth/64.
}
```





handle方法

```go
// 当对等方的消息处理程序收到新的远程消息时，将调用该句柄消息，表明处理程序无法使用和服务自己。 
// Handle is invoked from a peer's message handler when it receives a new remote
// message that the handler couldn't consume and serve itself.
func (h *ethHandler) Handle(peer *eth.Peer, packet eth.Packet) error {
	// Consume any broadcasts and announces, forwarding the rest to the downloader
	switch packet := packet.(type) {
	case *eth.BlockHeadersPacket: // 区块头
		return h.handleHeaders(peer, *packet)

	case *eth.BlockBodiesPacket: // 区块体
		txset, uncleset := packet.Unpack()
		return h.handleBodies(peer, txset, uncleset)

	case *eth.NodeDataPacket: // 节点数据
		if err := h.downloader.DeliverNodeData(peer.ID(), *packet); err != nil {
			log.Debug("Failed to deliver node state data", "err", err)
		}
		return nil

	case *eth.ReceiptsPacket: // 收据
		if err := h.downloader.DeliverReceipts(peer.ID(), *packet); err != nil {
			log.Debug("Failed to deliver receipts", "err", err)
		}
		return nil

	case *eth.NewBlockHashesPacket: // 区块hash
		hashes, numbers := packet.Unpack()
		return h.handleBlockAnnounces(peer, hashes, numbers)

	case *eth.NewBlockPacket: // 新区块
		return h.handleBlockBroadcast(peer, packet.Block, packet.TD)

	case *eth.NewPooledTransactionHashesPacket: // 新交易(hash)加入交易池
		return h.txFetcher.Notify(peer.ID(), *packet)

	case *eth.TransactionsPacket: // 交易
		return h.txFetcher.Enqueue(peer.ID(), *packet, false)

	case *eth.PooledTransactionsPacket: // 交易池交易
		return h.txFetcher.Enqueue(peer.ID(), *packet, true)

	default:
		return fmt.Errorf("unexpected eth packet type: %T", packet)
	}
}
```


总结一下。 我们现在的一些大的流程。

区块同步

1. 如果是自己挖的矿。通过goroutine minedBroadcastLoop()来进行广播。
2. 如果是接收到其他人的区块广播，(NewBlockHashesMsg/NewBlockMsg),是否fetcher会通知的peer？ TODO
3. goroutine syncer()中会定时的同BestPeer()来同步信息。

交易同步

1. 新建立连接。 把pending的交易发送给他。
2. 本地发送了一个交易，或者是接收到别人发来的交易信息。 txpool会产生一条消息，消息被传递到txCh通道。 然后被goroutine txBroadcastLoop()处理， 发送给其他不知道这个交易的peer。