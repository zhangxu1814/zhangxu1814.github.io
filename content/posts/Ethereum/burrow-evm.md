---
title: "Hyperledger Burrow中的evm及相关组件源码分析"
subtitle: ""
date: 2021-05-23T20:53:34+08:00
description: ""
# #上次修改时间
# lastmod: ""
draft: true
# author: ""
# authorLink: ""
# license: ""

tags: ["evm"]
categories: ["evm"]

# resources:
# - name: "resource-name"
#   src: "resource-path"
---
<!-- 此处内容将作为摘要，若为空，则将description变量的内容作为摘要 -->
<!--more-->

## acm

acm中包含了账户以及账户状态相关的代码

该文件夹的名称大概来源于“**ac**count **m**anagement”

### acmstate

#### state.go

`state.go` 中定义了两个hash值类型以及对应的编码/解码操作，还定义了对账户、存储、元数据的读/写/遍历操作接口

Metadatahash，表示元数据的hash，用于元数据寻址，为256位的字节数组；Codehash，表示账户中存储的代码的hash，用于CODEHASH指令，为256为字节数组。
```go
// MetadataHash is the keccak hash for the metadata. This is to make the metadata content-addressed
type MetadataHash [32]byte

// CodeHash is the keccak hash for the code for an account. This is used for the EVM CODEHASH opcode, and to find the
// correct Metadata for a contract
type CodeHash [32]byte
```

最常使用的接口为ReaderWriter接口，也就是[evm](#evmgo)在执行时所需的第一个参数，它包括了对于账户和存储的读写操作
```go
// Read and write account and storage state
type ReaderWriter interface {
	Reader
	Writer
}

// Read-only account and storage state
type Reader interface {
	AccountGetter
	StorageGetter
}

type Writer interface {
	AccountUpdater
	StorageSetter
}

type AccountGetter interface {
	// Get an account by its address return nil if it does not exist (which should not be an error)
	GetAccount(address crypto.Address) (*acm.Account, error)
}

type StorageGetter interface {
	// Retrieve a 32-byte value stored at key for the account at address, return Zero256 if key does not exist but
	// error if address does not
	GetStorage(address crypto.Address, key binary.Word256) (value []byte, err error)
}

type AccountUpdater interface {
	// Updates the fields of updatedAccount by address, creating the account
	// if it does not exist
	UpdateAccount(updatedAccount *acm.Account) error
	// Remove the account at address
	RemoveAccount(address crypto.Address) error
}

type StorageSetter interface {
	// Store a 32-byte value at key for the account at address, setting to Zero256 removes the key
	SetStorage(address crypto.Address, key binary.Word256, value []byte) error
}
```

#### state_cache.go

`state_cache.go` 定义了缓存的类型，对缓存内容的操作以及将缓存内容写回的方法。

```go
type Cache struct {
	sync.RWMutex                               //读写锁
	name     string                            //缓存名称
	backend  Reader                            //读操作接口，包括读取账户和读取存储
	accounts map[crypto.Address]*accountInfo   //具体缓存的内容
	readonly bool                              //只读标志位
}

type accountInfo struct {
	sync.RWMutex                               //读写锁
	account *acm.Account                       //账户
	storage map[binary.Word256][]byte          //账户的存储
	removed bool                               //账户是否已经移除
	updated bool                               //账户是否需要更新
}

type CacheOption func(*Cache) *Cache           //使用示例如ReadOnly

var ReadOnly CacheOption = func(cache *Cache) *Cache {
	cache.readonly = true
	return cache
}
```
Cache实现了ReaderWriter接口，但是所有方法均是对缓存中的accountInfo进行操作，每种方法都会先检查账户/存储是否在缓存中，如果没有，则将其加入缓存

Cache包含一个Sync方法，需要传入一个实现了Writer接口的类型作为参数，根据removed和updated标志位对账户和存储进行修改

## bcm

## execution

### engine

#### call_frame.go

`call_frame.go` 中定义了 `CallFrame` 结构体，相当于一个调用栈
```go
type CallFrame struct {
	// Cache this State wraps
	*acmstate.Cache
	// Where we sync
	backend acmstate.ReaderWriter
	// In order for nested cache to inherit any options
	cacheOptions []acmstate.CacheOption
	// Depth of the call stack
	callStackDepth uint64
	// Max call stack depth
	maxCallStackDepth uint64
}

// Create a new CallFrame to hold state updates at a particular level in the call stack
func NewCallFrame(st acmstate.ReaderWriter, cacheOptions ...acmstate.CacheOption) *CallFrame {
	return newCallFrame(st, 0, 0, cacheOptions...)
}

func newCallFrame(st acmstate.ReaderWriter, stackDepth uint64, maxCallStackDepth uint64, cacheOptions ...acmstate.CacheOption) *CallFrame {
	return &CallFrame{
		Cache:             acmstate.NewCache(st, cacheOptions...),
		backend:           st,
		cacheOptions:      cacheOptions,
		callStackDepth:    stackDepth,
		maxCallStackDepth: maxCallStackDepth,
	}
}
```
NewCallFrame函数暴露给外界用于创建新的CallFrame，该函数接受两个参数：实现了ReaderWriter接口的类型和任意多个CacheOption，并调用内部函数nameCallFrame来创建CallFrame

newCallFrame会根据传入的参数创建一个[Cache](#state_cachego)，用于缓存账户及存储信息

CallFrame包含一个Sync方法，用于将缓存写回，其内部会调用Cache的Sync()方法执行写回操作


#### call.go

call.go中实现了一个Call函数，为所有Callable.Call接口的实现类型提供了一层封装：
```go
// Call provides a standard wrapper for implementing Callable.Call with appropriate error handling and event firing.
func Call(state State, params CallParams, execute func(State, CallParams) ([]byte, error)) ([]byte, error) {
	maybe := new(errors.Maybe)
	if params.CallType == exec.CallTypeCall || params.CallType == exec.CallTypeCode {
		// NOTE: Delegate and Static CallTypes do not transfer the value to the callee.
		maybe.PushError(Transfer(state.CallFrame, params.Caller, params.Callee, &params.Value))
	}

	output := maybe.Bytes(execute(state, params))
	// fire the post call event (including exception if applicable) and make sure we return the accumulated call error
	maybe.PushError(FireCallEvent(state.CallFrame, maybe.Error(), state.EventSink, output, params))
	return output, maybe.Error()
}
```
其中第三个参数execute就是定义了具体的执行过程的函数，在burrow中，合约调用/原生合约/原生函数/wasm调用都定义了特定的执行函数，在实现[Callable.Call](#callablego)接口时调用上面的Call函数，并将特定的执行函数作为第三个参数传入

由该函数可以看出合约调用的一般流程：修改交易双方账户余额--执行合约--记录事件


#### callable.go

`callable.go` 中定义了合约调用/evm执行所需的一些参数和接口

```go
type Blockchain interface {
	LastBlockHeight() uint64
	LastBlockTime() time.Time
	BlockHash(height uint64) ([]byte, error)
	ChainID() string
}

type CallParams struct {
	CallType exec.CallType    //调用类型：Call CallCode DelegateCall StaticCall
	Origin   crypto.Address
	Caller   crypto.Address
	Callee   crypto.Address
	Input    []byte
	Value    big.Int
	Gas      *big.Int
}

type Callable interface {
	Call(state State, params CallParams) (output []byte, err error)
}
```

`Blockchain` 接口用于获取区块链相关信息，`CallParams` 结构体，表示执行智能合约所需的参数，它们用于evm的执行过程

`Callable` 接口，仅包含一个 `Call` 函数，合约调用/原生合约/原生函数/wasm调用都需要实现该接口


#### dispatcher.go

`dispatcher.go` 中定义了dispatcher的相关接口和函数，用于evm/wasm/native contract之间的相互调用

```go
type Dispatcher interface {
	// If this Dispatcher is capable of dispatching this account (e.g. if it has the correct bytecode) then return a
	// Callable that wraps the function, otherwise return nil
	Dispatch(acc *acm.Account) Callable
}
```
Dispatcher接口包含一个Dispatch函数，它接收一个账户类型作为参数，返回一个实现了[Callable](#callablego)接口的类型，也就是合约/原生合约/原生函数/wasm
```go
// An ExternalDispatcher is able to Dispatch accounts to external engines as well as Dispatch to itself
type ExternalDispatcher interface {
	Dispatcher
	SetExternals(externals Dispatcher)
}

// An ExternalDispatcher is able to Dispatch accounts to external engines as well as Dispatch to itself
// 包含接口的结构体类型在初始化时需要传入一个实现了该接口的类型进行赋值
type Externals struct {
	// Provide any foreign dispatchers to allow calls between VMs
	externals Dispatcher
}

//用于确认Externals是否实现了ExternalDispatcher接口
var _ ExternalDispatcher = (*Externals)(nil)

func (ed *Externals) Dispatch(acc *acm.Account) Callable {
	// Try external calls then fallback to EVM
	if ed.externals == nil {
		return nil
	}
	return ed.externals.Dispatch(acc)
}

func (ed *Externals) SetExternals(externals Dispatcher) {
	ed.externals = externals
}
```
ExternalDispatcher接口包含了一个Dispatcher和一个用于设置Dispatcher的SetExternals函数

#### options.go

`options.go` 中仅有一个Options结构体类型，它包含了创建evm实例所需要的参数：
```go
// Options are parameters that are generally stable across a burrow configuration.
// Defaults will be used for any zero values.
type Options struct {
	MemoryProvider           func(errors.Sink) Memory  //返回一个实现了Memory接口的类型用于内存读写,默认为dynamicMemory
	Natives                  Natives                   //原生合约或者原生函数，在burrow中的默认为账户权限操作和加密操作相关实现
	Nonce                    []byte
	DebugOpcodes             bool                      //是否为debug模式
	DumpTokens               bool
	CallStackMaxDepth        uint64                    //调用栈深度，用于初始化CallFrame
	DataStackInitialCapacity uint64                    //数据栈容量，用于初始化Stack，表示Stack中创建的切片的容量
	DataStackMaxDepth        uint64                    //数据栈深度，用于初始化Stack
	Logger                   *logging.Logger           //日志
}
```

#### engine/state.go

`state.go` 中仅包含了一个State结构体类型：
```go
type State struct {
	*CallFrame
	Blockchain
	exec.EventSink
}
```
State中包含三个字段：Callframe用于缓存状态以及对状态进行读写，Blockchain用于获取区块链信息，EventSink用于记录event

### evm

#### code.go

`code.go` 对编译好的合约代码进行了一层封装，对代码中的指令和数据进行了标识

```go
type Code struct {
	Bytecode     acm.Bytecode
	OpcodeBitset bitset.Bitset
}

// Build a Code object that includes analysis of which symbols are opcodes versus push data
func NewCode(code []byte) *Code {
	return &Code{
		Bytecode:     code,
		OpcodeBitset: opcodeBitset(code),
	}
}

// If code[i] is an opcode (rather than PUSH data) then bitset.IsSet(i) will be true
func opcodeBitset(code []byte) bitset.Bitset {
	bs := bitset.New(uint(len(code)))
	for i := 0; i < len(code); i++ {
		bs.Set(uint(i))
		symbol := asm.OpCode(code[i])
		if symbol >= asm.PUSH1 && symbol <= asm.PUSH32 {
			i += int(symbol - asm.PUSH1 + 1)
		}
	}
	return bs
}
```
opcodeBitset方法会将合约代码中的指令位设置为true


#### evm/contract.go

`contract.go` 中包含了合约具体执行的过程

```go
type Contract struct {
	*EVM
	*Code
}

func (c *Contract) Call(state engine.State, params engine.CallParams) ([]byte, error) {
	return engine.Call(state, params, c.execute)
}

// Executes the EVM code passed in the appropriate context
func (c *Contract) execute(st engine.State, params engine.CallParams) ([]byte, error) {
    c.debugf("(%d) (%s) %s (code=%d) gas: %v (d) %X\n",
		st.CallFrame.CallStackDepth(), params.Caller, params.Callee, c.Length(), *params.Gas, params.Input)

	if c.Length() == 0 {
		return nil, nil
	}

	if c.options.DumpTokens {
		dumpTokens(c.options.Nonce, params.Caller, params.Callee, c.GetBytecode())
	}

	// Program counter - the index into code that tracks current instruction
	var pc uint64
	// Return data from a call
	var returnData []byte

	// Maybe serves 3 purposes: 1. provides 'capture first error semantics', 2. reduces clutter of error handling
	// particular for 1, 3. acts a shared error sink for stack, memory, and the main execute loop
	maybe := new(errors.Maybe)

	// Provide stack and memory storage - passing in the callState as an error provider
	stack := NewStack(maybe, c.options.DataStackInitialCapacity, c.options.DataStackMaxDepth, params.Gas)
	memory := c.options.MemoryProvider(maybe)

    //主循环
    for {

    }

    return nil, maybe.Error()
}
```
Contract实现了[Callable.Call](#callablego)接口，其内部调用了[engine.Call](#callgo)函数，并将execute函数作为参数传入

execute是真正执行合约代码的函数，根据参数初始化pc、栈、内存，主循环为取指-执行循环，内部定义了每条指令的具体操作

#### evm.go

`evm.go` 中定义了EVM结构体类型以及EVM初始化函数：
```go
type EVM struct {
	options  engine.Options
	sequence uint64
	// Provide any foreign dispatchers to allow calls between VMs
    // Externals是一个实现了ExternalDispatcher接口的结构体类型
	engine.Externals
	externalDispatcher engine.Dispatcher
	// User dispatcher.CallableProvider to get access to other VMs
	logger *logging.Logger
}

func New(options engine.Options) *EVM {
	options = defaults.CompleteOptions(options) //如果Options中的MemoryProvider/Natives/Logger域为空，则将其设置为默认值
    //创建一个evm实例
	vm := &EVM{
		options: options,
	}
    //设置logger
	vm.logger = options.Logger.WithScope("NewVM").With("evm_nonce", options.Nonce)
	vm.externalDispatcher = engine.Dispatchers{&vm.Externals, options.Natives, vm}
	return vm
}

func Default() *EVM {
	return New(engine.Options{})
}
```
可以看到新建一个evm实例需要传入一个[Options](#optionsgo)结构体，它包含了evm实例所需的参数，如果Options中的MemoryProvider/Natives/Logger域为空，则CompleteOptions会将其设置为默认值

EVM结构体类型中包含一个engine.Externals匿名字段，Externals是一个实现了ExternalDispatcher接口的结构体类型，详见[dispatcher.go](#dispatchergo)，evm可以使用其SetExternals方法设置外部dispatcher，例如wasm（实际上对于evm来说外部dispatcher好像只有wasm）

EVM结构体类型中还包括一个externalDispatcher字段，其类型为Dispatcher，可以看到在新建evm实例时该字段被设置为一个Dispatchers类型，包含了上面提到的外部dispatcher，原生合约以及evm本身，因为EVM也实现了Dispatch方法：
```go
func (vm *EVM) Dispatch(acc *acm.Account) engine.Callable {
	// Let the EVM handle code-less (e.g. those created by a call) contracts (so only return nil if there is _other_ non-EVM code)
	if len(acc.EVMCode) == 0 && len(acc.Code()) != 0 {
		return nil
	}
	return vm.Contract(acc.EVMCode)
}

func (vm *EVM) Contract(code []byte) *Contract {
	return &Contract{
		EVM:  vm,
		Code: NewCode(code),
	}
}
```
EVM的Dispatch方法返回的是一个[Contract](#evmcontractgo)类型

```go
// Initiate an EVM call against the provided state pushing events to eventSink. code should contain the EVM bytecode,
// input the CallData (readable by CALLDATALOAD), value the amount of native token to transfer with the call
// an quantity metering the number of computational steps available to the execution according to the gas schedule.
func (vm *EVM) Execute(st acmstate.ReaderWriter, blockchain engine.Blockchain, eventSink exec.EventSink,
	params engine.CallParams, code []byte) ([]byte, error) {

	// Make it appear as if natives are stored in state
	st = native.NewState(vm.options.Natives, st)

	state := engine.State{
		CallFrame:  engine.NewCallFrame(st).WithMaxCallStackDepth(vm.options.CallStackMaxDepth),
		Blockchain: blockchain,
		EventSink:  eventSink,
	}

	output, err := vm.Contract(code).Call(state, params)
	if err == nil {
		// Only sync back when there was no exception
		err = state.CallFrame.Sync()
	}
	// Always return output - we may have a reverted exception for which the return is meaningful
	return output, err
}
```
Execute方法接收5个参数：ReaderWriter接口实现类型，用于读写账户状态；Blockchain接口实现类型，用于获取区块链相关信息；EventSink接口实现类型，用于记录event；CallParams结构体，包含了合约执行所需的参数；code，表示编译好的智能合约代码

Execute方法会调用native.NewState函数对ReaderWriter接口实现类型进行封装，使合约执行时先检查目标合约是否为原生合约

接下来会创建一个[State](#enginestatego)结构体，作为参数传入[Contract](#evmcontractgo)的Call方法来执行合约，并将结果保存至变量output中

当Call方法未出错时，调用Callframe的Sync方法将缓存中的账户状态变更写回，最后返回执行结果

#### stack.go

`stack.go` 中定义了evm中的栈以及相关栈操作

栈定义为 `Stack` 类型：
```go
// Not goroutine safe
type Stack struct {
	slice       []Word256
	maxCapacity uint64
	ptr         int

	gas     *big.Int
	errSink errors.Sink
}
```

### native

`native` 与原生合约有关，这里的原生合约(native contracts)的功能类似于以太坊evm中预编译好的合约，默认包括对于账户权限的操作和加密操作

#### contract.go

`contract.go` 中规定了原生合约声明函数的的规范，函数必须声明为以下形式：
```go
func unsetBase(context Context, args unsetBaseArgs) (unsetBaseRets, error) {}
```
必须有一个或两个参数以及两个返回值，第一个参数必须是 `Context` 类型：
```go
// Context is the first argument to any native function. This struct carries
// all the context an native needs to access e.g. state in burrow.
type Context struct {
	State engine.State
	engine.CallParams
	// TODO: this allows us to call back to EVM contracts if we wish - make use of it somewhere...
	externals engine.Dispatcher
	Logger    *logging.Logger
}
```
第二个参数以及第一个返回值必须是结构体类型，与solidity函数中的参数与返回值一一对应，第二个返回值必须是error类型