# 46 — Upgradeable Contracts & Proxy Patterns

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-11-smart-contracts-deeper.md](plan/part-11-smart-contracts-deeper.md#46-upgradeable-contracts-proxy-patterns) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 11 — Smart Contracts, Deeper |
| **Prerequisites** | [18](18-evm.md), [27](27-contract-security.md) |
| **Unlocks** | 47 |
| **Examples to build** | 16 (🟢 4 · 🟡 8 · 🔴 4) |
| **Topics in spec** | 9 |

*DELEGATECALL proxies, storage collisions, UUPS vs Transparent vs Beacon, and monitoring upgrades from Go*

## Goals

- Explain how a proxy separates code from storage via DELEGATECALL.
- Compare Transparent, UUPS, Beacon and Diamond patterns.
- Identify storage collisions and selector clashes.
- Detect and monitor implementation changes from a Go service.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-11-smart-contracts-deeper.md](plan/part-11-smart-contracts-deeper.md#46-upgradeable-contracts-proxy-patterns).

### 1. Why upgrades exist

_Not written yet._

### 2. The DELEGATECALL mechanism

_Not written yet._

### 3. Storage collisions

_Not written yet._

### 4. Selector clashes

_Not written yet._

### 5. The patterns

_Not written yet._

### 6. Initializers

_Not written yet._

### 7. Upgrade safety tooling

_Not written yet._

### 8. From the Go side

_Not written yet._

### 9. Reading an unknown proxy

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/46-upgradeable-contracts/`. -->

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

**Examples:** `examples/46-upgradeable-contracts/` — **16 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
