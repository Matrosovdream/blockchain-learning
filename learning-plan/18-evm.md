# 18 — The EVM: a Stack Machine You Can Build

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#18-the-evm-a-stack-machine-you-can-build) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 4 — Ethereum & the EVM |
| **Prerequisites** | [17](17-rlp-merkle-patricia-trie.md) |
| **Unlocks** | 22, 46, 62, 65 |
| **Examples to build** | 22 (🟢 6 · 🟡 10 · 🔴 6) |
| **Topics in spec** | 11 |

*opcodes, stack, memory, storage, calldata, gas accounting — and a mini-EVM written in Go*

## Goals

- Explain the EVM's execution model: 256-bit stack, volatile memory, persistent storage.
- Read raw bytecode and disassemble it.
- Implement an interpreter for a useful subset of opcodes in Go.
- Trace gas consumption and explain why storage is expensive.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **11 topics** with their sub-points: [plan/part-04-ethereum-the-evm.md](plan/part-04-ethereum-the-evm.md#18-the-evm-a-stack-machine-you-can-build).

### 1. The machine

_Not written yet._

### 2. Why 256 bits

_Not written yet._

### 3. Data locations and their costs

_Not written yet._

### 4. Opcode families

_Not written yet._

### 5. JUMPDEST analysis

_Not written yet._

### 6. Contract creation

_Not written yet._

### 7. The call family

_Not written yet._

### 8. Gas mechanics

_Not written yet._

### 9. Writing an interpreter in Go

_Not written yet._

### 10. Reading real traces

_Not written yet._

### 11. The road ahead

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/18-evm/`. -->

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

**Examples:** `examples/18-evm/` — **22 runnable Go programs** (🟢 6 easy · 🟡 10 medium · 🔴 6 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
