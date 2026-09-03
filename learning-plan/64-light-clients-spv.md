# 64 — Light Clients, SPV & Trustless Reads

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#64-light-clients-spv-trustless-reads) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 15 — Protocol Internals & go-ethereum |
| **Prerequisites** | [17](17-rlp-merkle-patricia-trie.md), [28](28-proof-of-stake.md) |
| **Unlocks** | 66, 67 |
| **Examples to build** | 13 (🟢 3 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 8 |

*verifying without trusting — Merkle proofs, header chains, sync committees and proof verification in Go*

## Goals

- Explain what a light client verifies and what it must still trust.
- Verify an account or storage proof against a state root in Go.
- Implement Bitcoin SPV verification of a transaction's inclusion.
- Describe Ethereum's sync-committee light client protocol.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#64-light-clients-spv-trustless-reads).

### 1. The trust problem

_Not written yet._

### 2. Bitcoin SPV

_Not written yet._

### 3. Ethereum state proofs

_Not written yet._

### 4. Receipt and transaction proofs

_Not written yet._

### 5. Ethereum's light client protocol

_Not written yet._

### 6. Where trustless reads matter

_Not written yet._

### 7. Practical middle grounds

_Not written yet._

### 8. Costs

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/64-light-clients-spv/`. -->

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

**Examples:** `examples/64-light-clients-spv/` — **13 runnable Go programs** (🟢 3 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
