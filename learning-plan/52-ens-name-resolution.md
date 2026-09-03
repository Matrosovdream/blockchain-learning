# 52 — ENS & Name Resolution

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#52-ens-name-resolution) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 12 — Identity, Wallets & dApp Backends |
| **Prerequisites** | [26](26-erc-standards.md) |
| **Unlocks** | — |
| **Examples to build** | 14 (🟢 4 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 9 |

*forward and reverse resolution, namehash, wildcard/CCIP-Read, and doing it correctly from Go*

## Goals

- Implement namehash and resolve an ENS name to an address in Go.
- Do reverse resolution correctly, including the forward-check.
- Handle avatars, text records and content hashes.
- Understand CCIP-Read and offchain/L2 names.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-12-identity-wallets-dapp-backends.md](plan/part-12-identity-wallets-dapp-backends.md#52-ens-name-resolution).

### 1. Why names

_Not written yet._

### 2. Namehash and normalization

_Not written yet._

### 3. Forward resolution

_Not written yet._

### 4. Reverse resolution

_Not written yet._

### 5. Text records and avatars

_Not written yet._

### 6. Content hashes

_Not written yet._

### 7. Wildcard resolution and CCIP-Read

_Not written yet._

### 8. Expiry and ownership

_Not written yet._

### 9. Other naming systems

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/52-ens-name-resolution/`. -->

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

**Examples:** `examples/52-ens-name-resolution/` — **14 runnable Go programs** (🟢 4 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
