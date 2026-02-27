---

# Tetcore Virtual Machine (TVM)

## Instruction Set and Execution Specification

### Version 1.0 — Deterministic Core

---

# 0. Purpose

The Tetcore Virtual Machine (TVM) is the deterministic execution environment for Tetcore smart contracts.

TVM is:

* Sovereign (not derived from EVM)
* Deterministic
* Gas-metered
* Capability-restricted
* Strongly typed at runtime boundary
* Replay-stable across nodes

TVM executes Tetcore Bytecode (TBC), produced by the Tetcore Contract Language (TCL) compiler.

---

# 1. Design Goals

1. Determinism across all nodes
2. No floating point arithmetic
3. No nondeterministic syscalls
4. Explicit gas accounting
5. Memory-bounded execution
6. Capability-based host interface
7. Versioned instruction set

---

# 2. Execution Model

A contract execution instance is:

```
ExecutionContext =
(
  caller: Address,
  contract: Address,
  value: u128,
  gas_limit: u64,
  gas_remaining: u64,
  input: bytes,
  block_height: u64,
  capabilities: CapabilitySet
)
```

Execution produces:

```
ExecutionResult =
(
  status: Success | Revert | OutOfGas,
  return_data: bytes,
  gas_used: u64,
  events: Vec<Event>
)
```

---

# 3. TVM Architecture

TVM is a register-based virtual machine.

## 3.1 Core Components

* Program Counter (PC)
* Gas Counter
* Register File (fixed 32 registers, 256-bit width)
* Stack (bounded)
* Linear Memory (bounded)
* Contract Storage Interface
* Host Interface (capabilities only)

---

# 4. Data Types

TVM supports the following primitive types:

* u8
* u16
* u32
* u64
* u128
* u256
* bool
* bytes (dynamic, bounded)
* Address (32 bytes)
* Hash (32 bytes)

No floating point types exist.

---

# 5. Gas Model

Each instruction has:

```
GasCost = BaseCost + MemoryCost + StorageCost
```

Rules:

* Arithmetic: low constant cost
* Memory expansion: quadratic incremental cost
* Storage write: high cost
* Storage read: moderate cost
* Hashing: cost proportional to input size
* Contract call: base + gas forwarded

If gas_remaining < instruction_cost:
→ Execution halts with OutOfGas.

Gas is deducted before instruction execution.

---

# 6. Instruction Set Overview

TVM instructions are grouped into:

1. Control Flow
2. Arithmetic
3. Comparison & Logic
4. Memory
5. Storage
6. Cryptography
7. Contract Calls
8. Host Capability Calls
9. Event Emission
10. Termination

---

# 7. Control Flow Instructions

| Opcode          | Description      |
| --------------- | ---------------- |
| NOP             | No operation     |
| JMP offset      | Jump relative    |
| JZ reg, offset  | Jump if zero     |
| JNZ reg, offset | Jump if nonzero  |
| CALL offset     | Internal call    |
| RET             | Return from call |

All jumps must remain within contract bytecode bounds.

---

# 8. Arithmetic Instructions

All arithmetic is wrapping unless marked SAFE.

| Opcode          | Description |
| --------------- | ----------- |
| ADD r1, r2 → r3 |             |
| SUB r1, r2 → r3 |             |
| MUL r1, r2 → r3 |             |
| DIV r1, r2 → r3 |             |
| MOD r1, r2 → r3 |             |
| ADD_SAFE        |             |
| SUB_SAFE        |             |

Division by zero → Revert.

SAFE variants revert on overflow.

---

# 9. Comparison & Logic

| Opcode | Description |
| ------ | ----------- |
| EQ     |             |
| NEQ    |             |
| GT     |             |
| LT     |             |
| GTE    |             |
| LTE    |             |
| AND    |             |
| OR     |             |
| XOR    |             |
| NOT    |             |

All operate on 256-bit registers.

---

# 10. Memory Model

Memory is linear and byte-addressable.

Instructions:

| Opcode              | Description |
| ------------------- | ----------- |
| MLOAD offset → reg  |             |
| MSTORE reg → offset |             |
| MCOPY src, dst, len |             |
| MSIZE → reg         |             |

Memory expansion costs gas.

Memory cleared between transactions.

---

# 11. Contract Storage Model

Each contract has namespaced storage:

```
StorageKey = H(contractAddress || userKey)
```

Instructions:

| Opcode           | Description |
| ---------------- | ----------- |
| SLOAD key → reg  |             |
| SSTORE reg → key |             |

SSTORE rules:

* Zero → Nonzero: high gas
* Nonzero → Zero: gas refund
* Nonzero → Nonzero: medium gas

Storage writes are persisted only if execution ends in Success.

---

# 12. Cryptographic Instructions

| Opcode                                 | Description |
| -------------------------------------- | ----------- |
| HASH len, offset → reg                 |             |
| VERIFY_SIG pubkey, msg_hash, sig → reg |             |
| KECCAK len, offset → reg               |             |

VERIFY_SIG uses Ed25519 verification.

---

# 13. Contract Interaction

| Opcode                                             | Description |
| -------------------------------------------------- | ----------- |
| CALL contract, gas, value, input_offset, input_len |             |
| STATICCALL contract, gas, input_offset, input_len  |             |
| DELEGATECALL (optional, v2)                        |             |

Rules:

* Calls consume caller gas.
* Calls create isolated execution context.
* Revert bubbles up unless caught.

---

# 14. Capability-Based Host Interface

Contracts cannot access kernel state directly.

Instead they receive capabilities:

CapabilitySet may include:

* TransferTokens
* ReadModelRegistry
* RegisterPolicy
* EmitProtocolEvent
* VerifyReceipt
* QueryVault
* GovernancePropose

Host call instruction:

```
HOSTCALL capability_id, input_offset, input_len
```

Host must:

* Validate capability exists
* Execute deterministic logic
* Charge gas

---

# 15. Event Emission

| Opcode                                  | Description |
| --------------------------------------- | ----------- |
| EMIT topic_count, data_offset, data_len |             |

Events are appended to block event log.

Events do not modify state.

---

# 16. Termination

| Opcode             | Description |
| ------------------ | ----------- |
| RETURN offset, len |             |
| REVERT offset, len |             |
| STOP               |             |

STOP = RETURN empty.

---

# 17. Determinism Constraints

TVM must not access:

* System time
* Randomness
* OS entropy
* Network IO
* Disk IO outside storage interface

All execution is pure relative to state and input.

---

# 18. Contract Deployment

Deployment transaction:

```
DeployContract(code, init_args)
```

Execution:

1. Instantiate VM with code
2. Run init function
3. If success:

   * Assign new Address
   * Store codeHash
   * Persist storage
4. If revert:

   * No state change

---

# 19. Bytecode Format (TBC)

Tetcore Bytecode format:

```
Header:
  - Version
  - InstructionCount
  - StaticGasEstimate
  - CodeHash

Body:
  - Instruction stream

Footer:
  - Metadata (optional)
```

Bytecode is immutable once deployed.

---

# 20. Versioning

Instruction set is versioned.

```
TVM v1.0
TVM v1.1
TVM v2.0
```

Contracts compiled for version X cannot run on incompatible versions.

---

# 21. Safety Invariants

1. Gas never negative.
2. Storage updates only on Success.
3. No memory access beyond bounds.
4. No infinite loops (gas bound).
5. Deterministic execution across nodes.

---

# 22. Execution Guarantees

For any input (state, tx):

[
Execute(state, tx) = state'
]

All honest nodes produce identical result.

---

# Conclusion

The Tetcore Virtual Machine is:

* Deterministic
* Gas-bounded
* Capability-controlled
* Strongly isolated
* Versioned
* Sovereign

It forms the programmable execution layer of Tetcore.

---
