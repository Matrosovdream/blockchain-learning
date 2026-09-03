# 20 — JSON-RPC & go-ethereum's `ethclient`

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#20-json-rpc-go-ethereums-ethclient) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 4 — Ethereum & the EVM |
| **Prerequisites** | [16](16-ethereum-architecture.md) |
| **Unlocks** | 21, 31, 33, 54 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*the node API, `ethclient` from Go, queries, filters, subscriptions, batching and provider realities*

## Goals

- Call the Ethereum JSON-RPC API directly and through `ethclient`.
- Read blocks, receipts, balances, storage and code from Go.
- Subscribe to new heads and logs over WebSocket, with reconnection.
- Handle provider limits, batching and errors realistically.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#20-json-rpc-go-ethereums-ethclient).

### 1. JSON-RPC 2.0 basics

_Not written yet._

### 2. The methods you will actually use

_Not written yet._

### 3. Block tags

_Not written yet._

### 4. ethclient in Go

_Not written yet._

### 5. Raw and batch calls

_Not written yet._

### 6. eth_call and state overrides

_Not written yet._

### 7. Subscriptions

_Not written yet._

### 8. Provider realities

_Not written yet._

### 9. Error handling and retries

_Not written yet._

### 10. Debug and trace namespaces

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/20-json-rpc-ethclient/`. -->

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

**Examples:** `examples/20-json-rpc-ethclient/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
