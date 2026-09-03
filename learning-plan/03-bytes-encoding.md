# 03 — Bytes, Hex, Big Integers & Encoding

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-01-foundations.md](plan/part-01-foundations.md#03-bytes-hex-big-integers-encoding) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 1 — Foundations |
| **Prerequisites** | [02](02-environment-setup.md) |
| **Unlocks** | 04 |
| **Examples to build** | 18 (🟢 6 · 🟡 7 · 🔴 5) |
| **Topics in spec** | 9 |

*the data primitives — `[]byte`, endianness, hex, base58, bech32, `big.Int`, fixed-size arrays*

## Goals

- Move fluently between `[]byte`, hex strings and integers in Go.
- Explain endianness and pick the right one when serializing.
- Use `math/big`/`uint256` for values that overflow `uint64` — which is most of them.
- Know why `[32]byte` and `[]byte` are different types and where each is used.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-01-foundations.md](plan/part-01-foundations.md#03-bytes-hex-big-integers-encoding).

### 1. Everything is bytes

_Not written yet._

### 2. Hex encoding

_Not written yet._

### 3. Endianness

_Not written yet._

### 4. Big integers

_Not written yet._

### 5. uint256

_Not written yet._

### 6. Fixed-width serialization

_Not written yet._

### 7. Address and hash encodings

_Not written yet._

### 8. Units and money formatting

_Not written yet._

### 9. Constant-time comparison

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/03-bytes-encoding/`. -->

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

**Examples:** `examples/03-bytes-encoding/` — **18 runnable Go programs** (🟢 6 easy · 🟡 7 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
