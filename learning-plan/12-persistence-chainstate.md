# 12 — Persistence & Chain State

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#12-persistence-chain-state) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [10](10-transactions-utxo.md) |
| **Unlocks** | 13 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 9 |

*storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety*

## Goals

- Persist blocks and chain state to an embedded key-value store from Go.
- Design key prefixes and iterate the chain efficiently.
- Write atomically so a crash never leaves a half-applied block.
- Reindex state from raw blocks.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#12-persistence-chain-state).

### 1. Why a key-value store

_Not written yet._

### 2. Key-space design

_Not written yet._

### 3. Serialization on disk

_Not written yet._

### 4. Atomic writes

_Not written yet._

### 5. Iterators and cursors

_Not written yet._

### 6. Reindexing

_Not written yet._

### 7. Pruning and snapshots

_Not written yet._

### 8. Crash safety

_Not written yet._

### 9. The storage interface

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/12-persistence-chainstate/`. -->

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

**Examples:** `examples/12-persistence-chainstate/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
