# 12 — Persistence & Chain State

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#12-persistence-chain-state)

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [10](10-transactions-utxo.md) |
| **Examples** | [18](examples/12-persistence-chainstate/) (🟢 5 · 🟡 8 · 🔴 5) |

*storing blocks and the UTXO set on disk, key layout, iterators, atomic batches and crash safety*

Lessons 08–11 built a chain that lives in a slice and a map and dies with the process. This lesson
puts it on disk. That sounds like plumbing, and most of it is — but it is plumbing with a property
the in-memory version never had to think about: the process can stop at *any* instruction, and
whatever was on disk at that instant is what the next process starts from.

That is the whole lesson, asked nine different ways. A node killed between deleting a block's inputs
and inserting its outputs does not crash when it restarts. It carries on with a state that matches no
block, rejects valid blocks forever, and reports itself healthy. Lesson [08](08-blocks-and-chain.md)
promised that "validate completely, then mutate" was how you avoid a corrupt database; this is where
that promise gets kept, and where it turns out a transaction lets you relax it.

[Example 18](examples/12-persistence-chainstate/3-hard.md#18-the-chain-on-disk) moves lesson 11's
chain, wallet and mempool onto a bbolt file, restarts the node in the middle of the story, and checks
the stored state against a replay of the stored blocks.

## Goals

- Persist blocks and chain state to an embedded key-value store from Go.
- Design key prefixes and iterate the chain efficiently.
- Write atomically so a crash never leaves a half-applied block.
- Reindex state from raw blocks.

## Concepts

### 1. Why a key-value store

List everything a node asks of its storage and the list is short:

| access pattern | shape |
|---|---|
| store a block, once, never change it | put by key |
| fetch a block by hash | point lookup |
| walk blocks by height, or from height X to Y | ordered range |
| is this outpoint unspent? | point lookup |
| every output of this transaction, every coin of this address | prefix range |

Every row is a lookup by exact key or a walk over a range of sorted keys. That is precisely what an
ordered key-value store does, and it is *all* one does.

A relational database's value is in the queries you did not anticipate: a planner, joins, secondary
indexes built on demand. A node has no unanticipated queries. Every access path is written into the
code, reviewed, and on the consensus path — and paying for SQL parsing, planning and per-row
bookkeeping on each of them buys nothing while costing write throughput during the one workload a
node cannot avoid: replaying hundreds of millions of blocks.

The storage engine is not a detail, either. Bitcoin Core used Berkeley DB for everything until 0.8
moved the block index and UTXO set to LevelDB. On 11 March 2013, block 225430 needed more BDB locks
than 0.7's default configuration allowed: every 0.8 node accepted it, every 0.7 node rejected it,
and the chain split for 24 blocks until miners downgraded ([BIP-50](https://github.com/bitcoin/bips/blob/master/bip-0050.mediawiki)).
Nothing in the block broke a rule. A limit inside the database had quietly become part of consensus.

The Go options, and the trade each one makes:

| | structure | writes | reads | used by |
|---|---|---|---|---|
| **bbolt** | B+tree, one file, memory-mapped, copy-on-write | one writer at a time, fsync per commit | lock-free snapshot readers, zero-copy | etcd |
| **Badger** | LSM tree + value log | fast and concurrent; optimistic, can fail with `ErrConflict` | good, more moving parts | Dgraph |
| **Pebble** | LSM tree (RocksDB's design, in Go) | fast, batched, background compaction | good | CockroachDB, go-ethereum |

This course uses **bbolt**: pure Go with no cgo, a single file, and real transactions that roll back.
Its costs — one writer, borrowed memory, a file that never shrinks — are all visible enough to learn
from, and every one of them turns up in the examples.

```go
db, err := bolt.Open(path, 0o600, &bolt.Options{Timeout: time.Second})
err = db.Update(func(tx *bolt.Tx) error {
    b, err := tx.CreateBucketIfNotExists([]byte("blocks"))
    if err != nil {
        return err
    }
    return b.Put(hash[:], block.Encode())
})
```

[Example 1](examples/12-persistence-chainstate/1-easy.md#1-a-block-on-disk) stores two blocks, closes
the file, reopens it and checks the decoded header still hashes to its key. It also hits the two bbolt
behaviours that bite first. A missing key returns `nil`, not an error — convert that into a typed
`ErrNotFound` once, at the boundary. And bbolt holds an exclusive lock on the file: a second `Open`
without `Options.Timeout` waits *forever*, which is what "my node hangs at startup" usually turns out
to be — the previous process has not finished exiting.

### 2. Key-space design

In a key-value store the keys *are* the schema. There is nothing else to design.

bbolt has **buckets**, separate B+trees inside one file. LevelDB and Pebble have one flat key space,
so the same structure is built from **prefixes**: go-ethereum stores a header under
`"h" + number + hash` and the canonical hash for a number under `"h" + number + "n"`; Bitcoin Core
stores a coin under `'C' + txid + vout` and its best block under `'B'`. The layout this lesson ends
up with, in bbolt terms:

| bucket | key | value |
|---|---|---|
| `blocks` | hash [32] | version byte, header, transactions |
| `heights` | height [8, big-endian] | hash — canonical chain only |
| `utxo` | txid [32] \| vout [4] | version, value, height, coinbase flag, pubkey hash |
| `byaddr` | pkh [20] \| txid \| vout | empty — the key is the data |
| `undo` | hash [32] | the entries this block spent (lesson 14) |
| `meta` | `"tip"`, `"utxo-tip"`, `"schema"` | a hash, a hash, a version |

**Numbers must be big-endian and fixed-width.** Ordered stores compare keys as byte strings,
lexicographically, and nothing else. [Example 2](examples/12-persistence-chainstate/1-easy.md#2-heights-as-keys)
stores ten heights three ways. As decimal strings a cursor walks them as `[0 1 10 11 2 255 256 257 8 9]`,
`Last()` says the highest height is 9, and a scan for 250..257 returns `[255 256 257 8 9]`. As
little-endian integers everything is correct up to 255 and wrong from block 256 onwards — a bug with a
fuse. Big-endian is the only encoding whose byte order is numeric order, and fixed width is part of the
reason: a varint sorts no better than decimal.

```go
func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }
```

A varint is fine where nothing iterates in numeric order — Bitcoin Core's coin key ends with the output
index as a varint, because nothing ever walks one transaction's outputs in order.

**The tip is a pointer, not a query.** It is tempting to derive it — the highest height stored, or
`Last()` on the height index. [Example 3](examples/12-persistence-chainstate/1-easy.md#3-the-height-index-and-the-tip-pointer)
stores a stale sibling of block 3 and a block 5 that has not been validated yet; "the highest block
stored" is then height 5, and it is not the tip. What a node has *stored* and the chain it is *on* are
different things from here to lesson 14. The tip pointer is a **commit marker**: the one key that says
which block every other piece of state corresponds to, updated in the same transaction as that state.

**Keep keys short.** Every byte of a key is paid once per entry, in every index that contains it.
The UTXO set's 36-byte key costs 3.6 GB across a hundred million outputs;
[example 8](examples/12-persistence-chainstate/2-medium.md#8-prefix-scans-over-the-utxo-set) adds an
address index with 56-byte keys and another 5.6 GB. That is why Bitcoin Core has no address index at
all — consensus never asks who owns what — and why explorers and Electrum servers build one outside
the node. Note the trick in that index's key, though: the 36 bytes after the address *are* the UTXO
key, so the index needs no value at all.

**Write the schema down first**, as a comment block above the code that uses it. In six months it is
the only documentation the bytes will have, and the next person will have only the bytes.

### 3. Serialization on disk

**One format for hashing and for storage.** Lesson 08's encoder already produces fixed-order,
fixed-width, length-prefixed bytes, because hashing needed them. Store those same bytes behind a
version byte and there is no second encoder to drift out of step with the first. There is also a free
integrity check: a block is stored under its own hash, so every read can recompute it.
[Example 6](examples/12-persistence-chainstate/2-medium.md#6-a-chain-on-disk-walked-from-the-tip)
flips one bit in a stored block, and the store hands the bytes back without complaint — bbolt checksums
nothing, and neither do most file systems. The recomputed Merkle root is the only thing that notices.

**Never `gob`.** It round-trips any Go value in one line, which is exactly why it gets used.
[Example 5](examples/12-persistence-chainstate/1-easy.md#5-why-not-gob) shows five ways it fails on disk:

- **Field names in every record.** A UTXO entry is 80 bytes of gob against 32 — 4.8 GB of type
  descriptions at a hundred million entries — because each value must decode on its own, so each one
  carries its type.
- **Renames lose data silently.** gob matches fields by name. Rename `Value` to `Amount` and old
  records decode with `err == nil` and `Amount == 0`. Fifty coins, gone, no error.
- **Zero values are never written.** Decode a normal output and then a zero-value output (Bitcoin's
  `OP_RETURN` outputs are exactly that) into the same reused struct, and the second reads as 50 coins.
- **Bytes depend on history.** One value encoded twice on one encoder is 80 bytes, then 33, and the
  second record cannot be decoded alone: `gob: unknown type id or corrupted data`.
- **Maps encode in iteration order**, so the same value produces different bytes on different runs.

Add interface values needing `gob.Register`, and a format with one implementation in one language, and
there is nothing left to recommend it. JSON shares the name problem and adds number precision. Neither
belongs anywhere near consensus data.

**Measure before compressing.** [Example 11](examples/12-persistence-chainstate/2-medium.md#11-version-bytes-a-migration-and-a-measurement)
deflates realistic data: a block shrinks to 87%, a hundred blocks as one stream to 84%, a header grows
to 107% and a UTXO entry to 118%. Blocks are mostly txids, signatures and public keys, which are as
close to random as bytes get by design, and the small records a key-value store actually holds get
*bigger*. Fifteen percent of disk, paid for with CPU on every read of every block. Bitcoin Core runs
LevelDB with compression switched off.

**Version everything.** A version byte on every record lets a decoder tell formats apart instead of
misreading one as another; a schema version on the database tells the program a migration is pending.

```go
switch v[0] {
case EntryV2:
    return decodeV2(v)
case EntryV1:
    return Entry{}, ErrNeedsMigration
default:
    return Entry{}, fmt.Errorf("%w: 0x%02x", ErrUnknownRecord, v[0])
}
```

Example 11 migrates a v1 UTXO set — no height, no coinbase flag — to v2 inside one write transaction,
schema byte included, and kills it after 17 of 59 entries: afterwards every entry is still v1 and the
schema still says 1. Run again, it finishes. Then the v1 binary opens the migrated file and **refuses**.
That refusal is the feature: a downgraded binary that "mostly works" reads 38-byte records as 29-byte
ones and writes old records into a new set. go-ethereum keeps a `DatabaseVersion` key for exactly this.

### 4. Atomic writes

Connecting a block touches four things: the block bytes, the height index, the UTXO set's deletes and
inserts, and the tip pointer. All of them change, or none of them do — which in bbolt is one sentence:
**one block, one `db.Update`**.

```go
func ApplyBlock(db *bolt.DB, b *Block) error {
    return db.Update(func(tx *bolt.Tx) error {
        if err := checkTip(tx, b); err != nil {
            return err
        }
        if err := applyUTXO(tx, b); err != nil { // deletes and inserts as it validates
            return err
        }
        if err := putBlock(tx, b); err != nil {
            return err
        }
        return moveTip(tx, b)
    })
}
```

The closure is the atomic unit. Return `nil` and bbolt commits everything; return an error and it rolls
everything back; panic, and a deferred rollback runs on the way up. Lessons 08 and 10 insisted on
building a `Delta` and applying it only after the whole block validated, because a map has no rollback.
A write transaction does: **the transaction is the delta**, so validation and mutation can share one
pass. Reads inside it see its own writes, which is what lets a transaction spend an output created
earlier in the same block.

[Example 9](examples/12-persistence-chainstate/2-medium.md#9-one-block-one-write-transaction) proves
each piece. A block whose second transaction spends a coin that never existed: the first transaction
had already deleted an input and inserted an output, `db.Update` returned the error, the closure ran
once, and the state fingerprint is unchanged. The same block with a panic halfway: unchanged. One valid
block applied with the tip written first to one copy and last to another: identical fingerprints.
**Order inside the transaction does not matter**, because nobody can observe its middle.

**`db.Update` never retries, and there is nothing to retry.** bbolt runs one writer at a time; a second
`Update` simply waits and then runs against the first one's result. So the tip check belongs *inside*
the closure — and example 9 shows what happens when it is not. Two competing blocks race for height 4:
checked inside, one connects and one is refused; checked in a `View` beforehand, *both* connect, and the
UTXO set holds the rewards of two blocks that can never be in one chain. No conflict error, no retry,
no crash. Badger is the opposite design, optimistic transactions that fail with `ErrConflict` and must
be retried; know which kind of store you have, and never paper over either silently.

**The invariant to test** is that the tip and the UTXO set describe the same block. Write a marker for
the set — `utxo-tip` — in the *same* transaction as every UTXO change, and compare it with the tip at
startup. [Example 10](examples/12-persistence-chainstate/2-medium.md#10-three-write-transactions-instead-of-one)
splits the work across three transactions, kills the writer after each step, and restarts:

| writer | fatal kill point | what the restarted node does |
|---|---|---|
| block, utxo, tip | after utxo | re-applies block 5: *input is not in the UTXO set* |
| block, tip, utxo | after tip | connects block 6: *input is not in the UTXO set* |
| one `db.Update` | none | always finishes block 5 and connects block 6 |

Both split writers wedge — rejecting valid blocks forever, without crashing — and at *different* points,
so there is no safe order to find. The startup check turns the silent wedge into a loud refusal. Bitcoin
Core keeps exactly this marker: the best-block key beside its coins, which it compares with the block
index at startup and replays blocks to reconcile.

### 5. Iterators and cursors

**Two ways to walk the chain.** Backwards: read the tip pointer, fetch the block, follow `PrevHash`,
repeat until the hash is zeros. It needs no index at all, which is why it still works along a branch no
index describes (lesson 14). Forwards: a cursor over `heights`. [Example 6](examples/12-persistence-chainstate/2-medium.md#6-a-chain-on-disk-walked-from-the-tip)
does both over five mined blocks, checking hash, Merkle root and proof of work on every read.

**Cursors** have five moves — `First`, `Last`, `Next`, `Prev` and `Seek` — and `Seek` is the one to
understand: it lands on the first key *greater than or equal to* the one you gave. It never says "not
found"; it returns whatever comes next. So every prefix scan is written the same way:

```go
c := tx.Bucket(bktAddr).Cursor()
for k, _ := c.Seek(pkh); k != nil && bytes.HasPrefix(k, pkh); k, _ = c.Next() {
    // one output owned by pkh
}
```

Drop the `HasPrefix` and the loop walks into the next address, and the one after. Example 8 computes
alice's balance three ways: a full scan reads 3,472 keys, the prefix scan reads 138, both say 160.66 —
and the scan without the prefix check says 7,037.48 without an error, because it is right only for
whichever address happens to sort last. One more cursor rule, from bbolt's own documentation: never
`Put` or `Delete` in the bucket a cursor is walking. Walk one bucket, write another.

**Values live only as long as the transaction.** bbolt does not copy data out of the file. It
memory-maps the file and `Get` returns a slice pointing straight into the map — not a copy of the record,
the record. The doc comment says it plainly: *"only valid for the life of the transaction"* and *"must
never be modified"*. [Example 4](examples/12-persistence-chainstate/1-easy.md#4-copying-a-value-out-of-a-transaction)
makes that visible, and shows the trap is not just `return v`:

```go
e.Value      = int64(binary.BigEndian.Uint64(v[1:9])) // copied by the assignment
copy(h.PrevHash[:], v[4:36])                          // copied: arrays are values
e.PubKeyHash = v[22:]                                 // NOT copied: still points into the map
e.PubKeyHash = bytes.Clone(v[22:])                    // the fix
```

The rule runs the other way too. `Put` keeps a reference to your value until commit, so reusing one
buffer for three `Put`s in a transaction stores the last value three times.

**This is the number-one bbolt bug**, and [example 7](examples/12-persistence-chainstate/2-medium.md#7-the-slice-that-outlived-its-transaction)
shows why it survives testing. A wallet looks up alice's 50-coin output and keeps the result. The node
connects ordinary blocks that never touch that coin. bbolt is copy-on-write: each transaction writes the
new version of a page somewhere else and frees the old one, and a later transaction reuses it. Two
blocks later, the entry the wallet kept says **50 coins, owned by mallory** — the value was copied when
it was decoded, the owner was not. The database is fine; a fresh read says alice. Nothing flags it:
`go vet` sees a slice, `-race` sees one goroutine, and a unit test that reads and then checks has no
write in between. It can be worse. When the file grows, bbolt remaps it, and a kept slice may then point at
memory that is not mapped at all: example 7 sizes its map up front so that cannot happen mid-demo, and
without that one option some of its runs die with SIGSEGV instead of printing anything. And writing into `Get`'s slice
faults immediately, because the mapping is read-only.

Copy at the storage boundary, every time, and prefer fixed-size arrays in decoded structs so that
aliasing is impossible by construction. Nothing above the storage layer should ever hold bbolt's memory.

### 6. Reindexing

The blocks are the source of truth; the UTXO set is *derived* from them. So whenever the derived state
is in doubt — a disk error, a bug fixed in the connect path, a migration nobody trusts, a startup check
that failed — the answer is the same: set it aside and **replay every block from genesis**.

[Example 15](examples/12-persistence-chainstate/3-hard.md#15-reindexing-rebuild-then-diff) damages a
stored set three realistic ways: a live coin vanishes (a bug in a disconnect path), one bit flips in a
value (worth 10,995 coins), and a spent coin comes back (an undo record applied twice). The first
already hurts: a valid block spending that coin is rejected, while every check that looks only at the
tip says the node is healthy.

A rebuild worth running on a real chain has four properties:

- **Separate.** Replay into a new bucket, never over the live set, so an interrupted rebuild destroys
  nothing.
- **Resumable.** Replay a batch of blocks per write transaction, and save the position *in the same
  transaction* as the work. Example 15 is stopped at block 150 and resumes from exactly there.
- **Observable.** Report progress after each commit — never from inside the closure (§8). A reindex of
  Bitcoin takes hours; one that prints nothing gets killed by the operator.
- **Honest.** Replay with the *same* state-transition function that connects new blocks. A rebuild that
  uses its own copy of the logic is checking nothing.

```go
for k, va := ca.First(); ... // two cursors, one per set, in lockstep
    switch cmpKeys(ka, kb) {
    case -1: diffs = append(diffs, "only in stored")
    case +1: diffs = append(diffs, "only in rebuilt")
    default: if !bytes.Equal(va, vb) { diffs = append(diffs, "value differs") }
    }
```

Comparing the two sets is a **merge join**: both buckets are sorted by the same key, so two cursors
walking in lockstep find every difference in one pass and no memory — all three here, in 3,272 steps,
and it works the same on a set that does not fit in RAM. Then one write transaction swaps the rebuilt
set in.

Bitcoin Core ships both halves as flags: `-reindex-chainstate` rebuilds the UTXO set from the blocks on
disk, and `-reindex` rebuilds the block index as well. But the most valuable use is not recovery. It is
a **test oracle**: after any sequence of connects, disconnects, crashes and restarts, the stored UTXO set
must equal a replay of the stored blocks. Example 18 ends by checking exactly that, and it is the
property test to run under every change in lessons 13 and 14.

### 7. Pruning and snapshots

To validate the next block a node needs the UTXO set, and to know which chain it is on it needs the
headers. It needs no old block body for either. **Spent outputs' history, old bodies, and indexes like
"which block holds this transaction"** exist to answer questions about the past — usually on someone
else's behalf.

[Example 13](examples/12-persistence-chainstate/2-medium.md#13-pruning-what-a-node-can-still-answer)
asks an archive node six questions, prunes it to its last 20 bodies and drops its transaction index,
and asks again. Keys and values fall from 761,835 bytes to 240,495:

| | archive | pruned |
|---|---|---|
| validate new blocks | yes | yes |
| current balances | yes | yes |
| verify the header chain | yes | yes |
| serve old blocks to a syncing peer | yes | the last 20 only |
| find a transaction by id | yes | no |
| a balance in the past | yes | no |
| reorg deeper than the window | yes | no — needs the old bodies (lesson 14) |

Two details matter. A pruned body must return **`ErrPruned`, not `ErrNotFound`** — a peer asking for
block 10 should hear "I don't keep that", not "that doesn't exist", and a missing body *above* the
prune point is corruption, not policy. And the pruning itself is chunked and resumable, with a
`pruned-below` marker moving in the same transaction as each chunk's deletes.

Bitcoin Core's `-prune=550` keeps at least 550 MiB of blocks and never fewer than the last 288, and is
incompatible with `-txindex`; pruned nodes advertise `NODE_NETWORK_LIMITED` (BIP-159) so peers know not
to ask them for old blocks. Ethereum draws the same line around *state*: a full node keeps recent state,
and asking for an account's balance at an old block needs an archive node.

**The file does not shrink.** Example 13's database is 2,097,152 bytes before pruning and 2,097,152
after, the freed pages still inside it. bbolt puts freed pages on its freelist for later writes and never
returns them to the operating system; getting the disk back means `bolt.Compact` into a new file —
524,288 bytes here — offline, with room for both copies. LSM stores reclaim space in background
compaction instead, which is one reason large nodes use them.

**Snapshots** go the other way: start a node from state instead of history.
[Example 17](examples/12-persistence-chainstate/3-hard.md#17-a-hot-backup-and-a-snapshot-to-start-from)
writes the UTXO set at height 50 — 201 entries, 12,953 bytes, against 118,784 for the whole database at
that height — and a brand-new node loads it on top of the header chain, connects blocks 51–80, and ends
with the full node's exact UTXO set after replaying 30 blocks instead of 81.

What makes that safe is not the file. A snapshot carries a digest of its entries, and example 17 tampers
with one satoshi twice: the careless edit fails the embedded digest, and the careful one — digest
recomputed — fails because it does not match **the digest compiled into the node**. That is the design
of Bitcoin Core's assumeutxo: the hash lives in the chain parameters, reviewed like any consensus
constant, and the node validates the historical chain in the background to confirm it. Bitcoin needs the
hardcoded hash because its headers commit to transactions but not to the UTXO set. Ethereum's headers
commit to the state root (lessons [15](15-account-model-state.md) and [17](17-rlp-merkle-patricia-trie.md)),
which is what lets snap sync verify downloaded state against a header instead.

### 8. Crash safety

**"It wrote" is not "it is durable."** `write` returns once the bytes are in the operating system's page
cache; they reach the disk later. `fsync` asks for them to be on stable storage before it returns — and
even that has a history. On macOS a plain `fsync` does not flush the drive's own cache; you need
`F_FULLFSYNC`, which Go's `File.Sync` uses on darwin. In 2018 PostgreSQL discovered that after an I/O
error Linux could report a failed `fsync`, mark the dirty pages clean anyway, and report the *next*
`fsync` as a success — data lost with a clean return code. PostgreSQL's answer was to crash on any
`fsync` failure rather than retry.

bbolt's commit is built around that ordering. Dirty pages are written to *new* locations and synced;
only then is a **meta page** written pointing at the new tree, and synced. There are two meta pages,
used alternately and each checksummed, and on open the newest valid one wins. Die before the meta page
lands and the previous tree is still whole on disk.

**Test it with a real kill.** Returning an error from a closure (example 10) proves the logic but never
reaches the commit path. [Example 14](examples/12-persistence-chainstate/3-hard.md#14-killing-the-writer-mid-batch)
re-executes itself as a child writer and sends it SIGKILL — no deferred calls, no rollback — then opens
what is left and compares it with a replay of the chain:

- killed **inside** the transaction for block 10: tip 9, UTXO set exactly block 9's
- killed **between** a split writer's transactions: tip 9, `utxo-tip` 10 — inconsistent, and correctly so
- killed at **twenty arbitrary moments**: twenty consistent databases
- the same twenty with **`NoSync: true`**: twenty consistent databases

That last line is the lesson. `NoSync` skips `fsync`, so a power cut can lose committed transactions or
leave a meta page pointing at pages that never reached the disk — and a killed process cannot show it,
because its writes are already in the page cache and the kernel flushes them regardless. **kill -9
tests atomicity; it cannot test durability.** That needs a lost page cache: a VM powered off, Linux's
`dm-flakey`, or LazyFS. Never ship chain state with `NoSync`.

**Make block application idempotent**, because after a crash some block *will* be applied twice. A node
replays from its last checkpoint; two peers deliver the same block; a reorg reconnects one.
[Example 12](examples/12-persistence-chainstate/2-medium.md#12-applying-the-same-block-twice) keeps a
per-address running total and replays three blocks. The naive version returns no error at all, and
afterwards alice's total says 30 coins while her coins add up to 15. Puts are idempotent; deleting a
missing key is silently fine; a `+=` is neither. The fix is a guard *inside* the transaction — "is this
exact block already connected at this height?" — that turns a replay into a no-op.

```go
if bytes.Equal(tx.Bucket(bktHeights).Get(be64(b.Header.Height)), hash[:]) {
    return nil // already connected: a replay is success, not an error
}
```

The same example finds the one bbolt API that runs your code twice on purpose. `db.Batch` coalesces
concurrent writers into one transaction, and if any function in the batch fails, the batch rolls back
and the others run again. Two competing blocks through `Batch`: four closure runs, one block connected,
**two** "announce to peers" messages. Its doc comment says the function "may be called multiple times,
regardless of whether it returns error or not". So nothing inside a transaction closure may have an
effect outside the database — not a message, not a metric, not a cache write. Do it after the call
returns `nil`.

**Back up with `Tx.WriteTo`, never `cp`.** A read transaction pins the pages of its version, so a copy
taken inside one is consistent even while the writer carries on: example 17's backup began at height 50,
blocks 51–60 were committed during the copy, and the backup opens at exactly 50 and matches a replay. A
plain file copy has no such promise — it can take a meta page from one moment and the pages it points to
from another. Two operational details come with it. Set `InitialMmapSize` large enough that the writer
never needs to grow the memory map, because a remap waits for every open reader and a backup *is* a
long reader. And `fsync` the backup file; a backup nobody synced is a hope.

### 9. The storage interface

Lesson 13 feeds the chain blocks from strangers; lesson 14 reorganises it; lesson 34 tests it thousands
of times with forks, crashes and garbage. None of those tests are about bbolt. So the chain logic talks
to a **narrow interface**, and what sits behind it is swappable:

```go
// Declared beside the chain logic that uses it — not in the storage package.
type Store interface {
    PutBlock(b *Block) error                          // store without connecting
    GetBlock(hash [32]byte) (*Block, error)           // ErrNotFound; the result is the caller's
    Tip() (Header, error)                             // ErrEmpty before genesis
    ApplyBlock(b *Block) error                        // all of it, or none of it
    IterUTXO(fn func(Outpoint, TxOutput) error) error // key order; outputs are the caller's
}
```

**The consumer owns the interface.** It is declared where it is used, holds only what that code calls,
and the bbolt package never mentions it — idiomatic Go, and the reason a test double implements five
methods instead of fifty. When lesson 14 needs undo data, the *chain* adds a method, and both
implementations follow.

**The contract is more than the method set.** "The result is the caller's", "all of it or none of it"
and "in key order" are not in any type signature, and they are exactly where two implementations drift
apart. The memory store has to validate *before* applying, because a map has no rollback, and has to
clone on the way in and out; the bbolt store may mutate as it validates, and has to copy out of the map.
Different code, same promises — and the only way to keep the promises identical is **one test suite,
run against every implementation**.

[Example 16](examples/12-persistence-chainstate/3-hard.md#16-one-store-two-implementations-one-suite)
runs ten checks against five stores: memory, bbolt, and three broken ones carrying this lesson's bugs.
Each broken store fails exactly one check. `mem-shared` returns the stored block pointer, so editing a
returned block edits the store. `mem-eager` mutates while validating, so a rejected block's first
transaction stays applied. `bolt-alias` hands out slices into the map, and its outputs change owner once
later blocks reuse the pages. The standard library uses the same pattern: `testing/fstest.TestFS` checks
any `fs.FS` implementation against the contract, and you write one line per implementation to use it.

```go
func TestMemStore(t *testing.T)  { storetest.Run(t, func(t *testing.T) Store { return NewMemStore() }) }
func TestBoltStore(t *testing.T) { storetest.Run(t, openTempBolt) }
```

## Exercises

Continue the program from [11](11-wallets-mempool.md) in `practice/12-persistence-chainstate/`.

1. **A block by hash.** Store lesson 10's blocks in bbolt under their hashes with `Options.Timeout`
   set, reopen the file, and assert every decoded header hashes to its key. Add a test that an unknown
   hash returns `ErrNotFound` and never a zero block.
2. **Keys that sort.** Table-test heights `{0, 9, 10, 255, 256, 65536}` under decimal, little-endian
   and big-endian keys, and assert that only big-endian gives numeric cursor order, a correct `Last()`
   and a correct range scan.
3. **Stored is not connected.** Build `blocks`, `heights` and a tip pointer. Store a stale sibling and an
   unconnected block, and assert `Tip()` and `AtHeight()` ignore both.
4. **Borrowed memory.** Write a test that keeps a value from a `View`, runs ten write transactions over
   the same bucket, and fails if the kept value changed. Make it fail with an aliasing decoder and pass
   with a copying one — with the page size pinned, so it fails the same way everywhere. Then write the
   `Put`-side twin with a reused buffer.
5. **One transaction per block.** Implement `ApplyBlock` in one `db.Update`, validating and mutating in
   one pass. Test an in-block spend, a block that fails on its *last* transaction, and a panic inside the
   closure; the last two must leave the state fingerprint unchanged.
6. **The marker.** Add `utxo-tip`, written with every UTXO change, and a startup check. Write a split
   writer, kill it after every step, and assert the check flags exactly the inconsistent cases.
7. **Idempotent replay.** Make `ApplyBlock` a no-op for an already-connected block, replay the last ten
   blocks, and assert the fingerprint does not move. Then move every side effect out of your transaction
   closures and prove with `db.Batch` and competing blocks that each one happens exactly once.
8. **Reindex and diff.** Implement a resumable `Reindex` into a separate bucket and a merge-join `Diff`.
   Damage the stored set three different ways, and assert `Diff` reports exactly those three and nothing
   else — then swap, and assert a second reindex diffs clean.
9. **kill -9.** Re-execute your binary as a child writer, SIGKILL it at arbitrary moments fifty times,
   and assert every reopened database equals a replay of its own stored blocks. Then write down, in a
   comment, what this test cannot detect.
10. **Prune.** Delete bodies below `tip-N` in resumable chunks with a `pruned-below` marker. Assert that
    `BlockAt` returns `ErrPruned` below the marker, `ErrNotFound` for a missing body above it, and that
    validation of a new block is unaffected.
11. **The interface.** Declare `Store` in your chain package, implement it twice, and write
    `storetest.Run(t, newStore)` with at least example 16's ten checks. Add a deliberately broken third
    implementation for each check and make sure the suite catches it.

## Best Practices & Pitfalls

- **Never return, cache or keep a slice that came from inside a bbolt transaction.**
  *Why:* it points into the memory map; later writes reuse the page and it silently changes meaning, or
  a remap unmaps it entirely (example 7).
- **Copy at the storage boundary, and decode into fixed-size arrays where you can.**
  *Why:* `bytes.Clone` once in the store means no caller ever holds bbolt's memory, and an array cannot
  alias anything.
- **Give `Put` a fresh slice for every value.**
  *Why:* bbolt keeps a reference to the value until commit, so a reused buffer stores its last contents
  under every key.
- **Encode numeric keys as fixed-width big-endian — never decimal strings.**
  *Why:* keys sort as bytes, so `"10"` sorts before `"9"`, `Last()` lies, and range scans skip and
  include the wrong blocks; little-endian is correct until 256.
- **Apply every part of a block — UTXO deltas, indexes, undo data, tip — in its one write transaction.**
  *Why:* split across transactions, a crash between them leaves a state that matches no block, and the
  node rejects valid blocks forever without crashing (example 10).
- **Write a state marker in the same transaction as the state, and check it at startup.**
  *Why:* it turns a silent inconsistency into a refusal to start.
- **Do not assume `db.Update` retries on conflict, and do not add silent retries of your own.**
  *Why:* bbolt has one writer and nothing to conflict with; a check made before the transaction is
  simply stale, and both racing blocks connect (example 9).
- **Keep every side effect out of transaction closures.**
  *Why:* a closure can run and still be rolled back, and `db.Batch` runs closures more than once.
- **Make block application idempotent with a guard inside the transaction.**
  *Why:* replays after a crash are guaranteed, and the naive version corrupts running totals without an
  error (example 12).
- **Recompute the hash — and the Merkle root — when you read a block back.**
  *Why:* the store returns rotted bytes without complaint; only the hash can tell.
- **Use lesson 08's encoder for storage; never `gob` or JSON.**
  *Why:* renamed fields decode to zero with no error, zero values vanish, and the bytes depend on
  history and map order (example 5).
- **Put a version byte on every record and a schema version on the database; refuse a newer one.**
  *Why:* migrations need to tell formats apart, and a downgraded binary will otherwise corrupt the file.
- **Measure before compressing.**
  *Why:* hashes, keys and signatures are incompressible, and small records grow.
- **Always set `Options.Timeout`.**
  *Why:* without it a second `Open` waits forever on the file lock.
- **Back up with `Tx.WriteTo`, set `InitialMmapSize` for long readers, and fsync the copy.**
  *Why:* `cp` of a live file can be torn, and a writer that must remap waits for every open reader.
- **Test crash safety with a real SIGKILL, and durability with a lost page cache.**
  *Why:* returned errors never reach the commit path, and kill -9 cannot see a missing fsync
  (example 14).
- **Never run chain state with `NoSync: true`.**
  *Why:* a power cut can lose committed blocks or leave the meta page pointing at pages that never
  reached the disk.
- **Reindex into a separate bucket with the same state-transition code, then swap atomically.**
  *Why:* an interrupted rebuild must destroy nothing, and a rebuild with its own logic verifies nothing.
- **Do not trust `Bucket.Stats` inside a write transaction.**
  *Why:* it walks committed pages, so it does not see the transaction's own writes.
- **Never `Put` or `Delete` in the bucket a cursor is walking.**
  *Why:* the cursor can skip or repeat keys after the tree changes under it.
- **Return `ErrPruned` for pruned data, distinct from `ErrNotFound`.**
  *Why:* "I don't keep it" and "it doesn't exist" lead a peer, or an operator, to different actions.
- **Plan for a bbolt file that never shrinks.**
  *Why:* freed pages are only reused, and reclaiming disk means an offline `Compact` into a new file.
- **Declare `Store` where it is used, and run one suite against every implementation.**
  *Why:* the promises that matter are not in the method signatures, and each implementation breaks a
  different one (example 16).

## Checklist

- [ ] I can explain why a node's storage is a key-value store, and name what bbolt, Badger and Pebble trade.
- [ ] I can store and fetch blocks in bbolt, and I know what `Get` returns for a missing key and what `Open` does without a timeout.
- [ ] I can design a key space with buckets or prefixes, and explain why heights are fixed-width big-endian.
- [ ] I can explain why the tip is a stored pointer and not the highest block stored.
- [ ] I can choose key layouts by what they cost per entry, and build a composite-key index.
- [ ] I can explain five ways `gob` fails as a storage format, and version records and schemas instead.
- [ ] I can connect a block in one write transaction, and explain why that lets validation and mutation share a pass.
- [ ] I can explain why `db.Update` never retries, and where the tip check must go.
- [ ] I can keep a state marker and detect, at startup, a writer that died between transactions.
- [ ] I can walk the chain backwards by `PrevHash` and forwards with a cursor, and write a correct prefix scan.
- [ ] I can explain why a slice from a bbolt transaction changes meaning later, and why no tool catches it.
- [ ] I can rebuild the UTXO set by replaying blocks, resumably, and diff it against the stored set with two cursors.
- [ ] I can prune old bodies, say exactly which questions a pruned node can no longer answer, and reclaim the disk.
- [ ] I can explain how a UTXO snapshot is verified, and why Bitcoin needs a hash in the source code to do it.
- [ ] I can explain fsync, bbolt's commit order, and why kill -9 tests atomicity but not durability.
- [ ] I can make block application idempotent and keep side effects out of transaction closures.
- [ ] I can take a consistent hot backup of a live bbolt database.
- [ ] I can declare a consumer-owned `Store` interface and run one conformance suite against every implementation.

## Resources

**Specifications & reference designs**

- bbolt: https://pkg.go.dev/go.etcd.io/bbolt
- go-ethereum core/rawdb schema: https://github.com/ethereum/go-ethereum/tree/master/core/rawdb
- Bitcoin Core chainstate notes (data files): https://github.com/bitcoin/bitcoin/blob/master/doc/files.md
- Bitcoin Core assumeutxo design: https://github.com/bitcoin/bitcoin/blob/master/doc/design/assumeutxo.md
- BIP-50 (the March 2013 chain fork): https://github.com/bitcoin/bips/blob/master/bip-0050.mediawiki
- BIP-159 (`NODE_NETWORK_LIMITED` for pruned nodes): https://github.com/bitcoin/bips/blob/master/bip-0159.mediawiki

**Storage engines & crash consistency**

- bbolt source and caveats: https://github.com/etcd-io/bbolt
- Pebble: https://github.com/cockroachdb/pebble
- Badger: https://github.com/dgraph-io/badger
- *All File Systems Are Not Created Equal* (Pillai et al., OSDI 2014): https://www.usenix.org/conference/osdi14/technical-sessions/presentation/pillai
- PostgreSQL's fsync errors ("fsyncgate"): https://wiki.postgresql.org/wiki/Fsync_Errors
- LazyFS, for testing lost writes: https://github.com/dsrhaslab/lazyfs

**Go**

- `os.File.Sync`: https://pkg.go.dev/os#File.Sync
- `encoding/binary`: https://pkg.go.dev/encoding/binary
- `testing/fstest.TestFS`, a shared conformance suite in the standard library: https://pkg.go.dev/testing/fstest#TestFS

---

**Examples:** [`examples/12-persistence-chainstate/`](examples/12-persistence-chainstate/) — **18 runnable
Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Example 7 keeps a slice past its transaction and
watches alice's coin change owner two blocks later; example 10 kills a split writer at every step and
finds a different fatal point for each ordering; example 14 SIGKILLs a real child process and shows why
turning off fsync does not change the result; example 18 puts lesson 11's chain on disk and restarts it.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
