# Blockchain Learning Plan

A step-by-step path from "what even is a block" to shipping production blockchain services. **Every example in this repo is Go.** Each step is a self-contained lesson with goals, long-form concept explanations, exercises, pitfalls, a checklist and resources — plus a library of small runnable Go programs graded easy → medium → hard.

Claude is the tutor: ask for a lesson to be written, ask for more examples, ask why anything works.

## How to use this plan

1. Work the steps **in order**. 01 → 30 is the spine; 31 → 41 is where it becomes a job.
2. For each step:
   - Read the lesson file (`NN-title.md`) end to end. The prose is the point — don't skim to the code.
   - Retype the examples in `examples/NN-title/` and run them. Retype, don't copy-paste.
   - Write the exercises in `practice/NN-title/`, then `gofmt` and `go vet` them.
   - Ask Claude to review your answers.
3. Tick the step off in [PROGRESS.md](PROGRESS.md).

Every lesson has a **Best Practices & Pitfalls** section. Blockchain punishes small mistakes with irreversible losses, so that section is not optional reading.

## The build plan

[PLAN.md](PLAN.md) is the authoring spec: for every one of the 41 lessons it lists the exact topics to cover, the example ideas, the prerequisites and the Go packages involved. That is the file to open when you want the next lesson written.

## Stack we're targeting

- **Go 1.22+** — standard library first, then the ecosystem libraries below
- **`github.com/ethereum/go-ethereum`** — used as a *library* (crypto, rlp, abi, core/types, ethclient), not just a node
- **`math/big` and `github.com/holiman/uint256`** — never `float64`, never `int64`, for money
- **Foundry (`anvil`, `forge`, `cast`) + `solc`** — for the contract side and for forked-chain tests
- **`go.etcd.io/bbolt`** for the from-scratch chain, **`database/sql`** + Postgres for the indexer
- **`log/slog`**, **`testing`** (table-driven + fuzz), **Prometheus** for the production lessons
- **`github.com/btcsuite/btcd`**, **Cosmos SDK**, **`gnark`** in Part 8

> **Note:** no Go module is scaffolded yet. When you reach Part 1 and want to run code:
> `mkdir practice && cd practice && go mod init github.com/Matrosovdream/blockchain-learning/practice`.
> Put each lesson's code in its own subfolder.

## Steps

Each line shows how many runnable Go examples that lesson gets. Once a lesson's examples
are built they live in [`examples/NN-title/`](examples/) and get linked here.

### Part 1 — Foundations

What a blockchain actually is, the tools you need, and the byte-level primitives every later lesson assumes. No cryptography yet — just the mental model and the keyboard setup.

- [01 — Introduction to Blockchain](01-introduction.md) — what problem a blockchain solves, the ledger, decentralization, the landscape — and what it is *not* · *12 examples planned*
- [02 — Environment Setup & Tooling](02-environment-setup.md) — Go toolchain, the module layout for this repo, node clients, testnets, faucets, and the CLI toolbox · *12 examples planned*
- [03 — Bytes, Hex, Big Integers & Encoding](03-bytes-encoding.md) — the data primitives — `[]byte`, endianness, hex, base58, base64, `big.Int`, and fixed-size arrays · *18 examples planned*

### Part 2 — Cryptography Foundations

The four crypto primitives every chain is built from: hashes, Merkle trees, signatures, and key derivation. You implement each one in Go before you ever see a block.

- [04 — Cryptographic Hash Functions](04-hash-functions.md) — SHA-256, Keccak-256, the five properties, commitments, and hashing structured data in Go · *22 examples planned*
- [05 — Merkle Trees & Proofs](05-merkle-trees.md) — building a tree, the root as a commitment, inclusion proofs, and the traps (odd nodes, second-preimage) · *22 examples planned*
- [06 — Keys & Digital Signatures (ECDSA on secp256k1)](06-keys-signatures.md) — private/public keys, the elliptic curve, signing, verifying, `v`-recovery, malleability and nonce disasters · *22 examples planned*
- [07 — Addresses, Encodings & HD Wallets](07-addresses-wallets-hd.md) — Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation paths · *22 examples planned*

### Part 3 — Build a Blockchain from Scratch (Go)

The core of the course. You write a working chain — blocks, mining, UTXO transactions, a wallet, persistence, a P2P network and fork choice — one lesson at a time, in plain Go.

- [08 — Blocks & the Chain](08-blocks-and-chain.md) — the block struct, hash linking, the genesis block, serialization and chain validation · *22 examples planned*
- [09 — Proof of Work & Mining](09-proof-of-work.md) — the difficulty target, nonce grinding, `bits`/compact encoding, difficulty retargeting and the energy argument · *22 examples planned*
- [10 — Transactions & the UTXO Model](10-transactions-utxo.md) — inputs, outputs, the coinbase, signing a transaction, script-less locking and the UTXO set · *22 examples planned*
- [11 — Wallets, Fees & the Mempool](11-wallets-mempool.md) — a keystore, address book, transaction construction, fee markets and mempool policy · *22 examples planned*
- [12 — Persistence & Chain State](12-persistence-chainstate.md) — storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety · *22 examples planned*
- [13 — P2P Networking & Gossip](13-p2p-networking.md) — peer discovery, a handshake, inventory gossip, block/tx propagation and a minimal node in Go · *22 examples planned*
- [14 — Consensus, Forks & Reorgs](14-consensus-forks.md) — fork choice, chain work, reorg handling, finality, and the attacks the rules exist to stop · *22 examples planned*
- [15 — The Account Model & World State](15-account-model-state.md) — accounts, nonces, balances, code and storage — the state trie and how Ethereum differs from UTXO · *22 examples planned*

### Part 4 — Ethereum & the EVM

From your toy chain to the real one. Ethereum's account model, RLP and the Patricia trie, the EVM as a stack machine (you build a mini one), the transaction types, and talking to a node from Go.

- [16 — Ethereum Architecture](16-ethereum-architecture.md) — the whole machine: accounts, blocks, receipts, gas, the execution/consensus split and the client stack · *18 examples planned*
- [17 — RLP & the Merkle Patricia Trie](17-rlp-merkle-patricia-trie.md) — Ethereum's serialization format and the trie that produces every root in the header · *22 examples planned*
- [18 — The EVM: a Stack Machine You Can Build](18-evm.md) — opcodes, the stack, memory, storage, calldata, gas accounting — and a mini-EVM written in Go · *26 examples planned*
- [19 — Ethereum Transactions Deep Dive](19-transaction-types.md) — legacy, 2930, 1559 and 4844 transaction types — signing, RLP, chain id, and computing the hash · *22 examples planned*
- [20 — JSON-RPC & go-ethereum's `ethclient`](20-json-rpc-ethclient.md) — the node API, `ethclient` from Go, queries, filters, subscriptions and the batching/rate-limit realities · *22 examples planned*
- [21 — Sending Transactions from Go](21-sending-transactions.md) — nonce management, gas estimation, 1559 fees, signing, broadcast, confirmation and stuck-tx recovery · *22 examples planned*

### Part 5 — Smart Contracts from Go

Enough Solidity to read a contract, then everything on the Go side: ABI encoding, `abigen` bindings, events and logs, the ERC standards, and the security bugs that drain contracts.

- [22 — Solidity Basics for Go Developers](22-solidity-basics.md) — enough Solidity to read, compile and reason about a contract — from a Go engineer's perspective · *18 examples planned*
- [23 — The Contract ABI: Encoding & Decoding](23-abi-encoding.md) — function selectors, head/tail encoding, dynamic types, and doing it all by hand in Go · *26 examples planned*
- [24 — Type-Safe Contracts with `abigen`](24-abigen-bindings.md) — generating Go bindings, deploying, calling, transacting, and the simulated backend · *22 examples planned*
- [25 — Events, Logs & Indexing Them](25-events-logs.md) — topics, the bloom filter, `eth_getLogs` at scale, decoding, and subscription vs polling · *26 examples planned*
- [26 — The ERC Standards from Go](26-erc-standards.md) — ERC-20, ERC-721, ERC-1155, ERC-165, permit and multicall — interacting with all of them · *22 examples planned*
- [27 — Smart Contract Security](27-contract-security.md) — the bug classes that drain contracts, how to spot them, and how to defend as an integrator · *22 examples planned*

### Part 6 — Consensus & Scaling

How agreement is actually reached at scale: Ethereum's proof of stake, the BFT family, and the layer-2 designs (rollups, channels, data availability) that everything is migrating to.

- [28 — Proof of Stake & Ethereum Consensus](28-proof-of-stake.md) — validators, slots and epochs, LMD-GHOST + Casper FFG, attestations, finality and slashing · *18 examples planned*
- [29 — Alternative Consensus: BFT, PoA & the Rest](29-alternative-consensus.md) — PBFT, Tendermint, HotStuff, Clique PoA, DPoS — what each assumes and what each buys · *15 examples planned*
- [30 — Layer 2 & Scaling](30-layer2-scaling.md) — rollups (optimistic and zk), data availability, blobs, channels, bridges — and what they mean for your Go code · *15 examples planned*

### Part 7 — Production Blockchain Engineering in Go

The job. Indexers that survive reorgs, key management that survives an audit, node operations, deterministic tests against a simulated chain, and the observability that tells you when RPC lies.

- [31 — Building a Blockchain Indexer in Go](31-blockchain-indexer.md) — ingesting blocks, reorg-safe writes, backfill, idempotency and the schema that makes queries possible · *26 examples planned*
- [32 — Key Management & Signing Services](32-key-management-signing.md) — keystores, KMS/HSM, hot vs cold, a signing service, and nonce management at scale · *22 examples planned*
- [33 — Node Operations & RPC Infrastructure](33-node-operations.md) — running geth, sync modes, storage, providers, failover and what breaks in production · *15 examples planned*
- [34 — Testing Blockchain Code in Go](34-testing-blockchain-go.md) — simulated backends, anvil forks, deterministic fixtures, fakes for RPC, and reorg tests · *22 examples planned*
- [35 — Observability & Reliability for Chain Services](35-observability-reliability.md) — metrics, tracing, alerting on lag and reorgs, retries, and designing for RPC that lies · *15 examples planned*

### Part 8 — Beyond Ethereum

The rest of the landscape, each through a Go lens: Bitcoin's script and Taproot, Cosmos app-chains (written in Go), Solana's account model, zero-knowledge proofs with `gnark`, and DeFi/MEV.

- [36 — Bitcoin Deep Dive with Go](36-bitcoin-deep-dive.md) — Script, P2PKH/P2SH/SegWit/Taproot, sighash types, PSBT and the `btcsuite` libraries · *22 examples planned*
- [37 — Cosmos SDK & CometBFT: App-Chains in Go](37-cosmos-tendermint.md) — building your own chain in Go — modules, keepers, ABCI, staking and IBC · *15 examples planned*
- [38 — Solana & Other Execution Models](38-solana-other-vms.md) — the accounts model, parallel execution, and talking to non-EVM chains from Go · *15 examples planned*
- [39 — Zero-Knowledge Proofs with `gnark`](39-zero-knowledge-proofs.md) — commitments, circuits, SNARKs vs STARKs, and writing and proving a circuit in Go · *15 examples planned*
- [40 — DeFi Primitives & MEV from a Go Integrator's View](40-defi-primitives-mev.md) — AMMs, lending, oracles, liquidations, MEV and flash loans — the math and the integration code · *22 examples planned*

### Part 9 — Capstone

One end-to-end system that uses the whole course.

- [41 — Capstone Project](41-capstone.md) — one end-to-end system: indexer + API + wallet service + contract integration, in Go

## Progress

See [PROGRESS.md](PROGRESS.md) for the current step and notes from past lessons.
