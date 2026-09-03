# 35 — Observability & Reliability for Chain Services

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#35-observability-reliability-for-chain-services) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 7 — Production Blockchain Engineering in Go |
| **Prerequisites** | [31](31-blockchain-indexer.md), [33](33-node-operations.md) |
| **Unlocks** | 41, 68 |
| **Examples to build** | 17 (🟢 4 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*metrics, tracing, alerting on lag and reorgs, retries, circuit breakers and reconciliation*

## Goals

- Instrument a chain service with the metrics that matter.
- Alert on the failure modes unique to blockchain.
- Implement retry, backoff, circuit breaking and failover around RPC.
- Design for at-least-once delivery and eventual consistency.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#35-observability-reliability-for-chain-services).

### 1. What is different here

_Not written yet._

### 2. The core metrics

_Not written yet._

### 3. Structured logging

_Not written yet._

### 4. Tracing

_Not written yet._

### 5. Alerts that are actionable

_Not written yet._

### 6. Retries

_Not written yet._

### 7. Circuit breakers and shedding

_Not written yet._

### 8. Reconciliation

_Not written yet._

### 9. Runbooks

_Not written yet._

### 10. Graceful shutdown

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/35-observability-reliability/`. -->

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

**Examples:** `examples/35-observability-reliability/` — **17 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
