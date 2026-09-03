# Part 7 — Production Blockchain Engineering in Go

The job. Indexers that survive reorgs, key management that survives an audit, node operations, deterministic tests, and the observability that tells you when RPC is lying to you.

**Core spine.** Lessons 31–35 · 85 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 31 | [Building a Blockchain Indexer in Go](#31-building-a-blockchain-indexer-in-go) | 25, 20 | 19 |
| 32 | [Key Management & Signing Services](#32-key-management-signing-services) | 21 | 17 |
| 33 | [Node Operations & RPC Infrastructure](#33-node-operations-rpc-infrastructure) | 20 | 15 |
| 34 | [Testing Blockchain Code in Go](#34-testing-blockchain-code-in-go) | 24, 31 | 17 |
| 35 | [Observability & Reliability for Chain Services](#35-observability-reliability-for-chain-services) | 31, 33 | 17 |

---

## 31 — Building a Blockchain Indexer in Go

**Lesson file:** [../31-blockchain-indexer.md](../31-blockchain-indexer.md) · **Examples folder:** `../examples/31-blockchain-indexer/`

| | |
|---|---|
| Prerequisites | [25](../25-events-logs.md), [20](../20-json-rpc-ethclient.md) |
| Unlocks | 34, 35, 41, 56, 57, 58 |
| Examples | **19** — 🟢 5 easy, 🟡 9 medium, 🔴 5 hard |
| Topics | 11 |

*ingesting blocks and logs, reorg-safe writes, backfill, idempotency and a schema that answers queries*

### Goals

- Build a service that ingests blocks and logs into a database, continuously.
- Handle reorgs correctly — detect, roll back, re-apply.
- Backfill history quickly without exhausting your provider.
- Design a schema and cursor model that make the indexer restartable and idempotent.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why you index**

   - RPC cannot answer 'all transfers for this user, newest first, page 3'.
   - The chain is an event log; your database is the materialised view.
   - The three consumers: your API, your accounting, your alerts.
   - Build vs buy (subgraphs, hosted APIs) — and why teams end up building anyway (lesson 56).

**2. The ingestion loop**

   - poll head → decide target range → fetch → decode → write → advance cursor.
   - A confirmation lag (e.g. head − 12) so most reorgs never reach your database.
   - Separating 'fetch' from 'apply' so you can retry each independently.
   - The loop must be restartable at any point with no duplicate effects.

**3. Reorg detection**

   - Store blockHash **and** parentHash for every block you index.
   - On each new block, check its parentHash equals your stored tip hash. Mismatch ⇒ reorg.
   - Walk backwards until stored hash == chain hash: that is the common ancestor.
   - Bound the walk; a reorg deeper than your history is an alert, not a retry.

**4. Reorg-safe writes**

   - Every row carries the block number and block hash that produced it.
   - Rollback = delete rows where block_number > ancestor. No mutation-in-place aggregates.
   - If you must keep aggregates, store them as a fold over block-scoped deltas so they can be recomputed.
   - Do rollback and re-apply in one database transaction.

**5. Idempotency**

   - Natural key: (block_hash, log_index) — globally unique and reorg-aware.
   - `INSERT ... ON CONFLICT DO NOTHING` / `DO UPDATE` so replay is free.
   - Effects outside the database (webhooks, emails) need their own dedupe key (lesson 51).
   - Test it: run the same range twice and assert identical row counts.

**6. Backfill vs live tail**

   - Two modes, one codebase: bounded historical ranges vs following the head.
   - Bounded concurrency with `errgroup.SetLimit`, but **ordered commits** — never write block N+1 before N.
   - A worker pool that fetches out of order into a reorder buffer.
   - Progress, resumability and ETA reporting for a multi-day backfill.

**7. Schema design**

   - Tables: `blocks`, `transactions`, `logs`, plus one decoded table per event type.
   - Indexes that queries actually need: (address, block_number DESC), (block_hash), (tx_hash).
   - Store raw log bytes alongside decoded columns so you can re-decode after a bug.
   - Numeric types: NUMERIC(78,0) for uint256 in Postgres, never BIGINT.

**8. Cursors and checkpoints**

   - One row holding last_indexed_block + hash, updated in the same transaction as the data.
   - This is what turns at-least-once processing into exactly-once *effects*.
   - Separate cursors per pipeline so a slow decoder does not block ingestion.
   - Recovering from a corrupted cursor by re-deriving it from the data.

**9. Provider pressure**

   - Adaptive chunk sizing: halve on error, grow on success, remember the last good size.
   - Rate limiting yourself with `golang.org/x/time/rate` before the provider does it for you.
   - Failover across providers, and the consistency traps that creates (lesson 33).
   - Cost accounting: log RPC calls per indexed block.

**10. Operations**

   - Metrics: lag in blocks and seconds, blocks/sec, reorg count and depth, RPC errors (lesson 35).
   - Gap detection: assert every block number between cursor and head exists.
   - A `reindex` command that replays a range without downtime.
   - Graceful shutdown that finishes the current block and commits the cursor.

**11. Testing it**

   - A `ChainSource` interface so tests inject a fake chain that can reorg on demand.
   - Golden tests over a recorded block range.
   - Property test: for any reorg depth ≤ N, final state equals a fresh index of the final chain.
   - Full treatment in lesson 34.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Indexing at `latest` with no confirmation lag and thrashing on every reorg.
- Storing only block numbers, so a reorg is undetectable.
- Mutating aggregate counters in place, making rollback impossible.
- Writing data and the cursor in separate transactions.
- Unbounded concurrency in backfill, which gets your API key rate-limited or banned.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 19).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Poll for new heads with a 12-block confirmation lag and print each.
- Store blocks with hash and parentHash in Postgres.
- Advance a cursor row in the same transaction as the data.

**🟡 Medium — 9 examples** *(concepts combined, and the traps)*

- Detect a reorg by parent-hash mismatch and find the common ancestor.
- Roll back three blocks' rows and re-apply the new branch in one transaction.
- Backfill 100k blocks of one contract's logs with bounded concurrency and ordered commits.
- Prove idempotency: run the same range twice and assert row counts are unchanged.
- Adaptive chunk sizing that halves on provider error and grows back.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A `ChainSource` fake that scripts a 5-block reorg; assert final DB state matches a clean index.
- Decode multiple event types into separate tables from one log stream, with per-type cursors.
- Gap detection plus an automatic repair pass for missing blocks.

### Packages & tools

`database/sql`, `github.com/jackc/pgx/v5`, `golang.org/x/sync/errgroup`, `golang.org/x/time/rate`, `context`, `log/slog`, `github.com/ethereum/go-ethereum/ethclient`

### Resources to cite

- pgx: https://pkg.go.dev/github.com/jackc/pgx/v5
- errgroup: https://pkg.go.dev/golang.org/x/sync/errgroup
- Ethereum JSON-RPC eth_getLogs: https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_getlogs

---

## 32 — Key Management & Signing Services

**Lesson file:** [../32-key-management-signing.md](../32-key-management-signing.md) · **Examples folder:** `../examples/32-key-management-signing/`

| | |
|---|---|
| Prerequisites | [21](../21-sending-transactions.md) |
| Unlocks | 41, 43, 58, 59, 68 |
| Examples | **17** — 🟢 4 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*keystores, KMS/HSM, hot/warm/cold tiers, a policy-enforcing signing service, and nonces at scale*

### Goals

- Store and use keys without a private key ever entering your source, logs or backups.
- Build a signing service with a real authorization boundary.
- Manage nonces for many concurrent senders reliably.
- Reason about tiering and blast radius.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Threat model first**

   - What an attacker gets from: process memory, environment variables, logs, crash dumps, backups, a stolen laptop.
   - Insider risk and the four-eyes principle.
   - Blast radius: what is the maximum loss if this exact key leaks right now?
   - Write the threat model down before writing code — it determines every choice below.

**2. Web3 keystore v3**

   - scrypt (or pbkdf2) KDF → AES-128-CTR ciphertext → keccak MAC over (derivedKey[16:32] ‖ ciphertext).
   - `accounts/keystore` in Go: `NewKeyStore`, `ImportECDSA`, `Unlock`, `SignTx`.
   - scrypt parameters (N=262144 standard, 4096 'light') and what each costs in time and memory.
   - The passphrase is the entire security; a weak one makes the file worthless.

**3. Cloud KMS and HSMs**

   - Sign without holding the key: the service returns a signature, never the secret.
   - secp256k1 support (AWS KMS, GCP KMS) and the DER-encoded output you must convert.
   - The recovery-id problem: KMS gives you (r, s) but not `v` — try both and match the address.
   - Low-s normalisation is your responsibility, not the KMS's.
   - Latency, cost per signature, and rate limits as design constraints.

**4. A signing service**

   - An internal API: take an *unsigned* transaction plus context, return a signature. Never expose keys.
   - Interface design in Go: `Signer interface { SignTx(ctx, addr, *types.Transaction) (*types.Transaction, error) }`.
   - Implementations: in-memory (tests), keystore (dev), KMS (prod) — same interface.
   - Authentication of callers (mTLS or signed requests) and per-caller authorization.

**5. Policy at the signing boundary**

   - Destination allowlists, per-transaction and per-window value caps, contract-method allowlists.
   - Rate limits per key; velocity checks; anomaly rules.
   - An append-only audit log: who asked, what was signed, what the policy decided.
   - Manual approval for anything over a threshold — the human circuit breaker.

**6. Hot / warm / cold tiering**

   - Hot: online, automated, small float, tight policy. Warm: online but approval-gated. Cold: offline, air-gapped.
   - Matching key value to protection level, and the sweep flows between tiers (lesson 59).
   - Deposit addresses derive from a watch-only xpub; spending keys live cold (lesson 58).
   - The operational cost of cold storage is the real constraint.

**7. Multisig and threshold signing**

   - On-chain multisig (Safe): m-of-n approvals enforced by a contract; visible, auditable, gas cost.
   - MPC/TSS: n parties jointly produce one signature; no key ever exists (lesson 43).
   - Trade-offs: on-chain multisig is simple and transparent; MPC is chain-agnostic and private.
   - Which to choose for treasury vs for automated operations.

**8. Nonce management at scale**

   - Per-address serialization is unavoidable — one signer goroutine or one row lock per address.
   - A database-backed allocator: `SELECT ... FOR UPDATE`, increment, commit with the signed transaction.
   - Gap detection and a sweeper that cancels or replaces stuck nonces (lesson 21).
   - Never share an address between two independent services.

**9. Many senders**

   - A pool of hot addresses to increase throughput past one-nonce-at-a-time.
   - Assignment strategies: round-robin, least-busy, sticky-per-user.
   - Funding and sweeping the pool; keeping balances above a floor.
   - Address rotation for privacy, and the accounting complexity it adds (lesson 60).

**10. Operational hygiene**

   - Key rotation: how to actually do it when addresses are baked into contracts and customer records.
   - Backup and recovery drills — an untested backup is not a backup.
   - Secret scanning in CI; `.gitignore` is not a control.
   - Redaction in `log/slog` with a custom `Handler`, and a `PrivateKey` type whose `String()` is `[REDACTED]`.
   - Zeroing memory in Go: `runtime.KeepAlive`, the limits of it, and why the GC makes guarantees impossible.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- A private key in an environment variable on a shared host.
- Logging a signed transaction's `v/r/s` alongside the payload and thinking it is safe (it is, but the key next to it is not).
- Using the same address from two services and interleaving nonces.
- Deriving `v` by guessing without verifying the recovered address.
- Storing a keystore file and its passphrase in the same secret store.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 17).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Create and load a keystore file, then sign a transaction with it.
- Define a `Signer` interface and implement the in-memory version.
- Redact a private key from `slog` output with a custom handler.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Implement the keystore-backed `Signer` and swap it in without changing callers.
- A signing HTTP service that refuses a destination outside its allowlist.
- Enforce a per-window value cap and return a typed policy error.
- A database-backed nonce allocator serializing 20 concurrent requests for one address.
- Recover `v` from an (r, s) pair by trying both parities and matching the expected address.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A hot-wallet pool with least-busy assignment, balance floors and automatic top-up.
- An append-only audit log with a hash chain so entries cannot be silently edited.
- Simulate a KMS signer (DER output, no recovery id) and produce a valid Ethereum transaction from it.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/keystore`, `github.com/ethereum/go-ethereum/crypto`, `log/slog`, `database/sql`, `sync`, `crypto/subtle`, `net/http`

### Resources to cite

- Web3 Secret Storage: https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/
- keystore package: https://pkg.go.dev/github.com/ethereum/go-ethereum/accounts/keystore
- AWS KMS ECDSA signing: https://docs.aws.amazon.com/kms/latest/developerguide/asymmetric-key-specs.html
- Safe (multisig) docs: https://docs.safe.global/

---

## 33 — Node Operations & RPC Infrastructure

**Lesson file:** [../33-node-operations.md](../33-node-operations.md) · **Examples folder:** `../examples/33-node-operations/`

| | |
|---|---|
| Prerequisites | [20](../20-json-rpc-ethclient.md) |
| Unlocks | 35, 63 |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 11 |

*running geth, sync modes, storage, providers, failover and what actually breaks in production*

### Goals

- Run and monitor an execution + consensus client pair.
- Choose a sync mode and understand its disk and time cost.
- Decide between self-hosting and a provider, with numbers.
- Build a resilient RPC client layer in Go.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The post-Merge reality**

   - You need two clients, an Engine API connection and a shared JWT secret file.
   - Neither works alone: the EL will not advance without the CL telling it the head.
   - Ports and interfaces: 8545 (HTTP), 8546 (WS), 8551 (Engine, authenticated), 30303 (p2p).
   - Client diversity as a network health issue and a personal risk-management one.

**2. Execution clients**

   - geth (reference, Go), erigon (Go, staged sync, much smaller archive), nethermind (C#), reth (Rust).
   - Very different disk profiles: geth snap ~1.2TB, erigon archive far smaller than geth archive.
   - Which to pick for: a dApp backend, an indexer, an archive/analytics workload.
   - Reading geth's logs: what 'Imported new chain segment' and 'Syncing' actually mean.

**3. Sync modes**

   - Snap sync: downloads state at a pivot, then fills in — days, not weeks.
   - Full sync: executes every block from genesis; only needed for verification research.
   - Archive: keeps every historical state — required for historical `eth_call` and `getProof`.
   - What each mode can answer, and the error you get when you ask for too much history.

**4. Hardware**

   - NVMe SSD is mandatory; the workload is random 4k reads and a spinning disk will never sync.
   - IOPS, RAM (16–32GB), and CPU as a distant third.
   - Disk growth rate and the pruning cadence; running out of disk is the #1 node outage.
   - Filesystem choice and the `noatime` mount option.

**5. Configuration that matters**

   - `--http.api`, `--ws.api` — expose only what you use; never expose `admin`/`personal` publicly.
   - `--cache`, `--maxpeers`, `--gcmode=archive`, `--history.transactions`.
   - `--authrpc.jwtsecret` and locking the Engine API to localhost.
   - Never expose 8545 to the internet without auth and rate limiting.

**6. Monitoring a node**

   - Peer count, sync distance, disk free, memory, block-import latency.
   - geth's `--metrics` endpoint into Prometheus; the dashboards worth having.
   - Alerting: sync distance > N, peers < M, disk < 15%, no new block in 60s.
   - Correlating node health with application lag (lesson 35).

**7. Providers**

   - Alchemy, Infura, QuickNode, Ankr — compute units, rate limits, archive access, trace support.
   - The failure modes: silent truncation, stale reads behind a load balancer, regional outages.
   - Reading the small print on `getLogs` ranges and `debug_*` availability.
   - Cost modelling: requests per indexed block × blocks per day.

**8. Build vs buy**

   - The honest comparison: a node is ~$150/month of hardware plus real operational time.
   - A provider is faster to start and becomes expensive at indexer scale.
   - The usual answer: providers for the tail, your own node for the bulk.
   - Reliability argues for both, always.

**9. A resilient RPC layer in Go**

   - An interface over `ethclient` so upstreams are swappable.
   - Multiple upstreams with health checks (head lag, error rate, latency).
   - Failover, and **hedged requests** — fire two, take the first, cancel the loser.
   - Per-method routing: archive queries to the archive node, head polling to the cheap one.
   - Sticky routing when you need read-your-writes.

**10. Consistency traps**

   - Two providers disagree about `latest`; a load balancer makes it nondeterministic.
   - You send a transaction to node A and query node B, which has not seen it.
   - Missing logs from a provider that pruned or lagged.
   - Mitigations: sticky sessions per workflow, minimum-height gating, cross-checking critical reads.

**11. Local development**

   - `anvil` for speed and cheats; `geth --dev` for realism; `anvil --fork-url` for mainnet state.
   - Deterministic accounts and instant mining as test infrastructure (lesson 34).
   - Docker Compose for a geth + prysm pair on a testnet.
   - Snapshot/restore of local chain state to make tests fast.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Exposing 8545 publicly — automated scanners will find it in minutes.
- Running out of disk with no alert; the node corrupts and you resync for days.
- Assuming one provider is enough for anything that handles money.
- Querying a full node for historical state and treating the error as a bug in your code.
- Load-balancing writes and reads across providers with no stickiness.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Start `geth --dev` and query it from Go.
- Print peer count and sync progress from a running node.
- Compare `latest` block numbers across two providers.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- A health-checked multi-upstream `ethclient` wrapper with automatic failover.
- Hedged request: fire two upstreams, take the first good answer, cancel the loser.
- Detect a stale provider by comparing head numbers and quarantine it.
- Route archive queries to one upstream and head polling to another.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Run geth + a consensus client with Docker Compose against a testnet and index from it.
- Reproduce the read-your-writes failure across two upstreams and fix it with height gating.
- Measure and chart p50/p99 latency per method per upstream over 10 minutes.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `context`, `sync`, `time`, `net/http`, `github.com/prometheus/client_golang/prometheus`

### Resources to cite

- geth docs: https://geth.ethereum.org/docs
- geth sync modes: https://geth.ethereum.org/docs/fundamentals/sync-modes
- Erigon: https://github.com/erigontech/erigon
- Ethereum docs — nodes and clients: https://ethereum.org/en/developers/docs/nodes-and-clients/

---

## 34 — Testing Blockchain Code in Go

**Lesson file:** [../34-testing-blockchain-go.md](../34-testing-blockchain-go.md) · **Examples folder:** `../examples/34-testing-blockchain-go/`

| | |
|---|---|
| Prerequisites | [24](../24-abigen-bindings.md), [31](../31-blockchain-indexer.md) |
| Unlocks | 41, 48, 57 |
| Examples | **17** — 🟢 4 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*simulated backends, forked chains, fakes for RPC, deterministic fixtures and reorg tests*

### Goals

- Test contract interaction with no node, using a simulated backend.
- Fork mainnet locally and test against real state.
- Fake the RPC layer behind an interface so tests are fast and deterministic.
- Write a test that reproduces a reorg.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The test pyramid for chain code**

   - Pure logic (encoders, fee math, selection) — plain Go tests, milliseconds.
   - Simulated chain — real EVM, no network, deterministic block production.
   - Forked chain — real mainnet state via `anvil --fork-url`, slower, needs network.
   - Testnet and mainnet — smoke tests only, never your main suite.
   - Most of your tests should be in the first two layers.

**2. The simulated backend**

   - `simulated.NewBackend(types.GenesisAlloc{...})` — in-process chain, instant, free.
   - `Commit()` mines exactly one block: nothing is nondeterministic unless you make it so.
   - `AdjustTime` for time-locked logic; mining empty blocks to advance the chain.
   - Its limits: no realistic gas market, no reorgs, no mempool competition, no other clients.

**3. Forking with anvil**

   - `anvil --fork-url <archive> --fork-block-number N` — real state, pinned for reproducibility.
   - Cheatcodes over RPC: `anvil_impersonateAccount`, `anvil_setBalance`, `anvil_setStorageAt`, `anvil_mine`.
   - Starting and stopping anvil from a Go test with `os/exec` and a readiness probe.
   - Always pin the fork block; an unpinned fork makes tests fail on someone else's schedule.

**4. Deterministic fixtures**

   - Fixed test keys (labelled!), a fixed chain id, a fixed genesis allocation.
   - Golden files for ABI-encoded blobs, RLP bytes and decoded structs.
   - Recorded RPC responses replayed from disk — a `httptest.Server` serving fixtures.
   - Regenerating goldens behind a `-update` flag, and reviewing the diff.

**5. Faking the RPC layer**

   - Depend on **your own narrow interface**, never on `*ethclient.Client`.
   - `type ChainSource interface { HeaderByNumber(...); FilterLogs(...) }` — two methods, easy to fake.
   - A scripted fake that returns a canned chain, including a branch swap at a chosen height.
   - Table-driven tests over the fake for every reorg depth from 1 to N.

**6. Testing reorgs**

   - Build chain A (blocks 1..10), index it, then present chain B forking at block 7.
   - Assert: rows above 7 are gone, rows for B's 8..12 exist, the cursor is correct.
   - Property test: final state after any reorg sequence equals a fresh index of the final chain.
   - This is the single most valuable test in an indexer's suite.

**7. Fuzzing**

   - `testing.F` on decoders: RLP, ABI, log decoding, address parsing.
   - Seed the corpus with real mainnet data.
   - What you are looking for: panics, unbounded allocation, infinite loops.
   - Running fuzz in CI with a time budget, and checking in the crashers.

**8. Property and invariant tests**

   - Value conservation across a simulated block of transfers.
   - Nonce monotonicity per sender; balances never negative.
   - Encode/decode round-trip for every type you serialize.
   - Comparing your mini-EVM against geth's `core/vm` on random bytecode (lesson 18).

**9. Foundry as a complement**

   - `forge test` for contract-side logic; `vm.prank`, `vm.expectRevert`, `vm.warp`.
   - Invariant/fuzz testing in Foundry for the contract, Go tests for the integration.
   - Where Go is the better tool: anything about *your* service's behaviour.
   - Keeping both suites in one CI pipeline.

**10. CI**

   - What runs with no network: unit, simulated backend, fakes — should be 90% of the suite.
   - Gating network-dependent tests behind a build tag or `-short`.
   - `go test -race` on everything concurrent (indexer, nonce manager, P2P).
   - Caching the go-ethereum build, which otherwise dominates CI time.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Depending on `*ethclient.Client` directly, making the code untestable without a node.
- Unpinned mainnet forks, so tests break when mainnet state changes.
- Tests that assume the simulated backend's gas numbers match mainnet.
- No reorg test — the bug will appear in production instead.
- Sleeping in tests instead of using `Commit()` or a deterministic clock.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 17).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Deploy and call a contract on a simulated backend inside `go test`.
- Define a `ChainSource` interface and a trivial fake.
- Golden-test an ABI encoder against a checked-in fixture.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Test a time-locked contract with `AdjustTime` and mined empty blocks.
- Start `anvil --fork-url` from a Go test, impersonate a whale and transfer USDC.
- Feed an indexer 3 blocks then a reorg through the fake and assert final DB state.
- Fuzz an ABI decoder and fix the first panic it finds.
- Run the concurrent nonce manager under `-race`.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A property test asserting final state after any reorg sequence equals a fresh index.
- Differential-test your mini-EVM against geth's `core/vm` on 1000 random programs.
- A `httptest`-based RPC fixture server that replays recorded responses for a whole test suite.

### Packages & tools

`testing`, `github.com/ethereum/go-ethereum/ethclient/simulated`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `net/http/httptest`, `os/exec`

### Resources to cite

- Go testing package: https://pkg.go.dev/testing
- Go fuzzing: https://go.dev/doc/security/fuzz/
- simulated backend: https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient/simulated
- Foundry cheatcodes: https://book.getfoundry.sh/cheatcodes/
- Anvil: https://book.getfoundry.sh/anvil/

---

## 35 — Observability & Reliability for Chain Services

**Lesson file:** [../35-observability-reliability.md](../35-observability-reliability.md) · **Examples folder:** `../examples/35-observability-reliability/`

| | |
|---|---|
| Prerequisites | [31](../31-blockchain-indexer.md), [33](../33-node-operations.md) |
| Unlocks | 41, 68 |
| Examples | **17** — 🟢 4 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*metrics, tracing, alerting on lag and reorgs, retries, circuit breakers and reconciliation*

### Goals

- Instrument a chain service with the metrics that matter.
- Alert on the failure modes unique to blockchain.
- Implement retry, backoff, circuit breaking and failover around RPC.
- Design for at-least-once delivery and eventual consistency.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What is different here**

   - Your data source is adversarial, eventually consistent, rate-limited and sometimes simply wrong.
   - Failures are financial and often irreversible — the cost of a missed alert is money, not latency.
   - Time is measured in blocks, not seconds; block time is variable.
   - You cannot roll back the chain, only your view of it.

**2. The core metrics**

   - Head lag: in blocks and in seconds (both, they diverge during outages).
   - Ingest rate (blocks/sec), decode errors, rows written.
   - Reorg count and depth histogram — the number nobody instruments until it bites.
   - RPC latency and error rate, labelled by method and upstream.
   - Pending transaction age; stuck-nonce count; hot-wallet balance.
   - Gas spend per operation and RPC calls per indexed block (cost observability).

**3. Structured logging**

   - `log/slog` with consistent keys: `block`, `tx`, `addr`, `chain_id`, `request_id`.
   - One log line per block at info; per-transaction at debug.
   - A redacting `slog.Handler` that scrubs key material and secrets (lesson 32).
   - Sampling high-volume logs so an incident does not blow up your log bill.

**4. Tracing**

   - OpenTelemetry spans across ingest → decode → persist → notify.
   - Propagating a transaction hash as a trace attribute so you can follow one payment end to end.
   - Span per RPC call with the upstream as an attribute — instantly shows which provider is slow.
   - Sampling strategy: always sample errors and financial operations.

**5. Alerts that are actionable**

   - Lag > N blocks for > M minutes (not instantaneously — blocks are bursty).
   - No new block in X seconds (chain halt or provider outage).
   - Reorg deeper than your confirmation lag — this means data may be wrong.
   - Hot-wallet balance below floor; nonce gap detected; pending transaction older than T.
   - Reconciliation mismatch. Each alert needs a runbook link.

**6. Retries**

   - Idempotency first — a retry is only safe if the operation is.
   - Exponential backoff with full jitter; always bounded by the caller's context deadline.
   - Classify errors: retry 429/5xx/timeout, never retry 'invalid params' or a revert.
   - Budget retries per request so a degraded upstream cannot amplify load.

**7. Circuit breakers and shedding**

   - Open after N consecutive failures; half-open probe on a timer.
   - Per-upstream breakers so one bad provider does not poison the pool.
   - Fail-open vs fail-closed: reads can degrade, signing must not.
   - Load shedding when the queue grows faster than you drain it.

**8. Reconciliation**

   - Periodically re-derive state from the chain and diff it against your database.
   - Examples: indexed ERC-20 balances vs `balanceOf`; internal ledger vs on-chain balances (lesson 60).
   - Run it on a schedule, alert on any nonzero diff, and store the diff for forensics.
   - This is the control that catches the bugs your tests did not.

**9. Runbooks**

   - Stuck transaction, provider outage, deep reorg, chain halt, gas spike, hot wallet drained.
   - Each runbook: symptom, diagnosis commands, mitigation, escalation, post-incident checklist.
   - Keep them next to the code, not in a wiki nobody opens.
   - Rehearse at least the 'provider outage' and 'stuck transaction' ones.

**10. Graceful shutdown**

   - Finish the current block, commit the cursor, then exit.
   - Never lose an in-flight signed transaction — persist before broadcasting (lesson 21).
   - `signal.NotifyContext` + `errgroup` + a drain deadline.
   - Kubernetes `preStop` and termination grace periods (lesson 68).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Alerting on instantaneous lag and drowning in false positives.
- Retrying a non-idempotent send and double-spending.
- No reorg-depth metric, so silent data corruption goes unnoticed for weeks.
- Unbounded retries with no context deadline, turning a blip into an outage.
- Logging full transaction payloads including anything sensitive.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 17).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Export a head-lag gauge to Prometheus from an ingest loop.
- Add `slog` fields for block number and chain id.
- Implement exponential backoff with jitter.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Export reorg count and depth histograms and trigger them with a fake reorg.
- A retry wrapper that classifies errors and honours a context deadline.
- A circuit breaker that opens after 5 failures and half-opens after 30 seconds.
- An OpenTelemetry span per RPC call, labelled by upstream and method.
- A redacting `slog.Handler` and a test proving keys never reach the output.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A reconciliation job diffing indexed ERC-20 balances against on-chain `balanceOf`, with an alert.
- Graceful shutdown that finishes the current block, commits the cursor and drains in-flight sends.
- A synthetic chaos test: kill the upstream mid-ingest and assert no data loss or duplication.

### Packages & tools

`github.com/prometheus/client_golang/prometheus`, `go.opentelemetry.io/otel`, `log/slog`, `context`, `os/signal`, `golang.org/x/sync/errgroup`, `time`

### Resources to cite

- Prometheus Go client: https://pkg.go.dev/github.com/prometheus/client_golang/prometheus
- OpenTelemetry Go: https://opentelemetry.io/docs/languages/go/
- Go log/slog: https://pkg.go.dev/log/slog
- Google SRE Book — alerting: https://sre.google/workbook/alerting-on-slos/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
