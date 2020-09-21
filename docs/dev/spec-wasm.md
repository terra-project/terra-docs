# WASM

The WASM module implements the execution environment for WebAssembly smart contracts, powered by [CosmWasm](https://cosmwasm.com).

## Concepts

### Smart Contracts

Smart contracts are autonomous agents that are able to interact with other entities on the Terra blockchain, such as human-owned accounts, validators, and other smart contracts. Each smart contract has:

- a unique **contract address** with an account that holds funds
- a **code ID**, where its logic is defined
- its own **key-value store**, where it can persist and retrieve data

#### Contract Address

Upon instantiation, each contract is automatically assigned a Terra account address, called the _contract address_. The address is procedurally generated on-chain without an accompanying private / public key pair, and can be completely determined by the contract's number order of existence. For instance, on two separate Terra networks, the first contract will always be assigned the address `terra18vd8fpwxzck93qlwghaj6arh4p7c5n896xzem5`, and similarly for the second, third, and so forth.

#### Code ID

On Terra, code upload and contract creation occur as separate events. A smart contract writer first uploads WASM bytecode onto the blockchain to obtain a _code ID_, which they can then use to initialize an instance of that contract. This scheme promotes efficient storage, as most contracts share the same underlying logic and vary only in their initial configuration. Vetted, high quality contracts for common use cases such as fungible tokens and multisig wallets can be easily reused without the need to upload new code.

#### Key-Value Store

Each smart contract is given its own dedicated keyspace in LevelDB, prefixed by the contract address. Contract code is safely sandboxed and can only can set and delete new keys and values within its assigned keyspace.

### Interaction

Users can interact with smart contracts in several ways.

#### Instantiation

A user can instantiate a new smart contract by sending a `MsgInstantiateContract`. In it, the user is able to:

- specify code will be used for the contract via a code ID
- define the initial parameters / configuration through an `InitMsg`
- provide the new contract's account with some initial funds
- denote whether the contract is migratable (can change code IDs)

The `InitMsg` is a JSON message whose expected format is defined in the contract's code. Every contract contains a section that defines how to set up the initial state depending on the provided `InitMsg`.

By default, the user that creates the contract is its initial owner.

#### Execution

A user can execute a smart contract to invoke one of its defined functions by sending a `MsgExecuteContract`. In it, the user is able to:

- specify which function to call with a `HandleMsg`
- send funds to the contract, which may be expected during execution

The `HandleMsg` is a JSON message that containing function call arguments and gets routed to the appropriate handling logic. From there, the contract executes the function's instructions, during which the contract's own state can modified. The contract can only modify outside state (such as state in other contracts or modules) after its own execution has ended, by returning a list of blockchain messages such as `MsgSend` and `MsgSwap`. These messages are appended to the same transaction as the `MsgExecuteContract`, and if any of the messages are invalid, the whole transaction is invalidated.

#### Migration

If a user is the contract's owner, and a contract is instantiated as migratable, they can issue a `MsgMigrateContract` to reset its code ID to a new one. The migration is be parameterized with a `MigrateMsg`, a JSON message.

#### Transfer of Ownership

The current owner of the smart contract can re-assign a new owner to the contract with `MsgUpdateContractOwner`.

#### Query

Contracts can define query functions, or read-only operations meant for data-retrieval. This allows contracts to expose rich, custom data endpoints with JSON responses instead of raw bytes from the low-level key-value store. Because the blockchain state cannot be changed, the node can directly run the query without a transaction.

Users can specify which query function alongside any arguments with a JSON `QueryMsg`. Even though there is no gas fee, the query function's execution is capped by gas determined by metered execution (which is not charged) as a form of spam-protection.

### Wasmer VM

The actual execution of WASM bytecode is performed by [wasmer](https://github.com/wasmerio/wasmer), which provides a lightweight sandboxed runtime with metered execution to account for the resource cost of computation.

#### Gas Meter

In addition to the regular gas fees incurred from creating the transaction, Terra also calculates a separate gas when executing smart contract code. This is tracked by the **gas meter**, which is during the execution of every opcode and gets translated back to native Terra gas via a constant multiplier (currently set to 100).

## Data

### CodeInfo

```go
type CodeInfo struct {
	CodeID   uint64           `json:"code_id"`
	CodeHash core.Base64Bytes `json:"code_hash"`
	Creator  sdk.AccAddress   `json:"creator"`
}
```

### ContractInfo

```go
type ContractInfo struct {
	Address    sdk.AccAddress   `json:"address"`
	Owner      sdk.AccAddress   `json:"owner"`
	CodeID     uint64           `json:"code_id"`
	InitMsg    core.Base64Bytes `json:"init_msg"`
	Migratable bool             `json:"migratable"`
}
```

## State

### LastCodeID

A counter for the last uploaded code ID.

- type: `uint64`

### LastInstanceID

A counter for the last instantiated contract number.

- type: `uint64`

### Code

Maps a code ID to `CodeInfo` entry.

- type: `map[uint64]CodeInfo`

### ContractInfo

Maps contract address to its corresponding `ContractInfo`.

- type: `map[bytes]ContractInfo`

### ContractStore

Maps contract address to its dedicated KVStore.

- type: `map[bytes]KVStore`

## Message Types

### MsgStoreCode

```go
type MsgStoreCode struct {
	Sender sdk.AccAddress `json:"sender" yaml:"sender"`
	// WASMByteCode can be raw or gzip compressed
	WASMByteCode core.Base64Bytes `json:"wasm_byte_code" yaml:"wasm_byte_code"`
}
```

::: details JSON Example

```json
{
  "type": "wasm/MsgStoreCode",
  "value": {
    "sender": "terra...",
    "wasm_byte_code": "QmFzZTY0LWVuY29kZWQgV0FTTSBiaW5hcnk="
  }
}
```

:::

### MsgInstantiateContract

```go
type MsgInstantiateContract struct {
	Owner      sdk.AccAddress   `json:"owner" yaml:"owner"`
	CodeID     uint64           `json:"code_id" yaml:"code_id"`
	InitMsg    core.Base64Bytes `json:"init_msg" yaml:"init_msg"`
	InitCoins  sdk.Coins        `json:"init_coins" yaml:"init_coins"`
	Migratable bool             `json:"migratable" yaml:"migratable"`
}
```

::: details JSON Example

```json
{
  "type": "wasm/MsgInstantiateContract",
  "value": {
    "owner": "terra...",
    "code_id": "23",
    "init_msg": "eyJlbmNvZGVkIjogIkpTT04gbWVzc2FnZSJ9",
    "init_coins": [
      {
        "denom": "uluna",
        "amount": "999"
      }
    ],
    "migratable": false
  }
}
```

:::

### MsgExecuteContract

```go
type MsgExecuteContract struct {
	Sender     sdk.AccAddress   `json:"sender" yaml:"sender"`
	Contract   sdk.AccAddress   `json:"contract" yaml:"contract"`
	ExecuteMsg core.Base64Bytes `json:"execute_msg" yaml:"execute_msg"`
	Coins      sdk.Coins        `json:"coins" yaml:"coins"`
}
```

::: details JSON Example

```json
{
  "type": "wasm/MsgExecuteContract",
  "value": {
    "sender": "terra...",
    "contract": "terra...",
    "execute_msg": "eyJlbmNvZGVkIjogIkpTT04gbWVzc2FnZSJ9",
    "coins": [
      {
        "denom": "uluna",
        "amount": "999"
      }
    ]
  }
}
```

:::

### MsgMigrateContract

```go
type MsgMigrateContract struct {
	Owner      sdk.AccAddress   `json:"owner" yaml:"owner"`
	Contract   sdk.AccAddress   `json:"contract" yaml:"contract"`
	NewCodeID  uint64           `json:"new_code_id" yaml:"new_code_id"`
	MigrateMsg core.Base64Bytes `json:"migrate_msg" yaml:"migrate_msg"`
}
```

::: details JSON Example

```json
{
  "type": "wasm/MsgMigrateContract",
  "value": {
    "owner": "terra...",
    "contract": "terra...",
    "new_code_id": "45",
    "migrate_msg": "eyJlbmNvZGVkIjogIkpTT04gbWVzc2FnZSJ9"
  }
}
```

:::

### MsgUpdateContractOwner

```go
type MsgUpdateContractOwner struct {
	Owner    sdk.AccAddress `json:"owner" yaml:"owner"`
	NewOwner sdk.AccAddress `json:"new_owner" yaml:"new_owner"`
	Contract sdk.AccAddress `json:"contract" yaml:"contract"`
}
```

::: details JSON Example

```json
{
  "type": "wasm/MsgUpdateContractOwner",
  "value": {
    "owner": "terra...",
    "new_owner": "terra...",
    "contract": "terra..."
  }
}
```

:::

## Transitions

## Parameters

The subspace for the WASM module is `wasm`.

```go
type Params struct {
	MaxContractSize    uint64 `json:"max_contract_size" yaml:"max_contract_size"`
	MaxContractGas     uint64 `json:"max_contract_gas" yaml:"max_contract_gas"`
	MaxContractMsgSize uint64 `json:"max_contract_msg_size" yaml:"max_contract_msg_size"`
}
```

### MaxContractSize

Maximum contract bytecode size, in bytes.

- type: `uint64`

### MaxContractGas

Maximum contract gas consumption during any execution.

- type: `uint64`

### MaxContractMsgSize

Maximum contract message size, in bytes.

- type: `uint64`

## Events