# 07 — Addresses, Encodings & HD Wallets

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#07-addresses-encodings-hd-wallets) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [06](06-keys-signatures.md) |
| **Unlocks** | 11 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 9 |

*Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation*

## Goals

- Derive an Ethereum address from a public key by hand, in Go.
- Implement and verify the EIP-55 mixed-case checksum.
- Generate a BIP-39 mnemonic and derive keys along a BIP-44 path.
- Explain xpub/xprv, hardened derivation, and the xpub-plus-child-key leak.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#07-addresses-encodings-hd-wallets).

### 1. Ethereum address derivation

_Not written yet._

### 2. EIP-55 checksums

_Not written yet._

### 3. Bitcoin address formats

_Not written yet._

### 4. BIP-39 mnemonics

_Not written yet._

### 5. BIP-32 hierarchical deterministic keys

_Not written yet._

### 6. Hardened vs non-hardened derivation

_Not written yet._

### 7. BIP-44 and friends

_Not written yet._

### 8. Contract and vanity addresses

_Not written yet._

### 9. Storing keys

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/07-addresses-wallets-hd/`. -->

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

**Examples:** `examples/07-addresses-wallets-hd/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
