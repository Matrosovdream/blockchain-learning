# blockchain-learning

My notes and practice for learning blockchain engineering from scratch — **every example in Go**.

## The lessons

**68 lessons** in [learning-plan/](learning-plan/), numbered 01 to 68, strictly easy → hard.

- **01 → 41 is the spine**, taken in order: what a ledger is → cryptography (hashes, Merkle trees, ECDSA,
  HD wallets) → **building a working blockchain from scratch in Go** (blocks, mining, UTXO, wallet,
  persistence, P2P, fork choice) → Ethereum and the EVM (including writing your own interpreter) →
  smart-contract interaction from Go → consensus and L2s → production engineering (indexers, key
  management, node ops, testing, observability) → Bitcoin, Cosmos, Solana, ZK, DeFi → a capstone.
- **42 → 68 are extensions**, taken in any order once their prerequisites are met: deeper cryptography
  (Schnorr/BLS, MPC, encryption at rest), smart contracts in depth (gas optimization, proxies, account
  abstraction, Foundry, the wider token standards), identity and dApp backends (SIWE, wallets, ENS,
  oracles), chain data at scale (mempool, IPFS, ETL, performance), custody and compliance (deposits,
  withdrawals, double-entry ledgers, AML), protocol internals (reading go-ethereum, private networks,
  light clients, precompiles, Verkle), and cross-chain plus production operations.

Each lesson is one markdown file with heavy prose explanations, exercises, pitfalls and a checklist.
Most lessons also get a set of small runnable Go programs under
[learning-plan/examples/](learning-plan/examples/), graded 🟢 easy → 🟡 medium → 🔴 hard.
**1072 examples** are planned across the course.

Where I'm at is tracked in [learning-plan/PROGRESS.md](learning-plan/PROGRESS.md).

## How to use it

1. Open the next lesson, e.g. [learning-plan/04-hash-functions.md](learning-plan/04-hash-functions.md).
2. Read it — the prose is the point, don't skim to the code.
3. Retype the examples into a scratch folder and run them:
   ```bash
   mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
   go mod init scratch        # first time only
   # paste an example into main.go, then:
   go run .
   ```
4. Write your own answers to the exercises in [practice/](practice/) — one folder per lesson.
5. Update PROGRESS.md when a lesson is done.

## Building the course

Nothing is written yet — the repo holds the structure, 68 lesson stubs, and the full plan.

- [learning-plan/PLAN.md](learning-plan/PLAN.md) — the authoring spec: conventions, writing rules, and
  **how to extend the course** (add examples, topics, lessons or whole parts).
- [learning-plan/plan/](learning-plan/plan/) — one spec file per part, holding every lesson's goals,
  numbered topics with sub-points, pitfalls to cover, tiered example seeds, packages and resources.
  **633 topics with 2623 sub-points** are specified.
- [learning-plan/plan/backlog.md](learning-plan/plan/backlog.md) — unscheduled ideas, and the ones
  deliberately rejected.

Ask Claude to write the next lesson from its spec, or to append more examples to an existing tier file.

## Safety rules for this repo

- **No mainnet.** Examples run against `ethclient/simulated`, a local `anvil`, or a testnet.
- **No real keys.** Any key material in a lesson is a hardcoded test key, labelled as such.
- **No `float64` for money.** `math/big` or `uint256`, always.
