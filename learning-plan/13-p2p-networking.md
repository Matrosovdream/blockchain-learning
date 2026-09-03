# 13 — P2P Networking & Gossip

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#13-p2p-networking-gossip) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [12](12-persistence-chainstate.md) |
| **Unlocks** | 14 |
| **Examples to build** | 18 (🟢 5 · 🟡 8 · 🔴 5) |
| **Topics in spec** | 9 |

*peer discovery, a handshake, inventory gossip, block/tx propagation and a minimal node in Go*

## Goals

- Write a minimal P2P node in Go: listen, dial, handshake, exchange messages.
- Implement inventory-based gossip so blocks and transactions propagate without flooding.
- Sync a new node from a peer's chain.
- Explain devp2p and libp2p at a high level.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **9 topics** with their sub-points: [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#13-p2p-networking-gossip).

### 1. The network layer as a separate concern

_Not written yet._

### 2. Message framing over TCP

_Not written yet._

### 3. Handshake and version negotiation

_Not written yet._

### 4. Peer lifecycle in Go

_Not written yet._

### 5. Gossip that does not melt the network

_Not written yet._

### 6. Initial block download

_Not written yet._

### 7. Orphans and future blocks

_Not written yet._

### 8. Discovery

_Not written yet._

### 9. Adversarial networking

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/13-p2p-networking/`. -->

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

**Examples:** `examples/13-p2p-networking/` — **18 runnable Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
