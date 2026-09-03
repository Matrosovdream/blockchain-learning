# 05 — Merkle Trees & Proofs

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#05-merkle-trees-proofs) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [04](04-hash-functions.md) |
| **Unlocks** | 08, 17, 39 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 8 |

*building a tree, the root as a commitment, inclusion proofs, and the traps (odd leaves, second preimage)*

## Goals

- Build a Merkle tree over a list of items in Go.
- Generate and verify an inclusion proof without holding the whole list.
- Explain why a block header only needs one 32-byte root.
- Know the odd-leaf duplication bug and the second-preimage defence.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **8 topics** with their sub-points: [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#05-merkle-trees-proofs).

### 1. The problem

_Not written yet._

### 2. Construction

_Not written yet._

### 3. Inclusion proofs

_Not written yet._

### 4. Odd numbers of leaves

_Not written yet._

### 5. Second-preimage resistance

_Not written yet._

### 6. Sorted-pair trees

_Not written yet._

### 7. Variants you will meet

_Not written yet._

### 8. Merkle proofs in production

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/05-merkle-trees/`. -->

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

**Examples:** `examples/05-merkle-trees/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
