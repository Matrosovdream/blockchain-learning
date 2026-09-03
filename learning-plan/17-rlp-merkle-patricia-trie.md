# 17 — RLP & the Merkle Patricia Trie

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#17-rlp-the-merkle-patricia-trie) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 4 — Ethereum & the EVM |
| **Prerequisites** | [05](05-merkle-trees.md), [16](16-ethereum-architecture.md) |
| **Unlocks** | 18, 19, 30, 62, 64, 66 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 11 |

*Ethereum's serialization format and the trie that produces every root in the header*

## Goals

- Encode and decode RLP by hand and with go-ethereum's `rlp` package.
- Explain the MPT node types and hex-prefix encoding.
- Build a small trie and compute its root.
- Verify an `eth_getProof` result against a state root.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#17-rlp-the-merkle-patricia-trie).

### 1. What RLP is and is not

_Not written yet._

### 2. The RLP rules in full

_Not written yet._

### 3. Encoding integers

_Not written yet._

### 4. go-ethereum's rlp package

_Not written yet._

### 5. Why a plain Merkle tree is not enough

_Not written yet._

### 6. MPT node types

_Not written yet._

### 7. Hex-prefix (compact) encoding

_Not written yet._

### 8. Node hashing and the inline rule

_Not written yet._

### 9. The four tries

_Not written yet._

### 10. Proofs

_Not written yet._

### 11. What comes next

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/17-rlp-merkle-patricia-trie/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 4 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/17-rlp-merkle-patricia-trie/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
