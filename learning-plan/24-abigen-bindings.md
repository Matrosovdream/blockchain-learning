# 24 — Type-Safe Contracts with `abigen`

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#24-type-safe-contracts-with-abigen) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 5 — Smart Contracts from Go |
| **Prerequisites** | [23](23-abi-encoding.md) |
| **Unlocks** | 26, 34 |
| **Examples to build** | 16 (🟢 5 · 🟡 7 · 🔴 4) |
| **Topics in spec** | 9 |

*generating Go bindings, deploying, calling, transacting, and the simulated backend*

## Goals

- Generate Go bindings for a contract with `abigen`.
- Deploy and interact with a contract entirely from Go.
- Use `CallOpts`/`TransactOpts` correctly, including historical reads.
- Test contract interaction against a simulated backend with no node running.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-05-smart-contracts-from-go.md](plan/part-05-smart-contracts-from-go.md#24-type-safe-contracts-with-abigen).

### 1. What abigen produces

_Not written yet._

### 2. The toolchain

_Not written yet._

### 3. CallOpts

_Not written yet._

### 4. TransactOpts

_Not written yet._

### 5. Deploying from Go

_Not written yet._

### 6. Events through bindings

_Not written yet._

### 7. The simulated backend

_Not written yet._

### 8. When bindings get in the way

_Not written yet._

### 9. Keeping generated code sane

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/24-abigen-bindings/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 5 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 3 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/24-abigen-bindings/` — **16 runnable Go programs** (🟢 5 easy · 🟡 7 medium · 🔴 4 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
