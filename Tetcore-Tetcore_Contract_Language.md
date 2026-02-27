# Tetcore Contract Language (TCL)

## Language Specification

### Version 1.0 — Deterministic, Capability-Based Contracts

---

# 0. Purpose

Tetcore Contract Language (TCL) is a sovereign smart contract language designed for deterministic execution on the Tetcore Virtual Machine (TVM).

TCL is:

* strongly typed
* deterministic
* resource-metered (gas-aware)
* capability-based (no ambient authority)
* optimized for protocol-native primitives (models, receipts, vaults, governance)
* compiled to Tetcore Bytecode (TBC)

TCL is not derived from Solidity and is not EVM-targeted.

---

# 1. Design Goals

1. Determinism across all nodes
2. Safe-by-default execution
3. No floating point or nondeterministic primitives
4. Explicit state access through storage APIs
5. Explicit authority via declared capabilities
6. Predictable compilation and reproducible builds
7. Practical ergonomics (Rust-like where helpful)

---

# 2. Execution and Program Model

A TCL contract is a package containing:

* type declarations
* storage schema
* event declarations
* entrypoints
* helper functions

Every contract defines these entrypoints:

* `init(ctx: Context, args: Bytes) -> Result<()>`
* `call(ctx: Context, method: Method, args: Bytes) -> Result<Bytes>`

Optionally, contracts may define typed method dispatch via `#[entry]` functions (see §8).

---

# 3. Lexical Structure

## 3.1 Character Set

UTF-8 source text. Identifiers restricted to ASCII letters, digits, `_`.

## 3.2 Whitespace and Comments

* Whitespace separates tokens.
* Line comments: `// ...`
* Block comments: `/* ... */` (nesting allowed).

## 3.3 Keywords (reserved)

`contract, storage, event, fn, pub, let, mut, if, else, match, for, while, loop, break, continue, return, revert, true, false, struct, enum, impl, use, const, capability, require, emit`

No keyword may be used as an identifier.

---

# 4. Types

TCL is statically typed with explicit conversions.

## 4.1 Primitive Types

* `u8, u16, u32, u64, u128, u256`
* `bool`
* `Address` (32 bytes)
* `Hash` (32 bytes)
* `Bytes` (dynamically sized, bounded)
* `String` (UTF-8, bounded; optional v1 feature)

## 4.2 Composite Types

* `struct` with named fields
* `enum` with tagged variants
* Fixed arrays: `[T; N]`
* Vectors: `Vec<T>` (bounded)
* Maps: `Map<K, V>` (backed by storage API; see §7)

## 4.3 Option and Result

Built-ins:

* `Option<T> = Some(T) | None`
* `Result<T> = Ok(T) | Err(ErrorCode)`

`ErrorCode` is `u32` in v1.

---

# 5. Determinism Rules

TCL programs must be deterministic:

Forbidden:

* reading system time
* randomness
* network IO
* filesystem IO
* floating point
* nondeterministic iteration over hash maps unless order is defined

Allowed:

* hashing (`hash(bytes)`)
* signature verification (`verify_sig(...)`)
* deterministic storage reads/writes
* deterministic loops bounded by gas

---

# 6. Memory and Gas Model

TCL compilation targets TVM’s gas model.

Gas costs arise from:

* instruction execution
* hashing input sizes
* memory growth
* storage reads/writes
* host capability calls

TCL provides compile-time estimations:

* `#[gas_hint(max = ...)]` optional annotation for auditing.

---

# 7. Storage Model (Contract State)

## 7.1 Storage Declaration

Each contract declares storage schema:

```tcl
storage {
  owner: Address;
  total_shares: u128;
  shares: Map<Address, u128>;
}
```

Semantics:

* primitive fields are stored at fixed keys derived from field identifiers
* maps are stored as key-spaces under a field prefix

## 7.2 Storage API

Compiler lowers storage access to TVM `SLOAD/SSTORE`.

TCL primitives:

* `storage.owner.get() -> Address`
* `storage.owner.set(value: Address)`
* `storage.shares.get(key: Address) -> Option<u128>`
* `storage.shares.set(key: Address, value: u128)`
* `storage.shares.remove(key: Address)`

Storage writes persist only on successful execution.

---

# 8. Entrypoints and Dispatch

## 8.1 Standard Entrypoints

Required functions:

```tcl
pub fn init(ctx: Context, args: Bytes) -> Result<()> { ... }
pub fn call(ctx: Context, method: Method, args: Bytes) -> Result<Bytes> { ... }
```

Where:

* `Context` provides caller, contract, value, gas, and capabilities
* `Method` identifies which function is invoked (see §8.3)

## 8.2 Typed Entrypoints

Contracts may declare typed methods:

```tcl
#[entry("deposit")]
pub fn deposit(ctx: Context, amount: u128) -> Result<()> { ... }

#[entry("claim")]
pub fn claim(ctx: Context) -> Result<u128> { ... }
```

Compiler generates the `call` dispatcher automatically.

## 8.3 Method Identifier

In v1, method selectors are:

[
selector = H("TCL:METHOD:v1" || method_name)[0..4]
]

A 32-bit selector (u32). `Method` is `u32`.

---

# 9. Control Flow

Supported statements:

* variable bindings: `let`, `let mut`
* `if/else`
* `match` (exhaustive)
* loops: `for`, `while`, `loop`
* `break`, `continue`
* `return`, `revert`

`revert` aborts execution and rolls back storage changes.

---

# 10. Functions

Syntax:

```tcl
fn add(a: u128, b: u128) -> u128 { a + b }
```

Functions may be `pub` or private.

No recursion in v1 (recommended) unless explicitly enabled with compiler limits.

---

# 11. Arithmetic and Safety

Default arithmetic is wrapping on overflow **only for u256**.

For all other integer types, overflow causes revert unless explicitly marked:

* `wrapping_add`
* `wrapping_mul`
* etc.

Division by zero always reverts.

---

# 12. Events

Contracts declare events:

```tcl
event Deposit {
  from: Address,
  amount: u128,
}
```

Emit:

```tcl
emit Deposit { from: ctx.caller, amount };
```

Events are appended to the block log.

Events do not modify state.

---

# 13. Capabilities and Host Calls

TCL has no ambient authority. Any privileged action requires explicit capabilities.

## 13.1 Declaring Capabilities

```tcl
capability TransferTokens;
capability ReadModelRegistry;
capability VerifyReceipt;
```

## 13.2 Requesting Capabilities

Contracts must declare required capabilities at compile time:

```tcl
#[requires(TransferTokens)]
pub fn payout(ctx: Context, to: Address, amount: u128) -> Result<()> { ... }
```

VM enforces that the capability is present in the execution context.

## 13.3 Host Call Interface

TCL provides builtins:

* `host.transfer(to: Address, amount: u128) -> Result<()>`
* `host.model.get(model_id: Hash) -> Result<ModelInfo>`
* `host.receipt.verify(itx_id: Hash) -> Result<bool>`
* `host.vault.query(model_id: Hash) -> Result<VaultInfo>`

All host functions:

* are deterministic
* are metered
* must be permitted by capabilities

---

# 14. Standard Library (v1)

Deterministic-only modules:

* `std::hash`
* `std::bytes`
* `std::address`
* `std::math` (integer only)
* `std::codec` (fixed deterministic encoding)

No floating-point or system APIs.

---

# 15. Encoding and ABI

TCL uses a deterministic ABI called **TABI**.

## 15.1 TABI Encoding Rules

* integers: little-endian fixed width
* `Bytes`: length-prefixed `u32` + payload
* `Vec<T>`: length-prefixed + concatenated elements
* `struct`: fields in declared order
* `enum`: tag `u8` + payload

## 15.2 Selector + Args

A method call payload:

```text
[selector:u32][args:TABI...]
```

Return values are TABI encoded.

---

# 16. Compilation Pipeline

1. Parse TCL → AST
2. Typecheck → typed IR
3. Lower to TVM IR
4. Gas estimation + bounds checks
5. Emit TBC bytecode
6. Produce metadata: ABI, selectors, source map hashes

Reproducible builds require:

* fixed compiler version
* fixed optimization level
* fixed metadata ordering

---

# 17. Safety Rules

Required compiler checks:

* type soundness
* storage key determinism
* bounded memory
* forbidden nondeterminism
* capability annotation correctness
* ABI determinism

---

# 18. Example Contract (Vault)

```tcl
contract Vault {

  capability TransferTokens;

  storage {
    owner: Address;
    total_shares: u128;
    shares: Map<Address, u128>;
    accrued: u128;
  }

  pub fn init(ctx: Context, args: Bytes) -> Result<()> {
    storage.owner.set(ctx.caller);
    storage.total_shares.set(0);
    storage.accrued.set(0);
    Ok(())
  }

  #[entry("deposit")]
  pub fn deposit(ctx: Context, amount: u128) -> Result<()> {
    require(amount > 0, 1);
    // token movement handled by host, if allowed by policy
    // host.transfer(contract, amount) would be executed by transaction value rules or explicit transfer capability.
    let prev = storage.shares.get(ctx.caller).unwrap_or(0);
    storage.shares.set(ctx.caller, prev + amount);
    storage.total_shares.set(storage.total_shares.get() + amount);
    emit Deposit { from: ctx.caller, amount };
    Ok(())
  }

  event Deposit { from: Address, amount: u128 }
}
```

---

# 19. Versioning

TCL is versioned independently:

* TCL 1.0 targets TVM 1.0
* Future versions may add:

  * generics
  * stronger borrow rules
  * verified contracts
  * formal proof annotations

---

# Conclusion

TCL is a sovereign contract language built for deterministic infrastructure-grade intelligence networks.

It is:

* strongly typed
* capability-controlled
* gas-metered
* storage-deterministic
* VM-native

