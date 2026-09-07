# 08 — Blocks & the Chain

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#08-blocks-the-chain)

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) *(first lesson)* |
| **Prerequisites** | [04](04-hash-functions.md), [05](05-merkle-trees.md) |
| **Unlocks** | 09, 10 |
| **Examples** | [18](examples/08-blocks-and-chain/) (🟢 5 · 🟡 8 · 🔴 5) |

*The block struct, hash linking, genesis, deterministic serialization and chain validation.*

**Part 3 starts here.** Over the next eight lessons one Go program grows into a working blockchain:
blocks, mining, UTXO transactions, a wallet, persistence, a P2P network, and fork choice. This lesson
builds the skeleton — and [example 18](examples/08-blocks-and-chain/3-hard.md#18-the-whole-thing-assembled)
is that program's first version, ending with a list of exactly what each following lesson changes
about it.

Everything from Part 2 gets used immediately: Keccak and SHA-256 from [04](04-hash-functions.md),
Merkle roots with domain separation from [05](05-merkle-trees.md), deterministic encoding from
[03](03-bytes-encoding.md).

## Goals

- Define a block and a header in Go and hash it deterministically.
- Link blocks by previous-hash and detect any tampering.
- Create a genesis block and validate a whole chain.
- Separate header from body and know why that split exists.

## Concepts

### 1. Header vs body

A block has two halves, and they are treated completely differently.

The **header** is small, fixed-size, and is what gets hashed and gossiped. The **body** is the actual
transactions — large, variable, and fetched only when needed. The header commits to the body through
a single Merkle root, so the body cannot be swapped without the header changing.

The numbers make the case. Example
[5](examples/08-blocks-and-chain/1-easy.md#5-header-vs-body-what-a-light-client-downloads) builds a
3,000-transaction block:

| | Per block | 800,000 blocks |
|---|---|---|
| Header | 92 bytes | 73 MB |
| Body | ~732 KB | ~600 GB |

A light client downloads the headers, verifies the chain's structure entirely from them, and asks for
a single transaction plus a Merkle proof when it needs one ([05](05-merkle-trees.md), example 17).
Change one transaction of three thousand and the root moves, so the substitution is detectable
without ever holding the body.

This split is not a storage optimisation. It is what makes headers-first sync
([13](13-p2p-networking.md)) and SPV wallets ([64](64-light-clients-spv.md)) possible at all.

### 2. A minimal header

Seven fields, all fixed-width, all comparable:

```go
type Header struct {
    Version    uint32   // so the layout can change later (topic 3)
    PrevHash   [32]byte // the previous header's hash — the "chain"
    MerkleRoot [32]byte // commits to the body (lesson 05)
    Timestamp  int64    // Unix seconds, never a time.Time (topic 7)
    Bits       uint32   // the difficulty target (lesson 09)
    Nonce      uint32   // what a miner grinds (lesson 09)
    Height     uint64   // convenient, not authoritative
}
```

There are no strings, no slices and no maps, which is deliberate three times over: the struct is
comparable with `==`, the serialization has no length prefixes, and there is nothing whose encoding
could vary. Example [1](examples/08-blocks-and-chain/1-easy.md#1-the-header-field-by-field) prints
every field's offset — byte 4 is *always* the first byte of `PrevHash`.

`Bits` and `Nonce` are placeholders here and become real in [09](09-proof-of-work.md).

**Height deserves scepticism.** It is a number *inside* the header, so a block can claim any height
it likes — nothing outside the block vouches for it. Worse, two competing branches can share a
height, so height can never answer "which chain is canonical?". Example
[9](examples/08-blocks-and-chain/2-medium.md#9-height-is-convenient-not-authoritative) builds a block
claiming height 9,999,999 with a perfectly valid link, then two different blocks at the same height
from the same parent. Under variable difficulty a chain of 100 easy blocks can be *longer* than 50
hard ones and far cheaper to produce, which is why fork choice uses accumulated work
([14](14-consensus-forks.md)).

So height is a convenience — a cheap parent check, a database index, a place to schedule rule changes
— always validated against the parent, never trusted.

**Do not cache the hash.** It is the obvious optimisation and it is a trap. Example
[11](examples/08-blocks-and-chain/2-medium.md#11-the-cached-hash-bug) caches a hash, mutates `Nonce`,
and gets the stale value back — and `sync.Once` makes it worse by guaranteeing the staleness is
permanent. What makes this bug nasty is that the block validates against *itself* consistently, so it
only surfaces when a peer recomputes from the bytes and reports "your node is sending invalid
blocks". Either don't cache (hashing 92 bytes takes about a microsecond), or seal the block on
construction and expose no mutators.

### 3. Deterministic serialization

The bytes you hash must be identical on every machine, in every Go version, forever. Write them by
hand:

```go
func (h Header) Bytes() []byte {
    buf := bytes.NewBuffer(make([]byte, 0, HeaderSize))
    binary.Write(buf, binary.BigEndian, h.Version)
    buf.Write(h.PrevHash[:])
    buf.Write(h.MerkleRoot[:])
    binary.Write(buf, binary.BigEndian, h.Timestamp)
    binary.Write(buf, binary.BigEndian, h.Bits)
    binary.Write(buf, binary.BigEndian, h.Nonce)
    binary.Write(buf, binary.BigEndian, h.Height)
    return buf.Bytes()
}
```

This is [04](04-hash-functions.md)'s canonical-preimage rule applied to blocks, and example
[7](examples/08-blocks-and-chain/2-medium.md#7-why-not-json-or-gob) shows concretely why the
convenient options are disqualified:

- **`gob` encodes field names.** Rename `Timestamp` to `CreatedAt` — a pure refactor — and the bytes
  change. With gob, a rename is a chain split.
- **JSON emits struct fields in declaration order.** Reorder two fields for readability and every
  hash changes. JSON also leaves whitespace, escaping and number formatting free.

Neither ties the format to anything stable. An explicit encoder ties it to a documented byte layout,
and renaming a Go field changes nothing on the wire.

**Write the round-trip test first.** Example
[6](examples/08-blocks-and-chain/2-medium.md#6-encode-decode-encode) asserts encode → decode → encode
is byte-identical, which is the test that catches a field-order mistake before anything depends on
it. Note the decoder validates its length *before* reading — topic 6 and example 16 return to why.

**Versioning** is what lets the layout change later without invalidating history. The rules, from
example [17](examples/08-blocks-and-chain/3-hard.md#17-changing-the-format-without-splitting-the-chain):
version first in the byte layout so it can always be read; encode by the header's *own* version, never
the newest; old blocks keep their exact bytes and hashes forever; activate at a height everyone agrees
on in advance. Get it wrong and old blocks rehash differently, which fails the whole chain — a
self-inflicted fork.

### 4. Hash linking

Each header stores its parent's hash. That is the entire "chain":

```go
child.PrevHash = parent.Hash()
```

Not a slice, not a pointer — a link made of content. Example
[4](examples/08-blocks-and-chain/1-easy.md#4-chaining-three-blocks) builds four blocks and walks
backwards from the tip by following `PrevHash` until it reaches the block that has none.

The property this buys is **tamper-evidence**, and example
[8](examples/08-blocks-and-chain/2-medium.md#8-tampering-breaks-every-later-block) traces it
carefully. Edit a transaction in block 1 and the Merkle check fails. Let the attacker repair the
Merkle root so block 1 is internally consistent — and the break simply *moves*, because block 2 still
stores block 1's old hash:

```
--- tampering with block 1's body ---
  block 1: body does not match merkle root
--- attacker recomputes block 1's merkle root ---
  block 2: prev hash does not match block 1
```

To hide the edit, the attacker must recompute block 1's hash, then block 2's `PrevHash`, then block
2's hash, and so on to the tip.

Be precise about what this does and does not give you. Hash linking makes tampering **detectable**,
and nothing more. It says nothing about *which* chain is correct — two valid chains can exist side by
side. Proof of work ([09](09-proof-of-work.md)) is what makes redoing that work expensive, and fork
choice ([14](14-consensus-forks.md)) is what picks between branches.

### 5. The genesis block

The first block has no parent, so nothing can validate it:

```go
func Genesis() Block {
    var zero [32]byte           // no parent
    body := []string{"The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"}
    return Block{Header: Header{Version: 1, PrevHash: zero, MerkleRoot: MerkleRoot(body),
        Timestamp: 1231006505, Bits: 0x1f00ffff, Height: 0}, Body: body}
}
```

It is hardcoded rather than mined, and every node must agree on it **byte for byte**. Two nodes with
different genesis blocks are simply on different networks and will never agree on anything. Example
[3](examples/08-blocks-and-chain/1-easy.md#3-the-genesis-block) pins its hash in a constant, which is
exactly what a test should do: if someone changes a field, the test fails loudly instead of the
network splitting quietly.

Bitcoin's genesis coinbase carries a newspaper headline, which proves the chain was not started
earlier. Ethereum's genesis instead carries an allocation of pre-funded accounts — a JSON file every
node loads identically ([63](63-private-networks.md)).

### 6. Validation rules

Split the checks in two, because a syncing node receives blocks out of order constantly.

**Stateless** checks need only the block, so they can run the instant it arrives — before you know
whether you even have its parent:

```go
func CheckStateless(b Block) error {
    if b.Header.Version != HeaderVersion { return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version) }
    if len(b.Body) == 0                  { return ErrEmptyBody }
    if len(b.Body) > MaxBodyLen          { return fmt.Errorf("%w: %d", ErrTooLarge, len(b.Body)) }
    if MerkleRoot(b.Body) != b.Header.MerkleRoot { return ErrBadMerkle }
    return nil
}
```

**Stateful** checks need the parent: does it exist, is the height one greater, is the timestamp
sane, is the difficulty right ([09](09-proof-of-work.md)), are there double spends
([10](10-transactions-utxo.md)).

**Return typed errors, not strings.** The network layer branches on the *kind* of failure — an
orphan gets queued and its parent requested, a bad version gets the peer dropped
([13](13-p2p-networking.md)). That means the kind has to be a value you can match with `errors.Is`.
Examples [10](examples/08-blocks-and-chain/2-medium.md#10-stateless-and-stateful-validation) and
[15](examples/08-blocks-and-chain/3-hard.md#15-a-chain-that-validates-before-it-mutates) build both
passes and exercise every rejection path.

One rule matters more than the others:

> **Validate completely, then mutate.** A validator that updates state as it goes leaves a
> half-applied block behind when a later check fails — and that is how a database gets corrupted
> ([12](12-persistence-chainstate.md)).

### 7. Timestamps

A block timestamp is not a clock reading. Miners choose it, and they have incentives to lie.

First, the Go problem. A `time.Time` from `time.Now()` carries a hidden **monotonic reading**
alongside the wall clock, so two values representing the same instant are not `==` — example
[12](examples/08-blocks-and-chain/2-medium.md#12-timestamps-are-int64-not-timetime) shows
`now == now.Round(0)` returning false while `now.Equal(...)` returns true. That makes `time.Time`
unusable in anything you hash. **Store `int64` Unix seconds** and convert at the edges only.

Second, the protocol problem. The timestamp is bounded from both sides:

| Bound | Rule | Prevents |
|---|---|---|
| Lower | must exceed **median-time-past** — the median of the last 11 timestamps | backdating, which feeds the **timewarp attack** on difficulty ([09](09-proof-of-work.md)) |
| Upper | at most ~2 hours ahead of the node's own clock | forward-dating to push difficulty down |

The median matters: it means a single miner with a wrong clock cannot move the lower bound. Example
[13](examples/08-blocks-and-chain/2-medium.md#13-median-time-past) builds an 11-block history where
one miner's clock is a day ahead and shows the median ignoring the outlier entirely.

So a block timestamp is a loosely bounded value the protocol needs for difficulty and timelocks —
never evidence of when a block was made, and never a clock a contract should trust
([27](27-contract-security.md), [53](53-oracles-randomness.md)).

### 8. Testing the chain

Two things make the next seven lessons tractable, and both are worth building now.

**A chain builder.** The point of a fixture is that valid chains are the **default** and invalidity is
requested explicitly, so the interesting line of a test is the only line:

```go
c := newChain().AddN(5)                                    // a valid five-block chain
bad := newChain().AddN(5).Corrupt(2, func(b *Block) {      // one specific defect
    b.Header.Height = 99
})
branchA := base.Fork(3).Add("A-1").Add("A-2")              // the reorg setup lesson 14 needs
```

Example [14](examples/08-blocks-and-chain/3-hard.md#14-a-chainbuilder-for-the-next-seven-lessons)
builds it. Every test in lessons 09–15 starts "given a valid chain, when X is wrong…".

**A fuzz target on the decoder**, because a decoder runs on bytes a hostile peer chose. Example
[16](examples/08-blocks-and-chain/3-hard.md#16-fuzzing-the-decoder) contains the classic bug — a
length prefix read from the wire and handed straight to `make`:

```go
var count uint32
binary.Read(r, binary.BigEndian, &count)
body := make([]string, 0, count)   // a peer sends 0xffffffff and you allocate 4 billion entries
```

The fix bounds every count against the bytes actually remaining, not against a constant alone. The
example then runs 20,000 deterministic mutations and reports **zero panics**, which is the bar: a
decoder must never panic on hostile input. Note that *accepting* some mutations is fine — a flipped
nonce is still a well-formed block; it fails validation, not parsing.

In a real test file that is:

```go
func FuzzDecode(f *testing.F) {
    f.Add(validEncoding)
    f.Fuzz(func(t *testing.T, data []byte) { DecodeSafe(data) })
}
```

Also worth pinning with golden tests: the genesis hash, and one known block hash. Both catch an
accidental format change immediately. [34](34-testing-blockchain-go.md) makes all of this part of a
real suite.

## Exercises

Write these in `practice/08-blocks-and-chain/`. From here on the exercises accumulate — you are
building one program, so keep it in a package and grow it.

1. **The header.** Define `Header`, write `Bytes()` and `ParseHeader`, and write the round-trip test
   before anything else. Then deliberately swap two fields in `Bytes` only, and confirm your test
   catches it.
2. **Genesis, pinned.** Write `Genesis()` and a test asserting its hash equals a hardcoded constant.
   Change one field and watch the test fail — that failure is the feature.
3. **Build and validate.** Write `Chain` with `Append`, implementing both validation passes with
   typed errors. Table-test every rejection path with `errors.Is`.
4. **Prove the tamper-evidence.** Build a 10-block chain, edit block 3's body, and write a test
   asserting validation fails. Then repair block 3's Merkle root and assert it *still* fails, at
   block 4. Explain in a comment why.
5. **The fixture.** Write `chainBuilder` with `Add`, `AddN`, `Corrupt` and `Fork`. Rewrite exercise 3
   using it and note how much shorter the tests get.
6. **Median-time-past.** Implement it over an 11-block window and test: exactly at MTP (reject), one
   second after (accept), and a block from a miner whose clock is two days ahead (accept, but confirm
   it does not move the median).
7. **Fuzz the decoder.** Write a real `FuzzDecode` target and run `go test -fuzz=FuzzDecode` for a
   minute. Fix whatever it finds. Then deliberately remove your length bound and confirm the fuzzer
   finds it again.
8. **Version 2.** Add a field that exists only from version 2, activating at height 100. Write tests
   asserting: v1 blocks hash identically to before, a v1 block at height 100 is rejected, and a v2
   block at height 99 is rejected.

## Best Practices & Pitfalls

- **Never hash a struct through `gob`, JSON or any reflection-based encoder.**
  *Why:* gob encodes field names, so a rename changes every hash; JSON follows declaration order, so
  reordering fields does. Both tie your consensus rules to your Go source.
- **Do not cache a block's hash unless the block is immutable.**
  *Why:* mutate a field afterwards and the block reports a hash that is not its hash — consistently,
  so it validates against itself and only fails when a peer recomputes it.
- **Store `int64` Unix seconds, never a `time.Time`.**
  *Why:* `time.Now()` carries a monotonic reading, so two values for the same instant are not `==`,
  and whether it lands in your bytes depends on how the value was constructed.
- **Validate against the parent's height; never trust the header's.**
  *Why:* height is a claim inside the block. Nothing outside vouches for it, and two branches can
  share one.
- **Use accumulated work, not height, for fork choice.**
  *Why:* under variable difficulty a longer chain can be a cheaper one. Building on height now means
  rewriting it in [14](14-consensus-forks.md).
- **Validate completely before mutating any state.**
  *Why:* a validator that updates as it goes leaves half-applied blocks when a later check fails —
  the fastest route to a corrupt database ([12](12-persistence-chainstate.md)).
- **Return typed sentinel errors from validation.**
  *Why:* the network layer must branch on the kind of failure — queue an orphan, drop a bad version.
  A string cannot carry that.
- **Bound every length read from the wire against the bytes actually remaining.**
  *Why:* a four-byte count can ask for a four-billion-entry allocation. This is the single most
  common remote crash in a P2P decoder.
- **Put the version first in the byte layout, and encode by the block's own version.**
  *Why:* a node must be able to hash every block ever made. Re-encoding old blocks under a new layout
  changes their hashes and fails the entire chain.
- **Pin the genesis hash in a test.**
  *Why:* an accidental field change otherwise produces a silent network split rather than a failing
  build.

## Checklist

- [ ] I can define a header with only fixed-width, comparable fields and explain why.
- [ ] I can write a deterministic `Bytes()` and its exact inverse, and test the round trip.
- [ ] I can explain why `gob` and JSON are disqualified for anything hashed.
- [ ] I can build a genesis block and say why it must be pinned in a test.
- [ ] I can chain blocks by hash and walk backwards from the tip.
- [ ] I can explain what tampering breaks, and why repairing one block just moves the failure.
- [ ] I know why height is a convenience and accumulated work is the authority.
- [ ] I can split validation into stateless and stateful passes with typed errors.
- [ ] I can explain the cached-hash bug and two ways to avoid it.
- [ ] I know why timestamps are `int64` and how median-time-past bounds them.
- [ ] I have a `chainBuilder` fixture I can carry into lesson 09.
- [ ] I can write a decoder that never panics on hostile input, and fuzz it.

## Resources

**Specifications**

- Bitcoin developer reference — block chain: https://developer.bitcoin.org/reference/block_chain.html
- Bitcoin — block headers: https://developer.bitcoin.org/reference/block_chain.html#block-headers
- Ethereum Yellow Paper (block structure): https://ethereum.github.io/yellowpaper/paper.pdf
- BIP-113 — median-time-past for locktime: https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki

**Go**

- `encoding/binary`: https://pkg.go.dev/encoding/binary
- `time` — monotonic clocks: https://pkg.go.dev/time#hdr-Monotonic_Clocks
- Go fuzzing: https://go.dev/doc/security/fuzz/
- `errors` — wrapping and `Is`: https://go.dev/blog/go1.13-errors

**Reference implementations**

- btcd `wire.BlockHeader`: https://github.com/btcsuite/btcd/blob/master/wire/blockheader.go
- go-ethereum `core/types.Header`: https://github.com/ethereum/go-ethereum/blob/master/core/types/block.go

---

**Examples:** [`examples/08-blocks-and-chain/`](examples/08-blocks-and-chain/) — **18 runnable Go
programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard), standard library only. Example 18 is the assembled
starting point for the rest of Part 3; example 14 is the test fixture every later lesson uses.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
