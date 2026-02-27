# Tetcore

## Formal Mathematical Refinement as a Compositional State Machine System

### Version 1.0

---

# 0. Objective

This document refines Tetcore into a **compositional state machine** specification: a mathematically precise model where

* global behavior is defined by **composition** of smaller machines (“modules”),
* each module exposes **interfaces** and **invariants**,
* the Protocol Kernel is the **composition operator** enforcing determinism, validity, and atomicity.

This refinement is suitable for:

* formal verification
* reference implementations
* correctness proofs
* interoperability standards

---

# 1. Core Mathematical Objects

## 1.1 Sets and Types

Let:

* ( \mathbb{B}^n ) be bitstrings of length (n)
* ( \mathbb{N} ) natural numbers
* ( \mathsf{Addr} \subseteq \mathbb{B}^{256} ) (32-byte addresses)
* ( \mathsf{Hash} \subseteq \mathbb{B}^{256} )
* ( \mathsf{Sig} \subseteq \mathbb{B}^{512} )
* ( \mathsf{Bytes} = \bigcup_{n \in \mathbb{N}} \mathbb{B}^{8n} )

Let transaction types be a tagged union:
[
\mathsf{TxKind} = \bigsqcup_{k \in K} \mathsf{Payload}_k
]

A signed transaction:
[
\mathsf{Tx} = (\mathsf{from}:\mathsf{Addr},\ \mathsf{nonce}:\mathbb{N},\ \mathsf{kind}:\mathsf{TxKind},\ \mathsf{sig}:\mathsf{Sig})
]

A block is an ordered list:
[
\mathsf{Block} = (\mathsf{height}:\mathbb{N},\ \mathsf{txs}:[\mathsf{Tx}])
]

---

## 1.2 Cryptographic Axioms

Let:

* ( \mathsf{H}:\mathsf{Bytes}\to\mathsf{Hash} ) be a cryptographic hash function
* ( \mathsf{Verify}:\mathsf{Addr}\times\mathsf{Hash}\times\mathsf{Sig}\to{\top,\bot} )

### Address derivation axiom

Each address (a) corresponds to a public key (pk) via a canonical derivation:
[
a = \mathsf{AddrOf}(pk) = \mathsf{H}(\texttt{"TETCORE:ADDR:v1"}\ | \ pk)
]

### Signature soundness axiom

For honest keypairs, valid signatures verify; forging is computationally infeasible.

---

# 2. Tetcore as a State Transition System

Tetcore is modeled as a deterministic state transition system (STS):

[
\mathcal{T} = (\mathsf{State},\ \mathsf{Input},\ \Delta,\ \mathsf{Init})
]

Where:

* ( \mathsf{State} ) is the set of valid global states
* ( \mathsf{Input} = \mathsf{Block} )
* ( \Delta:\mathsf{State}\times\mathsf{Block}\to\mathsf{State} ) is the deterministic transition function
* ( \mathsf{Init}\in \mathsf{State} ) is genesis state

Determinism requirement:
[
\forall S,B:\ \Delta(S,B)\ \text{is a single-valued function}
]

---

# 3. Compositional State Machine Architecture

Tetcore’s key refinement: **global state is a product of module states**, and the transition function is **composition of module transition functions** constrained by invariants.

---

## 3.1 Module State Product

Let there be (n) modules:
[
\mathcal{M}_1,\dots,\mathcal{M}_n
]

Each module has:

* state space ( \mathsf{State}_i )
* input space ( \mathsf{Input}_i )
* transition ( \delta_i:\mathsf{State}_i\times\mathsf{Input}_i\to\mathsf{State}_i )
* invariants ( \mathsf{Inv}_i \subseteq \mathsf{State}_i )

Global state is a product:
[
\mathsf{State} = \prod_{i=1}^{n}\mathsf{State}_i
]

Let:
[
S = (S_1,\dots,S_n)
]

Global invariants:
[
\mathsf{Inv}(S) \iff \bigwedge_{i=1}^{n} \mathsf{Inv}*i(S_i)\ \wedge\ \mathsf{Inv}*{cross}(S)
]

---

## 3.2 Kernel as a Composition Operator

The Protocol Kernel provides:

1. Transaction validity filtering
2. Ordered application
3. Atomicity across module updates
4. Deterministic conflict resolution

Define a per-transaction transition:
[
\Delta_{\mathsf{tx}}:\mathsf{State}\times\mathsf{Tx}\to\mathsf{State}
]

Then block transition is a fold:
[
\Delta(S,B) = \mathsf{foldl}(\Delta_{\mathsf{tx}},\ S,\ B.\mathsf{txs})
]

---

# 4. The Module Interface (Algebraic Form)

Each module ( \mathcal{M}_i ) is defined by a quintuple:

[
\mathcal{M}_i = (\mathsf{State}_i,\ \mathsf{Msg}_i,\ \mathsf{Apply}_i,\ \mathsf{Inv}_i,\ \mathsf{Exports}_i)
]

Where:

* ( \mathsf{Msg}_i ) is the set of messages the module can receive (derived from tx kinds)
* ( \mathsf{Apply}_i:\mathsf{State}\times\mathsf{State}_i\times\mathsf{Msg}_i\to\mathsf{State}_i )
* ( \mathsf{Exports}_i:\mathsf{State}\to\mathsf{View}_i ) is a read-only projection (queries)

**Important:** ( \mathsf{Apply}_i ) may depend on the full state (S) (for cross-module checks), but it may only mutate (S_i).

---

## 4.1 Message Routing

Define a routing function:
[
\mathsf{route}:\mathsf{TxKind}\to \mathcal{P}({1,\dots,n}\times \mathsf{Msg})
]

That maps a transaction kind into one or more module messages (for multi-module atomic operations).

Example: a receipt submission touches:

* Receipts module
* Economics module
* Accounts module (escrow/settlement)
* Vault module (if stakers share)

Thus a single (Tx) expands into a message bundle:
[
\mathsf{route}(Tx.kind)={(i,m_i)}_{i\in I}
]

---

# 5. Atomic Multi-Module Transactions (Two-Phase Semantics)

To preserve atomicity and determinism for bundled updates, Tetcore uses a conceptual two-phase semantics:

## 5.1 Phase A: Validate

[
\mathsf{Validate}(S,Tx)=\top \ \text{iff all checks pass}
]

Checks include:

* signature validity
* nonce validity
* balance/escrow capacity
* policy constraints
* capability authorization

## 5.2 Phase B: Commit

If valid, apply all module updates as one atomic composite update.

Formally, for message bundle ( {(i,m_i)} ):

[
S'_i = \mathsf{Apply}_i(S,S_i,m_i)
]
[
S' = (S'_1,\dots,S'_n)
]

Atomicity requirement:
[
\mathsf{Inv}(S)\wedge \mathsf{Validate}(S,Tx)\Rightarrow \mathsf{Inv}(S')
]

If any module would violate invariants, the transaction is rejected and:
[
S' = S
]

---

# 6. Canonical Module Decomposition

Define these canonical modules:

1. Accounts ( \mathcal{A} )
2. Model Registry ( \mathcal{MR} )
3. Policy Registry ( \mathcal{PR} )
4. Nodes (Validators & Inference) ( \mathcal{N} )
5. Prompt/Receipt Registry ( \mathcal{RR} )
6. Economics/Revenue Router ( \mathcal{E} )
7. Vaults ( \mathcal{V} )
8. Contracts (TCL/TVM) ( \mathcal{C} )
9. Governance ( \mathcal{G} )

Thus:
[
\mathsf{State} = \mathsf{State}*{\mathcal{A}}\times \mathsf{State}*{\mathcal{MR}}\times\dots\times \mathsf{State}_{\mathcal{G}}
]

---

# 7. Formal State Components and Invariants

## 7.1 Accounts Module ( \mathcal{A} )

State:
[
S_{\mathcal{A}} = {(bal:\mathsf{Addr}\to\mathbb{N}_{128},\ nonce:\mathsf{Addr}\to\mathbb{N})}
]

Invariants:

1. Non-negativity: ( \forall a,\ bal(a)\ge 0 )
2. Nonce monotonicity: (nonce(a)) increases by 1 for each accepted tx from (a)

---

## 7.2 Model Registry ( \mathcal{MR} )

A model version record:
[
ver = (owner, active, root, shardCount, (k,n), ttl, ledgers, pricing, revenue, staking)
]

Invariants:

* (1\le k\le n)
* If active, root is immutable
* Version identifiers are unique per model

---

## 7.3 Prompt/Receipt Registry ( \mathcal{RR} )

Prompt record:
[
p=(from, modelId, version, commit, params, maxOut, feeLimit, createdAt, deadline, status)
]

Receipt record:
[
r=(operator, modelRoot, outCommit, metering, feeCharged, sig, receivedAt, finalized)
]

Prompt status machine:
[
Pending \to Executed \to Finalized
]
with auxiliary:
[
Executed \to Challenged,\ Pending \to Expired
]

Invariant:

* One receipt per prompt (itxId) (or if multiple allowed, must be ordered and governed by replacement rules)

---

## 7.4 Economics Module ( \mathcal{E} )

State includes:

* escrow map (escrow: itxId \to \mathbb{N}_{128})
* revenue policies
* split tables

Invariants:

* Conservation: for finalized (itxId), routed amounts sum to (feeCharged)
* Escrow bounded: (escrow(itxId)\le feeLimit)

---

## 7.5 Vaults Module ( \mathcal{V} )

For each model (m):
[
vault(m)=(enabled,\ totalShares,\ shares:\mathsf{Addr}\to\mathbb{N}_{128},\ accrued)
]

Invariants:

* (totalShares=\sum_a shares(a))
* accrued never negative

---

## 7.6 Contracts Module ( \mathcal{C} )

Contract state:
[
S_{\mathcal{C}}=(code:\mathsf{Addr}\to\mathsf{Hash},\ storage:\mathsf{Addr}\times\mathsf{Hash}\to\mathsf{Bytes})
]

Invariants:

* Deterministic TVM execution
* Gas accounting monotonic
* Storage updates only on Success

---

## 7.7 Governance Module ( \mathcal{G} )

State includes:

* proposals
* votes
* validator set
* parameters
* timelocks

Invariants:

* proposal lifecycle well-formed
* upgrades are timelocked unless emergency-qualified
* parameter ranges remain within safety bounds

---

# 8. Composition via Monoidal Action

Tetcore composition is naturally modeled as a **monoid action** on state.

Let ( \mathsf{Tx}^* ) be the free monoid of finite tx sequences under concatenation.

Define:
[
\triangleright : \mathsf{Tx}^* \times \mathsf{State} \to \mathsf{State}
]

Such that:

1. Identity:
   [
   \epsilon \triangleright S = S
   ]
2. Associativity:
   [
   (x\cdot y)\triangleright S = y \triangleright (x \triangleright S)
   ]

Where (x,y\in \mathsf{Tx}^*).

This is equivalent to fold-application of (\Delta_{\mathsf{tx}}).

---

# 9. Compositionality Theorem (Refinement Goal)

The key refinement property Tetcore targets:

> Global correctness follows from local module correctness plus cross-module invariants.

Formally, if for each module:
[
\mathsf{Inv}(S)\wedge \mathsf{Validate}(S,Tx)\Rightarrow \mathsf{Inv}_i(S'_i)
]
and if cross-invariants are preserved by the combined update,
then:
[
\mathsf{Inv}(S)\wedge \mathsf{Validate}(S,Tx)\Rightarrow \mathsf{Inv}(S')
]

This is the proof structure used in formal verification:

* prove local preservation
* prove cross-invariant preservation
* conclude global safety

---

# 10. Receipt Settlement as a Composed Transaction (Example)

Consider (Tx = SubmitReceipt(itxId,\dots))

Routing produces message bundle:

[
{(\mathcal{RR}, m_{rr}),\ (\mathcal{E}, m_e),\ (\mathcal{A}, m_a),\ (\mathcal{V}, m_v)}
]

Validate checks:

* prompt exists, pending
* model root matches
* fee within limit and policy
* operator authorization/bond status

Commit updates:

* RR: set prompt→Executed, insert receipt
* E: compute splits, lock settlement amounts
* A: transfer amounts atomically
* V: credit vault accrued

Atomicity yields:

* no partial settlement
* deterministic revenue routing
* invariant preservation

---

# 11. Contracts as Composed Sub-State Machines

A contract method call is itself an STS:

[
\mathcal{T}*{call} = (\mathsf{CState},\ \mathsf{Input},\ \delta*{tvm})
]

But it is embedded into global execution as:

[
S' = \Delta_{\mathsf{tx}}(S, CallContract)
]

where (\Delta_{\mathsf{tx}}) delegates to (\delta_{tvm}) to compute the updated (S_{\mathcal{C}}), and may additionally produce constrained updates in other modules via capabilities.

Capability calls are modeled as **restricted morphisms**:
[
cap_j: \mathsf{CState}\times S \to ( \mathsf{CState}, S)
]
but only for the authorized subspaces (e.g., token transfers), preserving invariants.

---

# 12. Determinism as Confluence

Tetcore determinism is a confluence-style requirement over execution:

For any valid state (S) and block (B), all honest nodes produce the same (S'):
[
\forall i,j:\ \Delta_i(S,B)=\Delta_j(S,B)
]

Sufficient conditions:

* ordered tx list
* pure deterministic module functions
* deterministic contract VM
* deterministic capability host functions
* canonical serialization and hashing

---

# 13. Refinement Mapping to Implementations

A concrete implementation must provide:

1. Concrete encodings of (State_i)
2. Deterministic serialization for all state objects
3. Deterministic routing function (\mathsf{route})
4. Deterministic validate/commit semantics
5. A deterministic TVM and TABI

The implementation is correct if it is a **refinement** of this abstract machine:

* implementation steps simulate abstract transitions
* invariants match
* outputs (events/receipts/state roots) correspond

---

# 14. Summary

Tetcore is a compositional state machine system where:

* Global state is a product of module states
* Transactions are routed into module messages
* Validation is global; mutation is local + atomic
* Composition is a monoid action (fold) on state
* Correctness decomposes into local and cross invariants
* Contracts are embedded sub-machines constrained by capabilities
* Determinism is guaranteed by purity, canonical encoding, and gas-bounded VM execution

---

