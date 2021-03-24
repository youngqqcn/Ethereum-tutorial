## StateTransition.go
状态转换模型

Process 部分的代码是插入区块时被调用的，而我们发现 Process 在处理每一个交易时，最终会调用 core/state_transition.go 中的 Finalise，实际上，如果说 core/state_processor.go 是用来处理区块级别的交易，那么可以说 core/state_transition.go 是用来处理一个一个的交易，最终将获得世界状态，收据，gas 等信息。

Transition 完成世界状态转换的工作，它会利用世界状态来执行交易，然后改变当前的世界状态。

```go
/*
The State Transitioning Model

A state transition is a change made when a transaction is applied to the current world state
The state transitioning model does all the necessary work to work out a valid new state root.

1) Nonce handling
2) Pre pay gas
3) Create a new state object if the recipient is \0*32
4) Value transfer
== If contract creation ==
  4a) Attempt to run transaction data
  4b) If valid, use result as code for the new state object
== end ==
5) Run Script section
6) Derive new state root
*/
type StateTransition struct {
	gp         *GasPool
	msg        Message
	gas        uint64
	gasPrice   *big.Int
	initialGas uint64
	value      *big.Int
	data       []byte
	state      vm.StateDB
	evm        *vm.EVM
}

// Message represents a message sent to a contract.
type Message interface {
	From() common.Address
	To() *common.Address

	GasPrice() *big.Int
	Gas() uint64
	Value() *big.Int

	Nonce() uint64
	CheckNonce() bool
	Data() []byte
}
```


构造
	
```go
// NewStateTransition initialises and returns a new state transition object.
func NewStateTransition(evm *vm.EVM, msg Message, gp *GasPool) *StateTransition {
	return &StateTransition{
		gp:       gp,
		evm:      evm,
		msg:      msg,
		gasPrice: msg.GasPrice(),
		value:    msg.Value(),
		data:     msg.Data(),
		state:    evm.StateDB,
	}
}

```


执行Message
	
```go

// ApplyMessage computes the new state by applying the given message
// against the old state within the environment.
//
// ApplyMessage returns the bytes returned by any EVM execution (if it took place),
// the gas used (which includes gas refunds) and an error if it failed. An error always
// indicates a core error meaning that the message would always fail for that particular
// state and would never be accepted within a block.
func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) (*ExecutionResult, error) {
	return NewStateTransition(evm, msg, gp).TransitionDb()
}

```

> TransitionDb,TransitionDb 首先会调用 preCheck，preCheck 的作用是检测 Nonce 的值是否正确，然后通过 buyGas() 购买 Gas.在 preCheck() 之后，会调用 IntrinsicGas 方法，计算交易的固有 Gas 消耗，这个消耗有两部分，一部分是交易（或创建合约）预设消耗量，一部分是根据交易 data 的非0字节和0字节长度决定的消耗量，TransitionDb 方法中最终会从 gas 中减去这笔消耗。再接下来就是 EVM 的执行，如果是创建合约，调用的是 evm.Create(sender, st.data, st.gas, st.value)，如果是普通的交易，调用的是 evm.Call(sender, st.to(), st.data, st.gas, st.value)，这两个方法同样也会消耗 gas。接着调用 refundGas 方法执行退 gas 的操作。refundGas 首先会将用户剩下的 gas 还回去，但退回的金额不会超过用户使用的 gas 的 1/2。TransitionDb 最后会通过 st.state.AddBalance 奖励区块的挖掘者，这笔钱等于 gasPrice*(initialGas-gas)，至此，这个交易的 gas 消耗量计算就完成了


```go
// TransitionDb will transition the state by applying the current message and
// returning the evm execution result with following fields.
//
// - used gas:
//      total gas used (including gas being refunded)
// - returndata:
//      the returned data from evm
// - concrete execution error:
//      various **EVM** error which aborts the execution,
//      e.g. ErrOutOfGas, ErrExecutionReverted
//
// However if any consensus issue encountered, return the error directly with
// nil evm execution result.
func (st *StateTransition) TransitionDb() (*ExecutionResult, error) {
	// First check this message satisfies all consensus rules before
	// applying the message. The rules include these clauses
	//
	// 1. the nonce of the message caller is correct
	// 2. caller has enough balance to cover transaction fee(gaslimit * gasprice)
	// 3. the amount of gas required is available in the block
	// 4. the purchased gas is enough to cover intrinsic usage
	// 5. there is no overflow when calculating intrinsic gas
	// 6. caller has enough balance to cover asset transfer for **topmost** call

	// Check clauses 1-3, buy gas if everything is correct
	if err := st.preCheck(); err != nil {
		return nil, err
	}
	msg := st.msg
	sender := vm.AccountRef(msg.From())
	homestead := st.evm.ChainConfig().IsHomestead(st.evm.Context.BlockNumber)
	istanbul := st.evm.ChainConfig().IsIstanbul(st.evm.Context.BlockNumber)
	contractCreation := msg.To() == nil

	// Check clauses 4-5, subtract intrinsic gas if everything is correct
	gas, err := IntrinsicGas(st.data, contractCreation, homestead, istanbul)
	if err != nil {
		return nil, err
	}
	if st.gas < gas {
		return nil, fmt.Errorf("%w: have %d, want %d", ErrIntrinsicGas, st.gas, gas)
	}
	st.gas -= gas

	// Check clause 6
	if msg.Value().Sign() > 0 && !st.evm.Context.CanTransfer(st.state, msg.From(), msg.Value()) {
		return nil, fmt.Errorf("%w: address %v", ErrInsufficientFundsForTransfer, msg.From().Hex())
	}
	var (
		ret   []byte
		vmerr error // vm errors do not effect consensus and are therefore not assigned to err
	)
	if contractCreation {
		ret, _, st.gas, vmerr = st.evm.Create(sender, st.data, st.gas, st.value)
	} else {
		// Increment the nonce for the next transaction
		st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
		ret, st.gas, vmerr = st.evm.Call(sender, st.to(), st.data, st.gas, st.value)
	}
	st.refundGas() // 退还剩余的gas
	st.state.AddBalance(st.evm.Context.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

	return &ExecutionResult{
		UsedGas:    st.gasUsed(),
		Err:        vmerr,
		ReturnData: ret,
	}, nil
}
```



关于g0的计算，在黄皮书上由详细的介绍
和黄皮书有一定出入的部分在于`if contractCreation && homestead {igas.SetUint64(params.TxGasContractCreation)` 这是因为 `Gtxcreate+Gtransaction = TxGasContractCreation`

```go

// IntrinsicGas computes the 'intrinsic gas' for a message with the given data.
func IntrinsicGas(data []byte, contractCreation, isHomestead bool, isEIP2028 bool) (uint64, error) {
	// Set the starting gas for the raw transaction
	var gas uint64
	if contractCreation && isHomestead {
		gas = params.TxGasContractCreation
	} else {
		gas = params.TxGas
	}
	// Bump the required gas by the amount of transactional data
	if len(data) > 0 {
		// Zero and non-zero bytes are priced differently
		var nz uint64
		for _, byt := range data {
			if byt != 0 {
				nz++
			}
		}
		// Make sure we don't exceed uint64 for all data combinations
		nonZeroGas := params.TxDataNonZeroGasFrontier
		if isEIP2028 {
			nonZeroGas = params.TxDataNonZeroGasEIP2028
		}
		if (math.MaxUint64-gas)/nonZeroGas < nz {
			return 0, ErrGasUintOverflow
		}
		gas += nz * nonZeroGas

		z := uint64(len(data)) - nz
		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, ErrGasUintOverflow
		}
		gas += z * params.TxDataZeroGas
	}
	return gas, nil
}

```


执行前的检查


```go

func (st *StateTransition) preCheck() error {
	// Make sure this transaction's nonce is correct.
    // 当前本地的nonce 需要和 msg的Nonce一样 不然就是状态不同步了
	if st.msg.CheckNonce() {
		stNonce := st.state.GetNonce(st.msg.From())
		if msgNonce := st.msg.Nonce(); stNonce < msgNonce {
			return fmt.Errorf("%w: address %v, tx: %d state: %d", ErrNonceTooHigh,
				st.msg.From().Hex(), msgNonce, stNonce)
		} else if stNonce > msgNonce {
			return fmt.Errorf("%w: address %v, tx: %d state: %d", ErrNonceTooLow,
				st.msg.From().Hex(), msgNonce, stNonce)
		}
	}
	return st.buyGas()
}
```

buyGas， 实现Gas的预扣费，  首先就扣除你的GasLimit * GasPrice的钱。 然后根据计算完的状态在退还一部分。



```go
func (st *StateTransition) buyGas() error {
	mgval := new(big.Int).Mul(new(big.Int).SetUint64(st.msg.Gas()), st.gasPrice)
	if have, want := st.state.GetBalance(st.msg.From()), mgval; have.Cmp(want) < 0 {
		return fmt.Errorf("%w: address %v have %v want %v", ErrInsufficientFunds, st.msg.From().Hex(), have, want)
	}

    // 从区块的gaspool里面减去， 因为区块是由GasLimit限制整个区块的Gas使用的。
	if err := st.gp.SubGas(st.msg.Gas()); err != nil {
		return err
	}
	st.gas += st.msg.Gas()

	st.initialGas = st.msg.Gas()
	st.state.SubBalance(st.msg.From(), mgval)
	return nil
}
```

```go

func (st *StateTransition) refundGas() {

    // // 然后退税的总金额不会超过用户Gas总使用的1/2。 
	// Apply refund counter, capped to half of the used gas.
	refund := st.gasUsed() / 2
	if refund > st.state.GetRefund() {
		refund = st.state.GetRefund()
	}
	st.gas += refund

	// Return ETH for remaining gas, exchanged at the original rate.
	remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
	st.state.AddBalance(st.msg.From(), remaining)

	// Also return remaining gas to the block gas counter so it is
	// available for the next transaction.
	st.gp.AddGas(st.gas)
}
```


## StateProcessor.go
StateTransition是用来处理一个一个的交易的。那么StateProcessor就是用来处理区块级别的交易的。

结构和构造


```go
// StateProcessor is a basic Processor, which takes care of transitioning
// state from one point to another.
//
// StateProcessor implements Processor.
type StateProcessor struct {
	config *params.ChainConfig // Chain configuration options
	bc     *BlockChain         // Canonical block chain
	engine consensus.Engine    // Consensus engine used for block rewards
}

// NewStateProcessor initialises a new StateProcessor.
func NewStateProcessor(config *params.ChainConfig, bc *BlockChain, engine consensus.Engine) *StateProcessor {
	return &StateProcessor{
		config: config,
		bc:     bc,
		engine: engine,
	}
}
```


Process，这个方法会被blockchain调用,Process 根据以太坊规则运行交易来对改变 statedb 的状态，以奖励挖矿者或其他的叔父节点。Process 会返回执行过程中累计的收据和日志，并返回过程中使用的 gas，如果 gas 不足导致任何交易执行失败，返回错误。

Process 首先会声明变量，值得注意的是 GasPool 变量，它告诉你剩下还有多少 Gas 可以使用，在每一个交易的执行过程中，以太坊设计了 refund 的机制进行处理，偿还的 gas 也会加到 GasPool 中。接着处理 DAO 事件的硬分叉，接着遍历区块中的所有交易，通过调用 ApplyTransaction 获得每个交易的收据。这个过程结束后，调用一致性引擎的 Finalize，拿到区块奖励或叔块奖励，最终将世界状态写入到区块链中，也就是说 stateTrie 随着每次交易的执行变化，
	

```go

// 处理交易
// Process processes the state changes according to the Ethereum rules by running
// the transaction messages using the statedb and applying any rewards to both
// the processor (coinbase) and any included uncles.
//
// Process returns the receipts and logs accumulated during the process and
// returns the amount of gas that was used in the process. If any of the
// transactions failed to execute due to insufficient gas it will return an error.
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
	var (
		receipts types.Receipts
		usedGas  = new(uint64)
		header   = block.Header()
		allLogs  []*types.Log
		gp       = new(GasPool).AddGas(block.GasLimit())
	)

    // // DAO 事件的硬分叉处理 
	// Mutate the block and state according to any hard-fork specs
	if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
		misc.ApplyDAOHardFork(statedb)
	}
	blockContext := NewEVMBlockContext(header, p.bc, nil)
	vmenv := vm.NewEVM(blockContext, vm.TxContext{}, statedb, p.config, cfg)
	// Iterate over and process the individual transactions
	for i, tx := range block.Transactions() {
		msg, err := tx.AsMessage(types.MakeSigner(p.config, header.Number))
		if err != nil {
			return nil, nil, 0, err
		}
		statedb.Prepare(tx.Hash(), block.Hash(), i)
		receipt, err := applyTransaction(msg, p.config, p.bc, nil, gp, statedb, header, tx, usedGas, vmenv)
		if err != nil {
			return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
		}
		receipts = append(receipts, receipt)
		allLogs = append(allLogs, receipt.Logs...)
	}
	// Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
	p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles())

	return receipts, allLogs, *usedGas, nil
}

```

ApplyTransaction,对于每个交易，都会创建一个新的虚拟机环境，即根据输入参数封装 EVM 对象，接着调用 ApplyMessage 将交易应用于当前的状态中，Finalise 方法会调用 update 方法，将存放在 cache 的修改写入 trie 数据库中，返回的是使用的 gas，这部分代码涉及到 core/state_transition.go。

接着，如果是拜占庭硬分叉，直接调用 statedb.Finalise(true)，否则调用 statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()，最后，初始化一个收据对象，其 TxHash 为交易哈希，Logs 字段通过 statedb.GetLogs(tx.Hash()) 拿到，Bloom 字段通过 types.CreateBloom(types.Receipts{receipt}) 创建，最后返回收据和消耗的 gas。


```go
//ApplyTransaction尝试将事务应用于给定的状态数据库
//，并将输入参数用于其环境。它返回收据
//对于交易而言，使用的是gas，如果交易失败，则为错误，
//指示该块无效。 
// ApplyTransaction attempts to apply a transaction to the given state database
// and uses the input parameters for its environment. It returns the receipt
// for the transaction, gas used and an error if the transaction failed,
// indicating the block was invalid.
func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, error) {
    // 把交易转换成Message 
	// 这里如何验证消息确实是Sender发送的。 TODO
	msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
	if err != nil {
		return nil, err
	}
	// Create a new context to be used in the EVM environment
	blockContext := NewEVMBlockContext(header, bc, author)
	vmenv := vm.NewEVM(blockContext, vm.TxContext{}, statedb, config, cfg)
	return applyTransaction(msg, config, bc, author, gp, statedb, header, tx, usedGas, vmenv)
}

func applyTransaction(msg types.Message, config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, evm *vm.EVM) (*types.Receipt, error) {
	// Create a new context to be used in the EVM environment
	txContext := NewEVMTxContext(msg)
	// Add addresses to access list if applicable
	if config.IsYoloV2(header.Number) {
		statedb.AddAddressToAccessList(msg.From())
		if dst := msg.To(); dst != nil {
			statedb.AddAddressToAccessList(*dst)
			// If it's a create-tx, the destination will be added inside evm.create
		}
		for _, addr := range evm.ActivePrecompiles() {
			statedb.AddAddressToAccessList(addr)
		}
	}

	// Update the evm with the new transaction context.
	evm.Reset(txContext, statedb)
	// Apply the transaction to the current state (included in the env)
	result, err := ApplyMessage(evm, msg, gp)
	if err != nil {
		return nil, err
	}

    // 求得中间状态
	// Update the state with pending changes
	var root []byte
	if config.IsByzantium(header.Number) {
		statedb.Finalise(true)
	} else {
		root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
	}
	*usedGas += result.UsedGas

    // 创建一个收据, 用来存储中间状态的root, 以及交易使用的gas
	// Create a new receipt for the transaction, storing the intermediate root and gas used by the tx
	// based on the eip phase, we're passing whether the root touch-delete accounts.
	receipt := types.NewReceipt(root, result.Failed(), *usedGas)
	receipt.TxHash = tx.Hash()
	receipt.GasUsed = result.UsedGas

    // 如果是创建合约的交易.那么我们把创建地址存储到收据里面.
	// if the transaction created a contract, store the creation address in the receipt.
	if msg.To() == nil {
		receipt.ContractAddress = crypto.CreateAddress(evm.TxContext.Origin, tx.Nonce())
	}
	// Set the receipt logs and create a bloom for filtering
	receipt.Logs = statedb.GetLogs(tx.Hash())
	receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
	receipt.BlockHash = statedb.BlockHash()
	receipt.BlockNumber = header.Number
	receipt.TransactionIndex = uint(statedb.TxIndex())

    // 拿到所有的日志并创建日志的布隆过滤器.
	return receipt, err
}


```
