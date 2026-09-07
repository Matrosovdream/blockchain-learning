# 09 — Proof of Work & Mining

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#09-proof-of-work-mining)

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [08](08-blocks-and-chain.md) |
| **Examples** | [18](examples/09-proof-of-work/) (🟢 5 · 🟡 8 · 🔴 5) |

*The difficulty target, nonce grinding, compact `bits`, retargeting, and the honest cost discussion.*

Lesson 08 built a chain where tampering is *detectable*. This lesson makes it *expensive* — which is
the difference between a data structure and a consensus mechanism. It is also the lesson that gets
misdescribed most often, so a fair amount of it is spent on what proof of work does **not** do.

[Example 18](examples/09-proof-of-work/3-hard.md#18-proof-of-work-in-the-chain) folds all of it into
lesson 08's chain, and the diff is about forty lines.

## Goals

- Implement a PoW miner that finds a nonce below a target.
- Convert between difficulty, target and the compact `bits` encoding.
- Retarget difficulty from observed block times.
- Explain what PoW buys (Sybil resistance, objective ordering) and what it costs.

## Concepts

### 1. The puzzle

Find a nonce such that the header's hash, read as a 256-bit integer, is below a target:

```go
for {
    h.Nonce++
    sum := h.Hash()
    if new(big.Int).SetBytes(sum[:]).Cmp(target) < 0 {
        break
    }
}
```

That is the whole algorithm. Three properties make it work.

**It is asymmetric.** Example [1](examples/09-proof-of-work/1-easy.md#1-the-puzzle) takes 57,516
hashes to find a solution and one hash to check it. That ratio is what lets every node police every
miner for free — verification cost is what makes the system open.

**It is memoryless.** Each nonce is an independent trial, so no attempt teaches you anything about the
next. A miner who has tried a billion nonces is no closer than one who just started. This is why
hashrate translates *directly* into a share of blocks, with no advantage to having been mining longer.

**The work is bound to one exact header.** Every field is inside the hash, so a miner cannot reuse a
solution for a different block, or change anything after the fact. Example
[2](examples/09-proof-of-work/1-easy.md#2-verification-is-one-hash) tries changing the nonce,
timestamp, height and Merkle root, and each one invalidates the proof.

It is worth being precise about how this differs from a signature ([06](06-keys-signatures.md)). A
signature says *who authorised* something and needs an identity. Proof of work says *how much this
cost to produce* and needs no identity at all — which is exactly why it works in a network anyone can
join. This is [01](01-introduction.md)'s Sybil resistance made concrete.

### 2. Target, difficulty and bits

The target is a 256-bit number and smaller means harder. But the header has only four bytes for it
([08](08-blocks-and-chain.md) — fixed-size fields), so Bitcoin packs it as a base-256 float:

```
target = mantissa × 256^(exponent − 3)

0x1d00ffff  →  exponent 0x1d = 29, mantissa 0x00ffff = 65535
            →  00000000ffff0000000000000000000000000000000000000000000000000000
```

**Difficulty** is then a ratio: `max_target / target`. Difficulty 1 is Bitcoin's easiest allowed
target, which about 1 in 2³² hashes beats — and that 2³² factor is where the "hashes per block ≈
difficulty × 2³²" rule in topic 7 comes from. Example
[3](examples/09-proof-of-work/1-easy.md#3-compact-bits-to-a-256-bit-target) decodes several real
historical values.

Encoding *back* has two subtleties, both in example
[6](examples/09-proof-of-work/2-medium.md#6-target-back-to-bits-canonically):

- **The sign bit.** The top mantissa bit is reserved, so a value with it set must be shifted right and
  the exponent incremented.
- **The encoding is not unique.** `0x03000001` and `0x01010000` both decode to the target 1. A node
  must require the **canonical** spelling — otherwise the same difficulty could be written two ways,
  and two different block hashes would both be valid.

**Now the part that trips people up.** "Find a hash with N leading zeros" is how proof of work is
always explained, and it is not the rule. The rule is an integer comparison, and the two differ
whenever the target is not an exact power of two. Example
[5](examples/09-proof-of-work/1-easy.md#5-leading-zeros-is-a-picture-not-the-rule) shows two hashes
with the *same* leading-zero count on opposite sides of the target:

```
target  00000000ffff0000...
A       00000000aaaa0000...   < target: true
B       00000000ffff0000...1  < target: false   ← same zero count, still invalid
```

A hex-prefix test is worse still: it can only express targets that are powers of 16, so difficulty
could only move in 16× jumps — while retargeting needs adjustments of a few percent. The one line
that is always right:

```go
new(big.Int).SetBytes(hash[:]).Cmp(target) < 0
```

Finally, each extra bit of difficulty **doubles** the expected work. A single run tells you almost
nothing because the counts scatter enormously, so example
[4](examples/09-proof-of-work/1-easy.md#4-each-bit-doubles-the-work) averages 30 runs at each level
to show the law emerging.

### 3. The mining loop in Go

The loop runs billions of times and changes four bytes. Build the header once, patch the nonce in
place, and reuse everything:

```go
buf := h.Bytes()
h1, h2 := sha256.New(), sha256.New()
d1 := make([]byte, 0, sha256.Size)
d2 := make([]byte, 0, sha256.Size)
var val big.Int

for nonce := uint32(0); ; nonce++ {
    binary.BigEndian.PutUint32(buf[80:84], nonce)   // patch, never rebuild
    h1.Reset(); h1.Write(buf); d1 = h1.Sum(d1[:0])  // reuse the hasher
    h2.Reset(); h2.Write(d1);  d2 = h2.Sum(d2[:0])
    if val.SetBytes(d2).Cmp(target) < 0 {           // reuse the big.Int
        return nonce
    }
}
```

Example [7](examples/09-proof-of-work/2-medium.md#7-the-allocation-free-mining-loop) measures the
naive version at 1 allocation per attempt and this one at 0, using `testing.AllocsPerRun` — which is
deterministic, unlike wall-clock timing. This is [04](04-hash-functions.md)'s example 19 pattern at
full scale.

Be honest about why this matters: **it does not make a Go miner competitive.** Real mining is ASICs.
It matters because the identical pattern applies to *validating* millions of blocks during sync
([57](57-high-throughput-ingestion.md)).

**The nonce runs out.** `uint32` gives 4.3 billion values, and at mainnet difficulty a miner exhausts
that in well under a millisecond. So it needs more search space, in this order:

| Source | Cost |
|---|---|
| `Nonce` | free — increment |
| **extra nonce** in the coinbase | changes the Merkle root, so the Merkle path must be recomputed |
| `Timestamp` | can roll forward, within lesson 08's MTP and 2-hour bounds |
| the transaction set | different transactions, different root |

Example [8](examples/09-proof-of-work/2-medium.md#8-when-4-billion-nonces-are-not-enough) limits the
nonce to 8 bits so you can watch exhaustion happen, then loops the extra nonce. This is why the
coinbase has an arbitrary data field at all — the same field that carried Satoshi's newspaper headline
([08](08-blocks-and-chain.md)).

### 4. Parallel mining

Shard the nonce space across cores. A **strided** split — worker `w` takes nonces `w`, `w+N`, `w+2N` —
beats splitting into contiguous blocks, because it keeps every worker busy even when the solution
sits early in the space.

Example [14](examples/09-proof-of-work/3-hard.md#14-parallel-mining) gets three things right:

1. **A buffered result channel, sized for every worker.** If it were unbuffered, a worker that finds a
   solution *after* the winner has been read would block forever on the send.
2. **`cancel()` before `wg.Wait()`.** Cancel only *asks*; the wait is what guarantees no worker
   outlives the function.
3. **A periodic cancellation check, not per-hash.** A `select` on every iteration would cost more than
   the hashing.

Get 1 and 2 wrong and you have example
[15](examples/09-proof-of-work/3-hard.md#15-the-goroutine-leak): twenty calls leak steadily, and every
leaked goroutine is still burning CPU on a block that was solved long ago. During sync this function
runs constantly, so the leak compounds within a minute. Catch it with `runtime.NumGoroutine`,
`go test -race`, `goleak`, or pprof's goroutine profile.

Note that the winning nonce is **not** printed in these examples: which worker gets there first
depends on the scheduler. That is fine — many nonces satisfy any target, and a block needs one of
them, not a particular one. Speedup is also sub-linear, because of memory bandwidth and thermal
limits.

### 5. Difficulty retargeting

Hashrate changes, so the target must move to keep block times near schedule:

```
newTarget = oldTarget × actualTimespan / targetTimespan
```

Blocks came fast ⇒ actual < expected ⇒ smaller target ⇒ harder. Bitcoin does this every 2016 blocks
(two weeks at ten minutes), clamped to 4× in either direction so one period cannot move difficulty
too far.

Two rules matter in the implementation, and example
[9](examples/09-proof-of-work/2-medium.md#9-retargeting-integer-only) follows both:

- **Integer arithmetic only.** A float division can round differently across platforms, compilers or
  optimisation levels. Two nodes computing two different targets is a chain split with no attacker
  involved. `big.Int` Mul-then-Div is exact and identical everywhere.
- **Multiply first, divide second.** Dividing first truncates away the precision —
  [03](03-bytes-encoding.md)'s fixed-point rule.

**The timewarp attack** exploits the fact that the retarget reads timestamps, and miners choose those.
Bitcoin measures the span from the *first* to the *last* block of a period, so an attacker with
majority hashrate can backdate everything except the final block, making the chain believe blocks are
slow. Example [10](examples/09-proof-of-work/2-medium.md#10-the-timewarp-attack) drives difficulty
down 4× per period — 4096× over six periods, mined in hours.

What stops it: median-time-past from [08](08-blocks-and-chain.md) (the main brake — timestamps cannot
simply run backwards), the two-hour future limit, the 4× clamp, and the fact that it needs majority
hashrate anyway. Bitcoin also has a known off-by-one here — the span covers 2016 blocks but only 2015
intervals — kept for compatibility. Testnet has seen real timewarp exploitation; mainnet has not.

> Timestamp rules are consensus-critical, not bookkeeping.

### 6. Statistics of block times

Block discovery is a Poisson process, so the gap between blocks is **exponentially distributed**. The
consequences surprise almost everyone, and example
[11](examples/09-proof-of-work/2-medium.md#11-block-times-are-exponential) simulates 100,000 of them:

| | |
|---|---|
| Mean | 600 s — the target, as designed |
| **Median** | **414 s** — about 0.69× the mean |
| Blocks faster than target | **63%** |
| Longest of 100,000 | 116 minutes |

And it is memoryless: of blocks that already took over ten minutes, the same ~37% take over twenty.
Waiting does not make the next block more likely — there is no "due" block, exactly as in topic 1.

So a 40-minute gap is normal rather than evidence of an attack, "about 10 minutes" is a terrible basis
for a timeout, and you can conclude nothing about hashrate from a single block.

Which is why safety is measured in **confirmations**, not minutes. Example
[12](examples/09-proof-of-work/2-medium.md#12-confirmations-not-minutes) implements the Bitcoin
whitepaper's section 11 calculation — the probability an attacker with `q` of the hashrate ever
catches up from `z` behind:

| z | q=10% | q=30% | q=45% |
|---|---|---|---|
| 1 | 0.2046 | 0.6277 | 0.9198 |
| 6 | 2.43e-04 | 0.1321 | 0.7661 |
| 20 | 2.46e-12 | 0.0025 | 0.5366 |

The `z=6, q=10%` cell is 0.0002428 — the figure Satoshi published. Note that at `q ≥ 50%` the table
stops meaning anything: a majority attacker catches up with certainty given time, so more
confirmations buy delay, not safety.

Choosing a confirmation count is therefore an **economic** decision, not a cryptographic one: what
does an attack cost at this chain's hashrate, what is the transaction worth, and how long will the
counterparty wait? A small chain may need hundreds where Bitcoin needs six.

### 7. What PoW actually provides

Hashrate follows revenue, because miners spend up to what they earn:

```
hashes per block ≈ difficulty × 2³²
hashrate         ≈ hashes per block / block time
hashrate         ≈ (block reward + fees) / cost per hash
```

So **a chain's safety is proportional to what it pays miners** — not to how clever its cryptography
is. Example [13](examples/09-proof-of-work/2-medium.md#13-hashrate-difficulty-and-the-security-budget)
works the arithmetic and prints Bitcoin's halving schedule, where the subsidy trends to zero and fees
must eventually carry the entire budget.

That framing explains why small proof-of-work chains are cheap to attack even when they use the same
algorithm as a large one: the attack hashrate can be **rented** rather than built.

Now the honest accounting of what proof of work does:

- **It provides Sybil resistance** — influence costs money, so identities are not free.
- **It provides an objective ordering** anyone can verify without trusting anyone.
- **It says nothing about transaction validity.** Every node still checks every rule
  ([08](08-blocks-and-chain.md)). A miner with 100% hashrate cannot spend coins they have no key for,
  cannot create coins, and cannot make an invalid block valid.
- **It is not "security" in the abstract.** It is a specific, priced resistance to *history being
  rewritten*.

### 8. Attacks and criticism

**51% double-spend.** Wait for confirmations, then mine a private branch omitting the payment. Example
[16](examples/09-proof-of-work/3-hard.md#16-simulating-a-51-double-spend) simulates 20,000 attempts
per cell and matches the theory. This has happened repeatedly: Bitcoin Gold (May 2018, ~$18M, and
again in 2020) and Ethereum Classic (January 2019 and twice in August 2020, one reorg over 7,000
blocks deep). In each case the chain shared an algorithm with a much larger one, so hashrate was
rentable.

**Selfish mining** (Eyal & Sirer, 2013) is the more theoretically interesting result. A miner who
*withholds* blocks and releases them to invalidate honest work can earn **more than their fair share**
while holding well under half the hashrate. Example
[17](examples/09-proof-of-work/3-hard.md#17-selfish-mining) implements their SM1 state machine and
reproduces the published thresholds:

| γ (honest miners who build on the selfish block during a tie) | Profitable above |
|---|---|
| 0 | ⅓ of hashrate |
| 0.5 | ¼ |
| 1 | almost any share |

The significance is that this lowered the assumed safety threshold from 50% to nearer 25–33%. In
practice it is publicly detectable through an unusual orphan rate, it damages the coin the attacker is
paid in, and fast propagation keeps γ low — which is part of why compact block relay and dedicated
relay networks exist ([13](13-p2p-networking.md)).

**The energy question**, stated fairly. The electricity is not incidental — it *is* the security. A
chain whose blocks are cheap to produce is cheap to rewrite. So the debate is not "is this wasteful?"
in isolation but "is the property being bought worth this price, and is there a cheaper way to buy
it?" Proof of stake ([28](28-proof-of-stake.md)) argues there is: bond capital *inside* the system
instead of burning energy outside it, which also allows misbehaviour to be **punished** — something
proof of work cannot do. Ethereum's Merge cut its energy use by roughly 99.95%, and it is the reason
Part 6 exists.

## Exercises

Continue the program from [08](08-blocks-and-chain.md) in `practice/`.

1. **Mine your first block.** Add `Bits`, `BitsToTarget` and `CheckPoW` to your chain, and mine at
   16-bit difficulty. Assert the resulting block passes `CheckStateless` and that bumping the nonce
   makes it fail.
2. **Round-trip the encoding.** Implement `TargetToBits` including the sign-bit rule. Table-test it
   against five real `bits` values, and add a case asserting a non-canonical input comes back
   canonical.
3. **Prove the doubling.** Mine 50 headers each at 8, 10, 12 and 14 bits and report the mean attempts
   against 2ⁿ. Then report the standard deviation and explain why one run tells you nothing.
4. **Make it allocation-free.** Write the tight loop and assert `testing.AllocsPerRun` returns 0. Then
   benchmark it against the naive version with `-benchmem`.
5. **Retarget.** Implement the rule with integer math and clamping, and wire it into your chain's
   `Append` so a block with wrong `Bits` is rejected. Test: on-target, 2× fast, 2× slow, and both
   clamp boundaries.
6. **Mine in parallel.** Write the strided worker pool with context cancellation. Then write a test
   using `runtime.NumGoroutine` that fails if any worker outlives the call — and confirm it fails when
   you remove the `wg.Wait()`.
7. **Simulate the statistics.** Generate 100,000 exponential inter-block times and print the
   histogram. Then compute what fraction of 6-block confirmation windows take longer than an hour.
8. **Attack your own chain.** Simulate a double-spend at 30% hashrate against 1, 3 and 6
   confirmations, over 10,000 trials each. Compare your numbers with the whitepaper's formula and
   explain any divergence.

## Best Practices & Pitfalls

- **Compare hashes as big integers, never as hex strings or zero counts.**
  *Why:* a prefix test can only express targets that are powers of 16, and a zero count cannot
  distinguish two hashes on opposite sides of a real target (example 5).
- **Never use floating point in the retarget calculation.**
  *Why:* rounding can differ across platforms and compilers, so two honest nodes compute two different
  targets — a chain split with no attacker involved.
- **Multiply before dividing in the retarget.**
  *Why:* dividing first truncates away the precision the adjustment depends on.
- **Require the canonical `bits` encoding.**
  *Why:* the compact form is not unique. Accepting two spellings of one difficulty means two valid
  block hashes for the same block.
- **Buffer the result channel for every worker, and `wg.Wait()` after `cancel()`.**
  *Why:* an unbuffered channel blocks any worker that finishes after the winner is read, and cancel
  only asks — leaked goroutines keep hashing (example 15).
- **Patch the nonce bytes; never re-serialize the header per attempt.**
  *Why:* one allocation per attempt is billions per second in a miner, and the same pattern dominates
  block validation during sync.
- **Check cancellation periodically, not on every hash.**
  *Why:* a `select` per iteration costs more than the hashing it guards.
- **Never treat block time as a clock.**
  *Why:* inter-block times are exponential — 63% arrive early and hour-long gaps are routine. Count
  confirmations, not minutes.
- **Do not claim proof of work secures transaction validity.**
  *Why:* it secures *ordering*. Validity is enforced by every node independently, and a 100%-hashrate
  miner still cannot spend your coins.
- **Scale confirmation depth to the chain, not to habit.**
  *Why:* six confirmations reflects Bitcoin's hashrate economics. On a small chain sharing an
  algorithm with a large one, hashrate is rentable and six is nowhere near enough.

## Checklist

- [ ] I can implement a miner that finds a nonce below a target.
- [ ] I can explain why the search being memoryless makes hashrate share equal block share.
- [ ] I can expand `bits` to a target and compress a target back canonically.
- [ ] I can explain why the sign-bit rule exists and why non-canonical bits must be rejected.
- [ ] I know why "leading zeros" is a picture and the integer comparison is the rule.
- [ ] I can write an allocation-free mining loop and prove it with `AllocsPerRun`.
- [ ] I know why a `uint32` nonce is not enough and where the extra nonce lives.
- [ ] I can implement retargeting with integer-only arithmetic and clamping.
- [ ] I can explain the timewarp attack and which lesson 08 rule blocks it.
- [ ] I can explain why block times are exponential and what that means for timeouts.
- [ ] I can state the 51% and selfish-mining thresholds and what each attacker can actually do.
- [ ] I can explain what proof of work provides, what it costs, and what it does not do.

## Resources

**Specifications**

- Bitcoin developer guide — Proof of Work: https://developer.bitcoin.org/devguide/block_chain.html
- Bitcoin — target nBits: https://developer.bitcoin.org/reference/block_chain.html#target-nbits
- Bitcoin whitepaper §11 (the attacker race): https://bitcoin.org/bitcoin.pdf

**Papers**

- Eyal & Sirer, *Majority is not Enough: Bitcoin Mining is Vulnerable*: https://arxiv.org/abs/1311.0243
- Sompolinsky & Zohar, *Secure High-Rate Transaction Processing in Bitcoin* (GHOST): https://eprint.iacr.org/2013/881

**Go**

- `math/big`: https://pkg.go.dev/math/big
- `context`: https://pkg.go.dev/context
- `go.uber.org/goleak`: https://github.com/uber-go/goleak

**Incidents**

- Ethereum Classic 51% attacks (2019, 2020): https://ethereumclassic.org/blog/2020-08-07-ecip-1100
- Bitcoin Gold double-spend (2018): https://forum.bitcoingold.org/t/double-spend-attacks-on-exchanges/1362

---

**Examples:** [`examples/09-proof-of-work/`](examples/09-proof-of-work/) — **18 runnable Go programs**
(🟢 5 easy · 🟡 8 medium · 🔴 5 hard), standard library only. Example 12 reproduces the whitepaper's
published catch-up probabilities; example 17 reproduces Eyal & Sirer's selfish-mining thresholds;
example 18 folds proof of work into lesson 08's chain in about forty lines of diff.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
