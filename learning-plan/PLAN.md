# Build Plan — Blockchain Learning (Go)

The **authoring spec** for the course. [README.md](README.md) is what a reader opens; this is what you open when you want the next lesson written.

| | |
|---|---|
| Lessons | **68**, in 16 parts |
| Core spine | 01 → 41, strictly in order |
| Extensions | 42 → 68, take in any order once their prerequisites are met |
| Topics specified | **633**, with **2623** sub-points |
| Runnable Go examples planned | **1072** |
| Named pitfalls | 324 |
| Cited specs & references | 290 |

Every code example in this course is **Go**. Solidity appears only as the thing Go talks to.

## The spec is split by part

Each file below holds the full spec for its lessons: goals, the numbered topic list with sub-points, the pitfalls to cover, tiered example seeds, packages and resources.

| Part | Lessons | Spec file | Ex. |
|---|---|---|---|
| Part 1 — Foundations | 01–03 | [part-01-foundations](plan/part-01-foundations.md) | 42 |
| Part 2 — Cryptography Foundations | 04–07 | [part-02-cryptography-foundations](plan/part-02-cryptography-foundations.md) | 74 |
| Part 3 — Build a Blockchain from Scratch (Go) | 08–15 | [part-03-build-a-blockchain-from-scratch-go](plan/part-03-build-a-blockchain-from-scratch-go.md) | 144 |
| Part 4 — Ethereum & the EVM | 16–21 | [part-04-ethereum-the-evm](plan/part-04-ethereum-the-evm.md) | 110 |
| Part 5 — Smart Contracts from Go | 22–27 | [part-05-smart-contracts-from-go](plan/part-05-smart-contracts-from-go.md) | 109 |
| Part 6 — Consensus & Scaling | 28–30 | [part-06-consensus-scaling](plan/part-06-consensus-scaling.md) | 43 |
| Part 7 — Production Blockchain Engineering in Go | 31–35 | [part-07-production-blockchain-engineering-in-go](plan/part-07-production-blockchain-engineering-in-go.md) | 85 |
| Part 8 — Beyond Ethereum | 36–40 | [part-08-beyond-ethereum](plan/part-08-beyond-ethereum.md) | 76 |
| Part 9 — Capstone | 41–41 | [part-09-capstone](plan/part-09-capstone.md) | 0 |
| Part 10 — Cryptography, Deeper ·  *ext* | 42–44 | [part-10-cryptography-deeper](plan/part-10-cryptography-deeper.md) | 45 |
| Part 11 — Smart Contracts, Deeper ·  *ext* | 45–49 | [part-11-smart-contracts-deeper](plan/part-11-smart-contracts-deeper.md) | 77 |
| Part 12 — Identity, Wallets & dApp Backends ·  *ext* | 50–53 | [part-12-identity-wallets-dapp-backends](plan/part-12-identity-wallets-dapp-backends.md) | 60 |
| Part 13 — Chain Data at Scale ·  *ext* | 54–57 | [part-13-chain-data-at-scale](plan/part-13-chain-data-at-scale.md) | 60 |
| Part 14 — Custody, Payments & Compliance ·  *ext* | 58–61 | [part-14-custody-payments-compliance](plan/part-14-custody-payments-compliance.md) | 58 |
| Part 15 — Protocol Internals & go-ethereum ·  *ext* | 62–66 | [part-15-protocol-internals-go-ethereum](plan/part-15-protocol-internals-go-ethereum.md) | 63 |
| Part 16 — Cross-Chain & Production Operations ·  *ext* | 67–68 | [part-16-cross-chain-production-operations](plan/part-16-cross-chain-production-operations.md) | 26 |

[plan/backlog.md](plan/backlog.md) holds topics that are not scheduled yet, and the ones deliberately rejected.

---

## Authoring rules

### Lesson anatomy

Every `NN-title.md` has exactly these sections, in this order:

| Section | What goes in it |
|---|---|
| `# NN — Title` | status line, a facts table (part / prereqs / unlocks / example count), one-line blurb |
| `## Goals` | 4–5 bullets, each an observable capability |
| `## Concepts` | **the bulk of the lesson** — one `###` per numbered topic in the spec, in order |
| `## Exercises` | 5–8 numbered tasks the reader writes in `practice/NN-title/` |
| `## Best Practices & Pitfalls` | the spec's pitfalls plus the habits, each with a one-line *why* |
| `## Checklist` | `- [ ]` "I can …" lines mirroring the goals |
| `## Resources` | specs/EIPs/BIPs first, then reference implementations, then articles |

### Writing style

- **Explanation-heavy.** This course is read, not skimmed. Every topic gets prose *before* any code: what problem it solves, what breaks without it, then the mechanism.
- **Why before how.** "Nonces exist because the account model has no unique outputs to spend" beats "a nonce is a counter".
- Cover **every sub-point** the spec lists for a topic. They are the detail the lesson owes the reader.
- Short Go snippets inside `## Concepts` (5–20 lines). Full programs live in the example files.
- Name the real incident whenever one exists — the DAO, Parity, the PS3 nonce reuse, Ronin, Nomad. Concrete failures are what stick.
- `big.Int`/`uint256` for every on-chain value. A `float64` in a money path is a bug in a lesson too.
- Cross-link prerequisites with a relative link to the other lesson file.
- Length guide: a 10-topic lesson lands around 400–700 lines. Longer is fine; thinner is not.

### Example files

Each lesson gets `examples/NN-title/`:

```
README.md     index + how to run + the tier table
1-easy.md     🟢 one concept in isolation
2-medium.md   🟡 concepts combined, and the traps
3-hard.md     🔴 real-shaped, multi-concept programs
PROGRESS.md   a checkbox per example
```

Copy [`examples/_template/`](examples/_template/) to start one.

Rules for examples:

1. Each is a **complete `package main` program** — no fragments, no `...`.
2. Each has: a `## N. Title` heading, a tier + category line, 2–4 sentences of concept, numbered **Steps**, the code block, and a real **Output** block.
3. **Run it before adding it.** `go build` + `go vet` + `gofmt` clean; the Output block is real stdout.
4. Chain-dependent examples use `ethclient/simulated` or a local `anvil` — never mainnet, never a real key.
5. Any key material is a hardcoded test key with a comment saying so.
6. Numbering is continuous across the three tier files (1 → N) and is never renumbered once published.
7. The tier split in each spec (🟢/🟡/🔴 counts) is a target, not a cap — append more to any tier later.

### Build order

Write in numeric order through the spine; the prerequisites are real and Part 3 is one program growing across eight lessons. Extensions (42+) can be written whenever their prerequisites are done.

Within one lesson: `## Concepts` → 🟢 easy → 🟡 medium → 🔴 hard → `## Exercises` → the rest.

---

## Extending the course

The structure is built to grow. Four ways to extend it, smallest first.

### 1. Add examples to an existing lesson

Append to the right tier file, continue the numbering, add the entry to that folder's `README.md` index and `PROGRESS.md`. Nothing else changes. **Run it first.**

### 2. Add a topic to an existing lesson

Add a numbered entry with its sub-points to that lesson's section in its part spec file, then add the matching `###` sub-section to the lesson. Bump the topic count in the lesson's facts table.

### 3. Add a lesson

1. Give it the next free number (69, 70, …) — **numbers are never reused or renumbered**, because example folders, `practice/` folders and cross-links all reference them.
2. Add its full spec to the relevant part file: goals, numbered topics with sub-points, pitfalls, tiered example seeds, packages, resources.
3. Create `learning-plan/NN-slug.md` from the shape above.
4. Add it to [README.md](README.md) under its part, and add a row to [PROGRESS.md](PROGRESS.md).
5. If anything depends on it, add it to that lesson's prerequisites and its own "Unlocks" row.

### 4. Add a part

Create `plan/part-NN-slug.md` with the same layout as its siblings, add a row to the table above, and add the part heading to [README.md](README.md). Parts are numbered in order; extensions carry the *ext* marker.

### Keeping it honest

- A lesson is only *written* when all its sections are filled and its examples run.
- A topic with no sub-points in the spec is not specified yet — write the sub-points first.
- Ideas without a number go in [plan/backlog.md](plan/backlog.md), with a reason.
- Reject topics that cannot carry a Go example, and record why in the backlog's rejected table.

---

## Cheatsheets

Once a group of lessons is written, compress it into one dense sheet in [cheatsheets/](cheatsheets/) — API surface, byte layouts, formulas and traps, no prose. Planned groupings:

| Sheet | Lessons | What it holds |
|---|---|---|
| `01-03-foundations.md` | 01–03 | the model, tooling, bytes/hex/`big.Int`/units |
| `04-05-hashes-merkle.md` | 04–05 | hash APIs, Keccak vs SHA-3, Merkle construction & proof rules |
| `06-07-42-43-keys.md` | 06–07, 42–43 | ECDSA/Schnorr/BLS/ed25519, derivation paths, MuSig, Shamir, MPC |
| `08-12-chain-core.md` | 08–12 | block/header fields, PoW targets, UTXO, mempool policy, storage keys |
| `13-15-network-state.md` | 13–15 | framing, gossip, fork choice, the reorg algorithm, account state |
| `16-17-ethereum-encoding.md` | 16–17 | header fields, gas & 1559 math, RLP rules, MPT node types |
| `18-65-evm.md` | 18, 65 | opcode table, gas costs, call semantics, precompiles |
| `19-21-transactions.md` | 19–21 | the four tx types, signing hashes, fee fields, send-error taxonomy |
| `20-33-rpc.md` | 20, 33 | JSON-RPC methods, block tags, ethclient API, provider limits |
| `22-26-45-49-contracts.md` | 22–26, 45, 49 | Solidity↔Go type map, ABI layout, selectors, ERC surfaces |
| `27-46-security.md` | 27, 46 | vulnerability classes, proxy slots, the integrator checklist |
| `28-30-consensus-l2.md` | 28–30 | PoS terms, fork choice, BFT bounds, rollup/L2 comparison |
| `31-35-57-production.md` | 31, 35, 57 | the reorg algorithm, idempotency keys, metric list, pprof recipes |
| `32-44-58-61-custody.md` | 32, 44, 58–61 | keystore format, AEAD rules, deposit/withdrawal states, ledger invariants |
| `34-48-testing.md` | 34, 48 | simulated-backend API, anvil cheatcodes, forge cheatcodes, fuzz recipes |
| `36-38-other-chains.md` | 36–38 | Bitcoin script/sighash/weight, Cosmos ABCI, Solana accounts |
| `39-64-66-proofs.md` | 39, 64, 66 | SNARK pipeline, gnark API, proof verification, witness sizes |
| `40-53-defi-oracles.md` | 40, 53 | AMM formulas, health factors, feed safety checks, MEV taxonomy |
| `50-52-signatures-identity.md` | 50–52 | EIP-191/712/1271/4361 layouts, namehash, ENS resolution rules |
| `54-56-67-68-data-ops.md` | 54–56, 67, 68 | mempool APIs, ETL shapes, bridge models, deploy checklist |

*Reader index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Backlog: [plan/backlog.md](plan/backlog.md)*
