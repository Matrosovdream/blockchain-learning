# 43 — Multisig, MPC & Threshold Signing

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#43-multisig-mpc-threshold-signing) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 10 — Cryptography, Deeper |
| **Prerequisites** | [32](32-key-management-signing.md), [42](42-schnorr-bls-aggregate.md) |
| **Unlocks** | — |
| **Examples to build** | 14 (🟢 4 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 8 |

*removing the single point of failure — on-chain multisig, Shamir sharing, and threshold signatures*

## Goals

- Compare on-chain multisig, secret sharing and threshold signing on their real trade-offs.
- Implement Shamir's Secret Sharing in Go.
- Explain how a t-of-n threshold signature is produced without ever reconstructing the key.
- Interact with a Safe multisig from Go.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#43-multisig-mpc-threshold-signing).

### 1. The problem

_Not written yet._

### 2. On-chain multisig

_Not written yet._

### 3. Shamir's Secret Sharing

_Not written yet._

### 4. Threshold signatures (MPC/TSS)

_Not written yet._

### 5. Threshold ECDSA vs threshold Schnorr

_Not written yet._

### 6. Comparing the three

_Not written yet._

### 7. Operational design

_Not written yet._

### 8. Go implementations

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/43-multisig-mpc-threshold/`. -->

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

**Examples:** `examples/43-multisig-mpc-threshold/` — **14 runnable Go programs** (🟢 4 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
