# Part 8 — Beyond Ethereum

The rest of the landscape through a Go lens: Bitcoin's Script and Taproot, Cosmos app-chains, Solana's account model, zero-knowledge proofs with gnark, and DeFi/MEV.

**Core spine.** Lessons 36–40 · 76 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 36 | [Bitcoin Deep Dive with Go](#36-bitcoin-deep-dive-with-go) | 10, 11 | 17 |
| 37 | [Cosmos SDK & CometBFT: App-Chains in Go](#37-cosmos-sdk-cometbft-app-chains-in-go) | 29 | 15 |
| 38 | [Solana & Other Execution Models](#38-solana-other-execution-models) | 15 | 14 |
| 39 | [Zero-Knowledge Proofs with `gnark`](#39-zero-knowledge-proofs-with-gnark) | 05, 30 | 13 |
| 40 | [DeFi Primitives & MEV from a Go Integrator's View](#40-defi-primitives-mev-from-a-go-integrators-view) | 26, 27 | 17 |

---

## 36 — Bitcoin Deep Dive with Go

**Lesson file:** [../36-bitcoin-deep-dive.md](../36-bitcoin-deep-dive.md) · **Examples folder:** `../examples/36-bitcoin-deep-dive/`

| | |
|---|---|
| Prerequisites | [10](../10-transactions-utxo.md), [11](../11-wallets-mempool.md) |
| Unlocks | — |
| Examples | **17** — 🟢 4 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*Script, P2PKH→Taproot, sighash types, weight units, PSBT and the `btcsuite` libraries*

### Goals

- Read and evaluate Bitcoin Script.
- Distinguish the output types and build each in Go.
- Explain SegWit's txid/wtxid split and Taproot's key/script paths.
- Build, sign and inspect transactions with `btcd`/`btcutil`.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Bitcoin as the reference implementation**

   - Everything from Part 3, now with the real details you deliberately skipped.
   - Conservative by design: no Turing-completeness, no state beyond the UTXO set.
   - Why that constraint is a feature, and what it costs.
   - `bitcoind -regtest` as your local chain, driven from Go over RPC.

**2. Script**

   - A stack language: no loops, no recursion, bounded execution.
   - scriptSig (unlocking) + scriptPubKey (locking) evaluated together — historically concatenated.
   - Core opcodes: OP_DUP, OP_HASH160, OP_EQUALVERIFY, OP_CHECKSIG, OP_CHECKMULTISIG.
   - Evaluating P2PKH by hand with the stack printed after each opcode.

**3. Standard output types**

   - P2PK (early, wasteful), P2PKH (`1...`), P2SH (`3...`, hash of a redeem script).
   - P2WPKH and P2WSH (bech32, `bc1q...`) — witness data moved out of the txid preimage.
   - P2TR (bech32m, `bc1p...`) — Taproot.
   - Building each locking script in Go with `txscript` and deriving its address.

**4. Sighash types**

   - SIGHASH_ALL (default), NONE, SINGLE, and the ANYONECANPAY modifier.
   - What each commits to, and the coordination patterns each enables.
   - The SIGHASH_SINGLE bug (index out of range → hash of 1) — a real consensus quirk.
   - BIP-143 changed the sighash algorithm for SegWit to fix a quadratic hashing DoS.

**5. Malleability and SegWit**

   - Third parties could tweak scriptSig and change the txid without invalidating the transaction.
   - SegWit moves witness data out of the txid preimage: txid (no witness) vs wtxid (with).
   - This is what made Lightning safe to build.
   - The witness commitment in the coinbase, and the block-weight rule.

**6. Weight units and fees**

   - weight = 3×base + total; vbytes = weight/4. Witness data is discounted 4×.
   - Computing a transaction's vsize *before* signing, for accurate fee estimation.
   - Why SegWit and Taproot inputs are cheaper to spend.
   - Implementing a size estimator per input type in Go.

**7. Taproot**

   - Schnorr signatures (BIP-340): linear, so keys and signatures aggregate.
   - Key path: spend with a single signature — indistinguishable from any other Taproot spend.
   - Script path: reveal only the branch you use, from a MAST of alternatives.
   - Output key Q = P + t·G where t = taggedHash("TapTweak", P ‖ merkleRoot). Implement it.
   - Tagged hashes and why BIP-340 introduced them.

**8. Timelocks**

   - `nLockTime` (absolute, transaction level), `nSequence` (relative, input level).
   - OP_CHECKLOCKTIMEVERIFY (BIP-65) and OP_CHECKSEQUENCEVERIFY (BIP-112) at the script level.
   - Building a simple hashed timelock contract (HTLC) — the Lightning primitive.
   - Why these four mechanisms exist separately.

**9. PSBT**

   - BIP-174: a container for a partially signed transaction plus everything a signer needs.
   - Roles: creator, updater, signer, combiner, input finalizer, extractor.
   - Why hardware wallets and multisig coordination require it.
   - `btcutil/psbt` in Go: build, sign with one key, combine, finalize, extract.

**10. The Go ecosystem**

   - `btcd` (full node), `btcutil` (addresses, amounts), `txscript` (script engine), `chaincfg` (network params).
   - `btcec/v2` for secp256k1 and Schnorr.
   - `lnd` as production-grade reference Go code for anything Lightning-adjacent.
   - Driving `bitcoind -regtest` over JSON-RPC from Go for integration tests.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using mainnet `chaincfg.MainNetParams` in a regtest test and generating unspendable addresses.
- Estimating fees in bytes instead of vbytes on SegWit inputs.
- Assuming txid uniquely identifies a transaction pre-SegWit.
- Reversing (or not reversing) txid byte order at the wrong boundary.
- Signing with the wrong sighash type and producing a transaction anyone can modify.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 17).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Evaluate a P2PKH script step by step with the stack printed.
- Derive a P2PKH address from a public key with `btcutil` and check a test vector.
- Compute a transaction's weight and vsize.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build P2WPKH and P2SH-P2WPKH locking scripts and their addresses.
- Construct, sign and serialize a regtest transaction with `txscript`.
- Sign with SIGHASH_SINGLE|ANYONECANPAY and explain what a third party can still change.
- Estimate vsize per input type and compare with the signed transaction's actual size.
- Build and spend a CLTV-locked output on regtest.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Compute a Taproot output key from an internal key and a script-tree Merkle root; verify against `bitcoin-cli`.
- Spend a Taproot output via the key path with a BIP-340 Schnorr signature.
- Build a PSBT, sign it in a separate process, combine, finalize and broadcast on regtest.

### Packages & tools

`github.com/btcsuite/btcd`, `github.com/btcsuite/btcd/txscript`, `github.com/btcsuite/btcd/btcutil`, `github.com/btcsuite/btcd/chaincfg`, `github.com/btcsuite/btcd/btcec/v2`

### Resources to cite

- Bitcoin developer reference: https://developer.bitcoin.org/reference/
- BIP-141 SegWit: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- BIP-143 (SegWit sighash): https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- BIP-340 Schnorr / BIP-341 Taproot: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP-174 PSBT: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- btcd: https://github.com/btcsuite/btcd

---

## 37 — Cosmos SDK & CometBFT: App-Chains in Go

**Lesson file:** [../37-cosmos-tendermint.md](../37-cosmos-tendermint.md) · **Examples folder:** `../examples/37-cosmos-tendermint/`

| | |
|---|---|
| Prerequisites | [29](../29-alternative-consensus.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 10 |

*building your own chain in Go — ABCI, modules, keepers, staking and IBC*

### Goals

- Explain the ABCI boundary between the application and the consensus engine.
- Scaffold a Cosmos SDK chain and add a custom module.
- Describe keepers, the multistore and the block lifecycle.
- Explain IBC's light-client model at a working level.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why app-chains**

   - Sovereignty: your own fee token, your own state machine, no gas competition with strangers.
   - The trade: you must bootstrap validators and security yourself.
   - Interchain Security / replicated security as an answer to that.
   - When an app-chain beats a contract on a shared L1 — and when it does not.

**2. The stack**

   - CometBFT handles consensus, P2P and the mempool. Your app handles state.
   - ABCI is the socket between them — a small, well-defined interface.
   - This separation is why you can write a chain in pure Go with no consensus code.
   - Everything here is idiomatic Go you can read, which is unusual in this space.

**3. The ABCI interface**

   - `InitChain`, `PrepareProposal`, `ProcessProposal`, `FinalizeBlock`, `Commit`, `Query`, `CheckTx`.
   - ABCI 2.0 (0.38+) merged BeginBlock/DeliverTx/EndBlock into `FinalizeBlock` — old tutorials are stale.
   - `PrepareProposal`/`ProcessProposal` give the app control over block contents — this is where app-side MEV mitigation lives.
   - Determinism is absolute: same block, same state, on every validator, forever.

**4. The Cosmos SDK**

   - Modules as the unit of composition; each owns a store key and a set of `Msg` types.
   - Keepers: the only way to touch another module's state, with explicit permissions.
   - Protobuf-defined state and messages; `Msg` service and `Query` service.
   - The app wiring: `depinject`/`app.go`, module manager, and ordering of begin/end blockers.

**5. The multistore and IAVL**

   - A versioned, Merkle-ized key-value store per module.
   - IAVL: a balanced Merkle tree giving proofs for any key at any historical version.
   - The app hash in the block header is the root over all module stores.
   - Store keys, prefix stores, and iterating safely.

**6. Built-in modules**

   - `auth` (accounts, signatures), `bank` (balances, transfers), `staking`, `distribution`, `gov`, `slashing`, `mint`.
   - What you get for free vs what you must write.
   - Ante handlers: signature verification, fee deduction, sequence checks — the middleware chain.
   - Fee grants and authz as delegation primitives.

**7. Writing a custom module**

   - proto definitions → generated types → keeper → msg server → query server → CLI → genesis → tests.
   - Collections API for typed state access instead of raw byte keys.
   - Emitting typed events for indexers.
   - Keeper unit tests with the SDK's test fixtures — fast, no chain needed.

**8. Transactions in Cosmos**

   - A tx carries multiple `Msg`s, all atomic. Accounts have a sequence (nonce equivalent).
   - Signing modes: SIGN_MODE_DIRECT (protobuf) vs LEGACY_AMINO_JSON.
   - Gas: `gas_wanted` vs `gas_used`, and the fee = gas × gasPrice in the chain's token.
   - Building and broadcasting a tx from Go with the SDK client.

**9. IBC**

   - Light clients on each chain track the other's consensus state.
   - Connections, channels, packets, acknowledgements and timeouts.
   - Relayers are permissionless and unprivileged — they cannot forge, only relay.
   - Why this is trust-minimised in a way most bridges are not (lesson 67).

**10. Upgrades**

   - On-chain governance proposals; the `upgrade` module halting at a height.
   - Upgrade handlers and store migrations, version by version.
   - Cosmovisor for automated binary swaps.
   - Coordinated upgrades as the norm — very different from Ethereum's client-diversity model.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Any nondeterminism in state transitions (map iteration, `time.Now`, floats) — instant consensus failure.
- Following pre-0.38 tutorials that use BeginBlock/DeliverTx.
- Touching another module's store directly instead of through its keeper.
- Ignoring gas metering in a custom keeper and creating a DoS vector.
- Unbounded loops over state in a begin-blocker.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Implement a minimal ABCI application that counts transactions.
- Handle `FinalizeBlock` and `Commit` and return a deterministic app hash.
- Query state through the ABCI `Query` method.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Scaffold a chain and add a custom `Post` message type.
- Write the keeper and msg server for creating and reading posts.
- Add a query service and call it over gRPC from Go.
- Write a keeper unit test with SDK test fixtures.
- Emit a typed event and consume it from a Go indexer.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Add a begin-blocker with bounded work and gas accounting.
- Build, sign and broadcast a multi-`Msg` transaction from Go.
- Write a store migration and an upgrade handler, and run the upgrade on a local chain.

### Packages & tools

`github.com/cosmos/cosmos-sdk`, `github.com/cometbft/cometbft`, `google.golang.org/protobuf`, `google.golang.org/grpc`

### Resources to cite

- Cosmos SDK docs: https://docs.cosmos.network/
- CometBFT docs: https://docs.cometbft.com/
- ABCI 2.0 spec: https://docs.cometbft.com/v1.0/spec/abci/
- IBC protocol: https://ibc.cosmos.network/

---

## 38 — Solana & Other Execution Models

**Lesson file:** [../38-solana-other-vms.md](../38-solana-other-vms.md) · **Examples folder:** `../examples/38-solana-other-vms/`

| | |
|---|---|
| Prerequisites | [15](../15-account-model-state.md) |
| Unlocks | — |
| Examples | **14** — 🟢 4 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 10 |

*the accounts model, parallel execution, and talking to non-EVM chains from Go*

### Goals

- Explain Solana's account model and why it enables parallel execution.
- Compare SVM, Move and WASM execution with the EVM.
- Interact with a Solana cluster from Go.
- Choose sensibly when a project asks 'which chain?'

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The design space**

   - Execution environments differ on: state model, parallelism, gas metering, and language.
   - EVM: sequential, global state, 256-bit, Solidity.
   - SVM: parallel, explicit account access, 64-bit, Rust.
   - Move VM: resource types. WASM: general bytecode with per-chain host functions.

**2. Solana's account model**

   - Programs are **stateless code**; all state lives in accounts owned by programs.
   - Every instruction must declare the accounts it reads and writes, up front.
   - That declaration is what lets the scheduler run non-overlapping transactions in parallel (Sealevel).
   - The cost to developers: you must know your access set before execution, which forbids some patterns.

**3. Accounts in detail**

   - Fields: lamports, owner, data, executable, rentEpoch.
   - Rent and rent-exemption: an account must hold a minimum balance or be reclaimed.
   - Account size is fixed at creation; resizing is an explicit operation.
   - PDAs (program-derived addresses): deterministic, off-curve addresses a program can sign for.

**4. Programs and CPI**

   - Deploying a program; the BPF loader; upgradeable vs immutable programs.
   - Cross-program invocation (CPI) and signer seeds for PDAs.
   - Compute units instead of gas; per-transaction and per-instruction limits.
   - Anchor as the de facto framework, and its IDL — the ABI equivalent.

**5. Consensus and clocks**

   - Proof of History: a verifiable delay function producing a global ordering *before* consensus.
   - Tower BFT on top of PoH for actual consensus.
   - Commitment levels: processed, confirmed, finalized — the equivalent of block tags.
   - Slots, leader schedule, and why an RPC can be behind.

**6. Different primitives**

   - ed25519 signatures, not secp256k1 (lesson 06). Base58 addresses, not hex.
   - A transaction can carry multiple instructions and multiple signers.
   - Blockhash-based expiry instead of nonces — transactions expire in ~2 minutes.
   - Durable nonces for offline signing.

**7. Go clients**

   - `github.com/gagliardetto/solana-go`: RPC and WS clients, transaction building, Anchor IDL decoding.
   - Fetching accounts, decoding SPL token accounts, sending transfers.
   - Subscribing to account changes and logs.
   - The Go ecosystem is thinner than Rust/TS here — expect to read source.

**8. SPL tokens**

   - Mint account (supply, decimals, authorities) + token accounts (one per owner per mint).
   - Associated token accounts and their derivation.
   - Very different from ERC-20: the token program is shared, not per-token code.
   - Decoding the account layouts by hand in Go.

**9. Move and WASM chains**

   - Move (Aptos, Sui): resources as linear types — assets cannot be copied or dropped by construction.
   - Sui's object model and parallel execution by object ownership.
   - CosmWasm, NEAR, Polkadot: WASM bytecode, per-chain host functions, different SDKs.
   - What transfers across all of them: keys, hashes, Merkle proofs, the operational concerns.

**10. Multi-chain services in Go**

   - A chain-agnostic domain interface with per-chain adapters — the hexagonal pattern.
   - What genuinely differs: address format, finality semantics, fee model, nonce/expiry, decimals.
   - What is shared: indexing, reconciliation, key management, alerting.
   - The honest comparison table for choosing a chain.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming secp256k1 and hex addresses everywhere; Solana uses ed25519 and base58.
- Treating a `processed` commitment as final.
- Letting a transaction's blockhash expire while you queue it.
- Forgetting rent exemption and having an account reclaimed.
- Porting EVM patterns that assume dynamic state access.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 14).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Connect to a Solana devnet cluster and print the current slot.
- Read an account's lamport balance and owner.
- Derive an associated token account address.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Build, sign and send a SOL transfer with `solana-go`.
- Decode an SPL token account's data layout by hand and print mint, owner and amount.
- Compare `processed`/`confirmed`/`finalized` slot numbers over 50 polls.
- Subscribe to account changes and print each update.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Sketch and implement a `ChainClient` interface with EVM and Solana implementations, plus one shared test suite.
- Decode an Anchor IDL and call a program instruction from Go.
- Build a durable-nonce transaction and sign it offline.

### Packages & tools

`github.com/gagliardetto/solana-go`, `crypto/ed25519`, `context`

### Resources to cite

- Solana docs: https://solana.com/docs
- Solana account model: https://solana.com/docs/core/accounts
- solana-go: https://github.com/gagliardetto/solana-go
- Move language book: https://move-language.github.io/move/
- CosmWasm docs: https://docs.cosmwasm.com/

---

## 39 — Zero-Knowledge Proofs with `gnark`

**Lesson file:** [../39-zero-knowledge-proofs.md](../39-zero-knowledge-proofs.md) · **Examples folder:** `../examples/39-zero-knowledge-proofs/`

| | |
|---|---|
| Prerequisites | [05](../05-merkle-trees.md), [30](../30-layer2-scaling.md) |
| Unlocks | — |
| Examples | **13** — 🟢 4 easy, 🟡 6 medium, 🔴 3 hard |
| Topics | 10 |

*commitments, circuits, SNARKs vs STARKs, and writing and proving a circuit in Go*

### Goals

- Explain what a ZK proof proves, and what it does not.
- Describe the SNARK pipeline: circuit → constraints → setup → prove → verify.
- Write, compile and prove a circuit in Go with `gnark`.
- Judge where ZK is genuinely the right tool.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The three properties**

   - Completeness: an honest prover with a true statement always convinces the verifier.
   - Soundness: a cheating prover cannot convince the verifier of a false statement (except negligibly).
   - Zero-knowledge: the verifier learns nothing beyond the statement's truth.
   - The Ali Baba cave and graph-colouring intuitions, then straight to what is actually used.

**2. Interactive to non-interactive**

   - The interactive protocol: commit, challenge, respond.
   - Fiat–Shamir: replace the verifier's random challenge with a hash of the transcript.
   - The security caveat: hashing the *whole* transcript, or you get the Frozen Heart class of bugs.
   - 'Succinct' means the proof is small and fast to verify regardless of computation size.

**3. Arithmetic circuits and R1CS**

   - Turning a computation into constraints of the form ⟨a,z⟩·⟨b,z⟩ = ⟨c,z⟩.
   - Why every branch and loop must be flattened — circuits have no control flow.
   - Constraint count is your cost metric; comparisons and hashes are expensive.
   - Witness = the full assignment; public inputs vs private witness.

**4. The proving systems**

   - Groth16: tiny proofs (~200 bytes), fast verification, but a **per-circuit trusted setup**.
   - PLONK: universal and updatable setup, slightly bigger proofs.
   - STARKs: no trusted setup, hash-based, post-quantum, larger proofs.
   - The selection table: proof size, verify cost, setup, prover time.

**5. Trusted setup**

   - What the toxic waste is and what it would let an attacker do (forge proofs, not break privacy).
   - Multi-party ceremonies: secure if *one* participant is honest.
   - Perpetual Powers of Tau and the Ethereum KZG ceremony.
   - Why universal setups (PLONK, KZG) reduced this problem's practical weight.

**6. Commitments**

   - Pedersen commitments: hiding, binding, and additively homomorphic.
   - KZG polynomial commitments: constant-size, openable at any point — the basis of EIP-4844 blobs.
   - How commitments relate to hashes (lesson 04) and Merkle roots (lesson 05).
   - Verkle trees as vector commitments (lesson 66).

**7. gnark in Go**

   - Define a circuit as a struct with `frontend.Variable` fields and `gnark:",public"` tags.
   - Implement `Define(api frontend.API) error` using `api.Add`, `api.Mul`, `api.AssertIsEqual`.
   - `frontend.Compile` → constraint system; `groth16.Setup` → pk/vk; `Prove`; `Verify`.
   - The witness API: `frontend.NewWitness`, and separating public from full witness.
   - `gnark-crypto` for the field and curve arithmetic underneath.

**8. Circuit gadgets**

   - MiMC/Poseidon as SNARK-friendly hashes — orders of magnitude cheaper than Keccak in-circuit.
   - Merkle proof verification as a circuit (the membership gadget).
   - Range checks, comparisons and bit decomposition, and why they cost so much.
   - Reusing gnark's `std` gadgets instead of hand-rolling.

**9. On-chain verification**

   - `groth16.ExportSolidity` produces a Solidity verifier contract.
   - Verification gas cost (~200–300k for Groth16) and why proof size matters on-chain.
   - Deploying the verifier and calling it from Go with a generated proof.
   - The precompiles that make this possible (EIP-196/197, lesson 65).

**10. Applications and honest costs**

   - zk-rollups (lesson 30), private transfers (Zcash, Tornado), proof of membership, identity.
   - zkML and zkVMs: what is real today and what is a demo.
   - Costs: proving time (seconds to minutes), memory (GBs), and circuit-writing difficulty.
   - Recursion and proof aggregation: proving a proof, and why rollups need it.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming ZK means private — a proof can be fully public and reveal a lot by its statement.
- Under-constraining a circuit so invalid witnesses also produce valid proofs (the classic ZK bug).
- Using Keccak in-circuit and blowing up constraint count 100×.
- Deploying a Groth16 verifier for a circuit whose setup you did not verify.
- Forgetting that a proof says nothing about *who* generated it unless you constrain that too.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- A circuit proving knowledge of `x` with `x³ + x + 5 == y`; prove and verify in Go.
- Print the constraint count for that circuit.
- Separate public and private witnesses and show verification uses only the public part.

**🟡 Medium — 6 examples** *(concepts combined, and the traps)*

- Add a range check and observe the constraint-count increase.
- Use MiMC in-circuit and compare its constraint count with a naive alternative.
- A Merkle-membership circuit proving inclusion without revealing the leaf.
- Measure proving time and proof size as constraint count grows 10×.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Export the Solidity verifier, deploy it to `anvil`, and verify a proof on-chain from Go.
- Demonstrate an under-constrained circuit accepting a bogus witness, then fix it.
- Compare Groth16 and PLONK on the same circuit: setup time, proof size, verify cost.

### Packages & tools

`github.com/consensys/gnark`, `github.com/consensys/gnark-crypto`, `math/big`

### Resources to cite

- gnark docs: https://docs.gnark.consensys.io/
- gnark repository: https://github.com/Consensys/gnark
- Vitalik — QAPs / zk-SNARKs explained: https://vitalik.eth.limo/general/2016/12/10/qap.html
- ZKProof community reference: https://zkproof.org/

---

## 40 — DeFi Primitives & MEV from a Go Integrator's View

**Lesson file:** [../40-defi-primitives-mev.md](../40-defi-primitives-mev.md) · **Examples folder:** `../examples/40-defi-primitives-mev/`

| | |
|---|---|
| Prerequisites | [26](../26-erc-standards.md), [27](../27-contract-security.md) |
| Unlocks | 53 |
| Examples | **17** — 🟢 4 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 11 |

*AMMs, lending, liquidations, MEV and flash loans — the math, and the integration code that survives it*

### Goals

- Compute AMM swap outputs and price impact with exact integer math in Go.
- Explain lending health factors and liquidation mechanics.
- Describe MEV: sandwiches, arbitrage, liquidations, and PBS.
- Integrate safely: slippage, deadlines, simulation and private transactions.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The primitive stack**

   - Swap, lend, stake, derive — almost every protocol is a composition of these four.
   - Composability as the superpower and the systemic risk.
   - Reading a protocol: find the invariant it maintains, then find who can break it.
   - Your role as an integrator: you are a user with a robot's reflexes.

**2. Constant-product AMMs**

   - x·y = k; the swap formula with a 0.3% fee: out = (in·997·y)/(x·1000 + in·997).
   - Implement it in Go with `big.Int`, integer division, and rounding in the pool's favour.
   - Price impact and why it grows superlinearly with trade size.
   - Reserves, `getReserves` ordering by token address, and the token0/token1 trap.

**3. Slippage and deadlines**

   - `amountOutMin` is your only real protection and it must be computed off-chain.
   - Deadlines stop a transaction being held and executed later at a worse price.
   - Choosing a tolerance: too tight fails, too loose invites sandwiching.
   - Recomputing the quote immediately before sending, and aborting on divergence.

**4. Concentrated liquidity**

   - Uniswap v3: liquidity in price ranges, ticks, and `sqrtPriceX96` Q64.96 fixed point.
   - Converting sqrtPriceX96 to a human price in Go without losing precision — the exact formula.
   - Tick math, `tickSpacing`, and why quoting requires simulating across ticks.
   - Use the Quoter contract via `eth_call` rather than reimplementing v3 math.

**5. Impermanent loss**

   - The number, derived: IL = 2√r/(1+r) − 1 for a price ratio r.
   - Worked examples at 1.25×, 2×, 5× so it stops being a vibe.
   - When fees compensate for it and when they do not.
   - What an LP position actually is: a short-volatility trade.

**6. Lending protocols**

   - Collateral factors / LTV, borrow capacity, and the health factor formula.
   - Interest-rate curves with a kink at optimal utilisation; borrow vs supply APY.
   - Liquidation: trigger condition, close factor, liquidation bonus.
   - Computing a position's health factor in Go from on-chain reads, and monitoring it.

**7. Oracles**

   - Chainlink aggregators: `latestRoundData`, and the **staleness check you must write**.
   - Decimals on feeds (often 8, not 18) and the classic mis-scaling bug.
   - TWAPs from AMMs and their manipulation cost.
   - L2 sequencer-uptime feeds, and why ignoring them caused real liquidation incidents.
   - Full treatment in lesson 53.

**8. MEV**

   - The taxonomy: arbitrage (benign-ish), liquidations (necessary), sandwich (extractive), JIT liquidity.
   - The pipeline: searcher → builder → relay → proposer (lesson 28's PBS, from the other side).
   - Flashbots bundles, `eth_sendBundle`, and private order flow.
   - As an integrator you are usually the *prey*; design accordingly.

**9. Flash loans**

   - Borrow any amount within one transaction, repay by the end or it all reverts.
   - Atomicity as a primitive — and as an attack amplifier (capital is no longer a barrier).
   - The canonical attack shape: flash loan → manipulate oracle → borrow against it → repay.
   - Legitimate uses: collateral swaps, self-liquidation, arbitrage.

**10. Defensive integration in Go**

   - Simulate every state-changing call with `eth_call` (and state overrides) before sending.
   - Verify the receipt's *events*, not just its status — did you actually receive what you expected?
   - Cap approvals; approve exactly, revoke after, never infinite from a hot wallet.
   - Use private RPC endpoints (Flashbots Protect and equivalents) for anything sandwichable.
   - Circuit-break on abnormal price divergence between two sources.
   - Never trust a quote older than one block.

**11. Assets your service must model**

   - Stablecoins: fiat-backed vs crypto-backed vs algorithmic, and depeg risk.
   - Liquid staking tokens: rebasing vs reward-bearing, and what that does to your accounting.
   - Yield-bearing vaults (ERC-4626, lesson 49) and share-price accounting.
   - Why lesson 60's ledger design has to know about all of this.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using `float64` for price math — instant precision bugs in a financial path.
- Reading a Chainlink feed without a staleness or answer-bounds check.
- Assuming 18 decimals on a price feed (most are 8).
- Sending a swap without `amountOutMin` and a deadline.
- Trusting `getReserves` ordering without checking token0/token1.
- Granting infinite approval to a router from a hot wallet.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 17).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Compute a Uniswap v2 swap output with fees using `big.Int`.
- Read a pair's reserves and determine token0/token1 ordering.
- Read a Chainlink feed and print price with correct decimals.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Compute price impact and set `amountOutMin` for a 0.5% tolerance.
- Add a staleness check to the Chainlink read and reject an old round.
- Convert `sqrtPriceX96` to a human price exactly, without floats.
- Compute a lending position's health factor from on-chain reads.
- Compute impermanent loss for price ratios 1.25×, 2× and 5×.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Simulate a swap with `eth_call` before broadcasting and abort if the output moved more than tolerance.
- Verify a swap receipt by decoding the `Swap` event and checking the received amount.
- Build a liquidation monitor: track positions, compute health factors, and alert below threshold.
- Reproduce a flash-loan oracle manipulation on a forked chain and show the TWAP defence.

### Packages & tools

`math/big`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/ethclient`, `context`

### Resources to cite

- Uniswap v2 whitepaper: https://uniswap.org/whitepaper.pdf
- Uniswap v3 whitepaper: https://uniswap.org/whitepaper-v3.pdf
- Chainlink data feeds: https://docs.chain.link/data-feeds
- Flashbots docs: https://docs.flashbots.net/
- Aave v3 docs: https://aave.com/docs

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
