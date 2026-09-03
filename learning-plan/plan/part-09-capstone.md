# Part 9 — Capstone

One end-to-end system that uses the whole spine.

**Core spine.** Lessons 41–41 · 0 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 41 | [Capstone Project](#41-capstone-project) | 31, 32, 34, 35 | — |

---

## 41 — Capstone Project

**Lesson file:** [../41-capstone.md](../41-capstone.md) · **Examples folder:** `../examples/41-capstone/`

| | |
|---|---|
| Prerequisites | [31](../31-blockchain-indexer.md), [32](../32-key-management-signing.md), [34](../34-testing-blockchain-go.md), [35](../35-observability-reliability.md) |
| Unlocks | — |
| Examples | — (project deliverable) |
| Topics | 8 |

*one end-to-end system: indexer + API + signing service + contract integration, in Go*

### Goals

- Ship one production-shaped blockchain service in Go.
- Combine indexing, signing, contract interaction and observability.
- Handle reorgs, retries and restarts correctly, proven by tests.
- Document the trust assumptions and failure modes.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Choose one**

   - (a) ERC-20 payment gateway: per-user deposit addresses, detection, sweeps, withdrawals, ledger.
   - (b) NFT marketplace indexer: collections, ownership, listings, sales, a REST/GraphQL API.
   - (c) DAO treasury dashboard: multisig state, proposals, token positions, valuations.
   - (d) Finish your Part 3 chain: P2P, wallet CLI, block explorer, and a public testnet.

**2. Required components**

   - An ingestion pipeline with a confirmation lag and reorg handling (lesson 31).
   - A signing boundary with policy — no key in the application process (lesson 32).
   - Contract interaction through generated bindings behind a domain interface (lessons 24, 26).
   - A query API (REST or gRPC) with pagination and stable ordering.
   - Metrics, alerts and a reconciliation job (lesson 35).

**3. Non-functional requirements**

   - Restart-safe: kill it at any point, restart, no duplicate or lost effects.
   - Reorg-safe: a 10-block reorg leaves correct state.
   - Idempotent: replaying any range changes nothing.
   - Rate-limit-aware: it degrades rather than dies when the provider throttles.
   - Secret-safe: no key material in code, logs, or the container image.

**4. Architecture**

   - Hexagonal boundaries: the chain is an adapter, not a dependency of your domain.
   - `cmd/` binaries: indexer, api, signer, worker. `internal/` domain. `pkg/` only if genuinely shared.
   - Dependency injection at the composition root; no globals.
   - A `ChainSource` and a `Signer` interface, each with a fake for tests.

**5. The test plan**

   - Unit tests for all pure logic (encoding, fees, selection, health factors).
   - Simulated-backend tests for every contract interaction.
   - A scripted reorg test through the fake chain source.
   - A forked-chain integration test for at least one real-world contract.
   - `go test -race` clean; a fuzz target on your decoders.

**6. Deployment**

   - Multi-stage Dockerfile, non-root, distroless or scratch.
   - Config entirely from environment; fail fast on anything missing.
   - `/healthz` (liveness) and `/readyz` (ready, and 503 while draining).
   - Graceful shutdown on SIGTERM that commits the cursor and drains sends (lesson 68).

**7. The README that proves you understood**

   - What the system does, in three sentences.
   - The trust assumptions: which provider, which contracts, which keys, what happens if each is compromised.
   - The failure modes and what the system does in each.
   - How to run it locally in one command, and how to run the tests.

**8. A security review pass**

   - Walk your own code against lesson 27's integrator checklist and lesson 32's key checklist.
   - Check every external call for: value bounds, gas bounds, destination allowlist, receipt verification.
   - Confirm no secret reaches logs, metrics labels, or error strings.
   - Write down what you found — that document is the deliverable.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Skipping the reorg test because 'it probably works'.
- Putting the signing key in the same process as the API.
- Building aggregates that cannot be rolled back.
- Shipping without a reconciliation job, so silent drift is invisible.

### Deliverable

- Deliverables live under `projects/capstone/`. No graded example tiers — this is a build.

### Packages & tools

`everything from lessons 20–35`

### Resources to cite

- Standard Go project layout discussion: https://go.dev/doc/modules/layout
- The Twelve-Factor App: https://12factor.net/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
