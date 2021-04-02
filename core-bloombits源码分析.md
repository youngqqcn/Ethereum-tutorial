
此部分可以和 [eth-bloombits和filter源码分析.md](eth-bloombits和filter源码分析.md)一起看


# scheduler.go

scheduler是基于section的布隆过滤器的单个bit值检索的调度。 除了调度检索操作之外，这个结构还可以对请求进行重复数据删除并缓存结果，从而即使在复杂的过滤情况下也可以将网络/数据库开销降至最低

## 数据结构
request表示一个bloom检索任务，以便优先从本地数据库中或从网络中剪检索。 section 表示区块段号，每段4096个区块， bit代表检索的是布隆过滤器的哪一位(一共有2048位)。这个在之前的(eth-bloombits和filter源码分析.md)中有介绍。


```go

// request represents a bloom retrieval task to prioritize and pull from the local
// database or remotely from the network.
type request struct {
	section uint64 // Section index to retrieve the a bit-vector from
	bit     uint   // Bit index within the section to retrieve the vector of
}
```

response当前调度的请求的状态。 没发送一个请求，会生成一个response对象来最终这个请求的状态。
cached用来缓存这个section的结果。 

```go
// response represents the state of a requested bit-vector through a scheduler.
type response struct {
	cached []byte        // Cached bits to dedup multiple requests
	done   chan struct{} // Channel to allow waiting for completion
}

```

scheduler

```go
//调度程序处理布鲁姆过滤器检索操作的调度
//属于单个bloom位的整个段批处理。除了安排
//检索操作，此结构还会对请求和缓存进行重复数据删除
//结果，即使在复杂的过滤中也可以最大程度地减少网络/数据库的开销
//场景。
// scheduler handles the scheduling of bloom-filter retrieval operations for
// entire section-batches belonging to a single bloom bit. Beside scheduling the
// retrieval operations, this struct also deduplicates the requests and caches
// the results to minimize network/database overhead even in complex filtering
// scenarios.
type scheduler struct {
	bit       uint                 // Index of the bit in the bloom filter this scheduler is responsible for
	responses map[uint64]*response // Currently pending retrieval requests or already cached responses
	lock      sync.Mutex           // Lock protecting the responses from concurrent access
}

```

## 构造函数
newScheduler和reset方法


```go
// newScheduler creates a new bloom-filter retrieval scheduler for a specific
// bit index.
func newScheduler(idx uint) *scheduler {
	return &scheduler{
		bit:       idx,
		responses: make(map[uint64]*response),
	}
}

// reset cleans up any leftovers from previous runs. This is required before a
// restart to ensure the no previously requested but never delivered state will
// cause a lockup.
func (s *scheduler) reset() {
	s.lock.Lock()
	defer s.lock.Unlock()

	for section, res := range s.responses {
		if res.cached == nil {
			delete(s.responses, section)
		}
	}
}

```

## 运行 run方法
run方法创建了一个流水线， 从sections channel来接收需要请求的sections，通过done channel来按照请求的顺序返回结果。 并发的运行同样的scheduler是可以的，这样会导致任务重复。


```go
// run creates a retrieval pipeline, receiving section indexes from sections and
// returning the results in the same order through the done channel. Concurrent
// runs of the same scheduler are allowed, leading to retrieval task deduplication.
func (s *scheduler) run(sections chan uint64, dist chan *request, done chan []byte, quit chan struct{}, wg *sync.WaitGroup) {
	// Create a forwarder channel between requests and responses of the same size as
	// the distribution channel (since that will block the pipeline anyway).
	pend := make(chan uint64, cap(dist))

	// Start the pipeline schedulers to forward between user -> distributor -> user
	wg.Add(2)
	go s.scheduleRequests(sections, dist, pend, quit, wg)
	go s.scheduleDeliveries(pend, done, quit, wg)
}
```

## scheduler的流程图

![image](picture/chainindexer_2.png)
图中椭圆代表了goroutine. 矩形代表了channel. 三角形代表外部的方法调用。

1. scheduleRequests goroutine从sections接收到section消息
2. scheduleRequests把接收到的section组装成requtest发送到dist channel，并构建对象response[section]
3. scheduleRequests把上一部的section发送给pend队列。scheduleDelivers接收到pend消息，阻塞在response[section].done上面
4. 外部调用deliver方法，把seciton的request请求结果写入response[section].cached.并关闭response[section].done channel
5. scheduleDelivers接收到response[section].done 信息。 把response[section].cached 发送到done channel

## scheduleRequests
	
```go

// scheduleRequests reads section retrieval requests from the input channel,
// deduplicates the stream and pushes unique retrieval tasks into the distribution
// channel for a database or network layer to honour.
func (s *scheduler) scheduleRequests(reqs chan uint64, dist chan *request, pend chan uint64, quit chan struct{}, wg *sync.WaitGroup) {
	// Clean up the goroutine and pipeline when done
	defer wg.Done()
	defer close(pend)

	// Keep reading and scheduling section requests
	for {
		select {
		case <-quit:
			return

		case section, ok := <-reqs:
			// New section retrieval requested
			if !ok {
				return
			}
			// Deduplicate retrieval requests
			unique := false

			s.lock.Lock()
			if s.responses[section] == nil {
				s.responses[section] = &response{
					done: make(chan struct{}),
				}
				unique = true
			}
			s.lock.Unlock()

			// Schedule the section for retrieval and notify the deliverer to expect this section
			if unique {
				select {
				case <-quit:
					return
				case dist <- &request{bit: s.bit, section: section}:
				}
			}
			select {
			case <-quit:
				return
			case pend <- section:
			}
		}
	}
}

```


### generator.go
generator用来产生基于section的布隆过滤器索引数据的对象。 generator内部主要的数据结构是 bloom[2048][4096]bit 的数据结构。  输入是4096个header.logBloom数据。  比如第20个header的logBloom存储在  bloom[0:2048][20]

数据结构：

```go

// Generator takes a number of bloom filters and generates the rotated bloom bits
// to be used for batched filtering.
type Generator struct {
	blooms   [types.BloomBitLength][]byte // Rotated blooms for per-bit matching
	sections uint                         // Number of sections to batch together
	nextSec  uint                         // Next section to set when adding a bloom
}

```

构造：


```go
// NewGenerator creates a rotated bloom generator that can iteratively fill a
// batched bloom filter's bits.
func NewGenerator(sections uint) (*Generator, error) {
	if sections%8 != 0 {
		return nil, errors.New("section count not multiple of 8")
	}
	b := &Generator{sections: sections}
	for i := 0; i < types.BloomBitLength; i++ {
		b.blooms[i] = make([]byte, sections/8)
	}
	return b, nil
}
```

AddBloom增加一个区块头的logsBloom


```go

// AddBloom takes a single bloom filter and sets the corresponding bit column
// in memory accordingly.
func (b *Generator) AddBloom(index uint, bloom types.Bloom) error {
	// Make sure we're not adding more bloom filters than our capacity
	if b.nextSec >= b.sections {
		return errSectionOutOfBounds
	}
	if b.nextSec != index {
		return errors.New("bloom filter with unexpected index")
	}
	// Rotate the bloom and insert into our collection
	byteIndex := b.nextSec / 8
	bitIndex := byte(7 - b.nextSec%8)
	for byt := 0; byt < types.BloomByteLength; byt++ {
		bloomByte := bloom[types.BloomByteLength-1-byt]
		if bloomByte == 0 {
			continue
		}
		base := 8 * byt
		b.blooms[base+7][byteIndex] |= ((bloomByte >> 7) & 1) << bitIndex
		b.blooms[base+6][byteIndex] |= ((bloomByte >> 6) & 1) << bitIndex
		b.blooms[base+5][byteIndex] |= ((bloomByte >> 5) & 1) << bitIndex
		b.blooms[base+4][byteIndex] |= ((bloomByte >> 4) & 1) << bitIndex
		b.blooms[base+3][byteIndex] |= ((bloomByte >> 3) & 1) << bitIndex
		b.blooms[base+2][byteIndex] |= ((bloomByte >> 2) & 1) << bitIndex
		b.blooms[base+1][byteIndex] |= ((bloomByte >> 1) & 1) << bitIndex
		b.blooms[base][byteIndex] |= (bloomByte & 1) << bitIndex
	}
	b.nextSec++
	return nil
}

```

Bitset返回

```go
// Bitset returns the bit vector belonging to the given bit index after all
// blooms have been added.
func (b *Generator) Bitset(idx uint) ([]byte, error) {
	if b.nextSec != b.sections {
		return nil, errors.New("bloom not fully generated yet")
	}
	if idx >= types.BloomBitLength {
		return nil, errBloomBitOutOfBounds
	}
	return b.blooms[idx], nil
}

```
	

### matcher.go
Matcher是一个流水线系统的调度器和逻辑匹配器，它们对比特流执行二进制与/或操作，创建一个潜在块的流来检查数据内容。

数据结构

```go
// partialMatches with a non-nil vector represents a section in which some sub-
// matchers have already found potential matches. Subsequent sub-matchers will
// binary AND their matches with this vector. If vector is nil, it represents a
// section to be processed by the first sub-matcher.
type partialMatches struct {
	section uint64
	bitset  []byte
}
	

// Retrieval represents a request for retrieval task assignments for a given
// bit with the given number of fetch elements, or a response for such a request.
// It can also have the actual results set to be used as a delivery data struct.
//
// The contest and error fields are used by the light client to terminate matching
// early if an error is encountered on some path of the pipeline.
type Retrieval struct {
	Bit      uint
	Sections []uint64
	Bitsets  [][]byte

	Context context.Context
	Error   error
}


// Matcher is a pipelined system of schedulers and logic matchers which perform
// binary AND/OR operations on the bit-streams, creating a stream of potential
// blocks to inspect for data content.
type Matcher struct {
	sectionSize uint64 // Size of the data batches to filter on

	filters    [][]bloomIndexes    // Filter the system is matching for
	schedulers map[uint]*scheduler // Retrieval schedulers for loading bloom bits

	retrievers chan chan uint       // Retriever processes waiting for bit allocations
	counters   chan chan uint       // Retriever processes waiting for task count reports
	retrievals chan chan *Retrieval // Retriever processes waiting for task allocations
	deliveries chan *Retrieval      // Retriever processes waiting for task response deliveries

	running uint32 // Atomic flag whether a session is live or not
}
```

partialMatches 表示部分匹配的结果，Retrieval 表示一次区块布隆过滤器的检索工作，在使用过程中，该对象会被发送给 eth/bloombits.go 中的 startBloomHandlers 来处理，该方法从数据库中加载布隆过滤器索引，然后放在 Bitsets 里返回（待确认）。Matcher 是一个操作调度器（scheduler）和匹配器（matcher）的流水线系统，它会对比特流进行二进制的与/或操作，对数据内容进行检索，创建可能的区块。

matcher的大体流程图片，途中椭圆代表goroutine. 矩形代表channel。 三角形代表方法调用。

![image](picture/matcher_1.png)

1. 首先Matcher根据传入的filter的个数 创建了对应个数的 subMatch 。 每一个subMatch对应了一个filter对象。 每一个subMatch会把自己的查找结果和上一个查找结果按照位与的方式得到新的结果。 如果新的结果所有的bit位都有置位，就会把这个查找结果传递给下一个。 这是实现对所有的filter的结果求与的短路算法。  如果前面的计算已经不能匹配任何东西，那么就不用进行下面的条件的匹配了。
2. Matcher会根据fiters的布隆过滤器的组合下标的个数来启动对应个数的schedule。
3. subMatch会把请求发送给对应的schedule。
4. schedule会把请求调度后通过dist发送给distributor， 在distributor中管理起来。
5. 会启动多个(16)Multiplex线程，从distributor中获取请求，然后把请求发送给bloomRequests队列, startBloomHandlers会访问数据库，拿到数据然后返回给Multiplex。
6. Multiplex通过deliveries通道把回答告诉distributor。
7. distributor调用schedule的deliver方法，把结果发送给schedule
8. schedule把结果返回给subMatch。
9. subMatch把结果进行计算后发送给下一个subMatch进行处理。如果是最后一个subMatch，那么结果会进行处理后发送给results通道。


matcher

	filter := New(backend, 0, -1, []common.Address{addr}, [][]common.Hash{{hash1, hash2, hash3, hash4}}) 
	组间是与的关系 组内是或的关系。  (addr && hash1) ||(addr && hash2)||(addr && hash3)||(addr && hash4)
	

构造函数， 需要特别注意的是输入的filters这个参数。 这个参数是一个三维度的数组  [][]bloomIndexes === [第一维度][第二维度][3] 。 

	// 这个是filter.go里面的代码，对于理解filters这个参数比较有用。 filter.go是Matcher的调用者。 
	
	// 可以看到无论有多少个addresses，在filters里面也只占一个位置。 filters[0]=addresses
	// filters[1] = topics[0] = 多个topic
	// filters[2] = topics[1] = 多个topic
	// filters[n] = topics[n] = 多个topic

	// filter 的参数addresses 和 topics 的过滤算法是， (含有addresses中任意一个address) 并且 (含有topics[0]里面的任意一个topic) 并且 (含有topics[1]里面任意一个topic) 并且 (含有topics[n]里面的任意一个topic)

	// 可以看到 对于filter 实行的是  对第一维的数据 执行 与操作， 对于第二维度的数据， 执行或操作。
	
	// 而在NewMatcher方法中，把第三维的具体数据转换成 布隆过滤器的指定三个位置。 所以在filter.go里面的var filters [][][]byte 在Matcher里面的filters变成了 [][][3]
	
```go
// NewMatcher creates a new pipeline for retrieving bloom bit streams and doing
// address and topic filtering on them. Setting a filter component to `nil` is
// allowed and will result in that filter rule being skipped (OR 0x11...1).
func NewMatcher(sectionSize uint64, filters [][][]byte) *Matcher {
    // Create the matcher instance
    m := &Matcher{
        sectionSize: sectionSize,
        schedulers:  make(map[uint]*scheduler),
        retrievers:  make(chan chan uint),
        counters:    make(chan chan uint),
        retrievals:  make(chan chan *Retrieval),
        deliveries:  make(chan *Retrieval),
    }
    // Calculate the bloom bit indexes for the groups we're interested in
    m.filters = nil

    for _, filter := range filters {
        // Gather the bit indexes of the filter rule, special casing the nil filter
        if len(filter) == 0 {
            continue
        }
        bloomBits := make([]bloomIndexes, len(filter))
        for i, clause := range filter {
            if clause == nil {
                bloomBits = nil
                break
            }
            // clause 对应了输入的第三维度的数据，可能是一个address或者是一个topic
            // calcBloomIndexes计算了这个数据对应的(0-2048)的布隆过滤器中的三个下标， 就是说如果在布隆过滤器中对应的三位都为1，那么clause这个数据就有可能在这里。
            bloomBits[i] = calcBloomIndexes(clause)
        }
        // Accumulate the filter rules if no nil rule was within
        // 在计算中 如果bloomBits中只要其中的一条能够找到。那么就认为整个成立。
        if bloomBits != nil {
            // 不同的bloomBits 需要同时成立，整个结果才能成立。
            m.filters = append(m.filters, bloomBits)
        }
    }
    // For every bit, create a scheduler to load/download the bit vectors
    for _, bloomIndexLists := range m.filters {
        for _, bloomIndexList := range bloomIndexLists {
            for _, bloomIndex := range bloomIndexList {
                // 对于所有可能出现的下标。 我们都生成一个scheduler来进行对应位置的
                // 布隆过滤数据的检索。
                m.addScheduler(bloomIndex)
            }
        }
    }
    return m
}
```


Start 启动

Start 方法首先启动一个 session，这个 session 会被返回，它可以用来管理日志过滤的生命周期，调用者会将它作为 ServiceFilter 的参数，根据 bloomFilterThreads 这个常数值（默认为3），启动 bloomFilterThreads 个 session 的 Multiplex，该方法会不断地从 distributor 领取任务，将任务投递给 bloomRequest 队列，从队列中获取结果，然后投递给 distributor，这个 Multiplex 非常重要。

```go

// Start starts the matching process and returns a stream of bloom matches in
// a given range of blocks. If there are no more matches in the range, the result
// channel is closed.
func (m *Matcher) Start(ctx context.Context, begin, end uint64, results chan uint64) (*MatcherSession, error) {
	// Make sure we're not creating concurrent sessions
	if atomic.SwapUint32(&m.running, 1) == 1 {
		return nil, errors.New("matcher already running")
	}
	defer atomic.StoreUint32(&m.running, 0)

    // 启动了一个session，作为返回值，管理查找的生命周期。
	// Initiate a new matching round
	session := &MatcherSession{
		matcher: m,
		quit:    make(chan struct{}),
		ctx:     ctx,
	}
	for _, scheduler := range m.schedulers {
		scheduler.reset()
	}

    // 这个运行会建立起流程，返回了一个partialMatches类型的管道表示查询的部分结果。
	sink := m.run(begin, end, cap(results), session)

	// Read the output from the result sink and deliver to the user
	session.pend.Add(1)
	go func() {
		defer session.pend.Done()
		defer close(results)

		for {
			select {
			case <-session.quit:
				return

			case res, ok := <-sink:
                // New match result found
				// 找到返回结果 因为返回值是 section和 section中哪些区块可能有值的bitmap
				// 所以需要遍历这个bitmap，找到那些被置位的区块，把区块号返回回去。
				// New match result found
				if !ok {
					return
				}
				// Calculate the first and last blocks of the section
				sectionStart := res.section * m.sectionSize

				first := sectionStart
				if begin > first {
					first = begin
				}
				last := sectionStart + m.sectionSize - 1
				if end < last {
					last = end
				}
				// Iterate over all the blocks in the section and return the matching ones
				for i := first; i <= last; i++ {
					// Skip the entire byte if no matches are found inside (and we're processing an entire byte!)
					next := res.bitset[(i-sectionStart)/8]
					if next == 0 {
						if i%8 == 0 {
							i += 7
						}
						continue
					}
					// Some bit it set, do the actual submatching
					if bit := 7 - i%8; next&(1<<bit) != 0 {
						select {
						case <-session.quit:
							return
						case results <- i:
						}
					}
				}
			}
		}
	}()
	return session, nil
}
```




接下来会调用 run 方法，该方法会返回一个 channel，该 channel 会一直返回搜索的结果，直到返回一个退出的信号，Start 方法才会结束，对于过滤的结果，其中包括 section 和 bitmap，bitmap 表明了 section 中哪些区块可能存在值，这时需要遍历这个 bitmap，找到被置位的区块，然后把区块号返回到 results 通道

run方法

```go
// run creates a daisy-chain of sub-matchers, one for the address set and one
// for each topic set, each sub-matcher receiving a section only if the previous
// ones have all found a potential match in one of the blocks of the section,
// then binary AND-ing its own matches and forwarding the result to the next one.
//
// The method starts feeding the section indexes into the first sub-matcher on a
// new goroutine and returns a sink channel receiving the results.
func (m *Matcher) run(begin, end uint64, buffer int, session *MatcherSession) chan *partialMatches {
	// Create the source channel and feed section indexes into
	source := make(chan *partialMatches, buffer)

	session.pend.Add(1)
	go func() {
		defer session.pend.Done()
		defer close(source)

		for i := begin / m.sectionSize; i <= end/m.sectionSize; i++ {
            // 这个for循环 构造了subMatch的第一个输入源，剩下的subMatch把上一个的结果作为自己的源
			// 这个源的bitset字段都是0xff，代表完全的匹配，它将和我们这一步的匹配进行与操作，得到这一步匹配的结果。
			select {
			case <-session.quit:
				return
			case source <- &partialMatches{i, bytes.Repeat([]byte{0xff}, int(m.sectionSize/8))}:
			}
		}
	}()
	// Assemble the daisy-chained filtering pipeline
	next := source
	dist := make(chan *request, buffer)

	for _, bloom := range m.filters {
		next = m.subMatch(next, dist, bloom, session)
	}
	// Start the request distribution
	session.pend.Add(1)
	go m.distributor(dist, session)

	return next
}
```


run 方法会创建一个子匹配器的流水线，一个用于地址集合，一个用于 topic 集合。之所以称为流水线是因为它会一个一个地调用子匹配器，之前的子匹配器找到了匹配的区块后才会调用下一个子匹配器，接收到的结果会与自身结果匹配后结合，发到下一个子匹配器。最终返回一个接收结果的接收器通道。该方法首先起一个 go routine, 构造 subMatch 的第一个输入源，这个源的 bitset 字段是 0xff，表示完全匹配，这个结果会作为第一个子匹配器的输入。在结尾还会用新线程的方式调用 distributor，

subMatch函数

```go
// subMatch creates a sub-matcher that filters for a set of addresses or topics, binary OR-s those matches, then
// binary AND-s the result to the daisy-chain input (source) and forwards it to the daisy-chain output.
// The matches of each address/topic are calculated by fetching the given sections of the three bloom bit indexes belonging to
// that address/topic, and binary AND-ing those vectors together.
func (m *Matcher) subMatch(source chan *partialMatches, dist chan *request, bloom []bloomIndexes, session *MatcherSession) chan *partialMatches {
	// Start the concurrent schedulers for each bit required by the bloom filter
	sectionSources := make([][3]chan uint64, len(bloom))
	sectionSinks := make([][3]chan []byte, len(bloom))
	for i, bits := range bloom {
		for j, bit := range bits {
			sectionSources[i][j] = make(chan uint64, cap(source))
			sectionSinks[i][j] = make(chan []byte, cap(source))

			m.schedulers[bit].run(sectionSources[i][j], dist, sectionSinks[i][j], session.quit, &session.pend)
		}
	}

	process := make(chan *partialMatches, cap(source)) // entries from source are forwarded here after fetches have been initiated
	results := make(chan *partialMatches, cap(source))

	session.pend.Add(2)
	go func() {
		// Tear down the goroutine and terminate all source channels
		defer session.pend.Done()
		defer close(process)

		defer func() {
			for _, bloomSources := range sectionSources {
				for _, bitSource := range bloomSources {
					close(bitSource)
				}
			}
		}()
		// Read sections from the source channel and multiplex into all bit-schedulers
		for {
			select {
			case <-session.quit:
				return

			case subres, ok := <-source:
				// New subresult from previous link
				if !ok {
					return
				}
				// Multiplex the section index to all bit-schedulers
				for _, bloomSources := range sectionSources {
					for _, bitSource := range bloomSources {
						select {
						case <-session.quit:
							return
						case bitSource <- subres.section:
						}
					}
				}
				// Notify the processor that this section will become available
				select {
				case <-session.quit:
					return
				case process <- subres:
				}
			}
		}
	}()

	go func() {
		// Tear down the goroutine and terminate the final sink channel
		defer session.pend.Done()
		defer close(results)

		// Read the source notifications and collect the delivered results
		for {
			select {
			case <-session.quit:
				return

			case subres, ok := <-process:
				// Notified of a section being retrieved
				if !ok {
					return
				}
				// Gather all the sub-results and merge them together
				var orVector []byte
				for _, bloomSinks := range sectionSinks {
					var andVector []byte
					for _, bitSink := range bloomSinks {
						var data []byte
						select {
						case <-session.quit:
							return
						case data = <-bitSink:
						}
						if andVector == nil {
							andVector = make([]byte, int(m.sectionSize/8))
							copy(andVector, data)
						} else {
							bitutil.ANDBytes(andVector, andVector, data)
						}
					}
					if orVector == nil {
						orVector = andVector
					} else {
						bitutil.ORBytes(orVector, orVector, andVector)
					}
				}

				if orVector == nil {
					orVector = make([]byte, int(m.sectionSize/8))
				}
				if subres.bitset != nil {
					bitutil.ANDBytes(orVector, orVector, subres.bitset)
				}
				if bitutil.TestBytes(orVector) {
					select {
					case <-session.quit:
						return
					case results <- &partialMatches{subres.section, orVector}:
					}
				}
			}
		}
	}()
	return results
}
```


subMatch 会创建一个子匹配器，用于过滤一组 address 或 topic，在组内会进行 bit 位的或操作，然后将上一个结果与当前过滤结果进行位的与操作，如果结果不全为空，结果传递到下一个子匹配器。每个 address/topic 的匹配通过获取属于该 address/topics 的三个布隆过滤器位索引的给定部分以及这些向量二进制做与的运算。

注意这里的 bloom 参数的类型是 []bloomIndexes，首先根据这个值创建相应个数的 schedulers，调用其对应的 run 方法。前面我们有介绍 schedulers 的 run 方法，它的作用是根据后端的实现（从硬盘或网络中）执行过滤操作，结果可以通过调用 scheduler 的 deliver 获得。

distributor,接受来自scheduler的请求，并把他们放到一个set里面。 然后把这些任务指派给retrievers来填充他们。


```go
// distributor receives requests from the schedulers and queues them into a set
// of pending requests, which are assigned to retrievers wanting to fulfil them.
func (m *Matcher) distributor(dist chan *request, session *MatcherSession) {
	defer session.pend.Done()

	var (
		requests   = make(map[uint][]uint64) // Per-bit list of section requests, ordered by section number
		unallocs   = make(map[uint]struct{}) // Bits with pending requests but not allocated to any retriever
		retrievers chan chan uint            // Waiting retrievers (toggled to nil if unallocs is empty)
		allocs     int                       // Number of active allocations to handle graceful shutdown requests
		shutdown   = session.quit            // Shutdown request channel, will gracefully wait for pending requests
	)

	// assign is a helper method fo try to assign a pending bit an actively
	// listening servicer, or schedule it up for later when one arrives.
	assign := func(bit uint) {
		select {
		case fetcher := <-m.retrievers:
			allocs++
			fetcher <- bit
		default:
			// No retrievers active, start listening for new ones
			retrievers = m.retrievers
			unallocs[bit] = struct{}{}
		}
	}

	for {
		select {
		case <-shutdown:
			// Shutdown requested. No more retrievers can be allocated,
			// but we still need to wait until all pending requests have returned.
			shutdown = nil
			if allocs == 0 {
				return
			}

		case req := <-dist:
			// New retrieval request arrived to be distributed to some fetcher process
			queue := requests[req.bit]
			index := sort.Search(len(queue), func(i int) bool { return queue[i] >= req.section })
			requests[req.bit] = append(queue[:index], append([]uint64{req.section}, queue[index:]...)...)

			// If it's a new bit and we have waiting fetchers, allocate to them
			if len(queue) == 0 {
				assign(req.bit)
			}

		case fetcher := <-retrievers:
			// New retriever arrived, find the lowest section-ed bit to assign
			bit, best := uint(0), uint64(math.MaxUint64)
			for idx := range unallocs {
				if requests[idx][0] < best {
					bit, best = idx, requests[idx][0]
				}
			}
			// Stop tracking this bit (and alloc notifications if no more work is available)
			delete(unallocs, bit)
			if len(unallocs) == 0 {
				retrievers = nil
			}
			allocs++
			fetcher <- bit

		case fetcher := <-m.counters:
			// New task count request arrives, return number of items
			fetcher <- uint(len(requests[<-fetcher]))

		case fetcher := <-m.retrievals:
			// New fetcher waiting for tasks to retrieve, assign
			task := <-fetcher
			if want := len(task.Sections); want >= len(requests[task.Bit]) {
				task.Sections = requests[task.Bit]
				delete(requests, task.Bit)
			} else {
				task.Sections = append(task.Sections[:0], requests[task.Bit][:want]...)
				requests[task.Bit] = append(requests[task.Bit][:0], requests[task.Bit][want:]...)
			}
			fetcher <- task

			// If anything was left unallocated, try to assign to someone else
			if len(requests[task.Bit]) > 0 {
				assign(task.Bit)
			}

		case result := <-m.deliveries:
			// New retrieval task response from fetcher, split out missing sections and
			// deliver complete ones
			var (
				sections = make([]uint64, 0, len(result.Sections))
				bitsets  = make([][]byte, 0, len(result.Bitsets))
				missing  = make([]uint64, 0, len(result.Sections))
			)
			for i, bitset := range result.Bitsets {
				if len(bitset) == 0 {
					missing = append(missing, result.Sections[i])
					continue
				}
				sections = append(sections, result.Sections[i])
				bitsets = append(bitsets, bitset)
			}
			m.schedulers[result.Bit].deliver(sections, bitsets)
			allocs--

			// Reschedule missing sections and allocate bit if newly available
			if len(missing) > 0 {
				queue := requests[result.Bit]
				for _, section := range missing {
					index := sort.Search(len(queue), func(i int) bool { return queue[i] >= section })
					queue = append(queue[:index], append([]uint64{section}, queue[index:]...)...)
				}
				requests[result.Bit] = queue

				if len(queue) == len(missing) {
					assign(result.Bit)
				}
			}

			// End the session when all pending deliveries have arrived.
			if shutdown == nil && allocs == 0 {
				return
			}
		}
	}
}

```


任务领取AllocateRetrieval。 任务领取了一个任务。 会返回指定的bit的检索任务。

```go
// allocateRetrieval assigns a bloom bit index to a client process that can either
// immediately request and fetch the section contents assigned to this bit or wait
// a little while for more sections to be requested.
func (s *MatcherSession) allocateRetrieval() (uint, bool) {
	fetcher := make(chan uint)

	select {
	case <-s.quit:
		return 0, false
	case s.matcher.retrievers <- fetcher:
		bit, ok := <-fetcher
		return bit, ok
	}
}
```



AllocateSections,领取指定bit的section查询任务。

```go
// allocateSections assigns all or part of an already allocated bit-task queue
// to the requesting process.
func (s *MatcherSession) allocateSections(bit uint, count int) []uint64 {
	fetcher := make(chan *Retrieval)

	select {
	case <-s.quit:
		return nil
	case s.matcher.retrievals <- fetcher:
		task := &Retrieval{
			Bit:      bit,
			Sections: make([]uint64, count),
		}
		fetcher <- task
		return (<-fetcher).Sections
	}
}
```

DeliverSections，把结果投递给deliveries 通道。

```go
// deliverSections delivers a batch of section bit-vectors for a specific bloom
// bit index to be injected into the processing pipeline.
func (s *MatcherSession) deliverSections(bit uint, sections []uint64, bitsets [][]byte) {
	s.matcher.deliveries <- &Retrieval{Bit: bit, Sections: sections, Bitsets: bitsets}
}
```

任务的执行Multiplex,Multiplex函数不断的领取任务，把任务投递给bloomRequest队列。从队列获取结果。然后投递给distributor。 完成了整个过程。

```go
// Multiplex polls the matcher session for retrieval tasks and multiplexes it into
// the requested retrieval queue to be serviced together with other sessions.
//
// This method will block for the lifetime of the session. Even after termination
// of the session, any request in-flight need to be responded to! Empty responses
// are fine though in that case.
func (s *MatcherSession) Multiplex(batch int, wait time.Duration, mux chan chan *Retrieval) {
	for {
		// Allocate a new bloom bit index to retrieve data for, stopping when done
		bit, ok := s.allocateRetrieval()
		if !ok {
			return
		}
		// Bit allocated, throttle a bit if we're below our batch limit
		if s.pendingSections(bit) < batch {
			select {
			case <-s.quit:
				// Session terminating, we can't meaningfully service, abort
				s.allocateSections(bit, 0)
				s.deliverSections(bit, []uint64{}, [][]byte{})
				return

			case <-time.After(wait):
				// Throttling up, fetch whatever's available
			}
		}
		// Allocate as much as we can handle and request servicing
		sections := s.allocateSections(bit, batch)
		request := make(chan *Retrieval)

		select {
		case <-s.quit:
			// Session terminating, we can't meaningfully service, abort
			s.deliverSections(bit, sections, make([][]byte, len(sections)))
			return

		case mux <- request:
			// Retrieval accepted, something must arrive before we're aborting
			request <- &Retrieval{Bit: bit, Sections: sections, Context: s.ctx}

			result := <-request
			if result.Error != nil {
				s.err.Store(result.Error)
				s.Close()
			}
			s.deliverSections(result.Bit, result.Sections, result.Bitsets)
		}
	}
}

```

