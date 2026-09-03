# Blockchain Learning Plan

A step-by-step path from "what even is a block" to shipping production blockchain services. **Every example in this repo is Go.**

**68 lessons · 1072 runnable Go examples · 633 specified topics.** Each lesson is a self-contained file with long-form explanations, exercises, pitfalls, a checklist and resources, plus a library of small runnable programs graded 🟢 easy → 🟡 medium → 🔴 hard.

Claude is the tutor: ask for a lesson to be written, ask for more examples, ask why anything works.

## How to use this plan

1. Work the **spine (01 → 41)** in order. The extensions (42 → 68) can be taken in any order once their prerequisites are done — each lesson lists them.
2. For each step:
   - Read the lesson file end to end. The prose is the point; don't skim to the code.
   - Retype the examples in `examples/NN-title/` and run them. Retype, don't paste.
   - Write the exercises in `practice/NN-title/`, then `gofmt` and `go vet` them.
   - Ask Claude to review your answers.
3. Tick the step off in [PROGRESS.md](PROGRESS.md).

Every lesson has a **Best Practices & Pitfalls** section. Blockchain punishes small mistakes with irreversible losses, so that section is not optional reading.

## Building the course

[PLAN.md](PLAN.md) is the authoring spec — conventions, the writing rules, and how to extend the course. The per-lesson detail lives in [plan/](plan/), one file per part: topics with sub-points, pitfalls, example seeds, packages and resources.

## Stack we're targeting

- **Go 1.22+** — standard library first, then the ecosystem below
- **`github.com/ethereum/go-ethereum`** as a *library* (crypto, rlp, abi, core/types, core/vm, trie, ethclient)
- **`math/big`** and **`github.com/holiman/uint256`** — never `float64`, never `int64`, for money
- **Foundry (`anvil`, `forge`, `cast`) + `solc`** for the contract side and forked-chain tests
- **`go.etcd.io/bbolt`** for the from-scratch chain; **`database/sql`** + Postgres (`pgx`) for the indexer
- **`log/slog`**, **`testing`** (table-driven + fuzz), **Prometheus**, **OpenTelemetry** for production
- **`btcd`**, **Cosmos SDK**, **`solana-go`**, **`gnark`** in Part 8; **`geth` internals** in Part 15

> **Note:** no Go module is scaffolded yet. When you reach Part 1 and want to run code:
> `cd practice && go mod init github.com/Matrosovdream/blockchain-learning/practice`.
> Put each lesson's code in its own subfolder.

## Steps

Each line shows how many runnable Go examples that lesson gets. Example folders are linked once they are built.

### Part 1 — Foundations

What a blockchain actually is, the tools you need, and the byte-level primitives every later lesson assumes. No cryptography yet — just the mental model and the keyboard setup.

- [01 — Introduction to Blockchain](01-introduction.md) — what problem a blockchain solves, the ledger, decentralization, the landscape — and what it is *not* · *12 examples planned*
- [02 — Environment Setup & Tooling](02-environment-setup.md) — Go toolchain, the module layout for this repo, node clients, testnets, faucets, and the CLI toolbox · *12 examples planned*
- [03 — Bytes, Hex, Big Integers & Encoding](03-bytes-encoding.md) — the data primitives — `[]byte`, endianness, hex, base58, bech32, `big.Int`, fixed-size arrays · *18 examples planned*

### Part 2 — Cryptography Foundations

The four primitives every chain is built from: hashes, Merkle trees, signatures and key derivation. You implement each one in Go before you ever see a block.

- [04 — Cryptographic Hash Functions](04-hash-functions.md) — SHA-256, Keccak-256, the five properties, commitments, and hashing structured data deterministically · *20 examples planned*
- [05 — Merkle Trees & Proofs](05-merkle-trees.md) — building a tree, the root as a commitment, inclusion proofs, and the traps (odd leaves, second preimage) · *18 examples planned*
- [06 — Keys & Digital Signatures (ECDSA on secp256k1)](06-keys-signatures.md) — private/public keys, the curve, signing, verifying, public-key recovery, malleability and nonce disasters · *18 examples planned*
- [07 — Addresses, Encodings & HD Wallets](07-addresses-wallets-hd.md) — Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation · *18 examples planned*

### Part 3 — Build a Blockchain from Scratch (Go)

The heart of the course. One Go program grows across eight lessons into a working chain: blocks, mining, UTXO transactions, a wallet, persistence, a P2P network, fork choice — then the account model.

- [08 — Blocks & the Chain](08-blocks-and-chain.md) — the block struct, hash linking, genesis, deterministic serialization and chain validation · *18 examples planned*
- [09 — Proof of Work & Mining](09-proof-of-work.md) — the difficulty target, nonce grinding, compact `bits`, retargeting, and the honest cost discussion · *18 examples planned*
- [10 — Transactions & the UTXO Model](10-transactions-utxo.md) — inputs, outputs, the coinbase, signing over a trimmed copy, and maintaining the UTXO set · *18 examples planned*
- [11 — Wallets, Fees & the Mempool](11-wallets-mempool.md) — a keystore, transaction construction, the fee market, mempool policy and block assembly · *18 examples planned*
- [12 — Persistence & Chain State](12-persistence-chainstate.md) — storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety · *18 examples planned*
- [13 — P2P Networking & Gossip](13-p2p-networking.md) — peer discovery, a handshake, inventory gossip, block/tx propagation and a minimal node in Go · *18 examples planned*
- [14 — Consensus, Forks & Reorgs](14-consensus-forks.md) — fork choice by accumulated work, reorg handling, finality as probability, and the attacks · *18 examples planned*
- [15 — The Account Model & World State](15-account-model-state.md) — accounts, nonces, balances, code and storage — the state transition and how it differs from UTXO · *18 examples planned*

### Part 4 — Ethereum & the EVM

From your toy chain to the real one. Ethereum's architecture, RLP and the Patricia trie, the EVM as a stack machine you build yourself, the transaction types, and talking to a node from Go.

- [16 — Ethereum Architecture](16-ethereum-architecture.md) — the whole machine: accounts, blocks, receipts, gas, the execution/consensus split, blobs and upgrades · *16 examples planned*
- [17 — RLP & the Merkle Patricia Trie](17-rlp-merkle-patricia-trie.md) — Ethereum's serialization format and the trie that produces every root in the header · *18 examples planned*
- [18 — The EVM: a Stack Machine You Can Build](18-evm.md) — opcodes, stack, memory, storage, calldata, gas accounting — and a mini-EVM written in Go · *22 examples planned*
- [19 — Ethereum Transactions Deep Dive](19-transaction-types.md) — legacy, 2930, 1559 and 4844 types — signing, RLP, chain id, hashing and sender recovery · *18 examples planned*
- [20 — JSON-RPC & go-ethereum's `ethclient`](20-json-rpc-ethclient.md) — the node API, `ethclient` from Go, queries, filters, subscriptions, batching and provider realities · *18 examples planned*
- [21 — Sending Transactions from Go](21-sending-transactions.md) — nonce management, gas estimation, 1559 fees, signing, broadcast, confirmation and stuck-tx recovery · *18 examples planned*

### Part 5 — Smart Contracts from Go

Enough Solidity to read a contract, then everything on the Go side: ABI encoding, abigen bindings, events and logs, the ERC standards, and the bug classes that drain contracts.

- [22 — Solidity Basics for Go Developers](22-solidity-basics.md) — enough Solidity to read, compile and reason about a contract — mapped onto EVM mechanics you know · *16 examples planned*
- [23 — The Contract ABI: Encoding & Decoding](23-abi-encoding.md) — function selectors, head/tail layout, dynamic types, revert data and EIP-712 — by hand and with `abi` · *20 examples planned*
- [24 — Type-Safe Contracts with `abigen`](24-abigen-bindings.md) — generating Go bindings, deploying, calling, transacting, and the simulated backend · *16 examples planned*
- [25 — Events, Logs & Indexing Them](25-events-logs.md) — topics, the bloom filter, `eth_getLogs` at scale, decoding, subscriptions and reorg-safe consumption · *19 examples planned*
- [26 — The ERC Standards from Go](26-erc-standards.md) — ERC-20, ERC-721, ERC-1155, ERC-165 and Multicall — including every non-standard token that breaks your code · *19 examples planned*
- [27 — Smart Contract Security](27-contract-security.md) — the bug classes that drain contracts, the real incidents, and how a Go integrator defends itself · *19 examples planned*

### Part 6 — Consensus & Scaling

How agreement is reached at scale: Ethereum's proof of stake, the BFT family, and the layer-2 designs everything is migrating to.

- [28 — Proof of Stake & Ethereum Consensus](28-proof-of-stake.md) — validators, slots and epochs, LMD-GHOST + Casper FFG, attestations, finality, slashing and PBS · *15 examples planned*
- [29 — Alternative Consensus: BFT, PoA & the Rest](29-alternative-consensus.md) — PBFT, Tendermint, HotStuff, Clique, DPoS — what each assumes, what each buys, and one implemented in Go · *13 examples planned*
- [30 — Layer 2 & Scaling](30-layer2-scaling.md) — rollups, data availability, blobs, channels, and what L2s mean for your Go code · *15 examples planned*

### Part 7 — Production Blockchain Engineering in Go

The job. Indexers that survive reorgs, key management that survives an audit, node operations, deterministic tests, and the observability that tells you when RPC is lying to you.

- [31 — Building a Blockchain Indexer in Go](31-blockchain-indexer.md) — ingesting blocks and logs, reorg-safe writes, backfill, idempotency and a schema that answers queries · *19 examples planned*
- [32 — Key Management & Signing Services](32-key-management-signing.md) — keystores, KMS/HSM, hot/warm/cold tiers, a policy-enforcing signing service, and nonces at scale · *17 examples planned*
- [33 — Node Operations & RPC Infrastructure](33-node-operations.md) — running geth, sync modes, storage, providers, failover and what actually breaks in production · *15 examples planned*
- [34 — Testing Blockchain Code in Go](34-testing-blockchain-go.md) — simulated backends, forked chains, fakes for RPC, deterministic fixtures and reorg tests · *17 examples planned*
- [35 — Observability & Reliability for Chain Services](35-observability-reliability.md) — metrics, tracing, alerting on lag and reorgs, retries, circuit breakers and reconciliation · *17 examples planned*

### Part 8 — Beyond Ethereum

The rest of the landscape through a Go lens: Bitcoin's Script and Taproot, Cosmos app-chains, Solana's account model, zero-knowledge proofs with gnark, and DeFi/MEV.

- [36 — Bitcoin Deep Dive with Go](36-bitcoin-deep-dive.md) — Script, P2PKH→Taproot, sighash types, weight units, PSBT and the `btcsuite` libraries · *17 examples planned*
- [37 — Cosmos SDK & CometBFT: App-Chains in Go](37-cosmos-tendermint.md) — building your own chain in Go — ABCI, modules, keepers, staking and IBC · *15 examples planned*
- [38 — Solana & Other Execution Models](38-solana-other-vms.md) — the accounts model, parallel execution, and talking to non-EVM chains from Go · *14 examples planned*
- [39 — Zero-Knowledge Proofs with `gnark`](39-zero-knowledge-proofs.md) — commitments, circuits, SNARKs vs STARKs, and writing and proving a circuit in Go · *13 examples planned*
- [40 — DeFi Primitives & MEV from a Go Integrator's View](40-defi-primitives-mev.md) — AMMs, lending, liquidations, MEV and flash loans — the math, and the integration code that survives it · *17 examples planned*

### Part 9 — Capstone

One end-to-end system that uses the whole spine.

- [41 — Capstone Project](41-capstone.md) — one end-to-end system: indexer + API + signing service + contract integration, in Go · *project deliverable*

### Part 10 — Cryptography, Deeper

*Extension — beyond the 01–41 spine.* The signature schemes and key-protection machinery the core plan only previewed. Take any time after [06](06-keys-signatures.md); [43](43-multisig-mpc-threshold.md) pairs with [32](32-key-management-signing.md).

- [42 — Schnorr, BLS & Aggregate Signatures](42-schnorr-bls-aggregate.md) — the signature schemes that replaced ECDSA — linearity, aggregation, and where each is used · *15 examples planned*
- [43 — Multisig, MPC & Threshold Signing](43-multisig-mpc-threshold.md) — removing the single point of failure — on-chain multisig, Shamir sharing, and threshold signatures · *14 examples planned*
- [44 — Symmetric Crypto, KDFs & Encryption at Rest](44-symmetric-crypto-at-rest.md) — AES-GCM, ChaCha20, argon2/scrypt, envelope encryption, and the keystore format decoded · *16 examples planned*

### Part 11 — Smart Contracts, Deeper

*Extension — beyond the 01–41 spine.* Everything past 'can call a contract': writing efficient Solidity, upgrading it safely, account abstraction, a real test suite, and the token standards beyond ERC-20/721. Take after Part 5.

- [45 — Solidity in Depth & Gas Optimization](45-solidity-depth-gas.md) — the language past the basics, storage layout as a cost model, assembly, and measured optimization · *15 examples planned*
- [46 — Upgradeable Contracts & Proxy Patterns](46-upgradeable-contracts.md) — DELEGATECALL proxies, storage collisions, UUPS vs Transparent vs Beacon, and monitoring upgrades from Go · *16 examples planned*
- [47 — Account Abstraction: ERC-4337 & EIP-7702](47-account-abstraction.md) — smart accounts, UserOperations, bundlers, paymasters, and what changes for your Go backend · *16 examples planned*
- [48 — Foundry: Tests, Fuzzing & Invariants](48-foundry-testing.md) — the contract-side test suite that complements your Go tests — cheatcodes, fuzzing, invariants, forking · *15 examples planned*
- [49 — Vaults, Permits & the Wider Token Standards](49-token-standards-wider.md) — ERC-4626, EIP-2612, Permit2, ERC-2981, ERC-6551 and friends — the standards that show up in real integrations · *15 examples planned*

### Part 12 — Identity, Wallets & dApp Backends

*Extension — beyond the 01–41 spine.* The half of web3 that lives off-chain: signature-based login, wallet sessions, name resolution and oracles — the pieces a Go backend actually has to implement. Take after [23](23-abi-encoding.md).

- [50 — Off-Chain Signatures: EIP-191, EIP-712, EIP-1271 & SIWE](50-offchain-signatures-siwe.md) — signature-based login and authorization — the backend half of every dApp · *16 examples planned*
- [51 — Wallet Integration & dApp Backends](51-wallet-integration-backends.md) — the wallet RPC surface, WalletConnect, webhooks, and the Go service behind a dApp · *15 examples planned*
- [52 — ENS & Name Resolution](52-ens-name-resolution.md) — forward and reverse resolution, namehash, wildcard/CCIP-Read, and doing it correctly from Go · *14 examples planned*
- [53 — Oracles, Price Feeds & On-Chain Randomness](53-oracles-randomness.md) — getting off-chain truth on-chain safely — Chainlink, TWAPs, VRF, RANDAO, and running your own · *15 examples planned*

### Part 13 — Chain Data at Scale

*Extension — beyond the 01–41 spine.* Chain data as a data-engineering problem: the mempool, decentralized storage, ETL into warehouses, and making Go ingestion fast. Deepens [31](31-blockchain-indexer.md).

- [54 — The Mempool from Outside](54-mempool-from-outside.md) — watching pending transactions, gas oracles, simulation, and the realities of the public mempool · *15 examples planned*
- [55 — Decentralized Storage: IPFS, Arweave & NFT Metadata](55-decentralized-storage.md) — content addressing, CIDs, pinning, gateways, and resolving metadata reliably from Go · *15 examples planned*
- [56 — Analytics & ETL: Subgraphs, Warehouses & Query APIs](56-analytics-etl.md) — getting chain data into systems that answer questions — The Graph, warehouses, and the build-vs-buy call · *15 examples planned*
- [57 — High-Throughput Ingestion & Performance in Go](57-high-throughput-ingestion.md) — making the indexer fast — profiling, batching, bounded concurrency, allocation control and database throughput · *15 examples planned*

### Part 14 — Custody, Payments & Compliance

*Extension — beyond the 01–41 spine.* What an exchange, PSP or wallet company actually builds: deposit detection, sweeps, withdrawals, double-entry accounting and the regulatory surface. Deepens [32](32-key-management-signing.md).

- [58 — Deposit Detection & Address Management](58-deposit-detection.md) — per-user addresses, detecting incoming value of every kind, confirmations and crediting exactly once · *15 examples planned*
- [59 — Withdrawals, Sweeping & Fee Management](59-withdrawals-sweeping.md) — moving value out safely — sweeps, batching, gas funding, approvals and the withdrawal state machine · *15 examples planned*
- [60 — Ledger & Accounting for Crypto](60-ledger-accounting.md) — double-entry bookkeeping for on-chain value — the design that makes your numbers provable · *15 examples planned*
- [61 — Compliance: AML, Sanctions & the Travel Rule](61-compliance-aml.md) — the regulatory surface a custodial service must implement, and the Go code that implements it · *13 examples planned*

### Part 15 — Protocol Internals & go-ethereum

*Extension — beyond the 01–41 spine.* Down into the client itself — the largest, most idiomatic Go codebase in the ecosystem — plus private networks, light clients, forks and the road to statelessness.

- [62 — Reading the go-ethereum Codebase](62-geth-codebase.md) — navigating the largest idiomatic Go codebase in the ecosystem, and contributing to it · *13 examples planned*
- [63 — Running a Private Network & Custom Chains](63-private-networks.md) — genesis files, dev chains, PoA networks, forked mainnets and spinning up your own L2 · *13 examples planned*
- [64 — Light Clients, SPV & Trustless Reads](64-light-clients-spv.md) — verifying without trusting — Merkle proofs, header chains, sync committees and proof verification in Go · *13 examples planned*
- [65 — Precompiles, Hard Forks & Protocol Upgrades](65-precompiles-forks.md) — the EVM's built-in functions, how forks are specified and shipped, and how to track them · *13 examples planned*
- [66 — State Growth, Verkle & Stateless Ethereum](66-state-growth-verkle.md) — why state size is the long-term problem, and the vector commitments meant to fix it · *11 examples planned*

### Part 16 — Cross-Chain & Production Operations

*Extension — beyond the 01–41 spine.* Moving value between chains, and running all of it in production without losing funds.

- [67 — Bridges & Cross-Chain Messaging](67-bridges-cross-chain.md) — moving value and messages between chains — the designs, the proofs, and why bridges keep getting drained · *13 examples planned*
- [68 — Deploying & Operating Chain Services in Production](68-deploying-operating.md) — containers, config, secrets, migrations, rollouts and the incident response that chain services need · *13 examples planned*

## Progress

See [PROGRESS.md](PROGRESS.md) for the current step and notes from past lessons.
