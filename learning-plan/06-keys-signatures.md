# 06 — Keys & Digital Signatures (ECDSA on secp256k1)

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#06-keys-digital-signatures-ecdsa-on-secp256k1) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [04](04-hash-functions.md) |
| **Unlocks** | 07, 10, 19, 42, 50 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*private/public keys, the curve, signing, verifying, public-key recovery, malleability and nonce disasters*

## Goals

- Generate a secp256k1 keypair in Go and derive the public key.
- Sign a message hash, verify it, and recover the public key from the signature.
- Explain `v`, `r`, `s` and why low-`s` is enforced.
- Explain why a reused or predictable nonce leaks the private key.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#06-keys-digital-signatures-ecdsa-on-secp256k1).

### 1. Asymmetric cryptography in one page

_Not written yet._

### 2. Elliptic curves, intuitively

_Not written yet._

### 3. secp256k1 vs the alternatives

_Not written yet._

### 4. Key generation in Go

_Not written yet._

### 5. You sign a hash, never a message

_Not written yet._

### 6. ECDSA internals

_Not written yet._

### 7. The go-ethereum signature API

_Not written yet._

### 8. Malleability

_Not written yet._

### 9. Nonce catastrophes

_Not written yet._

### 10. What replaces ECDSA

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/06-keys-signatures/`. -->

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

**Examples:** `examples/06-keys-signatures/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
