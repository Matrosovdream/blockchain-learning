# 50 — Off-Chain Signatures: EIP-191, EIP-712, EIP-1271 & SIWE

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#50-off-chain-signatures-eip-191-eip-712-eip-1271-siwe) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 12 — Identity, Wallets & dApp Backends |
| **Prerequisites** | [23](23-abi-encoding.md), [06](06-keys-signatures.md) |
| **Unlocks** | 51 |
| **Examples to build** | 16 (🟢 4 · 🟡 8 · 🔴 4) |
| **Topics in spec** | 9 |

*signature-based login and authorization — the backend half of every dApp*

## Goals

- Verify a personal_sign signature in Go, correctly.
- Implement EIP-712 typed-data signing and verification end to end.
- Verify contract signatures with EIP-1271 (and EIP-6492 for undeployed accounts).
- Build a Sign-In With Ethereum flow with proper nonce and replay protection.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#50-off-chain-signatures-eip-191-eip-712-eip-1271-siwe).

### 1. Why sign off-chain

_Not written yet._

### 2. EIP-191 personal_sign

_Not written yet._

### 3. EIP-712 typed data

_Not written yet._

### 4. Replay protection

_Not written yet._

### 5. EIP-1271 contract signatures

_Not written yet._

### 6. Sign-In With Ethereum (EIP-4361)

_Not written yet._

### 7. Session management

_Not written yet._

### 8. Building the Go handlers

_Not written yet._

### 9. Signed off-chain orders

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/50-offchain-signatures-siwe/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 6 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/50-offchain-signatures-siwe/` — **16 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
