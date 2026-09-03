# 67 — Bridges & Cross-Chain Messaging

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-16-cross-chain-production-operations.md](plan/part-16-cross-chain-production-operations.md#67-bridges-cross-chain-messaging) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 16 — Cross-Chain & Production Operations |
| **Prerequisites** | [30](30-layer2-scaling.md), [64](64-light-clients-spv.md) |
| **Unlocks** | — |
| **Examples to build** | 13 (🟢 3 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 9 |

*moving value and messages between chains — the designs, the proofs, and why bridges keep getting drained*

## Goals

- Classify bridge designs by their trust assumptions.
- Explain how a canonical rollup bridge proves an L2 state on L1.
- Describe the major bridge hacks and the root cause of each.
- Build a Go service that tracks a cross-chain transfer end to end.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-16-cross-chain-production-operations.md](plan/part-16-cross-chain-production-operations.md#67-bridges-cross-chain-messaging).

### 1. The fundamental problem

_Not written yet._

### 2. Lock-mint, burn-mint and liquidity

_Not written yet._

### 3. Externally verified bridges

_Not written yet._

### 4. Natively verified bridges

_Not written yet._

### 5. Rollup canonical bridges

_Not written yet._

### 6. Optimistically verified bridges

_Not written yet._

### 7. General message passing

_Not written yet._

### 8. Tracking transfers in Go

_Not written yet._

### 9. Evaluating a bridge before you integrate

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/67-bridges-cross-chain/`. -->

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

**Examples:** `examples/67-bridges-cross-chain/` — **13 runnable Go programs** (🟢 3 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
