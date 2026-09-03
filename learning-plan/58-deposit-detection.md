# 58 — Deposit Detection & Address Management

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#58-deposit-detection-address-management) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 14 — Custody, Payments & Compliance |
| **Prerequisites** | [31](31-blockchain-indexer.md), [32](32-key-management-signing.md) |
| **Unlocks** | 59, 60, 61 |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 8 |

*per-user addresses, detecting incoming value of every kind, confirmations and crediting exactly once*

## Goals

- Assign deposit addresses to users safely and at scale.
- Detect native, token and internal-transfer deposits reliably.
- Apply a confirmation policy and credit exactly once.
- Handle the awkward cases: unexpected tokens, dust, wrong-chain sends.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#58-deposit-detection-address-management).

### 1. Address assignment strategies

_Not written yet._

### 2. Detecting native transfers

_Not written yet._

### 3. Detecting token deposits

_Not written yet._

### 4. Confirmation policy

_Not written yet._

### 5. Crediting exactly once

_Not written yet._

### 6. The awkward cases

_Not written yet._

### 7. Monitoring at scale

_Not written yet._

### 8. Reconciliation

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/58-deposit-detection/`. -->

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

**Examples:** `examples/58-deposit-detection/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
