# 10 — Transactions & the UTXO Model

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#10-transactions-the-utxo-model) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [06](06-keys-signatures.md), [08](08-blocks-and-chain.md) |
| **Unlocks** | 11, 12, 15, 36 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 8 |

*inputs, outputs, the coinbase, signing over a trimmed copy, and maintaining the UTXO set*

## Goals

- Model UTXO transactions in Go: inputs referencing prior outputs, outputs locking value.
- Sign and verify a transaction the way Bitcoin does.
- Build the coinbase transaction and enforce conservation of value.
- Maintain and query a UTXO set.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#10-transactions-the-utxo-model).

### 1. Accounts vs UTXO

_Not written yet._

### 2. Transaction structure

_Not written yet._

### 3. The coinbase transaction

_Not written yet._

### 4. What exactly gets signed

_Not written yet._

### 5. Verification

_Not written yet._

### 6. Conservation of value

_Not written yet._

### 7. The UTXO set

_Not written yet._

### 8. Coin selection

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/10-transactions-utxo/`. -->

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

**Examples:** `examples/10-transactions-utxo/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
