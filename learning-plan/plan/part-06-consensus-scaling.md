# Part 6 — Consensus & Scaling

How agreement is reached at scale: Ethereum's proof of stake, the BFT family, and the layer-2 designs everything is migrating to.

**Core spine.** Lessons 28–30 · 43 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 28 | [Proof of Stake & Ethereum Consensus](#28-proof-of-stake-ethereum-consensus) | 14, 16 | 15 |
| 29 | [Alternative Consensus: BFT, PoA & the Rest](#29-alternative-consensus-bft-poa-the-rest) | 28 | 13 |
| 30 | [Layer 2 & Scaling](#30-layer-2-scaling) | 17, 28 | 15 |

---

## 28 — Proof of Stake & Ethereum Consensus

**Lesson file:** [../28-proof-of-stake.md](../28-proof-of-stake.md) · **Examples folder:** `../examples/28-proof-of-stake/`

| | |
|---|---|
| Prerequisites | [14](../14-consensus-forks.md), [16](../16-ethereum-architecture.md) |
| Unlocks | 29, 30, 64 |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 11 |

*validators, slots and epochs, LMD-GHOST + Casper FFG, attestations, finality, slashing and PBS*

### Goals

- Explain how Ethereum reaches consensus today, end to end.
- Distinguish fork choice (LMD-GHOST) from finality (Casper FFG).
- Describe the validator lifecycle from deposit to withdrawal.
- Explain slashing conditions and what 'finalized' guarantees.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. From work to stake**

   - Sybil resistance bought with bonded capital instead of electricity.
   - The key difference: stake is *inside* the system, so misbehaviour can be punished (slashing).
   - Nothing-at-stake and long-range attacks — and how weak subjectivity answers them.
   - Energy reduction of ~99.95% at the Merge, and what was traded for it.

**2. The validator**

   - 32 ETH deposited to the deposit contract; a BLS12-381 signing key; separate withdrawal credentials.
   - Lifecycle: deposit → pending queue → active → (exiting) → exited → withdrawable.
   - Activation and exit queues (churn limit) and why they exist.
   - Effective balance, and how rewards and penalties scale with it.

**3. Time structure**

   - 12-second slots, 32 slots per epoch (6.4 minutes).
   - One proposer per slot, chosen by RANDAO; the rest of the validator set split into committees.
   - Every validator attests exactly once per epoch.
   - Missed slots are normal — your indexer must never assume a block per 12 seconds.

**4. Attestations**

   - Three votes in one message: head (LMD-GHOST), source and target checkpoints (FFG).
   - BLS aggregation: thousands of signatures compress into one — this is why BLS was chosen (lesson 42).
   - Aggregators, subnets, and inclusion delay rewards.
   - The attestation is the unit of consensus; the block is just a container.

**5. LMD-GHOST fork choice**

   - Latest Message Driven, Greediest Heaviest Observed SubTree.
   - At each fork, follow the child subtree with the most attesting stake — not the longest chain.
   - Only each validator's *latest* vote counts, which bounds the state you track.
   - Work a small block tree by hand, then implement it in Go.

**6. Casper FFG finality**

   - Checkpoints are the first slot of each epoch. Validators vote (source → target) links.
   - A target with ≥2/3 stake voting is *justified*; when its child is also justified, the parent is *finalized*.
   - Two epochs (~12.8 minutes) in the happy path.
   - The `safe` (justified) and `finalized` block tags map onto exactly this.

**7. Economic security**

   - Rewards: attestation (source/target/head), sync committee, proposer.
   - Penalties for missing duties; the **inactivity leak** that bleeds non-participants until finality resumes.
   - Finality is *economic*: reverting a finalized block costs ≥1/3 of all staked ETH.
   - The number to quote when someone asks 'how final is final'.

**8. Slashing**

   - Double proposal: two distinct blocks for one slot.
   - Surround vote and double vote: contradictory FFG attestations. Implement the surround check.
   - The correlation penalty: punishment scales with how many others were slashed at the same time.
   - Why running the same key on two machines is the most common way to get slashed.

**9. The Engine API in practice**

   - `engine_newPayloadV*` (execute this block) and `engine_forkchoiceUpdatedV*` (this is the head/safe/finalized).
   - JWT-authenticated localhost only; a shared secret file.
   - Optimistic sync: the EL may accept a payload before it has fully validated ancestors.
   - What breaks when the two clients disagree, and how it looks in logs.

**10. Proposer-builder separation**

   - MEV-Boost: the proposer sells the right to build the block to a marketplace of builders.
   - Relays, blinded blocks, and the trust assumptions at each step.
   - The censorship-resistance debate and inclusion lists.
   - Consequence for you: the block you see was probably not built by the validator who proposed it (lesson 40).

**11. Withdrawals and staking**

   - EIP-4895 push withdrawals; partial (rewards) vs full (exit) withdrawals.
   - 0x00 vs 0x01 vs 0x02 withdrawal credentials.
   - Liquid staking tokens, the centralisation pressure, and what an integrator must model.
   - Reading the beacon REST API from Go: `/eth/v1/beacon/states/head/finality_checkpoints`.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming a block per 12 seconds and stalling on a missed slot.
- Using `latest` for settlement when `finalized` exists.
- Running a validator key on two machines simultaneously — the classic slashing.
- Confusing justified with finalized when reading the beacon API.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Query the beacon API for the current head slot and epoch.
- Read the finalized checkpoint and print its epoch and root.
- Compute the slot number for a given timestamp.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Simulate LMD-GHOST over a small weighted block tree and pick the head.
- Implement the FFG justification/finalization state machine over 6 epochs.
- Detect a double vote between two attestations.
- Compare `latest`/`safe`/`finalized` block numbers over 100 polls and chart the lag.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Implement the surround-vote slashing check with correct source/target comparison.
- Model an inactivity leak: 40% offline, simulate balances until finality resumes.
- Fetch a block from a MEV-Boost relay's data API and compare its builder with the proposer.

### Packages & tools

`net/http`, `encoding/json`, `sort`, `math/big`, `context`

### Resources to cite

- Ethereum consensus specs: https://github.com/ethereum/consensus-specs
- Ethereum docs — Proof of Stake: https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/
- Gasper (Combining GHOST and Casper): https://arxiv.org/abs/2003.03052
- Beacon API spec: https://ethereum.github.io/beacon-APIs/
- EIP-4895 (withdrawals): https://eips.ethereum.org/EIPS/eip-4895

---

## 29 — Alternative Consensus: BFT, PoA & the Rest

**Lesson file:** [../29-alternative-consensus.md](../29-alternative-consensus.md) · **Examples folder:** `../examples/29-alternative-consensus/`

| | |
|---|---|
| Prerequisites | [28](../28-proof-of-stake.md) |
| Unlocks | 37 |
| Examples | **13** — 🟢 4 easy, 🟡 6 medium, 🔴 3 hard |
| Topics | 9 |

*PBFT, Tendermint, HotStuff, Clique, DPoS — what each assumes, what each buys, and one implemented in Go*

### Goals

- Explain the BFT family and where the 3f+1 bound comes from.
- Compare instant-finality BFT with probabilistic longest-chain consensus.
- Choose a consensus mechanism for a given requirement set.
- Implement a simplified BFT round in Go.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The formal frame**

   - Safety (nothing bad) vs liveness (something good eventually).
   - Synchrony assumptions: synchronous, partially synchronous, asynchronous.
   - FLP impossibility: no deterministic consensus in an asynchronous network with one crash fault.
   - How every real system escapes FLP: timeouts, randomness, or a synchrony assumption.

**2. CFT vs BFT**

   - Raft/Paxos tolerate crashes, not lies — fine inside one company, useless between adversaries.
   - Byzantine faults: arbitrary, including equivocation and collusion.
   - Why a blockchain needs BFT and a distributed database usually does not.
   - Raft's readability as a Go codebase (`hashicorp/raft`) is worth an hour of your time.

**3. The 3f+1 bound**

   - With n nodes and f Byzantine, you need n ≥ 3f+1 and quorums of 2f+1.
   - The derivation: two quorums must intersect in at least one honest node.
   - Why 2f+1 is not enough under partial synchrony.
   - Concretely: 4 nodes tolerate 1, 7 tolerate 2, 100 tolerate 33.

**4. PBFT**

   - Three phases: pre-prepare, prepare, commit — plus view change when the primary fails.
   - O(n²) messages per round; this is why classic BFT does not scale past ~100 nodes.
   - Implementing one round with 4 nodes in Go, with one node lying.
   - Where it is used: permissioned chains, and as the ancestor of everything below.

**5. Tendermint / CometBFT**

   - Propose → prevote → precommit, with locking rules that guarantee safety across rounds.
   - **Instant finality**: one block, then it is final. No reorgs, ever.
   - The trade: if >1/3 are offline the chain *halts* rather than forking.
   - Written in Go — you can read the actual implementation (lesson 37).

**6. HotStuff and linear BFT**

   - Three-chain commit rule with a leader collecting threshold signatures.
   - O(n) message complexity via signature aggregation — the key improvement.
   - Pipelining rounds so throughput does not depend on round latency.
   - Used by Diem-lineage chains and several modern L1s.

**7. Proof of Authority (Clique)**

   - A fixed signer set takes turns sealing blocks; in-turn/out-of-turn delays break ties.
   - Signer set changes by voting, encoded in the header's extraData.
   - Perfect for testnets, devnets and consortium chains; not for public value.
   - geth's `consensus/clique` is short and very readable Go.

**8. Delegated and other schemes**

   - DPoS: token holders elect a small validator set; throughput up, decentralisation down.
   - Proof of History (Solana): a verifiable delay function used as a clock, not consensus.
   - Proof of space/time, proof of elapsed time, and where each is used.
   - Nakamoto vs BFT as the fundamental axis: open membership and forks, vs fixed membership and finality.

**9. The decision table**

   - Validator-set size, finality latency, fork tolerance, permissioning, throughput, halt-vs-fork behaviour.
   - Which to pick for: a public L1, a consortium ledger, an app-chain, a testnet.
   - The honest answer is usually 'use an existing chain'.
   - Where Go helps: CometBFT, Clique and `hashicorp/raft` are all readable Go implementations.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming BFT finality means the chain cannot roll back — it means it halts instead.
- Deploying PoA and calling it decentralised.
- Choosing a 3f+1 quorum but running 3 nodes and tolerating 1 fault.
- Ignoring the message-complexity blowup when growing the validator set.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 13).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Compute the fault tolerance and quorum size for n = 4, 7, 10, 100.
- Implement a simple leader-election round-robin over a signer set.
- Show why 2 of 3 nodes cannot safely tolerate 1 Byzantine node.

**🟡 Medium — 6 examples** *(concepts combined, and the traps)*

- Implement one PBFT round with 4 nodes and 1 Byzantine node; show it commits.
- Run the same setup with 2 Byzantine nodes and show safety break.
- Implement Clique-style sealing with in-turn/out-of-turn delays.
- Count messages exchanged as n grows for PBFT vs a gossip broadcast.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Add view change to your PBFT round and recover from a failed primary.
- Implement Tendermint's lock/unlock rule and show it prevents a conflicting commit across rounds.
- Simulate a network partition and show a BFT chain halting while a Nakamoto chain forks.

### Packages & tools

`sync`, `context`, `time`, `sort`, `crypto/ed25519`, `testing`

### Resources to cite

- PBFT (Castro & Liskov): https://pmg.csail.mit.edu/papers/osdi99.pdf
- Tendermint / CometBFT docs: https://docs.cometbft.com/
- HotStuff: https://arxiv.org/abs/1803.05069
- EIP-225 (Clique PoA): https://eips.ethereum.org/EIPS/eip-225
- Raft: https://raft.github.io/

---

## 30 — Layer 2 & Scaling

**Lesson file:** [../30-layer2-scaling.md](../30-layer2-scaling.md) · **Examples folder:** `../examples/30-layer2-scaling/`

| | |
|---|---|
| Prerequisites | [17](../17-rlp-merkle-patricia-trie.md), [28](../28-proof-of-stake.md) |
| Unlocks | 39, 67 |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 10 |

*rollups, data availability, blobs, channels, and what L2s mean for your Go code*

### Goals

- Explain the rollup thesis and what makes an L2 an L2.
- Compare optimistic and zk rollups on trust, latency and cost.
- Describe how blobs (EIP-4844) changed L2 economics.
- Adapt Go client code for L2 quirks: fees, finality, reorgs and bridges.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The scalability trilemma**

   - Decentralisation, security, scalability — pick two, or move execution off the base layer.
   - Why raising the gas limit is not an answer: state growth and node requirements.
   - The rollup-centric roadmap in one paragraph.
   - What 'inherits L1 security' actually means, and where it stops being true.

**2. The rollup thesis**

   - Execute off-chain, publish *data* on-chain, prove or dispute the result on L1.
   - The three components: sequencer, batch submitter, and the L1 settlement contract.
   - Why data availability is the non-negotiable part — without it you cannot reconstruct state.
   - The test for 'is it really a rollup': can anyone reconstruct L2 state from L1 data alone?

**3. Optimistic rollups**

   - Assume valid, allow challenges. Fraud proofs, interactive bisection, the ~7-day window.
   - Why 7 days: enough time for an honest challenger to get a transaction onto L1 under censorship.
   - Forced inclusion / escape hatches when the sequencer censors or dies.
   - Arbitrum and OP Stack as the two dominant designs; their differing fraud-proof status.

**4. ZK rollups**

   - Validity proofs: every batch is proven correct before it settles.
   - Fast withdrawals (no challenge window) at the cost of proving time and expense.
   - EVM-equivalence types 1–4 and what each sacrifices.
   - Where the proving cost sits today and where it is heading (lesson 39).

**5. Data availability**

   - Calldata (16 gas/byte) → blobs (EIP-4844), a separate, much cheaper market.
   - KZG commitments, versioned hashes, and blob pruning after ~18 days.
   - DA layers and committees (Celestia, EigenDA, validiums) — and the trust they reintroduce.
   - Data availability sampling and full danksharding, in one paragraph.

**6. Sequencers**

   - Centralised today on nearly every L2. Soft confirmations in ~200ms vs L1 finality in ~15 minutes.
   - **The trap for your Go service**: an L2 'confirmation' is a promise, not settlement.
   - Sequencer downtime, reorgs of unposted blocks, and forced-inclusion paths.
   - Shared/decentralised sequencing proposals.

**7. Bridges**

   - Canonical (the rollup's own) vs third-party (liquidity/message bridges).
   - Lock-mint vs burn-mint vs liquidity-pool designs.
   - Bridges are the single biggest source of losses in the ecosystem — Ronin, Wormhole, Nomad, Harmony.
   - Full treatment in lesson 67.

**8. State and payment channels**

   - Lock funds, exchange signed balance updates off-chain, settle once.
   - Lightning as the production example; HTLCs for routing.
   - Where channels still beat rollups: bilateral, high-frequency, low-value.
   - Implement a two-party channel with unilateral close in Go.

**9. Sidechains and validiums**

   - Own consensus, own security — Polygon PoS, Gnosis Chain. Not rollups.
   - Validium: validity proofs but off-chain data — the data-availability trust reappears.
   - The L2Beat framing of stages and trust assumptions.
   - Why the label matters when you are custodying user funds.

**10. L2 differences that break naive Go code**

   - Fee model: L2 execution fee + **L1 data fee**; `eth_estimateGas` may not include the latter.
   - Extra RPC fields and namespaces (`optimism_`, `arbtrace_`); `ethclient` may drop unknown fields.
   - Different block times (sub-second) and different reorg behaviour.
   - Chain ids, and address-space collisions across chains.
   - `finalized` on an L2 may mean 'the sequencer said so'. Read the docs for each chain you support.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Treating an L2 soft confirmation as settlement.
- Estimating L2 fees without the L1 data component and under-funding a transaction.
- Assuming the same address is the same contract across chains.
- Using a third-party bridge without understanding who can mint on the destination.
- Reusing an L1 confirmation-depth policy on a chain with sub-second blocks.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Connect to an OP-stack testnet with `ethclient` and print chain id and block time.
- Read an L2 block and list the fields that differ from L1.
- Compute the total cost of an L2 transaction from its receipt.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Split an L2 transaction's cost into execution fee and L1 data fee.
- Decode a blob-carrying transaction's versioned hashes from a real L1 block.
- Compare `latest` vs `finalized` lag on an L2 and on L1 over 100 polls.
- Detect the batch-submission transaction on L1 for a given L2 block range.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Implement a two-party payment channel with signed balance updates and a unilateral close.
- Verify a KZG commitment against a blob's versioned hash with `crypto/kzg4844`.
- Write a chain-config abstraction so one Go service supports L1 and two L2s with per-chain confirmation policies.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/crypto/kzg4844`, `github.com/ethereum/go-ethereum/core/types`, `math/big`

### Resources to cite

- Ethereum docs — Layer 2: https://ethereum.org/en/layer-2/
- EIP-4844: https://eips.ethereum.org/EIPS/eip-4844
- L2Beat (risk framework): https://l2beat.com/
- OP Stack specs: https://specs.optimism.io/
- Arbitrum docs: https://docs.arbitrum.io/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
