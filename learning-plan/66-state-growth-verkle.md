# 66 — State Growth, Verkle & Stateless Ethereum

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#66-state-growth-verkle-stateless-ethereum) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 15 — Protocol Internals & go-ethereum |
| **Prerequisites** | [17](17-rlp-merkle-patricia-trie.md), [64](64-light-clients-spv.md) |
| **Unlocks** | — |
| **Examples to build** | 11 (🟢 3 · 🟡 5 · 🔴 3) |
| **Topics in spec** | 8 |

*why state size is the long-term problem, and the vector commitments meant to fix it*

## Goals

- Explain why state growth threatens node decentralisation.
- Compare Merkle Patricia proofs with Verkle proofs quantitatively.
- Describe statelessness, witnesses and block-level proofs.
- Understand state expiry and history expiry proposals.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#66-state-growth-verkle-stateless-ethereum).

### 1. The problem

_Not written yet._

### 2. Why MPT proofs are too big

_Not written yet._

### 3. Vector commitments and Verkle trees

_Not written yet._

### 4. Statelessness

_Not written yet._

### 5. The migration problem

_Not written yet._

### 6. State and history expiry

_Not written yet._

### 7. What this means for you

_Not written yet._

### 8. Doing the arithmetic in Go

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/66-state-growth-verkle/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 4 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/66-state-growth-verkle/` — **11 runnable Go programs** (🟢 3 easy · 🟡 5 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
