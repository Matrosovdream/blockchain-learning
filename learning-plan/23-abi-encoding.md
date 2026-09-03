# 23 — The Contract ABI: Encoding & Decoding

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#23-the-contract-abi-encoding-decoding) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 5 — Smart Contracts from Go |
| **Prerequisites** | [22](22-solidity-basics.md) |
| **Unlocks** | 24, 25, 47, 50 |
| **Examples to build** | 20 (🟢 5 · 🟡 9 · 🔴 6) |
| **Topics in spec** | 10 |

*function selectors, head/tail layout, dynamic types, revert data and EIP-712 — by hand and with `abi`*

## Goals

- Compute a function selector and encode arguments by hand.
- Explain the head/tail layout for dynamic types.
- Encode and decode with go-ethereum's `abi` package.
- Decode an unknown calldata blob and a revert reason.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#23-the-contract-abi-encoding-decoding).

### 1. The ABI is a calling convention

_Not written yet._

### 2. Function selectors

_Not written yet._

### 3. Static encoding

_Not written yet._

### 4. Dynamic encoding — head and tail

_Not written yet._

### 5. Nested dynamic types

_Not written yet._

### 6. Tuples and Go structs

_Not written yet._

### 7. Return data and revert data

_Not written yet._

### 8. The Go API

_Not written yet._

### 9. encodePacked and its hazard

_Not written yet._

### 10. EIP-712 typed structured data

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/23-abi-encoding/`. -->

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

**Examples:** `examples/23-abi-encoding/` — **20 runnable Go programs** (🟢 5 easy · 🟡 9 medium · 🔴 6 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
