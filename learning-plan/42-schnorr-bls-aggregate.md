# 42 — Schnorr, BLS & Aggregate Signatures

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#42-schnorr-bls-aggregate-signatures) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 10 — Cryptography, Deeper |
| **Prerequisites** | [06](06-keys-signatures.md) |
| **Unlocks** | 43 |
| **Examples to build** | 15 (🟢 4 · 🟡 7 · 🔴 4) |
| **Topics in spec** | 8 |

*the signature schemes that replaced ECDSA — linearity, aggregation, and where each is used*

## Goals

- Implement and verify a BIP-340 Schnorr signature in Go.
- Explain why Schnorr's linearity enables key and signature aggregation.
- Use BLS signatures and aggregate many into one.
- Say precisely which chain uses which scheme and why.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#42-schnorr-bls-aggregate-signatures).

### 1. Why ECDSA was replaced

_Not written yet._

### 2. Schnorr signatures

_Not written yet._

### 3. BIP-340

_Not written yet._

### 4. Key and signature aggregation

_Not written yet._

### 5. BLS signatures

_Not written yet._

### 6. BLS12-381

_Not written yet._

### 7. Where each scheme lives

_Not written yet._

### 8. Threshold and ring signatures

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/42-schnorr-bls-aggregate/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 4 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/42-schnorr-bls-aggregate/` — **15 runnable Go programs** (🟢 4 easy · 🟡 7 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
