# 33 — Node Operations & RPC Infrastructure

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#33-node-operations-rpc-infrastructure) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 7 — Production Blockchain Engineering in Go |
| **Prerequisites** | [20](20-json-rpc-ethclient.md) |
| **Unlocks** | 35, 63 |
| **Examples to build** | 15 (🟢 4 · 🟡 7 · 🔴 4) |
| **Topics in spec** | 11 |

*running geth, sync modes, storage, providers, failover and what actually breaks in production*

## Goals

- Run and monitor an execution + consensus client pair.
- Choose a sync mode and understand its disk and time cost.
- Decide between self-hosting and a provider, with numbers.
- Build a resilient RPC client layer in Go.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#33-node-operations-rpc-infrastructure).

### 1. The post-Merge reality

_Not written yet._

### 2. Execution clients

_Not written yet._

### 3. Sync modes

_Not written yet._

### 4. Hardware

_Not written yet._

### 5. Configuration that matters

_Not written yet._

### 6. Monitoring a node

_Not written yet._

### 7. Providers

_Not written yet._

### 8. Build vs buy

_Not written yet._

### 9. A resilient RPC layer in Go

_Not written yet._

### 10. Consistency traps

_Not written yet._

### 11. Local development

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/33-node-operations/`. -->

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

**Examples:** `examples/33-node-operations/` — **15 runnable Go programs** (🟢 4 easy · 🟡 7 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
