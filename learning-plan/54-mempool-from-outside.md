# 54 — The Mempool from Outside

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-13-chain-data-at-scale.md](plan/part-13-chain-data-at-scale.md#54-the-mempool-from-outside) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 13 — Chain Data at Scale |
| **Prerequisites** | [20](20-json-rpc-ethclient.md), [21](21-sending-transactions.md) |
| **Unlocks** | — |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 9 |

*watching pending transactions, gas oracles, simulation, and the realities of the public mempool*

## Goals

- Stream and decode pending transactions from Go.
- Build a gas oracle from mempool and block data.
- Simulate a pending transaction's effect before it lands.
- Understand why the public mempool is neither complete nor fair.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-13-chain-data-at-scale.md](plan/part-13-chain-data-at-scale.md#54-the-mempool-from-outside).

### 1. What the mempool is from outside

_Not written yet._

### 2. Streaming pending transactions

_Not written yet._

### 3. Decoding intent

_Not written yet._

### 4. Gas oracles

_Not written yet._

### 5. Transaction simulation

_Not written yet._

### 6. Watching for specific events

_Not written yet._

### 7. The mempool is not fair or complete

_Not written yet._

### 8. Sending privately

_Not written yet._

### 9. Operational concerns

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/54-mempool-from-outside/`. -->

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

**Examples:** `examples/54-mempool-from-outside/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
