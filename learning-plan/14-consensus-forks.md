# 14 — Consensus, Forks & Reorgs

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#14-consensus-forks-reorgs) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [13](13-p2p-networking.md) |
| **Unlocks** | 15, 28 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 9 |

*fork choice by accumulated work, reorg handling, finality as probability, and the attacks*

## Goals

- Implement heaviest-chain fork choice by accumulated work.
- Handle a reorg: find the common ancestor, roll back, roll forward.
- Explain probabilistic finality and confirmation counts.
- Name the failure modes: 51%, selfish mining, consensus-bug splits.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#14-consensus-forks-reorgs).

### 1. Consensus is agreement on order

_Not written yet._

### 2. Fork choice

_Not written yet._

### 3. The block tree

_Not written yet._

### 4. Reorg mechanics

_Not written yet._

### 5. Undo data

_Not written yet._

### 6. Returning transactions to the mempool

_Not written yet._

### 7. Confirmations and probabilistic finality

_Not written yet._

### 8. Forks of the other kind

_Not written yet._

### 9. Attacks

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/14-consensus-forks/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 3 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/14-consensus-forks/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
