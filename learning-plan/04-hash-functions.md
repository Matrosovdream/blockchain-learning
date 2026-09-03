# 04 — Cryptographic Hash Functions

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#04-cryptographic-hash-functions) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [03](03-bytes-encoding.md) |
| **Unlocks** | 05, 06, 08, 44, 55 |
| **Examples to build** | 20 (🟢 6 · 🟡 8 · 🔴 6) |
| **Topics in spec** | 9 |

*SHA-256, Keccak-256, the five properties, commitments, and hashing structured data deterministically*

## Goals

- Compute SHA-256, SHA-3 and Keccak-256 in Go and know which chain uses which.
- State the five properties of a cryptographic hash and what breaks when each fails.
- Use a hash as a commitment and as an identifier.
- Serialize a struct deterministically before hashing it.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#04-cryptographic-hash-functions).

### 1. What a hash function is

_Not written yet._

### 2. The five properties

_Not written yet._

### 3. SHA-256 vs SHA-3 vs Keccak-256

_Not written yet._

### 4. Double hashing

_Not written yet._

### 5. Length-extension attacks

_Not written yet._

### 6. Hashing structured data

_Not written yet._

### 7. Commitments

_Not written yet._

### 8. Hash as identifier

_Not written yet._

### 9. Performance

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/04-hash-functions/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 4 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 4 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/04-hash-functions/` — **20 runnable Go programs** (🟢 6 easy · 🟡 8 medium · 🔴 6 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
