# 36 — Bitcoin Deep Dive with Go

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#36-bitcoin-deep-dive-with-go) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 8 — Beyond Ethereum |
| **Prerequisites** | [10](10-transactions-utxo.md), [11](11-wallets-mempool.md) |
| **Unlocks** | — |
| **Examples to build** | 17 (🟢 4 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 10 |

*Script, P2PKH→Taproot, sighash types, weight units, PSBT and the `btcsuite` libraries*

## Goals

- Read and evaluate Bitcoin Script.
- Distinguish the output types and build each in Go.
- Explain SegWit's txid/wtxid split and Taproot's key/script paths.
- Build, sign and inspect transactions with `btcd`/`btcutil`.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-08-beyond-ethereum.md](plan/part-08-beyond-ethereum.md#36-bitcoin-deep-dive-with-go).

### 1. Bitcoin as the reference implementation

_Not written yet._

### 2. Script

_Not written yet._

### 3. Standard output types

_Not written yet._

### 4. Sighash types

_Not written yet._

### 5. Malleability and SegWit

_Not written yet._

### 6. Weight units and fees

_Not written yet._

### 7. Taproot

_Not written yet._

### 8. Timelocks

_Not written yet._

### 9. PSBT

_Not written yet._

### 10. The Go ecosystem

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/36-bitcoin-deep-dive/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 6 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/36-bitcoin-deep-dive/` — **17 runnable Go programs** (🟢 4 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
