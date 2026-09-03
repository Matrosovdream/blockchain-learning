# 19 — Ethereum Transactions Deep Dive

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#19-ethereum-transactions-deep-dive) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 4 — Ethereum & the EVM |
| **Prerequisites** | [06](06-keys-signatures.md), [17](17-rlp-merkle-patricia-trie.md) |
| **Unlocks** | 21 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*legacy, 2930, 1559 and 4844 types — signing, RLP, chain id, hashing and sender recovery*

## Goals

- Construct and sign every live Ethereum transaction type in Go.
- Explain EIP-155 chain-id replay protection and the `v` encoding.
- Compute a transaction hash and recover the sender from raw bytes.
- Choose the right type and fee fields for a situation.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#19-ethereum-transactions-deep-dive).

### 1. The typed-transaction envelope

_Not written yet._

### 2. Type 0 — legacy

_Not written yet._

### 3. EIP-155 replay protection

_Not written yet._

### 4. Type 1 — EIP-2930

_Not written yet._

### 5. Type 2 — EIP-1559

_Not written yet._

### 6. Type 3 — EIP-4844

_Not written yet._

### 7. The signing hash vs the transaction hash

_Not written yet._

### 8. The go-ethereum API

_Not written yet._

### 9. Sender recovery

_Not written yet._

### 10. Practical rules

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/19-transaction-types/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/19-transaction-types/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
