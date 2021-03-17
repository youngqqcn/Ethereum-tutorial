## cmd/geth
### geth/main.go
geth是cmd中最重要的命令程序，它是以太坊的入口。geth的命令行是通过`gopkg.in/urfave/cli.v1`这个库进行实现的，通过这个库，我们可以轻松定义命令行程序的子命令，命令选项，命令参数，描述信息等等。

geth 模块的入口在 `cmd/geth/main.go `中，它会调用 `urfave/cli` 的中 `app` 的 Run 方法，而 app 在 init 函数中初始化，在 Golang 中，如果有 `init` 方法，那么会在 `main` 函数之前执行 `init` 函数，它用于程序执行前的初始化工作。在 `geth` 模块中，`init()` 函数定义了命令行的入口是 `geth`，并且定义了 `geth` 的子命令、全局的命令选项、子命令的命令选项，按照 `urfave/cli` 的做法，不输入子命令会默认调用 `geth`，而 `geth` 方法其实就6行：


```go
// 它会调用 makeFullNode 函数初始化一个全节点，接着通过 startNode 函数启动一个全节点，以阻塞的方式运行，等待着节点被终止。
// geth is the main entry point into the system if no special subcommand is ran.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.

func geth(ctx *cli.Context) error {
	if args := ctx.Args(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}

	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	defer stack.Close()

	startNode(ctx, stack, backend)
	stack.Wait()
	return nil
}
```


```go
// makeFullNode核心的逻辑是首先通过配置文件和 flag 生成系统级的配置，然后将服务注入到节点。
// makeFullNode loads geth configuration and creates the Ethereum backend.
func makeFullNode(ctx *cli.Context) (*node.Node, ethapi.Backend) {
	stack, cfg := makeConfigNode(ctx)

	backend := utils.RegisterEthService(stack, &cfg.Eth)

	checkWhisper(ctx)
	// Configure GraphQL if requested
	if ctx.GlobalIsSet(utils.GraphQLEnabledFlag.Name) {
		utils.RegisterGraphQLService(stack, backend, cfg.Node)
	}
	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
		utils.RegisterEthStatsService(stack, backend, cfg.Ethstats.URL)
	}
	return stack, backend
}
```


```go
// makeConfigNode 会先载入默认配置，再载入配置文件中的配置，然后通过上下文的配置(在 cmd/geth/main.go 中的 init 方法中定义)进行设置。
// makeConfigNode loads geth configuration and creates a blank node instance.
func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
	// Load defaults.
	cfg := gethConfig{
		Eth:     eth.DefaultConfig,
		Node:    defaultNodeConfig(),
		Metrics: metrics.DefaultConfig,
	}

	// Load config file.
	if file := ctx.GlobalString(configFileFlag.Name); file != "" {
		if err := loadConfig(file, &cfg); err != nil {
			utils.Fatalf("%v", err)
		}

		if cfg.Shh != (whisperDeprecatedConfig{}) {
			log.Warn("Deprecated whisper config detected. Whisper has been moved to github.com/ethereum/whisper")
		}
	}
	// Apply flags.
	utils.SetNodeConfig(ctx, &cfg.Node)
	stack, err := node.New(&cfg.Node)
	if err != nil {
		utils.Fatalf("Failed to create the protocol stack: %v", err)
	}
	utils.SetEthConfig(ctx, stack, &cfg.Eth)
	if ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
		cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
	}
	utils.SetShhConfig(ctx, stack)

	applyMetricConfig(ctx, &cfg)

	return stack, cfg
}

```


```go
// `RegisterEthService` 的代码在 `cmd/utils/flags.go` 中，如果同步模式是轻量级同步模式，启动轻量级客户端，否则启动全节点，实际的注册方法是`RegisterLifecycle`。注入服务其实就是将新的服务注入到 node 对象的 `lifecycles` 数组中。

// cmd/utils/flags.go
// RegisterEthStatsService configures the Ethereum Stats daemon and adds it to
// the given node.
func RegisterEthStatsService(stack *node.Node, backend ethapi.Backend, url string) {
	if err := ethstats.New(stack, backend, backend.Engine(), url); err != nil {
		Fatalf("Failed to register the Ethereum Stats service: %v", err)
	}
}

// ethstats.ethstats.go
// New returns a monitoring service ready for stats reporting.
func New(node *node.Node, backend backend, engine consensus.Engine, url string) error {
	// Parse the netstats connection url
	re := regexp.MustCompile("([^:@]*)(:([^@]*))?@(.+)")
	parts := re.FindStringSubmatch(url)
	if len(parts) != 5 {
		return fmt.Errorf("invalid netstats url: \"%s\", should be nodename:secret@host:port", url)
	}
	ethstats := &Service{
		backend: backend,
		engine:  engine,
		server:  node.Server(),
		node:    parts[1],
		pass:    parts[3],
		host:    parts[4],
		pongCh:  make(chan struct{}),
		histCh:  make(chan []uint64, 1),
	}

	node.RegisterLifecycle(ethstats)
	return nil
}

// node/node.go
// RegisterLifecycle registers the given Lifecycle on the node.
func (n *Node) RegisterLifecycle(lifecycle Lifecycle) {
	n.lock.Lock()
	defer n.lock.Unlock()

	if n.state != initializingState {
		panic("can't register lifecycle on running/stopped node")
	}
	if containsLifecycle(n.lifecycles, lifecycle) {
		panic(fmt.Sprintf("attempt to register lifecycle %T more than once", lifecycle))
	}
	n.lifecycles = append(n.lifecycles, lifecycle)
}

```



```go
// startNode 方法启动节点，会开启所有已经注册的协议，解锁请求的账户，开启 RPC/IPC 接口，并开始挖矿。

// startNode boots up the system node and all registered protocols, after which
// it unlocks any requested accounts, and starts the RPC/IPC interfaces and the
// miner.
func startNode(ctx *cli.Context, stack *node.Node, backend ethapi.Backend) {
	debug.Memsize.Add("node", stack)

	// Start up the node itself
	utils.StartNode(ctx, stack)

	// Unlock any account specifically requested
	unlockAccounts(ctx, stack)

	// Register wallet event handlers to open and auto-derive wallets
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)

	// Create a client to interact with local geth node.
	rpcClient, err := stack.Attach()
	if err != nil {
		utils.Fatalf("Failed to attach to self: %v", err)
	}
	ethClient := ethclient.NewClient(rpcClient)

    // 钱包和账户
	go func() {
		// Open any wallets already attached
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			}
		}
		// Listen for wallet event till termination
		for event := range events {
			switch event.Kind {
			case accounts.WalletArrived:
				if err := event.Wallet.Open(""); err != nil {
					log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
				}
			case accounts.WalletOpened:
				status, _ := event.Wallet.Status()
				log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)

				var derivationPaths []accounts.DerivationPath
				if event.Wallet.URL().Scheme == "ledger" {
					derivationPaths = append(derivationPaths, accounts.LegacyLedgerBaseDerivationPath)
				}
				derivationPaths = append(derivationPaths, accounts.DefaultBaseDerivationPath)

				event.Wallet.SelfDerive(derivationPaths, ethClient)

			case accounts.WalletDropped:
				log.Info("Old wallet dropped", "url", event.Wallet.URL())
				event.Wallet.Close()
			}
		}
	}()

    // 状态同步监控
	// Spawn a standalone goroutine for status synchronization monitoring,
	// close the node when synchronization is complete if user required.
	if ctx.GlobalBool(utils.ExitWhenSyncedFlag.Name) {
		go func() {
			sub := stack.EventMux().Subscribe(downloader.DoneEvent{})
			defer sub.Unsubscribe()
			for {
				event := <-sub.Chan()
				if event == nil {
					continue
				}
				done, ok := event.Data.(downloader.DoneEvent)
				if !ok {
					continue
				}
				if timestamp := time.Unix(int64(done.Latest.Time), 0); time.Since(timestamp) < 10*time.Minute {
					log.Info("Synchronisation completed", "latestnum", done.Latest.Number, "latesthash", done.Latest.Hash(),
						"age", common.PrettyAge(timestamp))
					stack.Close()
				}
			}
		}()
	}

    // 挖矿 或  开发者模式
	// Start auxiliary services if enabled
	if ctx.GlobalBool(utils.MiningEnabledFlag.Name) || ctx.GlobalBool(utils.DeveloperFlag.Name) {
		// Mining only makes sense if a full Ethereum node is running
		if ctx.GlobalString(utils.SyncModeFlag.Name) == "light" {
			utils.Fatalf("Light clients do not support mining")
		}
		ethBackend, ok := backend.(*eth.EthAPIBackend)
		if !ok {
			utils.Fatalf("Ethereum service not running: %v", err)
		}

		// Set the gas price to the limits from the CLI and start mining
		gasprice := utils.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
		if ctx.GlobalIsSet(utils.LegacyMinerGasPriceFlag.Name) && !ctx.GlobalIsSet(utils.MinerGasPriceFlag.Name) {
			gasprice = utils.GlobalBig(ctx, utils.LegacyMinerGasPriceFlag.Name)
		}
		ethBackend.TxPool().SetGasPrice(gasprice) // 设置交易池的gasprice
		// start mining
		threads := ctx.GlobalInt(utils.MinerThreadsFlag.Name) // 挖矿的线程数
		if ctx.GlobalIsSet(utils.LegacyMinerThreadsFlag.Name) && !ctx.GlobalIsSet(utils.MinerThreadsFlag.Name) {
			threads = ctx.GlobalInt(utils.LegacyMinerThreadsFlag.Name)
			log.Warn("The flag --minerthreads is deprecated and will be removed in the future, please use --miner.threads")
		}
		if err := ethBackend.StartMining(threads); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}
```

```go
// 初始化, urfave/cli.v1的 cli.NewApp()
func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth  // 如果不传参数, 则没人执行此函数
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2021 The go-ethereum Authors"


    // 注册相关命令
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,            // 初始化
		importCommand,          // 导入
		exportCommand,          // 导入
		importPreimagesCommand, // 
		exportPreimagesCommand, //
		copydbCommand,          // 拷贝db
		removedbCommand,        // 删除db
		dumpCommand,            // dump
		dumpGenesisCommand,     // dump genesis
		inspectCommand,         // 检查

		// See accountcmd.go:
		accountCommand,         // 账户相关命令
		walletCommand,          // 钱包命令

		// See consolecmd.go:
		consoleCommand,         // 控制台
		attachCommand,          // attach
		javascriptCommand,      // js命令

		// See misccmd.go:
		makecacheCommand,       // 构造cache
		makedagCommand,         // 构造DAG
		versionCommand,         // 版本
		versionCheckCommand,    // 版本检查
		licenseCommand,         // 证书

		// See config.go
		dumpConfigCommand,
		// See cmd/utils/flags_legacy.go
		utils.ShowDeprecated,
	}
```


### geth/chaincmd.go

```go
// initCommand会进行初始化，生成初始区块，调用了SetupGenesisBlock

func initGenesis(ctx *cli.Context) error {
	genesisPath := ctx.Args().First()
	if len(genesisPath) == 0 {
		utils.Fatalf("Must supply path to genesis JSON file")
	}
	file, err := os.Open(genesisPath)
	if err != nil {
		utils.Fatalf("Failed to read genesis file: %v", err)
	}
	defer file.Close()
	genesis := new(core.Genesis)
	if err := json.NewDecoder(file).Decode(genesis); err != nil {
		utils.Fatalf("invalid genesis file: %v", err)
	}
	stack := makeFullNode(ctx)
	for _, name := range []string{"chaindata", "lightchaindata"} {
		chaindb, err := stack.OpenDatabase(name, 0, 0)
		if err != nil {
			utils.Fatalf("Failed to open database: %v", err)
		}
		_, hash, err := core.SetupGenesisBlock(chaindb, genesis)
		if err != nil {
			utils.Fatalf("Failed to write genesis block: %v", err)
		}
		log.Info("Successfully wrote genesis state", "database", name, "hash", hash)
	}
	return nil
}
```




```go
// dump出不同链类型的genesis区块

func dumpGenesis(ctx *cli.Context) error {
	genesis := utils.MakeGenesis(ctx)
	if genesis == nil {
		genesis = core.DefaultGenesisBlock()
	}
	if err := json.NewEncoder(os.Stdout).Encode(genesis); err != nil {
		utils.Fatalf("could not encode genesis")
	}
	return nil
}
```



```go
// ImportChain导入了一个区块链文件
func importChain(ctx *cli.Context) error {
	if len(ctx.Args()) < 1 {
		utils.Fatalf("This command requires an argument.")
	}
	// Start metrics export if enabled
	utils.SetupMetrics(ctx)
	// Start system runtime metrics collection
	go metrics.CollectProcessMetrics(3 * time.Second)

	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	chain, db := utils.MakeChain(ctx, stack, false)
	defer db.Close()

	// Start periodically gathering memory profiles
	var peakMemAlloc, peakMemSys uint64
	go func() {
		stats := new(runtime.MemStats)
		for {
			runtime.ReadMemStats(stats)
			if atomic.LoadUint64(&peakMemAlloc) < stats.Alloc {
				atomic.StoreUint64(&peakMemAlloc, stats.Alloc)
			}
			if atomic.LoadUint64(&peakMemSys) < stats.Sys {
				atomic.StoreUint64(&peakMemSys, stats.Sys)
			}
			time.Sleep(5 * time.Second)
		}
	}()
	// Import the chain
	start := time.Now()

	var importErr error

	if len(ctx.Args()) == 1 {
		if err := utils.ImportChain(chain, ctx.Args().First()); err != nil {
			importErr = err
			log.Error("Import error", "err", err)
		}
	} else {
		for _, arg := range ctx.Args() {
			if err := utils.ImportChain(chain, arg); err != nil {
				importErr = err
				log.Error("Import error", "file", arg, "err", err)
			}
		}
	}
	chain.Stop()
	fmt.Printf("Import done in %v.\n\n", time.Since(start))

	// Output pre-compaction stats mostly to see the import trashing
	stats, err := db.Stat("leveldb.stats")
	if err != nil {
		utils.Fatalf("Failed to read database stats: %v", err)
	}
	fmt.Println(stats)

	ioStats, err := db.Stat("leveldb.iostats")
	if err != nil {
		utils.Fatalf("Failed to read database iostats: %v", err)
	}
	fmt.Println(ioStats)

	// Print the memory statistics used by the importing
	mem := new(runtime.MemStats)
	runtime.ReadMemStats(mem)

	fmt.Printf("Object memory: %.3f MB current, %.3f MB peak\n", float64(mem.Alloc)/1024/1024, float64(atomic.LoadUint64(&peakMemAlloc))/1024/1024)
	fmt.Printf("System memory: %.3f MB current, %.3f MB peak\n", float64(mem.Sys)/1024/1024, float64(atomic.LoadUint64(&peakMemSys))/1024/1024)
	fmt.Printf("Allocations:   %.3f million\n", float64(mem.Mallocs)/1000000)
	fmt.Printf("GC pause:      %v\n\n", time.Duration(mem.PauseTotalNs))

	if ctx.GlobalBool(utils.NoCompactionFlag.Name) {
		return nil
	}

	// Compact the entire database to more accurately measure disk io and print the stats
	start = time.Now()
	fmt.Println("Compacting entire database...")
	if err = db.Compact(nil, nil); err != nil {
		utils.Fatalf("Compaction failed: %v", err)
	}
	fmt.Printf("Compaction done in %v.\n\n", time.Since(start))

	stats, err = db.Stat("leveldb.stats")
	if err != nil {
		utils.Fatalf("Failed to read database stats: %v", err)
	}
	fmt.Println(stats)

	ioStats, err = db.Stat("leveldb.iostats")
	if err != nil {
		utils.Fatalf("Failed to read database iostats: %v", err)
	}
	fmt.Println(ioStats)
	return importErr
}

```


```go
// exportChain导出一个区块链gz文件
func exportChain(ctx *cli.Context) error {
	if len(ctx.Args()) < 1 {
		utils.Fatalf("This command requires an argument.")
	}

	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	chain, _ := utils.MakeChain(ctx, stack, true)
	start := time.Now()

	var err error
	fp := ctx.Args().First()
	if len(ctx.Args()) < 3 {
		err = utils.ExportChain(chain, fp)
	} else {
		// This can be improved to allow for numbers larger than 9223372036854775807
		first, ferr := strconv.ParseInt(ctx.Args().Get(1), 10, 64)
		last, lerr := strconv.ParseInt(ctx.Args().Get(2), 10, 64)
		if ferr != nil || lerr != nil {
			utils.Fatalf("Export error in parsing parameters: block number not an integer\n")
		}
		if first < 0 || last < 0 {
			utils.Fatalf("Export error: block number must be greater than 0\n")
		}
		err = utils.ExportAppendChain(chain, fp, uint64(first), uint64(last))
	}

	if err != nil {
		utils.Fatalf("Export error: %v\n", err)
	}
	fmt.Printf("Export done in %v\n", time.Since(start))
	return nil
}

```


```go
// 将一个preimages导入当前节点
// importPreimages imports preimage data from the specified file.
func importPreimages(ctx *cli.Context) error {
	if len(ctx.Args()) < 1 {
		utils.Fatalf("This command requires an argument.")
	}

	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	db := utils.MakeChainDatabase(ctx, stack)
	start := time.Now()

	if err := utils.ImportPreimages(db, ctx.Args().First()); err != nil {
		utils.Fatalf("Import error: %v\n", err)
	}
	fmt.Printf("Import done in %v\n", time.Since(start))
	return nil
}

```



```go
// 从当前节点导出一个 image
// exportPreimages dumps the preimage data to specified json file in streaming way.
func exportPreimages(ctx *cli.Context) error {
	if len(ctx.Args()) < 1 {
		utils.Fatalf("This command requires an argument.")
	}

	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	db := utils.MakeChainDatabase(ctx, stack)
	start := time.Now()

	if err := utils.ExportPreimages(db, ctx.Args().First()); err != nil {
		utils.Fatalf("Export error: %v\n", err)
	}
	fmt.Printf("Export done in %v\n", time.Since(start))
	return nil
}

```


```go
// 复制一个本地区块文件到文件夹;在一个文件夹中创建一个本地区块链,但是这个过程并不是直接复制过去的，而是通过 `downloader` 模块里的 `NewFakePeer` 创建一个虚拟对等节点，然后再进行数据同步完成的。


func copyDb(ctx *cli.Context) error {
	// Ensure we have a source chain directory to copy
	if len(ctx.Args()) < 1 {
		utils.Fatalf("Source chaindata directory path argument missing")
	}
	if len(ctx.Args()) < 2 {
		utils.Fatalf("Source ancient chain directory path argument missing")
	}
	// Initialize a new chain for the running node to sync into
	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	chain, chainDb := utils.MakeChain(ctx, stack, false)
	syncMode := *utils.GlobalTextMarshaler(ctx, utils.SyncModeFlag.Name).(*downloader.SyncMode)

	var syncBloom *trie.SyncBloom
	if syncMode == downloader.FastSync {
		syncBloom = trie.NewSyncBloom(uint64(ctx.GlobalInt(utils.CacheFlag.Name)/2), chainDb)
	}
	dl := downloader.New(0, chainDb, syncBloom, new(event.TypeMux), chain, nil, nil)

	// Create a source peer to satisfy downloader requests from
	db, err := rawdb.NewLevelDBDatabaseWithFreezer(ctx.Args().First(), ctx.GlobalInt(utils.CacheFlag.Name)/2, 256, ctx.Args().Get(1), "")
	if err != nil {
		return err
	}
	hc, err := core.NewHeaderChain(db, chain.Config(), chain.Engine(), func() bool { return false })
	if err != nil {
		return err
	}
	peer := downloader.NewFakePeer("local", db, hc, dl)
	if err = dl.RegisterPeer("local", 63, peer); err != nil {
		return err
	}
	// Synchronise with the simulated peer
	start := time.Now()

	currentHeader := hc.CurrentHeader()
	if err = dl.Synchronise("local", currentHeader.Hash(), hc.GetTd(currentHeader.Hash(), currentHeader.Number.Uint64()), syncMode); err != nil {
		return err
	}
	for dl.Synchronising() {
		time.Sleep(10 * time.Millisecond)
	}
	fmt.Printf("Database copy done in %v\n", time.Since(start))

	// Compact the entire database to remove any sync overhead
	start = time.Now()
	fmt.Println("Compacting entire database...")
	if err = db.Compact(nil, nil); err != nil {
		utils.Fatalf("Compaction failed: %v", err)
	}
	fmt.Printf("Compaction done in %v.\n\n", time.Since(start))
	return nil
}
```




```go
// 在当前数据库中移除区块链,直接通过 `os` 模块移除这个文件夹。

func removeDB(ctx *cli.Context) error {
	stack, config := makeConfigNode(ctx)

	// Remove the full node state database
	path := stack.ResolvePath("chaindata")
	if common.FileExist(path) {
		confirmAndRemoveDB(path, "full node state database")
	} else {
		log.Info("Full node state database missing", "path", path)
	}
	// Remove the full node ancient database
	path = config.Eth.DatabaseFreezer
	switch {
	case path == "":
		path = filepath.Join(stack.ResolvePath("chaindata"), "ancient")
	case !filepath.IsAbs(path):
		path = config.Node.ResolvePath(path)
	}
	if common.FileExist(path) {
		confirmAndRemoveDB(path, "full node ancient database")
	} else {
		log.Info("Full node ancient database missing", "path", path)
	}
	// Remove the light node database
	path = stack.ResolvePath("lightchaindata")
	if common.FileExist(path) {
		confirmAndRemoveDB(path, "light node database")
	} else {
		log.Info("Light node database missing", "path", path)
	}
	return nil
}

// confirmAndRemoveDB prompts the user for a last confirmation and removes the
// folder if accepted.
func confirmAndRemoveDB(database string, kind string) {
	confirm, err := prompt.Stdin.PromptConfirm(fmt.Sprintf("Remove %s (%s)?", kind, database))
	switch {
	case err != nil:
		utils.Fatalf("%v", err)
	case !confirm:
		log.Info("Database deletion skipped", "path", database)
	default:
		start := time.Now()
		filepath.Walk(database, func(path string, info os.FileInfo, err error) error {
			// If we're at the top level folder, recurse into
			if path == database {
				return nil
			}
			// Delete all the files, but not subfolders
			if !info.IsDir() {
				os.Remove(path) // 直接删除
				return nil
			}
			return filepath.SkipDir
		})
		log.Info("Database successfully deleted", "path", database, "elapsed", common.PrettyDuration(time.Since(start)))
	}
}

```



```go 
// dump  转储某个区块

func dump(ctx *cli.Context) error {
	stack, _ := makeConfigNode(ctx)
	defer stack.Close()

	chain, chainDb := utils.MakeChain(ctx, stack, true)
	defer chainDb.Close()
	for _, arg := range ctx.Args() {
		var block *types.Block
		if hashish(arg) {
			block = chain.GetBlockByHash(common.HexToHash(arg))
		} else {
			num, _ := strconv.Atoi(arg)
			block = chain.GetBlockByNumber(uint64(num))
		}
		if block == nil {
			fmt.Println("{}")
			utils.Fatalf("block not found")
		} else {
			state, err := state.New(block.Root(), state.NewDatabase(chainDb), nil)
			if err != nil {
				utils.Fatalf("could not create new state: %v", err)
			}
			excludeCode := ctx.Bool(utils.ExcludeCodeFlag.Name)
			excludeStorage := ctx.Bool(utils.ExcludeStorageFlag.Name)
			includeMissing := ctx.Bool(utils.IncludeIncompletesFlag.Name)
			if ctx.Bool(utils.IterativeOutputFlag.Name) {
				state.IterativeDump(excludeCode, excludeStorage, !includeMissing, json.NewEncoder(os.Stdout))
			} else {
				if includeMissing {
					fmt.Printf("If you want to include accounts with missing preimages, you need iterative output, since" +
						" otherwise the accounts will overwrite each other in the resulting mapping.")
				}
				fmt.Printf("%v %s\n", includeMissing, state.Dump(excludeCode, excludeStorage, false))
			}
		}
	}
	return nil
}
```


### geth/accountCommand.go
这部分是管理账户

```go
// 列出所有账户
func accountList(ctx *cli.Context) error {
	stack, _ := makeConfigNode(ctx)
	var index int
	for _, wallet := range stack.AccountManager().Wallets() {
		for _, account := range wallet.Accounts() {
			fmt.Printf("Account #%d: {%x} %s\n", index, account.Address, &account.URL)
			index++
		}
	}
	return nil
}

```


```go
// 解锁
// tries unlocking the specified account a few times.
func unlockAccount(ks *keystore.KeyStore, address string, i int, passwords []string) (accounts.Account, string) {
	account, err := utils.MakeAddress(ks, address)
	if err != nil {
		utils.Fatalf("Could not list accounts: %v", err)
	}
	for trials := 0; trials < 3; trials++ {
		prompt := fmt.Sprintf("Unlocking account %s | Attempt %d/%d", address, trials+1, 3)
		password := utils.GetPassPhraseWithList(prompt, false, i, passwords)
		err = ks.Unlock(account, password)
		if err == nil {
			log.Info("Unlocked account", "address", account.Address.Hex())
			return account, password
		}
		if err, ok := err.(*keystore.AmbiguousAddrError); ok {
			log.Info("Unlocked account", "address", account.Address.Hex())
			return ambiguousAddrRecovery(ks, err, password), password
		}
		if err != keystore.ErrDecrypt {
			// No need to prompt again if the error is not decryption-related.
			break
		}
	}
	// All trials expended to unlock account, bail out
	utils.Fatalf("Failed to unlock account %s (%v)", address, err)

	return accounts.Account{}, ""
}

```


```go
// 地址恢复
func ambiguousAddrRecovery(ks *keystore.KeyStore, err *keystore.AmbiguousAddrError, auth string) accounts.Account {
	fmt.Printf("Multiple key files exist for address %x:\n", err.Addr)
	for _, a := range err.Matches {
		fmt.Println("  ", a.URL)
	}
	fmt.Println("Testing your password against all of them...")
	var match *accounts.Account
	for _, a := range err.Matches {
		if err := ks.Unlock(a, auth); err == nil {
			match = &a
			break
		}
	}
	if match == nil {
		utils.Fatalf("None of the listed files could be unlocked.")
	}
	fmt.Printf("Your password unlocked %s\n", match.URL)
	fmt.Println("In order to avoid this warning, you need to remove the following duplicate key files:")
	for _, a := range err.Matches {
		if a != *match {
			fmt.Println("  ", a.URL)
		}
	}
	return *match
}

```



```go
// 创建账户
// accountCreate creates a new account into the keystore defined by the CLI flags.
func accountCreate(ctx *cli.Context) error {
	cfg := gethConfig{Node: defaultNodeConfig()}
	// Load config file.
	if file := ctx.GlobalString(configFileFlag.Name); file != "" {
		if err := loadConfig(file, &cfg); err != nil {
			utils.Fatalf("%v", err)
		}
	}
	utils.SetNodeConfig(ctx, &cfg.Node)
	scryptN, scryptP, keydir, err := cfg.Node.AccountConfig()

	if err != nil {
		utils.Fatalf("Failed to read configuration: %v", err)
	}

	password := utils.GetPassPhraseWithList("Your new account is locked with a password. Please give a password. Do not forget this password.", true, 0, utils.MakePasswordList(ctx))

	account, err := keystore.StoreKey(keydir, password, scryptN, scryptP)

	if err != nil {
		utils.Fatalf("Failed to create account: %v", err)
	}
	fmt.Printf("\nYour new key was generated\n\n")
	fmt.Printf("Public address of the key:   %s\n", account.Address.Hex())
	fmt.Printf("Path of the secret key file: %s\n\n", account.URL.Path)
	fmt.Printf("- You can share your public address with anyone. Others need it to interact with you.\n")
	fmt.Printf("- You must NEVER share the secret key with anyone! The key controls access to your funds!\n")
	fmt.Printf("- You must BACKUP your key file! Without the key, it's impossible to access account funds!\n")
	fmt.Printf("- You must REMEMBER your password! Without the password, it's impossible to decrypt the key!\n\n")
	return nil
}
```



```go 
//  账户更新(密码)
// accountUpdate transitions an account from a previous format to the current
// one, also providing the possibility to change the pass-phrase.
func accountUpdate(ctx *cli.Context) error {
	if len(ctx.Args()) == 0 {
		utils.Fatalf("No accounts specified to update")
	}
	stack, _ := makeConfigNode(ctx)
	ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)

	for _, addr := range ctx.Args() {
		account, oldPassword := unlockAccount(ks, addr, 0, nil)
		newPassword := utils.GetPassPhraseWithList("Please give a new password. Do not forget this password.", true, 0, nil)
		if err := ks.Update(account, oldPassword, newPassword); err != nil {
			utils.Fatalf("Could not update the account: %v", err)
		}
	}
	return nil
}
```



```go 
// 导入钱包
func importWallet(ctx *cli.Context) error {
	keyfile := ctx.Args().First()
	if len(keyfile) == 0 {
		utils.Fatalf("keyfile must be given as argument")
	}
	keyJSON, err := ioutil.ReadFile(keyfile)
	if err != nil {
		utils.Fatalf("Could not read wallet file: %v", err)
	}

	stack, _ := makeConfigNode(ctx)
	passphrase := utils.GetPassPhraseWithList("", false, 0, utils.MakePasswordList(ctx))

	ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
	acct, err := ks.ImportPreSaleKey(keyJSON, passphrase)
	if err != nil {
		utils.Fatalf("%v", err)
	}
	fmt.Printf("Address: {%x}\n", acct.Address)
	return nil
}
```


```go
// 账户导入
func accountImport(ctx *cli.Context) error {
	keyfile := ctx.Args().First()
	if len(keyfile) == 0 {
		utils.Fatalf("keyfile must be given as argument")
	}
	key, err := crypto.LoadECDSA(keyfile)
	if err != nil {
		utils.Fatalf("Failed to load the private key: %v", err)
	}
	stack, _ := makeConfigNode(ctx)
	passphrase := utils.GetPassPhraseWithList("Your new account is locked with a password. Please give a password. Do not forget this password.", true, 0, utils.MakePasswordList(ctx))

	ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
	acct, err := ks.ImportECDSA(key, passphrase)
	if err != nil {
		utils.Fatalf("Could not create the account: %v", err)
	}
	fmt.Printf("Address: {%x}\n", acct.Address)
	return nil
}
```


### geth/consolecmd.go
```go

// 启动本地的一个交互式 Javascript 环境，功能是通过 console 模块提供的，而 console 模块使用了jsre模块, jsre是对 github.com/dop251/goja 的一个封装。goja 是一个 Golang 实现的 Javascript 解释器，可以实现在 Golang 中执行 Javascript，并且可以让在虚拟机里的 Javascript 调用 Golang 函数，实现 Golang 和 Javascript 的相互操作

// console/console.go
//
// Console is a JavaScript interpreted runtime environment. It is a fully fledged
// JavaScript console attached to a running node via an external or in-process RPC
// client.
type Console struct {
	client   *rpc.Client         // RPC client to execute Ethereum requests through
	jsre     *jsre.JSRE          // JavaScript runtime environment running the interpreter
	prompt   string              // Input prompt prefix string
	prompter prompt.UserPrompter // Input prompter to allow interactive user feedback
	histPath string              // Absolute path to the console scrollback history
	history  []string            // Scroll history maintained by the console
	printer  io.Writer           // Output writer to serialize any display strings to
}


// 启动一个geth 节点, 同时将javascript控制台连接到节点
// localConsole starts a new geth node, attaching a JavaScript console to it at the
// same time.
func localConsole(ctx *cli.Context) error {
	// Create and start the node based on the CLI flags
	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	startNode(ctx, stack, backend)
	defer stack.Close()

	// Attach to the newly started node and start the JavaScript console
	client, err := stack.Attach()
	if err != nil {
		utils.Fatalf("Failed to attach to the inproc geth: %v", err)
	}
	config := console.Config{
		DataDir: utils.MakeDataDir(ctx),
		DocRoot: ctx.GlobalString(utils.JSpathFlag.Name),
		Client:  client,
		Preload: utils.MakeConsolePreloads(ctx),
	}


	console, err := console.New(config)
	if err != nil {
		utils.Fatalf("Failed to start the JavaScript console: %v", err)
	}
	defer console.Stop(false)

	// If only a short execution was requested, evaluate and return
	if script := ctx.GlobalString(utils.ExecFlag.Name); script != "" {
		console.Evaluate(script)
		return nil
	}
	// Otherwise print the welcome screen and enter interactive mode
	console.Welcome()
	console.Interactive()

	return nil
}
```


```go
// 连接到远程geth, 连上 JavaScript控制台
//
// remoteConsole will connect to a remote geth instance, attaching a JavaScript
// console to it.
func remoteConsole(ctx *cli.Context) error {
	// Attach to a remotely running geth instance and start the JavaScript console
	endpoint := ctx.Args().First()
	if endpoint == "" {
		path := node.DefaultDataDir()
		if ctx.GlobalIsSet(utils.DataDirFlag.Name) {
			path = ctx.GlobalString(utils.DataDirFlag.Name)
		}
		if path != "" {
			if ctx.GlobalBool(utils.LegacyTestnetFlag.Name) || ctx.GlobalBool(utils.RopstenFlag.Name) {
				// Maintain compatibility with older Geth configurations storing the
				// Ropsten database in `testnet` instead of `ropsten`.
				legacyPath := filepath.Join(path, "testnet")
				if _, err := os.Stat(legacyPath); !os.IsNotExist(err) {
					path = legacyPath
				} else {
					path = filepath.Join(path, "ropsten")
				}
			} else if ctx.GlobalBool(utils.RinkebyFlag.Name) {
				path = filepath.Join(path, "rinkeby")
			} else if ctx.GlobalBool(utils.GoerliFlag.Name) {
				path = filepath.Join(path, "goerli")
			} else if ctx.GlobalBool(utils.YoloV2Flag.Name) {
				path = filepath.Join(path, "yolo-v2")
			}
		}
		endpoint = fmt.Sprintf("%s/geth.ipc", path)
	}
	client, err := dialRPC(endpoint)
	if err != nil {
		utils.Fatalf("Unable to attach to remote geth: %v", err)
	}
	config := console.Config{
		DataDir: utils.MakeDataDir(ctx),
		DocRoot: ctx.GlobalString(utils.JSpathFlag.Name),
		Client:  client,
		Preload: utils.MakeConsolePreloads(ctx),
	}

	console, err := console.New(config)
	if err != nil {
		utils.Fatalf("Failed to start the JavaScript console: %v", err)
	}
	defer console.Stop(false)

	if script := ctx.GlobalString(utils.ExecFlag.Name); script != "" {
		console.Evaluate(script)
		return nil
	}

	// Otherwise print the welcome screen and enter interactive mode
	console.Welcome()
	console.Interactive()

	return nil
}

```



```go

// 返回一个根据endpoint建立连接的RPC客户端

// dialRPC returns a RPC client which connects to the given endpoint.
// The check for empty endpoint implements the defaulting logic
// for "geth attach" and "geth monitor" with no argument.
func dialRPC(endpoint string) (*rpc.Client, error) {
	if endpoint == "" {
		endpoint = node.DefaultIPCEndpoint(clientIdentifier)
	} else if strings.HasPrefix(endpoint, "rpc:") || strings.HasPrefix(endpoint, "ipc:") {
		// Backwards compatibility with geth < 1.5 which required
		// these prefixes.
		endpoint = endpoint[4:]
	}
	return rpc.Dial(endpoint)
}
```


```go
// 新建一个临时的 geth 节点, 同时建立javascript控制台, 执行每个指定的文件

// ephemeralConsole starts a new geth node, attaches an ephemeral JavaScript
// console to it, executes each of the files specified as arguments and tears
// everything down.
func ephemeralConsole(ctx *cli.Context) error {
	// Create and start the node based on the CLI flags
	stack, backend := makeFullNode(ctx)
	startNode(ctx, stack, backend)
	defer stack.Close()

	// Attach to the newly started node and start the JavaScript console
	client, err := stack.Attach()
	if err != nil {
		utils.Fatalf("Failed to attach to the inproc geth: %v", err)
	}
	config := console.Config{
		DataDir: utils.MakeDataDir(ctx),
		DocRoot: ctx.GlobalString(utils.JSpathFlag.Name),
		Client:  client,
		Preload: utils.MakeConsolePreloads(ctx),
	}

	console, err := console.New(config)
	if err != nil {
		utils.Fatalf("Failed to start the JavaScript console: %v", err)
	}
	defer console.Stop(false)

	// Evaluate each of the specified JavaScript files
	for _, file := range ctx.Args() {
		if err = console.Execute(file); err != nil {
			utils.Fatalf("Failed to execute %s: %v", file, err)
		}
	}
	// Wait for pending callbacks, but stop for Ctrl-C.
	abort := make(chan os.Signal, 1)
	signal.Notify(abort, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		<-abort
		os.Exit(0)
	}()
	console.Stop(true)

	return nil
}

```


### geth/misccmd.go

```go

// makecache会在提供的文件夹中生成一个ethash验证缓存
// makecache generates an ethash verification cache into the provided folder.
func makecache(ctx *cli.Context) error {
	args := ctx.Args()
	if len(args) != 2 {
		utils.Fatalf(`Usage: geth makecache <block number> <outputdir>`)
	}
	block, err := strconv.ParseUint(args[0], 0, 64)
	if err != nil {
		utils.Fatalf("Invalid block number: %v", err)
	}
	ethash.MakeCache(block, args[1])

	return nil
}

```


```go

// makedag在提供的文件夹中生成一个ethash挖掘DAG
// makedag generates an ethash mining DAG into the provided folder.
func makedag(ctx *cli.Context) error {
	args := ctx.Args()
	if len(args) != 2 {
		utils.Fatalf(`Usage: geth makedag <block number> <outputdir>`)
	}
	block, err := strconv.ParseUint(args[0], 0, 64)
	if err != nil {
		utils.Fatalf("Invalid block number: %v", err)
	}
	ethash.MakeDataset(block, args[1])

	return nil
}
```



### geth/config.go

```go

// 导出配置
// dumpConfig is the dumpconfig command.
func dumpConfig(ctx *cli.Context) error {
	_, cfg := makeConfigNode(ctx)
	comment := ""

	if cfg.Eth.Genesis != nil {
		cfg.Eth.Genesis = nil
		comment += "# Note: this config doesn't contain the genesis block.\n\n"
	}

	out, err := tomlSettings.Marshal(&cfg)
	if err != nil {
		return err
	}

	dump := os.Stdout
	if ctx.NArg() > 0 {
		dump, err = os.OpenFile(ctx.Args().Get(0), os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0644)
		if err != nil {
			return err
		}
		defer dump.Close()
	}
	dump.WriteString(comment)
	dump.Write(out)

	return nil
}
```
