# 21 — Sending Transactions from Go

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#21-sending-transactions-from-go) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 4 — Ethereum & the EVM |
| **Prerequisites** | [19](19-transaction-types.md), [20](20-json-rpc-ethclient.md) |
| **Unlocks** | 32, 54, 59 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 11 |

*nonce management, gas estimation, 1559 fees, signing, broadcast, confirmation and stuck-tx recovery*

## Goals

- Send a signed transaction from Go, end to end.
- Manage nonces correctly under concurrency.
- Set 1559 fees that get included without overpaying.
- Wait for confirmations safely and recover a stuck transaction.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#21-sending-transactions-from-go).

### 1. The pipeline

_Not written yet._

### 2. Nonce sources

_Not written yet._

### 3. A nonce manager

_Not written yet._

### 4. Gas estimation

_Not written yet._

### 5. Fee strategy

_Not written yet._

### 6. Signing options

_Not written yet._

### 7. Broadcast and its error taxonomy

_Not written yet._

### 8. Waiting for a receipt

_Not written yet._

### 9. Confirmation depth and reorgs

_Not written yet._

### 10. Stuck transactions

_Not written yet._

### 11. Idempotency and crash safety

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/21-sending-transactions/`. -->

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

**Examples:** `examples/21-sending-transactions/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
