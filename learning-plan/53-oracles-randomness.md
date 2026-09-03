# 53 — Oracles, Price Feeds & On-Chain Randomness

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#53-oracles-price-feeds-on-chain-randomness) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 12 — Identity, Wallets & dApp Backends |
| **Prerequisites** | [40](40-defi-primitives-mev.md) |
| **Unlocks** | — |
| **Examples to build** | 15 (🟢 4 · 🟡 8 · 🔴 3) |
| **Topics in spec** | 9 |

*getting off-chain truth on-chain safely — Chainlink, TWAPs, VRF, RANDAO, and running your own*

## Goals

- Read a price feed correctly, with every safety check.
- Compare push and pull oracle designs and their failure modes.
- Explain VRF and RANDAO, and why naive on-chain randomness is exploitable.
- Build a minimal oracle service in Go.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#53-oracles-price-feeds-on-chain-randomness).

### 1. The oracle problem

_Not written yet._

### 2. Chainlink price feeds

_Not written yet._

### 3. Push vs pull oracles

_Not written yet._

### 4. AMM TWAPs

_Not written yet._

### 5. Oracle failure modes

_Not written yet._

### 6. On-chain randomness

_Not written yet._

### 7. Verifiable random functions

_Not written yet._

### 8. Building your own oracle service in Go

_Not written yet._

### 9. Other oracle types

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/53-oracles-randomness/`. -->

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

**Examples:** `examples/53-oracles-randomness/` — **15 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
