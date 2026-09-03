# 15 — The Account Model & World State

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#15-the-account-model-world-state) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [10](10-transactions-utxo.md), [14](14-consensus-forks.md) |
| **Unlocks** | 16, 38 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 8 |

*accounts, nonces, balances, code and storage — the state transition and how it differs from UTXO*

## Goals

- Model an account-based ledger in Go: balance, nonce, code hash, storage root.
- Explain nonces as replay protection and as ordering.
- Apply a transaction as a state transition and compute a new state root.
- Compare UTXO and account models honestly.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#15-the-account-model-world-state).

### 1. From coins to accounts

_Not written yet._

### 2. The account struct

_Not written yet._

### 3. Nonces

_Not written yet._

### 4. The state transition function

_Not written yet._

### 5. Journalling and reverts

_Not written yet._

### 6. The state root

_Not written yet._

### 7. Copy-on-write state in Go

_Not written yet._

### 8. UTXO vs accounts, honestly

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/15-account-model-state/`. -->

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

**Examples:** `examples/15-account-model-state/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
