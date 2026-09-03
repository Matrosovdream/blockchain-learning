# 51 — Wallet Integration & dApp Backends

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#51-wallet-integration-dapp-backends) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 12 — Identity, Wallets & dApp Backends |
| **Prerequisites** | [50](50-offchain-signatures-siwe.md) |
| **Unlocks** | — |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 10 |

*the wallet RPC surface, WalletConnect, webhooks, and the Go service behind a dApp*

## Goals

- Describe the wallet RPC methods a frontend calls and what your backend must supply.
- Explain WalletConnect's session model at a working level.
- Build the backend endpoints a dApp actually needs.
- Deliver webhooks reliably with signatures, retries and idempotency.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#51-wallet-integration-dapp-backends).

### 1. The division of labour

_Not written yet._

### 2. The wallet RPC surface (EIP-1193)

_Not written yet._

### 3. WalletConnect

_Not written yet._

### 4. What the backend must provide

_Not written yet._

### 5. Preparing transactions server-side

_Not written yet._

### 6. Tracking user transactions

_Not written yet._

### 7. Webhooks and notifications

_Not written yet._

### 8. Realtime to the browser

_Not written yet._

### 9. Multi-chain and multi-wallet

_Not written yet._

### 10. Security for dApp backends

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/51-wallet-integration-backends/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/51-wallet-integration-backends/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
