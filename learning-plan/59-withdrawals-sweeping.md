# 59 — Withdrawals, Sweeping & Fee Management

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#59-withdrawals-sweeping-fee-management) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 14 — Custody, Payments & Compliance |
| **Prerequisites** | [21](21-sending-transactions.md), [32](32-key-management-signing.md), [58](58-deposit-detection.md) |
| **Unlocks** | 60 |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 9 |

*moving value out safely — sweeps, batching, gas funding, approvals and the withdrawal state machine*

## Goals

- Design a withdrawal pipeline that never double-pays and never loses a request.
- Sweep deposit addresses efficiently, including the gas-funding problem.
- Batch withdrawals to cut cost, with correct failure attribution.
- Manage hot-wallet float and fee budgets.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-14-custody-payments-compliance.md](plan/part-14-custody-payments-compliance.md#59-withdrawals-sweeping-fee-management).

### 1. The withdrawal state machine

_Not written yet._

### 2. Never double-pay

_Not written yet._

### 3. Approval workflows

_Not written yet._

### 4. Sweeping deposit addresses

_Not written yet._

### 5. Batching withdrawals

_Not written yet._

### 6. Fee management

_Not written yet._

### 7. Hot wallet float

_Not written yet._

### 8. Token approvals

_Not written yet._

### 9. Stuck and failed withdrawals

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/59-withdrawals-sweeping/`. -->

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

**Examples:** `examples/59-withdrawals-sweeping/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
