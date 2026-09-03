# Part 14 — Custody, Payments & Compliance

What an exchange, PSP or wallet company actually builds: deposit detection, sweeps, withdrawals, double-entry accounting and the regulatory surface. Deepens [32](../32-key-management-signing.md).

**Extension.** Beyond the core 01–41 spine. Lessons 58–61 · 58 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 58 | [Deposit Detection & Address Management](#58-deposit-detection-address-management) | 31, 32 | 15 |
| 59 | [Withdrawals, Sweeping & Fee Management](#59-withdrawals-sweeping-fee-management) | 21, 32, 58 | 15 |
| 60 | [Ledger & Accounting for Crypto](#60-ledger-accounting-for-crypto) | 58, 59 | 15 |
| 61 | [Compliance: AML, Sanctions & the Travel Rule](#61-compliance-aml-sanctions-the-travel-rule) | 58, 60 | 13 |

---

## 58 — Deposit Detection & Address Management

**Lesson file:** [../58-deposit-detection.md](../58-deposit-detection.md) · **Examples folder:** `../examples/58-deposit-detection/`

| | |
|---|---|
| Prerequisites | [31](../31-blockchain-indexer.md), [32](../32-key-management-signing.md) |
| Unlocks | 59, 60, 61 |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 8 |

*per-user addresses, detecting incoming value of every kind, confirmations and crediting exactly once*

### Goals

- Assign deposit addresses to users safely and at scale.
- Detect native, token and internal-transfer deposits reliably.
- Apply a confirmation policy and credit exactly once.
- Handle the awkward cases: unexpected tokens, dust, wrong-chain sends.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Address assignment strategies**

   - Per-user HD-derived addresses from a watch-only xpub (lesson 07) — spending keys stay cold.
   - A single shared address with a memo/tag (used by some chains, dangerous on Ethereum).
   - Contract-per-user (CREATE2 deposit proxies) — gas cost vs sweep efficiency.
   - The gap limit problem: derive ahead, monitor a window, extend as addresses are used.
   - Never reuse an address across users; never derive on the fly without persisting the index.

**2. Detecting native transfers**

   - A plain ETH transfer produces **no log**. You must scan transactions, not logs.
   - Worse: a contract can send ETH internally with no transaction to your address at all.
   - Options: trace every block (`debug_traceBlock`/`trace_block`), or poll `getBalance` per address.
   - Balance polling scales badly; tracing needs a provider that supports it. Discuss the trade honestly.
   - The pragmatic hybrid: trace when available, balance-diff as a safety net.

**3. Detecting token deposits**

   - ERC-20 `Transfer` logs filtered by `to` in your address set (lesson 25).
   - Topic filtering with a large address set: chunk the OR list, or filter client-side.
   - ERC-721/1155 deposits, and whether you even want to accept them.
   - **Fee-on-transfer and rebasing tokens**: credit the measured balance delta, never the log's amount.

**4. Confirmation policy**

   - Per-chain, per-amount confirmation depths; a small deposit can credit sooner than a large one.
   - Provisional credit (visible, unusable) vs final credit (withdrawable).
   - Reorg handling: if a deposit's block is reorged out, the credit must be reversed (lesson 31).
   - Never credit at zero confirmations, whatever the product team says.

**5. Crediting exactly once**

   - Idempotency key: (chain_id, block_hash, tx_hash, log_index) or (tx_hash, trace_address).
   - Write the credit and the cursor in one database transaction.
   - A ledger entry, not a balance update (lesson 60).
   - Test it by replaying the same block ten times and asserting one credit.

**6. The awkward cases**

   - Tokens you do not support arriving at a deposit address — detect, record, do not credit.
   - Dust deposits that cost more to sweep than they are worth.
   - Deposits from contracts and from smart accounts (lesson 47) — the 'from' may be a bundler.
   - Deposits sent on the wrong chain to the same address — recoverable only if you control the key on that chain too.
   - Self-destructs and `SELFDESTRUCT` force-sends that no trace shows cleanly.

**7. Monitoring at scale**

   - Millions of addresses: an in-memory set with a bloom prefilter, refreshed from the database.
   - Address-set updates must be atomic with respect to the ingest loop.
   - Per-block cost: one `getLogs` for tokens, one trace or balance sweep for native.
   - Metrics: deposits detected, credit latency, unknown-token arrivals, reorg reversals.

**8. Reconciliation**

   - Periodically sum on-chain balances of all deposit addresses and compare with credited totals.
   - Any divergence is either a missed deposit or a bug — both are alerts.
   - Store the reconciliation result as a time series so drift is visible.
   - This job is what saves you when the detection logic has a subtle hole.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Detecting ETH deposits from logs — there are none; you will miss every one.
- Crediting the log's `value` for a fee-on-transfer token and over-crediting the user.
- Reusing deposit addresses across users, making attribution impossible.
- Deriving addresses without persisting the index and losing track of user funds.
- Crediting before confirmations and being drained by a reorg.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Derive 100 deposit addresses from a watch-only xpub.
- Filter `Transfer` logs by a set of `to` addresses.
- Apply a fixed confirmation depth before crediting.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Detect a native ETH deposit using `trace_block` and attribute it to a user.
- Detect the same deposit using balance diffing and compare the two methods.
- Credit exactly once with a (block_hash, log_index) idempotency key; replay the block and assert one credit.
- Credit a fee-on-transfer token by measured balance delta rather than the log amount.
- Reverse a credit when its block is reorged out.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A deposit monitor for 1M addresses using a bloom prefilter, with atomic address-set refresh.
- A tiered confirmation policy by amount, with provisional and final credit states.
- A reconciliation job summing on-chain deposit-address balances against credited totals, with alerting.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/rpc`, `database/sql`, `github.com/btcsuite/btcd/btcutil/hdkeychain`, `math/big`

### Resources to cite

- EIP-20 Transfer event: https://eips.ethereum.org/EIPS/eip-20
- geth tracing API: https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-debug
- BIP-32 (HD derivation): https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki

---

## 59 — Withdrawals, Sweeping & Fee Management

**Lesson file:** [../59-withdrawals-sweeping.md](../59-withdrawals-sweeping.md) · **Examples folder:** `../examples/59-withdrawals-sweeping/`

| | |
|---|---|
| Prerequisites | [21](../21-sending-transactions.md), [32](../32-key-management-signing.md), [58](../58-deposit-detection.md) |
| Unlocks | 60 |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*moving value out safely — sweeps, batching, gas funding, approvals and the withdrawal state machine*

### Goals

- Design a withdrawal pipeline that never double-pays and never loses a request.
- Sweep deposit addresses efficiently, including the gas-funding problem.
- Batch withdrawals to cut cost, with correct failure attribution.
- Manage hot-wallet float and fee budgets.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The withdrawal state machine**

   - requested → approved → queued → signed → broadcast → mined → confirmed → settled. Plus failed and cancelled.
   - Every transition persisted before the side effect, so a crash is always recoverable.
   - The critical invariant: at most one broadcast per withdrawal, and the nonce is recorded with it.
   - Idempotency at the API edge: a client-supplied request id that dedupes retries.

**2. Never double-pay**

   - Sign once, store the raw bytes, re-broadcast the *same* bytes on retry (lesson 21).
   - A unique constraint on (user, request_id) and on the assigned nonce.
   - Reconciling 'unknown' states: a transaction you broadcast but whose fate you do not know — query, do not resend a new one.
   - The doomsday scenario to design against: a partial failure that re-signs with a fresh nonce.

**3. Approval workflows**

   - Automatic below a threshold; manual (or multi-party) above it.
   - Destination allowlists, velocity limits, and cool-off periods for new destinations.
   - The signing service enforces policy, not the application (lesson 32).
   - An audit trail that survives a legal review.

**4. Sweeping deposit addresses**

   - Deposits land on per-user addresses; funds must be consolidated to a hot or cold wallet.
   - **The gas-funding problem**: an ERC-20 deposit address has no ETH, so it cannot pay for its own sweep.
   - Options: fund each address with dust ETH first (two transactions), or use CREATE2 deposit contracts, or ERC-4337 paymasters (lesson 47).
   - Sweep thresholds: do not sweep when the gas cost exceeds the value.
   - Batching sweeps in one transaction via a multicall-style helper contract.

**5. Batching withdrawals**

   - A disperse-style contract paying many recipients in one transaction.
   - Cost per recipient falls sharply; the trade is failure granularity — one bad recipient can revert the batch.
   - Mitigations: pre-validate recipients, use `try/catch` per transfer, or split the batch.
   - **Attribution**: you must be able to tell exactly which recipients were paid from the receipt's events.

**6. Fee management**

   - A gas budget per withdrawal and a policy for who absorbs fee spikes.
   - Dynamic fee selection with a maximum; queueing rather than overpaying during a spike.
   - Fee estimation for batches, and the headroom to add.
   - Tracking realised fee cost per withdrawal for accounting (lesson 60).

**7. Hot wallet float**

   - Keeping enough in the hot wallet to serve withdrawals, and not much more.
   - Automatic top-ups from warm storage when below a floor, with approval.
   - Alerting on low balance *before* withdrawals start failing.
   - The maximum-loss calculation that sets the float size.

**8. Token approvals**

   - If a helper contract moves your tokens, it needs an allowance. Never infinite from a hot wallet.
   - Approve exactly, per batch; or use Permit2 (lesson 49) to avoid standing approvals.
   - Monitor and revoke stale approvals.
   - Every approval is a standing liability — inventory them.

**9. Stuck and failed withdrawals**

   - Detect: pending age exceeds a threshold. Act: bump fees, then cancel (lesson 21).
   - A reverted withdrawal transaction still consumed gas — record the cost and retry as a new attempt.
   - Recipient contracts that reject transfers; blocklisted addresses (USDC/USDT freeze).
   - The runbook: what an operator does at 3am when the queue stops draining.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Re-signing with a new nonce on retry — the classic double-payment.
- Sweeping without a value threshold and paying more in gas than you move.
- Batching without per-recipient attribution, so a partial failure is unauditable.
- Standing infinite approvals from a hot wallet to a helper contract.
- No low-balance alert on the hot wallet, so withdrawals fail silently.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Model the withdrawal state machine as a Go type with valid transitions.
- Enforce idempotency on a request id.
- Compute whether a sweep is worth its gas cost.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Persist a signed withdrawal before broadcasting and re-broadcast identical bytes after a crash.
- Fund a deposit address with gas, then sweep its ERC-20 balance.
- Batch 50 withdrawals through a disperse contract and attribute each from the receipt's events.
- Detect a stuck withdrawal by pending age and bump its fees.
- Enforce a destination allowlist and a per-window value cap at the signing boundary.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A full withdrawal pipeline with crash injection at every state transition, proving exactly-once payment.
- A sweeper that batches, respects thresholds, and handles the gas-funding problem via CREATE2 deposit contracts.
- Hot-wallet float management with automatic top-up requests and alerting.

### Packages & tools

`database/sql`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/core/types`, `context`, `sync`, `log/slog`

### Resources to cite

- Ethereum transactions: https://ethereum.org/en/developers/docs/transactions/
- Permit2: https://github.com/Uniswap/permit2
- EIP-1014 CREATE2: https://eips.ethereum.org/EIPS/eip-1014

---

## 60 — Ledger & Accounting for Crypto

**Lesson file:** [../60-ledger-accounting.md](../60-ledger-accounting.md) · **Examples folder:** `../examples/60-ledger-accounting/`

| | |
|---|---|
| Prerequisites | [58](../58-deposit-detection.md), [59](../59-withdrawals-sweeping.md) |
| Unlocks | 61 |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*double-entry bookkeeping for on-chain value — the design that makes your numbers provable*

### Goals

- Design a double-entry ledger for multi-asset, multi-chain balances.
- Represent every on-chain event as balanced journal entries.
- Reconcile internal balances against on-chain reality.
- Handle the hard cases: fees, rebasing, wrapped assets, failed transactions.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why double-entry**

   - A single `balance` column is unauditable and impossible to debug after the fact.
   - Double entry: every event produces entries that sum to zero. The invariant is checkable, continuously.
   - Balance becomes a *derived* value: the sum of a account's entries.
   - This is 500-year-old technology and it is still the correct answer.

**2. The data model**

   - `accounts` (id, type, asset, owner) and `entries` (id, journal_id, account_id, amount, asset).
   - A journal groups entries from one event; `SUM(amount) = 0` per journal, per asset.
   - Account types: user, hot wallet, cold wallet, fees, external (the counterparty for on-chain movement).
   - Immutable, append-only entries. Corrections are new entries, never updates.
   - Postgres types: NUMERIC(78,0) for raw amounts, plus the asset's decimals separately.

**3. Modelling on-chain events**

   - Deposit: debit external, credit user. Withdrawal: debit user, credit external, debit user for fee.
   - Sweep: an internal transfer between two of your own accounts — net zero to users.
   - Trade: two journals or one multi-asset journal, depending on your model.
   - **Gas fees are a real expense** and must be recorded, including for failed transactions.
   - A reverted transaction still costs gas — the journal records the fee with no value movement.

**4. Multi-asset and multi-chain**

   - An asset is (chain_id, contract_address) or (chain_id, native) — USDC on Ethereum ≠ USDC on Polygon.
   - Never net across assets; the zero-sum invariant is per asset, per journal.
   - Decimals belong to the asset definition; raw integer amounts are the only stored values.
   - Bridged assets: the bridge is a counterparty account, and in-flight value is a real balance.

**5. The hard cases**

   - Rebasing tokens (stETH): balance changes with no event — record a periodic rebase journal.
   - Wrapped/reward-bearing (wstETH, ERC-4626 shares): track shares, value them separately.
   - Fee-on-transfer: the amount received ≠ the amount sent; both entries must reflect reality.
   - Airdrops and unexpected tokens: a journal crediting an 'unallocated' account.
   - Slashing, negative yield, and other value *decreases* that are not transfers.

**6. Reconciliation**

   - For every asset: sum of internal entries must equal on-chain balances of the addresses you control.
   - Run it on a schedule, store the result, and alert on any nonzero difference.
   - Break it down by address so a diff points somewhere.
   - Two independent code paths (ledger vs chain scan) checking each other — that is the point.

**7. Point-in-time and reporting**

   - Balance at any block: sum entries up to that block height. This is why entries carry block numbers.
   - Trial balance, and proof-of-reserves style reports.
   - Cost basis and realised P&L in fiat — needs a price oracle and a lot-tracking method (FIFO/LIFO).
   - Immutable period closes, and how to handle a correction after close.

**8. Idempotency and ordering**

   - The journal's external key is the chain event's idempotency key (lesson 58).
   - Ordering by (block_number, log_index) so replays are deterministic.
   - Reorg reversal: post a *reversing journal*, do not delete — the audit trail must show what happened.
   - This differs from lesson 31's indexer rollback, and the difference matters for auditors.

**9. Implementing it in Go**

   - A `Journal` type that refuses to persist unless entries balance per asset — enforce in code and with a database constraint.
   - Property tests: for any sequence of random operations, every asset's total is zero and no user balance is negative.
   - Serializable isolation or explicit row locks for concurrent posting to the same account.
   - Performance: entries grow forever; partition by month and keep a materialised balance snapshot.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- A mutable `balance` column — the root of every unexplainable discrepancy.
- Forgetting to record gas fees, so your books never match the chain.
- Netting across assets or across chains.
- Deleting entries on a reorg instead of posting reversals.
- Using floats or BIGINT for amounts.
- No reconciliation job, so drift is discovered by a customer.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Model accounts and entries as Go types with a balance-check method.
- Post a deposit journal and assert it sums to zero.
- Derive an account balance by summing entries.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Post deposit, sweep and withdrawal journals for one user and verify every invariant.
- Record a gas fee for a *failed* withdrawal transaction.
- Compute a user's balance at a historical block height.
- Post a reversing journal for a reorged deposit and show the audit trail.
- Enforce the balance invariant with a database constraint and prove it rejects a bad journal.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A reconciliation job comparing ledger totals against on-chain balances per asset, with a per-address breakdown.
- Property-test the ledger: random operation sequences never break zero-sum or produce a negative user balance.
- Handle a rebasing token: periodic rebase journals plus reconciliation that accounts for them.

### Packages & tools

`database/sql`, `github.com/jackc/pgx/v5`, `math/big`, `time`, `testing`

### Resources to cite

- Martin Fowler — Accounting patterns: https://martinfowler.com/eaaDev/AccountingNarrative.html
- Postgres NUMERIC: https://www.postgresql.org/docs/current/datatype-numeric.html
- Postgres transaction isolation: https://www.postgresql.org/docs/current/transaction-iso.html

---

## 61 — Compliance: AML, Sanctions & the Travel Rule

**Lesson file:** [../61-compliance-aml.md](../61-compliance-aml.md) · **Examples folder:** `../examples/61-compliance-aml/`

| | |
|---|---|
| Prerequisites | [58](../58-deposit-detection.md), [60](../60-ledger-accounting.md) |
| Unlocks | — |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 9 |

*the regulatory surface a custodial service must implement, and the Go code that implements it*

### Goals

- Explain the compliance obligations a custodial crypto service typically faces.
- Implement sanctions screening against on-chain addresses.
- Integrate a transaction-risk provider and act on its scores.
- Understand the Travel Rule and what it requires you to transmit.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Scope and honesty**

   - This lesson is engineering, not legal advice. Requirements vary by jurisdiction and change often.
   - The point: understand the *shape* of the obligations so you can build systems that satisfy them.
   - Who is in scope: custodial wallets, exchanges, payment processors — 'VASPs'.
   - Who is usually not: non-custodial wallets, node operators, protocol developers. Usually.

**2. KYC and identity**

   - Customer identification and verification at onboarding; risk-based due diligence tiers.
   - Where blockchain differs: you must link an off-chain identity to on-chain addresses.
   - Data minimisation and retention — you are now holding sensitive PII (encrypt it, lesson 44).
   - The engineering artefacts: a customer record, a verification state machine, an audit log.

**3. Sanctions screening**

   - OFAC SDN list includes **specific crypto addresses** — published as machine-readable data.
   - Screen: deposit senders, withdrawal destinations, and your own counterparties, continuously.
   - Downloading and parsing the SDN list in Go; keeping it fresh; alerting when it changes.
   - Tornado Cash and the sanctioning of a *contract* — the precedent that changed everyone's design.
   - Direct exposure vs indirect (n hops away) — you need a policy for both.

**4. Transaction monitoring and risk scoring**

   - Providers (Chainalysis, TRM, Elliptic) score addresses and flows by exposure to illicit sources.
   - Integrating a scoring API from Go: cache, rate limit, degrade safely, and never fail-open on a block decision.
   - Acting on scores: allow / review / block, with a human queue for review.
   - False positives are expensive for users; design the review path as a first-class feature.

**5. The Travel Rule**

   - FATF Recommendation 16: originator and beneficiary information must travel with transfers above a threshold.
   - Between VASPs, off-chain, over protocols like TRP, OpenVASP, or IVMS101-formatted messages.
   - The counterparty-discovery problem: which VASP owns the destination address?
   - Engineering: an outbound message pipeline with retries, and an inbound handler — plus what to do when the counterparty is unknown.

**6. Address ownership proofs**

   - Proving a customer controls a self-hosted wallet: a signed message (lesson 50) is the standard method.
   - The Address Ownership Proof Protocol (AOPP) and its controversy.
   - Satoshi test (micro-deposit) as an alternative.
   - Implementing signature-based proof of control in Go, with replay protection.

**7. Reporting and record-keeping**

   - Suspicious activity reporting, thresholds, and retention periods (typically 5+ years).
   - The records you must be able to produce: who, what, when, which addresses, which transactions.
   - This is exactly what lesson 60's immutable ledger gives you — that is not a coincidence.
   - Export formats and the ability to answer a regulator's question in a day, not a month.

**8. Privacy in tension**

   - Every control above conflicts with user privacy; state that honestly.
   - Data minimisation, encryption at rest, access controls, and deletion policy where permitted.
   - GDPR-style rights vs immutable ledgers — the pointer/crypto-shredding pattern.
   - Privacy tools (mixers, privacy chains) and the compliance position on them.

**9. Building it in Go**

   - A `ComplianceCheck` interface in the withdrawal path (lesson 59) — allow/review/block plus a reason.
   - Fail-closed for blocking decisions, with an explicit break-glass procedure.
   - Screening runs at request time *and* continuously (lists change; a clean address can become sanctioned).
   - Metrics and alerts: screening latency, block rate, review queue depth, list staleness.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Fail-open on a sanctions check when the provider is down.
- Screening only at onboarding and never re-screening as lists change.
- Storing PII unencrypted next to chain data.
- Treating a risk score as a verdict with no human review path.
- Assuming one jurisdiction's rules apply everywhere you operate.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Download and parse the OFAC SDN list and extract crypto addresses.
- Screen an address against the parsed list.
- Define a `ComplianceCheck` interface with allow/review/block outcomes.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Keep the sanctions list fresh with scheduled refresh and change alerting.
- Integrate a risk-scoring API with caching, rate limiting and fail-closed behaviour.
- Wire the compliance check into the withdrawal state machine as a blocking gate.
- Verify proof of address control via an EIP-191 signature with replay protection.
- Build a review queue with an audit trail of every decision.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Continuous re-screening of all customer addresses with alerting when a previously clean address is listed.
- An IVMS101-shaped Travel Rule message pipeline with retries and an inbound handler.
- A regulator-report exporter over the lesson 60 ledger: all activity for one customer over a date range.

### Packages & tools

`net/http`, `encoding/xml`, `encoding/json`, `database/sql`, `context`, `github.com/ethereum/go-ethereum/crypto`, `log/slog`

### Resources to cite

- OFAC SDN list (data files): https://sanctionslist.ofac.treas.gov/Home/SdnList
- FATF Recommendation 16 / Travel Rule: https://www.fatf-gafi.org/en/publications/Fatfrecommendations/Targeted-update-virtual-assets-vasps-2024.html
- IVMS101 standard: https://www.intervasp.org/
- OFAC Tornado Cash action: https://home.treasury.gov/news/press-releases/jy0916

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
