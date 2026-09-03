# 11 — Wallets, Fees & the Mempool

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#11-wallets-fees-the-mempool) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [07](07-addresses-wallets-hd.md), [10](10-transactions-utxo.md) |
| **Unlocks** | 36 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 9 |

*a keystore, transaction construction, the fee market, mempool policy and block assembly*

## Goals

- Build a wallet that stores keys and constructs spendable transactions.
- Implement a mempool with validation, replacement and eviction.
- Select transactions for a block by fee rate.
- Explain fee estimation and replace-by-fee.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#11-wallets-fees-the-mempool).

### 1. What a wallet is

_Not written yet._

### 2. Persisting keys

_Not written yet._

### 3. Building a spend

_Not written yet._

### 4. The mempool

_Not written yet._

### 5. Mempool policy vs consensus rules

_Not written yet._

### 6. Fees

_Not written yet._

### 7. Block assembly

_Not written yet._

### 8. Replacement and eviction

_Not written yet._

### 9. Concurrency

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/11-wallets-mempool/`. -->

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

**Examples:** `examples/11-wallets-mempool/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
