# 11 — Wallets, Fees & the Mempool

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#11-wallets-fees-the-mempool)

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [07](07-addresses-wallets-hd.md), [10](10-transactions-utxo.md) |
| **Examples** | [18](examples/11-wallets-mempool/) (🟢 5 · 🟡 8 · 🔴 5) |

*a keystore, transaction construction, the fee market, mempool policy and block assembly*

Lesson 10 built transactions and the state they mutate. It said nothing about where transactions come
from, how they reach a miner, or how a miner chooses between them. That is this lesson, and it is the
first one in Part 3 where **none of the code is consensus**.

That distinction is the spine of the whole thing. Delete the wallet and the mempool and the chain
still validates every block correctly — it just has nothing to put in one. Which means every rule
here is a local judgement call, every one of them is configurable, and a transaction your node
refuses can be mined by someone else's without anything being wrong.

[Example 18](examples/11-wallets-mempool/3-hard.md#18-a-wallet-a-mempool-and-a-miner) puts a wallet on
one side of lesson 10's chain and a mempool on the other.

## Goals

- Build a wallet that stores keys and constructs spendable transactions.
- Implement a mempool with validation, replacement and eviction.
- Select transactions for a block by fee rate.
- Explain fee estimation and replace-by-fee.

## Concepts

### 1. What a wallet is

A wallet holds no coins. It holds two things:

1. **Keys** — the ability to produce signatures.
2. **A view** — which unspent outputs those keys can open.

The coins are in the UTXO set ([10](10-transactions-utxo.md)), which belongs to the network. Lose the
view and you rescan; lose the keys and the money stays perfectly visible to everyone, forever.

Separating those two halves is what makes a **watch-only wallet** possible. BIP-32's public derivation
([07](07-addresses-wallets-hd.md)) lets an account's extended *public* key derive every address the
private key would, so a machine holding only the xpub can watch deposits, build unsigned transactions
and reconcile balances — and an attacker who takes that whole machine gets no ability to move
anything. This is not a niche feature; it is how every custody team is organised.

```go
acct, _ := master.Derive(...)  // m/44'/60'/0'
pub, _  := acct.Neuter()       // the same addresses, none of the authority
```

The **address book** has two branches, not one: `m/.../0/i` for receiving and `m/.../1/i` for change.
Keeping change on its own branch means it never lands on an address you have handed out, which
matters both for your own bookkeeping and for the analyst reading yours ([10 §8](10-transactions-utxo.md#8-coin-selection)).

A wallet does not know how many addresses it has used — the *chain* does. So restoring one means
scanning forward until **`GapLimit`** consecutive addresses have never been paid, and BIP-44 fixes
that at 20. [Example 1](examples/11-wallets-mempool/1-easy.md#1-what-a-wallet-actually-holds) pays
indices 0, 1, 3 and 40, and the rescan finds three of them. Index 40 is simply invisible. Nearly every
"my funds vanished after restore" report is this or a wrong derivation path.

Model the signing half behind an interface from the start, so the key can move to a hardware device or
a remote HSM without the spend path noticing:

```go
type Signer interface {
    PubKey(chain, index uint32) []byte
    Sign(chain, index uint32, hash []byte) ([]byte, error)
}
```

### 2. Persisting keys

Four things have to be right when a key goes on disk, and three of them are not cryptography.
[Example 2](examples/11-wallets-mempool/1-easy.md#2-a-key-on-disk) does all four in the shape of a
Web3 Secret Storage (keystore v3) file — the same format geth writes.

**Derive slowly.** A password is not a key. Put it through scrypt at N=2¹⁸, r=8, p=1, which costs
about 256 MB and a second — charged to *every guess an attacker makes* and paid once by you. The
memory cost is the point: it is what makes GPU and ASIC cracking uneconomic in a way PBKDF2 is not.

**Seal with an AEAD.** AES-256-GCM, with the account address as additional authenticated data so a
file cannot be relabelled to point at a different account. A wrong password, a flipped bit and a
tampered address field then all fail identically, and there is no oracle to probe. Plain AES-CTR would
have "decrypted" a corrupted file into 32 bytes of garbage — a perfectly valid private key for an
account holding nothing.

**Write it atomically.** Temp file in the same directory, `chmod 0600` *before* any bytes go in,
write, `fsync`, rename, then `fsync` the directory. Rename within a directory is atomic, so a reader
sees the old file or the new one and never a half-written one — and the final directory sync is what
makes the rename itself survive a power cut.

```go
f, _ := os.CreateTemp(dir, ".keystore-*.tmp")
f.Chmod(0o600)   // before any bytes are in it
f.Write(data)
f.Sync()         // the bytes, not just the page cache
f.Close()
os.Rename(tmp, path)
d, _ := os.Open(dir); d.Sync()   // the rename, too
```

**Never log the secret.** A type whose `String`, `Format` and `MarshalJSON` all redact costs ten lines
and closes off `%v`, `%s`, `%x`, `%+v`, `log.Print` and `encoding/json` at once:

```go
type Secret []byte

func (s Secret) String() string                  { return "<redacted>" }
func (s Secret) Format(f fmt.State, verb rune)   { fmt.Fprint(f, "<redacted>") }
func (s Secret) MarshalJSON() ([]byte, error)    { return []byte(`"<redacted>"`), nil }
func (s Secret) Reveal() []byte                  { return []byte(s) } // deliberately ugly to type
```

All of this protects a key **at rest**. It does nothing about a key in memory, in a swap file, in a
core dump, or in the shell history of whoever typed the password. Lesson [44](44-symmetric-crypto-at-rest.md)
covers those, and the honest answer for real money is a device that never exports the key at all.

### 3. Building a spend

Six stages, and the order is not negotiable:

1. select coins
2. estimate the size — from the *shape*, before signing
3. compute the fee — rate × size
4. create the change, or fold it into the fee if it would be dust
5. sign every input
6. check your own work

**Signing is last** because signing freezes everything ([10](10-transactions-utxo.md)). A wallet that
signs early and adjusts the fee afterwards produces transactions that every node silently drops.

Stages 1–3 are circular: the fee needs the size, the size needs the inputs, the inputs need the fee.
[Example 7](examples/11-wallets-mempool/2-medium.md#7-the-feesize-circularity) shows three ways to
break that. Selecting for the amount and adding the fee afterwards simply comes up short. Iterating to
a fixpoint works, until it doesn't: on a wallet of 200 dust coins it grinds through 47 rounds to 134
inputs and 382,000 satoshi of fees to move 20,000.

The fix is to stop treating the fee as separate from the coin. A coin's **effective value** is what it
is worth after paying for the input that spends it:

```go
func effectiveValue(c Coin, rate int64) int64 { return c.Value - rate*inputSize }
```

Now the target is fixed, one pass suffices, and the same dust wallet gets a *better* answer — 130
inputs and 12,000 satoshi less. It also makes the wallet honest with the user: at 200 sat/byte more
than half of a typical wallet has negative effective value and is not money any more.

**Dust change** should become fee. If the change output is worth less than it will cost to spend,
creating it makes you poorer, adds 31 vbytes now, and leaves a permanent entry in the global UTXO set.
Never create change you would not pay to spend.

[Example 6](examples/11-wallets-mempool/2-medium.md#6-building-a-spend-end-to-end) walks the whole
builder printing its working at every stage, and note where it fails when the wallet cannot pay:
*before* any signature exists. A wallet that discovers it is short only after signing has leaked its
public keys for nothing and is holding a signed object it must be careful not to broadcast.

### 4. The mempool

An in-memory set of transactions a node has validated but not seen in a block. It is **not
consensus**: every node has a different one, none of them agree, none of it survives a restart, and
two transactions in it can conflict. Only a block decides.

Two indexes, because there are two questions:

```go
type Mempool struct {
    byID    map[[32]byte]*Entry   // "do I already have this?"   O(1)
    claimed map[Outpoint][32]byte // "does this conflict?"        O(1)
    outputs map[Outpoint]TxOutput // outputs created by pool transactions
}
```

plus an order by fee rate — a `container/heap` when you pop the worst repeatedly (eviction,
[§8](#8-replacement-and-eviction)) and a sorted walk when you take the best once per block
([§7](#7-block-assembly)). Compare rates by cross-multiplying integers, never by dividing: two nodes
that round differently disagree for no reason.

**Admission** runs cheapest-first, because everything here is attacker-supplied
([10](10-transactions-utxo.md)):

1. structure and size — no cryptography
2. do its inputs exist? — map lookups
3. does it conflict with something already here? — map lookups
4. do the signatures verify? — the expensive one, last
5. does it pay enough? — policy

That ordering is a security property, not an optimisation: it must cost a node less to reject a
megabyte of garbage than it cost the attacker to send it.

Step 2 has a subtlety. An input may point at an output of a transaction *already in the pool* — that
is how a chain of unconfirmed transactions exists at all — so the lookup checks the UTXO set and then
the pool's own outputs. And if neither has it, the transaction may simply have arrived before its
parent, which happens constantly. Hold it in an **orphan pool**, indexed by what it is waiting for,
and retry when that outpoint appears. Bound it: an unbounded orphan pool is a free
memory-exhaustion attack.

[Example 9](examples/11-wallets-mempool/2-medium.md#9-mempool-admission) runs all of it, including the
promotion of an orphan the moment its parent arrives.

### 5. Mempool policy vs consensus rules

Two different words that both get called "valid".

**Consensus** is global and binding. Break it and any block containing your transaction is invalid,
and every node agrees. Changing these rules is a fork.

**Policy** is local and advisory. It decides what *this* node is willing to keep and relay. Every node
may set it differently. Changing it is a config flag.

[Example 10](examples/11-wallets-mempool/2-medium.md#10-policy-is-not-consensus) runs nine
transactions through both validators. Six are consensus-valid and still will not be relayed: too
cheap, dust change, a non-standard lock, too many outputs, too long a chain of unconfirmed ancestors.
A block containing any of them is perfectly good.

So the transaction is not *rejected*. It is **ignored**. It does not propagate, it does not appear in
any explorer, and it produces no error anywhere the sender can see. From the wallet's point of view it
evaporated — which is the entire content of a large fraction of support tickets.

| policy | Bitcoin Core | Ethereum (geth txpool) |
|---|---|---|
| minimum fee to relay | `minRelayTxFee`, 1 sat/vB | `--txpool.pricelimit`, 1 wei |
| replacement price bump | BIP-125 rules ([§8](#8-replacement-and-eviction)) | `--txpool.pricebump`, 10% |
| per-sender queue limit | n/a | `--txpool.accountslots`, 16 |
| total pool size | `-maxmempool`, 300 MB | `--txpool.globalslots`, 4096 |
| chain of unconfirmed | 25 ancestors, 25 descendants | nonce gap → "queued" |
| dust | 546 / 294 sat by script type | n/a — no UTXOs |
| exotic scripts | `IsStandard()`: fixed templates | n/a |
| expiry | `-mempoolexpiry`, 336 hours | `--txpool.lifetime`, 3 hours |

The distinction exists because consensus rules are expensive to change and must be identical
everywhere, so they are kept minimal. Everything that is merely a good idea goes in policy, where it
can be tuned per node and changed in a point release. Policy is also where new features are *tested*:
a rule is usually policy for years before anyone proposes making it consensus.

Three consequences worth internalising. Never report "invalid" when you mean "not relayed" — different
problem, different fix. A transaction no node will relay can still be mined by handing it to a miner
directly, which is what Bitcoin "accelerator" services and Ethereum's private order flow are. And
therefore **you cannot rely on policy for safety**: if something must never happen, it has to be a
consensus rule.

### 6. Fees

A fee is not a price. It is a **bid for space in the next block**. Blocks are capped by size, not by
transaction count, so a miner filling one is solving "most fees per byte" — which makes the fee
**rate** the number that decides everything and the absolute fee nearly irrelevant.

[Example 3](examples/11-wallets-mempool/1-easy.md#3-fee-rate-not-fee) sorts the same mempool both ways.
The biggest payer in the pool falls to second-last by rate, and a miner who sorts by absolute fee
collects 40,000 satoshi where one sorting by rate collects 98,260 — same transactions, same block.
Ethereum's arithmetic is identical with gas in place of bytes: a swap and a transfer paying the same
fee are not the same bid, because the swap is asking for seven times as much block.

So: **pick a rate and let the size decide the fee**, never the other way round.

Which requires knowing the size before the transaction exists, because the signatures are not made
yet. [Example 4](examples/11-wallets-mempool/1-easy.md#4-estimating-a-size-that-does-not-exist-yet)
pays for the unsigned size at a target of 20 sat/byte and achieves 9. Estimate from the *shape*:

```go
func EstimateSize(t *Transaction) int {
    return len(t.Serialize()) + len(t.Inputs)*(sigBytes+pubKeyBytes)
}
```

Real estimates are never exact. A Bitcoin signature is DER-encoded and DER stores `r` and `s` as
signed integers, so a 32-byte value with its top bit set needs a `0x00` pad. Low-s enforcement
([06](06-keys-signatures.md)) keeps `s` short, but `r` needs the pad about half the time — 71 or 72
bytes, occasionally 70. Estimate the *worst* case or you fall under your target rate whenever you get
unlucky. And since segwit, `weight = 4×base + witness` and `vsize = ceil(weight/4)`, so a P2WPKH input
costs 68 vbytes against a legacy P2PKH input's 148. An estimator that does not know which script types
it is spending is guessing by a factor of two.

**Estimating what rate to pay** is a different problem with no oracle. The method is Bitcoin Core's:
bucket recent confirmations by fee rate, record how many in each bucket confirmed within N blocks, and
for a target of N return the lowest bucket that clears some threshold —
[example 11](examples/11-wallets-mempool/2-medium.md#11-estimating-a-fee-from-history) uses 85%, as
Core's "economical" mode does. Over 600 blocks of bursty demand it produces a clean curve: 8 sat/byte
for the next block, 3 for twenty-five.

Then demand triples, and all three transactions paying those estimates are still unconfirmed sixty
blocks later. The estimate was not wrong when it was made — it described the last 600 blocks
accurately, and then the world changed. **A fee estimate is a prediction about other people**, and
there is no version of it that is a promise. Which is why a wallet should show the *target* ("about 30
minutes") rather than a number pretending to certainty, and should always leave itself a way to revise
([§8](#8-replacement-and-eviction)).

### 7. Block assembly

Choosing a block's contents is the knapsack problem, which is NP-hard — and everyone ships greedy
anyway. [Example 12](examples/11-wallets-mempool/2-medium.md#12-filling-a-block) measures why: against
a brute-force optimum over 65,536 subsets, greedy-by-fee-rate is exactly optimal about half the time
and captures 98.2% on average. The loss is a fraction of a percent, the algorithm is one sort, and the
exact version is exponential in a set of 50,000.

Four things greedy has to get right.

**Skip, do not stop.** When the best remaining transaction does not fit, keep walking — smaller ones
behind it still do. This is the most common bug here and it is worth a lot: on the example's mempool,
stopping collects 148,704 satoshi where skipping collects 180,666.

**Ancestors.** A transaction whose parent is still unconfirmed cannot go in without it, and must go in
*after* it. So the unit of selection is a **package**, ranked by the combined fee rate of a
transaction and its unconfirmed ancestors — which is what makes [§8](#8-replacement-and-eviction)'s
child-pays-for-parent work at all.

**The reserve.** The coinbase has to fit. Assembling to the full cap and then adding it produces an
oversize block that every node rejects — after you mined it.

**Determinism.** Ties must break on something stable, or the same mempool produces different blocks on
different runs and nothing is testable.

Bitcoin's real numbers are 4,000,000 weight units against a pool of 20,000–100,000 transactions, with
Core keeping the pool pre-sorted by ancestor fee rate so assembly is close to a linear scan. Ethereum
is harder: transactions from one sender must go in nonce order, and the *value* of an ordering depends
on what the transactions do to each other. That is where MEV comes from ([40](40-defi-primitives-mev.md)), and why
block building there is an auction rather than a sort.

### 8. Replacement and eviction

**Replace-by-fee.** The estimate was wrong and the transaction is stuck. The sender publishes a
replacement spending the same inputs at a higher fee. BIP-125 imposes five rules, and every one exists
to stop the same attack — making a node do work, or give up bandwidth, for free:

1. the original signals replaceability (`nSequence < 0xfffffffe`)
2. the replacement adds no new unconfirmed inputs
3. it pays a higher *absolute* fee than everything it evicts
4. it additionally pays the relay rate for its own size
5. it evicts no more than 100 transactions

[Example 14](examples/11-wallets-mempool/3-hard.md#14-replace-by-fee) trips each in turn. Note rule 3
is about absolute fee, not rate: a smaller replacement at the same fee is refused. And note what a
bump costs — the *full* new fee, not the difference. A replacement replaces; it does not top up. Only
the last fee is ever actually paid, because the replaced transactions never confirm.

Rule 1 has since been relaxed: Bitcoin Core made full-RBF the default in v28, settling a long argument
by admitting what was already true — a miner was always free to take the higher fee, so the signal was
never a guarantee. An unconfirmed transaction is a proposal, not a payment.

**Child-pays-for-parent** is the other half, and it is the one that works when the money is coming
*to* you. You cannot replace a transaction you did not send, but you can spend one of its outputs at a
high rate and dare the miner to take the child without the parent.
[Example 16](examples/11-wallets-mempool/3-hard.md#16-child-pays-for-parent) shows a miner sorting by
each transaction's own rate skipping the highest-rate transaction in the pool — because its parent is
not in the block — and collecting 76,500 where a package-aware miner collects 119,428. CPFP costs more
than RBF, because the child pays for the parent's bytes as well as its own; it is the only option for
a receiver.

**Pinning** is what happens when someone else has an interest in which of your transactions wins.
Rule 3 requires beating the absolute fee of everything evicted, *including descendants* — so attaching
a huge, cheap child to a counterparty's transaction makes replacing it arbitrarily expensive, while
the package as a whole is too cheap for any miner to touch. Unconfirmable and unreplaceable at once.
[Example 15](examples/11-wallets-mempool/3-hard.md#15-transaction-pinning) attaches a 99 kB child and
watches the cost of replacing a 214-byte transaction go from 642 satoshi to 99,642 — 465 sat/byte. Rule
5 gives a second version: 110 tiny descendants, and *no* fee is large enough.

That turns a policy detail into a theft primitive, because most contracts with a counterparty have a
deadline: a Lightning HTLC must be claimed before its timelock expires, an atomic swap's refund branch
opens at a fixed height. The fixes arrived in layers — the CPFP carve-out in Core 0.19, then **TRUC /
v3 transactions** (BIP-431), which cap a v3 transaction at one unconfirmed child of at most 1000
bytes and bring the same attack down to 1,642 satoshi, and **package relay** in Core 28, which lets a
parent and child be submitted and judged together.

**Eviction.** A mempool that never forgets is a memory-exhaustion attack with extra steps. Three
defences, in [example 13](examples/11-wallets-mempool/2-medium.md#13-eviction-the-floor-and-expiry):

- **A byte budget**, enforced by dropping the lowest fee rate first — the transaction a miner would
  have taken last, which is the only ranking that matches what the pool is *for*.
- **A dynamic floor**, raised to whatever was just evicted, so the same traffic does not come straight
  back in. It decays on a timer (Core halves it every twelve hours), or one busy afternoon would lock
  a node out of relaying cheap transactions until restart. This is the mechanism behind "my fee was
  fine yesterday".
- **Expiry**, dropping anything older than N hours whatever it paid. That is not about memory — the
  fee-rate eviction handles memory. It is about the pool telling the truth: a transaction nobody has
  mined in two weeks is not going to be, and holding it makes the wallet think it is still pending.

Evicting a parent must evict its descendants, since their inputs no longer exist anywhere the pool can
see. Which means eviction should really rank by *descendant* fee rate, not an entry's own — the same
package view that makes CPFP work.

### 9. Concurrency

The mempool is written by the network layer and read by the miner, both continuously. Two standard
answers, and [example 17](examples/11-wallets-mempool/3-hard.md#17-one-pool-many-goroutines) builds
both behind one interface.

Keep the synchronisation *out* of the data structure. A plain `state` with no locking, wrapped by
either strategy, is what makes swapping them a thirty-line change instead of a rewrite.

```go
type MutexPool struct {
    mu sync.RWMutex
    st *state
}

type ActorPool struct {          // one goroutine owns st; everyone else sends closures
    cmds chan func(*state)
    done chan struct{}
}
```

`RWMutex` lets any number of miners build templates at once under `RLock`, which matters because this
workload is read-heavy. The owning goroutine cannot violate an invariant and cannot deadlock, and its
bounded channel gives you backpressure for free — but every operation, including reads, goes through
one goroutine. For a mempool: use the mutex. Reach for the actor when the invariants are complicated
enough that you do not trust yourself to hold the right lock every time.

**The mistake that matters is neither of those.** It is holding the lock across validation. A
signature check is tens of microseconds; do it inside the critical section and every other goroutine
waits through it, so eight cores admit transactions at the speed of one. The example instruments
exactly that: 100% of the work serialized, versus 0%.

```go
// wrong                             // right
p.mu.Lock()                          validate(t)          // no lock held
validate(t)                          p.mu.Lock()
p.st.add(t)                          p.st.recheckConflicts(t)  // the world moved
p.mu.Unlock()                        p.st.add(t)
                                     p.mu.Unlock()
```

The re-check is not optional: something else may have claimed the same outpoint while you were
verifying.

Note also what the example does *not* print: any timing. A number from one machine on one run, which
would change with `GOMAXPROCS` and the scheduler's mood, is worse than no number. Measure it yourself:

```bash
go test -race ./...                  # correctness first, always
go test -bench=Pool -cpu=1,2,4,8     # does it actually scale?
go test -bench=Pool -mutexprofile=m.out && go tool pprof -top m.out
```

Write the `-race` test first. A mempool bug that only appears under load is a bug you will meet in
production, at 3am, on the one node that had a fast peer.

## Exercises

Continue the program from [10](10-transactions-utxo.md) in `practice/`.

1. **A wallet with a view.** Give your wallet an account key, a receive branch and a change branch,
   and a `Rescan` that walks forward until 20 consecutive addresses are unused. Test: fund indices 0,
   1, 3 and 40 and assert the rescan finds exactly three.
2. **Watch-only.** Neuter the account key and assert the watch-only wallet derives the same first 50
   addresses and returns an error from every signing path.
3. **The keystore.** Seal a key with scrypt + AES-GCM, write it with the temp-file-and-rename dance,
   reload it and assert the re-derived address matches. Then table-test three failures — wrong
   password, flipped ciphertext bit, edited address field — and assert all three return the *same*
   error value.
4. **Redaction.** Write the `Secret` type and a test that formats it with `%v`, `%s`, `%x`, `%+v`,
   inside a struct, and through `json.Marshal`, asserting the key never appears.
5. **Estimate, then sign.** Write `EstimateSize` from the shape, and assert after signing that the
   real size is within a byte or two. Then deliberately estimate the unsigned size and assert the
   achieved fee rate is less than half the target.
6. **The builder.** Implement `Spend(to, amount, feeRate)` doing all six stages, using effective
   values so there is no loop. Test: the dust-change case drops the output, and an unaffordable
   payment fails before any signature is produced.
7. **Preflight.** Write the ten checks over a `Draft` carrying the user's intent. Then write the test
   that matters: a change output redirected to a stranger's key hash must be blocked, even though the
   transaction is entirely valid.
8. **Admission.** Implement `Accept` with the five checks in the right order, plus a bounded orphan
   pool. Test that a child accepted before its parent is held and then promoted, and that the orphan
   pool never exceeds its cap.
9. **Policy versus consensus.** Write both validators and a table test where at least four
   transactions are consensus-valid and policy-rejected. Then flip the policy config and assert they
   become acceptable without touching the transactions.
10. **Estimation.** Build the bucketed tracker and estimator. Simulate 500 blocks with a seeded RNG,
    produce estimates for targets 1, 3, 6 and 25, then triple the arrival rate and assert every one of
    them now misses its target.
11. **Assembly.** Implement greedy-by-ancestor-fee-rate under a size cap, with a coinbase reserve.
    Test: a package must appear parents-first; a transaction that does not fit must be skipped rather
    than ending the loop; and a brute-force optimum on 12 transactions must be within 5%.
12. **Eviction.** Add a byte budget, a `container/heap` on fee rate, a floor that rises on eviction
    and decays over time, and expiry. Test that evicting a parent removes its descendants and that the
    accounting (`count` and `bytes`) stays consistent.
13. **RBF.** Implement all five BIP-125 rules and write one test per rule that fails only that rule.
    Then write the pinning test: attach a large cheap child and assert the required replacement fee
    exceeds a sane bound.
14. **Concurrency.** Put your mempool behind an `RWMutex`, then behind an owning goroutine, and run
    the same concurrent add/select/evict workload against both under `go test -race`. Assert the final
    state is byte-identical. Then move validation outside the lock and benchmark both with
    `-cpu=1,2,4,8`.

## Best Practices & Pitfalls

- **Estimate the signed size, not the unsigned one.**
  *Why:* signatures are most of an input. Paying for the unsigned size buys under half the fee rate
  you asked for, and by then the transaction is signed — the only fix is to replace it.
- **Pick a fee rate, not a fee.**
  *Why:* "5000 sat" is a good bid on a small transaction and a terrible one on a big one. Blocks are
  sold by the byte.
- **Assert the real size matches the estimate after signing.**
  *Why:* it is one line, and it catches every future change to your serialization format before it
  reaches the fee calculation.
- **Never create change below the dust threshold.**
  *Why:* it costs more to spend than it holds, so it becomes permanent global state that nobody will
  ever remove. Give it to the miner instead.
- **Select on effective values.**
  *Why:* it removes the fee/size circularity entirely, gives a better answer than iterating, and lets
  the wallet tell the user what is actually spendable at today's rate.
- **Sign last, and re-sign from scratch when anything changes.**
  *Why:* the sighash covers every value, recipient and ordering. There is no such thing as editing a
  signed transaction.
- **Check the intent, not just the validity, before broadcasting.**
  *Why:* half of the ways a wallet can be wrong produce perfectly valid transactions. A change output
  to the wrong address is irreversible and the network will relay it happily.
- **Never hold the mempool lock across signature verification.**
  *Why:* validation is tens of microseconds. Inside the lock it serializes every admission, and the
  pool admits at the speed of one core no matter how many you have.
- **Re-check conflicts after validating and before inserting.**
  *Why:* you released the lock to verify, so another goroutine may have claimed the same outpoint.
- **Order admission checks cheapest-first.**
  *Why:* the input is attacker-supplied. Rejecting must cost you less than sending cost them.
- **Bound every pool: the mempool, the orphan pool, and the per-sender queue.**
  *Why:* each unbounded structure is a free memory-exhaustion attack against a public endpoint.
- **Evict by descendant fee rate, and take descendants with the parent.**
  *Why:* dropping a cheap parent while keeping its expensive children leaves entries that can never
  be mined or relayed, and throws away the fee that made them worth keeping.
- **Never say "invalid" when you mean "not relayed".**
  *Why:* they are different problems with different fixes, and the user cannot tell them apart from
  the outside.
- **Never rely on mempool policy for safety.**
  *Why:* it is local, configurable, and bypassable by handing the transaction to a miner directly. If
  something must never happen, it must be a consensus rule.
- **Treat mempool acceptance as nothing at all.**
  *Why:* it is one node's opinion. It is not a confirmation, not a guarantee of relay, and with
  full-RBF not even a guarantee that this transaction is the one that confirms.
- **Signal replaceability on everything.**
  *Why:* there is no cost, and without it the only way to fix a bad estimate is to wait.
- **Assume a counterparty will pin you.**
  *Why:* any protocol with a deadline has to survive an adversary who can make your fee bump
  arbitrarily expensive. Use TRUC, keep bumping paths short, and never design a timeout that assumes
  your transaction will confirm.

## Checklist

- [ ] I can explain what a wallet holds and what it does not, and why watch-only is useful.
- [ ] I can implement the gap limit and say what breaks without it.
- [ ] I can encrypt a key at rest with a slow KDF and an AEAD, and write it atomically.
- [ ] I can make a secret unloggable through every formatting path Go offers.
- [ ] I can estimate a transaction's signed size from its shape and prove the estimate afterwards.
- [ ] I can explain why fee rate is the only number that matters, in bytes or in gas.
- [ ] I can break the fee/size circularity with effective values and say why iterating is worse.
- [ ] I can list the pre-broadcast checks a node cannot do for me.
- [ ] I can write mempool admission in the right order and say why that order is a security property.
- [ ] I can explain the difference between policy and consensus and give three examples of each.
- [ ] I can build a fee estimator from confirmation history and explain exactly what it does not know.
- [ ] I can assemble a block by ancestor fee rate and say what greedy costs against the optimum.
- [ ] I can implement eviction, a dynamic floor and expiry, and say what each protects against.
- [ ] I can state the five BIP-125 rules and what each one is defending.
- [ ] I can explain pinning, why it is a theft primitive, and what TRUC changes.
- [ ] I can choose between an RWMutex and an owning goroutine, and I know the mistake that matters more
      than either.

## Resources

**Specifications**

- BIP-125 (opt-in replace-by-fee): https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- BIP-431 (TRUC / v3 transactions): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- BIP-44 (account structure and the gap limit): https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- Bitcoin Core mempool policy: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/README.md
- Web3 Secret Storage (keystore v3): https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/

**Design notes**

- Bitcoin Core's branch-and-bound coin selection (Erhardt, 2016): https://murch.one/erhardt2016coinselection.pdf
- Package relay design: https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- geth txpool flags: https://geth.ethereum.org/docs/fundamentals/command-line-options

**Go**

- `sync` (`RWMutex`, `WaitGroup`): https://pkg.go.dev/sync
- `container/heap`: https://pkg.go.dev/container/heap
- `golang.org/x/crypto/scrypt`: https://pkg.go.dev/golang.org/x/crypto/scrypt
- `crypto/cipher` (AEAD): https://pkg.go.dev/crypto/cipher
- Detecting data races: https://go.dev/doc/articles/race_detector

---

**Examples:** [`examples/11-wallets-mempool/`](examples/11-wallets-mempool/) — **18 runnable Go
programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Example 7 runs the fee/size loop on a wallet of dust
and watches it grind through 47 rounds; example 11 builds Core's fee estimator and then breaks it with
a demand spike; example 15 turns BIP-125 rule 3 into a pinning attack and fixes it with TRUC; example
18 puts a wallet and a mempool either side of lesson 10's chain.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
