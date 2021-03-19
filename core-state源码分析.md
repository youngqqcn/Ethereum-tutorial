# core/state
core/state 包主要为以太坊的state trie提供了一层缓存层(cache)

state的结构主要如下图

![image](picture/state_1.png)

蓝色的矩形代表本模块， 灰色的矩形代表外部模块。

- database主要提供了trie树的抽象，提供trie树的缓存和合约代码长度的缓存。
- journal主要提供了操作日志，以及操作回滚的功能。 
- state_object是account对象的抽象，提供了账户的一些功能。 
- statedb主要是提供了state trie的部分功能。

## database.go
database.go 提供了一个数据库的抽象。

数据结构

```go
// Database wraps access to tries and contract code.
type Database interface {
    // Accessing tries:
    // OpenTrie opens the main account trie.
    // OpenStorageTrie opens the storage trie of an account.
    // OpenTrie 打开了主账号的trie树
    // OpenStorageTrie 打开了一个账号的storage trie
    OpenTrie(root common.Hash) (Trie, error)
    OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
    // Accessing contract code:
    // 访问合约代码
    ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
    // 访问合约的大小。 这个方法可能经常被调用。因为有缓存。
    ContractCodeSize(addrHash, codeHash common.Hash) (int, error)
    // CopyTrie returns an independent copy of the given trie.
    // CopyTrie 返回了一个指定trie的独立的copy
    CopyTrie(Trie) Trie


	// TrieDB retrieves the low level trie database used for data storage.
	TrieDB() *trie.Database
}


// MPT树
// Trie is a Ethereum Merkle Patricia trie.
type Trie interface {
    // GetKey returns the sha3 preimage of a hashed key that was previously used
    // to store a value.
    //
    // TODO(fjl): remove this when SecureTrie is removed
    GetKey([]byte) []byte

    // 获取
    // TryGet returns the value for key stored in the trie. The value bytes must
    // not be modified by the caller. If a node was not found in the database, a
    // trie.MissingNodeError is returned.
    TryGet(key []byte) ([]byte, error)

    // 更新
    // TryUpdate associates key with value in the trie. If value has length zero, any
    // existing value is deleted from the trie. The value bytes must not be modified
    // by the caller while they are stored in the trie. If a node was not found in the
    // database, a trie.MissingNodeError is returned.
    TryUpdate(key, value []byte) error

    // 删除
    // TryDelete removes any existing value for key from the trie. If a node was not
    // found in the database, a trie.MissingNodeError is returned.
    TryDelete(key []byte) error

    // 获取 根哈希
    // Hash returns the root hash of the trie. It does not write to the database and
    // can be used even if the trie doesn't have one.
    Hash() common.Hash


    // 提交
    // Commit writes all nodes to the trie's memory database, tracking the internal
    // and external (for account tries) references.
    Commit(onleaf trie.LeafCallback) (common.Hash, error)

    // 迭代器
    // NodeIterator returns an iterator that returns nodes of the trie. Iteration
    // starts at the key after the given start key.
    NodeIterator(startKey []byte) trie.NodeIterator

    // 证明
    // 
    // Prove constructs a Merkle proof for key. The result contains all encoded nodes
    // on the path to the value at key. The value itself is also included in the last
    // node and can be retrieved by verifying the proof.
    //
    // If the trie does not contain a value for key, the returned proof contains all
    // nodes of the longest existing prefix of the key (at least the root), ending
    // with the node that proves the absence of the key.
    Prove(key []byte, fromLevel uint, proofDb ethdb.KeyValueWriter) error
}

// NewDatabase creates a backing store for state. The returned database is safe for
// concurrent use and retains cached trie nodes in memory.
func NewDatabase(db ethdb.Database) Database {
    csc, _ := lru.New(codeSizeCacheSize)
    return &cachingDB{db: db, codeSizeCache: csc}
}

//cachingDB 对 Database 的实现
type cachingDB struct {
    db            ethdb.Database
    mu            sync.Mutex
    pastTries     []*trie.SecureTrie  //trie树的缓存
    codeSizeCache *lru.Cache		  //合约代码大小的缓存
}

// OpenTrie opens the main account trie at a specific root hash.
func (db *cachingDB) OpenTrie(root common.Hash) (Trie, error) {
	tr, err := trie.NewSecure(root, db.db)
	if err != nil {
		return nil, err
	}
	return tr, nil
}

// OpenStorageTrie opens the storage trie of an account.
func (db *cachingDB) OpenStorageTrie(addrHash, root common.Hash) (Trie, error) {
	tr, err := trie.NewSecure(root, db.db)
	if err != nil {
		return nil, err
	}
	return tr, nil
}

// CopyTrie returns an independent copy of the given trie.
func (db *cachingDB) CopyTrie(t Trie) Trie {
	switch t := t.(type) {
	case *trie.SecureTrie:
		return t.Copy()
	default:
		panic(fmt.Errorf("unknown trie type %T", t))
	}
}

// ContractCode retrieves a particular contract's code.
func (db *cachingDB) ContractCode(addrHash, codeHash common.Hash) ([]byte, error) {
	if code := db.codeCache.Get(nil, codeHash.Bytes()); len(code) > 0 {
		return code, nil
	}
	code := rawdb.ReadCode(db.db.DiskDB(), codeHash)
	if len(code) > 0 {
		db.codeCache.Set(codeHash.Bytes(), code)
		db.codeSizeCache.Add(codeHash, len(code))
		return code, nil
	}
	return nil, errors.New("not found")
}

// ContractCodeWithPrefix retrieves a particular contract's code. If the
// code can't be found in the cache, then check the existence with **new**
// db scheme.
func (db *cachingDB) ContractCodeWithPrefix(addrHash, codeHash common.Hash) ([]byte, error) {
	if code := db.codeCache.Get(nil, codeHash.Bytes()); len(code) > 0 {
		return code, nil
	}
	code := rawdb.ReadCodeWithPrefix(db.db.DiskDB(), codeHash)
	if len(code) > 0 {
		db.codeCache.Set(codeHash.Bytes(), code)
		db.codeSizeCache.Add(codeHash, len(code))
		return code, nil
	}
	return nil, errors.New("not found")
}

// ContractCodeSize retrieves a particular contracts code's size.
func (db *cachingDB) ContractCodeSize(addrHash, codeHash common.Hash) (int, error) {
	if cached, ok := db.codeSizeCache.Get(codeHash); ok {
		return cached.(int), nil
	}
	code, err := db.ContractCode(addrHash, codeHash)
	return len(code), err
}

// TrieDB retrieves any intermediate trie-node caching layer.
func (db *cachingDB) TrieDB() *trie.Database {
	return db.db
}
```



## journal.go
journal代表了操作日志， 并针对各种操作的日志提供了对应的回滚功能。 可以基于这个日志来做一些事务类型的操作。

```go
// journalEntry is a modification entry in the state change journal that can be
// reverted on demand.
type journalEntry interface {
	// revert undoes the changes introduced by this journal entry.
	revert(*StateDB)

	// dirtied returns the Ethereum address modified by this journal entry.
	dirtied() *common.Address
}

// journal contains the list of state modifications applied since the last state
// commit. These are tracked to be able to be reverted in case of an execution
// exception or revertal request.
type journal struct {
	entries []journalEntry         // Current changes tracked by the journal
	dirties map[common.Address]int // Dirty accounts and the number of changes
}
```
	

各种不同的日志类型以及undo方法。

```go
createObjectChange struct {  //创建对象的日志。 undo方法就是从StateDB中删除创建的对象。
    account *common.Address
}
func (ch createObjectChange) undo(s *StateDB) {
    delete(s.stateObjects, *ch.account)
    delete(s.stateObjectsDirty, *ch.account)
}
// 对于stateObject的修改， undo方法就是把值改为原来的对象。
resetObjectChange struct {
    prev *stateObject
}
func (ch resetObjectChange) undo(s *StateDB) {
    s.setStateObject(ch.prev)
}
// 自杀的更改。自杀应该是删除账号，但是如果没有commit的化，对象还没有从stateDB删除。
suicideChange struct {
    account     *common.Address
    prev        bool // whether account had already suicided
    prevbalance *big.Int
}
func (ch suicideChange) undo(s *StateDB) {
    obj := s.getStateObject(*ch.account)
    if obj != nil {
        obj.suicided = ch.prev
        obj.setBalance(ch.prevbalance)
    }
}

// Changes to individual accounts.
balanceChange struct {
    account *common.Address
    prev    *big.Int
}
nonceChange struct {
    account *common.Address
    prev    uint64
}
storageChange struct {
    account       *common.Address
    key, prevalue common.Hash
}
codeChange struct {
    account            *common.Address
    prevcode, prevhash []byte
}

func (ch balanceChange) undo(s *StateDB) {
    s.getStateObject(*ch.account).setBalance(ch.prev)
}
func (ch nonceChange) undo(s *StateDB) {
    s.getStateObject(*ch.account).setNonce(ch.prev)
}
func (ch codeChange) undo(s *StateDB) {
    s.getStateObject(*ch.account).setCode(common.BytesToHash(ch.prevhash), ch.prevcode)
}
func (ch storageChange) undo(s *StateDB) {
    s.getStateObject(*ch.account).setState(ch.key, ch.prevalue)
}

// 退还gas
refundChange struct {
    prev *big.Int
}
func (ch refundChange) undo(s *StateDB) {
    s.refund = ch.prev
}

// 增加了日志的修改
addLogChange struct {
    txhash common.Hash
}
func (ch addLogChange) undo(s *StateDB) {
    logs := s.logs[ch.txhash]
    if len(logs) == 1 {
        delete(s.logs, ch.txhash)
    } else {
        s.logs[ch.txhash] = logs[:len(logs)-1]
    }
    s.logSize--
}
// 这个是增加 VM看到的 SHA3的 原始byte[], 增加SHA3 hash -> byte[] 的对应关系
addPreimageChange struct {
    hash common.Hash
}
func (ch addPreimageChange) undo(s *StateDB) {
    delete(s.preimages, ch.hash)
}

touchChange struct {
    account   *common.Address
    prev      bool
    prevDirty bool
}
var ripemd = common.HexToAddress("0000000000000000000000000000000000000003")
func (ch touchChange) undo(s *StateDB) {
    if !ch.prev && *ch.account != ripemd {
        s.getStateObject(*ch.account).touched = ch.prev
        if !ch.prevDirty {
            delete(s.stateObjectsDirty, *ch.account)
        }
    }
}
```


## state_object.go
stateObject表示正在修改的以太坊帐户。

数据结构


```go
type Storage map[common.Hash]common.Hash

// stateObject represents an Ethereum account which is being modified.
// stateObject表示正在修改的以太坊帐户。
// The usage pattern is as follows:
// First you need to obtain a state object.
// Account values can be accessed and modified through the object.
// Finally, call CommitTrie to write the modified storage trie into a database.

使用模式如下：
首先你需要获得一个state_object。
帐户值可以通过对象访问和修改。
最后，调用CommitTrie将修改后的存储trie写入数据库。

type stateObject struct {
    address  common.Address
    addrHash common.Hash // hash of ethereum address of the account 以太坊账号地址的hash值
    data     Account  // 这个是实际的以太坊账号的信息
    db       *StateDB //状态数据库

    // DB error.
    // State objects are used by the consensus core and VM which are
    // unable to deal with database-level errors. Any error that occurs
    // during a database read is memoized here and will eventually be returned
    // by StateDB.Commit.
    // 
    // 数据库错误。
    // stateObject会被共识算法的核心和VM使用，在这些代码内部无法处理数据库级别的错误。 
    // 在数据库读取期间发生的任何错误都会在这里被存储，最终将由StateDB.Commit返回。
    dbErr error

    // Write caches.  写缓存
    trie Trie // storage trie, which becomes non-nil on first access 用户的存储trie ，在第一次访问的时候变得非空
    code Code // contract bytecode, which gets set when code is loaded 合约代码，当代码被加载的时候被设置

    cachedStorage Storage // Storage entry cache to avoid duplicate reads 用户存储对象的缓存，用来避免重复读
    dirtyStorage  Storage // Storage entries that need to be flushed to disk 需要刷入磁盘的用户存储对象

    // Cache flags.  Cache 标志
    // When an object is marked suicided it will be delete from the trie
    // during the "update" phase of the state transition.
    // 当一个对象被标记为自杀时，它将在状态转换的“更新”阶段期间从树中删除。
    dirtyCode bool // true if the code was updated 如果代码被更新，会设置为true
    suicided  bool
    touched   bool
    deleted   bool
    onDirty   func(addr common.Address) // Callback method to mark a state object newly dirty  第一次被设置为drity的时候会被调用。
}

// Account is the Ethereum consensus representation of accounts.
// These objects are stored in the main account trie.
// 帐户是以太坊共识表示的帐户。 这些对象存储在main account trie。
type Account struct {
    Nonce    uint64
    Balance  *big.Int
    Root     common.Hash // merkle root of the storage trie
    CodeHash []byte
}
```


构造函数


```go
// newObject creates a state object.
func newObject(db *StateDB, address common.Address, data Account, onDirty func(addr common.Address)) *stateObject {
    if data.Balance == nil {
        data.Balance = new(big.Int)
    }
    if data.CodeHash == nil {
        data.CodeHash = emptyCodeHash
    }
    return &stateObject{
        db:            db,
        address:       address,
        addrHash:      crypto.Keccak256Hash(address[:]),
        data:          data,
        cachedStorage: make(Storage),
        dirtyStorage:  make(Storage),
        onDirty:       onDirty,
    }
}
```


RLP的编码方式，只会编码 Account对象。

```go
// EncodeRLP implements rlp.Encoder.
func (c *stateObject) EncodeRLP(w io.Writer) error {
    return rlp.Encode(w, c.data)
}
```

一些状态改变的函数。


```go
func (self *stateObject) markSuicided() {
    self.suicided = true
    if self.onDirty != nil {
        self.onDirty(self.Address())
        self.onDirty = nil
    }
}

func (c *stateObject) touch() {
    c.db.journal = append(c.db.journal, touchChange{
        account:   &c.address,
        prev:      c.touched,
        prevDirty: c.onDirty == nil,
    })
    if c.onDirty != nil {
        c.onDirty(c.Address())
        c.onDirty = nil
    }
    c.touched = true
}
```


Storage的处理

```go
// getTrie返回账户的Storage Trie
func (c *stateObject) getTrie(db Database) Trie {
    if c.trie == nil {
        var err error
        c.trie, err = db.OpenStorageTrie(c.addrHash, c.data.Root)
        if err != nil {
            c.trie, _ = db.OpenStorageTrie(c.addrHash, common.Hash{})
            c.setError(fmt.Errorf("can't create storage trie: %v", err))
        }
    }
    return c.trie
}

// GetState returns a value in account storage.
// GetState 返回account storage 的一个值，这个值的类型是Hash类型。
// 说明account storage里面只能存储hash值？
// 如果缓存里面存在就从缓存里查找，否则从数据库里面查询。然后存储到缓存里面。
func (self *stateObject) GetState(db Database, key common.Hash) common.Hash {
    value, exists := self.cachedStorage[key]
    if exists {
        return value
    }
    // Load from DB in case it is missing.
    enc, err := self.getTrie(db).TryGet(key[:])
    if err != nil {
        self.setError(err)
        return common.Hash{}
    }
    if len(enc) > 0 {
        _, content, _, err := rlp.Split(enc)
        if err != nil {
            self.setError(err)
        }
        value.SetBytes(content)
    }
    if (value != common.Hash{}) {
        self.cachedStorage[key] = value
    }
    return value
}

// SetState updates a value in account storage.
// 往 account storeage 里面设置一个值 key value 的类型都是Hash类型。
func (self *stateObject) SetState(db Database, key, value common.Hash) {
    self.db.journal = append(self.db.journal, storageChange{
        account:  &self.address,
        key:      key,
        prevalue: self.GetState(db, key),
    })
    self.setState(key, value)
}

func (self *stateObject) setState(key, value common.Hash) {
    self.cachedStorage[key] = value
    self.dirtyStorage[key] = value

    if self.onDirty != nil {
        self.onDirty(self.Address())
        self.onDirty = nil
    }
}
```


提交 Commit

```go
// CommitTrie the storage trie of the object to dwb.
// This updates the trie root.
// 步骤，首先打开，然后修改，然后提交或者回滚
func (self *stateObject) CommitTrie(db Database, dbw trie.DatabaseWriter) error {
    self.updateTrie(db) // updateTrie把修改过的缓存写入Trie树
    if self.dbErr != nil {
        return self.dbErr
    }
    root, err := self.trie.CommitTo(dbw)
    if err == nil {
        self.data.Root = root
    }
    return err
}

// updateTrie writes cached storage modifications into the object's storage trie.
func (self *stateObject) updateTrie(db Database) Trie {
    tr := self.getTrie(db)
    for key, value := range self.dirtyStorage {
        delete(self.dirtyStorage, key)
        if (value == common.Hash{}) {
            self.setError(tr.TryDelete(key[:]))
            continue
        }
        // Encoding []byte cannot fail, ok to ignore the error.
        v, _ := rlp.EncodeToBytes(bytes.TrimLeft(value[:], "\x00"))
        self.setError(tr.TryUpdate(key[:], v))
    }
    return tr
}

// UpdateRoot sets the trie root to the current root hash of
// 把账号的root设置为当前的trie树的跟。
func (self *stateObject) updateRoot(db Database) {
    self.updateTrie(db)
    self.data.Root = self.trie.Hash()
}
```


额外的一些功能 ，deepCopy提供了state_object的深拷贝。
	
```go
func (self *stateObject) deepCopy(db *StateDB, onDirty func(addr common.Address)) *stateObject {
    stateObject := newObject(db, self.address, self.data, onDirty)
    if self.trie != nil {
        stateObject.trie = db.db.CopyTrie(self.trie)
    }
    stateObject.code = self.code
    stateObject.dirtyStorage = self.dirtyStorage.Copy()
    stateObject.cachedStorage = self.dirtyStorage.Copy()
    stateObject.suicided = self.suicided
    stateObject.dirtyCode = self.dirtyCode
    stateObject.deleted = self.deleted
    return stateObject
}
```

## statedb.go

stateDB用来存储以太坊中关于merkle trie的所有内容。 StateDB负责缓存和存储嵌套状态。 这是检索合约和账户的一般查询界面：

数据结构

```go

// StateDB structs within the ethereum protocol are used to store anything
// within the merkle trie. StateDBs take care of caching and storing
// nested states. It's the general query interface to retrieve:
// * Contracts
// * Accounts
type StateDB struct {
	db           Database
	prefetcher   *triePrefetcher

    // 更新前的状态hash
	originalRoot common.Hash // The pre-state root, before any changes were made
	trie         Trie
	hasher       crypto.KeccakState

    // 快照相关
	snaps         *snapshot.Tree
	snap          snapshot.Snapshot
	snapDestructs map[common.Hash]struct{}
	snapAccounts  map[common.Hash][]byte
	snapStorage   map[common.Hash]map[common.Hash][]byte

    // 活动的对象, 状态改变这些对象也会发生变化
	// This map holds 'live' objects, which will get modified while processing a state transition.
	stateObjects        map[common.Address]*stateObject
	stateObjectsPending map[common.Address]struct{} // State objects finalized but not yet written to the trie
	stateObjectsDirty   map[common.Address]struct{} // State objects modified in the current execution

	// DB error.
	// State objects are used by the consensus core and VM which are
	// unable to deal with database-level errors. Any error that occurs
	// during a database read is memoized here and will eventually be returned
	// by StateDB.Commit.
	dbErr error

    // 用于统计退还的gas
	// The refund counter, also used by state transitioning.
	refund uint64

	thash, bhash common.Hash
	txIndex      int
	logs         map[common.Hash][]*types.Log
	logSize      uint

	preimages map[common.Hash][]byte

	// Per-transaction access list
	accessList *accessList

	// Journal of state modifications. This is the backbone of
	// Snapshot and RevertToSnapshot.
	journal        *journal
	validRevisions []revision
	nextRevisionId int

    // 统计各种操作的耗时, 用于调试
	// Measurements gathered during execution for debugging purposes
	AccountReads         time.Duration
	AccountHashes        time.Duration
	AccountUpdates       time.Duration
	AccountCommits       time.Duration
	StorageReads         time.Duration
	StorageHashes        time.Duration
	StorageUpdates       time.Duration
	StorageCommits       time.Duration
	SnapshotAccountReads time.Duration
	SnapshotStorageReads time.Duration
	SnapshotCommits      time.Duration
}

```



其中 
- stateObjects 用来缓存状态对象，
- stateObjectsDirty 用来缓存被修改过的对象，这部分代码在 core/state/state_object.go 里。stateObject 主要依赖于 core/state/database.go，core/state/database.go 中的 Database（也是 StateDB 中的 Database） 封装了一下对 MPT 树的操作，可以增删改查世界状态，余额等。
- journal 表示操作日志，core/state/journal.go 针对各种操作日志提供了对应的回滚功能，可以基于这个日志做一些事务类型的操作。





构造函数

```go
// 一般的用法 statedb, _ := state.New(common.Hash{}, state.NewDatabase(db))

// Create a new state from a given trie
func New(root common.Hash, db Database) (*StateDB, error) {
    tr, err := db.OpenTrie(root)
    if err != nil {
        return nil, err
    }
    return &StateDB{
        db:                db,
        trie:              tr,
        stateObjects:      make(map[common.Address]*stateObject),
        stateObjectsDirty: make(map[common.Address]struct{}),
        refund:            new(big.Int),
        logs:              make(map[common.Hash][]*types.Log),
        preimages:         make(map[common.Hash][]byte),
    }, nil
}
```



初始化的时候会利用 OpenTrie 获取 trie，这部分功能由 `core/state/database.go` 提供，`core/state/database.go` 封装了与数据库交互的代码

### 对于Log的处理

state提供了Log的处理，这比较意外，因为Log实际上是存储在区块链中的，并没有存储在state trie中, state提供Log的处理， 使用了基于下面的几个函数。  奇怪的是暂时没看到如何删除logs里面的信息，如果不删除的话，会越积累越多。 TODO logs 删除

Prepare函数，在交易执行开始被执行。

AddLog函数，在交易执行过程中被VM执行。添加日志。同时把日志和交易关联起来，添加部分交易的信息。

GetLogs函数，交易完成取走。


```go
// Prepare sets the current transaction hash and index and block hash which is
// used when the EVM emits new state logs.
func (self *StateDB) Prepare(thash, bhash common.Hash, ti int) {
    self.thash = thash
    self.bhash = bhash
    self.txIndex = ti
}

func (self *StateDB) AddLog(log *types.Log) {
    self.journal = append(self.journal, addLogChange{txhash: self.thash})

    log.TxHash = self.thash
    log.BlockHash = self.bhash
    log.TxIndex = uint(self.txIndex)
    log.Index = self.logSize
    self.logs[self.thash] = append(self.logs[self.thash], log)
    self.logSize++
}
func (self *StateDB) GetLogs(hash common.Hash) []*types.Log {
    return self.logs[hash]
}

func (self *StateDB) Logs() []*types.Log {
    var logs []*types.Log
    for _, lgs := range self.logs {
        logs = append(logs, lgs...)
    }
    return logs
}
```

### stateObject处理
getStateObject,首先从缓存里面获取，如果没有就从trie树里面获取，并加载到缓存。


```go
// Retrieve a state object given my the address. Returns nil if not found.
func (self *StateDB) getStateObject(addr common.Address) (stateObject *stateObject) {
    // Prefer 'live' objects.
    if obj := self.stateObjects[addr]; obj != nil {
        if obj.deleted {
            return nil
        }
        return obj
    }

    // Load the object from the database.
    enc, err := self.trie.TryGet(addr[:])
    if len(enc) == 0 {
        self.setError(err)
        return nil
    }
    var data Account
    if err := rlp.DecodeBytes(enc, &data); err != nil {
        log.Error("Failed to decode state object", "addr", addr, "err", err)
        return nil
    }
    // Insert into the live set.
    obj := newObject(self, addr, data, self.MarkStateObjectDirty)
    self.setStateObject(obj)
    return obj
}
```


MarkStateObjectDirty， 设置一个stateObject为Dirty。 直接往stateObjectDirty对应的地址插入一个空结构体。

```go
// MarkStateObjectDirty adds the specified object to the dirty map to avoid costly
// state object cache iteration to find a handful of modified ones.
func (self *StateDB) MarkStateObjectDirty(addr common.Address) {
    self.stateObjectsDirty[addr] = struct{}{}
}
```

### 快照和回滚功能
Snapshot可以创建一个快照， 然后通过	RevertToSnapshot可以回滚到哪个状态，这个功能是通过journal来做到的。 每一步的修改都会往journal里面添加一个undo日志。 如果需要回滚只需要执行undo日志就行了。


```go
// Snapshot returns an identifier for the current revision of the state.
func (s *StateDB) Snapshot() int {
	id := s.nextRevisionId
	s.nextRevisionId++
	s.validRevisions = append(s.validRevisions, revision{id, s.journal.length()})
	return id
}

// RevertToSnapshot reverts all state changes made since the given revision.
func (s *StateDB) RevertToSnapshot(revid int) {
	// Find the snapshot in the stack of valid snapshots.
	idx := sort.Search(len(s.validRevisions), func(i int) bool {
		return s.validRevisions[i].id >= revid
	})
	if idx == len(s.validRevisions) || s.validRevisions[idx].id != revid {
		panic(fmt.Errorf("revision id %v cannot be reverted", revid))
	}
	snapshot := s.validRevisions[idx].journalIndex

	// Replay the journal to undo changes and remove invalidated snapshots
	s.journal.revert(s, snapshot)
	s.validRevisions = s.validRevisions[:idx]
}
```


### 获取中间状态的 root hash值
IntermediateRoot 用来计算当前的state trie的root的hash值。这个方法会在交易执行的过程中被调用。会被存入 transaction receipt

Finalise方法会调用update方法把存放在cache层的修改写入到trie数据库里面。 但是这个时候还没有写入底层的数据库。 还没有调用commit，数据还在内存里面，还没有落地成文件。


```go

// Finalise finalises the state by removing the s destructed objects and clears
// the journal as well as the refunds. Finalise, however, will not push any updates
// into the tries just yet. Only IntermediateRoot or Commit will do that.
func (s *StateDB) Finalise(deleteEmptyObjects bool) {
	addressesToPrefetch := make([][]byte, 0, len(s.journal.dirties))
	for addr := range s.journal.dirties {
		obj, exist := s.stateObjects[addr]
		if !exist {
			// ripeMD is 'touched' at block 1714175, in tx 0x1237f737031e40bcde4a8b7e717b2d15e3ecadfe49bb1bbc71ee9deb09c6fcf2
			// That tx goes out of gas, and although the notion of 'touched' does not exist there, the
			// touch-event will still be recorded in the journal. Since ripeMD is a special snowflake,
			// it will persist in the journal even though the journal is reverted. In this special circumstance,
			// it may exist in `s.journal.dirties` but not in `s.stateObjects`.
			// Thus, we can safely ignore it here
			continue
		}
		if obj.suicided || (deleteEmptyObjects && obj.empty()) {
			obj.deleted = true

			// If state snapshotting is active, also mark the destruction there.
			// Note, we can't do this only at the end of a block because multiple
			// transactions within the same block might self destruct and then
			// ressurrect an account; but the snapshotter needs both events.
			if s.snap != nil {
				s.snapDestructs[obj.addrHash] = struct{}{} // We need to maintain account deletions explicitly (will remain set indefinitely)
				delete(s.snapAccounts, obj.addrHash)       // Clear out any previously updated account data (may be recreated via a ressurrect)
				delete(s.snapStorage, obj.addrHash)        // Clear out any previously updated storage data (may be recreated via a ressurrect)
			}
		} else {
			obj.finalise(true) // Prefetch slots in the background
		}
		s.stateObjectsPending[addr] = struct{}{}
		s.stateObjectsDirty[addr] = struct{}{}

		// At this point, also ship the address off to the precacher. The precacher
		// will start loading tries, and when the change is eventually committed,
		// the commit-phase will be a lot faster
		addressesToPrefetch = append(addressesToPrefetch, common.CopyBytes(addr[:])) // Copy needed for closure
	}
	if s.prefetcher != nil && len(addressesToPrefetch) > 0 {
		s.prefetcher.prefetch(s.originalRoot, addressesToPrefetch)
	}
	// Invalidate journal because reverting across transactions is not allowed.
	s.clearJournalAndRefund()
}


// IntermediateRoot computes the current root hash of the state trie.
// It is called in between transactions to get the root hash that
// goes into transaction receipts.
func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash {
	// Finalise all the dirty storage states and write them into the tries
	s.Finalise(deleteEmptyObjects)

	// If there was a trie prefetcher operating, it gets aborted and irrevocably
	// modified after we start retrieving tries. Remove it from the statedb after
	// this round of use.
	//
	// This is weird pre-byzantium since the first tx runs with a prefetcher and
	// the remainder without, but pre-byzantium even the initial prefetcher is
	// useless, so no sleep lost.
	prefetcher := s.prefetcher
	if s.prefetcher != nil {
		defer func() {
			s.prefetcher.close()
			s.prefetcher = nil
		}()
	}
	// Although naively it makes sense to retrieve the account trie and then do
	// the contract storage and account updates sequentially, that short circuits
	// the account prefetcher. Instead, let's process all the storage updates
	// first, giving the account prefeches just a few more milliseconds of time
	// to pull useful data from disk.
	for addr := range s.stateObjectsPending {
		if obj := s.stateObjects[addr]; !obj.deleted {
			obj.updateRoot(s.db)
		}
	}
	// Now we're about to start to write changes to the trie. The trie is so far
	// _untouched_. We can check with the prefetcher, if it can give us a trie
	// which has the same root, but also has some content loaded into it.
	if prefetcher != nil {
		if trie := prefetcher.trie(s.originalRoot); trie != nil {
			s.trie = trie
		}
	}
	usedAddrs := make([][]byte, 0, len(s.stateObjectsPending))
	for addr := range s.stateObjectsPending {
		if obj := s.stateObjects[addr]; obj.deleted {
			s.deleteStateObject(obj)
		} else {
			s.updateStateObject(obj)
		}
		usedAddrs = append(usedAddrs, common.CopyBytes(addr[:])) // Copy needed for closure
	}
	if prefetcher != nil {
		prefetcher.used(s.originalRoot, usedAddrs)
	}
	if len(s.stateObjectsPending) > 0 {
		s.stateObjectsPending = make(map[common.Address]struct{})
	}
	// Track the amount of time wasted on hashing the account trie
	if metrics.EnabledExpensive {
		defer func(start time.Time) { s.AccountHashes += time.Since(start) }(time.Now())
	}
	return s.trie.Hash()
}
```

### commit方法
CommitTo用来提交更改。

```go

// Commit writes the state to the underlying in-memory trie database.
func (s *StateDB) Commit(deleteEmptyObjects bool) (common.Hash, error) {
	if s.dbErr != nil {
		return common.Hash{}, fmt.Errorf("commit aborted due to earlier error: %v", s.dbErr)
	}
	// Finalize any pending changes and merge everything into the tries
	s.IntermediateRoot(deleteEmptyObjects)

	// Commit objects to the trie, measuring the elapsed time
	codeWriter := s.db.TrieDB().DiskDB().NewBatch()
	for addr := range s.stateObjectsDirty {
		if obj := s.stateObjects[addr]; !obj.deleted {
			// Write any contract code associated with the state object
			if obj.code != nil && obj.dirtyCode {
				rawdb.WriteCode(codeWriter, common.BytesToHash(obj.CodeHash()), obj.code)
				obj.dirtyCode = false
			}
			// Write any storage changes in the state object to its storage trie
			if err := obj.CommitTrie(s.db); err != nil {
				return common.Hash{}, err
			}
		}
	}
	if len(s.stateObjectsDirty) > 0 {
		s.stateObjectsDirty = make(map[common.Address]struct{})
	}
	if codeWriter.ValueSize() > 0 {
		if err := codeWriter.Write(); err != nil {
			log.Crit("Failed to commit dirty codes", "error", err)
		}
	}
	// Write the account trie changes, measuing the amount of wasted time
	var start time.Time
	if metrics.EnabledExpensive {
		start = time.Now()
	}
	// The onleaf func is called _serially_, so we can reuse the same account
	// for unmarshalling every time.
	var account Account
	root, err := s.trie.Commit(func(path []byte, leaf []byte, parent common.Hash) error {
		if err := rlp.DecodeBytes(leaf, &account); err != nil {
			return nil
		}
		if account.Root != emptyRoot {
			s.db.TrieDB().Reference(account.Root, parent)
		}
		return nil
	})
	if metrics.EnabledExpensive {
		s.AccountCommits += time.Since(start)
	}
	// If snapshotting is enabled, update the snapshot tree with this new version
	if s.snap != nil {
		if metrics.EnabledExpensive {
			defer func(start time.Time) { s.SnapshotCommits += time.Since(start) }(time.Now())
		}
		// Only update if there's a state transition (skip empty Clique blocks)
		if parent := s.snap.Root(); parent != root {
			if err := s.snaps.Update(root, parent, s.snapDestructs, s.snapAccounts, s.snapStorage); err != nil {
				log.Warn("Failed to update snapshot tree", "from", parent, "to", root, "err", err)
			}
			// Keep 128 diff layers in the memory, persistent layer is 129th.
			// - head layer is paired with HEAD state
			// - head-1 layer is paired with HEAD-1 state
			// - head-127 layer(bottom-most diff layer) is paired with HEAD-127 state
			if err := s.snaps.Cap(root, 128); err != nil {
				log.Warn("Failed to cap snapshot tree", "root", root, "layers", 128, "err", err)
			}
		}
		s.snap, s.snapDestructs, s.snapAccounts, s.snapStorage = nil, nil, nil, nil
	}
	return root, err
}

```


### 总结
state包提供了用户和合约的状态管理的功能, 管理了状态和合约的各种状态转换, cache， trie， 数据库, 日志和回滚功能。

