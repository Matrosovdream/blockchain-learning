# 09 — Proof of Work & Mining

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#09-proof-of-work-mining) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [08](08-blocks-and-chain.md) |
| **Unlocks** | — |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 8 |

*the difficulty target, nonce grinding, compact `bits`, retargeting, and the honest cost discussion*

## Goals

- Implement a PoW miner that finds a nonce below a target.
- Convert between difficulty, target and the compact `bits` encoding.
- Retarget difficulty from observed block times.
- Explain what PoW buys (Sybil resistance, objective ordering) and what it costs.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#09-proof-of-work-mining).

### 1. The puzzle

_Not written yet._

### 2. Target, difficulty and bits

_Not written yet._

### 3. The mining loop in Go

_Not written yet._

### 4. Parallel mining

_Not written yet._

### 5. Difficulty retargeting

_Not written yet._

### 6. Statistics of block times

_Not written yet._

### 7. What PoW actually provides

_Not written yet._

### 8. Attacks and criticism

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/09-proof-of-work/`. -->

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

**Examples:** `examples/09-proof-of-work/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
