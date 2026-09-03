# 60 — Ledger & Accounting for Crypto

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#60-ledger-accounting-for-crypto) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 14 — Custody, Payments & Compliance |
| **Prerequisites** | [58](58-deposit-detection.md), [59](59-withdrawals-sweeping.md) |
| **Unlocks** | 61 |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 9 |

*double-entry bookkeeping for on-chain value — the design that makes your numbers provable*

## Goals

- Design a double-entry ledger for multi-asset, multi-chain balances.
- Represent every on-chain event as balanced journal entries.
- Reconcile internal balances against on-chain reality.
- Handle the hard cases: fees, rebasing, wrapped assets, failed transactions.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#60-ledger-accounting-for-crypto).

### 1. Why double-entry

_Not written yet._

### 2. The data model

_Not written yet._

### 3. Modelling on-chain events

_Not written yet._

### 4. Multi-asset and multi-chain

_Not written yet._

### 5. The hard cases

_Not written yet._

### 6. Reconciliation

_Not written yet._

### 7. Point-in-time and reporting

_Not written yet._

### 8. Idempotency and ordering

_Not written yet._

### 9. Implementing it in Go

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/60-ledger-accounting/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 6 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 3 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/60-ledger-accounting/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
