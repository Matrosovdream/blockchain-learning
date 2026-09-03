# 29 — Alternative Consensus: BFT, PoA & the Rest

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-06-consensus-scaling.md](plan/part-06-consensus-scaling.md#29-alternative-consensus-bft-poa-the-rest) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 6 — Consensus & Scaling |
| **Prerequisites** | [28](28-proof-of-stake.md) |
| **Unlocks** | 37 |
| **Examples to build** | 13 (🟢 4 · 🟡 6 · 🔴 3) |
| **Topics in spec** | 9 |

*PBFT, Tendermint, HotStuff, Clique, DPoS — what each assumes, what each buys, and one implemented in Go*

## Goals

- Explain the BFT family and where the 3f+1 bound comes from.
- Compare instant-finality BFT with probabilistic longest-chain consensus.
- Choose a consensus mechanism for a given requirement set.
- Implement a simplified BFT round in Go.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-06-consensus-scaling.md](plan/part-06-consensus-scaling.md#29-alternative-consensus-bft-poa-the-rest).

### 1. The formal frame

_Not written yet._

### 2. CFT vs BFT

_Not written yet._

### 3. The 3f+1 bound

_Not written yet._

### 4. PBFT

_Not written yet._

### 5. Tendermint / CometBFT

_Not written yet._

### 6. HotStuff and linear BFT

_Not written yet._

### 7. Proof of Authority (Clique)

_Not written yet._

### 8. Delegated and other schemes

_Not written yet._

### 9. The decision table

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/29-alternative-consensus/`. -->

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

**Examples:** `examples/29-alternative-consensus/` — **13 runnable Go programs** (🟢 4 easy · 🟡 6 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
