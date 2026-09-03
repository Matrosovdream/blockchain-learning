# 44 — Symmetric Crypto, KDFs & Encryption at Rest

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#44-symmetric-crypto-kdfs-encryption-at-rest) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 10 — Cryptography, Deeper |
| **Prerequisites** | [04](04-hash-functions.md) |
| **Unlocks** | — |
| **Examples to build** | 16 (🟢 4 · 🟡 8 · 🔴 4) |
| **Topics in spec** | 10 |

*AES-GCM, ChaCha20, argon2/scrypt, envelope encryption, and the keystore format decoded*

## Goals

- Encrypt and authenticate data correctly in Go with AEAD.
- Choose and parameterise a password KDF.
- Implement and decode a Web3 keystore v3 file by hand.
- Design envelope encryption with a KMS.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-10-cryptography-deeper.md](plan/part-10-cryptography-deeper.md#44-symmetric-crypto-kdfs-encryption-at-rest).

### 1. Symmetric vs asymmetric

_Not written yet._

### 2. AEAD is the only acceptable default

_Not written yet._

### 3. Nonces

_Not written yet._

### 4. Associated data

_Not written yet._

### 5. Password-based KDFs

_Not written yet._

### 6. HKDF and key hierarchies

_Not written yet._

### 7. The Web3 keystore v3 file, decoded

_Not written yet._

### 8. Envelope encryption

_Not written yet._

### 9. Key rotation and secret lifecycle

_Not written yet._

### 10. Go pitfalls

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/44-symmetric-crypto-at-rest/`. -->

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

**Examples:** `examples/44-symmetric-crypto-at-rest/` — **16 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
