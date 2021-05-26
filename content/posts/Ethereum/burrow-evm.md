---
title: "Hyperledger Burrow中的evm及相关组件源码分析"
subtitle: ""
date: 2021-05-23T20:53:34+08:00
description: ""
# #上次修改时间
# lastmod: ""
# draft: true
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
Hyperledger Burrow是Hyperledger下的一个子项目，它实现了符合以太坊规范的EVM，这篇文章中对evm及相关代码进行了分析
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

`state.go` 中还定义了一个函数用于获取全局账户权限
```go
// Get global permissions from the account at GlobalPermissionsAddress
func GlobalAccountPermissions(getter AccountGetter) (permission.AccountPermissions, error) {
	acc, err := getter.GetAccount(acm.GlobalPermissionsAddress)
	if err != nil {
		return permission.AccountPermissions{}, err
	}
	if acc == nil {
		return permission.AccountPermissions{}, fmt.Errorf("global permissions account is not defined but must be")
	}
	return acc.Permissions, nil
}
```
全局账户权限地址定义在[account.go](#acmaccountgo)中，为零值地址，该地址的权限字段包含了账户允许设置的全部权限

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

```go
// Syncs changes to the backend in deterministic order. Sends storage updates before updating
// the account they belong so that storage values can be taken account of in the update.
func (cache *Cache) Sync(st Writer) error {
	if cache.readonly {
		// Sync is (should be) a no-op for read-only - any modifications should have been caught in respective methods
		return nil
	}
	cache.Lock()
	defer cache.Unlock()
	var addresses crypto.Addresses
	for address := range cache.accounts {
		addresses = append(addresses, address)
	}

	sort.Sort(addresses)
	for _, address := range addresses {
		accInfo := cache.accounts[address]
		accInfo.RLock()
		if accInfo.removed {
			err := st.RemoveAccount(address)
			if err != nil {
				return err
			}
		} else if accInfo.updated {
			// First update account in case it needs to be created
			err := st.UpdateAccount(accInfo.account)
			if err != nil {
				return err
			}
			// Sort keys
			var keys binary.Words256
			for key := range accInfo.storage {
				keys = append(keys, key)
			}
			sort.Sort(keys)
			// Update account's storage
			for _, key := range keys {
				value := accInfo.storage[key]
				err := st.SetStorage(address, key, value)
				if err != nil {
					return err
				}
			}

		}
		accInfo.RUnlock()
	}

	return nil
}
```
Cache包含一个Sync方法，需要传入一个实现了Writer接口的类型作为参数，根据removed和updated标志位对账户和存储进行修改

#### acm/account.go

`account.go` 中定义了全局账户权限地址为零值地址，根据公钥/种子密钥新建账户的方法，以及修改账户余额、获取账户代码等账户相关的方法
```go
var GlobalPermissionsAddress = crypto.Address(binary.Zero160)

func NewAccount(pubKey *crypto.PublicKey) *Account {
	return &Account{
		Address:   pubKey.GetAddress(),
		PublicKey: pubKey,
	}
}

func NewAccountFromSecret(secret string) *Account {
	return NewAccount(crypto.PrivateKeyFromSecret(secret, crypto.CurveTypeEd25519).GetPublicKey())
}
```

## execution

`execution` 中包含了执行环境相关代码
### engine

#### account.go

`account.go` 中定义了部署合约代码、部署WASM代码、修改账户状态相关的函数，这些函数被用于[contract.go](#evmcontractgo)的指令执行中

#### accounts.go

`accounts.go` 中定义了获取账户、创建账户、权限检查相关的函数
```go
func EnsurePermission(callFrame *CallFrame, address crypto.Address, perm permission.PermFlag) error {
	hasPermission, err := HasPermission(callFrame, address, perm)
	if err != nil {
		return err
	} else if !hasPermission {
		return errors.PermissionDenied{
			Address: address,
			Perm:    perm,
		}
	}
	return nil
}

// CONTRACT: it is the duty of the contract writer to call known permissions
// we do not convey if a permission is not set
// (unlike in state/execution, where we guarantee HasPermission is called
// on known permissions and panics else)
// If the perm is not defined in the acc nor set by default in GlobalPermissions,
// this function returns false.
func HasPermission(st acmstate.Reader, address crypto.Address, perm permission.PermFlag) (bool, error) {
	acc, err := st.GetAccount(address)
	if err != nil {
		return false, err
	}
	if acc == nil {
		return false, fmt.Errorf("account %v does not exist", address)
	}
	globalPerms, err := acmstate.GlobalAccountPermissions(st)
	if err != nil {
		return false, err
	}
	perms := acc.Permissions.Base.Compose(globalPerms.Base)
	value, err := perms.Get(perm)
	if err != nil {
		return false, err
	}
	return value, nil
}
```
这里权限检查是通过调用[state.go](#statego)中的GlobalAccountPermissions函数来获取作为基准的权限，然后检查该账户设置的基准权限中是否包含目标权限

#### blockchain.go

`blockchain.go` 中定义了一个实现了[Blockchain](#callablego)接口的TestBlockChain类型用于测试

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

...

func (st *CallFrame) NewFrame(cacheOptions ...acmstate.CacheOption) (*CallFrame, error) {
	if st.maxCallStackDepth > 0 && st.maxCallStackDepth == st.callStackDepth {
		return nil, errors.Codes.CallStackOverflow
	}
	return newCallFrame(st.Cache, st.callStackDepth+1, st.maxCallStackDepth,
		append(st.cacheOptions, cacheOptions...)...), nil
}

func (st *CallFrame) Sync() error {
	err := st.Cache.Sync(st.backend)
	if err != nil {
		return errors.AsException(err)
	}
	return nil
}
```
NewCallFrame函数暴露给外界用于创建新的CallFrame，该函数接受两个参数：实现了ReaderWriter接口的类型和任意多个CacheOption，并调用内部函数nameCallFrame来创建CallFrame

newCallFrame会根据传入的参数创建一个[Cache](#state_cachego)，用于缓存账户及存储信息

使用NewFrame创建新的CallFrame时，会继承当前CallFrame的值，并将当前栈高度值加一，使其看起来就像在调用栈中压入了一个新的调用

CallFrame包含一个Sync方法，用于将缓存写回，其内部会调用[Cache](#state_cachego)的Sync()方法执行写回操作


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

```go
func CallFromSite(st State, dispatcher Dispatcher, site CallParams, target CallParams) ([]byte, error) {
	err := EnsurePermission(st.CallFrame, site.Callee, permission.Call)
	if err != nil {
		return nil, err
	}
	// Get the arguments from the memory
	// EVM contract
	err = UseGasNegative(site.Gas, GasGetAccount)
	if err != nil {
		return nil, err
	}
	// since CALL is used also for sending funds,
	// acc may not exist yet. This is an errors.CodedError for
	// CALLCODE, but not for CALL, though I don't think
	// ethereum actually cares
	acc, err := st.CallFrame.GetAccount(target.Callee)
	if err != nil {
		return nil, err
	}
	if acc == nil {
		if target.CallType != exec.CallTypeCall {
			return nil, errors.Codes.UnknownAddress
		}
		// We're sending funds to a new account so we must create it first
		err := st.CallFrame.CreateAccount(site.Callee, target.Callee)
		if err != nil {
			return nil, err
		}
		acc, err = st.CallFrame.GetAccount(target.Callee)
		if err != nil {
			return nil, err
		}
	}

	// Establish a stack frame and perform the call
	childCallFrame, err := st.CallFrame.NewFrame()
	if err != nil {
		return nil, err
	}
	childState := State{
		CallFrame:  childCallFrame,
		Blockchain: st.Blockchain,
		EventSink:  st.EventSink,
	}
	// Ensure that gasLimit is reasonable
	if site.Gas.Cmp(target.Gas) < 0 {
		// EIP150 - the 63/64 rule - rather than errors.CodedError we pass this specified fraction of the total available gas
		gas := new(big.Int)
		target.Gas.Sub(site.Gas, gas.Div(site.Gas, big64))
	}
	// NOTE: we will return any used gas later.
	site.Gas.Sub(site.Gas, target.Gas)

	// Setup callee params for call type
	target.Origin = site.Origin

	// Set up the caller/callee context
	switch target.CallType {
	case exec.CallTypeCall:
		// Calls contract at target from this contract normally
		// Value: transferred
		// Caller: this contract
		// Storage: target
		// Code: from target
		target.Caller = site.Callee

	case exec.CallTypeStatic:
		// Calls contract at target from this contract with no state mutation
		// Value: not transferred
		// Caller: this contract
		// Storage: target (read-only)
		// Code: from target
		target.Caller = site.Callee

		childState.CallFrame.ReadOnly()
		childState.EventSink = exec.NewLogFreeEventSink(childState.EventSink)

	case exec.CallTypeCode:
		// Calling this contract from itself as if it had the code at target
		// Value: transferred
		// Caller: this contract
		// Storage: this contract
		// Code: from target

		target.Caller = site.Callee
		target.Callee = site.Callee

	case exec.CallTypeDelegate:
		// Calling this contract from the original caller as if it had the code at target
		// Value: not transferred
		// Caller: original caller
		// Storage: this contract
		// Code: from target

		target.Caller = site.Caller
		target.Callee = site.Callee

	default:
		// Switch should be exhaustive so we should reach this
		panic("invalid call type")
	}

	dispatch := dispatcher.Dispatch(acc)
	if dispatch == nil {
		return nil, errors.Errorf(errors.Codes.NotCallable, "cannot call: %v", acc.Address)
	}
	returnData, err := dispatch.Call(childState, target)

	if err == nil {
		// Sync error is a hard stop
		err = childState.CallFrame.Sync()
	}

	// Handle remaining gas.
	//site.Gas.Add(site.Gas, target.Gas)
	return returnData, err
}
```
CallFromSite在执行合约代码中的CALL/CALLCODE/DELEGATECALL/STATICCALL指令时被调用，用于由合约代码创建新的合约账户，其执行流程大致如下：

调用[EnsurePermission](#accountsgo)检查当前合约是否拥有创建合约的权限 -- 减去创建账户需要消耗的gas值 -- 检查目标账户是否存在，若不存在，则根据调用类型选择创建新账户或是报错 -- 调用[NewFrame](#call_framego)创建一个childCallFrame -- 由childCallFrame创建childState -- 检查gas值 -- 根据调用类型设置目标账户参数 -- 调用dispatcher.Diapatch方法返回一个[Callable](#callablego)实现类型 -- 调用Call方法执行 -- 调用[Sync](#call_framego)方法写回账户状态

这里的dispatcher实际为[EVM](#evmgo)中的externalDispatcher

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

//TODO
`Callable` 接口，仅包含一个 `Call` 函数，合约调用/原生合约/原生函数/wasm调用都需要实现该接口，实现了该接口的类型有[evm contract](#evmgo)、[native contract](#contractgo)、[wasm contract]()


#### dispatcher.go

`dispatcher.go` 中定义了dispatcher的相关接口和函数，用于evm/wasm/native contract之间的相互调用

```go
type Dispatcher interface {
	// If this Dispatcher is capable of dispatching this account (e.g. if it has the correct bytecode) then return a
	// Callable that wraps the function, otherwise return nil
	Dispatch(acc *acm.Account) Callable
}
```
Dispatcher接口包含一个Dispatch函数，它接收一个账户类型作为参数，返回一个实现了[Callable](#callablego)接口的类型，也就是合约/原生合约/原生函数/wasm，具体返回那种类型会根据传入的地址进行判定
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
ExternalDispatcher接口包含了一个Dispatcher和一个用于设置Dispatcher的SetExternals函数，[EVM](#evmgo)中包含了一个匿名Externals类型，用于设置WASM
```go
type Dispatchers []Dispatcher

func (ds Dispatchers) Dispatch(acc *acm.Account) Callable {
	for _, d := range ds {
		callable := d.Dispatch(acc)
		if callable != nil {
			return callable
		}
	}
	return nil
}
```
Dispatchers类型实际为Dispatcher切片，它的Dispatch方法会遍历切片中保存的所有Dispatcher，并返回一个有效的Callable实现类型，由于账户地址是唯一的，也就是一个账户不可能既对应一个native contract，又对应一般的合约，因此Dispatchers中的顺序不重要，即在Dispatcher非空的情况下，只能返回零个或一个Callable

#### gas.go

`gas.go` 中定义了各种操作需要消耗的gas值以及计算gas值的函数，在burrow中，操作消耗的gas值均为1
```go
const (
	GasSha3          uint64 = 1
	GasGetAccount    uint64 = 1
	GasStorageUpdate uint64 = 1
	GasCreateAccount uint64 = 1

	GasBaseOp  uint64 = 0 // TODO: make this 1
	GasStackOp uint64 = 1

	GasEcRecover     uint64 = 1
	GasSha256Word    uint64 = 1
	GasSha256Base    uint64 = 1
	GasRipemd160Word uint64 = 1
	GasRipemd160Base uint64 = 1
	GasExpModWord    uint64 = 1
	GasExpModBase    uint64 = 1
	GasIdentityWord  uint64 = 1
	GasIdentityBase  uint64 = 1
)

// Try to deduct gasToUse from gasLeft.  If ok return false, otherwise
// set err and return true.
func UseGasNegative(gasLeft *big.Int, gasToUse uint64) errors.CodedError {
	delta := new(big.Int).SetUint64(gasToUse)
	if gasLeft.Cmp(delta) >= 0 {
		gasLeft.Sub(gasLeft, delta)
	} else {
		return errors.Codes.InsufficientGas
	}
	return nil
}
```

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

#### abi

##### primitives.go

```go
// EVM Solidity calls and return values are packed into
// pieces of 32 bytes, including a bool (wasting 255 out of 256 bits)
const ElementSize = 32

type EVMType interface {
	GetSignature() string
	getGoType() interface{}
	pack(v interface{}) ([]byte, error)
	unpack(data []byte, offset int, v interface{}) (int, error)
	Dynamic() bool
	ImplicitCast(o EVMType) bool
}
```
EVM Solidity合约调用以及返回值需要编码为32字节长的片段，EVMType中的pack和unpack方法即为每种类型具体的编码与解码方法

这里定义了EVMBool、EVMUint、EVMInt、EVMAddress、EVMBytes、EVMString、EVMFixed七种类型，他们均实现了EVMType接口，除了EVMFixed类型，其余类型均实现了pack/unpack方法

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

EVM结构体类型中还包括一个externalDispatcher字段，其类型为Dispatcher，可以看到在新建evm实例时该字段被设置为一个[Dispatchers](#dispatchergo)类型，包含了上面提到的外部dispatcher，原生合约以及evm本身，因为EVM也实现了Dispatch方法：
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
EVM的Dispatch方法返回的是一个[Contract](#evmcontractgo)类型，Dispatchers的Dispatch方法会遍历所有Dispatcher并返回一个有效的Callable

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