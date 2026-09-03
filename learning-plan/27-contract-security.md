# 27 — Smart Contract Security

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#27-smart-contract-security) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 5 — Smart Contracts from Go |
| **Prerequisites** | [26](26-erc-standards.md) |
| **Unlocks** | 40, 45, 46 |
| **Examples to build** | 19 (🟢 5 · 🟡 9 · 🔴 5) |
| **Topics in spec** | 11 |

*the bug classes that drain contracts, the real incidents, and how a Go integrator defends itself*

## Goals

- Recognise the major on-chain vulnerability classes and their real-world incidents.
- Apply checks-effects-interactions and reentrancy guards.
- Audit an integration for the risks *your* Go service creates.
- Use Slither, fuzzing and invariant tests at a working level.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#27-smart-contract-security).

### 1. Reentrancy

_Not written yet._

### 2. Access control

_Not written yet._

### 3. Arithmetic

_Not written yet._

### 4. External calls

_Not written yet._

### 5. Oracle manipulation

_Not written yet._

### 6. Front-running and MEV as a security property

_Not written yet._

### 7. Signature bugs

_Not written yet._

### 8. Proxies and upgrades

_Not written yet._

### 9. Randomness

_Not written yet._

### 10. Integrator-side defence in Go

_Not written yet._

### 11. Tooling

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/27-contract-security/`. -->

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

**Examples:** `examples/27-contract-security/` — **19 runnable Go programs** (🟢 5 easy · 🟡 9 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
