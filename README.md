# blockchain-learning

My notes and practice for learning blockchain engineering from scratch — **every example in Go**.

## The lessons

41 lessons in [learning-plan/](learning-plan/), numbered 01 to 41, strictly easy → hard. They run from
what a ledger is, through the cryptography (hashes, Merkle trees, ECDSA, HD wallets), into **building a
working blockchain from scratch in Go** (blocks, mining, UTXO transactions, a wallet, persistence, P2P,
fork choice), then Ethereum and the EVM, smart-contract interaction from Go, consensus and L2s, and
finally the production side — indexers, key management, node ops, testing and observability.

Each lesson is one markdown file with heavy prose explanations, exercises, pitfalls and a checklist.
Most lessons also get a set of small runnable Go programs under
[learning-plan/examples/](learning-plan/examples/), graded 🟢 easy → 🟡 medium → 🔴 hard.

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

Nothing is written yet — the repo currently holds the structure, 41 lesson stubs, and the plan.

[learning-plan/PLAN.md](learning-plan/PLAN.md) is the authoring spec: for every lesson it lists the
exact topics to cover, the example seeds, the prerequisites and the Go packages involved. Ask Claude to
write the next lesson from it, or to append more examples to an existing tier file.

## Safety rules for this repo

- **No mainnet.** Examples run against `ethclient/simulated`, a local `anvil`, or a testnet.
- **No real keys.** Any key material in a lesson is a hardcoded test key, labelled as such.
- **No `float64` for money.** `math/big` or `uint256`, always.
