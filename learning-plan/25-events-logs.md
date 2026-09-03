# 25 — Events, Logs & Indexing Them

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#25-events-logs-indexing-them) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 5 — Smart Contracts from Go |
| **Prerequisites** | [23](23-abi-encoding.md) |
| **Unlocks** | 26, 31 |
| **Examples to build** | 19 (🟢 5 · 🟡 9 · 🔴 5) |
| **Topics in spec** | 9 |

*topics, the bloom filter, `eth_getLogs` at scale, decoding, subscriptions and reorg-safe consumption*

## Goals

- Decode an event log into typed Go values, including indexed parameters.
- Query logs efficiently with topic filters and chunked block ranges.
- Explain the logs bloom filter and what it is actually good for.
- Choose between polling and subscribing, and survive reorgs either way.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#25-events-logs-indexing-them).

### 1. What a log is

_Not written yet._

### 2. Log structure

_Not written yet._

### 3. The indexed-dynamic-type trap

_Not written yet._

### 4. The bloom filter

_Not written yet._

### 5. eth_getLogs in practice

_Not written yet._

### 6. Decoding in Go

_Not written yet._

### 7. Polling vs subscribing

_Not written yet._

### 8. Reorgs and logs

_Not written yet._

### 9. Ordering and idempotency

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/25-events-logs/`. -->

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

**Examples:** `examples/25-events-logs/` — **19 runnable Go programs** (🟢 5 easy · 🟡 9 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
