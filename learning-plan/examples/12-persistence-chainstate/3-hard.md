# Step 12 — Persistence & Chain State · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

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

No chain and no node. Every example writes its database into a fresh temporary directory and
deletes it on exit; every simulation is seeded and nothing is timed, so all output reproduces
exactly. Example 14 starts copies of itself as child processes and kills them, so it runs on
macOS and Linux; `go run .` is fine, because the child is the same binary. Example 18's keys are
the published anvil development keys — test only.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [—](README.md)

---

## 14. Killing the writer mid-batch

`🔴 hard` · *Crash safety*

Example 10 "crashed" by returning an error, which never reaches bbolt's commit path. Here the writer is a separate process — this same program, started again as a child — and it dies by SIGKILL: no deferred calls, no rollback, no chance to tidy up. After every kill the parent opens whatever is left and compares it with a replay of the chain.

**Steps:**

1. Kill the child inside the write transaction for block 10, after it has deleted an input and inserted an output.
2. Kill a split writer between its UTXO transaction and its tip transaction.
3. Kill an ordinary writer twenty times at arbitrary moments: mid-update, mid-commit, mid-fsync.
4. Kill it twenty more times with `NoSync: true`.
5. Read why the fourth result is the most important one on the page.

```go
package main

import (
	"bufio"
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"sort"
	"strconv"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// kill -9, mid-batch.
//
// Example 10 "crashed" by returning early, which proves the logic but never
// touches the storage: a returned error does not reach bbolt's commit path.
// This example runs the writer as a separate PROCESS — this same program,
// started again as a child — and kills it with SIGKILL. No deferred calls, no
// rollback, no chance to tidy up. Then the parent opens whatever is left.
//
//   1. killed INSIDE a write transaction, at a chosen point
//   2. killed BETWEEN two write transactions, at a chosen point
//   3. killed at arbitrary moments, twenty times
//   4. the same twenty, with fsync switched off — and why the test can't tell
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   blocks    hash           version | header | txs
//   heights   height BE      hash
//   utxo      txid | vout    value [8] | pkh [20]
//   meta      "tip"          hash of the chain's tip
//             "utxo-tip"     hash of the block the UTXO set reflects
// ---------------------------------------------------------------------------

const (
	HeaderSize = 92
	Coin       = int64(100_000_000)
	ChainLen   = 400 // more than any trial reaches
	Trials     = 20
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
	keyUTXOTip = []byte("utxo-tip")
)

// ---------------------------------------- the chain (both processes build it)

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

func (h Header) Bytes() []byte {
	b := make([]byte, 0, HeaderSize)
	b = binary.BigEndian.AppendUint32(b, h.Version)
	b = append(b, h.PrevHash[:]...)
	b = append(b, h.MerkleRoot[:]...)
	b = binary.BigEndian.AppendUint64(b, uint64(h.Timestamp))
	b = binary.BigEndian.AppendUint32(b, h.Bits)
	b = binary.BigEndian.AppendUint32(b, h.Nonce)
	b = binary.BigEndian.AppendUint64(b, h.Height)
	return b
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxOutput struct {
	Value int64
	PKH   [20]byte
}

type Transaction struct {
	Data    []byte
	Inputs  []Outpoint
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	b := append(binary.BigEndian.AppendUint32(nil, uint32(len(t.Data))), t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = binary.BigEndian.AppendUint32(append(b, in.TxID[:]...), in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = append(binary.BigEndian.AppendUint64(b, uint64(o.Value)), o.PKH[:]...)
	}
	return b
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

type Block struct {
	Header Header
	Txs    []*Transaction
}

func (b *Block) Encode() []byte {
	out := append([]byte{0x01}, b.Header.Bytes()...)
	for _, t := range b.Txs {
		raw := t.Serialize()
		out = append(binary.BigEndian.AppendUint32(out, uint32(len(raw))), raw...)
	}
	return out
}

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

// buildChain is deterministic, so the parent can rebuild exactly the blocks
// the child was writing and check the database against them.
func buildChain(n int) []*Block {
	miner, alice := pkh("miner"), pkh("alice")
	var chain []*Block
	for h := 0; h < n; h++ {
		txs := []*Transaction{{Data: be64(uint64(h)), Outputs: []TxOutput{{50 * Coin, miner}}}}
		hdr := Header{Version: 1, Timestamp: 1700000000 + int64(h)*600, Bits: 0x2000ffff, Height: uint64(h)}
		if h > 0 {
			prev := chain[h-1]
			hdr.PrevHash = prev.Header.Hash()
			txs = append(txs, &Transaction{ // spend the previous coinbase
				Inputs:  []Outpoint{{prev.Txs[0].TxID(), 0}},
				Outputs: []TxOutput{{1 * Coin, alice}, {49 * Coin, miner}},
			})
		}
		hdr.MerkleRoot = sha256.Sum256(txs[len(txs)-1].Serialize())
		chain = append(chain, &Block{hdr, txs})
	}
	return chain
}

// ------------------------------------------------------------- writing

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func initSchema(db *bolt.DB) error {
	return db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		return nil
	})
}

func putBlock(tx *bolt.Tx, b *Block) error {
	hash := b.Header.Hash()
	if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
		return err
	}
	return tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:])
}

// applyUTXO calls midway (if set) after the inputs are gone and the first
// output is in: the most half-applied a block can be.
func applyUTXO(tx *bolt.Tx, b *Block, midway func()) error {
	utxo := tx.Bucket(bktUTXO)
	first := true
	for _, t := range b.Txs {
		for _, in := range t.Inputs {
			if err := utxo.Delete(utxoKey(in)); err != nil {
				return err
			}
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			v := append(be64(uint64(o.Value)), o.PKH[:]...)
			if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), v); err != nil {
				return err
			}
			if first && midway != nil && b.Header.Height > 0 {
				midway()
			}
			first = false
		}
	}
	hash := b.Header.Hash()
	return tx.Bucket(bktMeta).Put(keyUTXOTip, hash[:])
}

func moveTip(tx *bolt.Tx, b *Block) error {
	hash := b.Header.Hash()
	return tx.Bucket(bktMeta).Put(keyTip, hash[:])
}

// ----------------------------------------------------------------- the child

func child() {
	path, mode, pause := os.Getenv("L12_DB"), os.Getenv("L12_MODE"), os.Getenv("L12_PAUSE")
	pauseAt, _ := strconv.Atoi(os.Getenv("L12_PAUSE_AT"))
	db, err := bolt.Open(path, 0o600, &bolt.Options{Timeout: time.Second, NoSync: os.Getenv("L12_NOSYNC") == "1"})
	check(err)
	check(initSchema(db))

	// stop announces where it is, then blocks reading a pipe nobody writes to.
	// The only way out is the SIGKILL.
	stop := func(where string) {
		fmt.Println(where)
		_, _ = os.Stdin.Read(make([]byte, 1))
	}

	for h, b := range buildChain(ChainLen) {
		here := h == pauseAt
		switch mode {
		case "atomic":
			check(db.Update(func(tx *bolt.Tx) error {
				var midway func()
				if pause == "inside" && here {
					midway = func() { stop("paused") }
				}
				if err := putBlock(tx, b); err != nil {
					return err
				}
				if err := applyUTXO(tx, b, midway); err != nil {
					return err
				}
				return moveTip(tx, b)
			}))
		case "split":
			check(db.Update(func(tx *bolt.Tx) error { return putBlock(tx, b) }))
			check(db.Update(func(tx *bolt.Tx) error { return applyUTXO(tx, b, nil) }))
			if pause == "between" && here {
				stop("paused")
			}
			check(db.Update(func(tx *bolt.Tx) error { return moveTip(tx, b) }))
		}
		fmt.Printf("committed %d\n", h)
	}
}

// ---------------------------------------------------------------- the parent

// runChild starts the writer, reads its progress until it prints `until`,
// waits `delay`, and kills it.
func runChild(path, mode, pause string, pauseAt int, noSync bool, until string, delay time.Duration) {
	exe, err := os.Executable()
	check(err)
	cmd := exec.Command(exe)
	cmd.Env = append(os.Environ(), "L12_CHILD=1", "L12_DB="+path, "L12_MODE="+mode, "L12_PAUSE="+pause,
		"L12_PAUSE_AT="+strconv.Itoa(pauseAt), "L12_NOSYNC="+map[bool]string{true: "1", false: "0"}[noSync])
	stdin, err := cmd.StdinPipe() // held open and never written: the child's pause
	check(err)
	defer stdin.Close()
	out, err := cmd.StdoutPipe()
	check(err)
	cmd.Stderr = os.Stderr
	check(cmd.Start())

	sc := bufio.NewScanner(out)
	for sc.Scan() && sc.Text() != until {
	}
	time.Sleep(delay)
	check(cmd.Process.Kill()) // SIGKILL
	_ = cmd.Wait()
}

type Report struct {
	CheckErrors  int
	Tip, UTXOTip int // -1: none
	UTXOIsReplay bool
}

func (r Report) Consistent() bool { return r.CheckErrors == 0 && r.Tip == r.UTXOTip && r.UTXOIsReplay }

// inspect opens what the child left and compares it with the chain.
func inspect(path string, chain []*Block) (r Report) {
	db, err := bolt.Open(path, 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()
	check(db.View(func(tx *bolt.Tx) error {
		for range tx.Check() { // bbolt's own structural check: pages, freelist, tree
			r.CheckErrors++
		}
		height := func(hash []byte) int {
			if hash == nil {
				return -1
			}
			return int(binary.BigEndian.Uint64(tx.Bucket(bktBlocks).Get(hash)[1+84 : 1+92]))
		}
		meta := tx.Bucket(bktMeta)
		r.Tip, r.UTXOTip = height(meta.Get(keyTip)), height(meta.Get(keyUTXOTip))
		r.UTXOIsReplay = utxoFingerprint(tx) == replayFingerprint(chain, r.UTXOTip)
		return nil
	}))
	return r
}

func utxoFingerprint(tx *bolt.Tx) [32]byte {
	h := sha256.New()
	_ = tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
		h.Write(k)
		h.Write(v)
		return nil
	})
	return [32]byte(h.Sum(nil))
}

// replayFingerprint applies blocks 0..tip in memory: the state a correct
// database must hold if it says it reflects `tip`.
func replayFingerprint(chain []*Block, tip int) [32]byte {
	set := map[string][]byte{}
	for _, b := range chain[:tip+1] {
		for _, t := range b.Txs {
			for _, in := range t.Inputs {
				delete(set, string(utxoKey(in)))
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				set[string(utxoKey(Outpoint{id, uint32(vout)}))] = append(be64(uint64(o.Value)), o.PKH[:]...)
			}
		}
	}
	keys := make([]string, 0, len(set))
	for k := range set {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	h := sha256.New()
	for _, k := range keys {
		h.Write([]byte(k))
		h.Write(set[k])
	}
	return [32]byte(h.Sum(nil))
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	if os.Getenv("L12_CHILD") == "1" {
		child()
		return
	}
	dir, err := os.MkdirTemp("", "kill9-*")
	check(err)
	defer os.RemoveAll(dir)
	chain := buildChain(ChainLen)
	fresh := func(name string) string { return filepath.Join(dir, name+".db") }

	fmt.Println("=== 1. SIGKILL inside the write transaction for block 10 ===")
	p := fresh("inside")
	runChild(p, "atomic", "inside", 10, false, "paused", 0)
	r := inspect(p, chain)
	fmt.Println("  the child had deleted block 10's input and inserted its first output")
	fmt.Printf("  reopened: Check errors %d · tip %d · utxo-tip %d · UTXO set == replay to %d: %v\n",
		r.CheckErrors, r.Tip, r.UTXOTip, r.UTXOTip, r.UTXOIsReplay)
	db, err := bolt.Open(p, 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	check(db.View(func(tx *bolt.Tx) error {
		u := tx.Bucket(bktUTXO)
		fmt.Printf("  block 10's input is still unspent: %v · its first output does not exist: %v\n",
			u.Get(utxoKey(chain[10].Txs[1].Inputs[0])) != nil,
			u.Get(utxoKey(Outpoint{chain[10].Txs[0].TxID(), 0})) == nil)
		return nil
	}))
	check(db.Close())
	fmt.Println("  Uncommitted pages live in the writer's memory. The process died, and")
	fmt.Println("  the half-applied block died with it.")

	fmt.Println("\n=== 2. SIGKILL between the UTXO transaction and the tip transaction ===")
	p = fresh("between")
	runChild(p, "split", "between", 10, false, "paused", 0)
	r = inspect(p, chain)
	fmt.Printf("  reopened: Check errors %d · tip %d · utxo-tip %d · UTXO set == replay to %d: %v\n",
		r.CheckErrors, r.Tip, r.UTXOTip, r.UTXOTip, r.UTXOIsReplay)
	fmt.Printf("  consistent: %v\n", r.Consistent())
	fmt.Println("  bbolt kept every promise it made: two transactions, each whole. The")
	fmt.Println("  application's unit of work was split in three, and no storage engine")
	fmt.Println("  can make that atomic after the fact.")

	storm := func(noSync bool) {
		consistent, clean, agree, replay := 0, 0, 0, 0
		for trial := 0; trial < Trials; trial++ {
			p := fresh(fmt.Sprintf("storm-%v-%d", noSync, trial))
			// Kill somewhere after block 3+trial: mid-update, mid-commit, mid-fsync.
			runChild(p, "atomic", "none", -1, noSync, fmt.Sprintf("committed %d", 3+trial),
				time.Duration(trial*173)*time.Microsecond)
			r := inspect(p, chain)
			if r.CheckErrors == 0 {
				clean++
			}
			if r.Tip == r.UTXOTip {
				agree++
			}
			if r.UTXOIsReplay {
				replay++
			}
			if r.Consistent() {
				consistent++
			}
		}
		fmt.Printf("  trials %d · Check clean %d · tip == utxo-tip %d · UTXO == replay %d · consistent %d\n",
			Trials, clean, agree, replay, consistent)
	}

	fmt.Println("\n=== 3. twenty kills at arbitrary moments, one transaction per block ===")
	storm(false)
	fmt.Println("  (How far each child got varies from run to run, so it is not printed.)")
	fmt.Println("  bbolt commits in a fixed order: write the dirty pages somewhere new,")
	fmt.Println("  fsync, THEN write a meta page pointing at them, fsync. Two meta pages")
	fmt.Println("  alternate, each checksummed; on open, the newest valid one wins. Die")
	fmt.Println("  before the meta write and the old tree is still whole on disk.")

	fmt.Println("\n=== 4. the same twenty, with NoSync: true ===")
	storm(true)
	fmt.Println("  Just as clean — and that is the trap. NoSync skips fsync, so a")
	fmt.Println("  POWER CUT can lose committed transactions or leave a meta page")
	fmt.Println("  pointing at pages that never reached the disk. A killed process")
	fmt.Println("  cannot show that: its writes are already in the OS page cache, and")
	fmt.Println("  the kernel flushes them anyway. kill -9 tests atomicity. Durability")
	fmt.Println("  needs a lost page cache: a VM power-off, dm-flakey, or LazyFS.")
}
```

**Output:**

```
=== 1. SIGKILL inside the write transaction for block 10 ===
  the child had deleted block 10's input and inserted its first output
  reopened: Check errors 0 · tip 9 · utxo-tip 9 · UTXO set == replay to 9: true
  block 10's input is still unspent: true · its first output does not exist: true
  Uncommitted pages live in the writer's memory. The process died, and
  the half-applied block died with it.

=== 2. SIGKILL between the UTXO transaction and the tip transaction ===
  reopened: Check errors 0 · tip 9 · utxo-tip 10 · UTXO set == replay to 10: true
  consistent: false
  bbolt kept every promise it made: two transactions, each whole. The
  application's unit of work was split in three, and no storage engine
  can make that atomic after the fact.

=== 3. twenty kills at arbitrary moments, one transaction per block ===
  trials 20 · Check clean 20 · tip == utxo-tip 20 · UTXO == replay 20 · consistent 20
  (How far each child got varies from run to run, so it is not printed.)
  bbolt commits in a fixed order: write the dirty pages somewhere new,
  fsync, THEN write a meta page pointing at them, fsync. Two meta pages
  alternate, each checksummed; on open, the newest valid one wins. Die
  before the meta write and the old tree is still whole on disk.

=== 4. the same twenty, with NoSync: true ===
  trials 20 · Check clean 20 · tip == utxo-tip 20 · UTXO == replay 20 · consistent 20
  Just as clean — and that is the trap. NoSync skips fsync, so a
  POWER CUT can lose committed transactions or leave a meta page
  pointing at pages that never reached the disk. A killed process
  cannot show that: its writes are already in the OS page cache, and
  the kernel flushes them anyway. kill -9 tests atomicity. Durability
  needs a lost page cache: a VM power-off, dm-flakey, or LazyFS.
```

---

## 15. Reindexing: rebuild, then diff

`🔴 hard` · *Reindexing*

The blocks are the source of truth and the UTXO set is derived from them, so when the derived state is in doubt the answer is to set it aside and replay every block. The stored set is damaged three ways; a rebuild replays 300 bodies into a separate bucket in resumable batches; two cursors walk both sets in lockstep to find every difference; and one write transaction swaps the good set in.

**Steps:**

1. Connect 300 blocks, then damage the stored UTXO set three different ways.
2. Try to connect a valid block that spends the coin that vanished.
3. Reindex into a `rebuild` bucket 50 blocks at a time, stop at 150, and resume.
4. Diff the stored and rebuilt sets with a merge join over two cursors.
5. Swap the rebuilt set in, reindex and diff once more, and connect the block.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// Reindexing: rebuild the UTXO set from the blocks, then diff.
//
// The blocks are the source of truth; the UTXO set is derived from them.
// So whenever the derived state is in doubt — a disk error, a bug fixed in
// the connect path, a migration nobody trusts — the answer is the same:
// set it aside and replay every block from genesis.
//
// This example damages a stored UTXO set three ways, rebuilds it into a
// separate bucket (in batches, with progress, surviving an interruption),
// diffs the two with a pair of cursors, and swaps the good one in atomically.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key              value
//   -------   --------------   ------------------------------------------------
//   bodies    height BE        count [4] | transactions   the source of truth
//   utxo      txid | vout      value [8] | pkh [20]       the live set: DERIVED
//   rebuild   txid | vout      value [8] | pkh [20]       a set being rebuilt
//   meta      "tip"            height BE
//             "reindex-next"   height BE: the next block the rebuild replays
//
//   (Bodies are keyed by height to keep this example about replay. Example 6
//   keys them by hash, which is what a node that sees forks must do.)
// ---------------------------------------------------------------------------

const (
	Coin   = int64(100_000_000)
	Blocks = 300
	PerTx  = 50 // blocks replayed per write transaction
)

var (
	bktBodies  = []byte("bodies")
	bktUTXO    = []byte("utxo")
	bktRebuild = []byte("rebuild")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
	keyNext    = []byte("reindex-next")
)

var (
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrTruncated    = errors.New("record truncated")
	errStopped      = errors.New("stopped: the operator pressed Ctrl-C")
)

// ---------------------------------------------------------------- the model

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxOutput struct {
	Value int64
	PKH   [20]byte
}

type Transaction struct {
	Data    []byte
	Inputs  []Outpoint
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	b := append(binary.BigEndian.AppendUint32(nil, uint32(len(t.Data))), t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = binary.BigEndian.AppendUint32(append(b, in.TxID[:]...), in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = append(binary.BigEndian.AppendUint64(b, uint64(o.Value)), o.PKH[:]...)
	}
	return b
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

func encodeBody(txs []*Transaction) []byte {
	b := binary.BigEndian.AppendUint32(nil, uint32(len(txs)))
	for _, t := range txs {
		b = append(b, t.Serialize()...)
	}
	return b
}

type reader struct {
	p   []byte
	err error
}

func (r *reader) take(n int) []byte {
	if r.err != nil {
		return nil
	}
	if n < 0 || n > len(r.p) {
		r.err = ErrTruncated
		return nil
	}
	v := r.p[:n]
	r.p = r.p[n:]
	return v
}

func (r *reader) u32() uint32 {
	if v := r.take(4); v != nil {
		return binary.BigEndian.Uint32(v)
	}
	return 0
}

func decodeBody(p []byte) ([]*Transaction, error) {
	r := &reader{p: p}
	var txs []*Transaction
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		t := &Transaction{Data: bytes.Clone(r.take(int(r.u32())))}
		for m := r.u32(); m > 0 && r.err == nil; m-- {
			var op Outpoint
			copy(op.TxID[:], r.take(32))
			op.Index = r.u32()
			t.Inputs = append(t.Inputs, op)
		}
		for m := r.u32(); m > 0 && r.err == nil; m-- {
			var o TxOutput
			if v := r.take(8); v != nil {
				o.Value = int64(binary.BigEndian.Uint64(v))
			}
			copy(o.PKH[:], r.take(20))
			t.Outputs = append(t.Outputs, o)
		}
		txs = append(txs, t)
	}
	return txs, r.err
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func encodeOut(o TxOutput) []byte { return append(be64(uint64(o.Value)), o.PKH[:]...) }

// ---------------------------------------------------------------- the store

// applyTxs is the whole state transition, and it is SHARED: connecting a new
// block and replaying an old one must run the same code, or the rebuild is
// checking nothing.
func applyTxs(set *bolt.Bucket, txs []*Transaction) error {
	for i, t := range txs {
		for _, in := range t.Inputs {
			if set.Get(utxoKey(in)) == nil {
				return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
			}
			if err := set.Delete(utxoKey(in)); err != nil {
				return err
			}
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			if err := set.Put(utxoKey(Outpoint{id, uint32(vout)}), encodeOut(o)); err != nil {
				return err
			}
		}
	}
	return nil
}

func Connect(db *bolt.DB, height uint64, txs []*Transaction) error {
	return db.Update(func(tx *bolt.Tx) error {
		if err := applyTxs(tx.Bucket(bktUTXO), txs); err != nil {
			return err
		}
		if err := tx.Bucket(bktBodies).Put(be64(height), encodeBody(txs)); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, be64(height))
	})
}

// Reindex replays bodies into the `rebuild` bucket, PerTx blocks per write
// transaction. The position is saved in the SAME transaction as the work it
// describes, so stopping — at any moment — loses at most one batch, and the
// next call picks up exactly where the last commit left off. stopAt fakes an
// operator's Ctrl-C.
func Reindex(db *bolt.DB, stopAt uint64, progress func(done, total uint64, coins int)) error {
	for {
		var done, total uint64
		var coins int
		finished := false
		err := db.Update(func(tx *bolt.Tx) error {
			rebuild, err := tx.CreateBucketIfNotExists(bktRebuild)
			if err != nil {
				return err
			}
			meta := tx.Bucket(bktMeta)
			next := uint64(0)
			if v := meta.Get(keyNext); v != nil {
				next = binary.BigEndian.Uint64(v)
			}
			total = binary.BigEndian.Uint64(meta.Get(keyTip)) + 1
			if next >= total {
				finished = true
				return nil
			}
			if next >= stopAt {
				return errStopped
			}
			done = min(next+PerTx, total)
			for h := next; h < done; h++ {
				txs, err := decodeBody(tx.Bucket(bktBodies).Get(be64(h)))
				if err != nil {
					return fmt.Errorf("body %d: %w", h, err)
				}
				if err := applyTxs(rebuild, txs); err != nil {
					return fmt.Errorf("replaying block %d: %w", h, err) // the BLOCKS are bad: stop, loudly
				}
			}
			return meta.Put(keyNext, be64(done))
		})
		if err != nil || finished {
			return err
		}
		// Bucket.Stats walks COMMITTED pages: inside the write transaction it
		// would not have seen this batch yet. So count after the commit.
		if err := db.View(func(tx *bolt.Tx) error {
			coins = tx.Bucket(bktRebuild).Stats().KeyN
			return nil
		}); err != nil {
			return err
		}
		progress(done, total, coins) // after the commit, never inside the closure (example 12)
	}
}

// Diff walks both sets with one cursor each, in lockstep. Both are sorted by
// the same key, so every difference falls out of a single merge pass.
func Diff(tx *bolt.Tx, names map[[20]byte]string) (diffs []string, steps int) {
	ca, cb := tx.Bucket(bktUTXO).Cursor(), tx.Bucket(bktRebuild).Cursor()
	ka, va := ca.First()
	kb, vb := cb.First()
	for ka != nil || kb != nil {
		steps++
		switch c := cmpKeys(ka, kb); {
		case c < 0:
			diffs = append(diffs, fmt.Sprintf("only in stored    %s  %s", opString(ka), describe(va, names)))
			ka, va = ca.Next()
		case c > 0:
			diffs = append(diffs, fmt.Sprintf("only in rebuilt   %s  %s", opString(kb), describe(vb, names)))
			kb, vb = cb.Next()
		default:
			if !bytes.Equal(va, vb) {
				diffs = append(diffs, fmt.Sprintf("value differs     %s  stored %s, rebuilt %s",
					opString(ka), describe(va, names), describe(vb, names)))
			}
			ka, va = ca.Next()
			kb, vb = cb.Next()
		}
	}
	return diffs, steps
}

// cmpKeys orders nil — an exhausted cursor — after everything.
func cmpKeys(a, b []byte) int {
	switch {
	case a == nil:
		return 1
	case b == nil:
		return -1
	}
	return bytes.Compare(a, b)
}

// Swap replaces the live set with the rebuilt one in ONE write transaction:
// readers see the old set or the new one, never a mixture.
func Swap(db *bolt.DB) error {
	return db.Update(func(tx *bolt.Tx) error {
		if err := tx.DeleteBucket(bktUTXO); err != nil {
			return err
		}
		live, err := tx.CreateBucket(bktUTXO)
		if err != nil {
			return err
		}
		if err := tx.Bucket(bktRebuild).ForEach(live.Put); err != nil {
			return err
		}
		if err := tx.DeleteBucket(bktRebuild); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Delete(keyNext)
	})
}

// ------------------------------------------------------------------ helpers

type rng struct{ n uint64 }

func (r *rng) intn(n int) int {
	r.n++
	s := sha256.Sum256(be64(r.n))
	return int(binary.BigEndian.Uint64(s[:8]) % uint64(n))
}

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func btc(sat int64) string { return fmt.Sprintf("%d.%08d", sat/Coin, sat%Coin) }

func opString(k []byte) string {
	return fmt.Sprintf("%x…:%d", k[:4], binary.BigEndian.Uint32(k[32:]))
}

func describe(v []byte, names map[[20]byte]string) string {
	return btc(int64(binary.BigEndian.Uint64(v[:8]))) + " " + names[[20]byte(v[8:28])]
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "reindex-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()
	check(db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBodies, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucket(name); err != nil {
				return err
			}
		}
		return nil
	}))

	people := []string{"alice", "bob", "carol", "dave", "erin"}
	names := map[[20]byte]string{pkh("miner"): "miner"}
	for _, p := range people {
		names[pkh(p)] = p
	}

	// ------------------------------------------------------------ the chain
	type coin struct {
		op  Outpoint
		out TxOutput
	}
	r := &rng{}
	var pool, spent []coin
	for h := uint64(0); h < Blocks; h++ {
		txs := []*Transaction{{Data: be64(h), Outputs: []TxOutput{{50 * Coin, pkh("miner")}}}}
		for i := 0; i < 10 && len(pool) > 0; i++ {
			j := r.intn(len(pool))
			c := pool[j]
			pool[j], pool = pool[len(pool)-1], pool[:len(pool)-1]
			spent = append(spent, c)
			amount := c.out.Value / int64(2+r.intn(4))
			txs = append(txs, &Transaction{
				Inputs:  []Outpoint{c.op},
				Outputs: []TxOutput{{amount, pkh(people[r.intn(len(people))])}, {c.out.Value - amount, c.out.PKH}},
			})
		}
		check(Connect(db, h, txs))
		for _, t := range txs {
			id := t.TxID()
			for vout, o := range t.Outputs {
				pool = append(pool, coin{Outpoint{id, uint32(vout)}, o})
			}
		}
	}
	var stored int
	check(db.View(func(tx *bolt.Tx) error {
		stored = tx.Bucket(bktUTXO).Stats().KeyN
		return nil
	}))
	fmt.Printf("=== %d blocks connected; %d unspent outputs; %d spent along the way ===\n", Blocks, stored, len(spent))

	// ----------------------------------------------------------- the damage
	fmt.Println("\n=== three things go wrong in the stored UTXO set ===")
	victim, inflated, ghost := pool[len(pool)/3], pool[len(pool)/2], spent[len(spent)/4]
	check(db.Update(func(tx *bolt.Tx) error {
		u := tx.Bucket(bktUTXO)
		if err := u.Delete(utxoKey(victim.op)); err != nil {
			return err
		}
		wrong := inflated.out
		wrong.Value ^= 1 << 40 // one flipped bit, worth 10995.11627776 coins
		if err := u.Put(utxoKey(inflated.op), encodeOut(wrong)); err != nil {
			return err
		}
		return u.Put(utxoKey(ghost.op), encodeOut(ghost.out))
	}))
	fmt.Printf("  a live coin vanishes      %s  %s   (a bug in a disconnect path)\n",
		opString(utxoKey(victim.op)), describe(encodeOut(victim.out), names))
	fmt.Printf("  one bit flips in a value  %s  %s   (a bad sector)\n",
		opString(utxoKey(inflated.op)), describe(encodeOut(inflated.out), names))
	fmt.Printf("  a spent coin comes back   %s  %s   (an undo record applied twice)\n",
		opString(utxoKey(ghost.op)), describe(encodeOut(ghost.out), names))

	next := []*Transaction{
		{Data: be64(Blocks), Outputs: []TxOutput{{50 * Coin, pkh("miner")}}},
		{Inputs: []Outpoint{victim.op}, Outputs: []TxOutput{{victim.out.Value, pkh("erin")}}},
	}
	fmt.Printf("  block %d spends the vanished coin: %v\n", Blocks, Connect(db, Blocks, next))
	fmt.Println("  A perfectly valid block, rejected. Nothing crashed; the tip is fine;")
	fmt.Println("  every check that looks only at the tip says the node is healthy.")

	// ------------------------------------------------------------- reindex
	fmt.Printf("\n=== reindex: replay %d bodies into a new bucket, %d per transaction ===\n", Blocks, PerTx)
	progress := func(done, total uint64, coins int) {
		fmt.Printf("  %3d/%d blocks replayed   %5d coins in the rebuilt set\n", done, total, coins)
	}
	err = Reindex(db, 150, progress)
	fmt.Printf("  %v\n", err)
	check(db.View(func(tx *bolt.Tx) error {
		fmt.Printf("  restarted: resuming from height %d — the position was committed with the work\n",
			binary.BigEndian.Uint64(tx.Bucket(bktMeta).Get(keyNext)))
		return nil
	}))
	check(Reindex(db, ^uint64(0), progress))

	// ---------------------------------------------------------------- diff
	fmt.Println("\n=== diff: the stored set against the rebuilt one ===")
	check(db.View(func(tx *bolt.Tx) error {
		diffs, steps := Diff(tx, names)
		for _, d := range diffs {
			fmt.Printf("  %s\n", d)
		}
		fmt.Printf("  %d differences, found in %d cursor steps over two sorted buckets\n", len(diffs), steps)
		return nil
	}))
	fmt.Println("  All three, and nothing else. A merge join needs no memory beyond")
	fmt.Println("  two cursors, so it works the same on a set that does not fit in RAM.")

	// ---------------------------------------------------------------- swap
	fmt.Println("\n=== swap the rebuilt set in, in one write transaction ===")
	check(Swap(db))
	check(db.Update(func(tx *bolt.Tx) error { // an empty rebuild bucket, so Diff has two sides
		_, err := tx.CreateBucket(bktRebuild)
		return err
	}))
	check(Reindex(db, ^uint64(0), func(uint64, uint64, int) {}))
	check(db.View(func(tx *bolt.Tx) error {
		diffs, _ := Diff(tx, names)
		fmt.Printf("  reindexed again and diffed: %d differences\n", len(diffs))
		return nil
	}))
	check(Connect(db, Blocks, next))
	fmt.Printf("  block %d, which spends the coin that had vanished: connected\n", Blocks)
	fmt.Println()
	fmt.Println("  The same replay is the best test oracle you will write for lessons")
	fmt.Println("  10-14: after any sequence of connects, disconnects and crashes,")
	fmt.Println("  the stored set must equal a replay of the stored blocks. Bitcoin")
	fmt.Println("  Core ships it as -reindex-chainstate (and -reindex, which rebuilds")
	fmt.Println("  the block index too).")
}
```

**Output:**

```
=== 300 blocks connected; 3271 unspent outputs; 2971 spent along the way ===

=== three things go wrong in the stored UTXO set ===
  a live coin vanishes      54217199…:0  10.00000000 dave   (a bug in a disconnect path)
  one bit flips in a value  3f90dfab…:1  0.00312500 dave   (a bad sector)
  a spent coin comes back   c87c7c1f…:0  0.66666666 dave   (an undo record applied twice)
  block 300 spends the vanished coin: tx 1: input is not in the UTXO set
  A perfectly valid block, rejected. Nothing crashed; the tip is fine;
  every check that looks only at the tip says the node is healthy.

=== reindex: replay 300 bodies into a new bucket, 50 per transaction ===
   50/300 blocks replayed     521 coins in the rebuilt set
  100/300 blocks replayed    1071 coins in the rebuilt set
  150/300 blocks replayed    1621 coins in the rebuilt set
  stopped: the operator pressed Ctrl-C
  restarted: resuming from height 150 — the position was committed with the work
  200/300 blocks replayed    2171 coins in the rebuilt set
  250/300 blocks replayed    2721 coins in the rebuilt set
  300/300 blocks replayed    3271 coins in the rebuilt set

=== diff: the stored set against the rebuilt one ===
  value differs     3f90dfab…:1  stored 10995.11940276 dave, rebuilt 0.00312500 dave
  only in rebuilt   54217199…:0  10.00000000 dave
  only in stored    c87c7c1f…:0  0.66666666 dave
  3 differences, found in 3272 cursor steps over two sorted buckets
  All three, and nothing else. A merge join needs no memory beyond
  two cursors, so it works the same on a set that does not fit in RAM.

=== swap the rebuilt set in, in one write transaction ===
  reindexed again and diffed: 0 differences
  block 300, which spends the coin that had vanished: connected

  The same replay is the best test oracle you will write for lessons
  10-14: after any sequence of connects, disconnects and crashes,
  the stored set must equal a replay of the stored blocks. Bitcoin
  Core ships it as -reindex-chainstate (and -reindex, which rebuilds
  the block index too).
```

---

## 16. One Store, two implementations, one suite

`🔴 hard` · *The storage interface*

Lessons 13, 14 and 34 test chain logic thousands of times, and none of those tests are about bbolt. So the logic talks to a five-method `Store` owned by the code that uses it; a map and a bbolt file both implement it; and one suite runs against both — and against three broken implementations, each carrying a bug from earlier in this lesson, to prove the suite can tell the difference.

**Steps:**

1. Declare `Store` beside its consumer, with the contract written into the method comments.
2. Implement it with maps (validate, then apply; copy everything) and with bbolt (one transaction per block).
3. Write ten checks against the interface, over a fixture chain with deliberately bad blocks in it.
4. Run the suite against both real stores and three broken ones.
5. Read which check each broken store fails, and why the suite has to be shared.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"runtime/debug"
	"slices"
	"sort"
	"sync"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// One Store interface, two implementations, one test suite.
//
// Lessons 13, 14 and 34 test the chain logic hundreds of times — forks,
// reorgs, peers feeding garbage — and none of those tests are about bbolt.
// So the chain logic talks to a small interface, and what sits behind it is
// swappable: a map for tests, bbolt for real.
//
// Swappable only works if both behave the same, including in the ways nobody
// writes down. So ONE suite runs against both — and against three broken
// implementations, each with a bug from earlier in this lesson, to show that
// the suite can tell.
// ===========================================================================

const Coin = int64(100_000_000)

// ---------------------------------------------------------------- the model

type Header struct {
	PrevHash [32]byte
	Height   uint64
	TxRoot   [32]byte
}

func (h Header) Hash() [32]byte {
	b := binary.BigEndian.AppendUint64(append([]byte(nil), h.PrevHash[:]...), h.Height)
	f := sha256.Sum256(append(b, h.TxRoot[:]...))
	return sha256.Sum256(f[:])
}

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxOutput struct {
	Value      int64
	PubKeyHash []byte
}

type Transaction struct {
	Data    []byte
	Inputs  []Outpoint
	Outputs []TxOutput
}

type Block struct {
	Header Header
	Txs    []*Transaction
}

func (t *Transaction) Serialize() []byte {
	b := append(binary.BigEndian.AppendUint32(nil, uint32(len(t.Data))), t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = binary.BigEndian.AppendUint32(append(b, in.TxID[:]...), in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = append(binary.BigEndian.AppendUint32(b, uint32(len(o.PubKeyHash))), o.PubKeyHash...)
	}
	return b
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

// ------------------------------------------------------------- the interface

// Store is declared HERE, beside the chain logic that uses it, and holds only
// what that logic calls. The bbolt code below never mentions it. In Go the
// consumer owns the interface, so storage cannot grow methods nobody asked
// for, and a test double needs to implement five methods, not fifty.
type Store interface {
	// PutBlock stores a block without connecting it (headers-first sync).
	PutBlock(b *Block) error
	// GetBlock returns ErrNotFound for an unknown hash. The block returned is
	// the caller's: changing it must not change the store.
	GetBlock(hash [32]byte) (*Block, error)
	// Tip returns ErrEmpty before anything is applied.
	Tip() (Header, error)
	// ApplyBlock connects b on top of the tip: all of it, or none of it.
	ApplyBlock(b *Block) error
	// IterUTXO visits every unspent output in key order. The outputs are the
	// caller's to keep. fn must not write to the store.
	IterUTXO(fn func(Outpoint, TxOutput) error) error
	Close() error
}

var (
	ErrNotFound     = errors.New("not found")
	ErrEmpty        = errors.New("store is empty")
	ErrNotOnTip     = errors.New("parent is not the current tip")
	ErrMissingInput = errors.New("input is not in the UTXO set")
)

// ---------------------------------------------------- the memory Store

type MemStore struct {
	mu     sync.RWMutex
	blocks map[[32]byte]*Block
	utxo   map[Outpoint]TxOutput
	tip    *Header
}

func NewMemStore() *MemStore {
	return &MemStore{blocks: map[[32]byte]*Block{}, utxo: map[Outpoint]TxOutput{}}
}

func cloneOut(o TxOutput) TxOutput { return TxOutput{o.Value, bytes.Clone(o.PubKeyHash)} }

func cloneBlock(b *Block) *Block {
	c := &Block{Header: b.Header}
	for _, t := range b.Txs {
		ct := &Transaction{Data: bytes.Clone(t.Data), Inputs: slices.Clone(t.Inputs)}
		for _, o := range t.Outputs {
			ct.Outputs = append(ct.Outputs, cloneOut(o))
		}
		c.Txs = append(c.Txs, ct)
	}
	return c
}

func (m *MemStore) PutBlock(b *Block) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.blocks[b.Header.Hash()] = cloneBlock(b)
	return nil
}

func (m *MemStore) GetBlock(hash [32]byte) (*Block, error) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	b, ok := m.blocks[hash]
	if !ok {
		return nil, ErrNotFound
	}
	return cloneBlock(b), nil
}

func (m *MemStore) Tip() (Header, error) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	if m.tip == nil {
		return Header{}, ErrEmpty
	}
	return *m.tip, nil
}

func (m *MemStore) checkTip(b *Block) error {
	if (m.tip == nil && b.Header.Height != 0) || (m.tip != nil && b.Header.PrevHash != m.tip.Hash()) {
		return ErrNotOnTip
	}
	return nil
}

// ApplyBlock has no transaction to roll back, so it validates EVERYTHING into
// a delta first and touches the maps only after the whole block has passed.
func (m *MemStore) ApplyBlock(b *Block) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	if err := m.checkTip(b); err != nil {
		return err
	}
	var spend []Outpoint
	create := map[Outpoint]TxOutput{}
	for i, t := range b.Txs {
		for _, in := range t.Inputs {
			_, live := m.utxo[in]
			_, fresh := create[in]
			if (!live && !fresh) || slices.Contains(spend, in) {
				return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
			}
			spend = append(spend, in)
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			create[Outpoint{id, uint32(vout)}] = cloneOut(o)
		}
	}
	for op, o := range create { // insert, THEN delete (lesson 10)
		m.utxo[op] = o
	}
	for _, op := range spend {
		delete(m.utxo, op)
	}
	m.blocks[b.Header.Hash()] = cloneBlock(b)
	h := b.Header
	m.tip = &h
	return nil
}

func (m *MemStore) IterUTXO(fn func(Outpoint, TxOutput) error) error {
	m.mu.RLock()
	ops := make([]Outpoint, 0, len(m.utxo))
	outs := make(map[Outpoint]TxOutput, len(m.utxo))
	for op, o := range m.utxo {
		ops = append(ops, op)
		outs[op] = cloneOut(o)
	}
	m.mu.RUnlock()
	// Key order is part of the contract, and a map has none: sort by the same
	// bytes bbolt sorts by.
	sort.Slice(ops, func(i, j int) bool { return bytes.Compare(utxoKey(ops[i]), utxoKey(ops[j])) < 0 })
	for _, op := range ops {
		if err := fn(op, outs[op]); err != nil {
			return err
		}
	}
	return nil
}

func (m *MemStore) Close() error { return nil }

// ----------------------------------------------------- the bbolt Store

var (
	bktBlocks = []byte("blocks")
	bktUTXO   = []byte("utxo")
	bktMeta   = []byte("meta")
	keyTip    = []byte("tip")
)

type BoltStore struct{ db *bolt.DB }

func OpenBoltStore(path string) (*BoltStore, error) {
	db, err := bolt.Open(path, 0o600, &bolt.Options{
		Timeout:         time.Second,
		PageSize:        4096,
		InitialMmapSize: 1 << 24, // no remap while a reader is open (example 17)
	})
	if err != nil {
		return nil, err
	}
	err = db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		return nil
	})
	return &BoltStore{db}, err
}

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func opFromKey(k []byte) Outpoint {
	return Outpoint{[32]byte(k[:32]), binary.BigEndian.Uint32(k[32:])}
}

func encodeBlock(b *Block) []byte {
	out := binary.BigEndian.AppendUint64(append([]byte(nil), b.Header.PrevHash[:]...), b.Header.Height)
	out = append(out, b.Header.TxRoot[:]...)
	for _, t := range b.Txs {
		raw := t.Serialize()
		out = append(binary.BigEndian.AppendUint32(out, uint32(len(raw))), raw...)
	}
	return out
}

func decodeBlock(p []byte) *Block {
	take := func(n int) []byte { v := p[:n]; p = p[n:]; return v }
	u32 := func() uint32 { return binary.BigEndian.Uint32(take(4)) }
	b := &Block{}
	b.Header.PrevHash = [32]byte(take(32))
	b.Header.Height = binary.BigEndian.Uint64(take(8))
	b.Header.TxRoot = [32]byte(take(32))
	for len(p) > 0 {
		take(4)
		t := &Transaction{Data: bytes.Clone(take(int(u32())))}
		for n := u32(); n > 0; n-- {
			t.Inputs = append(t.Inputs, opFromKey(take(36)))
		}
		for n := u32(); n > 0; n-- {
			v := int64(binary.BigEndian.Uint64(take(8)))
			t.Outputs = append(t.Outputs, TxOutput{v, bytes.Clone(take(int(u32())))})
		}
		b.Txs = append(b.Txs, t)
	}
	return b
}

func (s *BoltStore) PutBlock(b *Block) error {
	hash := b.Header.Hash()
	return s.db.Update(func(tx *bolt.Tx) error { return tx.Bucket(bktBlocks).Put(hash[:], encodeBlock(b)) })
}

func (s *BoltStore) GetBlock(hash [32]byte) (b *Block, err error) {
	err = s.db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktBlocks).Get(hash[:])
		if v == nil {
			return ErrNotFound
		}
		b = decodeBlock(v) // every slice in b is freshly allocated
		return nil
	})
	return b, err
}

func (s *BoltStore) Tip() (h Header, err error) {
	err = s.db.View(func(tx *bolt.Tx) error {
		hash := tx.Bucket(bktMeta).Get(keyTip)
		if hash == nil {
			return ErrEmpty
		}
		h = decodeBlock(tx.Bucket(bktBlocks).Get(hash)).Header
		return nil
	})
	return h, err
}

// ApplyBlock mutates as it validates. That is safe here and nowhere else:
// returning an error rolls the transaction back.
func (s *BoltStore) ApplyBlock(b *Block) error {
	hash := b.Header.Hash()
	return s.db.Update(func(tx *bolt.Tx) error {
		tip := tx.Bucket(bktMeta).Get(keyTip)
		if (tip == nil && b.Header.Height != 0) || (tip != nil && !bytes.Equal(tip, b.Header.PrevHash[:])) {
			return ErrNotOnTip
		}
		utxo := tx.Bucket(bktUTXO)
		for i, t := range b.Txs {
			for _, in := range t.Inputs {
				if utxo.Get(utxoKey(in)) == nil {
					return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
				}
				if err := utxo.Delete(utxoKey(in)); err != nil {
					return err
				}
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				v := append(binary.BigEndian.AppendUint64(nil, uint64(o.Value)), o.PubKeyHash...)
				if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), v); err != nil {
					return err
				}
			}
		}
		if err := tx.Bucket(bktBlocks).Put(hash[:], encodeBlock(b)); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, hash[:])
	})
}

func (s *BoltStore) IterUTXO(fn func(Outpoint, TxOutput) error) error {
	return s.db.View(func(tx *bolt.Tx) error {
		return tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
			return fn(opFromKey(k), TxOutput{int64(binary.BigEndian.Uint64(v[:8])), bytes.Clone(v[8:])})
		})
	})
}

func (s *BoltStore) Close() error { return s.db.Close() }

// ------------------------------------------------ three broken Stores

// memShared hands out the stored block itself instead of a copy.
type memShared struct{ *MemStore }

func (m memShared) GetBlock(hash [32]byte) (*Block, error) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	if b, ok := m.blocks[hash]; ok {
		return b, nil
	}
	return nil, ErrNotFound
}

// memEager mutates the maps as it validates — lesson 10's broken Append.
type memEager struct{ *MemStore }

func (m memEager) ApplyBlock(b *Block) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	if err := m.checkTip(b); err != nil {
		return err
	}
	for i, t := range b.Txs {
		for _, in := range t.Inputs {
			if _, ok := m.utxo[in]; !ok {
				return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
			}
			delete(m.utxo, in)
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			m.utxo[Outpoint{id, uint32(vout)}] = cloneOut(o)
		}
	}
	m.blocks[b.Header.Hash()] = cloneBlock(b)
	h := b.Header
	m.tip = &h
	return nil
}

// boltAliased passes IterUTXO callers a window onto bbolt's memory (example 7).
type boltAliased struct{ *BoltStore }

func (s boltAliased) IterUTXO(fn func(Outpoint, TxOutput) error) error {
	return s.db.View(func(tx *bolt.Tx) error {
		return tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
			return fn(opFromKey(k), TxOutput{int64(binary.BigEndian.Uint64(v[:8])), v[8:]})
		})
	})
}

// ---------------------------------------------------------------- the suite

// T is the three lines of testing.T this suite needs, so it can run inside
// package main. In a real repository the suite is a function taking
// *testing.T and a constructor; see the end of main.
type T struct {
	failed bool
	msg    string
}

type fatal struct{}

func (t *T) Fatalf(format string, args ...any) {
	t.failed, t.msg = true, fmt.Sprintf(format, args...)
	panic(fatal{})
}

func must(t *T, err error) {
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
}

// Fixture is lesson 08's chainBuilder, cut down: a valid chain plus the two
// blocks the tests need to be wrong in specific ways.
type Fixture struct {
	Chain []*Block // 0..12; block 2 spends one of its own outputs
	Bad   *Block   // height 2: tx 1 is valid, tx 2 spends a coin that never existed
	Fork  *Block   // height 1, a sibling of Chain[1]
}

func newFixture() *Fixture {
	addr := func(name string) []byte { s := sha256.Sum256([]byte(name)); return s[:20] }
	mk := func(parent *Block, miner string, txs ...*Transaction) *Block {
		h := Header{}
		if parent != nil {
			h.PrevHash, h.Height = parent.Header.Hash(), parent.Header.Height+1
		}
		cb := &Transaction{Data: binary.BigEndian.AppendUint64(nil, h.Height), Outputs: []TxOutput{{50 * Coin, addr(miner)}}}
		all := append([]*Transaction{cb}, txs...)
		h.TxRoot = all[len(all)-1].TxID()
		return &Block{h, all}
	}
	fx := &Fixture{}
	fx.Chain = append(fx.Chain, mk(nil, "miner"))
	for h := 1; h <= 12; h++ {
		prev := fx.Chain[h-1]
		spend := &Transaction{
			Inputs: []Outpoint{{prev.Txs[0].TxID(), 0}},
			Outputs: []TxOutput{
				{10 * Coin, addr("alice")}, {15 * Coin, addr("bob")}, {20 * Coin, addr("carol")}, {5 * Coin, addr("miner")},
			},
		}
		txs := []*Transaction{spend}
		if h == 2 { // a child in the same block as its parent
			txs = append(txs, &Transaction{Inputs: []Outpoint{{spend.TxID(), 0}}, Outputs: []TxOutput{{10 * Coin, addr("dave")}}})
		}
		fx.Chain = append(fx.Chain, mk(prev, "miner", txs...))
	}
	ghost := Outpoint{sha256.Sum256([]byte("never existed")), 0}
	fx.Bad = mk(fx.Chain[1], "miner",
		&Transaction{Inputs: []Outpoint{{fx.Chain[1].Txs[0].TxID(), 0}}, Outputs: []TxOutput{{50 * Coin, addr("erin")}}},
		&Transaction{Inputs: []Outpoint{ghost}, Outputs: []TxOutput{{1000 * Coin, addr("erin")}}})
	fx.Fork = mk(fx.Chain[0], "someone else")
	return fx
}

func applyAll(t *T, s Store, blocks []*Block) {
	for _, b := range blocks {
		must(t, s.ApplyBlock(b))
	}
}

func collect(t *T, s Store) map[Outpoint]TxOutput {
	set := map[Outpoint]TxOutput{}
	must(t, s.IterUTXO(func(op Outpoint, o TxOutput) error { set[op] = o; return nil }))
	return set
}

func fingerprint(t *T, s Store) [32]byte {
	h := sha256.New()
	must(t, s.IterUTXO(func(op Outpoint, o TxOutput) error {
		h.Write(utxoKey(op))
		h.Write(binary.BigEndian.AppendUint64(nil, uint64(o.Value)))
		h.Write(o.PubKeyHash)
		return nil
	}))
	return [32]byte(h.Sum(nil))
}

type Case struct {
	Name string
	Run  func(t *T, s Store, fx *Fixture)
}

var suite = []Case{
	{"a fresh store has no tip", func(t *T, s Store, fx *Fixture) {
		if _, err := s.Tip(); !errors.Is(err, ErrEmpty) {
			t.Fatalf("Tip() = %v, want ErrEmpty", err)
		}
	}},
	{"applying genesis makes it the tip", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain[:1])
		tip, err := s.Tip()
		must(t, err)
		if tip.Hash() != fx.Chain[0].Header.Hash() {
			t.Fatalf("tip is not genesis")
		}
	}},
	{"GetBlock returns what PutBlock stored", func(t *T, s Store, fx *Fixture) {
		must(t, s.PutBlock(fx.Chain[2]))
		got, err := s.GetBlock(fx.Chain[2].Header.Hash())
		must(t, err)
		if !bytes.Equal(encodeBlock(got), encodeBlock(fx.Chain[2])) {
			t.Fatalf("the block changed on the way through")
		}
	}},
	{"an unknown hash is ErrNotFound", func(t *T, s Store, fx *Fixture) {
		if _, err := s.GetBlock([32]byte{1}); !errors.Is(err, ErrNotFound) {
			t.Fatalf("GetBlock = %v, want ErrNotFound", err)
		}
	}},
	{"a block off the tip is refused", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain[:2])
		if err := s.ApplyBlock(fx.Fork); !errors.Is(err, ErrNotOnTip) {
			t.Fatalf("ApplyBlock(fork) = %v, want ErrNotOnTip", err)
		}
	}},
	{"a rejected block leaves no trace", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain[:2])
		before := fingerprint(t, s)
		if err := s.ApplyBlock(fx.Bad); !errors.Is(err, ErrMissingInput) {
			t.Fatalf("ApplyBlock(bad) = %v, want ErrMissingInput", err)
		}
		if fingerprint(t, s) != before {
			t.Fatalf("the UTXO set changed, though the block was rejected")
		}
		applyAll(t, s, fx.Chain[2:3]) // and the real block 2 still applies
	}},
	{"a block may spend its own outputs", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain[:3])
		set := collect(t, s)
		parent, child := fx.Chain[2].Txs[1].TxID(), fx.Chain[2].Txs[2].TxID()
		if _, ok := set[Outpoint{parent, 0}]; ok {
			t.Fatalf("the output spent inside the block is still unspent")
		}
		if _, ok := set[Outpoint{child, 0}]; !ok {
			t.Fatalf("the child's output is missing")
		}
	}},
	{"IterUTXO walks in key order", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain)
		var prev []byte
		must(t, s.IterUTXO(func(op Outpoint, _ TxOutput) error {
			if k := utxoKey(op); prev != nil && bytes.Compare(prev, k) >= 0 {
				t.Fatalf("keys out of order")
			} else {
				prev = k
			}
			return nil
		}))
	}},
	{"outputs from IterUTXO are the caller's", func(t *T, s Store, fx *Fixture) {
		applyAll(t, s, fx.Chain[:6])
		kept := collect(t, s)
		applyAll(t, s, fx.Chain[6:]) // seven more blocks: pages rewritten, pages reused
		for op, now := range collect(t, s) {
			if was, ok := kept[op]; ok && !bytes.Equal(was.PubKeyHash, now.PubKeyHash) {
				t.Fatalf("an output kept from IterUTXO changed owner after later writes")
			}
		}
	}},
	{"changing a returned block changes nothing", func(t *T, s Store, fx *Fixture) {
		must(t, s.PutBlock(fx.Chain[1]))
		got, err := s.GetBlock(fx.Chain[1].Header.Hash())
		must(t, err)
		got.Txs[1].Outputs[0].Value = 1_000_000 * Coin
		again, err := s.GetBlock(fx.Chain[1].Header.Hash())
		must(t, err)
		if again.Txs[1].Outputs[0].Value != 10*Coin {
			t.Fatalf("the stored block now pays %d coins", again.Txs[1].Outputs[0].Value/Coin)
		}
	}},
}

type Impl struct {
	Name string
	Open func(path string) (Store, error)
}

func run(c Case, im Impl, path string, fx *Fixture) (t *T) {
	t = &T{}
	s, err := im.Open(path)
	if err != nil {
		return &T{true, err.Error()}
	}
	defer s.Close()
	defer func() {
		if r := recover(); r != nil {
			if _, ok := r.(fatal); !ok {
				t.failed, t.msg = true, fmt.Sprint("panic: ", r)
			}
		}
	}()
	c.Run(t, s, fx)
	return t
}

func main() {
	debug.SetPanicOnFault(true) // a stale mmap read becomes a failed test, not a dead process
	dir, err := os.MkdirTemp("", "store-*")
	if err != nil {
		panic(err)
	}
	defer os.RemoveAll(dir)

	impls := []Impl{
		{"memory", func(string) (Store, error) { return NewMemStore(), nil }},
		{"bbolt", func(p string) (Store, error) { return OpenBoltStore(p) }},
		{"mem-shared", func(string) (Store, error) { return memShared{NewMemStore()}, nil }},
		{"mem-eager", func(string) (Store, error) { return memEager{NewMemStore()}, nil }},
		{"bolt-alias", func(p string) (Store, error) {
			s, err := OpenBoltStore(p)
			if err != nil {
				return nil, err
			}
			return boltAliased{s}, nil
		}},
	}

	fx := newFixture()
	cell := func(i int, s string) string { // pad every column but the last
		if i == len(impls)-1 {
			return " " + s
		}
		return fmt.Sprintf(" %-10s", s)
	}
	fmt.Printf("%-42s", "")
	for i, im := range impls {
		fmt.Print(cell(i, im.Name))
	}
	fmt.Println()
	var failures []string
	n := 0
	for _, c := range suite {
		fmt.Printf("%-42s", c.Name)
		for i, im := range impls {
			n++
			t := run(c, im, filepath.Join(dir, fmt.Sprintf("%d.db", n)), fx)
			verdict := "PASS"
			if t.failed {
				verdict = "FAIL"
				failures = append(failures, fmt.Sprintf("  %-10s %s: %s", im.Name, c.Name, t.msg))
			}
			fmt.Print(cell(i, verdict))
		}
		fmt.Println()
	}

	fmt.Println("\n=== what failed, and why ===")
	for _, f := range failures {
		fmt.Println(f)
	}
	fmt.Println()
	fmt.Println("  mem-shared  GetBlock returns the stored pointer. A caller that edits")
	fmt.Println("              the block it got has edited the store — the in-memory")
	fmt.Println("              twin of bbolt's borrowed slices.")
	fmt.Println("  mem-eager   mutates while validating. bbolt forgives that with a")
	fmt.Println("              rollback; a map cannot, so the rejected block's first")
	fmt.Println("              transaction stayed applied.")
	fmt.Println("  bolt-alias  hands out slices into the mmap, and they changed owner")
	fmt.Println("              once later blocks reused the pages (example 7).")
	fmt.Println()
	fmt.Println("  The two correct implementations differ in exactly the places the")
	fmt.Println("  broken ones do — which is why the suite has to be SHARED. In a real")
	fmt.Println("  repository it is one function and two one-line tests:")
	fmt.Println()
	fmt.Println("      func TestMemStore(t *testing.T)  { storetest.Run(t, NewMemStore) }")
	fmt.Println("      func TestBoltStore(t *testing.T) { storetest.Run(t, openTempBolt(t)) }")
}
```

**Output:**

```
                                           memory     bbolt      mem-shared mem-eager  bolt-alias
a fresh store has no tip                   PASS       PASS       PASS       PASS       PASS
applying genesis makes it the tip          PASS       PASS       PASS       PASS       PASS
GetBlock returns what PutBlock stored      PASS       PASS       PASS       PASS       PASS
an unknown hash is ErrNotFound             PASS       PASS       PASS       PASS       PASS
a block off the tip is refused             PASS       PASS       PASS       PASS       PASS
a rejected block leaves no trace           PASS       PASS       PASS       FAIL       PASS
a block may spend its own outputs          PASS       PASS       PASS       PASS       PASS
IterUTXO walks in key order                PASS       PASS       PASS       PASS       PASS
outputs from IterUTXO are the caller's     PASS       PASS       PASS       PASS       FAIL
changing a returned block changes nothing  PASS       PASS       FAIL       PASS       PASS

=== what failed, and why ===
  mem-eager  a rejected block leaves no trace: the UTXO set changed, though the block was rejected
  bolt-alias outputs from IterUTXO are the caller's: an output kept from IterUTXO changed owner after later writes
  mem-shared changing a returned block changes nothing: the stored block now pays 1000000 coins

  mem-shared  GetBlock returns the stored pointer. A caller that edits
              the block it got has edited the store — the in-memory
              twin of bbolt's borrowed slices.
  mem-eager   mutates while validating. bbolt forgives that with a
              rollback; a map cannot, so the rejected block's first
              transaction stayed applied.
  bolt-alias  hands out slices into the mmap, and they changed owner
              once later blocks reused the pages (example 7).

  The two correct implementations differ in exactly the places the
  broken ones do — which is why the suite has to be SHARED. In a real
  repository it is one function and two one-line tests:

      func TestMemStore(t *testing.T)  { storetest.Run(t, NewMemStore) }
      func TestBoltStore(t *testing.T) { storetest.Run(t, openTempBolt(t)) }
```

---

## 17. A hot backup, and a snapshot to start from

`🔴 hard` · *Pruning and snapshots*

`Tx.WriteTo` copies the entire database from inside a read transaction while the writer carries on, because bbolt never overwrites a page a reader can see. A UTXO snapshot does less, and more: just the set at one height, with a digest that a brand-new node checks against a constant compiled into it — the idea behind Bitcoin Core's assumeutxo.

**Steps:**

1. Back up a live node at height 50 while it commits blocks 51–60 during the copy.
2. Open the backup and check it against a replay of the chain.
3. Write a UTXO snapshot from the backup and print its digest.
4. Bootstrap a new node from headers plus the snapshot, connect blocks 51–80, and compare it with the full node.
5. Hand it two tampered snapshots: one careless, one careful.

```go
package main

import (
	"bufio"
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"sort"
	"sync"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// A hot backup, and a snapshot to start from.
//
// Two ways to move a node's state without replaying the whole chain:
//
//   1. Tx.WriteTo — a consistent copy of the ENTIRE database, taken inside a
//      read transaction while the node goes on connecting blocks. A backup.
//   2. A snapshot of just the UTXO set at one height, with a digest that a new
//      node checks against a value compiled into its own source. A fast
//      bootstrap: the idea behind Bitcoin Core's assumeutxo.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   blocks    hash           version | header | txs   (header only, on a node
//                                                       that started from a snapshot)
//   heights   height BE      hash
//   utxo      txid | vout    value [8] | pkh [20]
//   meta      "tip"          hash of the chain's tip
//             "utxo-tip"     hash of the block the UTXO set reflects
//
// ------------------------------------------------------- the snapshot file
//
//   magic "UTXOSNAP" [8] | version [1] | height [8] | block hash [32] | count [8]
//   count × ( key [36] | value [28] )
//   digest: SHA-256 of the entries [32]
// ---------------------------------------------------------------------------

const (
	HeaderSize   = 92
	Coin         = int64(100_000_000)
	SnapHeadSize = 57
	SnapEntry    = 64

	// The digest of the height-50 snapshot, pinned the way a release pins it.
	// A new node trusts THIS constant — not the file, and not whoever sent it.
	TrustedSnapshot = "39591af7f8ec925381f944cfd47d985674dfde2c69b9d0735fd3f2246ba6df47"
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
	keyUTXOTip = []byte("utxo-tip")
)

var (
	ErrSnapshotCorrupt   = errors.New("snapshot does not match its own digest")
	ErrSnapshotUntrusted = errors.New("snapshot digest is not the one this node was built to trust")
	ErrSnapshotOffChain  = errors.New("snapshot block is not in this node's header chain")
)

// ---------------------------------------------------------------- the chain

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

func (h Header) Bytes() []byte {
	b := make([]byte, 0, HeaderSize)
	b = binary.BigEndian.AppendUint32(b, h.Version)
	b = append(b, h.PrevHash[:]...)
	b = append(b, h.MerkleRoot[:]...)
	b = binary.BigEndian.AppendUint64(b, uint64(h.Timestamp))
	b = binary.BigEndian.AppendUint32(b, h.Bits)
	b = binary.BigEndian.AppendUint32(b, h.Nonce)
	b = binary.BigEndian.AppendUint64(b, h.Height)
	return b
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxOutput struct {
	Value int64
	PKH   [20]byte
}

type Transaction struct {
	Data    []byte
	Inputs  []Outpoint
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	b := append(binary.BigEndian.AppendUint32(nil, uint32(len(t.Data))), t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = binary.BigEndian.AppendUint32(append(b, in.TxID[:]...), in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = append(binary.BigEndian.AppendUint64(b, uint64(o.Value)), o.PKH[:]...)
	}
	return b
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

type Block struct {
	Header Header
	Txs    []*Transaction
}

func (b *Block) Encode() []byte {
	out := append([]byte{0x01}, b.Header.Bytes()...)
	for _, t := range b.Txs {
		raw := t.Serialize()
		out = append(binary.BigEndian.AppendUint32(out, uint32(len(raw))), raw...)
	}
	return out
}

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func buildChain(n int) []*Block {
	miner := pkh("miner")
	people := [][20]byte{pkh("alice"), pkh("bob"), pkh("carol"), pkh("dave")}
	var chain []*Block
	for h := 0; h < n; h++ {
		txs := []*Transaction{{Data: be64(uint64(h)), Outputs: []TxOutput{{50 * Coin, miner}}}}
		hdr := Header{Version: 1, Timestamp: 1700000000 + int64(h)*600, Bits: 0x2000ffff, Height: uint64(h)}
		if h > 0 {
			prev := chain[h-1]
			hdr.PrevHash = prev.Header.Hash()
			txs = append(txs, &Transaction{
				Inputs: []Outpoint{{prev.Txs[0].TxID(), 0}},
				Outputs: []TxOutput{
					{20 * Coin, people[h%4]}, {15 * Coin, people[(h+1)%4]}, {10 * Coin, people[(h+2)%4]}, {5 * Coin, miner},
				},
			})
		}
		hdr.MerkleRoot = sha256.Sum256(txs[len(txs)-1].Serialize())
		chain = append(chain, &Block{hdr, txs})
	}
	return chain
}

// ---------------------------------------------------------------- the store

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func open(path string) *bolt.DB {
	db, err := bolt.Open(path, 0o600, &bolt.Options{
		Timeout:  time.Second,
		PageSize: 4096, // pinned, so the sizes below match everywhere
		// A writer that must grow the memory map waits for every open read
		// transaction to finish. A backup IS a long read transaction, so map
		// enough up front that the writer never has to (see DB.Begin's doc).
		InitialMmapSize: 64 << 20,
	})
	check(err)
	check(db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		return nil
	}))
	return db
}

func connect(db *bolt.DB, b *Block) error {
	hash := b.Header.Hash()
	return db.Update(func(tx *bolt.Tx) error {
		utxo := tx.Bucket(bktUTXO)
		for _, t := range b.Txs {
			for _, in := range t.Inputs {
				if utxo.Get(utxoKey(in)) == nil {
					return fmt.Errorf("block %d: input missing", b.Header.Height)
				}
				if err := utxo.Delete(utxoKey(in)); err != nil {
					return err
				}
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), append(be64(uint64(o.Value)), o.PKH[:]...)); err != nil {
					return err
				}
			}
		}
		if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
			return err
		}
		if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
			return err
		}
		if err := tx.Bucket(bktMeta).Put(keyUTXOTip, hash[:]); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, hash[:])
	})
}

func heightOf(tx *bolt.Tx, hash []byte) int {
	if hash == nil {
		return -1
	}
	return int(binary.BigEndian.Uint64(tx.Bucket(bktBlocks).Get(hash)[1+84 : 1+92]))
}

func utxoFingerprint(tx *bolt.Tx) [32]byte {
	h := sha256.New()
	_ = tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
		h.Write(k)
		h.Write(v)
		return nil
	})
	return [32]byte(h.Sum(nil))
}

func replayFingerprint(chain []*Block, tip int) [32]byte {
	set := map[string][]byte{}
	for _, b := range chain[:tip+1] {
		for _, t := range b.Txs {
			for _, in := range t.Inputs {
				delete(set, string(utxoKey(in)))
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				set[string(utxoKey(Outpoint{id, uint32(vout)}))] = append(be64(uint64(o.Value)), o.PKH[:]...)
			}
		}
	}
	keys := make([]string, 0, len(set))
	for k := range set {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	h := sha256.New()
	for _, k := range keys {
		h.Write([]byte(k))
		h.Write(set[k])
	}
	return [32]byte(h.Sum(nil))
}

// ------------------------------------------------------------ the backup

// gatedWriter holds the copy at its first write until told to go on, so the
// writer below is GUARANTEED to commit blocks while the copy is in progress.
type gatedWriter struct {
	w       io.Writer
	once    sync.Once
	started chan struct{}
	resume  chan struct{}
}

func (g *gatedWriter) Write(p []byte) (int, error) {
	g.once.Do(func() {
		close(g.started)
		<-g.resume
	})
	return g.w.Write(p)
}

// ---------------------------------------------------------- the snapshot

// WriteSnapshot streams the UTXO set out of ONE read transaction, so the
// entries, the height and the block hash all describe the same moment.
func WriteSnapshot(db *bolt.DB, path string) (digest [32]byte, count int, err error) {
	f, err := os.Create(path)
	if err != nil {
		return digest, 0, err
	}
	defer f.Close()
	w := bufio.NewWriter(f)
	err = db.View(func(tx *bolt.Tx) error {
		block := tx.Bucket(bktMeta).Get(keyUTXOTip) // the block the SET reflects, not merely the tip
		utxo := tx.Bucket(bktUTXO)
		count = utxo.Stats().KeyN
		w.WriteString("UTXOSNAP")
		w.WriteByte(1)
		w.Write(be64(uint64(heightOf(tx, block))))
		w.Write(block)
		w.Write(be64(uint64(count)))
		h := sha256.New()
		if err := utxo.ForEach(func(k, v []byte) error {
			w.Write(k)
			w.Write(v)
			h.Write(k)
			h.Write(v)
			return nil
		}); err != nil {
			return err
		}
		digest = [32]byte(h.Sum(nil))
		_, err := w.Write(digest[:])
		return err
	})
	if err == nil {
		err = w.Flush()
	}
	if err == nil {
		err = f.Sync() // a snapshot nobody synced is a hope, not a file
	}
	return digest, count, err
}

// LoadSnapshot checks a snapshot three ways before a byte of it touches the
// database: against its own digest (a bad download), against the digest this
// binary trusts (a lie), and against the node's header chain (the wrong
// block). Then it loads it in one write transaction.
func LoadSnapshot(db *bolt.DB, path, trusted string) (height uint64, count int, err error) {
	raw, err := os.ReadFile(path) // fine at this size; a real one is streamed
	if err != nil {
		return 0, 0, err
	}
	if len(raw) < SnapHeadSize+32 || string(raw[:8]) != "UTXOSNAP" || raw[8] != 1 {
		return 0, 0, ErrSnapshotCorrupt
	}
	height = binary.BigEndian.Uint64(raw[9:17])
	block := raw[17:49]
	n := binary.BigEndian.Uint64(raw[49:57])
	entries := raw[SnapHeadSize : len(raw)-32]
	if uint64(len(entries)) != n*SnapEntry {
		return 0, 0, ErrSnapshotCorrupt
	}
	sum := sha256.Sum256(entries)
	if !bytes.Equal(sum[:], raw[len(raw)-32:]) {
		return 0, 0, ErrSnapshotCorrupt
	}
	if hex.EncodeToString(sum[:]) != trusted {
		return 0, 0, fmt.Errorf("%w: got %x…", ErrSnapshotUntrusted, sum[:4])
	}
	err = db.Update(func(tx *bolt.Tx) error {
		if !bytes.Equal(tx.Bucket(bktHeights).Get(be64(height)), block) {
			return ErrSnapshotOffChain
		}
		utxo := tx.Bucket(bktUTXO)
		for p := entries; len(p) > 0; p = p[SnapEntry:] {
			if err := utxo.Put(p[:36], p[36:SnapEntry]); err != nil {
				return err
			}
		}
		if err := tx.Bucket(bktMeta).Put(keyUTXOTip, block); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, block)
	})
	return height, int(n), err
}

func fileSize(path string) int64 {
	fi, err := os.Stat(path)
	check(err)
	return fi.Size()
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "backup-*")
	check(err)
	defer os.RemoveAll(dir)
	chain := buildChain(81)

	livePath := filepath.Join(dir, "live.db")
	live := open(livePath)
	defer live.Close()
	for _, b := range chain[:51] {
		check(connect(live, b))
	}

	// -------------------------------------------------------------------- 1
	fmt.Println("=== 1. a hot backup while blocks keep arriving ===")
	backupPath := filepath.Join(dir, "backup.db")
	f, err := os.Create(backupPath)
	check(err)
	gate := &gatedWriter{w: f, started: make(chan struct{}), resume: make(chan struct{})}
	var copied int64
	copyDone := make(chan error, 1)
	go func() {
		copyDone <- live.View(func(tx *bolt.Tx) error {
			n, err := tx.WriteTo(gate)
			copied = n
			return err
		})
	}()
	<-gate.started // the read transaction is open and the first bytes are on their way
	for _, b := range chain[51:61] {
		check(connect(live, b))
	}
	close(gate.resume)
	check(<-copyDone)
	check(f.Sync())
	check(f.Close())
	for _, b := range chain[61:] {
		check(connect(live, b))
	}

	backup := open(backupPath)
	defer backup.Close()
	check(backup.View(func(tx *bolt.Tx) error {
		checkErrs := 0
		for range tx.Check() {
			checkErrs++
		}
		tip, utxoTip := heightOf(tx, tx.Bucket(bktMeta).Get(keyTip)), heightOf(tx, tx.Bucket(bktMeta).Get(keyUTXOTip))
		fmt.Println("  the copy began at height 50; blocks 51-60 were committed while it ran")
		fmt.Printf("  backup: %d bytes · Check errors %d · tip %d · utxo-tip %d · UTXO == replay to %d: %v\n",
			copied, checkErrs, tip, utxoTip, tip, utxoFingerprint(tx) == replayFingerprint(chain, tip))
		return nil
	}))
	check(live.View(func(tx *bolt.Tx) error {
		fmt.Printf("  live:   tip %d\n", heightOf(tx, tx.Bucket(bktMeta).Get(keyTip)))
		return nil
	}))
	fmt.Println()
	fmt.Println("  The backup is the database exactly as it was when the read")
	fmt.Println("  transaction began. bbolt never overwrites a page a reader can see:")
	fmt.Println("  the writer put blocks 51-60 on NEW pages, and the old ones stayed")
	fmt.Println("  put until the copy finished. Copying the live file with cp gets no")
	fmt.Println("  such promise — it can take the meta page from one moment and the")
	fmt.Println("  pages it points at from another.")

	// -------------------------------------------------------------------- 2
	fmt.Println("\n=== 2. a UTXO snapshot from the backup ===")
	snapPath := filepath.Join(dir, "utxo-50.snap")
	digest, count, err := WriteSnapshot(backup, snapPath)
	check(err)
	fmt.Printf("  height 50: %d entries in %d bytes; the whole database at height 50 was %d\n",
		count, fileSize(snapPath), copied)
	fmt.Printf("  digest %x\n", digest)
	fmt.Printf("  pinned in this binary: %s\n", TrustedSnapshot)

	fmt.Println("\n=== 3. a brand-new node starts from it ===")
	freshPath := filepath.Join(dir, "fresh.db")
	fresh := open(freshPath)
	defer fresh.Close()
	check(fresh.Update(func(tx *bolt.Tx) error { // headers-first (lesson 13): every header, no bodies
		for _, b := range chain[:51] {
			hash := b.Header.Hash()
			if err := tx.Bucket(bktBlocks).Put(hash[:], append([]byte{0x01}, b.Header.Bytes()...)); err != nil {
				return err
			}
			if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
				return err
			}
		}
		return nil
	}))
	height, n, err := LoadSnapshot(fresh, snapPath, TrustedSnapshot)
	check(err)
	fmt.Printf("  loaded %d entries at height %d, after checking all three\n", n, height)
	for _, b := range chain[51:] {
		check(connect(fresh, b))
	}
	var same bool
	check(fresh.View(func(tx *bolt.Tx) error {
		f := utxoFingerprint(tx)
		return live.View(func(ltx *bolt.Tx) error {
			same = f == utxoFingerprint(ltx)
			return nil
		})
	}))
	fmt.Printf("  connected blocks 51-80: 30 blocks replayed instead of 81\n")
	fmt.Printf("  its UTXO set at height 80 equals the full node's: %v\n", same)

	fmt.Println("\n=== 4. snapshots it must refuse ===")
	raw, err := os.ReadFile(snapPath)
	check(err)
	try := func(label string, mutate func([]byte)) {
		bad := bytes.Clone(raw)
		mutate(bad)
		p := filepath.Join(dir, "bad.snap")
		check(os.WriteFile(p, bad, 0o600))
		other := open(filepath.Join(dir, fmt.Sprintf("other-%d.db", len(label))))
		defer other.Close()
		check(other.Update(func(tx *bolt.Tx) error {
			for _, b := range chain[:51] {
				hash := b.Header.Hash()
				if err := tx.Bucket(bktBlocks).Put(hash[:], append([]byte{0x01}, b.Header.Bytes()...)); err != nil {
					return err
				}
				if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
					return err
				}
			}
			return nil
		}))
		_, _, err := LoadSnapshot(other, p, TrustedSnapshot)
		fmt.Printf("  %-44s %v\n", label, err)
	}
	firstValue := SnapHeadSize + 36
	try("one satoshi added, digest left alone", func(b []byte) { b[firstValue+7]++ })
	try("one satoshi added, digest recomputed", func(b []byte) {
		b[firstValue+7]++
		sum := sha256.Sum256(b[SnapHeadSize : len(b)-32])
		copy(b[len(b)-32:], sum[:])
	})
	fmt.Println()
	fmt.Println("  The embedded digest only proves the file arrived intact. Anyone who")
	fmt.Println("  can edit the file can recompute it. What makes a snapshot safe is")
	fmt.Println("  the digest reviewed into the node's source — Bitcoin Core keeps its")
	fmt.Println("  assumeutxo hashes in the chain parameters — plus the node validating")
	fmt.Println("  the old blocks in the background, to confirm it later.")
}
```

**Output:**

```
=== 1. a hot backup while blocks keep arriving ===
  the copy began at height 50; blocks 51-60 were committed while it ran
  backup: 118784 bytes · Check errors 0 · tip 50 · utxo-tip 50 · UTXO == replay to 50: true
  live:   tip 80

  The backup is the database exactly as it was when the read
  transaction began. bbolt never overwrites a page a reader can see:
  the writer put blocks 51-60 on NEW pages, and the old ones stayed
  put until the copy finished. Copying the live file with cp gets no
  such promise — it can take the meta page from one moment and the
  pages it points at from another.

=== 2. a UTXO snapshot from the backup ===
  height 50: 201 entries in 12953 bytes; the whole database at height 50 was 118784
  digest 39591af7f8ec925381f944cfd47d985674dfde2c69b9d0735fd3f2246ba6df47
  pinned in this binary: 39591af7f8ec925381f944cfd47d985674dfde2c69b9d0735fd3f2246ba6df47

=== 3. a brand-new node starts from it ===
  loaded 201 entries at height 50, after checking all three
  connected blocks 51-80: 30 blocks replayed instead of 81
  its UTXO set at height 80 equals the full node's: true

=== 4. snapshots it must refuse ===
  one satoshi added, digest left alone         snapshot does not match its own digest
  one satoshi added, digest recomputed         snapshot digest is not the one this node was built to trust: got 773222b6…

  The embedded digest only proves the file arrived intact. Anyone who
  can edit the file can recompute it. What makes a snapshot safe is
  the digest reviewed into the node's source — Bitcoin Core keeps its
  assumeutxo hashes in the chain parameters — plus the node validating
  the old blocks in the background, to confirm it later.
```

---

## 18. The chain on disk

`🔴 hard` · *Assembly*

Lesson 11's chain, wallet and mempool, with the chain moved into a bbolt file: blocks, heights, the UTXO set, an address index, undo records and a tip pointer, all written by one transaction per block. The node shuts down with a payment still pending and starts again on the same file; a block fails its very last check after touching every entry; and the stored state is checked against a replay of the stored blocks.

**Steps:**

1. Create the chain file, mature a coinbase, fund three wallets, and send four payments into room for three.
2. Shut the node down with one payment still waiting, reopen the file, and let the wallets broadcast again.
3. Mine a block whose coinbase claims one satoshi too many, and check what it left behind.
4. Check conservation, replay the stored blocks against the stored UTXO set, and count the undo records.
5. Read the diff from lesson 11, and what lesson 13 changes.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"math/big"
	"os"
	"path/filepath"
	"sort"
	"time"

	"github.com/ethereum/go-ethereum/crypto"
	bolt "go.etcd.io/bbolt"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Lesson 11's chain, wallet and mempool — with the chain on disk.
//
// NEW in this lesson:
//   + the chain is a bbolt file, not a slice and a map: blocks, a height
//     index, the UTXO set, an address index, undo records and a tip pointer
//   + Append is ONE write transaction. It validates and writes in a single
//     pass, and an error at any point — even the coinbase check, which can
//     only run once every fee is known — rolls all of it back
//   + OpenChain refuses a newer schema, and a state whose tip and UTXO
//     marker disagree
//   + a restart: the chain survives, the mempool does not, the wallet
//     broadcasts again
//   + Replay: the stored UTXO set checked against the stored blocks
//
// UNCHANGED from lessons 08-11 and elided so the new code is visible:
// difficulty retargeting, median-time-past, RBF, CPFP, eviction and chains
// of unconfirmed transactions. Lessons 10 and 11's example 18 have them.
// ===========================================================================

const (
	HeaderVersion    = 1
	HeaderSize       = 92
	CoinbaseMaturity = 5 // Bitcoin: 100
	HalvingPeriod    = 20
	Coin             = int64(100_000_000)
	InitialReward    = 50 * Coin
	MaxMoney         = 21_000_000 * Coin

	MaxBlockSize = 700 // bytes of transactions, so the fee market bites
	MinRelayRate = 1   // sat/byte

	SchemaVersion = 1    // NEW: the database format (example 11)
	RecordV1      = 0x01 // NEW: the first byte of every block and UTXO record
)

// ---------------------------------------------------- header & PoW (08, 09)

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

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

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

func BitsToTarget(bits uint32) *big.Int {
	exp, m := bits>>24, bits&0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

var maxTarget = BitsToTarget(0x2000ffff)

func CheckPoW(h Header) bool {
	t := BitsToTarget(h.Bits)
	if t.Sign() <= 0 || t.Cmp(maxTarget) > 0 {
		return false
	}
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(t) < 0
}

func Mine(h Header) Header {
	target := BitsToTarget(h.Bits)
	buf := h.Bytes()
	h1, h2 := sha256.New(), sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var val big.Int
	for nonce := uint32(0); ; nonce++ {
		binary.BigEndian.PutUint32(buf[80:84], nonce)
		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])
		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])
		if val.SetBytes(d2).Cmp(target) < 0 {
			h.Nonce = nonce
			return h
		}
	}
}

// ------------------------------------------------------- transactions (10)

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint
	Signature []byte
	PubKey    []byte
	Sequence  uint32
}

type TxOutput struct {
	Value      int64
	PubKeyHash []byte
}

type Transaction struct {
	Inputs  []TxInput
	Outputs []TxOutput
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

func (t *Transaction) Serialize() []byte {
	var b bytes.Buffer
	binary.Write(&b, binary.BigEndian, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b.Write(in.Prev.TxID[:])
		binary.Write(&b, binary.BigEndian, in.Prev.Index)
		binary.Write(&b, binary.BigEndian, in.Sequence)
		writeBytes(&b, in.Signature)
		writeBytes(&b, in.PubKey)
	}
	binary.Write(&b, binary.BigEndian, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		binary.Write(&b, binary.BigEndian, o.Value)
		writeBytes(&b, o.PubKeyHash)
	}
	return b.Bytes()
}

func writeBytes(b *bytes.Buffer, p []byte) {
	binary.Write(b, binary.BigEndian, uint32(len(p)))
	b.Write(p)
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

func (t *Transaction) Size() int64 { return int64(len(t.Serialize())) }

func (t *Transaction) TrimmedCopy() *Transaction {
	ins := make([]TxInput, len(t.Inputs))
	for i, in := range t.Inputs {
		ins[i] = TxInput{Prev: in.Prev, Sequence: in.Sequence}
	}
	outs := make([]TxOutput, len(t.Outputs))
	for i, o := range t.Outputs {
		outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
	}
	return &Transaction{Inputs: ins, Outputs: outs}
}

func (t *Transaction) SigHash(i int, prevPKH []byte) [32]byte {
	c := t.TrimmedCopy()
	c.Inputs[i].PubKey = prevPKH
	return c.TxID()
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func Subsidy(height int64) int64 {
	h := height / HalvingPeriod
	if h >= 64 {
		return 0
	}
	return InitialReward >> uint(h)
}

func NewCoinbase(height int64, to []byte, value int64) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height))
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data, Sequence: 0xffffffff}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
}

const tagLeaf, tagNode byte = 0x00, 0x01

func MerkleRoot(txs []*Transaction) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256([]byte{tagLeaf})
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		id := t.TxID()
		level[i] = sha256.Sum256(append([]byte{tagLeaf}, id[:]...))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i])
				continue
			}
			b := append([]byte{tagNode}, level[i][:]...)
			next = append(next, sha256.Sum256(append(b, level[i+1][:]...)))
		}
		level = next
	}
	return level[0]
}

type Block struct {
	Header Header
	Txs    []*Transaction
}

type UTXOEntry struct {
	Out      TxOutput
	Height   int64
	Coinbase bool
}

// ----------------------------------------------------------- records (NEW)

// Encode is the stored block: the same bytes lesson 08 hashes, behind a
// version byte (examples 1 and 11).
func (b Block) Encode() []byte {
	var buf bytes.Buffer
	buf.WriteByte(RecordV1)
	buf.Write(b.Header.Bytes())
	binary.Write(&buf, binary.BigEndian, uint32(len(b.Txs)))
	for _, t := range b.Txs {
		writeBytes(&buf, t.Serialize())
	}
	return buf.Bytes()
}

type reader struct {
	p   []byte
	err error
}

func (r *reader) take(n int) []byte {
	if r.err != nil {
		return nil
	}
	if n < 0 || n > len(r.p) {
		r.err = ErrBadRecord
		return nil
	}
	v := r.p[:n]
	r.p = r.p[n:]
	return v
}

func (r *reader) u32() uint32 {
	if v := r.take(4); v != nil {
		return binary.BigEndian.Uint32(v)
	}
	return 0
}

func (r *reader) u64() uint64 {
	if v := r.take(8); v != nil {
		return binary.BigEndian.Uint64(v)
	}
	return 0
}

// bytes COPIES: the input is memory that belongs to bbolt (example 4).
func (r *reader) bytes() []byte { return bytes.Clone(r.take(int(r.u32()))) }

func decodeHeader(r *reader) Header {
	var h Header
	h.Version = r.u32()
	copy(h.PrevHash[:], r.take(32))
	copy(h.MerkleRoot[:], r.take(32))
	h.Timestamp = int64(r.u64())
	h.Bits = r.u32()
	h.Nonce = r.u32()
	h.Height = r.u64()
	return h
}

func decodeBlock(p []byte) (Block, error) {
	if len(p) == 0 || p[0] != RecordV1 {
		return Block{}, ErrBadRecord
	}
	r := &reader{p: p[1:]}
	b := Block{Header: decodeHeader(r)}
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		tr := &reader{p: r.take(int(r.u32()))}
		t := &Transaction{}
		for m := tr.u32(); m > 0 && tr.err == nil; m-- {
			var in TxInput
			copy(in.Prev.TxID[:], tr.take(32))
			in.Prev.Index = tr.u32()
			in.Sequence = tr.u32()
			in.Signature = tr.bytes()
			in.PubKey = tr.bytes()
			t.Inputs = append(t.Inputs, in)
		}
		for m := tr.u32(); m > 0 && tr.err == nil; m-- {
			var o TxOutput
			o.Value = int64(tr.u64())
			o.PubKeyHash = tr.bytes()
			t.Outputs = append(t.Outputs, o)
		}
		if tr.err != nil {
			return Block{}, tr.err
		}
		b.Txs = append(b.Txs, t)
	}
	return b, r.err
}

// entry = version [1] | value [8] | height [8] | coinbase [1] | pkh
func encodeEntry(e UTXOEntry) []byte {
	b := []byte{RecordV1}
	b = binary.BigEndian.AppendUint64(b, uint64(e.Out.Value))
	b = binary.BigEndian.AppendUint64(b, uint64(e.Height))
	if e.Coinbase {
		b = append(b, 1)
	} else {
		b = append(b, 0)
	}
	return append(b, e.Out.PubKeyHash...)
}

func decodeEntry(v []byte) (UTXOEntry, error) {
	if len(v) < 18 || v[0] != RecordV1 {
		return UTXOEntry{}, ErrBadRecord
	}
	return UTXOEntry{
		Out:      TxOutput{Value: int64(binary.BigEndian.Uint64(v[1:9])), PubKeyHash: bytes.Clone(v[18:])},
		Height:   int64(binary.BigEndian.Uint64(v[9:17])),
		Coinbase: v[17] == 1,
	}, nil
}

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

// ------------------------------------------------------ the chain, on disk (NEW)

// ---------------------------------------------------------------- the schema
//
//   bucket    key                         value
//   -------   -------------------------   ----------------------------------------
//   blocks    hash [32]                   0x01 | header [92] | count | txs
//   heights   height [8] BE               hash — the canonical chain only
//   utxo      txid [32] | vout [4] BE     0x01 | value | height | coinbase | pkh
//   byaddr    pkh [20] | txid | vout      (empty)                        example 8
//   undo      hash [32]                   (key [36] | len [4] | entry)*   lesson 14
//   meta      "schema"                    SchemaVersion                  example 11
//             "tip"                       hash
//             "utxo-tip"                  hash, written with every UTXO change
//
//   A block is ONE write transaction touching all of them.
// ---------------------------------------------------------------------------

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktAddr    = []byte("byaddr")
	bktUndo    = []byte("undo")
	bktMeta    = []byte("meta")
	keySchema  = []byte("schema")
	keyTip     = []byte("tip")
	keyUTXOTip = []byte("utxo-tip")
)

var (
	ErrBadPoW       = errors.New("insufficient proof of work")
	ErrNoCoinbase   = errors.New("first transaction is not a coinbase")
	ErrBadMerkle    = errors.New("merkle root mismatch")
	ErrOversize     = errors.New("block is too large")
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrDoubleSpend  = errors.New("outpoint spent twice in one block")
	ErrImmature     = errors.New("coinbase output is not yet spendable")
	ErrBadSig       = errors.New("signature does not verify")
	ErrKeyMismatch  = errors.New("public key does not match the output")
	ErrNotCovered   = errors.New("outputs exceed inputs")
	ErrOverClaim    = errors.New("coinbase claims more than subsidy plus fees")

	ErrNotOnTip     = errors.New("parent is not the current tip")      // NEW
	ErrInconsistent = errors.New("tip and UTXO set disagree: reindex") // NEW
	ErrNewerSchema  = errors.New("database is newer than this binary") // NEW
	ErrBadRecord    = errors.New("corrupt or unknown record")          // NEW
)

type Chain struct {
	db *bolt.DB
}

// OpenChain opens (or creates) the chain file and refuses to run on a state
// it cannot trust.
func OpenChain(path string, minerPKH []byte) (*Chain, error) {
	db, err := bolt.Open(path, 0o600, &bolt.Options{Timeout: time.Second})
	if err != nil {
		return nil, err
	}
	fresh := false
	err = db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktUTXO, bktAddr, bktUndo, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		meta := tx.Bucket(bktMeta)
		switch v := meta.Get(keySchema); {
		case v == nil:
			fresh = true
			return meta.Put(keySchema, []byte{SchemaVersion})
		case v[0] > SchemaVersion:
			return fmt.Errorf("%w: v%d", ErrNewerSchema, v[0])
		}
		if !bytes.Equal(meta.Get(keyTip), meta.Get(keyUTXOTip)) {
			return ErrInconsistent
		}
		return nil
	})
	if err != nil {
		db.Close()
		return nil, err
	}
	c := &Chain{db}
	if fresh {
		cb := NewCoinbase(0, minerPKH, Subsidy(0))
		txs := []*Transaction{cb}
		h := Mine(Header{Version: HeaderVersion, MerkleRoot: MerkleRoot(txs),
			Timestamp: 1700000000, Bits: 0x2000ffff, Height: 0})
		if err := c.Append(Block{h, txs}); err != nil {
			db.Close()
			return nil, err
		}
	}
	return c, nil
}

func (c *Chain) Close() error { return c.db.Close() }

// Append validates b against the stored UTXO set and connects it, in ONE
// write transaction. Lesson 10 built a Delta and applied it afterwards; here
// the transaction IS the delta.
func (c *Chain) Append(b Block) error {
	return c.db.Update(func(tx *bolt.Tx) error { return connect(tx, b) })
}

func connect(tx *bolt.Tx, b Block) error {
	meta, utxo, byaddr := tx.Bucket(bktMeta), tx.Bucket(bktUTXO), tx.Bucket(bktAddr)
	tip := meta.Get(keyTip)
	if (tip == nil && b.Header.Height != 0) || (tip != nil && !bytes.Equal(tip, b.Header.PrevHash[:])) {
		return ErrNotOnTip
	}
	if !CheckPoW(b.Header) {
		return ErrBadPoW
	}
	if len(b.Txs) == 0 || !b.Txs[0].IsCoinbase() {
		return ErrNoCoinbase
	}
	if MerkleRoot(b.Txs) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	var body int64
	for _, t := range b.Txs[1:] {
		body += t.Size()
	}
	if body > MaxBlockSize {
		return fmt.Errorf("%w: %d > %d", ErrOversize, body, MaxBlockSize)
	}

	height := int64(b.Header.Height)
	spentBy := map[Outpoint]int{} // only to NAME the error: the store would catch it anyway
	var fees int64
	var undo []byte

	for i, t := range b.Txs {
		var in int64
		if !t.IsCoinbase() {
			for k, input := range t.Inputs {
				op := input.Prev
				if j, dup := spentBy[op]; dup {
					return fmt.Errorf("tx %d: %w (also tx %d)", i, ErrDoubleSpend, j)
				}
				key := utxoKey(op)
				v := utxo.Get(key) // outputs created earlier in THIS block are visible too
				if v == nil {
					return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
				}
				e, err := decodeEntry(v)
				if err != nil {
					return err
				}
				if e.Coinbase && height-e.Height < CoinbaseMaturity {
					return fmt.Errorf("tx %d: %w", i, ErrImmature)
				}
				if !bytes.Equal(hash160(input.PubKey), e.Out.PubKeyHash) {
					return fmt.Errorf("tx %d: %w", i, ErrKeyMismatch)
				}
				sh := t.SigHash(k, e.Out.PubKeyHash)
				if len(input.Signature) != 65 || !crypto.VerifySignature(input.PubKey, sh[:], input.Signature[:64]) {
					return fmt.Errorf("tx %d: %w", i, ErrBadSig)
				}
				spentBy[op] = i
				in += e.Out.Value

				undo = append(undo, key...) // what lesson 14 needs to put it back
				undo = binary.BigEndian.AppendUint32(undo, uint32(len(v)))
				undo = append(undo, v...) // append copies out of the mmap
				if err := utxo.Delete(key); err != nil {
					return err
				}
				if err := byaddr.Delete(append(bytes.Clone(e.Out.PubKeyHash), key...)); err != nil {
					return err
				}
			}
		}
		var out int64
		for _, o := range t.Outputs {
			if o.Value < 0 || o.Value > MaxMoney {
				return fmt.Errorf("tx %d: value out of range", i)
			}
			out += o.Value
		}
		if !t.IsCoinbase() {
			if out > in {
				return fmt.Errorf("tx %d: %w", i, ErrNotCovered)
			}
			fees += in - out
		}
		id := t.TxID()
		for k, o := range t.Outputs {
			key := utxoKey(Outpoint{id, uint32(k)})
			if err := utxo.Put(key, encodeEntry(UTXOEntry{o, height, t.IsCoinbase()})); err != nil {
				return err
			}
			if err := byaddr.Put(append(bytes.Clone(o.PubKeyHash), key...), []byte{}); err != nil {
				return err
			}
		}
	}

	// Only now is every fee known — and by now this transaction has deleted
	// and inserted every entry the block touches. An error here still undoes
	// all of it.
	var claimed int64
	for _, o := range b.Txs[0].Outputs {
		claimed += o.Value
	}
	if allowed := Subsidy(height) + fees; claimed > allowed {
		return fmt.Errorf("%w: %s > %s", ErrOverClaim, btc(claimed), btc(allowed))
	}

	hash := b.Header.Hash()
	if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
		return err
	}
	if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
		return err
	}
	if err := tx.Bucket(bktUndo).Put(hash[:], undo); err != nil {
		return err
	}
	if err := meta.Put(keyUTXOTip, hash[:]); err != nil {
		return err
	}
	return meta.Put(keyTip, hash[:])
}

func (c *Chain) Tip() (h Header) {
	check(c.db.View(func(tx *bolt.Tx) error {
		rec := tx.Bucket(bktBlocks).Get(tx.Bucket(bktMeta).Get(keyTip))
		if len(rec) < 1+HeaderSize {
			return ErrBadRecord
		}
		h = decodeHeader(&reader{p: rec[1:]})
		return nil
	}))
	return h
}

func (c *Chain) Height() int64 { return int64(c.Tip().Height) }

// Entry looks one output up. The result is a copy: nothing escapes the
// read transaction (example 7).
func (c *Chain) Entry(op Outpoint) (e UTXOEntry, ok bool) {
	check(c.db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktUTXO).Get(utxoKey(op))
		if v == nil {
			return nil
		}
		var err error
		e, err = decodeEntry(v)
		ok = err == nil
		return err
	}))
	return e, ok
}

// Owned is one unspent output a wallet can see: where it is, and what it holds.
type Owned struct {
	Op    Outpoint
	Entry UTXOEntry
}

// CoinsOf is a prefix scan of the address index (example 8).
func (c *Chain) CoinsOf(pkh []byte) (coins []Owned) {
	check(c.db.View(func(tx *bolt.Tx) error {
		utxo := tx.Bucket(bktUTXO)
		cur := tx.Bucket(bktAddr).Cursor()
		for k, _ := cur.Seek(pkh); k != nil && bytes.HasPrefix(k, pkh); k, _ = cur.Next() {
			key := k[len(pkh):]
			e, err := decodeEntry(utxo.Get(key))
			if err != nil {
				return err
			}
			coins = append(coins, Owned{Outpoint{[32]byte(key[:32]), binary.BigEndian.Uint32(key[32:])}, e})
		}
		return nil
	}))
	return coins
}

func (c *Chain) Balance(pkh []byte) (n int64) {
	for _, x := range c.CoinsOf(pkh) {
		n += x.Entry.Out.Value
	}
	return n
}

func (c *Chain) Fingerprint() (fp [32]byte) {
	check(c.db.View(func(tx *bolt.Tx) error {
		h := sha256.New()
		err := tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
			h.Write(k)
			h.Write(v)
			return nil
		})
		fp = [32]byte(h.Sum(nil))
		return err
	}))
	return fp
}

// Replay rebuilds the UTXO set from the stored blocks, in memory, and says
// whether it matches the stored set (example 15's oracle).
func (c *Chain) Replay() (match bool, blocks int, err error) {
	err = c.db.View(func(tx *bolt.Tx) error {
		set := map[string][]byte{}
		cur := tx.Bucket(bktHeights).Cursor()
		for k, hash := cur.First(); k != nil; k, hash = cur.Next() {
			b, err := decodeBlock(tx.Bucket(bktBlocks).Get(hash))
			if err != nil {
				return err
			}
			blocks++
			for _, t := range b.Txs {
				if !t.IsCoinbase() {
					for _, in := range t.Inputs {
						delete(set, string(utxoKey(in.Prev)))
					}
				}
				id := t.TxID()
				for vout, o := range t.Outputs {
					set[string(utxoKey(Outpoint{id, uint32(vout)}))] =
						encodeEntry(UTXOEntry{o, int64(b.Header.Height), t.IsCoinbase()})
				}
			}
		}
		keys := make([]string, 0, len(set))
		for k := range set {
			keys = append(keys, k)
		}
		sort.Strings(keys)
		h := sha256.New()
		for _, k := range keys {
			h.Write([]byte(k))
			h.Write(set[k])
		}
		stored := sha256.New()
		_ = tx.Bucket(bktUTXO).ForEach(func(k, v []byte) error {
			stored.Write(k)
			stored.Write(v)
			return nil
		})
		match = bytes.Equal(h.Sum(nil), stored.Sum(nil))
		return nil
	})
	return match, blocks, err
}

func (c *Chain) UndoStats() (blocks, spent int) {
	check(c.db.View(func(tx *bolt.Tx) error {
		return tx.Bucket(bktUndo).ForEach(func(_, v []byte) error {
			blocks++
			for len(v) > 0 {
				v = v[40+int(binary.BigEndian.Uint32(v[36:40])):]
				spent++
			}
			return nil
		})
	}))
	return blocks, spent
}

// ------------------------------------------------------------- mempool (11)

type PoolEntry struct {
	Tx   *Transaction
	Name string
	Size int64
	Fee  int64
}

func (e *PoolEntry) Rate() int64 { return e.Fee / e.Size }

type Mempool struct {
	byID    map[[32]byte]*PoolEntry
	claimed map[Outpoint][32]byte
}

func NewMempool() *Mempool {
	return &Mempool{byID: map[[32]byte]*PoolEntry{}, claimed: map[Outpoint][32]byte{}}
}

var (
	ErrPoolDup      = errors.New("already in the pool")
	ErrPoolOrphan   = errors.New("input not in the UTXO set")
	ErrPoolConflict = errors.New("conflicts with a pool transaction")
	ErrPoolCheap    = errors.New("below the relay fee rate")
)

// Accept is lesson 11's admission with replacement left out: a conflict is
// simply refused.
func (m *Mempool) Accept(c *Chain, name string, t *Transaction) error {
	id := t.TxID()
	if _, ok := m.byID[id]; ok {
		return ErrPoolDup
	}
	prevOuts := make([]TxOutput, 0, len(t.Inputs))
	for _, in := range t.Inputs {
		if _, taken := m.claimed[in.Prev]; taken {
			return ErrPoolConflict
		}
		e, ok := c.Entry(in.Prev) // NEW: a read transaction against the file
		if !ok {
			return fmt.Errorf("%w: %x…:%d", ErrPoolOrphan, in.Prev.TxID[:6], in.Prev.Index)
		}
		prevOuts = append(prevOuts, e.Out)
	}
	var in, out int64
	for _, p := range prevOuts {
		in += p.Value
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if out > in {
		return ErrNotCovered
	}
	size, fee := t.Size(), in-out
	if fee/size < MinRelayRate {
		return ErrPoolCheap
	}
	if err := verify(t, prevOuts); err != nil {
		return err
	}
	m.byID[id] = &PoolEntry{t, name, size, fee}
	for _, i := range t.Inputs {
		m.claimed[i.Prev] = id
	}
	return nil
}

func (m *Mempool) entries() []*PoolEntry {
	out := make([]*PoolEntry, 0, len(m.byID))
	for _, e := range m.byID {
		out = append(out, e)
	}
	sort.Slice(out, func(i, j int) bool {
		if a, b := out[i].Fee*out[j].Size, out[j].Fee*out[i].Size; a != b {
			return a > b
		}
		return out[i].Name < out[j].Name
	})
	return out
}

// Template: best fee rate first, skipping what does not fit.
func (m *Mempool) Template() (body []*Transaction, used, fees int64) {
	for _, e := range m.entries() {
		if used+e.Size > MaxBlockSize {
			continue
		}
		body = append(body, e.Tx)
		used += e.Size
		fees += e.Fee
	}
	return body, used, fees
}

func (m *Mempool) drop(id [32]byte) {
	if e, ok := m.byID[id]; ok {
		for _, in := range e.Tx.Inputs {
			delete(m.claimed, in.Prev)
		}
		delete(m.byID, id)
	}
}

func (m *Mempool) Confirmed(b Block) {
	for _, t := range b.Txs {
		m.drop(t.TxID())
		for _, in := range t.Inputs {
			if id, ok := m.claimed[in.Prev]; ok {
				m.drop(id)
			}
		}
	}
}

func verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: %w", i, ErrKeyMismatch)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: %w", i, ErrBadSig)
		}
	}
	return nil
}

// -------------------------------------------------------------- wallet (11)

type Sent struct {
	Name string
	Tx   *Transaction
}

type Wallet struct {
	Name    string
	priv    *ecdsa.PrivateKey
	pending []Sent // NEW: broadcast, not yet seen in a block
}

func NewWallet(name, hexKey string) *Wallet {
	k, err := crypto.HexToECDSA(hexKey)
	if err != nil {
		panic(err)
	}
	return &Wallet{Name: name, priv: k}
}

func (w *Wallet) PKH() []byte { return hash160(crypto.CompressPubkey(&w.priv.PublicKey)) }

const (
	overheadSize = 8
	inputSize    = 146
	outputSize   = 32
	dustLimit    = 294
)

func estimate(nIn, nOut int) int64 { return int64(overheadSize + nIn*inputSize + nOut*outputSize) }

var ErrInsufficient = errors.New("insufficient funds at this fee rate")

// Pay is lesson 11's: effective-value selection, fee from a rate, change, and
// signing last. The coins now come from a prefix scan of the address index.
func (w *Wallet) Pay(c *Chain, m *Mempool, to []byte, amount, rate int64) (*Transaction, error) {
	height := c.Height() + 1
	var coins []Owned
	for _, x := range c.CoinsOf(w.PKH()) {
		if _, claimed := m.claimed[x.Op]; claimed {
			continue
		}
		if x.Entry.Coinbase && height-x.Entry.Height < CoinbaseMaturity {
			continue
		}
		coins = append(coins, x)
	}
	sort.Slice(coins, func(i, j int) bool {
		if coins[i].Entry.Out.Value != coins[j].Entry.Out.Value {
			return coins[i].Entry.Out.Value > coins[j].Entry.Out.Value
		}
		return bytes.Compare(coins[i].Op.TxID[:], coins[j].Op.TxID[:]) < 0
	})

	target := amount + rate*(overheadSize+outputSize)
	var chosen []Owned
	var sumEff, total int64
	for _, x := range coins {
		ev := x.Entry.Out.Value - rate*inputSize
		if ev <= 0 {
			continue
		}
		chosen = append(chosen, x)
		sumEff += ev
		total += x.Entry.Out.Value
		if sumEff >= target {
			break
		}
	}
	if sumEff < target {
		return nil, ErrInsufficient
	}

	fee := estimate(len(chosen), 2) * rate
	outs := []TxOutput{{Value: amount, PubKeyHash: to}}
	if change := total - amount - fee; change >= dustLimit {
		outs = append(outs, TxOutput{Value: change, PubKeyHash: w.PKH()})
	}
	t := &Transaction{Outputs: outs}
	for _, x := range chosen {
		t.Inputs = append(t.Inputs, TxInput{Prev: x.Op, Sequence: 0xfffffffd})
	}
	pub := crypto.CompressPubkey(&w.priv.PublicKey)
	for i, x := range chosen {
		h := t.SigHash(i, x.Entry.Out.PubKeyHash)
		sig, err := crypto.Sign(h[:], w.priv)
		if err != nil {
			return nil, err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return t, nil
}

// Rebroadcast offers every pending transaction to a pool — typically a new,
// empty one after the node restarted. A transaction whose inputs are gone
// from the UTXO set was confirmed (or beaten) and is forgotten.
func (w *Wallet) Rebroadcast(c *Chain, m *Mempool) (accepted, forgotten int) {
	var still []Sent
	for _, s := range w.pending {
		live := true
		for _, in := range s.Tx.Inputs {
			if _, ok := c.Entry(in.Prev); !ok {
				live = false
			}
		}
		if !live {
			forgotten++
			continue
		}
		if err := m.Accept(c, s.Name, s.Tx); err == nil || errors.Is(err, ErrPoolDup) {
			accepted++
		}
		still = append(still, s)
	}
	w.pending = still
	return accepted, forgotten
}

// ------------------------------------------------------------------- output

func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func short(h [32]byte) string { return hex.EncodeToString(h[:6]) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	// The published anvil / Hardhat development keys. TEST ONLY: they are in
	// every tutorial, so anyone can spend from them on any chain.
	miner := NewWallet("miner", "7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6")
	alice := NewWallet("alice", "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	bob := NewWallet("bob", "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d")
	carol := NewWallet("carol", "5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a")
	wallets := []*Wallet{miner, alice, bob, carol}

	dir, err := os.MkdirTemp("", "chain-*")
	check(err)
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "chain.db")

	// build assembles and mines the next block; extra is added to what the
	// coinbase claims, so a block can be made to fail its last check.
	build := func(c *Chain, pool *Mempool, extra int64) Block {
		body, _, fees := pool.Template()
		height := c.Height() + 1
		txs := append([]*Transaction{NewCoinbase(height, miner.PKH(), Subsidy(height)+fees+extra)}, body...)
		p := c.Tip()
		return Block{Mine(Header{Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot(txs),
			Timestamp: p.Timestamp + 60, Bits: 0x2000ffff, Height: uint64(height)}), txs}
	}
	mine := func(c *Chain, pool *Mempool) {
		b := build(c, pool, 0)
		check(c.Append(b))
		pool.Confirmed(b)
		var used int64
		for _, t := range b.Txs[1:] {
			used += t.Size()
		}
		fees := b.Txs[0].Outputs[0].Value - Subsidy(int64(b.Header.Height))
		fmt.Printf("  block %-3d %s  %d tx, %d/%d bytes, fees %s\n",
			b.Header.Height, short(b.Header.Hash()), len(b.Txs)-1, used, MaxBlockSize, btc(fees))
	}
	pay := func(c *Chain, pool *Mempool, from, to *Wallet, amount, rate int64, name string, quiet bool) {
		t, err := from.Pay(c, pool, to.PKH(), amount, rate)
		if err == nil {
			err = pool.Accept(c, name, t)
		}
		if err != nil {
			fmt.Printf("  %-12s REJECTED: %v\n", name, err)
			return
		}
		from.pending = append(from.pending, Sent{name, t})
		if !quiet {
			fmt.Printf("  %-12s %d bytes at %2d sat/byte\n", name, t.Size(), pool.byID[t.TxID()].Rate())
		}
	}
	balances := func(c *Chain) {
		for _, w := range wallets {
			fmt.Printf("    %-6s %s\n", w.Name, btc(c.Balance(w.PKH())))
		}
	}

	// ------------------------------------------------------------- run one
	fmt.Println("=== first start: no file yet ===")
	c, err := OpenChain(path, miner.PKH())
	check(err)
	pool := NewMempool()
	fmt.Printf("  created %s, genesis %s\n", filepath.Base(path), short(c.Tip().Hash()))

	fmt.Println("\n=== six empty blocks so a coinbase matures, then fund three wallets ===")
	for i := 0; i < 6; i++ {
		mine(c, pool)
	}
	for round := 1; round <= 2; round++ {
		for _, w := range []*Wallet{alice, bob, carol} {
			pay(c, pool, miner, w, 20*Coin, 20, fmt.Sprintf("fund-%s-%d", w.Name, round), true)
		}
		mine(c, pool)
	}

	fmt.Printf("\n=== four payments, room for three ===\n")
	pay(c, pool, alice, bob, 1*Coin, 40, "alice-fast", false)
	pay(c, pool, bob, carol, 2*Coin, 30, "bob-normal", false)
	pay(c, pool, carol, alice, 1*Coin, 25, "carol-quick", false)
	pay(c, pool, bob, alice, 1*Coin, 2, "bob-cheap", false)
	mine(c, pool)
	fmt.Printf("  still waiting: %d transaction in the pool\n", len(pool.byID))

	fmt.Println("\n=== the node shuts down ===")
	tipBefore, fpBefore := c.Tip(), c.Fingerprint()
	check(c.Close())
	fmt.Printf("  closed at height %d. The pool lived only in memory, and is gone.\n", tipBefore.Height)

	// ------------------------------------------------------------- run two
	fmt.Println("\n=== ...and starts again on the same file ===")
	c, err = OpenChain(path, miner.PKH())
	check(err)
	pool = NewMempool()
	tip := c.Tip()
	fmt.Printf("  opened at height %d %s — schema v%d, tip == utxo-tip\n", tip.Height, short(tip.Hash()), SchemaVersion)
	fmt.Printf("  same tip as before: %v · same UTXO set: %v\n", tip == tipBefore, c.Fingerprint() == fpBefore)
	balances(c)
	fmt.Printf("  the new pool holds %d transactions\n", len(pool.byID))
	for _, w := range wallets {
		if accepted, forgotten := w.Rebroadcast(c, pool); accepted+forgotten > 0 {
			fmt.Printf("  %-6s re-broadcast %d, forgot %d already confirmed\n", w.Name, accepted, forgotten)
		}
	}
	mine(c, pool)

	fmt.Println("\n=== a block that fails its LAST check ===")
	pay(c, pool, alice, carol, 3*Coin, 10, "alice-late", false)
	pay(c, pool, carol, bob, 1*Coin, 10, "carol-late", false)
	tipBefore, fpBefore = c.Tip(), c.Fingerprint()
	bad := build(c, pool, 1) // the coinbase claims one satoshi too many
	var spent, created int
	for _, t := range bad.Txs[1:] {
		spent += len(t.Inputs)
	}
	for _, t := range bad.Txs {
		created += len(t.Outputs)
	}
	fmt.Printf("  Append: %v\n", c.Append(bad))
	fmt.Printf("  when that check ran, the transaction had deleted %d entries and inserted %d\n", spent, created)
	fmt.Printf("  tip unchanged: %v · UTXO set unchanged: %v\n", c.Tip() == tipBefore, c.Fingerprint() == fpBefore)
	mine(c, pool)

	// --------------------------------------------------------------- final
	fmt.Println("\n=== final state ===")
	balances(c)
	var total, issued int64
	for _, w := range wallets {
		total += c.Balance(w.PKH())
	}
	for h := int64(0); h <= c.Height(); h++ {
		issued += Subsidy(h)
	}
	fmt.Printf("  unspent %s == issued %s: %v\n", btc(total), btc(issued), total == issued)
	match, blocks, err := c.Replay()
	check(err)
	fmt.Printf("  stored UTXO set == replay of the %d stored blocks: %v\n", blocks, match)
	ub, us := c.UndoStats()
	fmt.Printf("  undo records for %d blocks, %d spent entries — waiting for lesson 14\n", ub, us)
	check(c.Close())

	fmt.Println("\n--- the diff from lesson 11 ---")
	fmt.Println("  + Chain holds a *bolt.DB instead of []Block and a UTXOSet map")
	fmt.Println("  + Append is one db.Update; the Delta and its apply step are gone,")
	fmt.Println("    because the write transaction is the delta")
	fmt.Println("  + every record carries a version byte; the file carries a schema")
	fmt.Println("  + an address index, so a wallet's coins are a prefix scan")
	fmt.Println("  + undo records, written in the same transaction as the block")
	fmt.Println("  + OpenChain's checks, and Replay as a test oracle")

	fmt.Println("\n--- what lesson 13 changes ---")
	fmt.Println("  Every block here was mined by the process that stored it. Lesson 13")
	fmt.Println("  adds peers: blocks arrive from strangers, before their parents, twice,")
	fmt.Println("  or never — and the store's PutBlock-without-connecting and its")
	fmt.Println("  refusal of anything off the tip are what the network layer leans on.")
}
```

**Output:**

```
=== first start: no file yet ===
  created chain.db, genesis 00b293508253

=== six empty blocks so a coinbase matures, then fund three wallets ===
  block 1   004416ff3431  0 tx, 0/700 bytes, fees 0.00000000
  block 2   005f3963382b  0 tx, 0/700 bytes, fees 0.00000000
  block 3   0038d982bc41  0 tx, 0/700 bytes, fees 0.00000000
  block 4   009e3908300e  0 tx, 0/700 bytes, fees 0.00000000
  block 5   000c63fa1b5f  0 tx, 0/700 bytes, fees 0.00000000
  block 6   009f1e42650e  0 tx, 0/700 bytes, fees 0.00000000
  block 7   003c03b4dbd4  3 tx, 654/700 bytes, fees 0.00013080
  block 8   003740b8030e  3 tx, 654/700 bytes, fees 0.00013080

=== four payments, room for three ===
  alice-fast   218 bytes at 40 sat/byte
  bob-normal   218 bytes at 30 sat/byte
  carol-quick  218 bytes at 25 sat/byte
  bob-cheap    218 bytes at  2 sat/byte
  block 9   005bd8edcb6d  3 tx, 654/700 bytes, fees 0.00020710
  still waiting: 1 transaction in the pool

=== the node shuts down ===
  closed at height 9. The pool lived only in memory, and is gone.

=== ...and starts again on the same file ===
  opened at height 9 005bd8edcb6d — schema v1, tip == utxo-tip
  same tip as before: true · same UTXO set: true
    miner  380.00020710
    alice  39.99991280
    bob    38.99993460
    carol  40.99994550
  the new pool holds 0 transactions
  miner  re-broadcast 0, forgot 6 already confirmed
  alice  re-broadcast 0, forgot 1 already confirmed
  bob    re-broadcast 1, forgot 1 already confirmed
  carol  re-broadcast 0, forgot 1 already confirmed
  block 10  00b000810b05  1 tx, 218/700 bytes, fees 0.00000436

=== a block that fails its LAST check ===
  alice-late   218 bytes at 10 sat/byte
  carol-late   218 bytes at 10 sat/byte
  Append: coinbase claims more than subsidy plus fees: 50.00004361 > 50.00004360
  when that check ran, the transaction had deleted 2 entries and inserted 5
  tip unchanged: true · UTXO set unchanged: true
  block 11  00eb4cb41657  2 tx, 436/700 bytes, fees 0.00004360

=== final state ===
    miner  480.00025506
    alice  37.99989100
    bob    38.99993024
    carol  42.99992370
  unspent 600.00000000 == issued 600.00000000: true
  stored UTXO set == replay of the 12 stored blocks: true
  undo records for 12 blocks, 12 spent entries — waiting for lesson 14

--- the diff from lesson 11 ---
  + Chain holds a *bolt.DB instead of []Block and a UTXOSet map
  + Append is one db.Update; the Delta and its apply step are gone,
    because the write transaction is the delta
  + every record carries a version byte; the file carries a schema
  + an address index, so a wallet's coins are a prefix scan
  + undo records, written in the same transaction as the block
  + OpenChain's checks, and Replay as a test oracle

--- what lesson 13 changes ---
  Every block here was mined by the process that stored it. Lesson 13
  adds peers: blocks arrive from strangers, before their parents, twice,
  or never — and the store's PutBlock-without-connecting and its
  refusal of anything off the tip are what the network layer leans on.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
