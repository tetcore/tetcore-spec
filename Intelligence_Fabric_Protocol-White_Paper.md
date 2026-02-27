# Intelligence Fabric Protocol (IFP)

## Formal Protocol Specification and Execution State Machines

### Version 2.0 (Sovereign Identity + Native VM Edition)

This document defines IFP as an implementation-neutral engineering standard.

---

# 0. Scope

This specification defines:

* The global state model
* The transaction algebra
* The deterministic state transition function
* The validator consensus assumptions
* The multi-ledger interface
* The inference lifecycle state machine
* The economic settlement state machine
* The contract execution model
* Safety and liveness invariants

This document is independent of any specific implementation.

---

# 1. Cryptographic Primitives

## 1.1 Signature Scheme

Signature algorithm: Ed25519

Let:

* sk ∈ {0,1}²⁵⁶ be private key
* pk ∈ {0,1}²⁵⁶ be public key
* sig ∈ {0,1}⁵¹² be signature

Verification:

[
Verify(pk, msg, sig) ∈ {true, false}
]

---

## 1.2 Address Derivation

[
Addr = H("IFP:ADDR:v1" || pk)
]

Where:

* H is cryptographic hash function
* Address is 32 bytes

---

# 2. Global State Definition

At block height h, the canonical state is:

[
S_h =
(A_h,
M_h,
P_h,
N_h,
R_h,
V_h,
C_h)
]

Where:

A = Accounts
M = Model registry
P = Policies
N = Node registry
R = Receipt and prompt registry
V = Vault registry
C = Contract state

---

# 3. Account Model

For address a:

[
Account(a) =
(balance_a,
nonce_a,
codeHash_a,
storageRoot_a)
]

Constraints:

* balance_a ≥ 0
* nonce strictly increases per valid transaction

---

# 4. Transaction Model

A signed transaction:

[
Tx =
(from,
nonce,
type,
payload,
signature)
]

Validity conditions:

1. Verify(pk_from, hash(tx_body), signature) = true
2. nonce = Account(from).nonce
3. balance sufficient for fees

---

# 5. Deterministic State Transition

State transition per block:

[
S_{h+1} = \Delta(S_h, B_h)
]

Where:

[
\Delta(S_h, B_h) =
Apply(tx_n, ... Apply(tx_1, S_h))
]

Determinism Requirement:

[
\Delta(S, B) \text{ must produce identical output on all honest nodes}
]

---

# 6. Model Registry

A model version:

[
ModelVersion =
(owner,
active,
rootHash,
shardCount,
erasure_k,
erasure_n,
ttl,
storageLedgers,
pricingPolicy,
revenuePolicy,
stakingPolicy)
]

Constraint:

[
1 ≤ erasure_k ≤ erasure_n
]

Root commitment:

[
rootHash = MerkleRoot(H(s_1), ..., H(s_n))
]

---

# 7. Prompt Transaction State Machine

## 7.1 Prompt Creation

Transaction:

SubmitPrompt

Creates entry:

[
Prompt(itxId) =
(from,
modelId,
version,
promptCommitment,
paramsHash,
maxOutput,
feeLimit,
createdAt,
status=Pending)
]

Escrow:

[
balance_{from} -= feeLimit
]

---

## 7.2 Prompt State Machine

States:

Pending
Executed
Challenged
Finalized
Expired

Transitions:

Pending → Executed (receipt accepted)
Executed → Finalized (challenge window passes)
Executed → Challenged (valid dispute)
Pending → Expired (deadline exceeded)

---

# 8. Receipt State Machine

Receipt:

[
Receipt =
(operator,
modelRoot,
outputCommitment,
metering,
feeCharged,
operatorSignature,
receivedAt,
finalized=false)
]

Verification conditions:

1. modelRoot matches ModelVersion.rootHash
2. feeCharged ≤ feeLimit
3. feeCharged valid under pricing policy
4. operatorSignature valid

---

## 8.1 Finalization Rule

If:

[
blockHeight ≥ receivedAt + challengeWindow
]

Then:

Receipt.finalized = true
Prompt.status = Finalized
Trigger revenue settlement

---

# 9. Pricing Function

For owner mode:

[
Fee =
b +
α·p +
β·o +
γ·c
]

Where:

p = prompt tokens
o = output tokens
c = complexity units

Constraint:

[
Fee ≤ feeLimit
]

---

# 10. Revenue Settlement

Let:

F = feeCharged
Split vector:

[
S = (s_i, s_m, s_s, s_v, s_t)
]

Such that:

[
s_i + s_m + s_s + s_v + s_t = 1
]

Distribution:

InferenceOperator = F·s_i
ModelOwner = F·s_m
ShardOwners = F·s_s
Validators = F·s_v
VaultParticipants = F·s_t

All transfers executed atomically.

---

# 11. Vault State Machine

Vault for model m:

[
Vault(m) =
(enabled,
totalShares,
shares[a],
accruedRevenue)
]

Deposit:

[
shares[a] += amount
]
[
totalShares += amount
]

Claim:

[
payout_a =
accruedRevenue ·
\frac{shares[a]}{totalShares}
]

---

# 12. Consensus Assumptions

Let:

n = number of validators
f = number of Byzantine validators

Safety condition:

[
n ≥ 3f + 1
]

Finality achieved when ≥ 2f + 1 validators sign block.

---

# 13. Contract Execution Model

Contracts are programs:

[
Contract =
(code,
storage)
]

Execution:

[
C_{h+1} =
Execute(C_h, tx, gasLimit)
]

Constraints:

* deterministic instruction set
* no floating point
* bounded memory
* gas metering enforced

Contracts may only access:

* own storage
* explicit capability-based host functions

---

# 14. Multi-Ledger Interface

Core chain stores commitments.

Storage ledgers store shard fragments.

Availability checkpoint:

[
Checkpoint =
(ledgerId,
modelId,
version,
epoch,
commitment,
expiryHeight)
]

After expiryHeight + challengeWindow:

Fragments may be pruned.

---

# 15. Safety Invariants

Invariant 1:

[
balance_a ≥ 0
]

Invariant 2:

[
nonces strictly increase
]

Invariant 3:

Revenue split sums to 1

Invariant 4:

ModelVersion.rootHash immutable once activated

Invariant 5:

Deterministic replay equivalence

---

# 16. Liveness Conditions

Liveness requires:

* honest majority of validators
* at least one honest inference operator per model
* relay availability

If these hold:

* Pending prompts eventually finalize
* Governance proposals eventually resolve

---

# 17. Formal Determinism Requirement

For all valid states S and blocks B:

[
\Delta(S, B) = S'
]

And for all honest nodes i, j:

[
\Delta_i(S, B) = \Delta_j(S, B)
]

---

# 18. Conclusion

The Intelligence Fabric Protocol defines:

* a deterministic state machine
* a sovereign identity system
* shard-based model commitments
* relay-mediated inference execution
* programmable pricing
* programmable revenue routing
* vault-based participation
* contract-native extensibility

This specification is sufficient to implement an interoperable IFP-compliant network.

---
