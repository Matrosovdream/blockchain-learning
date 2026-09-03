# Part 3 — Build a Blockchain from Scratch (Go)

The heart of the course. One Go program grows across eight lessons into a working chain: blocks, mining, UTXO transactions, a wallet, persistence, a P2P network, fork choice — then the account model.

**Core spine.** Lessons 08–15 · 144 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 08 | [Blocks & the Chain](#08-blocks-the-chain) | 04, 05 | 18 |
| 09 | [Proof of Work & Mining](#09-proof-of-work-mining) | 08 | 18 |
| 10 | [Transactions & the UTXO Model](#10-transactions-the-utxo-model) | 06, 08 | 18 |
| 11 | [Wallets, Fees & the Mempool](#11-wallets-fees-the-mempool) | 07, 10 | 18 |
| 12 | [Persistence & Chain State](#12-persistence-chain-state) | 10 | 18 |
| 13 | [P2P Networking & Gossip](#13-p2p-networking-gossip) | 12 | 18 |
| 14 | [Consensus, Forks & Reorgs](#14-consensus-forks-reorgs) | 13 | 18 |
| 15 | [The Account Model & World State](#15-the-account-model-world-state) | 10, 14 | 18 |

---

## 08 — Blocks & the Chain

**Lesson file:** [../08-blocks-and-chain.md](../08-blocks-and-chain.md) · **Examples folder:** `../examples/08-blocks-and-chain/`

| | |
|---|---|
| Prerequisites | [04](../04-hash-functions.md), [05](../05-merkle-trees.md) |
| Unlocks | 09, 10 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 8 |

*the block struct, hash linking, genesis, deterministic serialization and chain validation*

### Goals

- Define a block and a header in Go and hash it deterministically.
- Link blocks by previous-hash and detect any tampering.
- Create a genesis block and validate a whole chain.
- Separate header from body and know why that split exists.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Header vs body**

   - The header is what gets hashed and gossiped; the body is fetched on demand.
   - Light clients sync headers only — 80 bytes per Bitcoin block vs megabytes of body.
   - The body is bound to the header by the Merkle root, so it cannot be swapped.
   - This split drives the whole P2P design in lesson 13 (headers-first sync).

**2. A minimal header**

   - Fields: version, prevHash [32]byte, merkleRoot [32]byte, timestamp int64, bits uint32, nonce uint32, height uint64.
   - Why height is convenient but not authoritative (accumulated work is — lesson 14).
   - Go modelling: a `Header` struct with exported fields, a `Block` holding `Header` + `[]*Transaction`.
   - A cached `hash` field with a `sync.Once` or explicit invalidation — and the aliasing bug it causes.

**3. Deterministic serialization**

   - Fixed field order, fixed widths, big-endian: `binary.Write` into a `bytes.Buffer`, or manual.
   - Never JSON, never `gob`, never a map — hashing must be byte-identical across machines and versions.
   - Round-trip tests: encode → decode → encode, assert byte equality.
   - Versioning: a leading version byte so you can change the format later without a chain split.

**4. Hash linking**

   - Each header embeds the previous header's hash — a linked list built from content, not pointers.
   - Tamper-evidence: change block 5 and every subsequent hash is wrong.
   - Demonstrate it: mutate a transaction, recompute, print which blocks now fail.
   - This gives integrity only — it says nothing about *which* chain is correct.

**5. The genesis block**

   - Hardcoded, prevHash all zeros, no parent to validate against.
   - Every node must agree on it byte-for-byte or they are on different networks.
   - Bitcoin's genesis coinbase message; Ethereum's genesis allocation file.
   - In Go: a package-level `Genesis()` constructor and a test pinning its hash.

**6. Validation rules**

   - Stateless (context-free): structure, hash matches the header, PoW target, timestamp sanity, size cap.
   - Stateful (needs the chain): parent exists, height = parent+1, difficulty correct, no double spends.
   - Why separating them matters: stateless checks can run before you have the parent.
   - Return typed errors (`ErrBadPoW`, `ErrOrphan`) so the network layer can decide what to do.

**7. Timestamps**

   - Not a clock: median-time-past as the lower bound, a few hours of future drift as the upper.
   - Why miners can and do lie a little, and the timewarp attack that exploits it.
   - Using `time.Time` in Go but storing Unix seconds on the wire.
   - Monotonic-clock stripping (`t.Round(0)`) before serializing.

**8. Testing the chain**

   - Table-driven tests over each validation rule with a helper that builds valid blocks.
   - A `chainBuilder` test fixture — you will reuse it for the next seven lessons.
   - Golden tests: pin the genesis hash and one known block hash.
   - Fuzzing the header decoder with `testing.F` (link forward to lesson 34).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Hashing a struct through `gob`/JSON and getting different hashes on different Go versions.
- Caching the block hash and forgetting to invalidate after mutating a field.
- Storing `time.Time` directly — the monotonic clock reading breaks byte-equality.
- Validating height instead of work, then being unable to handle forks in lesson 14.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Define `Header`, serialize it, hash it, print the hex.
- Build a genesis block and pin its hash in a test.
- Chain three blocks and print each prevHash link.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Serialize and deserialize a header, asserting a byte-identical round trip.
- Mutate block 1's transactions and show blocks 2..N fail validation.
- Put a Merkle root over 4 transactions into the header and re-verify it.
- Split validation into stateless and stateful passes with typed errors.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A `chainBuilder` test helper that produces valid N-block chains for later lessons.
- Enforce median-time-past and reject a block with a backdated timestamp.
- Fuzz the header decoder and fix the first panic it finds.

### Packages & tools

`encoding/binary`, `bytes`, `crypto/sha256`, `time`, `sync`, `errors`, `testing`

### Resources to cite

- Bitcoin block chain reference: https://developer.bitcoin.org/reference/block_chain.html
- Ethereum Yellow Paper (block structure): https://ethereum.github.io/yellowpaper/paper.pdf
- Go — encoding/binary: https://pkg.go.dev/encoding/binary

---

## 09 — Proof of Work & Mining

**Lesson file:** [../09-proof-of-work.md](../09-proof-of-work.md) · **Examples folder:** `../examples/09-proof-of-work/`

| | |
|---|---|
| Prerequisites | [08](../08-blocks-and-chain.md) |
| Unlocks | — |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 8 |

*the difficulty target, nonce grinding, compact `bits`, retargeting, and the honest cost discussion*

### Goals

- Implement a PoW miner that finds a nonce below a target.
- Convert between difficulty, target and the compact `bits` encoding.
- Retarget difficulty from observed block times.
- Explain what PoW buys (Sybil resistance, objective ordering) and what it costs.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The puzzle**

   - Find nonce such that H(header) < target. Asymmetric: expensive to find, one hash to verify.
   - Why this is the *only* known way to make identity costly without a registry.
   - The search is memoryless — no partial progress, which is what makes it fair.
   - Verification cost is what lets every node police every miner.

**2. Target, difficulty and bits**

   - target is a 256-bit number; smaller target ⇒ harder. difficulty = max_target / target.
   - Bitcoin's compact `bits`: 1 exponent byte + 3 mantissa bytes, a float-like packing.
   - Expanding bits → target and compressing back, in Go with `big.Int`.
   - The leading-zeros mental model is a teaching aid; the real check is a big-integer comparison.

**3. The mining loop in Go**

   - Reuse one hasher and one serialization buffer; only the nonce bytes change.
   - `uint32` nonce exhaustion (4.3 billion) and the extra-nonce / timestamp-roll workaround.
   - Allocation-free inner loop — benchmark it and watch `ReportAllocs` go to zero.
   - Where the time actually goes: profile before optimising.

**4. Parallel mining**

   - Shard the nonce space across `runtime.NumCPU()` goroutines.
   - `context.WithCancel` so the first winner stops the rest; a buffered result channel to avoid a leak.
   - The goroutine-leak trap: an unbuffered channel with no reader after cancellation.
   - Measuring speedup and showing it is sub-linear (memory bandwidth, thermal).

**5. Difficulty retargeting**

   - Bitcoin: every 2016 blocks, adjust by actual/expected time, clamped to 4× either way.
   - Ethereum's old per-block adjustment and the difficulty bomb.
   - Implementing retarget in Go with integer math only — no floats in consensus code.
   - Why clamping exists: to resist hashrate manipulation across the boundary.

**6. Statistics of block times**

   - Block discovery is a Poisson process; inter-block times are exponentially distributed.
   - Consequence: with a 10-minute target you will regularly see 1-minute and 40-minute gaps.
   - Simulating it in Go and printing the distribution — this surprises everyone.
   - Why confirmation counts, not wall-clock time, are the safety parameter.

**7. What PoW actually provides**

   - Sybil resistance and an objectively verifiable ordering — not 'security' in the abstract.
   - Nothing about transaction validity: every node still checks every rule.
   - The security budget = block reward + fees, and what happens as the subsidy halves.
   - Cost-of-attack framing: rental hashrate markets make small chains cheap to attack.

**8. Attacks and criticism**

   - 51% double-spend: the mechanics, and real cases (ETC 2019/2020, Bitcoin Gold 2018).
   - Selfish mining: withholding blocks to waste others' work, profitable above ~25–33%.
   - Timewarp, and why timestamp rules from lesson 08 matter.
   - The energy debate stated fairly, and why Ethereum moved to stake (lesson 28).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Comparing hashes as hex strings instead of big integers — works for toy difficulties, wrong in general.
- Using floats in the retarget calculation; different platforms, different chains.
- Leaking goroutines when a miner wins and the losers have nowhere to send their result.
- Re-serializing the whole header every iteration instead of patching the nonce bytes.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Mine a block at 16-bit difficulty; print nonce, hash and elapsed time.
- Expand a compact `bits` value into a 256-bit target.
- Verify someone else's block: one hash, one comparison.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Compress a target back into `bits` and round-trip it.
- Measure time-to-solve as difficulty increases one bit at a time and show the doubling.
- Implement Bitcoin-style retargeting with clamping over a simulated 2016-block window.
- Simulate 1000 block times and print the exponential distribution as a histogram.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A parallel miner: N goroutines, disjoint nonce ranges, first winner cancels the rest via `context`.
- Handle nonce-space exhaustion with an extra-nonce field in the coinbase.
- Simulate selfish mining at 30% hashrate over 10k blocks and report the revenue share.

### Packages & tools

`math/big`, `context`, `sync`, `runtime`, `time`, `crypto/sha256`, `testing`

### Resources to cite

- Bitcoin — Proof of Work: https://developer.bitcoin.org/devguide/block_chain.html
- Bitcoin nBits/compact target: https://developer.bitcoin.org/reference/block_chain.html#target-nbits
- Selfish mining (Eyal & Sirer): https://arxiv.org/abs/1311.0243

---

## 10 — Transactions & the UTXO Model

**Lesson file:** [../10-transactions-utxo.md](../10-transactions-utxo.md) · **Examples folder:** `../examples/10-transactions-utxo/`

| | |
|---|---|
| Prerequisites | [06](../06-keys-signatures.md), [08](../08-blocks-and-chain.md) |
| Unlocks | 11, 12, 15, 36 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 8 |

*inputs, outputs, the coinbase, signing over a trimmed copy, and maintaining the UTXO set*

### Goals

- Model UTXO transactions in Go: inputs referencing prior outputs, outputs locking value.
- Sign and verify a transaction the way Bitcoin does.
- Build the coinbase transaction and enforce conservation of value.
- Maintain and query a UTXO set.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Accounts vs UTXO**

   - Two ways to answer 'who owns what': a balance map, or a set of unspent coins.
   - UTXO is stateless per transaction — validity depends only on the inputs it names.
   - That property is what enables parallel validation and simple SPV proofs.
   - The cost: no notion of 'balance' anywhere in the protocol; wallets compute it.

**2. Transaction structure**

   - `TxInput{PrevTxID [32]byte, OutIndex int, Signature []byte, PubKey []byte}`.
   - `TxOutput{Value int64, PubKeyHash []byte}`.
   - The txid = hash of the serialized transaction; outputs are addressed as (txid, index).
   - Outputs are consumed whole — change is an explicit output back to yourself.

**3. The coinbase transaction**

   - No inputs (or one null input), creates the block subsidy plus collected fees.
   - Arbitrary data field: used for extra-nonce (lesson 09), signalling, and Satoshi's headline.
   - Maturity rule: coinbase outputs unspendable for N blocks, because reorgs orphan them.
   - Halving schedule and its effect on the security budget.

**4. What exactly gets signed**

   - A *trimmed copy*: signatures cleared, and the referenced output's locking script substituted.
   - Why: a signature cannot commit to itself, and each input signs independently.
   - Implementing `TrimmedCopy()` in Go and the subtle bug of mutating the original.
   - Bitcoin's sighash types (ALL/NONE/SINGLE/ANYONECANPAY) previewed — full treatment in lesson 36.

**5. Verification**

   - Per input: hash the trimmed copy, recover/compare the pubkey, check the signature.
   - Check that HASH160(pubkey) equals the referenced output's PubKeyHash.
   - Any single failing input invalidates the whole transaction.
   - Return which input failed — debuggability matters when you write the mempool.

**6. Conservation of value**

   - sum(inputs) ≥ sum(outputs); the difference is the fee, claimed by the miner.
   - No negative outputs, no overflow (this was a real Bitcoin bug in 2010 — CVE-2010-5139).
   - Within-block double-spend detection: track spent outpoints as you validate.
   - Coinbase value ≤ subsidy + total fees.

**7. The UTXO set**

   - The real state: a map from outpoint → output, built by scanning the chain.
   - Applying a block: delete spent outpoints, insert new outputs — one atomic delta.
   - Size and growth; the dust problem; why exchanges consolidate.
   - In Go: `map[Outpoint]TxOutput` in memory now, on disk in lesson 12.

**8. Coin selection**

   - Greedy largest-first, smallest-first, knapsack/branch-and-bound — and their fee consequences.
   - Change output creation and the dust threshold.
   - Privacy leakage: change identification, common-input-ownership heuristic.
   - Implementing one selector behind an interface so you can swap strategies.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Mutating the original transaction inside `TrimmedCopy` and corrupting later inputs.
- Signing before all outputs are final — any later edit invalidates the signature.
- Integer overflow in the value sum; use checked addition or `big.Int`.
- Forgetting coinbase maturity and building on outputs that a reorg erases.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Build a coinbase transaction and print its txid.
- Create a two-output transaction (payment + change) and assert value conservation.
- Serialize and hash a transaction deterministically.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Sign a single-input transaction, verify it, then change an output's value and show failure.
- Implement `TrimmedCopy` and prove the original is untouched.
- Build the UTXO set from a 3-block chain and compute an address balance.
- Detect a double-spend of the same outpoint inside one block.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Implement two coin-selection strategies behind one interface and compare fees and change dust.
- Enforce coinbase maturity and reject a spend of an immature output.
- Reject an overflow attack: two outputs whose values sum past int64.

### Packages & tools

`crypto/ecdsa`, `bytes`, `encoding/binary`, `math/big`, `github.com/ethereum/go-ethereum/crypto`

### Resources to cite

- Bitcoin transactions reference: https://developer.bitcoin.org/reference/transactions.html
- Bitcoin devguide — transactions: https://developer.bitcoin.org/devguide/transactions.html
- CVE-2010-5139 (value overflow): https://en.bitcoin.it/wiki/Value_overflow_incident

---

## 11 — Wallets, Fees & the Mempool

**Lesson file:** [../11-wallets-mempool.md](../11-wallets-mempool.md) · **Examples folder:** `../examples/11-wallets-mempool/`

| | |
|---|---|
| Prerequisites | [07](../07-addresses-wallets-hd.md), [10](../10-transactions-utxo.md) |
| Unlocks | 36 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 9 |

*a keystore, transaction construction, the fee market, mempool policy and block assembly*

### Goals

- Build a wallet that stores keys and constructs spendable transactions.
- Implement a mempool with validation, replacement and eviction.
- Select transactions for a block by fee rate.
- Explain fee estimation and replace-by-fee.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What a wallet is**

   - Keys plus a view of the chain — it holds no coins, it holds the ability to spend them.
   - Watch-only wallets from an xpub (lesson 07) and why custody teams use them.
   - The address book: derived addresses, gap limit, and rescanning.
   - Go modelling: a `Wallet` owning a keystore and a UTXO view, behind an interface.

**2. Persisting keys**

   - A keystore file per key, or one encrypted bundle; JSON over `gob` for forward compatibility.
   - Encrypting at rest: scrypt (N=2^18) to derive a key, AES-GCM to seal — full detail in lesson 44.
   - File permissions (0600), fsync, and atomic replace via temp file + rename.
   - Never logging the key; a `String()` method that redacts.

**3. Building a spend**

   - Select UTXOs → estimate size → compute fee → create change → sign every input.
   - The circularity: fee depends on size, size depends on inputs, inputs depend on fee. Iterate to a fixpoint.
   - Dust: a change output smaller than its own spending cost should become fee instead.
   - Sanity checks before broadcast: value conservation, all inputs signed, change goes to you.

**4. The mempool**

   - An in-memory set of validated-but-unconfirmed transactions, per node, never consensus.
   - Data structure in Go: `map[TxID]*Entry` plus a fee-rate index (`container/heap` or a sorted slice).
   - Admission checks: signature valid, inputs unspent and not already claimed, fee above the floor.
   - Orphan pool for transactions whose parents you have not seen.

**5. Mempool policy vs consensus rules**

   - Policy is local and advisory (min fee, size limits, standardness); consensus is global and binding.
   - A transaction can be consensus-valid and still rejected by your node.
   - Why this distinction causes 'my transaction disappeared' support tickets.
   - Standardness in Bitcoin, and the equivalent in Ethereum's txpool.

**6. Fees**

   - Fee *rate* (sat/vB, gwei/gas) is what matters, not absolute fee — blocks are size-limited.
   - Estimating: histogram of recent inclusions by target confirmation count.
   - Fee spikes, and why an estimate is a prediction, not a promise.
   - Child-pays-for-parent: package fee rate as the selection unit.

**7. Block assembly**

   - A knapsack problem; greedy by fee rate is near-optimal and is what everyone ships.
   - Ancestor/descendant sets must be included together and in order.
   - Size/weight cap and the coinbase reserve.
   - Measuring: total fees collected vs a brute-force optimum on small inputs.

**8. Replacement and eviction**

   - RBF (BIP-125): signal, higher absolute fee *and* higher fee rate, pay for the replaced bandwidth.
   - Transaction pinning: how a low-fee descendant can block a replacement.
   - Eviction under memory pressure: drop lowest fee rate first, raise the dynamic floor.
   - Expiry: drop transactions older than N hours.

**9. Concurrency**

   - The mempool is written by the network layer and read by the miner — a real Go concurrency problem.
   - `sync.RWMutex` vs a single owning goroutine with a command channel; measure both.
   - Avoiding lock-holding across validation (which is slow) — validate first, then insert.
   - Race-detector tests: `go test -race` with concurrent add/select/evict.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Estimating size before signing and under-paying — signatures add ~72 bytes per input.
- Creating dust change that costs more to spend than it is worth.
- Holding the mempool lock while verifying signatures — throughput collapses.
- Treating mempool acceptance as confirmation.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Create a wallet, save it encrypted, reload it and re-derive the same address.
- Compute a fee from a size estimate and a fee rate.
- Add three transactions to a mempool and list them by fee rate.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build a spend end to end: select, fee, change, sign, verify.
- Reject a double-spend at mempool admission.
- Pack a block under a size cap by greedy fee rate and report total fees.
- Evict the lowest-fee-rate transactions when the pool exceeds a byte budget.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Replace a pending transaction with a higher-fee version (RBF) and evict the original.
- Implement child-pays-for-parent package selection and show it beats per-transaction greedy.
- Run concurrent add/select/evict under `-race` with a mutex, then with an owning goroutine, and benchmark both.

### Packages & tools

`sync`, `container/heap`, `encoding/json`, `golang.org/x/crypto/scrypt`, `crypto/aes`, `crypto/cipher`, `os`, `testing`

### Resources to cite

- BIP-125 (RBF): https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- Bitcoin Core mempool policy: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/README.md
- Web3 Secret Storage definition: https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/

---

## 12 — Persistence & Chain State

**Lesson file:** [../12-persistence-chainstate.md](../12-persistence-chainstate.md) · **Examples folder:** `../examples/12-persistence-chainstate/`

| | |
|---|---|
| Prerequisites | [10](../10-transactions-utxo.md) |
| Unlocks | 13 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 9 |

*storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety*

### Goals

- Persist blocks and chain state to an embedded key-value store from Go.
- Design key prefixes and iterate the chain efficiently.
- Write atomically so a crash never leaves a half-applied block.
- Reindex state from raw blocks.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why a key-value store**

   - Access patterns: write once, read by hash, iterate by height, prefix-scan the UTXO set.
   - No joins, no ad-hoc queries — SQL buys you nothing and costs write throughput.
   - The Go options: `bbolt` (B+tree, single file, MVCC reads), `badger` (LSM, fast writes), `pebble` (what geth uses).
   - This course uses bbolt: pure Go, zero dependencies, transactional.

**2. Key-space design**

   - Buckets/prefixes: `b:<hash>` → block, `h:<height>` → hash, `u:<txid>:<idx>` → output, `meta:tip`.
   - Big-endian height keys so lexicographic order equals numeric order — a classic mistake.
   - Keeping keys short: every byte is repeated in every index entry.
   - Documenting the schema in a comment block; you will forget it.

**3. Serialization on disk**

   - Reuse the deterministic encoder from lesson 08 — one format for hashing and storage.
   - Why `gob` is tempting and wrong: type registration, version fragility, no cross-language reads.
   - Compression: is it worth it? Measure before adding it.
   - Schema versioning: a version byte per record so migrations are possible.

**4. Atomic writes**

   - One write transaction per block: tip pointer, block bytes, UTXO deletes and inserts, all or nothing.
   - bbolt's `db.Update` closure and what happens on a returned error.
   - Ordering inside the batch does not matter; atomicity does.
   - The invariant to test: tip height always matches the UTXO set's state.

**5. Iterators and cursors**

   - Walking the chain backwards from the tip by following prevHash.
   - bbolt cursors: `Seek`, `Next`, prefix scans over the UTXO bucket.
   - The read-transaction lifetime rule: values are only valid inside the transaction — copy them out.
   - This is the number-one bbolt bug: returning a slice that points into an mmap'd page.

**6. Reindexing**

   - Rebuild the UTXO set by replaying every block from genesis.
   - Why you need it: corruption recovery, schema migration, and as a correctness oracle in tests.
   - Progress reporting and resumability for a long rebuild.
   - Comparing rebuilt state against stored state — a property test for lessons 10–14.

**7. Pruning and snapshots**

   - What you can discard: spent outputs' history, old block bodies (keep headers).
   - Archive vs full node, and what each can answer.
   - Snapshot/checkpoint files for fast bootstrap.
   - The trade-off table: disk vs the ability to serve historical queries.

**8. Crash safety**

   - fsync semantics, and why 'it wrote' is not 'it is durable'.
   - Testing it: kill the process mid-batch, reopen, assert invariants hold.
   - Idempotent block application so a replay after a crash is harmless.
   - Backups: bbolt's `Tx.WriteTo` for a consistent hot copy.

**9. The storage interface**

   - Define a narrow `Store` interface (`PutBlock`, `GetBlock`, `Tip`, `ApplyBlock`, `IterUTXO`).
   - An in-memory implementation for tests; the bbolt one for real use.
   - This boundary is what makes lessons 13, 14 and 34 tractable.
   - Keep the interface owned by the consumer, not the implementation (idiomatic Go).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Returning a byte slice from inside a bbolt transaction — it points at memory that gets unmapped.
- Encoding heights as decimal strings, so `10` sorts before `9`.
- Applying UTXO deltas outside the block's write transaction — a crash leaves state inconsistent.
- Assuming `db.Update` retries on conflict; it does not, and neither should you silently.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Open a bbolt database, put and get a block by hash.
- Store a height→hash index and read the tip.
- Copy a value out of a read transaction correctly.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Persist a 5-block chain and iterate from tip to genesis.
- Apply a block's UTXO deltas in one atomic transaction.
- Prefix-scan the UTXO bucket to compute an address balance.
- Show the mmap'd-slice bug and then fix it with an explicit copy.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Simulate a crash mid-batch and prove nothing was partially applied.
- Rebuild the UTXO set by replaying blocks and diff it against the stored set.
- Implement the `Store` interface twice (memory + bbolt) and run one test suite against both.

### Packages & tools

`go.etcd.io/bbolt`, `encoding/binary`, `errors`, `os`, `testing`

### Resources to cite

- bbolt: https://pkg.go.dev/go.etcd.io/bbolt
- go-ethereum core/rawdb schema: https://github.com/ethereum/go-ethereum/tree/master/core/rawdb
- Bitcoin Core chainstate notes: https://github.com/bitcoin/bitcoin/blob/master/doc/files.md

---

## 13 — P2P Networking & Gossip

**Lesson file:** [../13-p2p-networking.md](../13-p2p-networking.md) · **Examples folder:** `../examples/13-p2p-networking/`

| | |
|---|---|
| Prerequisites | [12](../12-persistence-chainstate.md) |
| Unlocks | 14 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 9 |

*peer discovery, a handshake, inventory gossip, block/tx propagation and a minimal node in Go*

### Goals

- Write a minimal P2P node in Go: listen, dial, handshake, exchange messages.
- Implement inventory-based gossip so blocks and transactions propagate without flooding.
- Sync a new node from a peer's chain.
- Explain devp2p and libp2p at a high level.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The network layer as a separate concern**

   - Consensus decides what is true; the network decides who hears about it and when.
   - Nothing the network does can make an invalid block valid — always re-validate.
   - Latency and propagation delay drive orphan rate and fork frequency.
   - Design goal: eventual delivery, bounded bandwidth, resistance to spam.

**2. Message framing over TCP**

   - TCP is a byte stream with no message boundaries — you must frame.
   - Length-prefix (4 bytes big-endian) + command byte + payload; cap the length or you have a DoS.
   - `bufio.Reader`/`Writer`, `io.ReadFull`, and never trusting a length you were sent.
   - A `Codec` type with `ReadMsg`/`WriteMsg` and a round-trip test.

**3. Handshake and version negotiation**

   - Exchange: protocol version, chain/network id, genesis hash, best height, node id.
   - Reject peers on a different network before doing any work.
   - Timeouts on the handshake — an unauthenticated peer must not hold resources.
   - Real chains authenticate and encrypt here (RLPx, Noise); we do the plaintext version and name what is missing.

**4. Peer lifecycle in Go**

   - Goroutine per connection for reads; a separate write pump owning the socket writer.
   - Never write to a `net.Conn` from two goroutines — this is the classic P2P bug.
   - `SetReadDeadline`/`SetWriteDeadline` as the only reliable cancellation for a blocked socket.
   - Clean teardown: close the connection, drain, `sync.WaitGroup` on the goroutines.

**5. Gossip that does not melt the network**

   - Never broadcast full blocks blindly. Announce `inv` (hashes), peers request `getdata`.
   - Dedupe with a recently-seen set (bounded, TTL'd) so announcements do not loop forever.
   - Rate-limit per peer; bound in-flight requests per peer.
   - Relay only after validation — otherwise you are a spam amplifier.

**6. Initial block download**

   - Headers-first: fetch and validate the header chain, then fetch bodies in parallel.
   - Why headers-first: cheap to validate, lets you detect a bogus chain before downloading megabytes.
   - Parallel body fetch with bounded concurrency and in-order commit.
   - Checkpoints/anchors to bound the work an attacker can force.

**7. Orphans and future blocks**

   - A block whose parent you lack: hold it briefly, request the parent, or drop it.
   - Bounded orphan pool — unbounded is a memory DoS.
   - The same pattern for transactions with missing parents.
   - Re-processing the pool when the missing parent arrives.

**8. Discovery**

   - Hardcoded seeds → DNS seeds → a DHT. The bootstrap problem is real.
   - devp2p discv4/discv5: Kademlia over UDP, ENR records; libp2p on the consensus layer and IPFS.
   - Maintaining a peer database with quality scores and last-seen times.
   - Outbound vs inbound slot management, so you cannot be surrounded.

**9. Adversarial networking**

   - Eclipse attacks: fill a victim's peer slots and control its view. Countermeasures: diverse buckets, anchor peers.
   - Peer scoring and ban lists for invalid messages.
   - Sybil resistance at the network layer is different from consensus-layer Sybil resistance.
   - Message size caps, per-peer rate limits, and slow-loris timeouts.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Writing to one connection from multiple goroutines — interleaved, corrupt frames.
- Trusting a peer-supplied length prefix and allocating gigabytes.
- Unbounded orphan or seen-set maps that grow until OOM.
- Relaying before validating, turning your node into an attacker's megaphone.
- Using `context` alone to cancel a blocked socket read — you need a deadline.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- A length-prefixed codec with a round-trip test.
- Dial and accept a TCP connection and exchange one message.
- Reject a peer whose genesis hash differs.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- A two-node handshake printing each side's view of the other.
- Broadcast a block via `inv` → `getdata` → `block` to three peers.
- Dedupe repeated announcements with a bounded TTL set.
- Enforce a max message size and drop a peer that exceeds it.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Bring a fresh node from height 0 to the network tip with headers-first sync.
- Parallel body download with bounded concurrency and in-order commit.
- A write-pump peer implementation that survives `-race` under concurrent broadcast and disconnect.

### Packages & tools

`net`, `bufio`, `encoding/binary`, `context`, `sync`, `time`, `testing`

### Resources to cite

- devp2p specs: https://github.com/ethereum/devp2p
- libp2p docs: https://docs.libp2p.io/
- Bitcoin P2P network reference: https://developer.bitcoin.org/reference/p2p_networking.html
- Go — package net: https://pkg.go.dev/net

---

## 14 — Consensus, Forks & Reorgs

**Lesson file:** [../14-consensus-forks.md](../14-consensus-forks.md) · **Examples folder:** `../examples/14-consensus-forks/`

| | |
|---|---|
| Prerequisites | [13](../13-p2p-networking.md) |
| Unlocks | 15, 28 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 9 |

*fork choice by accumulated work, reorg handling, finality as probability, and the attacks*

### Goals

- Implement heaviest-chain fork choice by accumulated work.
- Handle a reorg: find the common ancestor, roll back, roll forward.
- Explain probabilistic finality and confirmation counts.
- Name the failure modes: 51%, selfish mining, consensus-bug splits.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Consensus is agreement on order**

   - Every node has the same rules; disagreement is only ever about which history is canonical.
   - Validity is objective and local; canonicity is a network-wide choice.
   - Restating lesson 01 now that you have blocks, work and a network.
   - Why this is the last piece: without it, lessons 08–13 are just a database.

**2. Fork choice**

   - Not 'longest chain' — most accumulated work. Height lies under variable difficulty.
   - Work per block = 2²⁵⁶ / (target+1); sum it along the branch, store it per block.
   - Tie-breaking: first-seen, which is why propagation speed matters commercially.
   - In Go: a `big.Int` total-work field on each block index entry.

**3. The block tree**

   - You store a tree, not a chain: every valid block, whichever branch it is on.
   - A block index: hash → {header, height, totalWork, parent, status}.
   - Finding the common ancestor: walk both tips back by height, then step in lockstep.
   - Memory: keep the index in RAM, bodies on disk.

**4. Reorg mechanics**

   - Disconnect blocks from the old tip to the fork point, in reverse order.
   - Disconnecting means restoring spent UTXOs and removing created ones — you need undo data.
   - Then connect the new branch in order, validating each block against the evolving state.
   - All of it inside one atomic write batch (lesson 12), or you can corrupt state.

**5. Undo data**

   - Per block, record what it spent so you can restore it. Bitcoin calls these 'undo files'.
   - The alternative — replay from genesis — is correct and unusably slow.
   - Sizing: undo data is roughly proportional to inputs consumed.
   - Testing: apply then undo a block and assert state is byte-identical to before.

**6. Returning transactions to the mempool**

   - Transactions in disconnected blocks are valid again and must go back to the pool.
   - Except coinbases, which are destroyed with their block.
   - Some will conflict with the new branch — re-validate rather than blindly re-adding.
   - The step everyone forgets, and the reason users see 'my payment vanished'.

**7. Confirmations and probabilistic finality**

   - No block is ever final; the probability an attacker catches up decays exponentially with depth.
   - The Nakamoto race model — implement the calculation and print a table.
   - Why exchanges use different confirmation counts per chain and per amount.
   - The contrast with BFT finality (lesson 29) and Ethereum's economic finality (lesson 28).

**8. Forks of the other kind**

   - Soft fork (tightening rules, backward-compatible) vs hard fork (loosening/changing, not).
   - Chain splits: ETH/ETC, BTC/BCH — social consensus, not code.
   - Accidental consensus forks from implementation differences (Bitcoin's 2013 BDB fork).
   - Why client diversity is a safety property and a monoculture is a risk.

**9. Attacks**

   - 51% double-spend: the mechanics end to end, and real incidents (ETC, Bitcoin Gold).
   - Selfish mining revisited with the fork-choice code in front of you.
   - Deep-reorg defence: checkpoints, and why they are philosophically contentious.
   - Cost of attack vs value secured — the number that actually matters.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Choosing by height instead of accumulated work — an attacker mines many easy blocks.
- Reorging without undo data and silently corrupting the UTXO set.
- Forgetting to return disconnected transactions to the mempool.
- Treating one confirmation as final for anything valuable.
- Doing a reorg across several write transactions so a crash leaves you mid-reorg.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Compute a block's work from its target and sum it along a chain.
- Build a two-branch tree and pick the heavier tip.
- Find the common ancestor of two tips.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Reorg two blocks deep and assert balances match the new branch.
- Record and apply undo data; assert apply-then-undo is a no-op.
- Return disconnected transactions to the mempool and confirm they are re-mineable.
- Reject a longer-but-lighter branch.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Do a full reorg inside one atomic write batch and crash-test it.
- Simulate an attacker at 10/20/30/40% hashrate over 10k trials and print double-spend success by confirmation depth.
- Reproduce a selfish-mining revenue advantage against honest miners in simulation.

### Packages & tools

`math/big`, `sort`, `sync`, `go.etcd.io/bbolt`, `testing`

### Resources to cite

- Bitcoin whitepaper §11 (attacker race): https://bitcoin.org/bitcoin.pdf
- Bitcoin March 2013 chain fork: https://github.com/bitcoin/bips/blob/master/bip-0050.mediawiki
- Ethereum fork history: https://ethereum.org/en/history/

---

## 15 — The Account Model & World State

**Lesson file:** [../15-account-model-state.md](../15-account-model-state.md) · **Examples folder:** `../examples/15-account-model-state/`

| | |
|---|---|
| Prerequisites | [10](../10-transactions-utxo.md), [14](../14-consensus-forks.md) |
| Unlocks | 16, 38 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 8 |

*accounts, nonces, balances, code and storage — the state transition and how it differs from UTXO*

### Goals

- Model an account-based ledger in Go: balance, nonce, code hash, storage root.
- Explain nonces as replay protection and as ordering.
- Apply a transaction as a state transition and compute a new state root.
- Compare UTXO and account models honestly.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. From coins to accounts**

   - State is `map[Address]Account`, not a set of unspent outputs.
   - A transfer mutates two entries instead of consuming and creating coins.
   - This is why Ethereum can have contracts with persistent storage and Bitcoin cannot easily.
   - The cost: transactions are no longer independently validatable.

**2. The account struct**

   - nonce uint64, balance *big.Int, storageRoot [32]byte, codeHash [32]byte.
   - EOA: codeHash = keccak(""), storageRoot = empty trie root. Contract: both set.
   - Empty-account semantics and EIP-161 (state-clearing) — a real source of consensus bugs.
   - In Go: a struct plus a `IsEmpty()` helper, and RLP encoding (lesson 17).

**3. Nonces**

   - Strictly sequential per sender, starting at 0. Every transaction increments it.
   - Three jobs: replay protection, ordering, and preventing accidental duplicates.
   - The gap problem: nonce 5 pending, nonce 7 submitted ⇒ 7 waits forever (queued, not pending).
   - Contract nonces count CREATEs, which is why CREATE addresses are predictable (lesson 07).

**4. The state transition function**

   - σ' = Υ(σ, T): check nonce, check balance ≥ gas·price + value, deduct, execute, refund, increment nonce.
   - Intrinsic gas before execution; the fee is charged even if execution reverts.
   - In Go: a `StateTransition` type with explicit ordered steps and typed errors per rule.
   - Order matters and is consensus-critical — get it wrong and you fork.

**5. Journalling and reverts**

   - A failed call must undo its writes but still burn gas — you need an undo journal.
   - Journal entries: balance change, nonce change, storage write, account creation, log emission.
   - Snapshots as journal indices; revert = replay entries backwards to a marker.
   - This is exactly how geth's `StateDB` works, and you build a small version of it.

**6. The state root**

   - A commitment to the whole world state, in the header, updated every block.
   - Toy version now: hash the sorted account list. Real version in lesson 17 (Patricia trie).
   - Why a commitment is needed: light clients, fraud proofs, fast sync.
   - Show that changing one balance changes the root.

**7. Copy-on-write state in Go**

   - A layered state: a base map plus a dirty overlay per transaction.
   - `maps.Clone` vs an explicit journal — measure both for a 100k-account state.
   - Why you never mutate the committed map directly.
   - Concurrency: state is single-writer during execution; parallelism comes elsewhere.

**8. UTXO vs accounts, honestly**

   - Replay protection: free in UTXO (outputs are unique), needs nonces in accounts.
   - Parallel validation: natural in UTXO, hard in accounts (this is Solana's whole pitch — lesson 38).
   - Privacy: UTXO's change addresses give weak unlinkability; accounts are fully linkable.
   - State growth: UTXO set shrinks when coins consolidate; account state only grows (lesson 66).
   - Developer experience: accounts win decisively for contracts.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Incrementing the nonce only on success — it must increment even when execution reverts.
- Deducting gas after execution instead of before; an out-of-gas transaction must still pay.
- Reverting by re-reading from disk instead of a journal — slow and wrong under nesting.
- Using a Go map's iteration order anywhere near the state root.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Apply a transfer to a `map[Address]Account` and print before/after.
- Reject a transaction with the wrong nonce.
- Compute a toy state root by hashing sorted accounts.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Charge intrinsic gas up front and refund the unused remainder.
- Show the pending-vs-queued distinction with a nonce gap.
- Implement snapshot/revert with a journal and prove balances are restored exactly.
- Show that any state change moves the state root.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Nested calls with nested snapshots: inner revert, outer commit, correct final state.
- A copy-on-write state overlay benchmarked against full map cloning at 100k accounts.
- Replay a block of 200 transactions and assert the resulting state root is deterministic across runs.

### Packages & tools

`math/big`, `sort`, `maps`, `crypto/sha256`, `github.com/ethereum/go-ethereum/common`, `testing`

### Resources to cite

- Ethereum Yellow Paper (state transition): https://ethereum.github.io/yellowpaper/paper.pdf
- Ethereum docs — Accounts: https://ethereum.org/en/developers/docs/accounts/
- EIP-161 state trie clearing: https://eips.ethereum.org/EIPS/eip-161
- geth core/state: https://github.com/ethereum/go-ethereum/tree/master/core/state

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
