# 08 — Blocks & the Chain

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#08-blocks-the-chain) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [04](04-hash-functions.md), [05](05-merkle-trees.md) |
| **Unlocks** | 09, 10 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 8 |

*the block struct, hash linking, genesis, deterministic serialization and chain validation*

## Goals

- Define a block and a header in Go and hash it deterministically.
- Link blocks by previous-hash and detect any tampering.
- Create a genesis block and validate a whole chain.
- Separate header from body and know why that split exists.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#08-blocks-the-chain).

### 1. Header vs body

_Not written yet._

### 2. A minimal header

_Not written yet._

### 3. Deterministic serialization

_Not written yet._

### 4. Hash linking

_Not written yet._

### 5. The genesis block

_Not written yet._

### 6. Validation rules

_Not written yet._

### 7. Timestamps

_Not written yet._

### 8. Testing the chain

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/08-blocks-and-chain/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 4 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 3 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/08-blocks-and-chain/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
