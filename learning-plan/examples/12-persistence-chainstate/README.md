# Step 12 — Persistence & Chain State · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                                # first time only
go get go.etcd.io/bbolt@v1.5.0                     # the store: every example except 5
go get github.com/ethereum/go-ethereum@latest      # secp256k1, example 18 only
go get golang.org/x/crypto@latest                  # RIPEMD-160, example 18 only
# paste the example into main.go, then:
go run .
```

No chain and no node. Every example writes its database into a fresh temporary directory and deletes
it on exit; every simulation is seeded and nothing is timed, so all output reproduces exactly. Where a
file size matters the page size is pinned to 4096 bytes, because bbolt otherwise uses the operating
system's (16384 on Apple Silicon). Example 5 needs no dependencies.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | a block on disk, keys that sort, the tip pointer, borrowed memory, why not gob |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | walking the chain, the stale slice, prefix scans, atomic blocks, split writers, versions, replays, pruning |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | a real kill -9, reindexing, one interface for every store, backups and snapshots, assembly |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. A block on disk](1-easy.md#1-a-block-on-disk)
- [2. Heights as keys](1-easy.md#2-heights-as-keys)
- [3. The height index and the tip pointer](1-easy.md#3-the-height-index-and-the-tip-pointer)
- [4. Copying a value out of a transaction](1-easy.md#4-copying-a-value-out-of-a-transaction)
- [5. Why not gob](1-easy.md#5-why-not-gob)

### 🟡 [Medium](2-medium.md)

- [6. A chain on disk, walked from the tip](2-medium.md#6-a-chain-on-disk-walked-from-the-tip)
- [7. The slice that outlived its transaction](2-medium.md#7-the-slice-that-outlived-its-transaction)
- [8. Prefix scans over the UTXO set](2-medium.md#8-prefix-scans-over-the-utxo-set)
- [9. One block, one write transaction](2-medium.md#9-one-block-one-write-transaction)
- [10. Three write transactions instead of one](2-medium.md#10-three-write-transactions-instead-of-one)
- [11. Version bytes, a migration, and a measurement](2-medium.md#11-version-bytes-a-migration-and-a-measurement)
- [12. Applying the same block twice](2-medium.md#12-applying-the-same-block-twice)
- [13. Pruning: what a node can still answer](2-medium.md#13-pruning-what-a-node-can-still-answer)

### 🔴 [Hard](3-hard.md)

- [14. Killing the writer mid-batch](3-hard.md#14-killing-the-writer-mid-batch)
- [15. Reindexing: rebuild, then diff](3-hard.md#15-reindexing-rebuild-then-diff)
- [16. One Store, two implementations, one suite](3-hard.md#16-one-store-two-implementations-one-suite)
- [17. A hot backup, and a snapshot to start from](3-hard.md#17-a-hot-backup-and-a-snapshot-to-start-from)
- [18. The chain on disk](3-hard.md#18-the-chain-on-disk)

## Three that change how you trust a database

**[7. The slice that outlived its transaction](2-medium.md#7-the-slice-that-outlived-its-transaction)**
keeps the result of a lookup for alice's 50 coins while the node connects ordinary blocks. Two blocks
later the kept entry says the coins belong to mallory. The database is fine; `go vet`, `-race` and a
unit test all pass.

**[10. Three write transactions instead of one](2-medium.md#10-three-write-transactions-instead-of-one)**
kills a writer after every step and restarts the node. Each ordering has a different fatal point, both
leave a node that rejects valid blocks forever without crashing, and the one-transaction writer has no
fatal point at all.

**[14. Killing the writer mid-batch](3-hard.md#14-killing-the-writer-mid-batch)** SIGKILLs a real child
process twenty times and finds twenty consistent databases — then does it again with fsync switched off,
and finds twenty more. kill -9 tests atomicity. It cannot see durability.

## The arc

1–3 — a block on disk, keys that sort, and the pointer that says which chain you are on.  
4–5 — memory bbolt lends you, and the encoder that must not be gob.  
6–8 — walking the chain, the slice that changed owner, and prefix scans.  
9–10 — one transaction per block, and what three transactions cost.  
11–12 — formats that change, and blocks that arrive twice.  
13 — what pruning keeps, and what it gives away.  
14–15 — a real kill -9, and rebuilding the state from the blocks.  
16–17 — one interface for every store; backups and snapshots.  
18 — lesson 11's chain, on disk.

---

*Lesson: [../../12-persistence-chainstate.md](../../12-persistence-chainstate.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
