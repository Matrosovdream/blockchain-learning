# Part 1 — Foundations

What a blockchain actually is, the tools you need, and the byte-level primitives every later lesson assumes. No cryptography yet — just the mental model and the keyboard setup.

**Core spine.** Lessons 01–03 · 42 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 01 | [Introduction to Blockchain](#01-introduction-to-blockchain) | — | 12 |
| 02 | [Environment Setup & Tooling](#02-environment-setup-tooling) | 01 | 12 |
| 03 | [Bytes, Hex, Big Integers & Encoding](#03-bytes-hex-big-integers-encoding) | 02 | 18 |

---

## 01 — Introduction to Blockchain

**Lesson file:** [../01-introduction.md](../01-introduction.md) · **Examples folder:** `../examples/01-introduction/`

| | |
|---|---|
| Prerequisites | none — start here |
| Unlocks | 02 |
| Examples | **12** — 🟢 5 easy, 🟡 4 medium, 🔴 3 hard |
| Topics | 9 |

*what problem a blockchain solves, the ledger, decentralization, the landscape — and what it is *not**

### Goals

- Explain what a blockchain is without using the word 'blockchain'.
- Name the problem it solves — double-spend without a trusted third party — and say why it is hard.
- Place Bitcoin, Ethereum, L2s and app-chains on one map.
- Decide honestly when a database is the better answer.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The double-spend problem**

   - Why a digital coin is trivially copyable and physical cash is not.
   - The classic fix: one trusted server keeps the balances — and everything that implies.
   - What 'without a trusted third party' actually costs: latency, throughput and energy.
   - Why the hard part is *ordering* two conflicting spends, not storing them.

**2. A ledger as an append-only log**

   - State as a fold over an ordered list of transactions.
   - Replicated state machines: same log + same rules ⇒ same state, on every node.
   - Why determinism is mandatory (no clocks, no randomness, no floats in consensus code).
   - The log is the source of truth; balances are a derived index.

**3. Decentralization is a spectrum**

   - Permissionless (Bitcoin, Ethereum) vs permissioned (Fabric, Corda) vs 'a database with signatures'.
   - The axes that matter: who can read, who can write, who can validate, who can change the rules.
   - Node types: full, archive, light, validator — and who actually verifies anything.
   - Client diversity and why a single implementation is a systemic risk.

**4. The three ideas chained together**

   - Hash linking gives integrity — tamper with block 5 and blocks 6..N break.
   - Digital signatures give authorization — only the key holder can move funds.
   - Consensus gives ordering — everyone agrees which conflicting spend came first.
   - None of the three is new; the combination plus Sybil resistance is.

**5. Byzantine fault tolerance and Sybil resistance**

   - The Byzantine Generals framing, in plain language.
   - Why 'one vote per node' fails when identities are free (the Sybil attack).
   - Proof of work as costly-to-fake identity; proof of stake as bonded identity.
   - This — not the Merkle tree — is the actual 2009 innovation.

**6. History in one page**

   - Chaum's DigiCash, Hashcash, b-money, bit gold — the failed and partial precursors.
   - Bitcoin 2009: the whitepaper's contribution in one sentence.
   - Ethereum 2015: programmability, and what that opened and broke.
   - The Merge (Sept 2022), EIP-1559 (2021), EIP-4844 blobs (2024) — the upgrades that matter to you.

**7. The landscape map**

   - L1s, L2 rollups, sidechains, app-chains, validiums — what distinguishes each.
   - Where value and activity actually sit today, and how fast it moves.
   - Where Go sits: geth, prysm, btcd, lnd, Cosmos SDK, most indexers and infrastructure.
   - The languages you will meet next to Go: Solidity, Rust, and why.

**8. Vocabulary you will meet constantly**

   - node, client, block, transaction, mempool, gas, nonce, fork, finality, validator.
   - EOA vs contract account; on-chain vs off-chain; L1 vs L2.
   - The units: wei/gwei/ether, satoshi/BTC.
   - Words the industry uses badly ('smart contract', 'wallet', 'token') and what they really mean.

**9. When *not* to use a blockchain**

   - The honest comparison table vs Postgres: throughput, latency, cost, privacy, reversibility.
   - The question that settles it: who are the mutually distrusting parties?
   - Irreversibility as a feature and as a liability.
   - Cases where the answer is genuinely yes: bearer assets, cross-org settlement, censorship resistance.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Confusing 'immutable' with 'correct' — a chain faithfully records a bad transaction forever.
- Assuming decentralization; most systems have a sequencer, a multisig or an upgrade key.
- Believing throughput numbers without asking about finality and validator count.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 12).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Model a ledger as a slice of transfers and fold it into balances.
- Replay the same transfer twice against a naive ledger and print the double-spend.
- Hash-link three records and show that editing record 1 breaks the chain.

**🟡 Medium — 4 examples** *(concepts combined, and the traps)*

- A `map[string]int64` state machine applying an ordered op list — two orders, two final states.
- Reject a transfer that overdraws, and show why the check must happen at apply time, not submit time.
- Count how many messages N nodes exchange to agree, as N grows.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A tiny 'trusted server' ledger vs a 'replicated' one, both in one program, printing where they diverge under a partition.
- Simulate 5 nodes receiving two conflicting transfers in different orders and show that without a rule they never converge.

### Packages & tools

`fmt`, `slices`, `crypto/sha256`

### Resources to cite

- Bitcoin whitepaper — https://bitcoin.org/bitcoin.pdf
- Ethereum docs — Intro to Ethereum: https://ethereum.org/en/developers/docs/intro-to-ethereum/
- Ethereum docs — Intro to Ether: https://ethereum.org/en/developers/docs/intro-to-ether/
- Why decentralization matters — https://ethereum.org/en/developers/docs/networks/

---

## 02 — Environment Setup & Tooling

**Lesson file:** [../02-environment-setup.md](../02-environment-setup.md) · **Examples folder:** `../examples/02-environment-setup/`

| | |
|---|---|
| Prerequisites | [01](../01-introduction.md) |
| Unlocks | 03 |
| Examples | **12** — 🟢 5 easy, 🟡 4 medium, 🔴 3 hard |
| Topics | 8 |

*Go toolchain, the module layout for this repo, node clients, testnets, faucets, and the CLI toolbox*

### Goals

- Have a Go module ready to run every example in this repo.
- Install and sanity-check a chain to talk to: `anvil`, local geth, or a hosted RPC.
- Get testnet funds and read your own transaction on an explorer.
- Establish secret hygiene before you ever hold a key that matters.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Go setup for this repo**

   - `go version` (1.22+), `GOPATH`/`GOMODCACHE`, and why this repo pins nothing globally.
   - `practice/` as one module: `go mod init github.com/Matrosovdream/blockchain-learning/practice`.
   - One subfolder per lesson, each its own `package main`; `go run ./04-hash-functions`.
   - The scratch loop for examples: `/tmp/bc-ex` + `go mod init scratch` + paste + `go run .`.

**2. The Go blockchain toolbox**

   - `github.com/ethereum/go-ethereum` used as a *library*: `crypto`, `rlp`, `accounts/abi`, `core/types`, `ethclient`.
   - `github.com/holiman/uint256` — the EVM-native 256-bit integer, allocation-free.
   - `github.com/btcsuite/btcd` for Bitcoin; `golang.org/x/crypto` for sha3/scrypt/ripemd160.
   - Why go-ethereum is a large dependency and how to keep build times sane.

**3. Choosing a chain to talk to**

   - `anvil` (Foundry): instant, funded accounts, forkable — the default for this course.
   - `geth --dev`: a real client in dev mode, useful for lessons 33 and 62.
   - Hosted RPC (Alchemy/Infura/QuickNode): free tiers, rate limits, archive access.
   - A decision table: what each option can and cannot answer.

**4. The Solidity toolchain (yes, you need it)**

   - Foundry: `forge build`, `forge test`, `forge create`, `cast call`, `cast send`, `cast 4byte`.
   - `solc` directly, and pinning the compiler version (bytecode differs between versions).
   - `cast` as the fastest way to check what your Go code *should* be producing.
   - Installing via `foundryup`, and verifying with `forge --version`.

**5. Testnets, faucets and explorers**

   - Sepolia and Hoodi today; why testnets get deprecated and how to check which is current.
   - Faucets, their rate limits, and why you never learn on mainnet.
   - Etherscan as a debugging tool: reading a tx, its receipt, its logs, its internal calls.
   - Verifying a contract's source, and why unverified bytecode is a red flag.

**6. Secret hygiene from day one**

   - `.env` + `os.Getenv`, fail-fast on missing config, and never a default private key in code.
   - The `.gitignore` rules in this repo: `.env*`, `*.key`, `keystore/`, `mnemonic.txt`.
   - Test keys must be labelled as test keys, in a comment, every time.
   - Secret scanning: `gitleaks`/`trufflehog` in CI before you have anything to lose.

**7. The CLI toolbox**

   - `cast`, `jq`, `curl` for raw JSON-RPC, `websocat` for subscriptions.
   - `xxd`/`hexdump` for looking at raw bytes, which you will do constantly.
   - A `Makefile` for the repetitive commands.
   - Editor setup: gopls, and a Solidity extension for reading contracts.

**8. Your first verification script**

   - Dial an RPC endpoint, print chain id, latest block number, base fee.
   - Handle the three failure modes: bad URL, wrong network, rate-limited.
   - Add a `context.WithTimeout` — every RPC call in this course has a deadline.
   - Commit it as `practice/02-environment-setup/main.go`; you will reuse it all course.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Committing a `.env` or a keystore — the single most common way people lose testnet *and* mainnet funds.
- Testing against mainnet 'just to read' and getting rate-limited mid-lesson.
- Mixing chain ids: a tx signed for one chain is invalid on another (that is the point — see lesson 19).

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 12).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Print Go version, module path and the go-ethereum version you resolved.
- Read `RPC_URL` from the environment with a clear fatal error when it is missing.
- Dial a node and print chain id + latest block number.

**🟡 Medium — 4 examples** *(concepts combined, and the traps)*

- Print latest block number, base fee and gas price with a 5-second context deadline.
- Start `anvil`, list its 10 funded dev accounts and print account 0's balance in ether.
- Hit the same node over raw JSON-RPC with `net/http` and compare against `ethclient`.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A small `chainenv` package that loads config, dials, verifies the chain id matches expectations, and returns a ready client.
- Detect whether the endpoint is a full or archive node by querying a very old block and interpreting the error.

### Packages & tools

`os`, `context`, `log/slog`, `net/http`, `github.com/ethereum/go-ethereum/ethclient`

### Resources to cite

- Go — Getting started: https://go.dev/doc/tutorial/getting-started
- Foundry Book: https://book.getfoundry.sh/
- go-ethereum docs: https://geth.ethereum.org/docs
- Ethereum networks & testnets: https://ethereum.org/en/developers/docs/networks/

---

## 03 — Bytes, Hex, Big Integers & Encoding

**Lesson file:** [../03-bytes-encoding.md](../03-bytes-encoding.md) · **Examples folder:** `../examples/03-bytes-encoding/`

| | |
|---|---|
| Prerequisites | [02](../02-environment-setup.md) |
| Unlocks | 04 |
| Examples | **18** — 🟢 6 easy, 🟡 7 medium, 🔴 5 hard |
| Topics | 9 |

*the data primitives — `[]byte`, endianness, hex, base58, bech32, `big.Int`, fixed-size arrays*

### Goals

- Move fluently between `[]byte`, hex strings and integers in Go.
- Explain endianness and pick the right one when serializing.
- Use `math/big`/`uint256` for values that overflow `uint64` — which is most of them.
- Know why `[32]byte` and `[]byte` are different types and where each is used.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Everything is bytes**

   - `[]byte` (slice, nil-able, variable) vs `[32]byte`/`[20]byte` (array, comparable, map-key-able).
   - Why go-ethereum defines `common.Hash` (`[32]byte`) and `common.Address` (`[20]byte`) as arrays.
   - Copying semantics: arrays copy, slices alias — the bug that corrupts a hash you thought you saved.
   - `bytes.Equal`, `bytes.Compare`, `copy`, and preallocating with `make([]byte, n)`.

**2. Hex encoding**

   - `encoding/hex` — `EncodeToString`, `DecodeString`, and the odd-length error.
   - The `0x` prefix convention, and why JSON-RPC always uses it.
   - `common.Hex2Bytes` vs `hexutil.Decode` — the second validates the prefix, the first does not.
   - `hexutil` quantity vs data rules: quantities have no leading zeros, data is byte-aligned.

**3. Endianness**

   - Big-endian on the wire for Ethereum; little-endian inside Bitcoin's serialization.
   - The classic confusion: a Bitcoin txid displayed reversed from how it is hashed.
   - `encoding/binary.BigEndian`/`LittleEndian`, `PutUint32/64`, and reading them back.
   - A rule of thumb: if a hex string 'looks backwards', you have an endianness bug.

**4. Big integers**

   - Why 256 bits: 1 ETH = 10^18 wei, and totals overflow `uint64` immediately.
   - `math/big`: `NewInt`, `SetString` (with base!), `Add`/`Mul`/`Div`, `Cmp`, `Text`.
   - The aliasing trap: `z.Add(x, y)` writes into `z`; never share a `*big.Int` across goroutines.
   - `big.Rat` for exact display math, and never `float64` anywhere near money.

**5. uint256**

   - `github.com/holiman/uint256`: fixed 4×uint64, no allocation, EVM-exact wrapping.
   - When to prefer it (hot loops, EVM interpreters) vs `big.Int` (APIs, readability).
   - Conversion helpers and the overflow-flag API.
   - Why the EVM wraps on overflow and your Go code usually must not.

**6. Fixed-width serialization**

   - Left-padding to 32 bytes — the EVM's universal word size.
   - `common.LeftPadBytes`/`RightPadBytes`, and rolling your own.
   - `binary.Write` with a struct, and why you should not (padding, field order, tags).
   - Deterministic encoding as a hard requirement before hashing (lesson 04).

**7. Address and hash encodings**

   - Base64 (rare on-chain), base58 (Bitcoin/Solana), base58check (version + checksum), bech32/bech32m (SegWit/Taproot).
   - Why base58 drops `0`, `O`, `I`, `l` — human transcription errors.
   - Checksums as a UX feature: base58check, EIP-55 (lesson 07), bech32's BCH code.
   - Implementing base58 encode/decode with `big.Int` division.

**8. Units and money formatting**

   - wei / gwei / ether; satoshi / BTC; token decimals are *per token* and not always 18.
   - Formatting for humans with `big.Rat` or integer string surgery — never floats.
   - Parsing user input safely: reject more decimals than the token supports.
   - A `Wei` named type so the compiler stops you mixing units.

**9. Constant-time comparison**

   - `crypto/subtle.ConstantTimeCompare` for secrets, MACs and tokens.
   - Why `bytes.Equal` leaks length and first-difference position via timing.
   - Where it matters in this course: API keys, HMAC webhooks (lesson 51), signature checks.
   - It does *not* matter for public data like block hashes.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- `float64` anywhere in a money path — precision loss that silently mints or burns value.
- Reusing a `*big.Int` receiver and mutating a value someone else holds.
- Forgetting `SetString`'s base argument and parsing hex as decimal.
- Treating a `[]byte` returned by a library as owned — copy it before storing.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 6 examples** *(one concept in isolation)*

- Hex round-trip a string, including the odd-length error case.
- Convert 1.5 ETH to wei with `big.Int` and format it back to a decimal string.
- Show `[32]byte` copy vs `[]byte` alias with a mutation.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Left-pad a `uint64` into a 32-byte EVM word and print it.
- Reverse a Bitcoin txid between internal and display order.
- Format a raw USDC balance (6 decimals) into a human string with `big.Rat`.
- Demonstrate the `big.Int` aliasing trap and then fix it.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Implement base58 encode/decode and round-trip a Bitcoin address payload.
- Implement base58check with a double-SHA256 checksum and reject a corrupted address.
- Benchmark `big.Int` vs `uint256` for a million additions and report allocations.

### Packages & tools

`encoding/hex`, `encoding/binary`, `math/big`, `crypto/subtle`, `github.com/holiman/uint256`, `github.com/ethereum/go-ethereum/common`

### Resources to cite

- Go — package math/big: https://pkg.go.dev/math/big
- Go — package encoding/binary: https://pkg.go.dev/encoding/binary
- hexutil (go-ethereum): https://pkg.go.dev/github.com/ethereum/go-ethereum/common/hexutil
- BIP-173 bech32: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
