# Part 15 — Protocol Internals & go-ethereum

Down into the client itself — the largest, most idiomatic Go codebase in the ecosystem — plus private networks, light clients, forks and the road to statelessness.

**Extension.** Beyond the core 01–41 spine. Lessons 62–66 · 63 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 62 | [Reading the go-ethereum Codebase](#62-reading-the-go-ethereum-codebase) | 18, 17 | 13 |
| 63 | [Running a Private Network & Custom Chains](#63-running-a-private-network-custom-chains) | 33, 62 | 13 |
| 64 | [Light Clients, SPV & Trustless Reads](#64-light-clients-spv-trustless-reads) | 17, 28 | 13 |
| 65 | [Precompiles, Hard Forks & Protocol Upgrades](#65-precompiles-hard-forks-protocol-upgrades) | 18, 62 | 13 |
| 66 | [State Growth, Verkle & Stateless Ethereum](#66-state-growth-verkle-stateless-ethereum) | 17, 64 | 11 |

---

## 62 — Reading the go-ethereum Codebase

**Lesson file:** [../62-geth-codebase.md](../62-geth-codebase.md) · **Examples folder:** `../examples/62-geth-codebase/`

| | |
|---|---|
| Prerequisites | [18](../18-evm.md), [17](../17-rlp-merkle-patricia-trie.md) |
| Unlocks | 63, 65 |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 9 |

*navigating the largest idiomatic Go codebase in the ecosystem, and contributing to it*

### Goals

- Navigate go-ethereum's package structure with confidence.
- Trace a transaction from JSON-RPC through the EVM to state commit, in the real code.
- Build geth from source and run a patched binary.
- Use geth as a library rather than only as a node.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why read it**

   - It is the reference implementation — when the docs are ambiguous, the code is the answer.
   - It is ~500k lines of production Go written by strong engineers; the patterns are worth stealing.
   - You already use it as a library; understanding it makes you far more effective.
   - This is the single highest-leverage lesson for a Go developer in this course.

**2. The package map**

   - `core/` — blockchain, state, EVM, transaction pool. The heart.
   - `core/types` — Block, Transaction, Receipt, Log; you already use these daily.
   - `core/vm` — the EVM interpreter you built a version of in lesson 18.
   - `core/state` — StateDB, journal, snapshots (lesson 15's ideas, industrial strength).
   - `trie/` — the MPT from lesson 17. `rlp/` — lesson 17's encoder.
   - `eth/` — the protocol: handlers, sync, filters. `p2p/` — devp2p, discovery, RLPx (lesson 13).
   - `accounts/` — abi, bind, keystore (lessons 23, 24, 32). `internal/ethapi` — the RPC handlers.
   - `consensus/` — clique and beacon. `params/` — chain configs and fork blocks.

**3. Following a transaction**

   - `internal/ethapi/api.go` `SendRawTransaction` → decode → `txpool.Add`.
   - `core/txpool` validation and queueing (lesson 11's mempool, for real).
   - Block production/import → `core/blockchain.go` `insertChain` → `processBlock`.
   - `core/state_processor.go` → `ApplyTransaction` → `ApplyMessage` → `core/vm.EVMInterpreter.Run`.
   - State commit → trie update → `stateRoot`. Trace this path once with a debugger and it all clicks.

**4. Reading `core/vm`**

   - `interpreter.go`'s main loop next to yours from lesson 18 — same shape, more rigour.
   - `jump_table.go`: the opcode table with gas functions, stack requirements and memory sizing.
   - `instructions.go`: every opcode's implementation, readable one at a time.
   - `gas_table.go` and `operations` — where EIP-2929 and friends actually live.

**5. Reading `core/state`**

   - `StateDB` as the mutable overlay; `stateObject` per account.
   - `journal.go`: exactly the undo-log pattern from lesson 15.
   - `Snapshot()`/`RevertToSnapshot()` and how nested calls use them.
   - The snapshot layer (`core/state/snapshot`) as a flat-state accelerator over the trie.

**6. Go patterns worth stealing**

   - `event.Feed` and `event.Subscription` — a clean typed pub/sub.
   - Interface design: small consumer-defined interfaces everywhere.
   - `rlp` struct tags and code generation (`rlpgen`).
   - Error wrapping conventions and the sentinel errors you already catch.
   - Their metrics, logging and lifecycle (`node.Service`) patterns.

**7. Building and patching**

   - `make geth`, build tags, and the version stamping via ldflags.
   - Adding a log line inside `ApplyTransaction` and running your patched binary on `--dev`.
   - Running the test suite; `go test ./core/...` and what it covers.
   - The hive test suite for cross-client conformance, in one paragraph.

**8. Using geth as a library**

   - You already do: `crypto`, `rlp`, `abi`, `core/types`, `ethclient`.
   - Less-known but useful: `core/vm` for local execution, `trie` for proof verification, `p2p/enode` for ENRs.
   - Embedding a full node in your Go process with `node.New` + `eth.New` — when it is worth it.
   - Version pinning pain: the API changes; pin and read the release notes.

**9. Contributing**

   - The contribution guide, commit conventions and review culture.
   - Good first issues: documentation, tests, small bug fixes.
   - Reading a real PR end to end to see the standard expected.
   - Other Go clients worth reading: erigon (staged sync), prysm (consensus layer).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reading top-down and drowning — follow one transaction's path instead.
- Assuming an API is stable; go-ethereum moves and breaks callers between minor versions.
- Copying internal packages into your project instead of pinning the module.
- Trying to understand sync before understanding state and the EVM.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Clone and build geth from source; print its version.
- Find where `eth_sendRawTransaction` is handled and read the function.
- Locate the opcode implementation for `SSTORE`.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Trace `ApplyTransaction` → `ApplyMessage` → `Run` and write the call chain out.
- Add a log line inside the EVM interpreter loop and run the patched binary on `--dev`.
- Read `core/state/journal.go` and map each entry type to lesson 15's undo journal.
- Use `core/vm` directly from your own Go program to execute a snippet of bytecode.
- Verify a Merkle proof with the `trie` package directly.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Run a geth node embedded inside your own Go process with `node.New` and query it in-process.
- Diff geth's `core/vm` interpreter against your lesson 18 implementation and list every divergence.
- Write and submit a genuine documentation or test improvement to go-ethereum.

### Packages & tools

`github.com/ethereum/go-ethereum/core`, `github.com/ethereum/go-ethereum/core/vm`, `github.com/ethereum/go-ethereum/core/state`, `github.com/ethereum/go-ethereum/trie`, `github.com/ethereum/go-ethereum/node`, `github.com/ethereum/go-ethereum/eth`

### Resources to cite

- go-ethereum repository: https://github.com/ethereum/go-ethereum
- geth developer docs: https://geth.ethereum.org/docs/developers
- Contribution guide: https://geth.ethereum.org/docs/developers/geth-developer/contributing
- Erigon: https://github.com/erigontech/erigon

---

## 63 — Running a Private Network & Custom Chains

**Lesson file:** [../63-private-networks.md](../63-private-networks.md) · **Examples folder:** `../examples/63-private-networks/`

| | |
|---|---|
| Prerequisites | [33](../33-node-operations.md), [62](../62-geth-codebase.md) |
| Unlocks | — |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 9 |

*genesis files, dev chains, PoA networks, forked mainnets and spinning up your own L2*

### Goals

- Create a genesis file and launch a private network.
- Run a multi-node Clique PoA network with Docker Compose.
- Fork mainnet locally with state and use it as a realistic test environment.
- Understand what launching an OP-Stack chain involves.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why a private network**

   - Deterministic integration tests with real client behaviour, not a simulator.
   - Consortium and enterprise deployments.
   - Experimenting with forks, precompiles and parameters safely.
   - Reproducing a mainnet bug in an environment you control.

**2. The genesis file**

   - `config` (chainId, fork blocks/timestamps, consensus engine), `alloc`, `gasLimit`, `difficulty`, `extraData`.
   - Every fork activation is a field; getting one wrong gives you a subtly different EVM.
   - Prefunding accounts and predeploying contracts via `alloc` (including code and storage).
   - `geth init` and why every node must use the byte-identical genesis (lesson 08).

**3. Dev mode**

   - `geth --dev` — instant mining, a prefunded account, zero setup.
   - `--dev.period` for realistic block times.
   - Its limits: single node, no real consensus, no p2p.
   - `anvil` as the faster alternative, with cheatcodes (lesson 34).

**4. Clique PoA networks**

   - Signers encoded in the genesis `extraData`; block sealing round-robin with in-turn/out-of-turn delays.
   - Bootstrapping: generate keys, build genesis, `geth init`, start with `--unlock` and `--mine`.
   - Peering: static nodes, bootnodes, `admin_addPeer`, and the enode URL format.
   - Adding and removing signers by vote — and watching the extraData change.

**5. Multi-node with Docker Compose**

   - Three signers plus one RPC node; a shared genesis volume; a bootnode.
   - Networking: container DNS, exposed ports, and the bind-to-0.0.0.0 rule.
   - Health checks and start ordering.
   - Driving the whole stack from Go integration tests (lesson 34).

**6. Forking mainnet with state**

   - `anvil --fork-url --fork-block-number` for a stateful local mainnet.
   - Cheatcodes: impersonate accounts, set balances and storage, mine on demand, time travel.
   - Persistence and snapshots (`evm_snapshot`/`evm_revert`) for fast test resets.
   - Limits: only fetched state is available, and the fork provider must be an archive node.

**7. Custom chain parameters**

   - Changing block gas limit, block time, base-fee parameters.
   - Custom precompiles via a patched geth (lesson 62) — when and why.
   - Chain id selection and why you must not collide with a real chain.
   - Fork scheduling: activating a fork at a future block and testing the transition.

**8. Launching an L2**

   - OP Stack: op-geth (execution), op-node (derivation), op-batcher, op-proposer.
   - Deploying the L1 contracts, generating the rollup config, and starting the stack.
   - Rollup-as-a-service providers and what they actually run for you.
   - The honest cost: sequencer operation, DA fees, bridge security, and ongoing upgrades.

**9. Operating a private network**

   - Backup and restore of chain data; resetting a broken network.
   - Monitoring the same way as a public node (lesson 33).
   - Faucets and account management for developers.
   - The trap: private networks drift from mainnet behaviour, so always test against a fork too.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- A genesis mismatch between nodes — they will never peer, with an unhelpful error.
- Reusing a real chain id and enabling cross-chain replay.
- Wrong fork activation timestamps, giving you an EVM that differs from mainnet in subtle ways.
- Binding a node to 127.0.0.1 inside a container and wondering why peering fails.
- Treating private-network test results as proof of mainnet behaviour.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Write a genesis file with two prefunded accounts and `geth init` it.
- Run `geth --dev` and send a transaction from Go.
- Print a node's enode URL and chain configuration.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Launch a single-signer Clique network and mine blocks.
- Predeploy a contract via genesis `alloc` with code and storage.
- Peer two nodes via a bootnode and confirm block propagation.
- Fork mainnet at a pinned block with anvil and read a real USDC balance.
- Use `evm_snapshot`/`evm_revert` to reset state between Go tests.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A three-signer Clique network in Docker Compose, driven by a Go integration test suite.
- Vote a new signer into the Clique set from Go and observe extraData change.
- Schedule a fork activation at a future block and test behaviour on both sides of the transition.

### Packages & tools

`os/exec`, `github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/rpc`, `encoding/json`, `testing`

### Resources to cite

- geth private networks: https://geth.ethereum.org/docs/fundamentals/private-network
- EIP-225 (Clique): https://eips.ethereum.org/EIPS/eip-225
- Anvil: https://book.getfoundry.sh/anvil/
- OP Stack — run a chain: https://docs.optimism.io/builders/chain-operators/self-hosted

---

## 64 — Light Clients, SPV & Trustless Reads

**Lesson file:** [../64-light-clients-spv.md](../64-light-clients-spv.md) · **Examples folder:** `../examples/64-light-clients-spv/`

| | |
|---|---|
| Prerequisites | [17](../17-rlp-merkle-patricia-trie.md), [28](../28-proof-of-stake.md) |
| Unlocks | 66, 67 |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 8 |

*verifying without trusting — Merkle proofs, header chains, sync committees and proof verification in Go*

### Goals

- Explain what a light client verifies and what it must still trust.
- Verify an account or storage proof against a state root in Go.
- Implement Bitcoin SPV verification of a transaction's inclusion.
- Describe Ethereum's sync-committee light client protocol.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The trust problem**

   - When you call an RPC provider, you are trusting it completely — it can lie about any read.
   - Most applications accept this. Some cannot: bridges, high-value automation, censorship-resistant apps.
   - A light client replaces trust with verification, at the cost of complexity and a header chain.
   - The spectrum: full node → light client → RPC with proofs → blind RPC.

**2. Bitcoin SPV**

   - Download only headers (80 bytes each) and verify the PoW chain (lessons 08, 09).
   - Verify a transaction's inclusion with a Merkle proof against the header's merkleRoot (lesson 05).
   - What SPV does *not* verify: that the transaction is valid, or that no double-spend exists elsewhere.
   - Bloom filters (BIP-37) and their privacy failure; compact block filters (BIP-157/158) as the fix.
   - Implementing header-chain validation and an SPV proof check in Go.

**3. Ethereum state proofs**

   - `eth_getProof` returns an account proof plus per-slot storage proofs (EIP-1186).
   - Verification: walk the MPT nodes, checking each hash and following the nibble path (lesson 17).
   - You still need a trusted `stateRoot` — that is what the header chain gives you.
   - This makes a *read* trustless: the provider cannot lie about a balance or a storage slot.
   - Implementing full proof verification in Go with `trie.VerifyProof`, and by hand.

**4. Receipt and transaction proofs**

   - Proving a transaction was included: a Merkle proof against transactionsRoot.
   - Proving an event fired: a proof against receiptsRoot plus the log's position.
   - This is exactly how cross-chain messaging works (lesson 67).
   - Building both proofs in Go from a block's data.

**5. Ethereum's light client protocol**

   - The sync committee: 512 validators, rotating every ~27 hours, signing headers with BLS.
   - A light client follows sync-committee updates and can verify any header with one BLS aggregate check.
   - Weak subjectivity: you need a recent trusted checkpoint to start; you cannot bootstrap from genesis safely.
   - The beacon API light-client endpoints, and Helios as a working implementation.
   - Fetching a light-client update and verifying it in Go.

**6. Where trustless reads matter**

   - Bridges — a lie here is a mint on the destination chain (lesson 67).
   - Automated high-value operations: liquidations, treasury moves.
   - Censorship resistance: detecting that a provider is hiding data from you.
   - Verifiable RPC middleware: proxy a provider but verify everything it returns.

**7. Practical middle grounds**

   - Cross-check two independent providers and alert on divergence (lesson 33).
   - Verify proofs only for high-value reads, trust the rest.
   - Pin a recent finalized block hash from a second source and verify chain continuity.
   - Being explicit in your design docs about which reads are trusted and which are verified.

**8. Costs**

   - Proof size and verification CPU; the header chain's storage.
   - Latency: an extra round trip for the proof.
   - Provider support: not all expose `eth_getProof`.
   - Verkle tries will shrink proofs dramatically (lesson 66).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Verifying a proof against a stateRoot you got from the same untrusted provider — that proves nothing.
- Bootstrapping a light client from genesis under proof of stake (long-range attacks).
- Assuming SPV inclusion means the transaction was valid.
- Ignoring the <32-byte inline node rule and failing valid proofs (lesson 17).
- Using BIP-37 bloom filters and leaking your entire wallet to the serving node.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Fetch `eth_getProof` for an account and print the proof node count.
- Verify a Bitcoin Merkle inclusion proof against a header's merkleRoot.
- Validate a chain of 100 Bitcoin headers for PoW and linkage.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Verify an Ethereum account proof against a block's stateRoot with `trie.VerifyProof`.
- Verify a storage-slot proof by hand, walking the nodes yourself.
- Build a transaction-inclusion proof from a block and verify it against transactionsRoot.
- Build a receipt proof and verify a log's presence against receiptsRoot.
- Cross-check a balance across two providers and detect an injected lie.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A verifying RPC middleware in Go that proves every `eth_getBalance` and `eth_getStorageAt` result.
- Fetch and verify a sync-committee light-client update with a BLS aggregate check.
- An SPV wallet in Go: header chain, compact block filters, and verified transaction inclusion.

### Packages & tools

`github.com/ethereum/go-ethereum/trie`, `github.com/ethereum/go-ethereum/rlp`, `github.com/ethereum/go-ethereum/ethclient`, `github.com/btcsuite/btcd`, `github.com/consensys/gnark-crypto/ecc/bls12-381`

### Resources to cite

- EIP-1186 (eth_getProof): https://eips.ethereum.org/EIPS/eip-1186
- Ethereum light client specs: https://github.com/ethereum/consensus-specs/tree/dev/specs/altair/light-client
- Helios light client: https://github.com/a16z/helios
- BIP-157/158 (compact block filters): https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki
- Bitcoin SPV (whitepaper §8): https://bitcoin.org/bitcoin.pdf

---

## 65 — Precompiles, Hard Forks & Protocol Upgrades

**Lesson file:** [../65-precompiles-forks.md](../65-precompiles-forks.md) · **Examples folder:** `../examples/65-precompiles-forks/`

| | |
|---|---|
| Prerequisites | [18](../18-evm.md), [62](../62-geth-codebase.md) |
| Unlocks | — |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 8 |

*the EVM's built-in functions, how forks are specified and shipped, and how to track them*

### Goals

- List the precompiles, what each does, and their gas costs.
- Call precompiles from Solidity and from Go.
- Explain how a hard fork is specified, scheduled and activated.
- Track upcoming changes and assess their impact on your code.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What a precompile is**

   - A native implementation at a fixed address (0x01…0x11+) that the EVM special-cases.
   - Why: some cryptography is impossibly expensive as EVM bytecode.
   - They behave like contracts (CALL them, they return data) but have no code.
   - `eth_getCode` on a precompile returns empty — a real source of confusion in tooling.

**2. The precompile list**

   - 0x01 ecrecover — the one you use constantly (lessons 06, 50).
   - 0x02 sha256, 0x03 ripemd160, 0x04 identity (datacopy).
   - 0x05 modexp (EIP-198) — RSA and general modular exponentiation.
   - 0x06/0x07/0x08 bn256 add/mul/pairing (EIP-196/197) — what makes on-chain SNARK verification possible (lesson 39).
   - 0x09 blake2f (EIP-152).
   - 0x0a point evaluation (EIP-4844) — KZG proof verification for blobs (lesson 30).
   - 0x0b–0x11 BLS12-381 operations (EIP-2537) — cheaper BLS verification (lesson 42).
   - Gas costs per precompile and why modexp's is dynamic.

**3. Calling them**

   - From Solidity: `staticcall` to the address with correctly encoded input; check the success flag.
   - The failure mode: a failed precompile call returns false and empty data, which naive code reads as zero.
   - `ecrecover` returning address(0) — the classic bug (lesson 27).
   - From Go: `core/vm.PrecompiledContractsCancun` lets you run them locally, off-chain.

**4. How a fork is specified**

   - An EIP: motivation, specification, rationale, backwards compatibility, test cases, security considerations.
   - EIP statuses: Draft → Review → Last Call → Final; and Meta EIPs that define a fork's contents.
   - All Core Devs calls, and how EIPs are actually selected for inclusion.
   - Execution-spec tests as the cross-client conformance suite.

**5. How a fork ships**

   - A block number (pre-Merge) or a timestamp (post-Merge) in every client's chain config.
   - `params/config.go` in geth — read it and see every fork ever.
   - Testnet activation first (Sepolia, Hoodi), then mainnet, typically weeks apart.
   - Client releases must all land before activation; a straggler splits the network.

**6. Notable forks and what they changed**

   - Homestead, DAO fork, Tangerine/Spurious (EIP-150/155/161 — gas repricing after the 2016 DoS).
   - Byzantium (REVERT, precompiles), Constantinople/Istanbul (repricing, CHAINID, EXTCODEHASH).
   - London (EIP-1559), The Merge (PoS), Shanghai (withdrawals, PUSH0), Cancun (blobs, TSTORE, MCOPY).
   - Prague/Pectra (EIP-7702, BLS precompiles) and what is queued after.
   - Each one changed something your code might depend on.

**7. Impact assessment**

   - Gas repricings break hardcoded gas limits and `transfer`-based patterns.
   - New transaction types break naive decoders (lesson 19).
   - New opcodes break contracts deployed to chains that have not upgraded (the PUSH0/L2 problem).
   - New header fields break strict struct decoding.
   - A checklist to run against your codebase before every fork.

**8. Staying current**

   - The sources: EIP repository, ethereum/pm, client release notes, execution-specs.
   - A quarterly ritual: read the next fork's Meta EIP and assess each included EIP against your code.
   - Automating it: alert on new client releases and on chain-config changes.
   - Testing against a fork before activation using a private network (lesson 63).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming a precompile call succeeded without checking the success flag.
- Comparing `ecrecover`'s result against an uninitialized address variable.
- Deploying PUSH0 bytecode to a chain that has not activated Shanghai.
- Hardcoding gas amounts that a repricing fork invalidates.
- Strictly decoding block headers so a new field breaks your service on fork day.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Call `sha256` and `identity` precompiles from Go via `eth_call`.
- Run `ecrecover` locally with `core/vm` and compare with `crypto.Ecrecover`.
- Print geth's chain config and list every fork activation.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Call `modexp` with correctly encoded input and verify the result.
- Show a failed precompile call returning empty data and the naive zero-read bug.
- Call the bn256 pairing precompile and verify a simple pairing equation.
- Show that `eth_getCode` on a precompile returns empty.
- Compare gas cost of `ecrecover` via precompile against a pure-Solidity implementation.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Verify a Groth16 proof on-chain using the bn256 precompiles and a proof generated in lesson 39.
- A fork-impact checker in Go: scan your codebase for hardcoded gas, transaction types and opcodes at risk.
- Run a private network across a fork activation and test behaviour on both sides.

### Packages & tools

`github.com/ethereum/go-ethereum/core/vm`, `github.com/ethereum/go-ethereum/params`, `github.com/ethereum/go-ethereum/ethclient`, `math/big`

### Resources to cite

- EIP repository: https://eips.ethereum.org/
- Ethereum history / forks: https://ethereum.org/en/history/
- geth params/config.go: https://github.com/ethereum/go-ethereum/blob/master/params/config.go
- execution-specs: https://github.com/ethereum/execution-specs
- EIP-2537 (BLS precompiles): https://eips.ethereum.org/EIPS/eip-2537

---

## 66 — State Growth, Verkle & Stateless Ethereum

**Lesson file:** [../66-state-growth-verkle.md](../66-state-growth-verkle.md) · **Examples folder:** `../examples/66-state-growth-verkle/`

| | |
|---|---|
| Prerequisites | [17](../17-rlp-merkle-patricia-trie.md), [64](../64-light-clients-spv.md) |
| Unlocks | — |
| Examples | **11** — 🟢 3 easy, 🟡 5 medium, 🔴 3 hard |
| Topics | 8 |

*why state size is the long-term problem, and the vector commitments meant to fix it*

### Goals

- Explain why state growth threatens node decentralisation.
- Compare Merkle Patricia proofs with Verkle proofs quantitatively.
- Describe statelessness, witnesses and block-level proofs.
- Understand state expiry and history expiry proposals.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The problem**

   - State grows monotonically; every new account and storage slot is forever.
   - Node disk and IO requirements rise, so fewer people run nodes, so the network centralises.
   - The state is random-access, so it must be on fast storage — this is the binding constraint.
   - Quantify it: current state size, growth rate, and what that implies in five years.

**2. Why MPT proofs are too big**

   - Branch nodes have 16 children; a proof must include all siblings at every level.
   - A single account proof is several KB; a block's full witness would be megabytes.
   - That kills statelessness: you cannot ship a multi-MB witness with every block.
   - Compute it concretely for a realistic block.

**3. Vector commitments and Verkle trees**

   - A vector commitment lets you open one position without revealing or including siblings.
   - Verkle = Vector commitment + Merkle tree structure, with a much wider branching factor (256).
   - Proof size becomes roughly constant (~150 bytes per proof, aggregatable) instead of O(width × depth).
   - The cost: elliptic-curve cryptography instead of hashing, so proving is slower.
   - IPA vs KZG-based constructions, in one paragraph each.

**4. Statelessness**

   - A stateless client validates a block using only the block and its witness — no state database.
   - Weak statelessness: block producers hold state, validators do not.
   - What this unlocks: trivially cheap validators, faster sync, better light clients.
   - The dependency chain: it needs small witnesses, which needs Verkle (or a SNARK).

**5. The migration problem**

   - Converting the entire state trie from MPT to Verkle, live, without halting the chain.
   - Overlay approaches: write to the new tree, read from both, convert lazily.
   - Why this has taken years and keeps moving.
   - The competing path: prove MPT execution with a SNARK instead of changing the tree.

**6. State and history expiry**

   - EIP-4444: clients stop serving history older than a year — history moves to portal/archives.
   - State expiry proposals: dormant state becomes inactive and must be revived with a proof.
   - Address-space extension and the periodification designs.
   - The user-visible consequences and why they are contentious.

**7. What this means for you**

   - Archive data becomes harder to get from a node and easier to get from an archive service.
   - Your indexer becomes *more* important, not less — you may be the only one keeping history.
   - Proof verification code (lesson 64) will need a Verkle path eventually.
   - Do not build anything that assumes a full node can answer arbitrary historical queries forever.

**8. Doing the arithmetic in Go**

   - Compute MPT proof sizes for real accounts from `eth_getProof`.
   - Model a block witness size from the accounts and slots a block touches.
   - Compare against Verkle's published proof sizes.
   - Experiment with `go-verkle` to build a small tree and measure a proof.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming archive access will always be cheap and available.
- Designing a service that re-derives history from a node on demand.
- Confusing history expiry (EIP-4444) with state expiry — different proposals, different consequences.
- Treating Verkle as imminent; timelines here have slipped repeatedly.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 11).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Fetch `eth_getProof` for an account and measure the proof's byte size.
- Compute the current state size from a node's metrics or a public source.
- Compare proof sizes for a shallow and a deep account.

**🟡 Medium — 5 examples** *(concepts combined, and the traps)*

- Model a block's witness size from the accounts and storage slots its transactions touch.
- Compare measured MPT proof sizes against published Verkle proof sizes for the same data.
- Build a small Verkle tree with `go-verkle` and produce a proof.
- Measure proving and verification time for that Verkle proof.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Estimate total witness size for 100 real mainnet blocks and chart the distribution.
- Implement both an MPT and a Verkle proof verifier behind one Go interface.
- Write an impact assessment for your own indexer under EIP-4444 history expiry.

### Packages & tools

`github.com/ethereum/go-ethereum/trie`, `github.com/ethereum/go-verkle`, `github.com/ethereum/go-ethereum/ethclient`, `math/big`

### Resources to cite

- Verkle trees (Vitalik): https://vitalik.eth.limo/general/2021/06/18/verkle.html
- Ethereum roadmap — The Verge: https://ethereum.org/en/roadmap/verkle-trees/
- EIP-4444 (history expiry): https://eips.ethereum.org/EIPS/eip-4444
- go-verkle: https://github.com/ethereum/go-verkle
- Statelessness overview: https://ethereum.org/en/roadmap/statelessness/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
