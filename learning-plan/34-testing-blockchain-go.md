# 34 — Testing Blockchain Code in Go

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#34-testing-blockchain-code-in-go) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 7 — Production Blockchain Engineering in Go |
| **Prerequisites** | [24](24-abigen-bindings.md), [31](31-blockchain-indexer.md) |
| **Unlocks** | 41, 48, 57 |
| **Examples to build** | 17 (🟢 4 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*simulated backends, forked chains, fakes for RPC, deterministic fixtures and reorg tests*

## Goals

- Test contract interaction with no node, using a simulated backend.
- Fork mainnet locally and test against real state.
- Fake the RPC layer behind an interface so tests are fast and deterministic.
- Write a test that reproduces a reorg.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#34-testing-blockchain-code-in-go).

### 1. The test pyramid for chain code

_Not written yet._

### 2. The simulated backend

_Not written yet._

### 3. Forking with anvil

_Not written yet._

### 4. Deterministic fixtures

_Not written yet._

### 5. Faking the RPC layer

_Not written yet._

### 6. Testing reorgs

_Not written yet._

### 7. Fuzzing

_Not written yet._

### 8. Property and invariant tests

_Not written yet._

### 9. Foundry as a complement

_Not written yet._

### 10. CI

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/34-testing-blockchain-go/`. -->

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

**Examples:** `examples/34-testing-blockchain-go/` — **17 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
