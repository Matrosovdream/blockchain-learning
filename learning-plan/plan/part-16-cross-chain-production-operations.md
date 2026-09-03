# Part 16 — Cross-Chain & Production Operations

Moving value between chains, and running all of it in production without losing funds.

**Extension.** Beyond the core 01–41 spine. Lessons 67–68 · 26 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 67 | [Bridges & Cross-Chain Messaging](#67-bridges-cross-chain-messaging) | 30, 64 | 13 |
| 68 | [Deploying & Operating Chain Services in Production](#68-deploying-operating-chain-services-in-production) | 35, 32 | 13 |

---

## 67 — Bridges & Cross-Chain Messaging

**Lesson file:** [../67-bridges-cross-chain.md](../67-bridges-cross-chain.md) · **Examples folder:** `../examples/67-bridges-cross-chain/`

| | |
|---|---|
| Prerequisites | [30](../30-layer2-scaling.md), [64](../64-light-clients-spv.md) |
| Unlocks | — |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 9 |

*moving value and messages between chains — the designs, the proofs, and why bridges keep getting drained*

### Goals

- Classify bridge designs by their trust assumptions.
- Explain how a canonical rollup bridge proves an L2 state on L1.
- Describe the major bridge hacks and the root cause of each.
- Build a Go service that tracks a cross-chain transfer end to end.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The fundamental problem**

   - Chain A cannot read chain B. Every bridge is an oracle for the other chain's state (lesson 53).
   - So the question is never 'how does it work' but 'who do I trust, and what happens if they lie'.
   - The three families: externally verified, optimistically verified, natively verified.
   - Bridges hold concentrated value with a single verification assumption — hence the hack record.

**2. Lock-mint, burn-mint and liquidity**

   - Lock-and-mint: lock on A, mint a wrapped representation on B. The wrapped token is an IOU.
   - Burn-and-mint: for tokens with a native issuer on both chains (e.g. CCTP for USDC).
   - Liquidity networks: no minting; a market maker fronts funds on the destination and rebalances later.
   - Consequences for your accounting: a wrapped asset is a distinct asset (lesson 60).

**3. Externally verified bridges**

   - A multisig or MPC committee attests that something happened on the source chain.
   - Fast and chain-agnostic; the security is exactly the committee's honesty and key hygiene.
   - Ronin (2022, $624M): 5 of 9 validator keys compromised.
   - Harmony (2022, $100M): 2-of-5 multisig. Wormhole (2022, $326M): a signature-verification bug.
   - The pattern: the verification assumption was much weaker than the value secured.

**4. Natively verified bridges**

   - A light client of chain A runs on chain B and verifies its consensus (lesson 64).
   - IBC (lesson 37) is the mature example; Ethereum ↔ rollup canonical bridges are another.
   - Trust-minimised: no committee, only the source chain's consensus.
   - Cost: expensive on-chain verification, and hard between chains with different consensus.

**5. Rollup canonical bridges**

   - Deposits: an L1 transaction enqueues a message that the L2 derives — trust-minimised by construction.
   - Withdrawals (optimistic): initiate on L2, wait the challenge window, prove and finalize on L1.
   - The withdrawal proof is a storage proof against the L2 output root posted to L1 (lessons 17, 64).
   - Withdrawals (zk): a validity proof removes the challenge window.
   - Implementing the three-step withdrawal flow (initiate → prove → finalize) in Go.

**6. Optimistically verified bridges**

   - Nomad, Across: assert a message, allow a challenge window, slash on fraud.
   - Nomad (2022, $190M): a botched upgrade made *every* message provable — and it became a free-for-all.
   - The lesson: an upgrade to a bridge is a security event, and initialization bugs are catastrophic.
   - Bonded relayers and the economics of honest challenging.

**7. General message passing**

   - Beyond tokens: arbitrary calls across chains (LayerZero, CCIP, Axelar, Hyperlane).
   - Modular security: some let you choose or stack verification modules.
   - Replay protection and message ordering across chains.
   - The receiving contract must authenticate the sender — a very common bug.

**8. Tracking transfers in Go**

   - A cross-chain transfer is a state machine spanning two indexers.
   - Correlating source and destination events by a message id or nonce.
   - Timeouts and stuck states: funds locked on A with nothing minted on B.
   - Reorgs on the source chain after the destination has acted — the nightmare scenario, and the confirmation depth that prevents it.
   - Metrics: in-flight value, median completion time, stuck-transfer count.

**9. Evaluating a bridge before you integrate**

   - Who can mint on the destination? Who can upgrade the contracts? Is there a timelock?
   - What is the verification assumption, stated in one sentence?
   - What is the value secured vs the cost to compromise the verifier?
   - L2Beat and similar risk frameworks; and the answer 'do not bridge' as a legitimate option.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Treating a wrapped asset as the same asset in your ledger.
- Acting on the destination chain before the source chain's transfer is final.
- Integrating a bridge without knowing who holds the minting keys.
- A receiving contract that does not authenticate the cross-chain sender.
- No timeout handling, so stuck transfers accumulate silently.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Classify three real bridges by their verification model.
- Read a rollup's L1 bridge contract state from Go.
- Track a deposit from L1 to an L2 by correlating events.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Implement the three-step optimistic withdrawal flow (initiate, prove, finalize) against a testnet.
- Verify an L2 withdrawal storage proof against the output root posted on L1.
- Correlate source and destination events by message id across two indexers.
- Detect and alert on a stuck transfer past its expected completion time.
- Model a wrapped asset correctly in a double-entry ledger (lesson 60).

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A cross-chain transfer tracker: two indexers, one state machine, timeouts, metrics and reorg safety.
- Reproduce the Nomad root cause in a simplified contract and show why every message became provable.
- A bridge risk report generator that reads on-chain config (owners, timelocks, thresholds) for a given bridge.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/trie`, `database/sql`, `context`, `golang.org/x/sync/errgroup`

### Resources to cite

- L2Beat bridge risk: https://l2beat.com/bridges/summary
- OP Stack withdrawal flow: https://docs.optimism.io/stack/protocol/withdrawal-flow
- IBC: https://ibc.cosmos.network/
- Chainlink — cross-chain security: https://docs.chain.link/ccip/concepts
- rekt.news bridge incidents: https://rekt.news/

---

## 68 — Deploying & Operating Chain Services in Production

**Lesson file:** [../68-deploying-operating.md](../68-deploying-operating.md) · **Examples folder:** `../examples/68-deploying-operating/`

| | |
|---|---|
| Prerequisites | [35](../35-observability-reliability.md), [32](../32-key-management-signing.md) |
| Unlocks | — |
| Examples | **13** — 🟢 3 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 10 |

*containers, config, secrets, migrations, rollouts and the incident response that chain services need*

### Goals

- Package a Go chain service into a minimal, secure container.
- Configure and inject secrets without ever baking a key into an image.
- Deploy with health checks, graceful shutdown and zero-downtime rollouts.
- Run the incident playbooks that chain services specifically need.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Packaging**

   - Multi-stage Dockerfile: build with the full toolchain, ship a static binary on distroless or scratch.
   - `CGO_ENABLED=0`, `-trimpath`, and `-ldflags "-s -w -X main.version=..."` for a small, stamped binary.
   - Non-root user, read-only root filesystem, no shell in the final image.
   - The go-ethereum dependency makes builds slow — layer caching and `go mod download` as its own step.
   - CA certificates and tzdata: the two things `scratch` images always forget.

**2. Configuration**

   - 12-factor: everything from the environment, validated at startup, fail fast on anything missing.
   - Chain configuration as data: RPC URLs, chain ids, contract addresses, confirmation depths, per environment.
   - Never a default private key, never a default mainnet RPC.
   - Config validation as a testable function, with a `--check-config` mode.

**3. Secrets**

   - Never in the image, never in environment variables on a shared host if you can avoid it.
   - Kubernetes Secrets (understand they are only base64 at rest by default), external secret operators, cloud secret managers.
   - Short-lived credentials and workload identity over long-lived keys.
   - Signing keys never reach the application pod at all — they live behind the signing service (lesson 32).

**4. Health checks that mean something**

   - `/healthz` (liveness): the process is alive. Keep it trivial and dependency-free.
   - `/readyz` (readiness): dependencies are reachable **and indexer lag is within bounds** — this is the chain-specific part.
   - Returning 503 while draining so the load balancer stops sending traffic before shutdown.
   - A lagging indexer should go unready, not be restarted — get this distinction right.

**5. Graceful shutdown**

   - `signal.NotifyContext` for SIGTERM; drain deadline shorter than the platform's grace period.
   - Finish the current block, commit the cursor, wait for in-flight sends to be persisted (lesson 21).
   - Kubernetes `terminationGracePeriodSeconds` and a `preStop` sleep so endpoints update first.
   - Test it: SIGTERM under load, assert no lost or duplicated work.

**6. Database migrations**

   - Versioned, forward-only migrations run as a separate job, not on application start.
   - Backward-compatible schema changes so old and new pods can run simultaneously during a rollout.
   - The expand/contract pattern: add column → dual-write → backfill → switch reads → drop.
   - Migrations that rewrite indexed chain data can take hours — plan for it.

**7. Deployment topology**

   - Singleton components (the indexer, the nonce-owning sender) must not run two replicas.
   - Leader election or a `StatefulSet` with one replica; a database advisory lock as a simple alternative.
   - Horizontally scalable components: the read API, the webhook deliverer.
   - Separating the signer into its own deployment with its own network policy.

**8. Rollouts**

   - Rolling updates with `maxUnavailable: 0` for the API; recreate for singletons.
   - Canary a new indexer version by running it against a copy of the data and diffing.
   - Rollback plan, including whether the schema change is reversible.
   - Never deploy an indexer change and a schema change in the same release if you can avoid it.

**9. Chain-specific incident playbooks**

   - Provider outage: fail over, and what to do when all providers are down.
   - Deep reorg beyond your confirmation depth: stop, assess, re-index the affected range, reverse credits.
   - Stuck transaction queue: diagnose nonce gap vs fee too low vs provider rejection; bump or cancel.
   - Hot wallet drained: kill the signer, revoke approvals, assess blast radius, preserve evidence.
   - Chain halt or hard-fork surprise: pause writes, wait for client guidance, verify chain id and fork state.
   - Each playbook: symptom, diagnosis commands, mitigation, escalation, post-incident actions.

**10. Cost and capacity**

   - RPC calls per indexed block × blocks per day = your provider bill. Instrument it.
   - Database growth per day and the retention/partitioning plan.
   - Gas spend as a first-class budget with alerts.
   - Capacity planning against chain growth, not just user growth.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Two indexer replicas running simultaneously and double-writing.
- Readiness probes that ignore indexer lag, so you serve hours-stale data as healthy.
- Running migrations on application startup with multiple replicas.
- A grace period shorter than your drain time, so in-flight sends are lost.
- Secrets in environment variables visible in a crash dump or a `/proc` read.
- No playbook for a deep reorg, so the response is improvised at 3am.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 3 examples** *(one concept in isolation)*

- Write a multi-stage Dockerfile producing a distroless, non-root image.
- Validate configuration at startup and fail fast with a clear message.
- Implement `/healthz` and `/readyz` with distinct semantics.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Make `/readyz` fail when indexer lag exceeds a threshold.
- Graceful shutdown on SIGTERM that commits the cursor and drains sends.
- Stamp version and git commit into the binary and expose them on `/version`.
- Run migrations as a separate job and verify a rolling update works with both schema versions.
- Acquire a Postgres advisory lock so only one indexer instance is ever active.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A full deployment: Dockerfile, compose or Kubernetes manifests, migrations job, probes, graceful shutdown — tested end to end.
- SIGTERM under load and prove no lost or duplicated work with a reconciliation check.
- Write and rehearse the deep-reorg and stuck-queue playbooks against a scripted failure.

### Packages & tools

`os/signal`, `context`, `net/http`, `database/sql`, `log/slog`, `golang.org/x/sync/errgroup`, `github.com/prometheus/client_golang/prometheus`

### Resources to cite

- The Twelve-Factor App: https://12factor.net/
- Distroless images: https://github.com/GoogleContainerTools/distroless
- Kubernetes probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Kubernetes pod termination: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination
- Go — signal.NotifyContext: https://pkg.go.dev/os/signal#NotifyContext

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
