# Part 13 — Chain Data at Scale

Chain data as a data-engineering problem: the mempool, decentralized storage, ETL into warehouses, and making Go ingestion fast. Deepens [31](../31-blockchain-indexer.md).

**Extension.** Beyond the core 01–41 spine. Lessons 54–57 · 60 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 54 | [The Mempool from Outside](#54-the-mempool-from-outside) | 20, 21 | 15 |
| 55 | [Decentralized Storage: IPFS, Arweave & NFT Metadata](#55-decentralized-storage-ipfs-arweave-nft-metadata) | 04, 26 | 15 |
| 56 | [Analytics & ETL: Subgraphs, Warehouses & Query APIs](#56-analytics-etl-subgraphs-warehouses-query-apis) | 31 | 15 |
| 57 | [High-Throughput Ingestion & Performance in Go](#57-high-throughput-ingestion-performance-in-go) | 31, 34 | 15 |

---

## 54 — The Mempool from Outside

**Lesson file:** [../54-mempool-from-outside.md](../54-mempool-from-outside.md) · **Examples folder:** `../examples/54-mempool-from-outside/`

| | |
|---|---|
| Prerequisites | [20](../20-json-rpc-ethclient.md), [21](../21-sending-transactions.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*watching pending transactions, gas oracles, simulation, and the realities of the public mempool*

### Goals

- Stream and decode pending transactions from Go.
- Build a gas oracle from mempool and block data.
- Simulate a pending transaction's effect before it lands.
- Understand why the public mempool is neither complete nor fair.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What the mempool is from outside**

   - Not one mempool — every node has its own, and they differ.
   - `newPendingTransactions` subscriptions give hashes (or full transactions on some providers).
   - `txpool_content`/`txpool_status` on geth: pending vs queued, per sender.
   - What you cannot see: private order flow, which is now a large fraction of volume.

**2. Streaming pending transactions**

   - `eth_subscribe("newPendingTransactions")` over WebSocket; hashes then `eth_getTransactionByHash`.
   - The volume problem: thousands per second, most irrelevant. Filter early and cheaply.
   - Filtering by `to`, by selector (first 4 bytes of data), or by sender.
   - Backpressure in Go: a bounded channel and dropping rather than queueing unboundedly.

**3. Decoding intent**

   - Selector → known method; decode calldata (lesson 23) to see what someone is about to do.
   - Router calls, multicalls and nested calldata — you often need to decode two levels.
   - Building a selector registry for the contracts you care about.
   - This is the foundation of monitoring, alerting and MEV searching alike.

**4. Gas oracles**

   - From blocks: `eth_feeHistory` percentiles over the last N blocks — the robust default.
   - From the mempool: the distribution of pending priority fees, and its bias (spam, stuck transactions).
   - Base-fee prediction: the next block's base fee is *deterministic* from the current one; compute it exactly.
   - Building a tiered estimator (slow/standard/fast) in Go with confidence targets.
   - Validating it: record predictions vs actual inclusion delay over a day.

**5. Transaction simulation**

   - `eth_call` at `pending` with the transaction's parameters to preview its effect.
   - State overrides to test 'what if this pending transaction lands first'.
   - `debug_traceCall` for a full trace including internal calls and state diffs.
   - Bundle simulation (`eth_callBundle` on Flashbots-style endpoints) for ordered sequences.

**6. Watching for specific events**

   - Front-run detection: a pending transaction touching a pool you are about to trade in.
   - Liquidation opportunities: a pending oracle update that will move a position under water.
   - Governance and admin actions: a pending `upgradeTo` on a contract you depend on — alert immediately.
   - Your own transactions: detecting that a replacement is needed (lesson 21).

**7. The mempool is not fair or complete**

   - Private RPCs (Flashbots Protect, MEV Blocker) bypass the public mempool entirely.
   - Builders receive order flow you will never see.
   - Regional propagation differences mean your view lags by tens to hundreds of milliseconds.
   - Conclusion: never build a system that *requires* seeing every transaction.

**8. Sending privately**

   - `eth_sendPrivateTransaction` / `eth_sendBundle` to a relay instead of the public mempool.
   - Trade-offs: no public visibility, but you depend on the relay and may wait longer.
   - When to use it: large swaps, liquidations, anything sandwichable, anything time-sensitive.
   - Doing it from Go over raw JSON-RPC with the required signature header.

**9. Operational concerns**

   - A pending-transaction stream is a firehose; measure ingest rate and drop rate.
   - Deduplicate across multiple upstream nodes.
   - Memory bounds: TTL every pending entry, because most never land.
   - This is a lesson where `-race` and profiling both matter.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming you see every transaction — private order flow makes this false.
- Unbounded in-memory maps of pending transactions with no TTL.
- Basing a gas oracle purely on the mempool, which is full of stuck and spam transactions.
- Acting on a pending transaction as if it were confirmed.
- Subscribing without backpressure and OOMing under a volume spike.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Subscribe to pending transaction hashes and count them per second.
- Fetch a pending transaction by hash and print its fields.
- Compute the next block's base fee exactly from the current header.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Filter pending transactions by `to` address and by 4-byte selector.
- Decode a pending router call's calldata to see the intended swap.
- Build a tiered gas estimator from `eth_feeHistory` percentiles.
- Simulate a pending transaction with `eth_call` at pending state.
- Apply backpressure with a bounded channel and report the drop rate.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A gas oracle that records predictions and measures actual inclusion delay over 1000 blocks.
- Detect a pending `upgradeTo` call on a watched proxy and fire an alert within one block.
- Send a transaction through a private relay from Go and compare inclusion with the public mempool.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/rpc`, `github.com/ethereum/go-ethereum/accounts/abi`, `context`, `sync`, `time`

### Resources to cite

- geth txpool API: https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-txpool
- eth_feeHistory: https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_feehistory
- Flashbots Protect: https://docs.flashbots.net/flashbots-protect/overview
- EIP-1559 base fee formula: https://eips.ethereum.org/EIPS/eip-1559

---

## 55 — Decentralized Storage: IPFS, Arweave & NFT Metadata

**Lesson file:** [../55-decentralized-storage.md](../55-decentralized-storage.md) · **Examples folder:** `../examples/55-decentralized-storage/`

| | |
|---|---|
| Prerequisites | [04](../04-hash-functions.md), [26](../26-erc-standards.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*content addressing, CIDs, pinning, gateways, and resolving metadata reliably from Go*

### Goals

- Explain content addressing and compute a CID.
- Fetch and pin content on IPFS from Go.
- Resolve NFT metadata robustly, including the failure cases.
- Compare IPFS, Arweave and on-chain storage honestly.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Content addressing**

   - The name *is* the hash — fetch by what it is, not where it is.
   - Self-verifying: you can check the content matches the address you asked for. Always do.
   - The direct descendant of lesson 04's 'hash as identifier'.
   - Immutability for free, and the mutability problem it creates (solved by IPNS/DNSLink).

**2. CIDs and multiformats**

   - CIDv0 (`Qm…`, base58, implicit dag-pb + sha2-256) vs CIDv1 (`bafy…`, base32, explicit).
   - Multibase, multicodec, multihash — the self-describing prefix stack.
   - Parsing and converting CIDs in Go; converting v0↔v1 for the same content.
   - Why the same file can have different CIDs (chunk size, dag layout, raw leaves).

**3. IPFS in practice**

   - DHT-based peer discovery; content lives only where someone pins it.
   - **Pinning is the whole game** — unpinned content disappears. Pinning services (Pinata, web3.storage) and self-hosting.
   - Gateways: `ipfs.io`, `dweb.link`, and why depending on a public gateway is a reliability bug.
   - Talking to a Kubo node from Go over its HTTP API: `add`, `cat`, `pin/add`, `pin/ls`.

**4. Arweave and permanence**

   - Pay once, stored (in principle) forever — a fundamentally different economic model.
   - Bundlers, transaction anchoring, and gateway retrieval.
   - When permanence is worth the price, and when IPFS + a pin is enough.
   - Fetching Arweave data from Go and verifying the transaction id.

**5. On-chain storage**

   - SSTORE cost makes anything above a few hundred bytes absurd.
   - SVG and JSON generated on-chain (`tokenURI` returning a data URI) — fully self-contained NFTs.
   - Contract code as storage (SSTORE2 pattern), and its read cost.
   - Blobs (EIP-4844) are *not* storage — they are pruned.

**6. NFT metadata resolution**

   - `tokenURI(id)` → a URI, which may be `ipfs://`, `https://`, `ar://`, or a `data:` URI.
   - The metadata JSON: `name`, `description`, `image`, `attributes` — a de facto standard, loosely followed.
   - The image is a *second* fetch, often to a different scheme.
   - ERC-1155's `{id}` substitution rule with a zero-padded 64-hex-char id.
   - Every step can fail, be slow, be huge, or be hostile.

**7. Robust resolution in Go**

   - A scheme-dispatching resolver: `ipfs://` → gateway(s), `ar://` → Arweave, `data:` → decode inline.
   - Multiple gateways with a race or failover; per-request timeouts; total size caps.
   - **Verify the CID** against the fetched bytes — a gateway is untrusted infrastructure.
   - Caching: content-addressed data is immutable, so cache aggressively by CID.
   - Sanitising: metadata is attacker-controlled; bound string lengths, validate URLs, never render raw HTML.

**8. Operational reality**

   - Metadata rot: collections whose gateway or pin vanished.
   - Centralised `https://` metadata that the team can change after mint.
   - Rate limits on public gateways; run your own or pay for one.
   - A background refresh job with backoff and a permanent-failure state.

**9. Where else this shows up**

   - ENS `contenthash` for decentralised websites (lesson 52).
   - Subgraph and rollup data availability.
   - Off-chain order metadata and large calldata alternatives.
   - The general pattern: commit a hash on-chain, store the bytes elsewhere.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Trusting a gateway's response without verifying the CID.
- Depending on one public gateway with no fallback or timeout.
- Fetching metadata with no size cap — a 2GB 'image' will take your service down.
- Rendering metadata strings without sanitisation.
- Assuming `ipfs://` content will still be there next year without a pin you control.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Parse a CID and print its version, codec and multihash.
- Convert a CIDv0 to CIDv1 and back.
- Fetch a known CID through a gateway and print its size.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Compute a CID for local bytes and verify it matches what a node returns.
- Add and pin content via a Kubo node's HTTP API from Go.
- Resolve an NFT's `tokenURI`, fetch the metadata JSON and the image, with timeouts and size caps.
- Handle a `data:` URI `tokenURI` with base64-encoded on-chain JSON.
- Do the ERC-1155 `{id}` substitution correctly with zero padding.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A robust metadata resolver: multi-gateway failover, CID verification, size caps, caching, sanitisation.
- A background refresh worker with exponential backoff and a permanent-failure state per token.
- Compare cost and retrieval reliability of the same 10KB asset on IPFS, Arweave and on-chain SSTORE2.

### Packages & tools

`net/http`, `context`, `github.com/ipfs/go-cid`, `github.com/multiformats/go-multihash`, `encoding/base64`, `io`, `crypto/sha256`

### Resources to cite

- IPFS docs: https://docs.ipfs.tech/
- CID spec: https://github.com/multiformats/cid
- Kubo RPC API: https://docs.ipfs.tech/reference/kubo/rpc/
- Arweave docs: https://docs.arweave.org/
- ERC-1155 metadata URI: https://eips.ethereum.org/EIPS/eip-1155#metadata

---

## 56 — Analytics & ETL: Subgraphs, Warehouses & Query APIs

**Lesson file:** [../56-analytics-etl.md](../56-analytics-etl.md) · **Examples folder:** `../examples/56-analytics-etl/`

| | |
|---|---|
| Prerequisites | [31](../31-blockchain-indexer.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 8 |

*getting chain data into systems that answer questions — The Graph, warehouses, and the build-vs-buy call*

### Goals

- Decide between a subgraph, a hosted API and your own indexer, with reasons.
- Query The Graph from Go and handle its consistency model.
- Design an ETL pipeline into a warehouse for analytics.
- Serve chain data through a paginated, stable API.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The build-vs-buy decision**

   - Hosted APIs (Alchemy/Covalent/Moralis): fastest to ship, least control, per-request cost.
   - Subgraphs (The Graph): declarative, decentralised, good for contract-scoped data.
   - Your own indexer (lesson 31): full control, full operational burden.
   - Dune/Flipside: SQL over someone else's warehouse — great for analysis, not for production reads.
   - A decision table by: latency, cost at scale, custom logic, reorg handling, SLA.

**2. The Graph**

   - subgraph.yaml manifest, GraphQL schema, AssemblyScript mappings from events to entities.
   - Deployment to the decentralised network vs a self-hosted graph-node.
   - Consistency: `_meta { block { number } }` tells you how far behind the subgraph is — always check it.
   - Querying from Go: plain `net/http` POST with a GraphQL body, or a typed client.
   - Limits: no arbitrary computation, no cross-subgraph joins, indexing lag, and reindexing pain.

**3. Warehouse ETL**

   - Bronze/silver/gold layering: raw blocks and logs → decoded events → business entities.
   - Batch loading from your indexer's Postgres into BigQuery/Snowflake/ClickHouse.
   - Partitioning by block number or date; clustering by address.
   - Late-arriving and *revised* data — reorgs mean rows change, which most ETL assumes never happens.
   - Handling that: block-scoped partitions you can drop and rewrite.

**4. Public datasets**

   - Google BigQuery's public Ethereum dataset; Dune's decoded tables.
   - When to use them: backfills, research, one-off analysis.
   - When not to: anything with an SLA or sub-hour freshness.
   - Querying BigQuery from Go for a bulk historical backfill.

**5. Decoding at scale**

   - A signature registry (4byte + event topics) to decode contracts you have no ABI for.
   - Storing raw and decoded side by side so a decoder bug is recoverable.
   - Handling proxies: decode with the implementation's ABI, attribute to the proxy address (lesson 46).
   - Batch re-decoding as a background job when you add an ABI.

**6. Serving the data**

   - Cursor (keyset) pagination on (block_number, log_index) — never OFFSET at chain scale.
   - Stable ordering and a documented consistency guarantee ('data up to block N').
   - Expose the indexed-block height in every response so clients can reason about freshness.
   - Caching by block height: immutable below the finalized head, so cache aggressively.

**7. Aggregations**

   - Materialised views vs on-the-fly aggregation vs a pre-computed rollup table.
   - Reorg-safe aggregation: fold over block-scoped deltas, never mutate a counter.
   - Time-bucketed metrics (hourly volume, daily active addresses) and their refresh strategy.
   - ClickHouse's aggregating merge trees as a good fit for this shape.

**8. Data quality**

   - Completeness checks: every block present, every transaction accounted for.
   - Cross-validation against a second source (a subgraph, a public dataset, direct RPC).
   - Reconciliation as a scheduled job with alerts (lesson 35).
   - Freshness SLOs and how to publish them honestly.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Querying a subgraph without checking `_meta` and serving hours-stale data as live.
- OFFSET pagination over millions of rows.
- ETL that assumes append-only, then silently keeping reorged-out rows forever.
- Aggregating with in-place counters that cannot be rolled back.
- Building on Dune/Flipside for a production read path.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Query a public subgraph from Go and print the results.
- Read `_meta` and report the subgraph's indexing lag.
- Implement keyset pagination over (block_number, log_index).

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build a GraphQL query with variables and decode it into typed Go structs.
- Load decoded events from Postgres into a partitioned analytics table.
- Rewrite a block-range partition after a simulated reorg and verify correctness.
- Serve a paginated transfers endpoint with a documented `indexed_to_block` field.
- Cross-validate your indexed totals against a subgraph and report the diff.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A bronze/silver/gold pipeline in Go with reorg-safe partition rewrites.
- A signature registry that decodes unknown contract calls and events, with a re-decode backfill job.
- A completeness checker that finds and repairs gaps across a 1M-block range.

### Packages & tools

`net/http`, `encoding/json`, `database/sql`, `github.com/jackc/pgx/v5`, `context`, `golang.org/x/sync/errgroup`

### Resources to cite

- The Graph docs: https://thegraph.com/docs/
- BigQuery public crypto datasets: https://cloud.google.com/blog/products/data-analytics/ethereum-bigquery-public-dataset-smart-contract-analytics
- Dune docs: https://docs.dune.com/
- ClickHouse: https://clickhouse.com/docs

---

## 57 — High-Throughput Ingestion & Performance in Go

**Lesson file:** [../57-high-throughput-ingestion.md](../57-high-throughput-ingestion.md) · **Examples folder:** `../examples/57-high-throughput-ingestion/`

| | |
|---|---|
| Prerequisites | [31](../31-blockchain-indexer.md), [34](../34-testing-blockchain-go.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*making the indexer fast — profiling, batching, bounded concurrency, allocation control and database throughput*

### Goals

- Profile a Go ingestion pipeline and find the real bottleneck.
- Design a bounded, ordered concurrent pipeline.
- Cut allocations in the hot decode path.
- Push a database hard without losing correctness.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Measure first**

   - `pprof` CPU, heap, block and mutex profiles; `go test -bench` with `-benchmem`.
   - `runtime/trace` for goroutine scheduling and blocking analysis.
   - The usual answer: you are network-bound on RPC, not CPU-bound. Prove it before optimising.
   - Establishing a baseline metric (blocks/sec) and a target before touching anything.

**2. The pipeline shape**

   - fetch → decode → transform → write, as stages connected by bounded channels.
   - Bounded channels give backpressure for free; unbounded queues just move the failure.
   - `errgroup.WithContext` + `SetLimit` for bounded concurrency with error propagation.
   - Ordered commit despite out-of-order fetch: a reorder buffer keyed by block number.

**3. Concurrency without corruption**

   - Fetch in parallel, write sequentially — the safest split for a chain indexer.
   - Why parallel writes across blocks break reorg handling.
   - `-race` on every concurrent path; a race in an indexer is silent data corruption.
   - Worker pool vs per-item goroutine: measure, do not assume.

**4. RPC throughput**

   - Batch requests (lesson 20) — the single biggest win, often 5–10×.
   - Connection pooling: `http.Transport` `MaxIdleConnsPerHost` (the default of 2 is a common bottleneck).
   - Multiple upstreams in parallel with per-upstream rate limits (lesson 33).
   - Compression, and whether the provider supports it.
   - Caching immutable data: blocks below the finalized head never change.

**5. Allocation control**

   - Escape analysis with `-gcflags=-m`; the allocations you did not know about.
   - `sync.Pool` for decode buffers; reusing `[]byte` and hashers.
   - Avoiding `[]interface{}` in hot paths; `uint256` over `big.Int` (lesson 03).
   - Preallocating slices with known capacity; `strings.Builder` over concatenation.
   - Measure with `-benchmem`; a 90% allocation reduction is normal and often halves latency.

**6. GC tuning**

   - `GOGC` and `GOMEMLIMIT` — what each does and when to touch them (rarely).
   - Fewer pointers means less scan work: value slices and indices over pointer graphs.
   - Large in-memory caches and their GC cost.
   - When the right fix is 'allocate less', not 'tune the GC'.

**7. Database throughput**

   - `COPY` (pgx `CopyFrom`) for bulk insert — orders of magnitude faster than INSERT.
   - Batched multi-row upserts when you need conflict handling.
   - Transaction sizing: too small is slow, too large blocks and bloats WAL.
   - Index maintenance cost on write; dropping and rebuilding indexes for a big backfill.
   - Connection pool sizing (`pgxpool`) and why more connections is usually slower.

**8. Caching**

   - What is safe to cache forever: finalized blocks, receipts, ABIs, decoded static data.
   - What is never safe: anything at `latest`, balances, pending state.
   - An LRU with size bounds; measuring hit rate and evictions.
   - Cache stampede protection with `singleflight`.

**9. Benchmarking honestly**

   - A reproducible harness: fixed block range, recorded RPC responses, fixed database state.
   - Report p50/p95/p99, not the mean; report allocations and bytes.
   - `benchstat` for before/after comparisons with statistical significance.
   - Regression benchmarks in CI so a fix does not silently undo itself.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Optimising the decoder when you are RPC-bound.
- Unbounded goroutines per block — thousands of in-flight requests and instant rate limiting.
- Parallel database writes across blocks, breaking reorg rollback.
- Leaving `MaxIdleConnsPerHost` at its default of 2.
- Reporting mean latency and hiding the tail.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Benchmark a log decoder with `-benchmem` and record the baseline.
- Profile an ingest loop with pprof and identify the top consumer.
- Bound concurrency with `errgroup.SetLimit`.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Batch RPC receipt fetches and measure the speedup against sequential.
- Add a reorder buffer so parallel fetch commits in block order.
- Cut decoder allocations by 80% with `sync.Pool` and preallocation; prove it with `benchstat`.
- Bulk-insert 100k rows with `CopyFrom` and compare against batched INSERT.
- Tune `MaxIdleConnsPerHost` and measure the throughput change.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A full pipeline hitting a target blocks/sec with bounded memory, verified under `-race`.
- An LRU cache with `singleflight` stampede protection; report hit rate and p99 improvement.
- A CI benchmark gate that fails when ingestion throughput regresses more than 10%.

### Packages & tools

`runtime/pprof`, `runtime/trace`, `sync`, `golang.org/x/sync/errgroup`, `golang.org/x/sync/singleflight`, `github.com/jackc/pgx/v5/pgxpool`, `testing`

### Resources to cite

- Go — Profiling Go programs: https://go.dev/blog/pprof
- Go — Diagnostics: https://go.dev/doc/diagnostics
- benchstat: https://pkg.go.dev/golang.org/x/perf/cmd/benchstat
- pgx CopyFrom: https://pkg.go.dev/github.com/jackc/pgx/v5#Conn.CopyFrom
- Go GC guide: https://go.dev/doc/gc-guide

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
