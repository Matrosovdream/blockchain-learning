# 62 — Reading the go-ethereum Codebase

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#62-reading-the-go-ethereum-codebase) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 15 — Protocol Internals & go-ethereum |
| **Prerequisites** | [18](18-evm.md), [17](17-rlp-merkle-patricia-trie.md) |
| **Unlocks** | 63, 65 |
| **Examples to build** | 13 (🟢 3 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 9 |

*navigating the largest idiomatic Go codebase in the ecosystem, and contributing to it*

## Goals

- Navigate go-ethereum's package structure with confidence.
- Trace a transaction from JSON-RPC through the EVM to state commit, in the real code.
- Build geth from source and run a patched binary.
- Use geth as a library rather than only as a node.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-15-protocol-internals-go-ethereum.md](plan/part-15-protocol-internals-go-ethereum.md#62-reading-the-go-ethereum-codebase).

### 1. Why read it

_Not written yet._

### 2. The package map

_Not written yet._

### 3. Following a transaction

_Not written yet._

### 4. Reading `core/vm`

_Not written yet._

### 5. Reading `core/state`

_Not written yet._

### 6. Go patterns worth stealing

_Not written yet._

### 7. Building and patching

_Not written yet._

### 8. Using geth as a library

_Not written yet._

### 9. Contributing

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/62-geth-codebase/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 4 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 4 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/62-geth-codebase/` — **13 runnable Go programs** (🟢 3 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
