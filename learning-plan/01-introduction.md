# 01 — Introduction to Blockchain

> **Status:** ✅ written. Examples: 12/12 built and run.
> **Spec:** [plan/part-01-foundations.md](plan/part-01-foundations.md#01-introduction-to-blockchain)

| | |
|---|---|
| **Part** | Part 1 — Foundations |
| **Prerequisites** | none — start here |
| **Unlocks** | 02 |
| **Examples** | [12](examples/01-introduction/) (🟢 5 · 🟡 4 · 🔴 3) |

*What problem a blockchain solves, the ledger, decentralization, the landscape — and what it is **not**.*

No cryptography yet, and no Ethereum. This lesson builds the mental model everything else hangs
off. If you only remember one sentence from it, make it this one: **a blockchain is a machine for
agreeing on the order of events between parties who do not trust each other.** Everything in the
next 67 lessons is an implementation detail of that.

## Goals

- Explain what a blockchain is without using the word "blockchain".
- Name the problem it solves — double-spend without a trusted third party — and say why it is hard.
- Place Bitcoin, Ethereum, L2s and app-chains on one map.
- Decide honestly when a database is the better answer.

## Concepts

### 1. The double-spend problem

Hand someone a £10 note and you no longer have it. That is not a design achievement, it is
physics: the note is a physical object and objects are in one place at a time. Now try the same
with a digital coin. A digital coin is a number in a file, and copying a number is free, instant
and leaves no trace. Send `alice → bob: 10` to one person and send the identical bytes to someone
else, and unless something stops you, you have spent the same money twice.

This is the **double-spend problem**, and it is the only problem Bitcoin set out to solve.
Example [3](examples/01-introduction/1-easy.md#3-the-double-spend) shows it in nine lines: apply
the same transfer twice to a ledger with no rules and alice ends up at −100 having spent 200.

The classic fix is a **trusted third party**. Your bank keeps the balances. When you pay, the bank
decrements one row and increments another, and because there is exactly one bank, there is exactly
one answer to "did that money already move?" This works extremely well. It is fast, cheap, and
reversible when something goes wrong. It also means the bank can freeze your account, refuse your
transaction, be compelled by a court, get hacked, or simply be down on a Sunday. For most purposes
that trade is fine. For some it is not, and those are the only cases where the rest of this course
is worth the cost.

Removing the trusted third party is expensive, and it is worth being blunt about the bill:

| | Trusted server | Public blockchain |
|---|---|---|
| Throughput | ~10⁵ tx/s | ~15 tx/s (Ethereum L1) |
| Confirmation | milliseconds | seconds to ~13 minutes for finality |
| Cost per write | ~0 | cents to dollars |
| Energy | negligible | material (proof of work) or moderate (proof of stake) |

Notice what is *not* on that list: storage. Storing transactions is trivial — any database does it.
The hard part is **ordering**. When two conflicting spends of the same coin arrive at different
machines at roughly the same moment, both are individually valid, and only one can be applied.
Nothing in the data itself says which came first; "first" is not a property of bytes. Somebody, or
some rule, has to decide — and every participant has to end up agreeing on the same decision.

Hold onto that. The rest of the lesson is an argument that ordering is the whole game.

### 2. A ledger as an append-only log

Forget blocks for a moment. The underlying structure is an **ordered, append-only log of events**,
plus a function that applies one event to some state. Balances are not stored anywhere. They are
what you get by folding the log from the beginning:

```go
type Transfer struct {
    From, To string
    Amount   int64
}

func apply(balances map[string]int64, t Transfer) {
    balances[t.From] -= t.Amount
    balances[t.To] += t.Amount
}
```

That is example [1](examples/01-introduction/1-easy.md#1-the-ledger-is-the-log). Example
[2](examples/01-introduction/1-easy.md#2-balances-are-derived-not-stored) makes the consequence
explicit: fold the first *n* events instead of all of them and you have the balance at height *n*.
No snapshot required. This is why you can ask a chain "what did this address own at block
18,000,000?" and why an indexer ([31](31-blockchain-indexer.md)) can rebuild its entire database
from nothing but the chain.

Run that same log through the same `apply` on a thousand machines and you get a **replicated state
machine**: same log + same rules ⇒ same state, everywhere. The log is the source of truth; the
balance map is a derived index, like a materialised view.

The catch is in "same rules". Every input to a state transition has to be **deterministic**, or two
honest nodes running correct code reach different states and the network splits for no reason.
Three things are therefore banned from consensus code, and Go makes the first one easy to trip over:

- **Map iteration order.** Go deliberately randomises it. Example
  [8](examples/01-introduction/2-medium.md#8-determinism-map-order-breaks-agreement) folds eight
  balances into a digest by walking the map and gets a different answer nearly every time; sorting
  the keys first makes it stable.
- **Wall-clock time and randomness.** `time.Now()` and `math/rand` differ per machine per run.
  Chains that need a timestamp treat it as a *consensus input* with strict bounds
  ([08](08-blocks-and-chain.md)), not as a clock. Chains that need randomness derive it from the
  protocol itself ([53](53-oracles-randomness.md)).
- **Floating point.** Rounding differs across platforms and compilers. All on-chain arithmetic is
  integer arithmetic, which is why [03](03-bytes-encoding.md) spends a whole lesson on `math/big`.

> **Rule for this repo:** no `float64` anywhere near money, ever. Not in a lesson, not in an
> example, not in a quick script.

### 3. Decentralization is a spectrum

"Decentralized" is used as a boolean and it is not one. It is a set of independent questions, and a
given system answers each differently:

| Axis | Question |
|---|---|
| Read | Who can see the data? |
| Write | Who can submit a transaction? |
| Validate | Who checks that the rules were followed? |
| Govern | Who can change the rules, and how fast? |

Bitcoin and Ethereum answer "anyone" to the first three and "rough social consensus, slowly" to the
fourth — that is **permissionless**. Hyperledger Fabric and Corda answer "members of this
consortium" — **permissioned**. And a great many things marketed as blockchains answer "us" to all
four, which makes them a database with signatures on it. That is not an insult; it may be exactly
the right design. It just should not be sold as something it is not.

Who actually verifies anything depends on what kind of node you run:

- **Full node** — downloads every block, checks every rule, keeps current state. Trusts nobody.
- **Archive node** — a full node that also keeps every historical state. Big, and what you need for
  historical queries ([33](33-node-operations.md)).
- **Light client** — downloads headers only and verifies specific facts with proofs
  ([64](64-light-clients-spv.md)). Trusts the header chain.
- **Validator / miner** — a full node that also proposes blocks and is paid for it.

Most people who say they "use Ethereum" are running none of these; they are calling someone else's
node over JSON-RPC, which means they trust that provider completely. That is a defensible choice
made explicitly, and a security hole made accidentally. Lessons [20](20-json-rpc-ethclient.md) and
[64](64-light-clients-spv.md) are about knowing which one you are doing.

One more axis that matters more than it looks: **client diversity**. Ethereum is a specification
with several independent implementations (geth, erigon, nethermind, reth on the execution side;
prysm, lighthouse, teku, nimbus on the consensus side). If one implementation has 90% market share,
a bug in it is a bug in the network — and that has happened. Diversity is a safety property, not a
nicety.

### 4. The three ideas chained together

A blockchain is three well-understood techniques, each solving one requirement:

**Hash linking gives integrity.** Each record's hash covers both its own contents and the previous
record's hash, so the records form a chain in which nothing can be edited quietly:

```go
func (r Record) hash() [32]byte {
    h := sha256.New()
    h.Write(r.Prev[:])          // commits to everything before this record
    h.Write([]byte(r.Data))     // and to this record's own contents
    var out [32]byte
    copy(out[:], h.Sum(nil))
    return out
}
```

Examples [4](examples/01-introduction/1-easy.md#4-hash-linking-records) and
[5](examples/01-introduction/1-easy.md#5-tampering-breaks-every-record-after-it) build that chain
and then break it: change record 0's data and record 1 is left pointing at a hash that no longer
exists. Note carefully what this buys — not that data *cannot* change, but that changes cannot be
*hidden*. Lessons [04](04-hash-functions.md) and [05](05-merkle-trees.md) make this rigorous.

**Digital signatures give authorization.** Only the holder of a private key can produce a valid
signature over a transaction, so only they can move their funds. There are no accounts to log into
and no password reset. Lesson [06](06-keys-signatures.md).

**Consensus gives ordering.** Everyone agrees which of two conflicting spends came first. This is
the part that had no good answer before 2009.

None of the three was new. Merkle trees date to 1979, digital signatures to the 1970s, and
Byzantine fault tolerance to 1982. What was new was combining them with an answer to the question
in the next section.

### 5. Byzantine fault tolerance and Sybil resistance

The **Byzantine Generals** framing, stripped of the story: several parties must agree on a single
decision, they can only communicate over unreliable channels, and an unknown number of them are
actively lying. A protocol is *Byzantine fault tolerant* if the honest participants still converge
on one answer despite that.

The classical solutions all work by voting. Collect messages from participants, and once more than
two-thirds agree, commit. That gives you PBFT and its descendants ([29](29-alternative-consensus.md)),
and it works well — with two constraints. First, it costs messages: example
[9](examples/01-introduction/2-medium.md#9-the-cost-of-agreement) tabulates n·(n−1) per round, which
is 200 million messages per decision at 10,000 nodes. Second, and fatally for an open network, it
assumes you **know who the participants are**.

On the open internet, identities are free. I can spin up ten thousand nodes this afternoon, and
"one vote per node" becomes "one vote per machine I can afford to boot". That is the **Sybil
attack**, and it is why the classical literature could not be pointed at digital cash directly.

The 2009 insight was to make identity **costly** rather than free:

- **Proof of work** — your influence is proportional to computation you actually performed, which
  costs electricity you actually paid for. You cannot fake it; you can only buy it
  ([09](09-proof-of-work.md)).
- **Proof of stake** — your influence is proportional to capital you have bonded inside the system,
  which can be destroyed if you misbehave ([28](28-proof-of-stake.md)). Note the extra property:
  because the stake lives *inside* the system, cheating can be punished, which proof of work cannot
  do.

**This is the actual innovation** — not the Merkle tree, not the linked hashes, not the peer-to-peer
network. Sybil-resistant open-membership consensus. Everything else was standard computer science.

Examples [10](examples/01-introduction/3-hard.md#10-trusted-server-vs-replicas-under-a-partition)
through [12](examples/01-introduction/3-hard.md#12-one-shared-ordering-rule-five-identical-states)
walk the argument end to end. Ten contrasts a trusted server (solves double-spending trivially,
and is a single point of control) with five replicas under a network partition (stay available,
end up with two irreconcilable histories). Eleven shows five nodes receiving the *same* two
conflicting transfers in different arrival orders and splitting permanently into two camps —
nothing was lost or censored, propagation delay alone did it. Twelve adds one shared rule — collect
first, then order by a hash of the transaction — and all five nodes agree with nobody in charge.

That toy rule ("sort by txid") is not safe in the real world, because an attacker can grind
transaction contents until the id sorts where they want it. A real chain replaces it with a rule
that is expensive to game. That substitution is the whole subject of Part 3 and lesson
[28](28-proof-of-stake.md).

### 6. History in one page

Digital cash was tried repeatedly before it worked, and every attempt contributed a piece:

- **1983–1995 — David Chaum's DigiCash.** Genuinely good blind-signature cryptography giving real
  payer privacy. Centrally issued, so it needed a company; the company went bankrupt in 1998 and
  the money went with it.
- **1997 — Adam Back's Hashcash.** Proof of work, invented as an anti-spam measure: make the sender
  burn a little CPU per email. Bitcoin uses it almost unchanged.
- **1998 — Wei Dai's b-money and Nick Szabo's bit gold.** Both proposed distributed ledgers with
  proof-of-work-created money. Neither specified how the participants agree on a single history —
  precisely the gap.
- **2008–2009 — Bitcoin.** Satoshi Nakamoto's contribution, in one sentence: **use proof of work to
  make the longest chain the canonical one, so open, permissionless membership becomes possible
  without a trusted party.** The whitepaper is nine pages and you should read it this week.
- **2015 — Ethereum.** Replace "a ledger of coin transfers" with "a replicated general-purpose
  computer". This opened programmable money, tokens, DeFi and NFTs; it also opened an enormous
  attack surface, which lesson [27](27-contract-security.md) is entirely about.

Four Ethereum upgrades matter to your code specifically:

- **EIP-1559 (Aug 2021)** changed how fees work: a burned base fee plus a tip to the proposer.
  Every fee calculation you write assumes it ([16](16-ethereum-architecture.md), [21](21-sending-transactions.md)).
- **The Merge (Sep 2022)** swapped proof of work for proof of stake. Running a node now means
  running *two* programs ([33](33-node-operations.md)).
- **Shanghai (Apr 2023)** enabled staking withdrawals.
- **EIP-4844 / blobs (Mar 2024)** gave rollups a cheap data lane and cut L2 fees by roughly an
  order of magnitude ([30](30-layer2-scaling.md)).

### 7. The landscape map

Five categories, distinguished by where execution happens and where security comes from:

| Kind | Executes | Security from | Examples |
|---|---|---|---|
| **L1** | its own nodes | its own consensus | Bitcoin, Ethereum, Solana |
| **L2 rollup** | its own sequencer | an L1, via proofs + data posted there | Arbitrum, Optimism, Base, zkSync |
| **Sidechain** | its own nodes | its own (usually smaller) validator set | Polygon PoS, Gnosis Chain |
| **App-chain** | its own nodes | its own validators, one application | Cosmos chains, dYdX |
| **Validium** | its own sequencer | L1 proofs, but data kept off-chain | Immutable X |

The distinction that matters is the security one. A rollup that posts its data to Ethereum can be
reconstructed by anyone from Ethereum alone; a sidechain cannot, and if its validators collude your
funds are gone. Marketing blurs this constantly. [L2Beat](https://l2beat.com/) exists to unblur it.

Where things sit today changes fast enough that any number printed here would be wrong by the time
you read it. Check current figures yourself rather than trusting a lesson. The *shape* is stable:
Bitcoin is the largest store of value, Ethereum has the most developer activity and settles the
most value in contracts, most Ethereum transactions have moved to L2s, and Solana is the main
high-throughput alternative.

**Where Go sits.** Unusually well, which is why this course exists:

- **geth** — the reference Ethereum execution client, and also the library you will import in
  almost every lesson from [17](17-rlp-merkle-patricia-trie.md) onward.
- **prysm** — a major Ethereum consensus client.
- **btcd** and **lnd** — Bitcoin and Lightning.
- **Cosmos SDK** and **CometBFT** — an entire app-chain framework, in Go ([37](37-cosmos-tendermint.md)).
- Most **indexers, bridges, exchange backends and wallet infrastructure** in the industry.

The languages you will meet beside Go: **Solidity** for contracts on Ethereum (lessons
[22](22-solidity-basics.md)–[27](27-contract-security.md) — you need to read it, not master it),
and **Rust** for Solana programs and several newer clients. This course keeps every example in Go
and treats Solidity purely as the thing Go talks to.

### 8. Vocabulary you will meet constantly

Learn these now; they appear in every lesson from here on.

| Term | What it actually means |
|---|---|
| **Node** | a machine running the protocol |
| **Client** | the software implementing the protocol (geth, prysm) |
| **Block** | a batch of transactions plus a header, added as one unit |
| **Transaction** | a signed instruction to change state |
| **Mempool** | a node's local set of valid-but-not-yet-included transactions |
| **Gas** | Ethereum's unit of computational work; you pay per unit |
| **Nonce** | a per-sender counter preventing replay and fixing order |
| **Fork** | two valid histories existing at once — or a rule change |
| **Finality** | the point past which a block will not be reverted |
| **Validator** | a staked participant that proposes and attests to blocks |

Three distinctions worth getting right immediately:

- **EOA vs contract account.** An externally owned account is controlled by a private key. A
  contract account is controlled by its code. Both have 20-byte addresses and you often cannot tell
  them apart without looking ([15](15-account-model-state.md)) — and account abstraction
  ([47](47-account-abstraction.md)) is busy blurring the line further.
- **On-chain vs off-chain.** On-chain means every node stores and verifies it, and you paid for
  that. Off-chain means anything else. Most of a working system is off-chain, which is why this
  course spends Parts 7 and 13–14 there.
- **L1 vs L2.** Where does execution happen, and where does security come from — see the table above.

**Units.** Ethereum's smallest unit is the **wei**; 10¹⁸ wei = 1 ether, and 10⁹ wei = 1 **gwei**
(the unit gas prices are quoted in). Bitcoin's is the **satoshi**, 10⁸ per BTC. You always compute
in the smallest unit with integers and format for display at the very last moment
([03](03-bytes-encoding.md)).

And four words the industry uses badly:

- **"Smart contract"** — neither smart nor a contract. It is a program with an address, deployed
  once, run by every node. Nick Szabo's term, borrowed and stretched.
- **"Wallet"** — holds no coins. It holds *keys* and a view of the chain. Losing your wallet app
  loses nothing if you have the seed phrase; losing the seed phrase loses everything
  ([07](07-addresses-wallets-hd.md)).
- **"Token"** — usually a row in a contract's mapping, not an object you possess. "You own 5 USDC"
  means "the USDC contract's storage says your address maps to 5000000" ([26](26-erc-standards.md)).
- **"Immutable"** — see the pitfalls below.

### 9. When *not* to use a blockchain

The most valuable skill in this field is telling someone their idea does not need one. Here is the
comparison, honestly:

| | Postgres | Public blockchain |
|---|---|---|
| Writes/sec | 10⁴–10⁵ | ~15 (Ethereum L1), ~thousands (L2s) |
| Latency | ~1 ms | seconds, minutes to finality |
| Cost per write | ~0 | cents to dollars |
| Privacy | you control it | everything is public forever |
| Schema change | a migration | a hard fork, or never |
| Undo a mistake | `UPDATE` | you cannot |
| Operational cost | one team | your team, plus a whole network |

A blockchain is worse on every axis except one, and the exception is the whole point: **no single
party can be compelled to change the record**. So the question that settles it is not "would a
ledger be nice here?" — it is:

> **Who are the mutually distrusting parties, and what specifically stops one of them cheating today?**

If the answer is "there's only us" or "a contract and the courts", you want Postgres. If it is "a
dozen competitors who will not accept one of us running the database", you might genuinely want a
shared ledger — though possibly a permissioned one.

**Irreversibility cuts both ways.** It is the feature: nobody can claw back your funds, censor your
transaction, or rewrite history. It is also the liability: send to a wrong address and it is gone;
approve a malicious contract and it is gone; ship a buggy contract and you cannot patch it
([46](46-upgradeable-contracts.md) exists because of this, and introduces its own admin key).
There is no support line. Design for that from the first line of code
([32](32-key-management-signing.md)).

The cases where the answer is genuinely yes:

- **Bearer assets** — value that moves without a custodian's permission.
- **Cross-organisation settlement** — several parties, no natural operator, disputes are expensive.
- **Censorship resistance** — the transaction must go through even if a powerful party objects.
- **Verifiable public state** — anyone must be able to audit the record independently.
- **Composability** — other people's programs must be able to build on yours without asking.

If your use case is not on that list, say so out loud. You will be right more often than not, and
the honesty is worth more than the project.

## Exercises

Write these in `practice/01-introduction/`. They are short — this lesson is about the model, not
the code. Aim for programs you can explain, not clever ones.

1. **Fold your own ledger.** Model a five-event log with three participants and compute the final
   balances. Then compute the balance of one participant at every height, without storing any
   intermediate state.
2. **Break it.** Add a `Ledger.Apply` that enforces "you cannot send what you do not have", then
   construct two transactions that are each individually valid but cannot both apply. Print both
   possible outcomes.
3. **Chain and tamper.** Hash-link four records. Write `Verify() (int, bool)` returning the index
   of the first broken link. Change record 1 and confirm it reports 2. Then also fix record 2's
   `Prev` and confirm it reports 3 — explain in a comment why an attacker must redo *every*
   subsequent record.
4. **Find the nondeterminism.** Write a function that folds a `map[string]int64` into a digest.
   Call it ten times in one process and count how many distinct results you get. Fix it. Then list,
   in a comment, three other things in Go that would break consensus the same way.
5. **Cost the agreement.** Extend example 9: at 1 KB per message and 100 ms round-trip, how long
   would one decision take for 21, 100 and 1000 nodes, and how much bandwidth would each node need?
   Write down the number of nodes at which you would stop using a voting protocol.
6. **Argue the other side.** Pick a real product you know — payroll, ticketing, medical records,
   loyalty points, supply chain. Write ten honest lines answering: who are the mutually distrusting
   parties, what stops one of them cheating today, and what specifically would a public chain add?
   Reach a conclusion. Most of these should end in "use Postgres" — that is the correct answer, and
   being able to say it is the point of the exercise.
7. **Read the whitepaper.** Read [Bitcoin: A Peer-to-Peer Electronic Cash System](https://bitcoin.org/bitcoin.pdf)
   end to end — it is nine pages. Write one paragraph on section 11 ("Calculations"), which you
   will implement in lesson [14](14-consensus-forks.md).

## Best Practices & Pitfalls

- **"Immutable" means unchangeable, not correct.** A chain records a mistaken, fraudulent or
  outright stupid transaction exactly as faithfully as a good one, forever. *Why it matters:* every
  validation you skip becomes permanent. Validate before you sign, not after you send
  ([27](27-contract-security.md), [40](40-defi-primitives-mev.md)).
- **Do not assume decentralization — go and look.** Nearly every L2 has a single sequencer, most
  major contracts have an upgrade key, and many "DAOs" are a 3-of-5 multisig. *Why it matters:* the
  real security of a system is its weakest control point, and if you are custodying user funds you
  need to know who that is. Ask: who can upgrade this, who can pause it, who can censor me
  ([46](46-upgradeable-contracts.md), [67](67-bridges-cross-chain.md)).
- **Throughput numbers are meaningless without finality and validator count.** "100,000 TPS" from a
  chain with 20 validators and no finality guarantee is not comparable to Ethereum's 15.
  *Why it matters:* you will be asked to choose a chain, and TPS is the number people quote because
  it is the easiest one to game.
- **Determinism is not a style preference.** Map iteration, `time.Now()`, `math/rand` and
  `float64` are all correctness bugs in code that must agree across machines. *Why it matters:* the
  failure mode is not a crash, it is a silent chain split — see example 8.
- **Compute in the smallest unit, format only for display.** Wei and satoshi, in integers.
  *Why it matters:* a float rounding error in a money path either mints or destroys value, and the
  chain will faithfully record it ([03](03-bytes-encoding.md)).
- **Know whether you are verifying or trusting.** Calling someone else's RPC endpoint is trusting
  them completely. That is often fine — but it should be a decision you made, not one you drifted
  into ([20](20-json-rpc-ethclient.md), [64](64-light-clients-spv.md)).
- **Be the person who says "you don't need a blockchain".** *Why it matters:* credibility. Anyone
  can be enthusiastic; being able to name the three cases where it genuinely helps, and say so when
  none of them apply, is what makes the rest of your judgement worth listening to.

## Checklist

- [ ] I can explain what a blockchain is without using the word "blockchain".
- [ ] I can state the double-spend problem and explain why *ordering*, not storage, is the hard part.
- [ ] I can name the three assembled ideas — hash linking, signatures, consensus — and what each provides.
- [ ] I can explain Sybil resistance and why it, not the Merkle tree, was the 2009 innovation.
- [ ] I can place L1s, rollups, sidechains, app-chains and validiums on one map and say what secures each.
- [ ] I can list three things that break determinism in Go and explain the failure mode.
- [ ] I can use node, block, transaction, mempool, gas, nonce, fork, finality and validator correctly.
- [ ] I can convert between wei, gwei and ether, and explain why the arithmetic is integer-only.
- [ ] I can argue honestly for Postgres over a blockchain for a given product, and name the cases where it genuinely is the right answer.

## Resources

**Read first**

- Bitcoin whitepaper — https://bitcoin.org/bitcoin.pdf *(9 pages; read all of it)*
- Ethereum docs — Intro to Ethereum: https://ethereum.org/en/developers/docs/intro-to-ethereum/
- Ethereum docs — Intro to Ether: https://ethereum.org/en/developers/docs/intro-to-ether/

**Reference**

- Ethereum docs — Nodes and clients: https://ethereum.org/en/developers/docs/nodes-and-clients/
- Ethereum docs — Networks: https://ethereum.org/en/developers/docs/networks/
- Ethereum history and upgrades: https://ethereum.org/en/history/
- L2Beat — L2 risk and trust assumptions: https://l2beat.com/

**Background**

- Lamport, Shostak & Pease, *The Byzantine Generals Problem* (1982) — https://lamport.azurewebsites.net/pubs/byz.pdf
- Adam Back, *Hashcash* (2002) — http://www.hashcash.org/hashcash.pdf
- Go docs — why map iteration order is randomised: https://go.dev/blog/maps

---

**Examples:** [`examples/01-introduction/`](examples/01-introduction/) — **12 runnable Go programs**
(🟢 5 easy · 🟡 4 medium · 🔴 3 hard). They build one argument in order: a ledger is a log → a naive
log double-spends → hashing detects tampering → but ordering decides validity → agreement needs
determinism and costs O(n²) → so replicas diverge → and one shared ordering rule fixes it.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
