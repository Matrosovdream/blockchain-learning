# 39 — Zero-Knowledge Proofs with `gnark`

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#39-zero-knowledge-proofs-with-gnark) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 8 — Beyond Ethereum |
| **Prerequisites** | [05](05-merkle-trees.md), [30](30-layer2-scaling.md) |
| **Unlocks** | — |
| **Examples to build** | 13 (🟢 4 · 🟡 6 · 🔴 3) |
| **Topics in spec** | 10 |

*commitments, circuits, SNARKs vs STARKs, and writing and proving a circuit in Go*

## Goals

- Explain what a ZK proof proves, and what it does not.
- Describe the SNARK pipeline: circuit → constraints → setup → prove → verify.
- Write, compile and prove a circuit in Go with `gnark`.
- Judge where ZK is genuinely the right tool.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#39-zero-knowledge-proofs-with-gnark).

### 1. The three properties

_Not written yet._

### 2. Interactive to non-interactive

_Not written yet._

### 3. Arithmetic circuits and R1CS

_Not written yet._

### 4. The proving systems

_Not written yet._

### 5. Trusted setup

_Not written yet._

### 6. Commitments

_Not written yet._

### 7. gnark in Go

_Not written yet._

### 8. Circuit gadgets

_Not written yet._

### 9. On-chain verification

_Not written yet._

### 10. Applications and honest costs

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/39-zero-knowledge-proofs/`. -->

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

**Examples:** `examples/39-zero-knowledge-proofs/` — **13 runnable Go programs** (🟢 4 easy · 🟡 6 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
