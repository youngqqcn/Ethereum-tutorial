## chain_indexer 区块链索引

chain_indexer.go 源码解析

chain_indexer 顾名思义， 就是用来给区块链创建索引的功能。 之前在eth协议的时候，介绍过BloomIndexer的功能，其实BloomIndexer是chain_indexer的一个特殊的实现， 可以理解为派生类， 主要的功能其实实在chain_indexer这里面实现的。虽说是派生类，但是chain_indexer其实就只被BloomIndexer使用。也就是给区块链的布隆过滤器创建了索引，以便快速的响应用户的日志搜索功能。 下面就来分析这部分的代码。



### 数据结构


```go
// ChainIndexerBackend defines the methods needed to process chain segments in
// the background and write the segment results into the database. These can be
// used to create filter blooms or CHTs.
// ChainIndexerBackend定义了处理区块链片段的方法，并把处理结果写入数据库。 这些可以用来创建布隆过滤器或者CHTs.
// BloomIndexer 其实就是实现了这个接口 ChainIndexerBackend 这里的CHTs不知道是什么东西。
type ChainIndexerBackend interface {
    // Reset initiates the processing of a new chain segment, potentially terminating
    // any partially completed operations (in case of a reorg).
    // Reset 方法用来初始化一个新的区块链片段，可能会终止任何没有完成的操作。
    Reset(section uint64)

    // Process crunches through the next header in the chain segment. The caller
    // will ensure a sequential order of headers.
    // 对区块链片段中的下一个区块头进行处理。 调用者将确保区块头的连续顺序。
    Process(header *types.Header)

    // Commit finalizes the section metadata and stores it into the database.
    // 完成区块链片段的元数据并将其存储到数据库中。	
    Commit() error

    // 删掉超过规定时间的老的index
    // Prune deletes the chain index older than the given threshold.
    Prune(threshold uint64) error
}

// ========================?????
// ChainIndexer does a post-processing job for equally sized sections of the
// canonical chain (like BlooomBits and CHT structures). A ChainIndexer is
// connected to the blockchain through the event system by starting a
// ChainHeadEventLoop in a goroutine.
//
// Further child ChainIndexers can be added which use the output of the parent
// section indexer. These child indexers receive new head notifications only
// after an entire section has been finished or in case of rollbacks that might
// affect already finished sections.
type ChainIndexer struct {
    chainDb  ethdb.Database      // Chain database to index the data from 区块链所在的数据库
    indexDb  ethdb.Database      // Prefixed table-view of the db to write index metadata into 索引存储的数据库
    backend  ChainIndexerBackend // Background processor generating the index data content  索引生成的后端。
    children []*ChainIndexer     // Child indexers to cascade chain updates to	子索引

    active uint32          // Flag whether the event loop was started
    update chan struct{}   // Notification channel that headers should be processed  接收到的headers
    quit   chan chan error // Quit channel to tear down running goroutines
    ctx       context.Context
	ctxCancel func()

    sectionSize uint64 // Number of blocks in a single chain segment to process	section的大小。 默认是4096个区块为一个section
    confirmsReq uint64 // Number of confirmations before processing a completed segment   处理完成的段之前的确认次数

    storedSections uint64 // Number of sections successfully indexed into the database  成功索引到数据库的部分数量
    knownSections  uint64 // Number of sections known to be complete (block wise) 已知完成的部分数量
    cascadedHead   uint64 // Block number of the last completed section cascaded to subindexers 级联到子索引的最后一个完成部分的块号

    checkpointSections uint64      // Number of sections covered by the checkpoint
	checkpointHead     common.Hash // Section head belonging to the checkpoint


    throttling time.Duration // Disk throttling to prevent a heavy upgrade from hogging resources 磁盘限制，以防止大量资源的大量升级

    log  log.Logger
    lock sync.RWMutex
}
```

首先需要了解一些定义，这部分代码中经常出现的 section 是指一组区块头，而这个一组的数量默认为 4096。

ChainIndexerBackend 是一个接口，它定义了处理区块链 section 的方法，这个接口目前有 BloomIndexer 这个实现。其中 Reset(section uint64) 用来初始化一个新的区块链 section，可能会终止任何没有完成的操作；Process(header *types.Header) 对区块链 section 中的下一个区块头进行处理，增加新区块头到 index，调用者需要确保区块头的连续顺序；Commit() error 完成区块链 section 的元数据提交，并将其存储到数据库。

以下是 ChainIndexer 结构体中较重要的一些属性：

|属性|	描述|
|---|---|
|chainDb|区块链所在的数据库|
|indexDb|	索引所在的数据库|
|backend|	生成索引的后端，它实现了 ChainIndexerBacken 所定义的接口，这里的实现我们只探讨 eth/bloombits 中的 BloomIndexer，在 light 模式中有其他实现|
|children|	子链的索引，这是为了处理临时分叉的情况|
|active|	事件循环是否开始的标志|
|update|	新生成区块头发送到这个 channel|
|quit	|退出事件循环的 channel|
|sectionSize|	索引器会一组一组处理区块头，默认的大小是 4096|
|confirmReq|	处理完成的 section 之前的确认次数|
|storedSections|	已经成功进行索引的 section 的数量|
|knownSections	|已知的 section 数量|
|cascadedHead	|级联到子索引最后一个完成的 section 的区块数|
|throttling	|对磁盘的限制，防止大量区块进行索引|


构造函数NewChainIndexer, 

```go
// 这个方法是在eth/bloombits.go里面被调用的
const (
    // bloomConfirms is the number of confirmation blocks before a bloom section is
    // considered probably final and its rotated bits are calculated.
    // bloomConfirms 用来表示确认区块数量， 表示经过这么多区块之后， bloom section被认为是已经不会更改了。
    bloomConfirms = 256

    // bloomThrottling is the time to wait between processing two consecutive index
    // sections. It's useful during chain upgrades to prevent disk overload.
    // bloomThrottling是处理两个连续索引段之间的等待时间。 在区块链升级过程中防止磁盘过载是很有用的。
    bloomThrottling = 100 * time.Millisecond
)

func NewBloomIndexer(db ethdb.Database, size uint64) *core.ChainIndexer {
    backend := &BloomIndexer{
        db:   db,
        size: size,
    }
    // 可以看到indexDb和chainDb实际是同一个数据库， 但是indexDb的每个key前面附加了一个BloomBitsIndexPrefix的前缀。
    table := ethdb.NewTable(db, string(core.BloomBitsIndexPrefix))

    return core.NewChainIndexer(db, table, backend, size, bloomConfirms, bloomThrottling, "bloombits")
}


// NewChainIndexer creates a new chain indexer to do background processing on
// chain segments of a given size after certain number of confirmations passed.
// The throttling parameter might be used to prevent database thrashing.

func NewChainIndexer(chainDb, indexDb ethdb.Database, backend ChainIndexerBackend, section, confirm uint64, throttling time.Duration, kind string) *ChainIndexer {
    c := &ChainIndexer{
        chainDb:     chainDb,
        indexDb:     indexDb,
        backend:     backend,
        update:      make(chan struct{}, 1),
        quit:        make(chan chan error),
        sectionSize: section,
        confirmsReq: confirm,
        throttling:  throttling,
        log:         log.New("type", kind),
    }
    // Initialize database dependent fields and start the updater
    c.loadValidSections()


    // 启动事件循环
    // updateLoop is the main event loop of the indexer which pushes chain segments
    // down into the processing backend.
    go c.updateLoop()

    return c
}
```

loadValidSections,用来从数据库里面加载我们之前的处理信息， storedSections表示我们已经处理到哪里了。

```go
// loadValidSections reads the number of valid sections from the index database
// and caches is into the local state.
func (c *ChainIndexer) loadValidSections() {
    data, _ := c.indexDb.Get([]byte("count"))
    if len(data) == 8 {
        c.storedSections = binary.BigEndian.Uint64(data[:])
    }
}
```





```go
// updateLoop,是主要的事件循环，用于调用backend来处理区块链section，这个需要注意的是，所有的主索引节点和所有的 child indexer 都会启动这个goroutine 方法。当已知的 section 数大于存储的 section 数，这时需要开始索引，先通过调用 SectionHead 拿到上一个 section 的最后一个区块的哈希值，接着调用 processSection 开始新 section 的索引。

// updateLoop is the main event loop of the indexer which pushes chain segments
// down into the processing backend.
func (c *ChainIndexer) updateLoop() {
	var (
		updating bool
		updated  time.Time
	)

	for {
		select {
		case errc := <-c.quit:
			// Chain indexer terminating, report no failure and abort
			errc <- nil
			return

		case <-c.update: //当需要使用backend处理的时候，其他goroutine会往这个channel上面发送消息
			// Section headers completed (or rolled back), update the index
			c.lock.Lock()
			if c.knownSections > c.storedSections { // 如果当前以知的Section 大于已经存储的Section
				// Periodically print an upgrade log message to the user
				if time.Since(updated) > 8*time.Second {
					if c.knownSections > c.storedSections+1 {  
						updating = true
						c.log.Info("Upgrading chain index", "percentage", c.storedSections*100/c.knownSections)
					}
					updated = time.Now()
				}
				// Cache the current section count and head to allow unlocking the mutex
				c.verifyLastHead()
				section := c.storedSections
				var oldHead common.Hash
				if section > 0 {
					oldHead = c.SectionHead(section - 1) // section - 1 代表section的下标是从0开始的。 
				}
				// Process the newly defined section in the background
				c.lock.Unlock()
                // 处理 返回新的section的最后一个区块的hash值
				newHead, err := c.processSection(section, oldHead)
				if err != nil {
					select {
					case <-c.ctx.Done():
						<-c.quit <- nil
						return
					default:
					}
					c.log.Error("Section processing failed", "error", err)
				}
				c.lock.Lock()

				// If processing succeeded and no reorgs occurred, mark the section completed
				if err == nil && (section == 0 || oldHead == c.SectionHead(section-1)) {
					c.setSectionHead(section, newHead)
					c.setValidSections(section + 1)
					if c.storedSections == c.knownSections && updating {
						updating = false
						c.log.Info("Finished upgrading chain index")
					}
					c.cascadedHead = c.storedSections*c.sectionSize - 1
					for _, child := range c.children {
						c.log.Trace("Cascading chain index update", "head", c.cascadedHead)
						child.newHead(c.cascadedHead, false)
					}
				} else { //如果处理失败，那么在有新的通知之前不会重试。
					// If processing failed, don't retry until further notification
					c.log.Debug("Chain index processing failed", "section", section, "err", err)
					c.verifyLastHead()
					c.knownSections = c.storedSections
				}
			}
			// If there are still further sections to process, reschedule
			if c.knownSections > c.storedSections {
				time.AfterFunc(c.throttling, func() {
					select {
					case c.update <- struct{}{}:
					default:
					}
				})
			}
			c.lock.Unlock()
		}
	}
}
```

Start方法。 这个方法在eth协议启动的时候被调用,这个方法接收两个参数，一个是当前的区块头，一个是事件订阅器，通过这个订阅器可以获取区块链的改变信息。


```go
// Start creates a goroutine to feed chain head events into the indexer for
// cascading background processing. Children do not need to be started, they
// are notified about new events by their parents.
func (c *ChainIndexer) Start(chain ChainIndexerChain) {
    events := make(chan ChainHeadEvent, 10)
    sub := chain.SubscribeChainHeadEvent(events)

    go c.eventLoop(chain.CurrentHeader(), events, sub)
}

// eventLoop is a secondary - optional - event loop of the indexer which is only
// started for the outermost indexer to push chain head events into a processing
// queue.
// eventLoop 循环只会在最外面的索引节点被调用。 所有的Child indexer不会被启动这个方法。 
//
// eventLoop is a secondary - optional - event loop of the indexer which is only
// started for the outermost indexer to push chain head events into a processing
// queue.
func (c *ChainIndexer) eventLoop(currentHeader *types.Header, events chan ChainHeadEvent, sub event.Subscription) {
	// Mark the chain indexer as active, requiring an additional teardown
	atomic.StoreUint32(&c.active, 1)

	defer sub.Unsubscribe()

	// Fire the initial new head event to start any outstanding processing
	c.newHead(currentHeader.Number.Uint64(), false)

	var (
		prevHeader = currentHeader
		prevHash   = currentHeader.Hash()
	)
	for {
		select {
		case errc := <-c.quit:
			// Chain indexer terminating, report no failure and abort
			errc <- nil
			return

		case ev, ok := <-events:
			// Received a new event, ensure it's not nil (closing) and update
			if !ok {
				errc := <-c.quit
				errc <- nil
				return
			}
			header := ev.Block.Header()
			if header.ParentHash != prevHash {
				// Reorg to the common ancestor if needed (might not exist in light sync mode, skip reorg then)
				// TODO(karalabe, zsfelfoldi): This seems a bit brittle, can we detect this case explicitly?

				if rawdb.ReadCanonicalHash(c.chainDb, prevHeader.Number.Uint64()) != prevHash {
					if h := rawdb.FindCommonAncestor(c.chainDb, prevHeader, header); h != nil {
						c.newHead(h.Number.Uint64(), true)
					}
				}
			}
			c.newHead(header.Number.Uint64(), false)

			prevHeader, prevHash = header, header.Hash()
		}
	}
}

```

newHead方法,通知indexer新的区块链头，或者是需要重建索引，newHead方法会触发

```go
// newHead通知索引器有关新链头和/或重组的信息。
// newHead notifies the indexer about new chain heads and/or reorgs.
func (c *ChainIndexer) newHead(head uint64, reorg bool) {
	c.lock.Lock()
	defer c.lock.Unlock()

    //如果发生重组，请在此之前使所有部分无效
	// If a reorg happened, invalidate all sections until that point
	if reorg {
		// Revert the known section number to the reorg point
		known := (head + 1) / c.sectionSize
		stored := known
		if known < c.checkpointSections {
			known = 0
		}
		if stored < c.checkpointSections {
			stored = c.checkpointSections
		}
		if known < c.knownSections {
			c.knownSections = known
		}
		// Revert the stored sections from the database to the reorg point
		if stored < c.storedSections {
			c.setValidSections(stored)
		}
		// Update the new head number to the finalized section end and notify children
		head = known * c.sectionSize

		if head < c.cascadedHead {
			c.cascadedHead = head
			for _, child := range c.children {
				child.newHead(c.cascadedHead, true)
			}
		}
		return
	}

    // 没有重组，计算新已知部分的数量，并在足够高的情况下进行更新
	// No reorg, calculate the number of newly known sections and update if high enough
	var sections uint64
	if head >= c.confirmsReq {
		sections = (head + 1 - c.confirmsReq) / c.sectionSize
		if sections < c.checkpointSections {
			sections = 0
		}
		if sections > c.knownSections {
			if c.knownSections < c.checkpointSections {
				// syncing reached the checkpoint, verify section head
				syncedHead := rawdb.ReadCanonicalHash(c.chainDb, c.checkpointSections*c.sectionSize-1)
				if syncedHead != c.checkpointHead {
					c.log.Error("Synced chain does not match checkpoint", "number", c.checkpointSections*c.sectionSize-1, "expected", c.checkpointHead, "synced", syncedHead)
					return
				}
			}
			c.knownSections = sections

			select {
			case c.update <- struct{}{}:
			default:
			}
		}
	}
}

```

父子索引数据的关系
父Indexer负责事件的监听,然后把结果通过newHead传递给子Indexer的updateLoop来处理。

![image](picture/chainindexer_1.png)

setValidSections方法，写入当前已经存储的sections的数量。 如果传入的值小于已经存储的数量，那么从数据库里面删除对应的section

```go
// setValidSections writes the number of valid sections to the index database
func (c *ChainIndexer) setValidSections(sections uint64) {
    // Set the current number of valid sections in the database
    var data [8]byte
    binary.BigEndian.PutUint64(data[:], sections)
    c.indexDb.Put([]byte("count"), data[:])

    // Remove any reorged sections, caching the valids in the mean time
    for c.storedSections > sections {
        c.storedSections--
        c.removeSectionHead(c.storedSections)
    }
    c.storedSections = sections // needed if new > old
}
```

processSection


```go
// processSection processes an entire section by calling backend functions while
// ensuring the continuity of the passed headers. Since the chain mutex is not
// held while processing, the continuity can be broken by a long reorg, in which
// case the function returns with an error.

//processSection通过调用后端函数来处理整个部分，同时确保传递的头文件的连续性。 由于链接互斥锁在处理过程中没有保持，连续性可能会被重新打断，在这种情况下，函数返回一个错误。
func (c *ChainIndexer) processSection(section uint64, lastHead common.Hash) (common.Hash, error) {
    c.log.Trace("Processing new chain section", "section", section)

    // Reset and partial processing
    c.backend.Reset(section)

    for number := section * c.sectionSize; number < (section+1)*c.sectionSize; number++ {
        hash := GetCanonicalHash(c.chainDb, number)
        if hash == (common.Hash{}) {
            return common.Hash{}, fmt.Errorf("canonical block #%d unknown", number)
        }
        header := GetHeader(c.chainDb, hash, number)
        if header == nil {
            return common.Hash{}, fmt.Errorf("block #%d [%x…] not found", number, hash[:4])
        } else if header.ParentHash != lastHead {
            return common.Hash{}, fmt.Errorf("chain reorged during section processing")
        }
        c.backend.Process(header)
        lastHead = header.Hash()
    }
    if err := c.backend.Commit(); err != nil {
        c.log.Error("Section commit failed", "error", err)
        return common.Hash{}, err
    }
    return lastHead, nil
}
```