# 26 — The ERC Standards from Go

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#26-the-erc-standards-from-go) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 5 — Smart Contracts from Go |
| **Prerequisites** | [24](24-abigen-bindings.md), [25](25-events-logs.md) |
| **Unlocks** | 27, 40, 49, 52, 55 |
| **Examples to build** | 19 (🟢 5 · 🟡 9 · 🔴 5) |
| **Topics in spec** | 9 |

*ERC-20, ERC-721, ERC-1155, ERC-165 and Multicall — including every non-standard token that breaks your code*

## Goals

- Interact with ERC-20, ERC-721 and ERC-1155 tokens from Go.
- Handle decimals, non-standard tokens and missing return values.
- Detect interface support with ERC-165.
- Batch reads to cut RPC calls by orders of magnitude.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#26-the-erc-standards-from-go).

### 1. Why standards exist

_Not written yet._

### 2. ERC-20

_Not written yet._

### 3. The non-standard token minefield

_Not written yet._

### 4. Decimals are display-only

_Not written yet._

### 5. ERC-721

_Not written yet._

### 6. ERC-1155

_Not written yet._

### 7. ERC-165 interface detection

_Not written yet._

### 8. Batching reads

_Not written yet._

### 9. Token accounting for an indexer

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/26-erc-standards/`. -->

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

**Examples:** `examples/26-erc-standards/` — **19 runnable Go programs** (🟢 5 easy · 🟡 9 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
