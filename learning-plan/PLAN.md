# Build Plan — Blockchain Learning (Go)

This is the **authoring spec** for the course: what every lesson must contain, in what order, with which examples. [README.md](README.md) is the reader-facing index; this file is the one you open when you want the next lesson built.

- **41 lessons**, 9 parts, strictly easy → hard.
- **Every code example is Go.** Solidity appears only as the thing Go talks to (lessons 22–27).
- **811 runnable Go examples** planned across the course (40 lessons × ~15–26; lesson 41 is a build).

---

## Authoring rules

### Lesson anatomy

Every `NN-title.md` has exactly these sections, in this order:

| Section | What goes in it |
|---|---|
| `# NN — Title` | plus a one-line status/part/prereq header |
| `## Goals` | 4–5 bullets, each an observable capability |
| `## Concepts` | **the bulk of the lesson** — long-form prose, one `###` per topic below |
| `## Exercises` | 5–8 numbered tasks the reader writes in `practice/NN-title/` |
| `## Best Practices & Pitfalls` | the habits and the traps, each with a one-line *why* |
| `## Checklist` | `- [ ]` "I can …" lines mirroring the goals |
| `## Resources` | specs/EIPs/BIPs first, then reference implementations, then articles |

### Writing style

- **Explanation-heavy.** This course is read, not skimmed. Every concept gets a paragraph of prose *before* any code: what problem it solves, what breaks without it, then the mechanism.
- **Explain the why before the how.** "Nonces exist because …" beats "a nonce is a counter".
- Short Go snippets inside `## Concepts` (5–20 lines); full programs live in the example files.
- Name the real-world incident whenever one exists (the DAO, the PS3 nonce reuse, Parity's uninitialized library, CVE-2012-2459). Concrete failures are what stick.
- Use `big.Int`/`uint256` for every on-chain value. A `float64` in a money path is a bug in a lesson too.
- Cross-link lessons with a relative link to the other lesson file, so the reader can walk back to a prerequisite.

### Example files

Each lesson gets `examples/NN-title/` containing:

```
README.md     index + how to run + the tier table
1-easy.md     🟢 concepts in isolation
2-medium.md   🟡 combinations, the common traps
3-hard.md     🔴 real-shaped, multi-concept programs
PROGRESS.md   a checkbox per example
```

Copy [`examples/_template/`](examples/_template/) to start one.

Rules for examples:

1. Each is a **complete `package main` program** — no fragments, no `...`.
2. Each has: a `## N. Title` heading, a tier + category line, 2–4 sentences of concept, numbered **Steps**, the code block, and a real **Output** block.
3. **Run it before adding it.** `go build` + `go vet` + `gofmt` clean, and the Output block is real stdout.
4. Examples that need a chain use `ethclient/simulated` or `anvil` — never mainnet, never a real key.
5. Any key material in an example is a hardcoded test key with a comment saying so.
6. Numbering is continuous across the three tier files (1 → N) and never renumbered once published.

### Build order

Write lessons in numeric order — the prerequisites below are real, and Part 3 in particular is one program growing across eight lessons. Within one lesson: `## Concepts` first, then the easy tier, then medium, then hard, then the exercises.

---

## The 41 lessons

## Part 1 — Foundations

What a blockchain actually is, the tools you need, and the byte-level primitives every later lesson assumes. No cryptography yet — just the mental model and the keyboard setup.

| # | Lesson | Examples |
|---|---|---|
| 01 | [Introduction to Blockchain](#01--introduction-to-blockchain) | 12 |
| 02 | [Environment Setup & Tooling](#02--environment-setup--tooling) | 12 |
| 03 | [Bytes, Hex, Big Integers & Encoding](#03--bytes-hex-big-integers--encoding) | 18 |

### 01 — Introduction to Blockchain

**File:** [01-introduction.md](01-introduction.md) · **Prerequisites:** none — start here · **Examples:** 12

*what problem a blockchain solves, the ledger, decentralization, the landscape — and what it is *not**

**Goals**

- Explain what a blockchain is without using the word 'blockchain'.
- Name the problem it solves (double-spend without a trusted third party) and why that is hard.
- Place Bitcoin, Ethereum, L2s and app-chains on one map.
- Decide honestly when a database is the better answer.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The double-spend problem: why digital money is hard and physical cash is not.
2. A ledger as an append-only log; replicated state machines; why order matters more than storage.
3. Decentralization as a spectrum — permissionless vs permissioned vs 'a database with signatures'.
4. The three properties chained together: hash linking (integrity), signatures (authorization), consensus (ordering).
5. The Byzantine Generals framing and why Sybil resistance (PoW/PoS) is the actual innovation.
6. History in one page: Chaum → b-money/Hashcash → Bitcoin 2009 → Ethereum 2015 → PoS Merge 2022 → rollups.
7. The landscape map: L1s, L2s, app-chains, sidechains, and where Go sits in each ecosystem.
8. Vocabulary you will meet constantly: node, client, block, tx, gas, finality, fork, mempool, validator.
9. The honest trade-off table: throughput, latency, cost and trust vs Postgres.
10. Why Go: geth, prysm, btcd, Cosmos SDK, and most indexers/infra are written in Go.

**Example seeds** — expand to 12, graded easy → hard:

- Print a 'ledger' as a slice of transfers and compute balances — no crypto yet, just the model.
- Show a double-spend by replaying the same transfer twice against a naive ledger.
- A tiny `map[string]int64` state machine that applies an ordered list of ops — two different orders, two different states.

**Go packages:** `fmt`, `slices`

---

### 02 — Environment Setup & Tooling

**File:** [02-environment-setup.md](02-environment-setup.md) · **Prerequisites:** [01](01-introduction.md) · **Examples:** 12

*Go toolchain, the module layout for this repo, node clients, testnets, faucets, and the CLI toolbox*

**Goals**

- Have a Go module ready to run every example in this repo.
- Install and sanity-check the client/tooling you will use (geth or an RPC provider, Foundry, solc).
- Get testnet funds and read a transaction on an explorer.
- Know which environment variables and secrets never go in git.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Go 1.22+ install check; `go mod init`; the `practice/` module used by this repo.
2. The Go blockchain toolbox: `github.com/ethereum/go-ethereum` (as a library, not just a node), `github.com/btcsuite/btcd`, `golang.org/x/crypto`, `github.com/holiman/uint256`.
3. Node options and the trade-off: local geth (`--dev`, `--sepolia`), Foundry's `anvil`, or a hosted RPC provider.
4. Foundry (`forge`, `cast`, `anvil`) and `solc` — why the Solidity toolchain is worth installing even for a Go dev.
5. Testnets today: Sepolia and Hoodi; faucets; why you never use mainnet to learn.
6. Block explorers as a debugging tool: reading a tx, its receipt, its logs, its internal calls.
7. Secrets discipline from day one: `.env`, never commit a private key, the `*.local` gitignore rule.
8. A verification script: connect over JSON-RPC, print the chain id, latest block number and base fee.

**Example seeds** — expand to 12, graded easy → hard:

- `go run` a program that dials an RPC URL and prints chain id + latest block number.
- Read an env var with a fallback and fail fast when a required secret is missing.
- Start `anvil`, list its funded dev accounts, and print the balance of account 0 from Go.

**Go packages:** `os`, `log/slog`, `github.com/ethereum/go-ethereum/ethclient`

---

### 03 — Bytes, Hex, Big Integers & Encoding

**File:** [03-bytes-encoding.md](03-bytes-encoding.md) · **Prerequisites:** [02](02-environment-setup.md) · **Examples:** 18

*the data primitives — `[]byte`, endianness, hex, base58, base64, `big.Int`, and fixed-size arrays*

**Goals**

- Move fluently between `[]byte`, hex strings and integers in Go.
- Explain endianness and pick the right one when serializing.
- Use `math/big` for values that overflow `uint64` — which is most of them.
- Understand why `[32]byte` and `[]byte` are different types and when each is used.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Everything on a chain is bytes: `[]byte` vs `[32]byte` vs `[20]byte`, and why go-ethereum has `common.Hash`/`common.Address`.
2. Hex encoding: `encoding/hex`, the `0x` prefix convention, `hexutil` and its quantity-vs-data rules.
3. Endianness: big-endian on the wire (Ethereum) vs little-endian (Bitcoin) — the classic reversed-txid confusion.
4. `math/big`: why wei (1e18) needs 256 bits, `SetString`, `Text`, arithmetic without operators, and the aliasing trap.
5. `uint256` as the EVM-native, allocation-free alternative to `big.Int`.
6. Fixed-width serialization: `encoding/binary`, `PutUint64`, left-padding to 32 bytes.
7. Base64 vs base58 vs base58check vs bech32 — what each is for and why base58 drops `0`, `O`, `I`, `l`.
8. Units: wei / gwei / ether, satoshi / BTC — and why you never use `float64` for money.
9. Constant-time comparison (`crypto/subtle`) and when `bytes.Equal` leaks timing.

**Example seeds** — expand to 18, graded easy → hard:

- Hex round-trip: string → bytes → string, including the odd-length error case.
- Convert 1.5 ETH to wei with `big.Int` and format it back to a decimal string.
- Left-pad a `uint64` into a 32-byte word the way the EVM does.
- Implement base58 encode/decode and show why the alphabet excludes look-alike characters.

**Go packages:** `encoding/hex`, `encoding/binary`, `math/big`, `crypto/subtle`, `github.com/holiman/uint256`

---

## Part 2 — Cryptography Foundations

The four crypto primitives every chain is built from: hashes, Merkle trees, signatures, and key derivation. You implement each one in Go before you ever see a block.

| # | Lesson | Examples |
|---|---|---|
| 04 | [Cryptographic Hash Functions](#04--cryptographic-hash-functions) | 22 |
| 05 | [Merkle Trees & Proofs](#05--merkle-trees--proofs) | 22 |
| 06 | [Keys & Digital Signatures (ECDSA on secp256k1)](#06--keys--digital-signatures-ecdsa-on-secp256k1) | 22 |
| 07 | [Addresses, Encodings & HD Wallets](#07--addresses-encodings--hd-wallets) | 22 |

### 04 — Cryptographic Hash Functions

**File:** [04-hash-functions.md](04-hash-functions.md) · **Prerequisites:** [03](03-bytes-encoding.md) · **Examples:** 22

*SHA-256, Keccak-256, the five properties, commitments, and hashing structured data in Go*

**Goals**

- Compute SHA-256, SHA-3 and Keccak-256 in Go and know which chain uses which.
- State the five properties of a cryptographic hash and what breaks when each fails.
- Use a hash as a commitment and as an identifier.
- Serialize a struct deterministically before hashing it — the mistake that breaks everything later.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. What a hash function is: arbitrary input → fixed output, deterministic, one-way.
2. The five properties: determinism, preimage resistance, second-preimage resistance, collision resistance, avalanche.
3. SHA-256 (`crypto/sha256`), SHA-3/Keccak-256 — and the crucial gotcha that Ethereum's 'sha3' is *original Keccak*, not NIST SHA-3.
4. Double-SHA256 in Bitcoin and why Satoshi did it.
5. `golang.org/x/crypto/sha3` and go-ethereum's `crypto.Keccak256` / `crypto.Keccak256Hash`.
6. Hashing structured data: canonical serialization first, or two equal structs hash differently.
7. Commitment schemes: hash(secret‖nonce), reveal later — the basis of commit–reveal and of PoW puzzles.
8. Length-extension attacks on SHA-256 (and why Keccak/SHA-3 is immune) — the reason HMAC exists.
9. Hash-to-identifier: block hashes, txids, content addressing, and why 'the hash *is* the name'.
10. Birthday bound: why 256 bits gives ~128-bit collision security, and why 160-bit addresses are still fine.

**Example seeds** — expand to 22, graded easy → hard:

- Hash a string with SHA-256 and print hex; change one bit and diff the output (avalanche).
- Keccak-256 vs SHA3-256 of the same input — show they differ, and that Ethereum uses the former.
- A commit–reveal: publish hash(secret‖nonce), then reveal and verify.
- Hash a struct via JSON with sorted keys vs `gob` — demonstrate the non-determinism trap.

**Go packages:** `crypto/sha256`, `golang.org/x/crypto/sha3`, `github.com/ethereum/go-ethereum/crypto`

---

### 05 — Merkle Trees & Proofs

**File:** [05-merkle-trees.md](05-merkle-trees.md) · **Prerequisites:** [04](04-hash-functions.md) · **Examples:** 22

*building a tree, the root as a commitment, inclusion proofs, and the traps (odd nodes, second-preimage)*

**Goals**

- Build a Merkle tree over a list of items in Go.
- Generate and verify an inclusion proof without holding the whole list.
- Explain why a block header only needs one 32-byte root.
- Know the odd-leaf duplication bug and the second-preimage defence.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The problem: prove one item is in a set of a million without sending the million.
2. Construction: hash the leaves, pair and hash upward, the root commits to everything.
3. The root in a block header — how SPV/light clients work.
4. Inclusion (Merkle) proofs: the sibling path, and verification as a fold from leaf to root.
5. Proof size is O(log n) — the whole point; work the numbers for 1M leaves.
6. Odd number of leaves: duplication (Bitcoin's CVE-2012-2459 malleability) vs promotion vs padding.
7. Second-preimage resistance: domain-separate leaves (`0x00‖leaf`) from internal nodes (`0x01‖l‖r`).
8. Sorted vs unsorted pairs; the OpenZeppelin sorted-pair convention used by airdrop allowlists.
9. Variants you will meet later: Merkle Patricia trie (lesson 17), sparse Merkle trees, Verkle trees.
10. Merkle proofs in production: allowlists, log/receipt proofs, rollup withdrawal proofs.

**Example seeds** — expand to 22, graded easy → hard:

- Build a tree over 4 leaves and print every level.
- Generate a proof for leaf 2 and verify it; then corrupt one sibling and watch verification fail.
- Handle 5 leaves three different ways and compare roots.
- Implement a sorted-pair Merkle allowlist compatible with the OpenZeppelin verifier.

**Go packages:** `crypto/sha256`, `github.com/ethereum/go-ethereum/crypto`

---

### 06 — Keys & Digital Signatures (ECDSA on secp256k1)

**File:** [06-keys-signatures.md](06-keys-signatures.md) · **Prerequisites:** [04](04-hash-functions.md) · **Examples:** 22

*private/public keys, the elliptic curve, signing, verifying, `v`-recovery, malleability and nonce disasters*

**Goals**

- Generate a secp256k1 keypair in Go and derive the public key.
- Sign a message hash and verify it; recover the public key from a signature.
- Explain what the `v`, `r`, `s` values are and why low-`s` is enforced.
- Understand why a reused or predictable nonce leaks the private key.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Asymmetric crypto in one page: a secret you keep, a value you publish, and a proof you own the secret.
2. Elliptic curves intuitively: a group, a generator point G, scalar multiplication, and why it is one-way.
3. secp256k1 (Bitcoin/Ethereum) vs P-256 (`crypto/ecdsa` default) vs ed25519 (Solana, Cosmos) — pick per chain.
4. Key generation in Go: `crypto.GenerateKey`, `FromECDSA`/`ToECDSA`, and safe entropy (`crypto/rand`, never `math/rand`).
5. You sign a *hash*, never a message; EIP-191 personal_sign prefixing and why domain separation matters.
6. ECDSA signing internals: the per-signature nonce `k`, `r`, `s`, and public-key recovery giving the `v` byte.
7. `crypto.Sign` / `crypto.Ecrecover` / `crypto.SigToPub` in go-ethereum, and the 65-byte `r‖s‖v` layout.
8. Signature malleability: `(r, s)` and `(r, -s)` both verify — EIP-2 low-`s` enforcement and Bitcoin's BIP-62.
9. The nonce catastrophes: PlayStation 3, Bitcoin Android 2013 — reuse `k` and your key is trivially recovered; RFC 6979 deterministic nonces.
10. Preview of what replaces this: Schnorr/BIP-340 (Taproot) and BLS (Ethereum consensus).

**Example seeds** — expand to 22, graded easy → hard:

- Generate a key, print the private key hex and the uncompressed public key.
- Sign a Keccak hash, verify it, then flip one byte of the message and watch verification fail.
- Recover the public key from a signature and prove it equals the signer's.
- Show malleability: derive the complementary `s` and confirm the raw verifier accepts both.

**Go packages:** `crypto/rand`, `crypto/ecdsa`, `github.com/ethereum/go-ethereum/crypto`

---

### 07 — Addresses, Encodings & HD Wallets

**File:** [07-addresses-wallets-hd.md](07-addresses-wallets-hd.md) · **Prerequisites:** [06](06-keys-signatures.md) · **Examples:** 22

*Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation paths*

**Goals**

- Derive an Ethereum address from a public key, by hand, in Go.
- Implement and verify the EIP-55 mixed-case checksum.
- Generate a BIP-39 mnemonic and derive keys along a BIP-44 path.
- Explain xpub/xprv, hardened vs non-hardened derivation, and the xpub leak.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Ethereum address = last 20 bytes of Keccak-256 of the 64-byte uncompressed public key — derived, never registered.
2. EIP-55: a checksum smuggled into letter case; how it is computed and why wallets reject a bad one.
3. Bitcoin's path: SHA-256 → RIPEMD-160 (HASH160) → version byte → base58check; then bech32/bech32m for SegWit/Taproot.
4. Why address formats differ per chain, and the cost of sending to the wrong one.
5. BIP-39: entropy → mnemonic words → checksum bits → PBKDF2 → 512-bit seed; the optional passphrase ('25th word').
6. BIP-32 hierarchical deterministic keys: master key from seed, child derivation, chain codes.
7. Hardened (`'`) vs non-hardened derivation, and the xpub + one child privkey → master key leak.
8. BIP-44 paths: `m/44'/60'/0'/0/x` for Ethereum, `m/44'/0'/0'/0/x` for Bitcoin — coin types and account structure.
9. Vanity addresses, CREATE/CREATE2 contract addresses (preview of lesson 18), and why address ≠ account.
10. Storage reality: keystore files (scrypt + AES), hardware wallets, and what a seed phrase is worth.

**Example seeds** — expand to 22, graded easy → hard:

- Public key → Keccak → last 20 bytes → address; compare against a known vector.
- Implement EIP-55 checksumming and validate a mixed-case address.
- Mnemonic → seed → `m/44'/60'/0'/0/0` → address, matching what MetaMask shows.
- Derive 5 sequential receive addresses from one xpub without any private key.

**Go packages:** `github.com/ethereum/go-ethereum/crypto`, `github.com/tyler-smith/go-bip39`, `github.com/tyler-smith/go-bip32`

---

## Part 3 — Build a Blockchain from Scratch (Go)

The core of the course. You write a working chain — blocks, mining, UTXO transactions, a wallet, persistence, a P2P network and fork choice — one lesson at a time, in plain Go.

| # | Lesson | Examples |
|---|---|---|
| 08 | [Blocks & the Chain](#08--blocks--the-chain) | 22 |
| 09 | [Proof of Work & Mining](#09--proof-of-work--mining) | 22 |
| 10 | [Transactions & the UTXO Model](#10--transactions--the-utxo-model) | 22 |
| 11 | [Wallets, Fees & the Mempool](#11--wallets-fees--the-mempool) | 22 |
| 12 | [Persistence & Chain State](#12--persistence--chain-state) | 22 |
| 13 | [P2P Networking & Gossip](#13--p2p-networking--gossip) | 22 |
| 14 | [Consensus, Forks & Reorgs](#14--consensus-forks--reorgs) | 22 |
| 15 | [The Account Model & World State](#15--the-account-model--world-state) | 22 |

### 08 — Blocks & the Chain

**File:** [08-blocks-and-chain.md](08-blocks-and-chain.md) · **Prerequisites:** [04](04-hash-functions.md), [05](05-merkle-trees.md) · **Examples:** 22

*the block struct, hash linking, the genesis block, serialization and chain validation*

**Goals**

- Define a block and a header in Go and hash it deterministically.
- Link blocks by previous-hash and detect any tampering.
- Create a genesis block and validate a whole chain.
- Separate header from body and know why the split exists.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Header vs body: what must be hashed vs what can be fetched separately; why light clients only need headers.
2. A minimal header: version, prev hash, Merkle root, timestamp, difficulty/bits, nonce, height.
3. Deterministic serialization before hashing — `encoding/binary` and a fixed field order (never JSON).
4. The chain as a linked list built from hashes rather than pointers; tamper-evidence demonstrated.
5. The genesis block: hardcoded, prev-hash all zeros, and why every node must agree on it byte-for-byte.
6. Block validation rules, split into stateless (structure, hash, timestamp bounds) and stateful (prev exists, height, difficulty).
7. Timestamps as a soft constraint: median-time-past, future drift limits, and why they are not a clock.
8. Modelling it in Go: value vs pointer receivers, `String()` for debugging, caching the computed hash.
9. Table-driven tests over the validation rules — the habit that carries through Part 3.

**Example seeds** — expand to 22, graded easy → hard:

- Define `Block`, hash it, and print the hex.
- Chain three blocks, print each prev-hash link, then mutate block 1's data and show block 2 no longer validates.
- Serialize a header to fixed-width bytes and back, asserting a byte-identical round trip.
- Put a Merkle root over 4 transactions into the header and re-verify it.

**Go packages:** `encoding/binary`, `bytes`, `crypto/sha256`, `time`

---

### 09 — Proof of Work & Mining

**File:** [09-proof-of-work.md](09-proof-of-work.md) · **Prerequisites:** [08](08-blocks-and-chain.md) · **Examples:** 22

*the difficulty target, nonce grinding, `bits`/compact encoding, difficulty retargeting and the energy argument*

**Goals**

- Implement a PoW miner that finds a nonce below a target.
- Convert between difficulty, target and the compact `bits` encoding.
- Retarget difficulty from observed block times.
- Explain what PoW actually buys (Sybil resistance) and what it costs.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The puzzle: find a nonce such that H(header) < target — easy to verify, expensive to find.
2. Target vs difficulty vs `bits`: the compact 4-byte float-like encoding and how to expand it.
3. Leading-zeros intuition vs the real big-integer comparison, and why the former is only a teaching aid.
4. Mining loop in Go: `big.Int` comparison, `uint32` nonce exhaustion and the extra-nonce trick.
5. Difficulty retargeting: Bitcoin's 2016-block window, and Ethereum's old per-block adjustment.
6. Hashrate, expected time to a block, and why block time is exponentially distributed (variance surprises people).
7. Concurrency: sharding the nonce space across goroutines with `context` cancellation — a real Go concurrency exercise.
8. What PoW actually provides: Sybil resistance and an objective ordering — not 'security' in the abstract.
9. Attacks: 51%, selfish mining, timewarp; and the honest cost/energy discussion.
10. Why Ethereum left PoW, and where PoW still makes sense.

**Example seeds** — expand to 22, graded easy → hard:

- Mine a block at difficulty 16 bits and print nonce, hash and elapsed time.
- Expand a compact `bits` value to a 256-bit target and back.
- Measure how time-to-solve scales as difficulty increases by one bit at a time.
- Parallel miner: N goroutines over disjoint nonce ranges, first winner cancels the rest via `context`.

**Go packages:** `math/big`, `context`, `sync`, `time`

---

### 10 — Transactions & the UTXO Model

**File:** [10-transactions-utxo.md](10-transactions-utxo.md) · **Prerequisites:** [06](06-keys-signatures.md), [08](08-blocks-and-chain.md) · **Examples:** 22

*inputs, outputs, the coinbase, signing a transaction, script-less locking and the UTXO set*

**Goals**

- Model UTXO transactions in Go: inputs referencing prior outputs, outputs locking value.
- Sign and verify a transaction the way Bitcoin does — over a trimmed copy.
- Build the coinbase transaction and enforce conservation of value.
- Maintain and query a UTXO set.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Accounts vs UTXO: two ways to represent 'who owns what', and their very different trade-offs.
2. The UTXO model: outputs are coins; inputs spend them entirely; change goes back to yourself.
3. Transaction structure: `[]TxInput{PrevTxID, OutIdx, Signature, PubKey}` and `[]TxOutput{Value, PubKeyHash}`.
4. The coinbase transaction: no inputs, creates the subsidy, and why it needs arbitrary data.
5. What exactly gets signed: a trimmed copy with signatures stripped — and why this is subtle.
6. Verification: for each input, recover/compare the pubkey hash and check the signature over the trimmed tx.
7. Conservation: sum(inputs) ≥ sum(outputs), the difference is the fee; double-spend detection inside a block.
8. The UTXO set as the real state: building it by scanning the chain, and the dust/fragmentation problem.
9. Coin selection: greedy vs knapsack, and how it leaks privacy.
10. Locking scripts previewed — this lesson uses a hardcoded P2PKH-shaped check; lesson 36 does real Script.

**Example seeds** — expand to 22, graded easy → hard:

- Build a coinbase tx and print its id.
- Spend one output into two (payment + change) and assert value conservation.
- Sign a tx, verify it, then swap an output's value and show verification fails.
- Build the UTXO set from a 3-block chain and compute an address balance.

**Go packages:** `crypto/ecdsa`, `bytes`, `encoding/gob`, `math/big`

---

### 11 — Wallets, Fees & the Mempool

**File:** [11-wallets-mempool.md](11-wallets-mempool.md) · **Prerequisites:** [07](07-addresses-wallets-hd.md), [10](10-transactions-utxo.md) · **Examples:** 22

*a keystore, address book, transaction construction, fee markets and mempool policy*

**Goals**

- Build a wallet that stores keys and constructs spendable transactions.
- Implement a mempool with validation, replacement and eviction.
- Select transactions for a block by fee rate.
- Explain fee estimation and replace-by-fee.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. A wallet is not coins — it is keys plus a view of the chain.
2. Persisting keys in Go: a keystore file, `encoding/gob` vs JSON, and encrypting at rest with scrypt + AES-GCM.
3. Building a spend: select UTXOs, compute the fee, create the change output, sign every input.
4. The mempool: an in-memory set of validated-but-unconfirmed txs; the Go data structure (map + fee-ordered index).
5. Mempool policy vs consensus rules — the distinction that confuses everyone.
6. Fee rate (sat/vB, gwei/gas): why size matters, not just value.
7. Block assembly as a knapsack problem; greedy fee-rate packing and child-pays-for-parent.
8. Replace-by-fee, transaction pinning, and eviction under memory pressure.
9. Concurrency: the mempool is hammered by the network and by the miner — `sync.RWMutex` vs a single owning goroutine.
10. Nonce/sequence ordering and why a gap stalls everything behind it (previewed for the account model).

**Example seeds** — expand to 22, graded easy → hard:

- Create a wallet, save it encrypted, reload it and re-derive the same address.
- Add 5 txs at different fee rates to a mempool and pack a block under a size cap.
- Reject a double-spend at mempool admission.
- Replace a pending tx with a higher-fee version (RBF) and evict the original.

**Go packages:** `encoding/gob`, `golang.org/x/crypto/scrypt`, `crypto/aes`, `sync`, `container/heap`

---

### 12 — Persistence & Chain State

**File:** [12-persistence-chainstate.md](12-persistence-chainstate.md) · **Prerequisites:** [10](10-transactions-utxo.md) · **Examples:** 22

*storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety*

**Goals**

- Persist blocks and chain state to an embedded key-value store from Go.
- Design key prefixes and iterate the chain efficiently.
- Write atomically so a crash never leaves a half-applied block.
- Reindex state from raw blocks.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Why a key-value store and not SQL: chains are written once, read by hash, and iterated by height.
2. The Go options: `bbolt` (pure Go, B+tree, simple), `badger`, `pebble`/`leveldb` (what geth actually uses).
3. Key-space design: `b:<hash>` → block, `h:<height>` → hash, `u:<txid>:<idx>` → utxo, `l` → chain tip.
4. Serialization on disk: `gob` for speed of writing, and why a real chain uses an explicit format (RLP/protobuf).
5. Atomicity: one write batch per block so tip, block and UTXO deltas commit together.
6. Iterators and cursors: walking the chain backwards from the tip; prefix scans over the UTXO set.
7. Reindexing: rebuilding the UTXO set from blocks — the recovery path and a correctness oracle for tests.
8. Pruning and snapshots: what you can throw away, and the archive-vs-full-node distinction.
9. Crash-safety testing: kill mid-write, reopen, assert invariants.
10. The interface boundary: define a `Store` interface so tests use an in-memory fake (this pays off in lesson 34).

**Example seeds** — expand to 22, graded easy → hard:

- Open a bbolt DB, put and get a block by hash.
- Persist a 5-block chain and iterate from tip to genesis.
- Apply a block's UTXO deltas in one atomic batch; simulate a failure and show nothing was applied.
- Rebuild the UTXO set by replaying blocks and diff it against the stored one.

**Go packages:** `go.etcd.io/bbolt`, `encoding/gob`, `errors`

---

### 13 — P2P Networking & Gossip

**File:** [13-p2p-networking.md](13-p2p-networking.md) · **Prerequisites:** [12](12-persistence-chainstate.md) · **Examples:** 22

*peer discovery, a handshake, inventory gossip, block/tx propagation and a minimal node in Go*

**Goals**

- Write a minimal P2P node in Go: listen, dial, handshake, exchange messages.
- Implement inventory-based gossip so blocks and txs propagate without flooding.
- Sync a new node from a peer's chain.
- Explain devp2p/libp2p at a high level and why real networks are harder.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The network layer as a separate concern: consensus decides truth, the network decides who hears about it.
2. Message framing over TCP: length-prefix, a command byte, and why you must not assume message boundaries.
3. Handshake and version negotiation: chain id, protocol version, best height, genesis check.
4. Peer lifecycle in Go: goroutine-per-connection, a write pump, deadlines as the only real cancellation.
5. Gossip that does not melt: announce inventory hashes, let peers request what they lack (`inv`/`getdata`).
6. Initial block download: headers-first, then bodies; the checkpoint/anchor idea.
7. Orphan/future blocks and the pending pool.
8. Discovery: hardcoded seeds → DNS seeds → Kademlia (devp2p discv4/discv5, libp2p).
9. Eclipse attacks, peer scoring, rate limits and ban lists — the adversarial network.
10. How this maps to real stacks: devp2p/RLPx in geth, libp2p in the consensus layer and IPFS.

**Example seeds** — expand to 22, graded easy → hard:

- A 2-node handshake over TCP printing each side's view of the other.
- Length-prefixed message codec with a round-trip test.
- Broadcast a mined block to 3 peers using `inv` → `getdata` → `block`.
- Bring a fresh node from height 0 to the network tip via headers-first sync.

**Go packages:** `net`, `bufio`, `encoding/binary`, `context`, `sync`, `time`

---

### 14 — Consensus, Forks & Reorgs

**File:** [14-consensus-forks.md](14-consensus-forks.md) · **Prerequisites:** [13](13-p2p-networking.md) · **Examples:** 22

*fork choice, chain work, reorg handling, finality, and the attacks the rules exist to stop*

**Goals**

- Implement longest-/heaviest-chain fork choice by accumulated work.
- Handle a reorg: find the common ancestor, roll back, roll forward.
- Explain probabilistic vs economic finality and confirmation counts.
- Name the failure modes: 51%, selfish mining, chain splits.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Consensus is agreement on *order*, not on data — a repeat of lesson 01 now that you have the machinery.
2. Fork choice: not 'longest chain' but most accumulated work; why height is a lie under variable difficulty.
3. The block tree: keeping side chains, the tip index, and the common-ancestor search in Go.
4. Reorg mechanics: disconnect blocks back to the fork point (restoring UTXOs), then connect the new branch.
5. Returning disconnected transactions to the mempool — the step everyone forgets.
6. Confirmations as probability, not proof; the exponential attacker-catch-up model.
7. Soft forks vs hard forks vs chain splits; consensus bugs as accidental forks (the 2013 BDB fork).
8. 51% attacks and double-spends in practice (ETC, BTG); selfish mining; the cost model.
9. Why later designs want *finality* — a bridge into proof of stake (lesson 28).
10. Testing reorgs: construct competing chains in a unit test and assert the state after switching.

**Example seeds** — expand to 22, graded easy → hard:

- Build two competing branches and pick the winner by accumulated work.
- Reorg 2 blocks deep and assert balances match the new branch.
- Return the losing branch's txs to the mempool and confirm they are re-mineable.
- Simulate an attacker with 30% hashrate over 1000 trials and chart the double-spend success rate by confirmations.

**Go packages:** `math/big`, `sort`, `sync`

---

### 15 — The Account Model & World State

**File:** [15-account-model-state.md](15-account-model-state.md) · **Prerequisites:** [10](10-transactions-utxo.md), [14](14-consensus-forks.md) · **Examples:** 22

*accounts, nonces, balances, code and storage — the state trie and how Ethereum differs from UTXO*

**Goals**

- Model an account-based ledger in Go: balance, nonce, code hash, storage root.
- Explain nonces as replay protection and ordering.
- Apply a transaction as a state transition and compute a new state root.
- Compare UTXO and account models honestly, including privacy and parallelism.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. From coins to accounts: state as `map[Address]Account` instead of a set of unspent outputs.
2. The account struct: nonce, balance, storageRoot, codeHash — and EOA vs contract account.
3. Nonces: strictly sequential per sender; replay protection, ordering, and the stuck-nonce gap problem.
4. The state transition function `σ' = Υ(σ, T)` in plain language, then in Go.
5. The state root in the header: a commitment to the entire world state (details in lesson 17).
6. Why the account model makes contracts natural and UTXO makes them awkward.
7. The trade-offs: replay protection, parallel validation, privacy, and state growth.
8. Journalling and reverts: how a failed transaction undoes its writes but still burns gas.
9. Snapshots/checkpoints in Go: copy-on-write maps vs a journal of undo entries.
10. Where each model is used today, including hybrids.

**Example seeds** — expand to 22, graded easy → hard:

- Apply a transfer to a `map[Address]Account` and print before/after state.
- Reject a tx with a wrong nonce; then show the queued-vs-pending distinction.
- Implement a journal so a reverted call restores balances exactly.
- Hash the sorted account set into a state root and show any change moves the root.

**Go packages:** `sort`, `math/big`, `maps`

---

## Part 4 — Ethereum & the EVM

From your toy chain to the real one. Ethereum's account model, RLP and the Patricia trie, the EVM as a stack machine (you build a mini one), the transaction types, and talking to a node from Go.

| # | Lesson | Examples |
|---|---|---|
| 16 | [Ethereum Architecture](#16--ethereum-architecture) | 18 |
| 17 | [RLP & the Merkle Patricia Trie](#17--rlp--the-merkle-patricia-trie) | 22 |
| 18 | [The EVM: a Stack Machine You Can Build](#18--the-evm-a-stack-machine-you-can-build) | 26 |
| 19 | [Ethereum Transactions Deep Dive](#19--ethereum-transactions-deep-dive) | 22 |
| 20 | [JSON-RPC & go-ethereum's `ethclient`](#20--json-rpc--go-ethereums-ethclient) | 22 |
| 21 | [Sending Transactions from Go](#21--sending-transactions-from-go) | 22 |

### 16 — Ethereum Architecture

**File:** [16-ethereum-architecture.md](16-ethereum-architecture.md) · **Prerequisites:** [15](15-account-model-state.md) · **Examples:** 18

*the whole machine: accounts, blocks, receipts, gas, the execution/consensus split and the client stack*

**Goals**

- Draw Ethereum end-to-end: from a signed tx to a finalized state change.
- Read a real block header field by field.
- Explain gas, the EIP-1559 fee market and where the money goes.
- Describe the post-Merge execution/consensus client split and the Engine API.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The world computer framing: one shared state machine, replicated, with metered execution.
2. Block anatomy post-Merge: parentHash, stateRoot, transactionsRoot, receiptsRoot, logsBloom, baseFeePerGas, withdrawalsRoot, blobGasUsed.
3. Three tries, three roots — state, transactions, receipts — and what each proves.
4. Receipts: status, cumulative gas, logs, bloom — the record of what a tx *did*.
5. Gas: why metering exists (halting problem, DoS), gas limit vs gas used, out-of-gas semantics.
6. EIP-1559: base fee (burned) + priority fee (to the proposer), the target-50% adjustment, and `maxFeePerGas`.
7. The Merge: execution layer (geth/erigon/nethermind) + consensus layer (prysm/lighthouse) over the Engine API and JWT.
8. Slots, epochs, proposers and attesters — just enough to read a block explorer (details in lesson 28).
9. Blobs (EIP-4844) and the separate blob-gas market — why L2s got cheap.
10. The upgrade cadence: EIPs, hard-fork names, and how to read an EIP.
11. Where Go sits: geth is the reference implementation and also your library.

**Example seeds** — expand to 18, graded easy → hard:

- Fetch a real block header and print every field with units.
- Fetch a receipt and decode status, gas used and log count.
- Compute the effective gas price of a 1559 tx and split the burn from the tip.
- Show the base fee changing across 20 consecutive blocks against target utilization.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient`, `core/types`, `math/big`

---

### 17 — RLP & the Merkle Patricia Trie

**File:** [17-rlp-merkle-patricia-trie.md](17-rlp-merkle-patricia-trie.md) · **Prerequisites:** [05](05-merkle-trees.md), [16](16-ethereum-architecture.md) · **Examples:** 22

*Ethereum's serialization format and the trie that produces every root in the header*

**Goals**

- Encode and decode RLP by hand and with go-ethereum's `rlp` package.
- Explain the MPT node types and hex-prefix encoding.
- Build a small trie and compute its root.
- Verify a Merkle proof against a state root.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. RLP is not a schema: it encodes only nested byte arrays — the type must be known by the reader.
2. The RLP rules in full: single bytes, short/long strings, short/long lists, and the length-of-length trick.
3. Encoding integers: big-endian, no leading zeros, and why zero is the empty string.
4. `rlp.Encode`/`rlp.DecodeBytes` in Go, struct tags, and the `rlp:"optional"`/`"nil"` cases.
5. Why a plain Merkle tree is not enough: the state changes constantly and needs key→value lookup, not just membership.
6. The Merkle Patricia Trie: radix trie + Merkle hashing; branch, extension and leaf nodes.
7. Hex-prefix (compact) encoding of nibble paths, and the odd-length/terminator flag.
8. Node hashing, the <32-byte inline-node rule, and the empty root constant.
9. The four tries in Ethereum: world state, per-account storage, transactions, receipts.
10. Proofs: `eth_getProof`, verifying an account or storage slot against the state root — trustless reads.
11. What comes next: Verkle tries and stateless clients, in one paragraph.

**Example seeds** — expand to 22, graded easy → hard:

- RLP-encode `"dog"`, `[]`, `[[],[[]]]` and a 1024-byte string; check against the yellow-paper vectors.
- Round-trip a struct through `rlp` and inspect the bytes.
- Insert 4 key/value pairs into a trie and print the root after each insert.
- Fetch an `eth_getProof` result and verify a storage slot against the block's state root.

**Go packages:** `github.com/ethereum/go-ethereum/rlp`, `trie`, `common`

---

### 18 — The EVM: a Stack Machine You Can Build

**File:** [18-evm.md](18-evm.md) · **Prerequisites:** [17](17-rlp-merkle-patricia-trie.md) · **Examples:** 26

*opcodes, the stack, memory, storage, calldata, gas accounting — and a mini-EVM written in Go*

**Goals**

- Explain the EVM's execution model: 256-bit stack, volatile memory, persistent storage.
- Read raw bytecode and disassemble it.
- Implement an interpreter for a useful subset of opcodes in Go.
- Trace gas consumption and explain why storage is so expensive.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The machine: a 1024-deep stack of 256-bit words, byte-addressed memory, a persistent key→value storage per contract.
2. Why 256 bits: hashes and addresses fit natively; the cost is that Go needs `big.Int`/`uint256`.
3. The data locations and their costs: stack (free-ish), memory (quadratic expansion), storage (SSTORE, warm/cold, EIP-2929), calldata, code, returndata.
4. The opcode families: arithmetic, comparison/bitwise, environment, memory/storage, control flow (JUMP/JUMPDEST), logging, system (CALL/CREATE).
5. JUMPDEST analysis: why jumps must land on a marked byte and how that stops jump-oriented attacks.
6. Contract creation: init code returns runtime code; CREATE vs CREATE2 addressing (counterfactual deployment).
7. The call family: CALL, DELEGATECALL (the proxy/upgrade mechanism), STATICCALL, and value/gas semantics.
8. The 63/64 gas rule, out-of-gas, REVERT vs INVALID vs STOP, and how revert reasons are returned.
9. Writing an interpreter in Go: a `[]byte` program counter loop, a jump table of handlers, an explicit gas meter.
10. Reading real traces: `debug_traceTransaction`, structured logs, and how to find where gas went.
11. The road ahead: EOF, account abstraction (EIP-4337/7702) in one paragraph each.

**Example seeds** — expand to 26, graded easy → hard:

- Disassemble `0x6001600201` and explain each byte.
- Interpret PUSH/ADD/MUL/POP with a stack printed after every step.
- Add MSTORE/MLOAD with quadratic memory-expansion gas.
- Implement SSTORE/SLOAD with warm/cold accounting and show the cost difference.
- Run a real compiled contract's runtime bytecode through your interpreter and match the result.

**Go packages:** `github.com/holiman/uint256`, `github.com/ethereum/go-ethereum/core/vm`, `core/asm`

---

### 19 — Ethereum Transactions Deep Dive

**File:** [19-transaction-types.md](19-transaction-types.md) · **Prerequisites:** [06](06-keys-signatures.md), [17](17-rlp-merkle-patricia-trie.md) · **Examples:** 22

*legacy, 2930, 1559 and 4844 transaction types — signing, RLP, chain id, and computing the hash*

**Goals**

- Construct and sign every live Ethereum transaction type in Go.
- Explain EIP-155 chain-id replay protection and the `v` encoding.
- Compute a transaction hash and sender address from raw bytes.
- Choose the right type and fee fields for a given situation.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The typed-transaction envelope (EIP-2718): a leading type byte, then type-specific RLP payload.
2. Type 0 (legacy): nonce, gasPrice, gasLimit, to, value, data, v, r, s.
3. EIP-155: chain id folded into `v` (`v = 35 + 2*chainId + parity`) and why it exists (ETH/ETC replay).
4. Type 1 (EIP-2930): access lists and cold-access discounts.
5. Type 2 (EIP-1559): maxFeePerGas vs maxPriorityFeePerGas vs baseFee; the refund and the burn.
6. Type 3 (EIP-4844): blob-carrying transactions, KZG commitments, and the separate blob fee market.
7. The signing hash: which fields, in which order, with which prefix — get this wrong and the sender is a stranger.
8. `types.NewTx`, `types.SignTx`, `types.Sender`, and the `Signer` types in go-ethereum.
9. The transaction hash vs the signing hash — two different digests people constantly confuse.
10. Sender recovery: from `(r, s, v)` plus the signing hash, with no 'from' field ever transmitted.
11. Contract creation transactions (`to == nil`) and the resulting address.
12. Practical rules: gas limit vs estimate headroom, nonce gaps, stuck-tx replacement.

**Example seeds** — expand to 22, graded easy → hard:

- Build and sign a legacy tx; print its RLP and hash.
- Build a 1559 tx and decode the typed envelope byte by byte.
- Recover the sender from a raw signed tx pulled off mainnet.
- Show that changing the chain id changes the signature and breaks replay.

**Go packages:** `github.com/ethereum/go-ethereum/core/types`, `crypto`, `rlp`, `math/big`

---

### 20 — JSON-RPC & go-ethereum's `ethclient`

**File:** [20-json-rpc-ethclient.md](20-json-rpc-ethclient.md) · **Prerequisites:** [16](16-ethereum-architecture.md) · **Examples:** 22

*the node API, `ethclient` from Go, queries, filters, subscriptions and the batching/rate-limit realities*

**Goals**

- Call the Ethereum JSON-RPC API directly and through `ethclient`.
- Read blocks, receipts, balances, storage and code from Go.
- Subscribe to new heads and logs over WebSocket, with reconnection.
- Handle provider limits, batching and errors like an engineer, not an optimist.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. JSON-RPC 2.0 over HTTP/WS/IPC: the request envelope, the `0x` quantity encoding, and error objects.
2. The methods you will use daily: `eth_blockNumber`, `getBlockByNumber`, `getTransactionReceipt`, `getBalance`, `getLogs`, `call`, `estimateGas`, `getProof`.
3. Block tags: `latest`, `safe`, `finalized`, `pending` — and why `latest` can be reorged out.
4. `ethclient` in Go: `Dial`, `BlockByNumber`, `TransactionReceipt`, `FilterLogs`, `SubscribeNewHead`.
5. `rpc.Client` for raw and batch calls when `ethclient` has no wrapper.
6. `eth_call` and state overrides: reading contract state without spending gas; historical calls at a block.
7. Subscriptions over WebSocket, `ethereum.Subscription`, `Err()` channels, and mandatory reconnect logic.
8. Provider realities: rate limits, `getLogs` block-range caps, archive vs full nodes, inconsistent error strings.
9. Retries with backoff, context deadlines, and never trusting a single provider (bridge to lesson 35).
10. Debug/trace namespaces (`debug_traceTransaction`, `trace_block`) and which providers expose them.

**Example seeds** — expand to 22, graded easy → hard:

- Dial a node, print chain id, latest block, gas price and base fee.
- Fetch a block and list its transactions with values in ether.
- `FilterLogs` for one contract over a 1000-block range, chunked to respect provider caps.
- Subscribe to new heads and print each one; kill the connection and show the reconnect loop recovering.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient`, `rpc`, `context`, `time`

---

### 21 — Sending Transactions from Go

**File:** [21-sending-transactions.md](21-sending-transactions.md) · **Prerequisites:** [19](19-transaction-types.md), [20](20-json-rpc-ethclient.md) · **Examples:** 22

*nonce management, gas estimation, 1559 fees, signing, broadcast, confirmation and stuck-tx recovery*

**Goals**

- Send a signed transaction from Go, end to end.
- Manage nonces correctly under concurrency.
- Set 1559 fees that actually get included without overpaying.
- Wait for confirmations safely and recover a stuck transaction.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The full pipeline: build → estimate → price → sign → send → wait → confirm → handle reorg.
2. Nonce sources: `PendingNonceAt` vs `NonceAt`, and why both lie under concurrency.
3. A nonce manager in Go: a mutex-guarded per-address counter, reservation, and release-on-failure.
4. Gas estimation: `EstimateGas`, why it can fail on state-dependent reverts, and how much headroom to add.
5. Fee strategy: `SuggestGasTipCap`, base fee × 2 headroom, and how to reason about `maxFeePerGas`.
6. Signing options: raw `types.SignTx`, `bind.TransactOpts`, keystore, or a remote signer (bridge to lesson 32).
7. Broadcast and its error taxonomy: `already known`, `replacement underpriced`, `nonce too low`, `insufficient funds`.
8. Waiting for a receipt: `bind.WaitMined`, polling with backoff, and checking `receipt.Status` — success is not automatic.
9. Confirmation depth and reorg safety: never treat 1 confirmation as final for value.
10. Stuck transactions: cancel (self-send, same nonce, higher fee) and speed-up.
11. Idempotency: making a resend safe when your process crashes between sign and send.

**Example seeds** — expand to 22, graded easy → hard:

- Send 0.001 ETH on a local `anvil` chain and wait for the receipt.
- Fire 5 concurrent sends from one key with a nonce manager and show all 5 land in order.
- Trigger `replacement underpriced`, then successfully replace the tx with a higher tip.
- Cancel a pending transaction by replacing it with a zero-value self-send at the same nonce.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient`, `accounts/abi/bind`, `core/types`, `sync`

---

## Part 5 — Smart Contracts from Go

Enough Solidity to read a contract, then everything on the Go side: ABI encoding, `abigen` bindings, events and logs, the ERC standards, and the security bugs that drain contracts.

| # | Lesson | Examples |
|---|---|---|
| 22 | [Solidity Basics for Go Developers](#22--solidity-basics-for-go-developers) | 18 |
| 23 | [The Contract ABI: Encoding & Decoding](#23--the-contract-abi-encoding--decoding) | 26 |
| 24 | [Type-Safe Contracts with `abigen`](#24--type-safe-contracts-with-abigen) | 22 |
| 25 | [Events, Logs & Indexing Them](#25--events-logs--indexing-them) | 26 |
| 26 | [The ERC Standards from Go](#26--the-erc-standards-from-go) | 22 |
| 27 | [Smart Contract Security](#27--smart-contract-security) | 22 |

### 22 — Solidity Basics for Go Developers

**File:** [22-solidity-basics.md](22-solidity-basics.md) · **Prerequisites:** [18](18-evm.md) · **Examples:** 18

*enough Solidity to read, compile and reason about a contract — from a Go engineer's perspective*

**Goals**

- Read a Solidity contract and predict its storage layout and gas behaviour.
- Write, compile and deploy a small contract with `solc`/Foundry.
- Map Solidity concepts onto the EVM mechanics you already know.
- Recognise the constructs that show up in every ABI you will decode.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Contract anatomy: pragma, license, state variables, constructor, functions, modifiers, events, errors.
2. The type system: `uint256` by default, `address`/`address payable`, `bytes32` vs `bytes` vs `string`, fixed vs dynamic arrays, structs, enums, mappings.
3. Data location as a first-class concept: `storage` vs `memory` vs `calldata` — and the aliasing surprise.
4. Storage layout: slot packing, mappings at `keccak(key . slot)`, dynamic arrays — this is what you read with `eth_getStorageAt`.
5. Visibility (`public`/`external`/`internal`/`private`) and mutability (`view`/`pure`/`payable`) and what each costs.
6. Events and indexed parameters — the producer side of lesson 25.
7. Errors: `require`, `revert`, custom errors, and how a revert reason reaches your Go client.
8. Inheritance, interfaces, abstract contracts, and libraries — enough to navigate OpenZeppelin.
9. Compilation output: ABI JSON, deployed bytecode, metadata hash, and why bytecode differs by compiler version.
10. The Go dev's cheat sheet: Solidity type → ABI type → Go type mapping.
11. Foundry workflow: `forge build`, `forge test`, `forge create`, and `cast` for one-off calls.

**Example seeds** — expand to 18, graded easy → hard:

- Compile a `Counter` contract with `solc` and print the ABI and bytecode from Go.
- Read a public variable's storage slot directly with `eth_getStorageAt` and decode it.
- Compute a mapping slot with `keccak256(key . slot)` in Go and read a balance without any ABI call.
- Trigger a custom error and decode its selector and arguments in Go.

**Go packages:** `os/exec (solc)`, `encoding/json`, `github.com/ethereum/go-ethereum/accounts/abi`

---

### 23 — The Contract ABI: Encoding & Decoding

**File:** [23-abi-encoding.md](23-abi-encoding.md) · **Prerequisites:** [22](22-solidity-basics.md) · **Examples:** 26

*function selectors, head/tail encoding, dynamic types, and doing it all by hand in Go*

**Goals**

- Compute a function selector and encode arguments by hand.
- Explain the head/tail layout for dynamic types.
- Encode and decode with go-ethereum's `abi` package.
- Decode an unknown calldata blob from a block explorer.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The ABI is a calling convention, not a format: calldata = 4-byte selector ‖ encoded args.
2. Selectors: `keccak256("transfer(address,uint256)")[:4]`, canonical signature rules, and selector collisions.
3. Static encoding: everything padded to 32 bytes, left-padded for numbers/addresses, right-padded for `bytesN`.
4. Dynamic encoding: the head holds an offset, the tail holds length ‖ data — worked through byte by byte.
5. Nested dynamic types (`string[]`, `bytes[]`, tuples with dynamic members) and the offsets-relative-to-what rule.
6. Tuples/structs and how they map to Go structs with `abi:` tags.
7. Return data decoding and the empty-return / non-contract-address pitfall.
8. Revert data: `Error(string)` selector `0x08c379a0`, `Panic(uint256)` `0x4e487b71`, and custom errors.
9. Go APIs: `abi.JSON`, `Pack`, `Unpack`, `UnpackIntoInterface`, `abi.Arguments`, and the `abi.NewType` builder.
10. `abi.encodePacked` and its collision hazard — why signing packed data is a bug factory.
11. EIP-712 typed structured data: domain separator, struct hash, and signing off-chain orders.

**Example seeds** — expand to 26, graded easy → hard:

- Compute the `transfer(address,uint256)` selector and compare with the known `0xa9059cbb`.
- Encode `(address, uint256)` by hand into 68 bytes and diff against `abi.Pack`.
- Encode `(uint256, string, uint256[])` and annotate every 32-byte word.
- Decode a real mainnet calldata blob with only the ABI JSON.
- Sign and verify an EIP-712 typed message in Go.

**Go packages:** `github.com/ethereum/go-ethereum/accounts/abi`, `crypto`, `common`

---

### 24 — Type-Safe Contracts with `abigen`

**File:** [24-abigen-bindings.md](24-abigen-bindings.md) · **Prerequisites:** [23](23-abi-encoding.md) · **Examples:** 22

*generating Go bindings, deploying, calling, transacting, and the simulated backend*

**Goals**

- Generate Go bindings for a contract with `abigen`.
- Deploy and interact with a contract entirely from Go.
- Use `CallOpts`/`TransactOpts` correctly, including historical calls.
- Test contract interaction against a simulated backend with no node running.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. What `abigen` produces: a deployer, a caller, a transactor, a filterer, and typed event structs.
2. The toolchain: `solc --abi --bin` → `abigen --abi --bin --pkg --out`, and doing it via `go:generate`.
3. `bind.CallOpts`: `Pending`, `BlockNumber` for historical reads, and `Context` for deadlines.
4. `bind.TransactOpts`: signer function, nonce, value, gas fields — and why you usually let it fill them in.
5. Deploying from Go and waiting for the deployment receipt and address.
6. The generated event filterer and watcher, and how it maps onto lesson 25's raw logs.
7. `simulated.Backend` (formerly `backends.SimulatedBackend`): an in-process chain for tests, with `Commit()` control over block production.
8. When bindings get in the way: multicall, proxies whose ABI differs from their code, and raw `abi.Pack` fallbacks.
9. Versioning pain: abigen and go-ethereum API churn, and pinning your generator.
10. Keeping generated code out of review noise — where to put it and how to regenerate reproducibly.

**Example seeds** — expand to 22, graded easy → hard:

- Generate bindings for `Counter` and call `Number()` from Go.
- Deploy to a simulated backend, `Increment()`, commit, and read the new value.
- Read an ERC-20's `balanceOf` at a historical block with `CallOpts.BlockNumber`.
- Wire `go:generate` so `go generate ./...` rebuilds bindings from the ABI.

**Go packages:** `github.com/ethereum/go-ethereum/accounts/abi/bind`, `ethclient/simulated`, `testing`

---

### 25 — Events, Logs & Indexing Them

**File:** [25-events-logs.md](25-events-logs.md) · **Prerequisites:** [23](23-abi-encoding.md) · **Examples:** 26

*topics, the bloom filter, `eth_getLogs` at scale, decoding, and subscription vs polling*

**Goals**

- Decode an event log into typed Go values, including indexed parameters.
- Query logs efficiently with topic filters and block ranges.
- Explain the logs bloom filter and what it is actually good for.
- Choose between polling and subscribing, and survive reorgs either way.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. What a log is: a contract-emitted record, cheap to write, unreadable to contracts, essential to applications.
2. Log structure: address, up to 4 topics, data, and the position fields (block, index, tx hash).
3. Topic0 as the event signature hash; indexed parameters as topics 1–3; the rest ABI-encoded in `data`.
4. The indexed-dynamic-type gotcha: indexed `string`/`bytes` store a *hash*, so the value is unrecoverable.
5. Anonymous events and when they are used.
6. The bloom filter in the header: a probabilistic 'maybe' index; false positives by design; why it is only a prefilter.
7. `eth_getLogs` in practice: address + topic filters, OR-semantics arrays, block-range limits, and chunking.
8. Decoding in Go: `abi.Events`, `UnpackLog`/`UnpackIntoInterface`, and manual topic decoding.
9. Polling vs `SubscribeFilterLogs`: latency, gaps, and why production systems poll *and* subscribe.
10. Reorgs and logs: `removed: true`, and the rule that you must be able to un-apply an event (bridge to lesson 31).
11. Ordering guarantees: (blockNumber, logIndex) as the only stable sort key.

**Example seeds** — expand to 26, graded easy → hard:

- Compute an event's topic0 and match it against a real Transfer log.
- Decode a Transfer log into `from`, `to`, `value` with typed Go values.
- Fetch all Transfers for one address across a 50k-block range using chunking.
- Show an indexed `string` parameter arriving as a hash and explain the consequence.
- Handle a `removed: true` log by reversing its effect on local state.

**Go packages:** `github.com/ethereum/go-ethereum/accounts/abi`, `ethclient`, `core/types`

---

### 26 — The ERC Standards from Go

**File:** [26-erc-standards.md](26-erc-standards.md) · **Prerequisites:** [24](24-abigen-bindings.md), [25](25-events-logs.md) · **Examples:** 22

*ERC-20, ERC-721, ERC-1155, ERC-165, permit and multicall — interacting with all of them*

**Goals**

- Interact with ERC-20, ERC-721 and ERC-1155 tokens from Go.
- Handle decimals, non-standard tokens and missing return values.
- Detect interface support with ERC-165.
- Use permit (EIP-2612) and batch reads to cut RPC calls.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Why standards exist: a shared ABI is what makes wallets, DEXs and indexers possible.
2. ERC-20: the six functions, two events, and the allowance model; `approve` race and `increaseAllowance`.
3. The non-standard token minefield: USDT returning nothing, tokens with 6 decimals, fee-on-transfer, rebasing.
4. Decimals are display-only: always compute in the smallest unit with `big.Int`.
5. ERC-721: ownership, `tokenURI`, `safeTransferFrom` and the receiver hook; enumerable/metadata extensions.
6. ERC-1155: multi-token semantics, batch ops, and the different event shape.
7. Metadata and IPFS: resolving `ipfs://` URIs, gateways, and caching.
8. ERC-165 interface detection and its limits.
9. EIP-2612 `permit`: gasless approvals via an EIP-712 signature — the client-side flow in Go.
10. Reading many balances cheaply: Multicall3, and `eth_call` state overrides as an alternative.
11. Token accounting for an indexer: mints, burns, and reconstructing balances from Transfer logs.

**Example seeds** — expand to 22, graded easy → hard:

- Read name/symbol/decimals/totalSupply of a mainnet ERC-20 from Go.
- Format a raw balance into a human string using decimals — with `big.Rat`, not floats.
- Handle a token whose `transfer` returns no value without the binding panicking.
- Batch 100 `balanceOf` reads into one Multicall3 `eth_call`.

**Go packages:** `github.com/ethereum/go-ethereum/accounts/abi/bind`, `math/big`

---

### 27 — Smart Contract Security

**File:** [27-contract-security.md](27-contract-security.md) · **Prerequisites:** [26](26-erc-standards.md) · **Examples:** 22

*the bug classes that drain contracts, how to spot them, and how to defend as an integrator*

**Goals**

- Recognise the major on-chain vulnerability classes and their real-world incidents.
- Apply checks-effects-interactions and reentrancy guards.
- Audit an integration for the risks *your* Go service creates.
- Use the standard tooling (Slither, fuzzing, invariants) at a working level.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Reentrancy: the DAO, and how a callback re-enters before state is written; checks-effects-interactions; `nonReentrant`; read-only reentrancy.
2. Access control: missing `onlyOwner`, uninitialized proxies (Parity), `tx.origin` phishing.
3. Arithmetic: pre-0.8 overflow, and the modern hazards — unchecked blocks, precision loss from division order.
4. External calls: unchecked `call` return values, gas griefing, `transfer`'s 2300-gas trap, push-vs-pull payments.
5. Oracle manipulation: spot price vs TWAP, flash-loan-amplified attacks.
6. Front-running and MEV as a security property: sandwiches, commit–reveal, private mempools.
7. Signature bugs: replay across chains/contracts, missing nonce, `ecrecover` returning zero, malleability.
8. Proxies and upgrades: storage collisions, function-selector clashes, initializer discipline.
9. Randomness: why `block.timestamp`/`blockhash` are attacker-controlled, and what VRFs do.
10. Integrator-side defence in Go: never trust `eth_call` on unverified code, simulate before sending, validate token behaviour, cap approvals, verify receipts and log contents rather than assuming.
11. Tooling: Slither, Echidna/Foundry fuzzing and invariant tests, and what an audit does and does not buy.

**Example seeds** — expand to 22, graded easy → hard:

- Deploy a vulnerable vault on a simulated chain and drain it with a reentrant attacker contract.
- Fix it with checks-effects-interactions and prove the same attack fails.
- Replay an EIP-712 signature on a second chain id to demonstrate missing domain separation.
- From Go: detect a fee-on-transfer token by simulating a transfer and comparing balance deltas.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient/simulated`, `accounts/abi/bind`, `testing`

---

## Part 6 — Consensus & Scaling

How agreement is actually reached at scale: Ethereum's proof of stake, the BFT family, and the layer-2 designs (rollups, channels, data availability) that everything is migrating to.

| # | Lesson | Examples |
|---|---|---|
| 28 | [Proof of Stake & Ethereum Consensus](#28--proof-of-stake--ethereum-consensus) | 18 |
| 29 | [Alternative Consensus: BFT, PoA & the Rest](#29--alternative-consensus-bft-poa--the-rest) | 15 |
| 30 | [Layer 2 & Scaling](#30--layer-2--scaling) | 15 |

### 28 — Proof of Stake & Ethereum Consensus

**File:** [28-proof-of-stake.md](28-proof-of-stake.md) · **Prerequisites:** [14](14-consensus-forks.md), [16](16-ethereum-architecture.md) · **Examples:** 18

*validators, slots and epochs, LMD-GHOST + Casper FFG, attestations, finality and slashing*

**Goals**

- Explain how Ethereum reaches consensus today, end to end.
- Distinguish fork choice (LMD-GHOST) from finality (Casper FFG).
- Describe the validator lifecycle: deposit, activation, duties, exit, withdrawal.
- Explain slashing conditions and what 'finalized' actually guarantees.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. From work to stake: Sybil resistance bought with capital instead of electricity.
2. The validator: 32 ETH, a BLS key pair, and separate withdrawal credentials.
3. Time structure: 12-second slots, 32-slot epochs, committees, and the proposer/attester roles.
4. Attestations: what a validator votes on (head, source, target) and how votes aggregate with BLS.
5. LMD-GHOST fork choice: latest message driven, greediest heaviest observed subtree — with a worked example.
6. Casper FFG finality: justification and finalization, the two-epoch path, and `safe` vs `finalized` block tags.
7. Economic security: rewards, penalties, inactivity leak, and why finality is *economic*, not absolute.
8. Slashing conditions: double proposal, surround votes — and the correlation penalty.
9. The execution/consensus split in practice: Engine API, `forkchoiceUpdated`, `newPayload`, JWT auth.
10. MEV-Boost and proposer-builder separation: who actually builds the block you see.
11. Withdrawals (EIP-4895), the exit queue, and liquid staking's centralization pressure.
12. Reading the beacon chain from Go: the beacon REST API, and the Go clients (Prysm) as reference code.

**Example seeds** — expand to 18, graded easy → hard:

- Simulate LMD-GHOST over a small block tree with weighted votes and pick the head.
- Implement the FFG justification/finalization state machine over 6 epochs.
- Detect a surround vote between two attestations — the slashing check.
- Query the beacon API for the current finalized checkpoint and epoch from Go.

**Go packages:** `net/http`, `encoding/json`, `sort`

---

### 29 — Alternative Consensus: BFT, PoA & the Rest

**File:** [29-alternative-consensus.md](29-alternative-consensus.md) · **Prerequisites:** [28](28-proof-of-stake.md) · **Examples:** 15

*PBFT, Tendermint, HotStuff, Clique PoA, DPoS — what each assumes and what each buys*

**Goals**

- Explain the BFT family and the 3f+1 bound.
- Compare instant-finality BFT with probabilistic longest-chain consensus.
- Choose a consensus mechanism for a given requirement set.
- Implement a simplified BFT round in Go.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The formal framing: safety vs liveness, synchrony assumptions, and the FLP impossibility result.
2. Crash fault tolerance (Raft/Paxos) vs Byzantine fault tolerance — and why chains need the latter.
3. The 3f+1 bound: why you need 2f+1 votes and where the number comes from.
4. PBFT: pre-prepare, prepare, commit, view change — and its O(n²) message cost.
5. Tendermint/CometBFT: propose, prevote, precommit, instant finality, and the halt-instead-of-fork trade-off.
6. HotStuff and its linear communication; used by Diem-lineage chains.
7. Proof of Authority (Clique): signer sets, block sealing, and why it suits testnets and consortium chains.
8. DPoS and delegated variants; Nakamoto vs BFT on the decentralization axis.
9. Proof of History, proof of space, and other Sybil-resistance mechanisms in one paragraph each.
10. The decision table: validator-set size, finality latency, fork tolerance, permissioning, throughput.
11. Go relevance: CometBFT and the Cosmos SDK are Go, and geth's Clique is a readable implementation.

**Example seeds** — expand to 15, graded easy → hard:

- Implement one PBFT-style round with 4 nodes and 1 Byzantine node; show it still commits.
- Show the same setup failing with 2 Byzantine nodes out of 4.
- Implement Clique-style round-robin sealing with in-turn/out-of-turn delays.
- Compare message counts of PBFT vs a gossip broadcast as n grows.

**Go packages:** `sync`, `context`, `time`, `sort`

---

### 30 — Layer 2 & Scaling

**File:** [30-layer2-scaling.md](30-layer2-scaling.md) · **Prerequisites:** [17](17-rlp-merkle-patricia-trie.md), [28](28-proof-of-stake.md) · **Examples:** 15

*rollups (optimistic and zk), data availability, blobs, channels, bridges — and what they mean for your Go code*

**Goals**

- Explain the rollup thesis and what makes an L2 an L2.
- Compare optimistic and zk rollups on trust, latency and cost.
- Describe how data availability and EIP-4844 blobs changed L2 economics.
- Adapt Go client code for L2 quirks: finality, fees, reorgs and bridges.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The scalability trilemma, and why 'just raise the gas limit' is not an answer.
2. The rollup thesis: execute off-chain, publish data on-chain, settle disputes or proofs on L1.
3. Optimistic rollups: fraud proofs, the 7-day challenge window, sequencers and forced inclusion.
4. ZK rollups: validity proofs, prover cost, EVM-equivalence levels (type 1–4).
5. Data availability: calldata → blobs (EIP-4844), KZG commitments, blob gas, and DA layers/committees.
6. Sequencers: centralized today; soft confirmations vs L1 finality — the trap for Go services.
7. Bridges: canonical vs third-party, lock-mint vs burn-mint, and why bridges are the biggest hack target.
8. State channels and payment channels (Lightning): the older scaling idea and where it still wins.
9. Sidechains and validiums — and why they are *not* rollups.
10. L2 differences that break naive code: different fee components (L1 data fee), non-standard RPC fields, reorg behaviour, chain-id handling.
11. Reading an L2 from Go: OP-stack and Arbitrum RPC extras, and where `ethclient` still just works.

**Example seeds** — expand to 15, graded easy → hard:

- Compute the true cost of an L2 transaction: L2 execution + L1 data fee.
- Decode a blob-carrying transaction's commitments from a real block.
- Talk to an OP-stack testnet with `ethclient` and print the fields that differ from L1.
- Implement a two-party payment channel with signed balance updates and a unilateral close.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient`, `crypto/kzg4844`, `math/big`

---

## Part 7 — Production Blockchain Engineering in Go

The job. Indexers that survive reorgs, key management that survives an audit, node operations, deterministic tests against a simulated chain, and the observability that tells you when RPC lies.

| # | Lesson | Examples |
|---|---|---|
| 31 | [Building a Blockchain Indexer in Go](#31--building-a-blockchain-indexer-in-go) | 26 |
| 32 | [Key Management & Signing Services](#32--key-management--signing-services) | 22 |
| 33 | [Node Operations & RPC Infrastructure](#33--node-operations--rpc-infrastructure) | 15 |
| 34 | [Testing Blockchain Code in Go](#34--testing-blockchain-code-in-go) | 22 |
| 35 | [Observability & Reliability for Chain Services](#35--observability--reliability-for-chain-services) | 15 |

### 31 — Building a Blockchain Indexer in Go

**File:** [31-blockchain-indexer.md](31-blockchain-indexer.md) · **Prerequisites:** [25](25-events-logs.md), [20](20-json-rpc-ethclient.md) · **Examples:** 26

*ingesting blocks, reorg-safe writes, backfill, idempotency and the schema that makes queries possible*

**Goals**

- Build a service that ingests blocks and logs into a database, continuously.
- Handle reorgs correctly — detect, roll back, re-apply.
- Backfill history at speed without hammering your provider.
- Design a schema and cursor model that make the indexer restartable and idempotent.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Why you index: RPC cannot answer 'all transfers for this user, sorted, paginated'.
2. The ingestion loop: poll head → fetch range → decode → write → advance cursor, with a confirmation lag.
3. Reorg detection: store block hash *and* parent hash; on mismatch, walk back to the common ancestor.
4. Reorg-safe writes: block-scoped rows, delete-by-block rollback, and never mutating aggregates in place.
5. Idempotency: a natural key of (blockHash, logIndex) plus upserts, so replay is safe.
6. Backfill vs live tail: two modes, chunked ranges, bounded concurrency with `errgroup`, and ordered commits.
7. Schema design: blocks, transactions, logs, and a decoded per-event table; the indexes queries actually need.
8. Cursors and checkpoints: exactly-once effects from at-least-once processing.
9. Rate limits and provider failover; adaptive chunk sizing when `getLogs` returns 'too many results'.
10. Operational concerns: lag metrics, gap detection, and a reindex command.
11. Testing it: a fake chain source that can produce a reorg on demand (bridge to lesson 34).

**Example seeds** — expand to 26, graded easy → hard:

- Poll for new heads and print each with a 12-block confirmation lag.
- Detect a reorg by comparing stored parent hashes and roll back 3 blocks.
- Backfill 100k blocks of one contract's logs with bounded concurrency and ordered commits.
- Prove idempotency: run the same range twice and assert row counts are unchanged.
- Adaptive chunk sizing that halves the range on a provider error and grows back on success.

**Go packages:** `database/sql`, `golang.org/x/sync/errgroup`, `context`, `log/slog`

---

### 32 — Key Management & Signing Services

**File:** [32-key-management-signing.md](32-key-management-signing.md) · **Prerequisites:** [21](21-sending-transactions.md) · **Examples:** 22

*keystores, KMS/HSM, hot vs cold, a signing service, and nonce management at scale*

**Goals**

- Store and use keys without putting a private key in your source or logs.
- Build a signing service with an authorization boundary.
- Manage nonces for many concurrent senders reliably.
- Reason about hot/warm/cold tiers and blast radius.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The threat model first: what an attacker gets from your process memory, your env, your logs, your backups.
2. Web3 keystore v3: scrypt KDF + AES-128-CTR; `accounts/keystore` in Go; why the passphrase is the weak link.
3. Cloud KMS and HSMs: signing without ever holding the key; secp256k1 support and the recovery-id problem.
4. A signing service in Go: an internal API that takes an unsigned tx and policy, returns a signature — never a key.
5. Policy at the signing boundary: allowlisted destinations, value caps, rate limits, and an audit log.
6. Hot / warm / cold tiers, and matching key value to protection level.
7. Multi-signature and threshold signing (MPC/TSS) at a conceptual level, and Safe (multisig contract) as the on-chain alternative.
8. Nonce management at scale: per-address serialization, a database-backed allocator, gap recovery and stuck-tx sweepers.
9. Many senders: a hot-wallet pool, address rotation, and funding/sweeping flows.
10. Operational hygiene: key rotation, backup and recovery drills, secret scanning in CI, redaction in `slog`.
11. The Go specifics: zeroing memory (and its limits), `crypto/subtle`, avoiding key material in errors.

**Example seeds** — expand to 22, graded easy → hard:

- Create and load a keystore file, then sign a transaction with it.
- A signing HTTP service that refuses a destination outside its allowlist.
- A DB-backed nonce allocator that serializes 20 concurrent send requests for one address.
- Redact a private key from structured logs with a custom `slog` handler.

**Go packages:** `github.com/ethereum/go-ethereum/accounts/keystore`, `log/slog`, `database/sql`, `sync`

---

### 33 — Node Operations & RPC Infrastructure

**File:** [33-node-operations.md](33-node-operations.md) · **Prerequisites:** [20](20-json-rpc-ethclient.md) · **Examples:** 15

*running geth, sync modes, storage, providers, failover and what breaks in production*

**Goals**

- Run and monitor an Ethereum node pair (execution + consensus).
- Choose a sync mode and understand its storage and time cost.
- Decide between self-hosting and a provider, with numbers.
- Build a resilient RPC client layer with failover in Go.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The post-Merge reality: you need two clients, an Engine API connection and a JWT secret.
2. Execution clients: geth, erigon, nethermind, reth — and their very different disk profiles.
3. Sync modes: snap vs full vs archive; what each can answer and what it costs in TB and days.
4. Hardware: NVMe is non-negotiable; IOPS, RAM and the pruning cadence.
5. Configuration that matters: `--http.api`, `--ws`, `--txlookuplimit`, `--gcmode`, cache sizing.
6. Monitoring a node: peer count, sync progress, disk growth, and the metrics endpoint into Prometheus.
7. Providers: Alchemy/Infura/QuickNode class services — rate limits, compute units, archive access, and their failure modes.
8. Build vs buy: a cost/reliability comparison you can actually defend.
9. A resilient RPC layer in Go: multiple upstreams, health checks, hedged/failover requests, and per-method routing (archive vs full).
10. Consistency traps across providers: differing `latest`, missing logs, stale reads — and read-your-writes strategies.
11. Local development: `anvil` and geth `--dev` for the fast loop.

**Example seeds** — expand to 15, graded easy → hard:

- Start geth in dev mode and query it from Go.
- A failover `ethclient` wrapper that tries N upstreams with health tracking.
- Hedged request: fire two providers, take the first good answer, cancel the loser.
- Detect a stale provider by comparing head block numbers across upstreams.

**Go packages:** `github.com/ethereum/go-ethereum/ethclient`, `context`, `sync`, `time`

---

### 34 — Testing Blockchain Code in Go

**File:** [34-testing-blockchain-go.md](34-testing-blockchain-go.md) · **Prerequisites:** [24](24-abigen-bindings.md), [31](31-blockchain-indexer.md) · **Examples:** 22

*simulated backends, anvil forks, deterministic fixtures, fakes for RPC, and reorg tests*

**Goals**

- Test contract interaction with no node, using a simulated backend.
- Fork mainnet locally and test against real state.
- Fake the RPC layer behind an interface so tests are fast and deterministic.
- Write a test that reproduces a reorg.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The test pyramid for blockchain code: pure logic → simulated chain → forked chain → testnet → mainnet.
2. `ethclient/simulated`: an in-process chain, `Commit()` for deterministic block production, and instant mining.
3. Time and blocks as test inputs: `AdjustTime`, mining empty blocks, and testing time-locked logic.
4. Forking with `anvil --fork-url`: real state, impersonation (`anvil_impersonateAccount`), balance/storage cheats.
5. Deterministic fixtures: fixed keys, fixed chain id, golden ABI-encoded blobs, and recorded RPC responses.
6. Faking RPC: define your own narrow interface (`BlockByNumber`, `FilterLogs`) instead of depending on `*ethclient.Client`.
7. A fake chain source that emits a scripted reorg — how to test lesson 31's rollback path.
8. Table-driven tests for encoders/decoders; fuzzing ABI and RLP decoders with `testing/F`.
9. Property tests for invariants: value conservation, nonce monotonicity, balance non-negativity.
10. Foundry tests as a complement (`forge test`, invariant tests) and where Go tests are the better tool.
11. CI: what can run without network (most of it), and how to gate the rest.

**Example seeds** — expand to 22, graded easy → hard:

- Deploy and exercise a contract on a simulated backend inside `go test`.
- Fake `FilterLogs` to feed an indexer 3 blocks then a reorg, and assert final DB state.
- Fuzz an ABI decoder and fix the panic it finds.
- Fork mainnet with anvil, impersonate a whale, and transfer USDC in a test.

**Go packages:** `testing`, `github.com/ethereum/go-ethereum/ethclient/simulated`, `os/exec`

---

### 35 — Observability & Reliability for Chain Services

**File:** [35-observability-reliability.md](35-observability-reliability.md) · **Prerequisites:** [31](31-blockchain-indexer.md), [33](33-node-operations.md) · **Examples:** 15

*metrics, tracing, alerting on lag and reorgs, retries, and designing for RPC that lies*

**Goals**

- Instrument a chain service with the metrics that matter.
- Alert on the failure modes unique to blockchain: lag, reorgs, stuck nonces, gas spikes.
- Implement retry, backoff, circuit breaking and failover around RPC.
- Design for at-least-once delivery and eventual consistency.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. What is different here: your data source is adversarial, eventually consistent, rate-limited and occasionally wrong.
2. The core metrics: head lag (blocks and seconds), ingest rate, reorg depth/frequency, RPC latency/error rate by method and upstream, pending-tx age, wallet balance.
3. Structured logging with `log/slog`: correlating by block number, tx hash and request id; redacting secrets.
4. Tracing a transaction's life across services with OpenTelemetry.
5. Alerts that are actually actionable: lag > N blocks, no new block in X seconds, reorg deeper than the confirmation lag, hot-wallet balance below threshold, nonce gap detected.
6. Retries done right: idempotency first, then exponential backoff with jitter and a context deadline.
7. Circuit breakers and load shedding around providers; fail-open vs fail-closed decisions per endpoint.
8. Reconciliation jobs: periodically re-derive state from the chain and diff against your database.
9. Runbooks: stuck transaction, provider outage, deep reorg, mempool spike, chain halt.
10. Cost observability: RPC call counts per feature, and gas spend per operation.
11. Graceful shutdown: never lose an in-flight signed transaction.

**Example seeds** — expand to 15, graded easy → hard:

- Export head-lag and reorg-depth Prometheus metrics from an ingest loop.
- Retry with exponential backoff + jitter honouring a context deadline.
- A circuit breaker around an RPC upstream that opens after N failures and half-opens on a timer.
- A reconciliation job that diffs indexed ERC-20 balances against on-chain `balanceOf`.

**Go packages:** `github.com/prometheus/client_golang`, `log/slog`, `context`, `time`

---

## Part 8 — Beyond Ethereum

The rest of the landscape, each through a Go lens: Bitcoin's script and Taproot, Cosmos app-chains (written in Go), Solana's account model, zero-knowledge proofs with `gnark`, and DeFi/MEV.

| # | Lesson | Examples |
|---|---|---|
| 36 | [Bitcoin Deep Dive with Go](#36--bitcoin-deep-dive-with-go) | 22 |
| 37 | [Cosmos SDK & CometBFT: App-Chains in Go](#37--cosmos-sdk--cometbft-app-chains-in-go) | 15 |
| 38 | [Solana & Other Execution Models](#38--solana--other-execution-models) | 15 |
| 39 | [Zero-Knowledge Proofs with `gnark`](#39--zero-knowledge-proofs-with-gnark) | 15 |
| 40 | [DeFi Primitives & MEV from a Go Integrator's View](#40--defi-primitives--mev-from-a-go-integrators-view) | 22 |

### 36 — Bitcoin Deep Dive with Go

**File:** [36-bitcoin-deep-dive.md](36-bitcoin-deep-dive.md) · **Prerequisites:** [10](10-transactions-utxo.md), [11](11-wallets-mempool.md) · **Examples:** 22

*Script, P2PKH/P2SH/SegWit/Taproot, sighash types, PSBT and the `btcsuite` libraries*

**Goals**

- Read and evaluate Bitcoin Script.
- Distinguish the output types and build each in Go.
- Explain SegWit's txid/wtxid split and Taproot's key/script paths.
- Build, sign and inspect transactions with `btcd`/`btcutil`.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Bitcoin as the reference implementation of everything in Part 3 — now with the real details.
2. Script: a stack language, no loops, and the locking/unlocking-script pairing.
3. Standard output types: P2PK, P2PKH, P2SH, P2WPKH, P2WSH, P2TR — and their address encodings.
4. Sighash types (ALL/NONE/SINGLE/ANYONECANPAY) and what each commits to.
5. Transaction malleability, SegWit's fix, and the txid vs wtxid distinction.
6. Weight units and vbytes: how fees are really computed post-SegWit.
7. Taproot: Schnorr signatures (BIP-340), key-path vs script-path spending, MAST and the privacy win.
8. Timelocks: `nLockTime`, `nSequence`, CLTV/CSV — the primitives behind Lightning.
9. PSBT (BIP-174): the multi-party signing workflow and why hardware wallets need it.
10. The Go ecosystem: `btcd`, `btcutil`, `txscript`, `chaincfg` — and `lnd` as production-grade reference code.
11. Running `bitcoind -regtest` and driving it from Go over RPC.

**Example seeds** — expand to 22, graded easy → hard:

- Evaluate a P2PKH script step by step with the stack printed.
- Build a P2WPKH address from a public key with `btcutil` and verify against a test vector.
- Construct, sign and serialize a regtest transaction with `txscript`.
- Compute a Taproot key-path output key from an internal key and a Merkle root.

**Go packages:** `github.com/btcsuite/btcd`, `btcutil`, `txscript`, `chaincfg`

---

### 37 — Cosmos SDK & CometBFT: App-Chains in Go

**File:** [37-cosmos-tendermint.md](37-cosmos-tendermint.md) · **Prerequisites:** [29](29-alternative-consensus.md) · **Examples:** 15

*building your own chain in Go — modules, keepers, ABCI, staking and IBC*

**Goals**

- Explain the ABCI boundary between the app and the consensus engine.
- Scaffold a Cosmos SDK chain and add a custom module.
- Describe keepers, the multistore and the state-machine lifecycle.
- Explain IBC's light-client model at a working level.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Why app-chains: sovereignty, custom fees, and no gas competition with strangers.
2. The stack: CometBFT (consensus + networking) ⟷ ABCI ⟷ your Go application.
3. ABCI methods: InitChain, PrepareProposal, ProcessProposal, FinalizeBlock, Commit — the modern (0.38+) flow.
4. The Cosmos SDK: modules, keepers, `Msg` services, and protobuf-defined state.
5. The multistore and IAVL: versioned, Merkle-ized state with proofs.
6. Built-in modules: auth, bank, staking, gov, distribution — what you get for free.
7. Writing a custom module: proto definitions → keeper → msg server → CLI/gRPC → tests.
8. Transactions in Cosmos: `Msg`s, `AccountKeeper` sequences, fee grants and ante handlers.
9. IBC: light clients, connections, channels, packets and timeouts — trust-minimized bridging.
10. Upgrades: on-chain governance, upgrade handlers, and store migrations.
11. Why this matters for a Go developer: the whole stack is idiomatic Go you can read and extend.

**Example seeds** — expand to 15, graded easy → hard:

- Implement a minimal ABCI application in Go that counts transactions.
- Add a `Msg` handler and keeper method for a custom `Post` type.
- Query state through gRPC and the CLI.
- Write a keeper unit test with the SDK's test fixtures.

**Go packages:** `github.com/cosmos/cosmos-sdk`, `github.com/cometbft/cometbft`, `google.golang.org/protobuf`

---

### 38 — Solana & Other Execution Models

**File:** [38-solana-other-vms.md](38-solana-other-vms.md) · **Prerequisites:** [15](15-account-model-state.md) · **Examples:** 15

*the accounts model, parallel execution, and talking to non-EVM chains from Go*

**Goals**

- Explain Solana's account model and why it enables parallel execution.
- Compare SVM, Move and WASM-based execution with the EVM.
- Interact with a Solana cluster from Go.
- Choose sensibly when a project asks 'which chain?'

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Beyond the EVM: the design space of execution environments.
2. Solana's model: programs are stateless, accounts hold state, and every transaction declares what it touches.
3. Why declared access lists enable Sealevel's parallel execution — and what that costs developers.
4. Proof of History as a clock, not a consensus mechanism; Tower BFT on top.
5. Rent, account sizing, PDAs (program-derived addresses) and CPI (cross-program invocation).
6. ed25519 signatures and base58 addresses — different primitives from lesson 6/7.
7. Go clients: `gagliardetto/solana-go` — RPC, transaction building, and Anchor IDL decoding.
8. Move (Aptos/Sui): resources as linear types, and what that prevents by construction.
9. WASM chains (CosmWasm, NEAR, Polkadot): a different bytecode, the same problems.
10. The comparison table: throughput, fees, finality, tooling maturity, decentralization.
11. Multi-chain services in Go: a chain-agnostic interface with per-chain adapters.

**Example seeds** — expand to 15, graded easy → hard:

- Connect to a Solana devnet cluster and read an account balance from Go.
- Build, sign and send a SOL transfer with `solana-go`.
- Decode an SPL token account's data layout by hand.
- Sketch a `ChainClient` interface with EVM and Solana implementations.

**Go packages:** `github.com/gagliardetto/solana-go`, `crypto/ed25519`

---

### 39 — Zero-Knowledge Proofs with `gnark`

**File:** [39-zero-knowledge-proofs.md](39-zero-knowledge-proofs.md) · **Prerequisites:** [05](05-merkle-trees.md), [30](30-layer2-scaling.md) · **Examples:** 15

*commitments, circuits, SNARKs vs STARKs, and writing and proving a circuit in Go*

**Goals**

- Explain what a ZK proof proves, and what it does not.
- Describe the SNARK pipeline: circuit → constraint system → setup → prove → verify.
- Write, compile and prove a circuit in Go with `gnark`.
- Judge where ZK is genuinely the right tool.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The three properties: completeness, soundness, zero-knowledge — with a non-mathematical intuition.
2. Interactive vs non-interactive proofs and the Fiat–Shamir transform.
3. Arithmetic circuits and R1CS: turning a computation into constraints.
4. The proving systems: Groth16 (tiny proofs, per-circuit trusted setup), PLONK (universal setup), STARKs (no setup, hash-based, post-quantum).
5. Trusted setup ceremonies and what a compromise would mean.
6. Commitments as the foundation: Pedersen and KZG — and where KZG shows up in EIP-4844.
7. `gnark` in Go: defining a circuit struct, `Define`, witness assignment, `frontend.Compile`, prove and verify.
8. On-chain verification: exporting a Solidity verifier and the gas cost of verification.
9. Applications: zk-rollups, private transfers (Tornado/Zcash), identity and proof-of-membership, zkML hype vs reality.
10. Costs: proving time, memory, circuit-writing difficulty — the honest engineering picture.
11. Recursion and aggregation in one paragraph, and why they matter for rollups.

**Example seeds** — expand to 15, graded easy → hard:

- A `gnark` circuit proving you know `x` with `x³ + x + 5 == y`; prove and verify in Go.
- A Merkle-membership circuit proving inclusion without revealing the leaf.
- Measure proving time and proof size as constraint count grows.
- Export the Solidity verifier and estimate its on-chain verification gas.

**Go packages:** `github.com/consensys/gnark`, `gnark-crypto`

---

### 40 — DeFi Primitives & MEV from a Go Integrator's View

**File:** [40-defi-primitives-mev.md](40-defi-primitives-mev.md) · **Prerequisites:** [26](26-erc-standards.md), [27](27-contract-security.md) · **Examples:** 22

*AMMs, lending, oracles, liquidations, MEV and flash loans — the math and the integration code*

**Goals**

- Compute AMM swap outputs and price impact in Go with exact integer math.
- Explain lending health factors and liquidation mechanics.
- Describe MEV: sandwiches, arbitrage, liquidations, and PBS.
- Integrate safely: slippage, deadlines, simulation and private transactions.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. The primitive stack: swap, lend, stake, derive — nearly everything is a composition of these.
2. Constant-product AMMs: `x·y=k`, the swap formula with fees, price impact, and integer math in Go with `big.Int`.
3. Slippage and deadlines: `amountOutMin` as your only real protection, computed off-chain.
4. Concentrated liquidity (Uniswap v3): ticks, `sqrtPriceX96`, and the Q64.96 fixed-point math you must get right.
5. Impermanent loss explained with numbers, not vibes.
6. Lending protocols: collateral factors, health factor, interest-rate curves, and the liquidation trigger.
7. Oracles: Chainlink feeds, TWAPs, staleness checks, and reading a feed from Go.
8. MEV: the taxonomy (arbitrage, sandwich, liquidation, JIT), the searcher→builder→proposer pipeline, and Flashbots.
9. Flash loans: atomicity as a primitive, and the attack pattern it enables.
10. Defensive integration in Go: simulate with `eth_call` before sending, bound approvals, use private RPC endpoints, verify receipts and emitted events rather than assuming success.
11. Stablecoins, LSTs and yield sources — what your service must model when it holds them.

**Example seeds** — expand to 22, graded easy → hard:

- Compute a Uniswap v2 swap output with fees and compare against an on-chain quote.
- Compute price impact and set `amountOutMin` for a 0.5% slippage tolerance.
- Read a Chainlink price feed with a staleness check from Go.
- Simulate a swap with `eth_call` before broadcasting and abort if the output moved.

**Go packages:** `math/big`, `github.com/ethereum/go-ethereum/accounts/abi/bind`

---

## Part 9 — Capstone

One end-to-end system that uses the whole course.

| # | Lesson | Examples |
|---|---|---|
| 41 | [Capstone Project](#41--capstone-project) | — |

### 41 — Capstone Project

**File:** [41-capstone.md](41-capstone.md) · **Prerequisites:** [31](31-blockchain-indexer.md), [32](32-key-management-signing.md), [34](34-testing-blockchain-go.md), [35](35-observability-reliability.md) · **Examples:** — (project)

*one end-to-end system: indexer + API + wallet service + contract integration, in Go*

**Goals**

- Ship one production-shaped blockchain service in Go.
- Combine indexing, signing, contract interaction and observability.
- Handle reorgs, retries and restarts correctly under test.
- Document and deploy it.

**Topics to cover** — one `###` sub-section each in `## Concepts`:

1. Pick one: (a) an ERC-20 payment gateway with deposit detection, sweeps and withdrawals; (b) an NFT marketplace indexer with a REST/GraphQL API; (c) a DAO/treasury dashboard; (d) your own L1 from Part 3, finished and networked.
2. Required components: an ingestion pipeline (31), a signing boundary (32), contract bindings (24/26), a query API, and metrics/alerts (35).
3. Non-functional requirements: restart-safe, reorg-safe, idempotent, rate-limit-aware, secret-safe.
4. Architecture: hexagonal boundaries so the chain is an adapter, not a dependency of your domain.
5. The test plan: unit + simulated backend + forked-chain integration + a scripted reorg test.
6. Deployment: Docker, config via env, health/readiness endpoints, graceful shutdown.
7. A written README that explains the trust assumptions and failure modes — the thing that proves you understood.
8. A security review pass over your own code using lesson 27 and 32 as the checklist.

**Example seeds** — expand to the project, graded easy → hard:

- No example tiers — this is a build. Deliverables live under `projects/`.

**Go packages:** `everything`

---

## Cheatsheets (after the lessons)

Once a group of lessons is written, compress it into one dense reference sheet in [cheatsheets/](cheatsheets/) — API surface, shapes and traps, no prose. Planned groupings:

| Sheet | Lessons | What it holds |
|---|---|---|
| `01-03.md` | 01-03 | Foundations & tooling |
| `04-07.md` | 04-07 | Crypto primitives |
| `08-12.md` | 08-12 | Blocks, PoW, UTXO, storage |
| `13-15.md` | 13-15 | P2P, forks, state |
| `16-19.md` | 16-19 | Ethereum, RLP/MPT, transactions |
| `18.md` | 18 | EVM opcodes & gas |
| `20-21.md` | 20-21 | JSON-RPC & sending transactions |
| `22-26.md` | 22-26 | Solidity, ABI, abigen, logs, ERCs |
| `27.md` | 27 | Security checklist |
| `28-30.md` | 28-30 | Consensus & L2 |
| `31-35.md` | 31-35 | Production: indexing, keys, ops, testing |
| `36-40.md` | 36-40 | Bitcoin, Cosmos, Solana, ZK, DeFi |

