# 32 — Key Management & Signing Services

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#32-key-management-signing-services) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 7 — Production Blockchain Engineering in Go |
| **Prerequisites** | [21](21-sending-transactions.md) |
| **Unlocks** | 41, 43, 58, 59, 68 |
| **Examples to build** | 17 (🟢 4 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*keystores, KMS/HSM, hot/warm/cold tiers, a policy-enforcing signing service, and nonces at scale*

## Goals

- Store and use keys without a private key ever entering your source, logs or backups.
- Build a signing service with a real authorization boundary.
- Manage nonces for many concurrent senders reliably.
- Reason about tiering and blast radius.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-07-production-blockchain-engineering-in-go.md](plan/part-07-production-blockchain-engineering-in-go.md#32-key-management-signing-services).

### 1. Threat model first

_Not written yet._

### 2. Web3 keystore v3

_Not written yet._

### 3. Cloud KMS and HSMs

_Not written yet._

### 4. A signing service

_Not written yet._

### 5. Policy at the signing boundary

_Not written yet._

### 6. Hot / warm / cold tiering

_Not written yet._

### 7. Multisig and threshold signing

_Not written yet._

### 8. Nonce management at scale

_Not written yet._

### 9. Many senders

_Not written yet._

### 10. Operational hygiene

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/32-key-management-signing/`. -->

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

**Examples:** `examples/32-key-management-signing/` — **17 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
