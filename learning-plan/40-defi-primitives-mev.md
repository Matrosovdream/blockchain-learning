# 40 — DeFi Primitives & MEV from a Go Integrator's View

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#40-defi-primitives-mev-from-a-go-integrators-view) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 8 — Beyond Ethereum |
| **Prerequisites** | [26](26-erc-standards.md), [27](27-contract-security.md) |
| **Unlocks** | 53 |
| **Examples to build** | 17 (🟢 4 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 11 |

*AMMs, lending, liquidations, MEV and flash loans — the math, and the integration code that survives it*

## Goals

- Compute AMM swap outputs and price impact with exact integer math in Go.
- Explain lending health factors and liquidation mechanics.
- Describe MEV: sandwiches, arbitrage, liquidations, and PBS.
- Integrate safely: slippage, deadlines, simulation and private transactions.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#40-defi-primitives-mev-from-a-go-integrators-view).

### 1. The primitive stack

_Not written yet._

### 2. Constant-product AMMs

_Not written yet._

### 3. Slippage and deadlines

_Not written yet._

### 4. Concentrated liquidity

_Not written yet._

### 5. Impermanent loss

_Not written yet._

### 6. Lending protocols

_Not written yet._

### 7. Oracles

_Not written yet._

### 8. MEV

_Not written yet._

### 9. Flash loans

_Not written yet._

### 10. Defensive integration in Go

_Not written yet._

### 11. Assets your service must model

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/40-defi-primitives-mev/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 6 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/40-defi-primitives-mev/` — **17 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
