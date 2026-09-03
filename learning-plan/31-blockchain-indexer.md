# 31 — Building a Blockchain Indexer in Go

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#31-building-a-blockchain-indexer-in-go) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 7 — Production Blockchain Engineering in Go |
| **Prerequisites** | [25](25-events-logs.md), [20](20-json-rpc-ethclient.md) |
| **Unlocks** | 34, 35, 41, 56, 57, 58 |
| **Examples to build** | 19 (🟢 5 · 🟡 9 · 🔴 5) |
| **Topics in spec** | 11 |

*ingesting blocks and logs, reorg-safe writes, backfill, idempotency and a schema that answers queries*

## Goals

- Build a service that ingests blocks and logs into a database, continuously.
- Handle reorgs correctly — detect, roll back, re-apply.
- Backfill history quickly without exhausting your provider.
- Design a schema and cursor model that make the indexer restartable and idempotent.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#31-building-a-blockchain-indexer-in-go).

### 1. Why you index

_Not written yet._

### 2. The ingestion loop

_Not written yet._

### 3. Reorg detection

_Not written yet._

### 4. Reorg-safe writes

_Not written yet._

### 5. Idempotency

_Not written yet._

### 6. Backfill vs live tail

_Not written yet._

### 7. Schema design

_Not written yet._

### 8. Cursors and checkpoints

_Not written yet._

### 9. Provider pressure

_Not written yet._

### 10. Operations

_Not written yet._

### 11. Testing it

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/31-blockchain-indexer/`. -->

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

**Examples:** `examples/31-blockchain-indexer/` — **19 runnable Go programs** (🟢 5 easy · 🟡 9 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
