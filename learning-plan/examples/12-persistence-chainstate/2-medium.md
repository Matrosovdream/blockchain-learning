# Step 12 — Persistence & Chain State · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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
exactly. Examples 9 and 12 use goroutines, and are built so the outcome they print does not
depend on which goroutine runs first.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. A chain on disk, walked from the tip

`🟡 medium` · *Iterators and cursors*

Five mined blocks with transactions go into example 3's schema; the process closes the file and a new one opens it, which is all a restart is. The chain is then read back two ways — following `PrevHash` from the tip, and walking a cursor over the height index — with every block checked as it is read. Then one bit rots on disk, and only the hash notices.

**Steps:**

1. Mine five blocks at a trivial difficulty and connect each one in its own write transaction.
2. Close the file, reopen it, and walk backwards from the tip by following `PrevHash`.
3. Walk forwards with a cursor over `heights`, checking that each block links to the one before.
4. `Seek` to height 2 and read the range 2..3.
5. Flip one bit in a stored block, and walk back from the tip again.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"math/big"
	"os"
	"path/filepath"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// A chain on disk, walked from the tip.
//
// Five mined blocks with transactions go into the schema from example 3. The
// process closes the file and a new one opens it — which is all a restart
// is — and then reads the chain back two ways:
//
//   backwards   meta tip -> block -> PrevHash -> block -> ... -> genesis
//   forwards    a cursor over `heights`, one block lookup per step
//
// Every block is CHECKED as it is read. A database stores bytes; only the
// hash can say whether they are still the bytes that were written.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key               value
//   -------   ---------------   -------------------------------------------
//   blocks    hash      [32]    version byte + header [92] + transactions
//   heights   height [8] BE     hash of the canonical block at that height
//   meta      "tip"             hash of the block the state reflects
// ---------------------------------------------------------------------------

const (
	HeaderSize    = 92
	BlockRecordV1 = 0x01
	Coin          = int64(100_000_000)
	Bits          = 0x2000ffff // about 256 hashes a block: real proof of work, instantly
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
)

var (
	ErrNotFound     = errors.New("not found")
	ErrNotOnTip     = errors.New("parent is not the current tip")
	ErrHashMismatch = errors.New("header does not hash to the key it is stored under")
	ErrBadMerkle    = errors.New("transactions do not match the merkle root")
	ErrBadPoW       = errors.New("insufficient proof of work")
	ErrBrokenLink   = errors.New("PrevHash does not match the block below")
	ErrVersion      = errors.New("unknown record version")
	ErrTruncated    = errors.New("record truncated")
	ErrTrailing     = errors.New("trailing bytes after record")
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

func BitsToTarget(bits uint32) *big.Int {
	exp, m := bits>>24, bits&0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

func CheckPoW(h Header) bool {
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(BitsToTarget(h.Bits)) < 0
}

// Mine is lesson 09's loop without the allocation-free tricks.
func Mine(h Header) Header {
	for !CheckPoW(h) {
		h.Nonce++
	}
	return h
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

func (t *Transaction) Serialize() []byte {
	var b []byte
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = append(b, in.Prev.TxID[:]...)
		b = binary.BigEndian.AppendUint32(b, in.Prev.Index)
		b = binary.BigEndian.AppendUint32(b, in.Sequence)
		b = appendBytes(b, in.Signature)
		b = appendBytes(b, in.PubKey)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = appendBytes(b, o.PubKeyHash)
	}
	return b
}

func appendBytes(b, p []byte) []byte {
	b = binary.BigEndian.AppendUint32(b, uint32(len(p)))
	return append(b, p...)
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func NewCoinbase(height uint64, to []byte, value int64) *Transaction {
	return &Transaction{
		Inputs: []TxInput{{Prev: nullOutpoint, Sequence: 0xffffffff,
			Signature: binary.BigEndian.AppendUint64(nil, height)}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
}

const tagLeaf, tagNode byte = 0x00, 0x01

func MerkleRoot(txs []*Transaction) [32]byte {
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

// ----------------------------------------------------- the record (ex. 1)

type Block struct {
	Header Header
	Txs    []*Transaction
}

func (b *Block) Encode() []byte {
	out := []byte{BlockRecordV1}
	out = append(out, b.Header.Bytes()...)
	out = binary.BigEndian.AppendUint32(out, uint32(len(b.Txs)))
	for _, t := range b.Txs {
		out = appendBytes(out, t.Serialize())
	}
	return out
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

func (r *reader) u8() byte {
	if v := r.take(1); v != nil {
		return v[0]
	}
	return 0
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

func (r *reader) bytes() []byte { return bytes.Clone(r.take(int(r.u32()))) }

func decodeTx(p []byte) (*Transaction, error) {
	r := &reader{p: p}
	t := &Transaction{}
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		var in TxInput
		copy(in.Prev.TxID[:], r.take(32))
		in.Prev.Index = r.u32()
		in.Sequence = r.u32()
		in.Signature = r.bytes()
		in.PubKey = r.bytes()
		t.Inputs = append(t.Inputs, in)
	}
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		var o TxOutput
		o.Value = int64(r.u64())
		o.PubKeyHash = r.bytes()
		t.Outputs = append(t.Outputs, o)
	}
	if r.err == nil && len(r.p) != 0 {
		r.err = ErrTrailing
	}
	return t, r.err
}

func DecodeBlock(p []byte) (*Block, error) {
	r := &reader{p: p}
	if v := r.u8(); r.err == nil && v != BlockRecordV1 {
		return nil, fmt.Errorf("%w: 0x%02x", ErrVersion, v)
	}
	var h Header
	h.Version = r.u32()
	copy(h.PrevHash[:], r.take(32))
	copy(h.MerkleRoot[:], r.take(32))
	h.Timestamp = int64(r.u64())
	h.Bits = r.u32()
	h.Nonce = r.u32()
	h.Height = r.u64()
	b := &Block{Header: h}
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		raw := r.take(int(r.u32()))
		if r.err != nil {
			break
		}
		t, err := decodeTx(raw)
		if err != nil {
			return nil, err
		}
		b.Txs = append(b.Txs, t)
	}
	if r.err == nil && len(r.p) != 0 {
		r.err = ErrTrailing
	}
	if r.err != nil {
		return nil, r.err
	}
	return b, nil
}

// ---------------------------------------------------------------- the store

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func initSchema(db *bolt.DB) error {
	return db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		return nil
	})
}

func connect(db *bolt.DB, b *Block) error {
	hash := b.Header.Hash()
	return db.Update(func(tx *bolt.Tx) error {
		meta := tx.Bucket(bktMeta)
		if tip := meta.Get(keyTip); b.Header.Height > 0 && !bytes.Equal(tip, b.Header.PrevHash[:]) {
			return ErrNotOnTip
		}
		if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
			return err
		}
		if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
			return err
		}
		return meta.Put(keyTip, hash[:])
	})
}

// load reads a block and CHECKS it. Bytes from disk are just bytes until they
// hash to the key they were filed under.
func load(tx *bolt.Tx, hash [32]byte) (*Block, error) {
	v := tx.Bucket(bktBlocks).Get(hash[:])
	if v == nil {
		return nil, fmt.Errorf("%w: block %x…", ErrNotFound, hash[:4])
	}
	b, err := DecodeBlock(v)
	if err != nil {
		return nil, err
	}
	if b.Header.Hash() != hash {
		return nil, ErrHashMismatch
	}
	if MerkleRoot(b.Txs) != b.Header.MerkleRoot {
		return nil, ErrBadMerkle
	}
	if !CheckPoW(b.Header) {
		return nil, ErrBadPoW
	}
	return b, nil
}

// WalkBack follows PrevHash from the tip to genesis. It needs the blocks and
// one pointer — no index — which is why it still works on a branch that no
// index describes (lesson 14).
func WalkBack(db *bolt.DB, visit func(*Block)) (lookups int, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		tip := tx.Bucket(bktMeta).Get(keyTip)
		lookups++
		if tip == nil {
			return fmt.Errorf("%w: no tip", ErrNotFound)
		}
		var hash [32]byte
		copy(hash[:], tip)
		for {
			b, err := load(tx, hash)
			lookups++
			if err != nil {
				return fmt.Errorf("block %x…: %w", hash[:4], err)
			}
			visit(b)
			if b.Header.PrevHash == [32]byte{} {
				return nil
			}
			hash = b.Header.PrevHash
		}
	})
	return lookups, err
}

// WalkForward seeks the height index and walks the cursor, checking that
// each block links to the one before it.
func WalkForward(db *bolt.DB, from, to uint64, visit func(*Block)) (lookups int, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		c := tx.Bucket(bktHeights).Cursor()
		var prev [32]byte
		for k, v := c.Seek(be64(from)); k != nil && binary.BigEndian.Uint64(k) <= to; k, v = c.Next() {
			var hash [32]byte
			copy(hash[:], v)
			b, err := load(tx, hash)
			lookups += 2 // one cursor step, one block lookup
			if err != nil {
				return fmt.Errorf("height %d: %w", binary.BigEndian.Uint64(k), err)
			}
			if prev != ([32]byte{}) && b.Header.PrevHash != prev {
				return fmt.Errorf("height %d: %w", b.Header.Height, ErrBrokenLink)
			}
			prev = hash
			visit(b)
		}
		return nil
	})
	return lookups, err
}

// ------------------------------------------------------------------ helpers

func addr(name string) []byte {
	s := sha256.Sum256([]byte(name))
	return s[:20] // stands in for HASH160(pubkey)
}

func pay(from Outpoint, value int64, to, change []byte, amount int64) *Transaction {
	return &Transaction{
		// Signatures elided: storage never verifies them (lesson 10 does).
		Inputs: []TxInput{{Prev: from, Sequence: 0xffffffff}},
		Outputs: []TxOutput{
			{Value: amount, PubKeyHash: to},
			{Value: value - amount, PubKeyHash: change},
		},
	}
}

func short(h [32]byte) string { return fmt.Sprintf("%x", h[:5]) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "walk-*")
	check(err)
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "chain.db")
	opts := &bolt.Options{Timeout: time.Second}

	miner, alice, bob := addr("miner"), addr("alice"), addr("bob")

	// Build five blocks: coinbases, and two payments along the way.
	var chain []*Block
	add := func(extra ...*Transaction) {
		h := Header{Version: 1, Timestamp: 1700000000, Bits: Bits}
		if n := len(chain); n > 0 {
			p := chain[n-1].Header
			h.PrevHash, h.Height, h.Timestamp = p.Hash(), p.Height+1, p.Timestamp+600
		}
		txs := append([]*Transaction{NewCoinbase(h.Height, miner, 50*Coin)}, extra...)
		h.MerkleRoot = MerkleRoot(txs)
		chain = append(chain, &Block{Header: Mine(h), Txs: txs})
	}
	add()
	add()
	add(pay(Outpoint{chain[0].Txs[0].TxID(), 0}, 50*Coin, alice, miner, 30*Coin))
	add(pay(Outpoint{chain[2].Txs[1].TxID(), 0}, 30*Coin, bob, alice, 12*Coin))
	add()

	fmt.Println("=== write five blocks, one transaction each, then close ===")
	db, err := bolt.Open(path, 0o600, opts)
	check(err)
	check(initSchema(db))
	for _, b := range chain {
		check(connect(db, b))
		fmt.Printf("  connected height %d  %s  %d tx  nonce %d\n",
			b.Header.Height, short(b.Header.Hash()), len(b.Txs), b.Header.Nonce)
	}
	check(db.Close())

	fmt.Println("\n=== a new process opens the file: backwards from the tip ===")
	db, err = bolt.Open(path, 0o600, opts)
	check(err)
	defer db.Close()
	n, err := WalkBack(db, func(b *Block) {
		fmt.Printf("  height %d  %s  <- prev %s\n", b.Header.Height, short(b.Header.Hash()), short(b.Header.PrevHash))
	})
	check(err)
	fmt.Printf("  %d lookups: the tip pointer, then one per block\n", n)

	fmt.Println("\n=== forwards, with a cursor over the height index ===")
	n, err = WalkForward(db, 0, ^uint64(0), func(b *Block) {
		fmt.Printf("  height %d  %s  %d tx\n", b.Header.Height, short(b.Header.Hash()), len(b.Txs))
	})
	check(err)
	fmt.Printf("  %d lookups\n", n)

	fmt.Println("\n=== a range: heights 2..3 ===")
	_, err = WalkForward(db, 2, 3, func(b *Block) {
		fmt.Printf("  height %d  %s  pays %x… %d.%08d\n", b.Header.Height, short(b.Header.Hash()),
			b.Txs[1].Outputs[0].PubKeyHash[:3], b.Txs[1].Outputs[0].Value/Coin, b.Txs[1].Outputs[0].Value%Coin)
	})
	check(err)
	fmt.Println("  Seek lands on the first key >= 2 and Next walks in key order —")
	fmt.Println("  correct only because the keys are big-endian (example 2).")

	fmt.Println("\n=== one bit rots in block 2 ===")
	target := chain[2].Header.Hash()
	check(db.Update(func(tx *bolt.Tx) error {
		bk := tx.Bucket(bktBlocks)
		v := bytes.Clone(bk.Get(target[:])) // never write into memory Get returned
		v[len(v)-1] ^= 0x01                 // the last byte of the last output's pkh
		return bk.Put(target[:], v)
	}))
	fmt.Println("  (flipped one bit in the stored record; the key is unchanged)")
	_, err = WalkBack(db, func(b *Block) {
		fmt.Printf("  height %d  ok\n", b.Header.Height)
	})
	fmt.Printf("  WalkBack: %v\n", err)
	fmt.Printf("  errors.Is(err, ErrBadMerkle): %v\n", errors.Is(err, ErrBadMerkle))
	fmt.Println()
	fmt.Println("  The store handed back exactly what it was given, without complaint.")
	fmt.Println("  bbolt checksums nothing; neither do most file systems. The only")
	fmt.Println("  thing that knew those bytes were wrong was the hash — and only")
	fmt.Println("  because load() recomputed it instead of trusting the disk.")
	fmt.Println("  This is Bitcoin Core's 'Corrupted block database detected', and")
	fmt.Println("  its answer is example 15's: rebuild from what still verifies.")
}
```

**Output:**

```
=== write five blocks, one transaction each, then close ===
  connected height 0  005ea39aaf  1 tx  nonce 160
  connected height 1  0084f6346d  1 tx  nonce 29
  connected height 2  003d3fb354  2 tx  nonce 434
  connected height 3  004325a1c2  2 tx  nonce 774
  connected height 4  00139464c6  1 tx  nonce 385

=== a new process opens the file: backwards from the tip ===
  height 4  00139464c6  <- prev 004325a1c2
  height 3  004325a1c2  <- prev 003d3fb354
  height 2  003d3fb354  <- prev 0084f6346d
  height 1  0084f6346d  <- prev 005ea39aaf
  height 0  005ea39aaf  <- prev 0000000000
  6 lookups: the tip pointer, then one per block

=== forwards, with a cursor over the height index ===
  height 0  005ea39aaf  1 tx
  height 1  0084f6346d  1 tx
  height 2  003d3fb354  2 tx
  height 3  004325a1c2  2 tx
  height 4  00139464c6  1 tx
  10 lookups

=== a range: heights 2..3 ===
  height 2  003d3fb354  pays 2bd806… 30.00000000
  height 3  004325a1c2  pays 81b637… 12.00000000
  Seek lands on the first key >= 2 and Next walks in key order —
  correct only because the keys are big-endian (example 2).

=== one bit rots in block 2 ===
  (flipped one bit in the stored record; the key is unchanged)
  height 4  ok
  height 3  ok
  WalkBack: block 003d3fb3…: transactions do not match the merkle root
  errors.Is(err, ErrBadMerkle): true

  The store handed back exactly what it was given, without complaint.
  bbolt checksums nothing; neither do most file systems. The only
  thing that knew those bytes were wrong was the hash — and only
  because load() recomputed it instead of trusting the disk.
  This is Bitcoin Core's 'Corrupted block database detected', and
  its answer is example 15's: rebuild from what still verifies.
```

---

## 7. The slice that outlived its transaction

`🟡 medium` · *Iterators and cursors*

Example 4 kept a slice past the end of its transaction and nothing went wrong, because nothing wrote. Here a wallet keeps one — for alice's coin — while the node carries on connecting blocks that never touch that coin. bbolt is copy-on-write: a transaction puts the new version of a page somewhere else and frees the old one, and a later transaction reuses it. Two blocks later, the entry the wallet kept says the coin belongs to mallory.

**Steps:**

1. Store nineteen coins in 4096-byte pages, alice's in the middle, with the memory map sized so bbolt never has to remap the file.
2. Look alice's coin up three ways — the raw slice, a decoder that aliases, a decoder that copies — and keep all three.
3. Connect four ordinary blocks, each spending someone else's coin and paying mallory, and reprint all three after each.
4. Read the coin fresh from the database, then ask the question the wallet kept the entry to answer.
5. Write into `Get`'s slice, and see what bbolt's read-only mapping does about it.

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
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// The slice that outlived its transaction.
//
// Example 4 showed that Get returns a window onto bbolt's memory map, and
// that keeping the window after the transaction ends LOOKS fine. This example
// keeps one — a wallet caching a coin it looked up — and then lets the node
// do ordinary work: a few blocks, each spending one coin and paying mallory.
//
// bbolt is copy-on-write. A transaction that changes a page writes the new
// version to a different page and frees the old one, and a later transaction
// reuses it. The cached window never moved. The page under it did.
// ===========================================================================

const (
	RecordV1 = 0x01
	Coin     = int64(100_000_000)
	PageSize = 4096 // pinned, so the page reuse below happens the same way everywhere
)

var (
	bktUTXO    = []byte("utxo")
	ErrMissing = errors.New("output not found")
)

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type UTXOEntry struct {
	Value      int64
	PubKeyHash []byte
}

// On disk: version [1] | value [8] | len [4] | pkh [len]
func encodeEntry(e UTXOEntry) []byte {
	b := []byte{RecordV1}
	b = binary.BigEndian.AppendUint64(b, uint64(e.Value))
	b = binary.BigEndian.AppendUint32(b, uint32(len(e.PubKeyHash)))
	return append(b, e.PubKeyHash...)
}

func key(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

// ------------------------------------------------ three ways to fetch a coin

// GetRaw hands back Get's slice itself. BUG.
func GetRaw(db *bolt.DB, op Outpoint) (v []byte, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		if v = tx.Bucket(bktUTXO).Get(key(op)); v == nil {
			return ErrMissing
		}
		return nil
	})
	return v, err
}

// GetAliased decodes — and slices PubKeyHash out of Get's slice. BUG, and the
// one that reads like correct code in review.
func GetAliased(db *bolt.DB, op Outpoint) (e UTXOEntry, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktUTXO).Get(key(op))
		if v == nil {
			return ErrMissing
		}
		e = UTXOEntry{Value: int64(binary.BigEndian.Uint64(v[1:9])), PubKeyHash: v[13:]}
		return nil
	})
	return e, err
}

// GetCopied is GetAliased plus bytes.Clone. The whole fix is one call, made
// at the storage boundary.
func GetCopied(db *bolt.DB, op Outpoint) (e UTXOEntry, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktUTXO).Get(key(op))
		if v == nil {
			return ErrMissing
		}
		e = UTXOEntry{Value: int64(binary.BigEndian.Uint64(v[1:9])), PubKeyHash: bytes.Clone(v[13:])}
		return nil
	})
	return e, err
}

// connectBlock is an ordinary block: spend one coin, create another.
func connectBlock(db *bolt.DB, spend, create Outpoint, e UTXOEntry) error {
	return db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket(bktUTXO)
		if b.Get(key(spend)) == nil {
			return ErrMissing
		}
		if err := b.Delete(key(spend)); err != nil {
			return err
		}
		return b.Put(key(create), encodeEntry(e))
	})
}

// ------------------------------------------------------------------ helpers

// txid makes up a transaction id whose first byte decides where it sorts.
func txid(first byte, label string) [32]byte {
	s := sha256.Sum256([]byte(label))
	s[0] = first
	return s
}

func who(pkh []byte) string {
	for name, b := range map[string]byte{"alice": 0xa1, "bob": 0xb0, "mallory": 0x66} {
		if bytes.Equal(pkh, bytes.Repeat([]byte{b}, 20)) {
			return name
		}
	}
	return fmt.Sprintf("%x…", pkh[:4])
}

func btc(sat int64) string { return fmt.Sprintf("%d.%08d", sat/Coin, sat%Coin) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "stale-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{
		Timeout:  time.Second,
		PageSize: PageSize,
		// Map a megabyte up front, so bbolt never has to REMAP the file while
		// this runs. A remap unmaps the old region, and reading a kept slice
		// after that is not a wrong answer but a SIGSEGV. Delete this line and
		// run the program a dozen times to watch some of the runs die.
		InitialMmapSize: 1 << 20,
	})
	check(err)
	defer db.Close()

	alice := bytes.Repeat([]byte{0xa1}, 20)
	bob := bytes.Repeat([]byte{0xb0}, 20)
	mallory := bytes.Repeat([]byte{0x66}, 20)

	// Nineteen coins: ten that sort before alice's, alice's, eight after.
	aliceCoin := Outpoint{txid(0x40, "alice's coin"), 0}
	var early []Outpoint
	check(db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucket(bktUTXO)
		if err != nil {
			return err
		}
		put := func(op Outpoint, e UTXOEntry) error { return b.Put(key(op), encodeEntry(e)) }
		for i := 0; i < 10; i++ {
			op := Outpoint{txid(0x20+byte(i), fmt.Sprintf("early-%d", i)), 0}
			early = append(early, op)
			if err := put(op, UTXOEntry{Value: int64(i+1) * Coin, PubKeyHash: bob}); err != nil {
				return err
			}
		}
		if err := put(aliceCoin, UTXOEntry{Value: 50 * Coin, PubKeyHash: alice}); err != nil {
			return err
		}
		for i := 0; i < 8; i++ {
			op := Outpoint{txid(0x60+byte(i), fmt.Sprintf("late-%d", i)), 0}
			if err := put(op, UTXOEntry{Value: int64(i+1) * Coin, PubKeyHash: bob}); err != nil {
				return err
			}
		}
		return nil
	}))

	fmt.Println("=== a wallet looks up alice's coin three ways, and keeps the results ===")
	raw, err := GetRaw(db, aliceCoin)
	check(err)
	aliased, err := GetAliased(db, aliceCoin)
	check(err)
	copied, err := GetCopied(db, aliceCoin)
	check(err)

	row := func(label string) {
		fmt.Printf("  %-15s %-22s %-22s %s\n", label,
			btc(int64(binary.BigEndian.Uint64(raw[1:9])))+" "+who(raw[13:33]),
			btc(aliased.Value)+" "+who(aliased.PubKeyHash),
			btc(copied.Value)+" "+who(copied.PubKeyHash))
	}
	fmt.Printf("  %-15s %-22s %-22s %s\n", "", "raw []byte", "entry, pkh aliased", "entry, pkh copied")
	row("just read")

	fmt.Println("\n=== blocks arrive; each spends someone else's coin and pays mallory ===")
	for h := 1; h <= 4; h++ {
		create := Outpoint{txid(0x40+byte(h), fmt.Sprintf("block-%d", h)), 0}
		check(connectBlock(db, early[h-1], create, UTXOEntry{Value: int64(h) * 1000, PubKeyHash: mallory}))
		row(fmt.Sprintf("after block %d", h))
	}

	fmt.Println("\n=== the database itself is fine ===")
	fresh, err := GetCopied(db, aliceCoin)
	check(err)
	fmt.Printf("  a fresh read of alice's coin: %s %s\n", btc(fresh.Value), who(fresh.PubKeyHash))
	fmt.Println("  No block touched alice's coin. Only the wallet's view of it changed.")

	fmt.Println("\n=== the check the wallet kept the entry for ===")
	fmt.Printf("  owner is mallory, per the aliased entry: %v\n", bytes.Equal(aliased.PubKeyHash, mallory))
	fmt.Printf("  owner is mallory, per the copied entry:  %v\n", bytes.Equal(copied.PubKeyHash, mallory))

	fmt.Println("\n=== and writing into Get's slice ===")
	old := debug.SetPanicOnFault(true) // turn the fault into a panic we can catch
	faulted := false
	check(db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktUTXO).Get(key(aliceCoin))
		defer func() { faulted = recover() != nil }()
		v[1] ^= 0xff // "just fix the value up in place"
		return nil
	}))
	debug.SetPanicOnFault(old)
	fmt.Printf("  the write faulted: %v\n", faulted)
	fmt.Println("  bbolt maps the file read-only. Without SetPanicOnFault that is not a")
	fmt.Println("  panic but a signal, and the process dies on the spot.")

	fmt.Println("\n=== why no tool catches it ===")
	fmt.Println("  go vet       : a slice is a slice; nothing to flag")
	fmt.Println("  go test -race: one goroutine, no data race")
	fmt.Println("  unit tests   : read, then check, with no write in between — passes")
	fmt.Println("  It needs a write, after the read, that happens to reuse that page.")
	fmt.Println("  In production that is every block. Copy at the boundary.")
}
```

**Output:**

```
=== a wallet looks up alice's coin three ways, and keeps the results ===
                  raw []byte             entry, pkh aliased     entry, pkh copied
  just read       50.00000000 alice      50.00000000 alice      50.00000000 alice

=== blocks arrive; each spends someone else's coin and pays mallory ===
  after block 1   50.00000000 alice      50.00000000 alice      50.00000000 alice
  after block 2   0.00002000 mallory     50.00000000 mallory    50.00000000 alice
  after block 3   0.00002000 mallory     50.00000000 mallory    50.00000000 alice
  after block 4   0.00004000 mallory     50.00000000 mallory    50.00000000 alice

=== the database itself is fine ===
  a fresh read of alice's coin: 50.00000000 alice
  No block touched alice's coin. Only the wallet's view of it changed.

=== the check the wallet kept the entry for ===
  owner is mallory, per the aliased entry: true
  owner is mallory, per the copied entry:  false

=== and writing into Get's slice ===
  the write faulted: true
  bbolt maps the file read-only. Without SetPanicOnFault that is not a
  panic but a signal, and the process dies on the spot.

=== why no tool catches it ===
  go vet       : a slice is a slice; nothing to flag
  go test -race: one goroutine, no data race
  unit tests   : read, then check, with no write in between — passes
  It needs a write, after the read, that happens to reuse that page.
  In production that is every block. Copy at the boundary.
```

---

## 8. Prefix scans over the UTXO set

`🟡 medium` · *Key-space design*

A composite key does two jobs. `txid | vout` identifies an output, and because keys sort as bytes, every output of one transaction sits in one contiguous run that a `Seek` and a prefix check can walk. "What does this address own?" needs the address at the *front* of some key, which means a second bucket — and this example measures what that index buys, and what every byte of it costs.

**Steps:**

1. Load the outputs of 2,000 transactions into `utxo`, and again into `byaddr` keyed address-first; spend 30% of them.
2. List the unspent outputs of three transactions with `Seek` plus `bytes.HasPrefix`.
3. Compute alice's balance by scanning every entry, then by a prefix scan of the index.
4. Leave out the prefix check, and watch the balance come back wrong with no error.
5. Price both key layouts at a hundred million outputs.

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
// Prefix scans over the UTXO set.
//
// A composite key — txid, then output index — does two jobs. It is the
// unique id of an output, and because keys sort as bytes, every output of one
// transaction sits in one contiguous run. "Seek to the prefix, walk while the
// key still starts with it" is a range query for free.
//
// The same trick answers "what does this address own?" — but only if the
// address is at the FRONT of some key. The UTXO set is keyed by outpoint, so
// that takes a second bucket, and this example measures what it buys and
// what it costs.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket   key                                     value
//   ------   -------------------------------------   ------------------------
//   utxo     txid [32] | vout [4] BE           36    entry                 38
//   byaddr   pkh [20] | txid [32] | vout [4]   56    (empty: the key is the data)
//
//   entry = version [1] | value [8] | height [8] | coinbase [1] | pkh [20]
//
//   The two buckets change in the same write transaction, always.
// ---------------------------------------------------------------------------

const (
	RecordV1  = 0x01
	EntrySize = 38
	Coin      = int64(100_000_000)

	nTx    = 2000
	nAddrs = 50
)

var (
	bktUTXO = []byte("utxo")
	bktAddr = []byte("byaddr")

	ErrMissing  = errors.New("outpoint not in the UTXO set")
	ErrCorrupt  = errors.New("corrupt UTXO entry")
	ErrDangling = errors.New("address index points at a missing output")
)

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

// Entry uses a fixed-size array for the key hash, so decoding copies it by
// assignment and nothing can alias the database's memory (example 4).
type Entry struct {
	Value    int64
	Height   uint64
	Coinbase bool
	PKH      [20]byte
}

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

// addrKey puts the address FIRST, so one Seek finds all of its outputs — and
// the 36 bytes after it are, byte for byte, that output's utxo key.
func addrKey(pkh [20]byte, op Outpoint) []byte {
	k := make([]byte, 0, 56)
	k = append(k, pkh[:]...)
	return append(k, utxoKey(op)...)
}

func encodeEntry(e Entry) []byte {
	b := make([]byte, 0, EntrySize)
	b = append(b, RecordV1)
	b = binary.BigEndian.AppendUint64(b, uint64(e.Value))
	b = binary.BigEndian.AppendUint64(b, e.Height)
	if e.Coinbase {
		b = append(b, 1)
	} else {
		b = append(b, 0)
	}
	return append(b, e.PKH[:]...)
}

func decodeEntry(v []byte) (Entry, error) {
	if len(v) != EntrySize || v[0] != RecordV1 {
		return Entry{}, ErrCorrupt
	}
	var e Entry
	e.Value = int64(binary.BigEndian.Uint64(v[1:9]))
	e.Height = binary.BigEndian.Uint64(v[9:17])
	e.Coinbase = v[17] == 1
	copy(e.PKH[:], v[18:38])
	return e, nil
}

// ------------------------------------------------------------ writes (both)

func create(utxo, idx *bolt.Bucket, op Outpoint, e Entry) error {
	if err := utxo.Put(utxoKey(op), encodeEntry(e)); err != nil {
		return err
	}
	return idx.Put(addrKey(e.PKH, op), []byte{})
}

func spend(utxo, idx *bolt.Bucket, op Outpoint) error {
	v := utxo.Get(utxoKey(op))
	if v == nil {
		return ErrMissing
	}
	e, err := decodeEntry(v) // decoded BEFORE the delete frees that memory
	if err != nil {
		return err
	}
	if err := utxo.Delete(utxoKey(op)); err != nil {
		return err
	}
	return idx.Delete(addrKey(e.PKH, op))
}

// ------------------------------------------------------------------ queries

// OutputsOf lists the unspent outputs of one transaction.
func OutputsOf(tx *bolt.Tx, txid [32]byte) (vouts []uint32, read int) {
	c := tx.Bucket(bktUTXO).Cursor()
	for k, _ := c.Seek(txid[:]); k != nil && bytes.HasPrefix(k, txid[:]); k, _ = c.Next() {
		read++
		vouts = append(vouts, binary.BigEndian.Uint32(k[32:]))
	}
	return vouts, read
}

// BalanceScan decodes every entry in the set and keeps the ones it wants.
func BalanceScan(tx *bolt.Tx, pkh [20]byte) (total int64, coins, read int, err error) {
	err = tx.Bucket(bktUTXO).ForEach(func(_, v []byte) error {
		read++
		e, err := decodeEntry(v)
		if err != nil {
			return err
		}
		if e.PKH == pkh {
			total += e.Value
			coins++
		}
		return nil
	})
	return total, coins, read, err
}

// BalanceIndexed seeks to the address in byaddr, walks while the prefix
// matches, and fetches each output by the key embedded in the index key.
func BalanceIndexed(tx *bolt.Tx, pkh [20]byte) (total int64, coins, read int, err error) {
	utxo := tx.Bucket(bktUTXO)
	c := tx.Bucket(bktAddr).Cursor()
	for k, _ := c.Seek(pkh[:]); k != nil && bytes.HasPrefix(k, pkh[:]); k, _ = c.Next() {
		v := utxo.Get(k[20:])
		read += 2
		if v == nil {
			return 0, 0, read, fmt.Errorf("%w: %x…", ErrDangling, k[20:24])
		}
		e, err := decodeEntry(v)
		if err != nil {
			return 0, 0, read, err
		}
		total += e.Value
		coins++
	}
	return total, coins, read, nil
}

// BalanceNoStop is BalanceIndexed with the prefix check forgotten.
func BalanceNoStop(tx *bolt.Tx, pkh [20]byte) (total int64, coins int, err error) {
	utxo := tx.Bucket(bktUTXO)
	c := tx.Bucket(bktAddr).Cursor()
	for k, _ := c.Seek(pkh[:]); k != nil; k, _ = c.Next() { // BUG: never stops
		e, err := decodeEntry(utxo.Get(k[20:]))
		if err != nil {
			return 0, 0, err
		}
		total += e.Value
		coins++
	}
	return total, coins, nil
}

// ------------------------------------------------------------------ helpers

// rng is a counter hashed with SHA-256: deterministic, so output reproduces.
type rng struct{ n uint64 }

func (r *rng) next() uint64 {
	r.n++
	s := sha256.Sum256(binary.BigEndian.AppendUint64(nil, r.n))
	return binary.BigEndian.Uint64(s[:8])
}

func addrOf(i int) (string, [20]byte) {
	name := fmt.Sprintf("user-%02d", i)
	if names := []string{"alice", "bob", "carol"}; i < len(names) {
		name = names[i]
	}
	s := sha256.Sum256([]byte(name))
	var pkh [20]byte
	copy(pkh[:], s[:20])
	return name, pkh
}

func btc(sat int64) string { return fmt.Sprintf("%d.%08d", sat/Coin, sat%Coin) }

func gb(n int64) string { return fmt.Sprintf("%d.%d GB", n/1_000_000_000, n/100_000_000%10) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "prefix-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()

	r := &rng{}
	var created []Outpoint
	check(db.Update(func(tx *bolt.Tx) error {
		utxo, err := tx.CreateBucket(bktUTXO)
		if err != nil {
			return err
		}
		idx, err := tx.CreateBucket(bktAddr)
		if err != nil {
			return err
		}
		for i := 0; i < nTx; i++ {
			txid := sha256.Sum256([]byte(fmt.Sprintf("tx-%d", i)))
			nOut := 1 + int(r.next()%4)
			for vout := 0; vout < nOut; vout++ {
				_, pkh := addrOf(int(r.next() % nAddrs))
				op := Outpoint{txid, uint32(vout)}
				e := Entry{Value: int64(1+r.next()%5000) * 100_000, Height: uint64(i / 10), PKH: pkh}
				if err := create(utxo, idx, op, e); err != nil {
					return err
				}
				created = append(created, op)
			}
		}
		return nil
	}))

	spent := 0
	check(db.Update(func(tx *bolt.Tx) error {
		utxo, idx := tx.Bucket(bktUTXO), tx.Bucket(bktAddr)
		for _, op := range created {
			if r.next()%10 < 3 {
				if err := spend(utxo, idx, op); err != nil {
					return err
				}
				spent++
			}
		}
		return nil
	}))

	check(db.View(func(tx *bolt.Tx) error {
		nUTXO := tx.Bucket(bktUTXO).Stats().KeyN
		nIdx := tx.Bucket(bktAddr).Stats().KeyN

		fmt.Printf("=== %d transactions, %d outputs to %d addresses, %d spent ===\n", nTx, len(created), nAddrs, spent)
		fmt.Printf("  utxo    %5d entries   keys 36 bytes   %6d bytes of keys\n", nUTXO, nUTXO*36)
		fmt.Printf("  byaddr  %5d entries   keys 56 bytes   %6d bytes of keys\n", nIdx, nIdx*56)

		fmt.Println("\n=== every unspent output of one transaction ===")
		for _, i := range []int{7, 8, 11} {
			txid := sha256.Sum256([]byte(fmt.Sprintf("tx-%d", i)))
			vouts, read := OutputsOf(tx, txid)
			fmt.Printf("  tx-%-3d %x…  outputs %-9s  %d keys read\n", i, txid[:4], fmt.Sprint(vouts), read)
		}
		fmt.Println("  Seek(txid) lands on the first key >= txid. If that key does not")
		fmt.Println("  start with txid, the transaction has nothing unspent — Seek never")
		fmt.Println("  says 'not found', it just returns whatever comes next.")

		name, alice := addrOf(0)
		fmt.Printf("\n=== %s's balance, three ways ===\n", name)
		t1, c1, read1, err := BalanceScan(tx, alice)
		if err != nil {
			return err
		}
		t2, c2, read2, err := BalanceIndexed(tx, alice)
		if err != nil {
			return err
		}
		t3, c3, err := BalanceNoStop(tx, alice)
		if err != nil {
			return err
		}
		fmt.Printf("  full scan of utxo        %s in %2d coins   %5d keys read\n", btc(t1), c1, read1)
		fmt.Printf("  prefix scan of byaddr    %s in %2d coins   %5d keys read\n", btc(t2), c2, read2)
		fmt.Printf("  same answer: %v\n", t1 == t2 && c1 == c2)
		fmt.Printf("  prefix scan, no stop     %s in %d coins  <- everyone who sorts after %s\n", btc(t3), c3, name)
		fmt.Println()
		fmt.Println("  The forgotten HasPrefix is the classic prefix-scan bug. It returns")
		fmt.Println("  a plausible number, never an error, and the answer is right for")
		fmt.Println("  whichever address happens to sort last.")

		fmt.Println("\n=== what the index costs at chain scale ===")
		const utxos = 100_000_000
		fmt.Printf("  utxo keys, 100M outputs    %s\n", gb(36*utxos))
		fmt.Printf("  byaddr keys, 100M outputs  %s   (plus B+tree overhead, plus a write per output)\n", gb(56*utxos))
		fmt.Println("  Every byte of a key is paid once per entry, in every index that")
		fmt.Println("  contains it. Bitcoin Core keeps NO address index for this reason —")
		fmt.Println("  consensus never asks who owns what. Explorers and wallet servers")
		fmt.Println("  (Electrum servers, Esplora) build one, outside the node.")
		return nil
	}))
}
```

**Output:**

```
=== 2000 transactions, 4997 outputs to 50 addresses, 1525 spent ===
  utxo     3472 entries   keys 36 bytes   124992 bytes of keys
  byaddr   3472 entries   keys 56 bytes   194432 bytes of keys

=== every unspent output of one transaction ===
  tx-7   05320dd8…  outputs [0 1 2]    3 keys read
  tx-8   33a82344…  outputs [0 2 3]    3 keys read
  tx-11  fab2f5a3…  outputs [0 1 2]    3 keys read
  Seek(txid) lands on the first key >= txid. If that key does not
  start with txid, the transaction has nothing unspent — Seek never
  says 'not found', it just returns whatever comes next.

=== alice's balance, three ways ===
  full scan of utxo        160.66000000 in 69 coins    3472 keys read
  prefix scan of byaddr    160.66000000 in 69 coins     138 keys read
  same answer: true
  prefix scan, no stop     7037.48000000 in 2844 coins  <- everyone who sorts after alice

  The forgotten HasPrefix is the classic prefix-scan bug. It returns
  a plausible number, never an error, and the answer is right for
  whichever address happens to sort last.

=== what the index costs at chain scale ===
  utxo keys, 100M outputs    3.6 GB
  byaddr keys, 100M outputs  5.6 GB   (plus B+tree overhead, plus a write per output)
  Every byte of a key is paid once per entry, in every index that
  contains it. Bitcoin Core keeps NO address index for this reason —
  consensus never asks who owns what. Explorers and wallet servers
  (Electrum servers, Esplora) build one, outside the node.
```

---

## 9. One block, one write transaction

`🟡 medium` · *Atomic writes*

Connecting a block touches four things — the block bytes, the height index, the UTXO set's deletes and inserts, and the tip pointer — and they must change together or not at all. Put them in one `db.Update` and the closure becomes the atomic unit: an error or a panic anywhere rolls back everything, which is what finally lets validation and mutation share a single pass. Then two blocks race for the same height, with the tip checked inside the transaction and outside it.

**Steps:**

1. Connect three blocks, one of which spends an output created earlier in the same block.
2. Apply a block whose second transaction spends a coin that never existed, and fingerprint the state before and after.
3. Apply it again, panicking halfway through this time, and fingerprint again.
4. Apply one valid block to two copies of the database, with the writes in opposite orders.
5. Race two competing blocks for height 4: once with the tip checked inside `db.Update`, once before it.

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
	"sync"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// One block, one write transaction.
//
// Connecting a block touches four things: the block bytes, the height index,
// the UTXO set (deletes AND inserts) and the tip pointer. If an error, a
// panic or a crash can separate any two of them, the node is left with a
// state that matches no block — and it will not know.
//
// So all four go in ONE db.Update. The closure is the atomic unit: return nil
// and bbolt commits everything; return an error, or panic, and it rolls back
// everything. That quietly relaxes lesson 10's rule. In memory you had to
// validate completely before mutating. Inside a write transaction you may
// mutate as you validate, because returning an error undoes it.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key                       value
//   -------   -----------------------   -------------------------------------
//   blocks    hash [32]                 version [1] | header [92] | txs
//   heights   height [8] BE             hash [32]
//   utxo      txid [32] | vout [4] BE   value [8] | pkh [20]
//   meta      "tip"                     hash [32]
// ---------------------------------------------------------------------------

const (
	HeaderSize    = 92
	BlockRecordV1 = 0x01
	Coin          = int64(100_000_000)
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
)

var (
	ErrNotOnTip     = errors.New("parent is not the current tip")
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrNotCovered   = errors.New("outputs exceed inputs")
)

// ------------------------------------------------- blocks & txs (08, 10)

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
	Value      int64
	PubKeyHash [20]byte
}

// Transaction is lesson 10's, with the signatures left out: storage never
// checks them. Data carries the coinbase height (BIP-34).
type Transaction struct {
	Data    []byte
	Inputs  []Outpoint
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	b := binary.BigEndian.AppendUint32(nil, uint32(len(t.Data)))
	b = append(b, t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = append(b, in.TxID[:]...)
		b = binary.BigEndian.AppendUint32(b, in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = append(b, o.PubKeyHash[:]...)
	}
	return b
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

const tagLeaf, tagNode byte = 0x00, 0x01

func MerkleRoot(txs []*Transaction) [32]byte {
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

func (b *Block) Encode() []byte {
	out := append([]byte{BlockRecordV1}, b.Header.Bytes()...)
	out = binary.BigEndian.AppendUint32(out, uint32(len(b.Txs)))
	for _, t := range b.Txs {
		raw := t.Serialize()
		out = binary.BigEndian.AppendUint32(out, uint32(len(raw)))
		out = append(out, raw...)
	}
	return out
}

// ---------------------------------------------------------------- the store

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func encodeOut(o TxOutput) []byte {
	return append(binary.BigEndian.AppendUint64(nil, uint64(o.Value)), o.PubKeyHash[:]...)
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

func checkTip(tx *bolt.Tx, b *Block) error {
	tip := tx.Bucket(bktMeta).Get(keyTip)
	if b.Header.Height > 0 && !bytes.Equal(tip, b.Header.PrevHash[:]) {
		return ErrNotOnTip
	}
	return nil
}

// applyUTXO validates and mutates in one pass. That is safe ONLY because it
// runs inside a write transaction: an error anywhere rolls all of it back.
func applyUTXO(tx *bolt.Tx, b *Block) error {
	utxo := tx.Bucket(bktUTXO)
	for i, t := range b.Txs {
		var in, out int64
		for _, prev := range t.Inputs {
			k := utxoKey(prev)
			v := utxo.Get(k) // sees outputs created earlier in this same transaction
			if v == nil {
				return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
			}
			in += int64(binary.BigEndian.Uint64(v[:8]))
			if err := utxo.Delete(k); err != nil {
				return err
			}
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			out += o.Value
			if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), encodeOut(o)); err != nil {
				return err
			}
		}
		if i > 0 && out > in {
			return fmt.Errorf("tx %d: %w", i, ErrNotCovered)
		}
	}
	return nil
}

func putBlock(tx *bolt.Tx, b *Block) error {
	hash := b.Header.Hash()
	if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
		return err
	}
	return tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:])
}

func moveTip(tx *bolt.Tx, b *Block) error {
	hash := b.Header.Hash()
	return tx.Bucket(bktMeta).Put(keyTip, hash[:])
}

// applyIn is the whole block. checkTip is a parameter only so the race
// below can show what leaving it out costs.
func applyIn(tx *bolt.Tx, b *Block, withTipCheck bool) error {
	if withTipCheck {
		if err := checkTip(tx, b); err != nil {
			return err
		}
	}
	if err := applyUTXO(tx, b); err != nil {
		return err
	}
	if err := putBlock(tx, b); err != nil {
		return err
	}
	return moveTip(tx, b)
}

// ApplyBlock: one block, one db.Update.
func ApplyBlock(db *bolt.DB, b *Block) error {
	return db.Update(func(tx *bolt.Tx) error { return applyIn(tx, b, true) })
}

// ApplyBlockTipFirst does the same writes in the opposite order.
func ApplyBlockTipFirst(db *bolt.DB, b *Block) error {
	return db.Update(func(tx *bolt.Tx) error {
		if err := checkTip(tx, b); err != nil {
			return err
		}
		if err := moveTip(tx, b); err != nil {
			return err
		}
		if err := putBlock(tx, b); err != nil {
			return err
		}
		return applyUTXO(tx, b)
	})
}

// CheckThenApply reads the tip in one transaction and writes in another,
// trusting what it read. afterRead holds the window between them open.
func CheckThenApply(db *bolt.DB, b *Block, afterRead func()) error {
	var tip []byte
	if err := db.View(func(tx *bolt.Tx) error {
		tip = bytes.Clone(tx.Bucket(bktMeta).Get(keyTip))
		return nil
	}); err != nil {
		return err
	}
	afterRead()
	if !bytes.Equal(tip, b.Header.PrevHash[:]) {
		return ErrNotOnTip
	}
	return db.Update(func(tx *bolt.Tx) error { return applyIn(tx, b, false) })
}

// --------------------------------------------------------------- inspection

type State struct {
	Fingerprint [32]byte
	Tip         uint64
	UTXOs       int
}

func (s State) String() string {
	return fmt.Sprintf("tip %d, %2d utxos, fingerprint %x", s.Tip, s.UTXOs, s.Fingerprint[:4])
}

// ReadState hashes every key and value in every bucket, in order.
func ReadState(db *bolt.DB) (s State) {
	lp := func(p []byte) []byte { return append(binary.BigEndian.AppendUint32(nil, uint32(len(p))), p...) }
	check(db.View(func(tx *bolt.Tx) error {
		h := sha256.New()
		err := tx.ForEach(func(name []byte, b *bolt.Bucket) error {
			h.Write(lp(name))
			return b.ForEach(func(k, v []byte) error {
				h.Write(lp(k))
				h.Write(lp(v))
				return nil
			})
		})
		copy(s.Fingerprint[:], h.Sum(nil))
		s.UTXOs = tx.Bucket(bktUTXO).Stats().KeyN
		if tip := tx.Bucket(bktMeta).Get(keyTip); tip != nil {
			s.Tip = binary.BigEndian.Uint64(tx.Bucket(bktBlocks).Get(tip)[1+84 : 1+92])
		}
		return err
	}))
	return s
}

func hasOutput(db *bolt.DB, op Outpoint) (ok bool) {
	check(db.View(func(tx *bolt.Tx) error {
		ok = tx.Bucket(bktUTXO).Get(utxoKey(op)) != nil
		return nil
	}))
	return ok
}

// race runs apply on every block concurrently and counts the outcomes.
func race(apply func(*Block) error, blocks ...*Block) (connected, refused int) {
	errs := make([]error, len(blocks))
	var wg sync.WaitGroup
	for i, b := range blocks {
		wg.Add(1)
		go func() {
			defer wg.Done()
			errs[i] = apply(b)
		}()
	}
	wg.Wait()
	for _, err := range errs {
		switch {
		case err == nil:
			connected++
		case errors.Is(err, ErrNotOnTip):
			refused++
		default:
			panic(err)
		}
	}
	return connected, refused
}

// ------------------------------------------------------------------ helpers

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func newBlock(parent *Block, miner [20]byte, txs ...*Transaction) *Block {
	h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Header.Hash()
		h.Height = parent.Header.Height + 1
		h.Timestamp = parent.Header.Timestamp + 600
	}
	cb := &Transaction{
		Data:    be64(h.Height),
		Outputs: []TxOutput{{Value: 50 * Coin, PubKeyHash: miner}},
	}
	all := append([]*Transaction{cb}, txs...)
	h.MerkleRoot = MerkleRoot(all)
	return &Block{Header: h, Txs: all} // no proof of work: storage does not check it
}

// pay spends one coin worth `value`: `amount` to `to`, the rest back to `change`.
func pay(from Outpoint, value int64, to [20]byte, amount int64, change [20]byte) *Transaction {
	return &Transaction{
		Inputs:  []Outpoint{from},
		Outputs: []TxOutput{{Value: amount, PubKeyHash: to}, {Value: value - amount, PubKeyHash: change}},
	}
}

func outpoint(t *Transaction, i uint32) Outpoint { return Outpoint{t.TxID(), i} }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "atomic-*")
	check(err)
	defer os.RemoveAll(dir)
	opts := &bolt.Options{Timeout: time.Second}
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, opts)
	check(err)
	defer db.Close()
	check(initSchema(db))

	miner, alice, bob, carol := pkh("miner"), pkh("alice"), pkh("bob"), pkh("carol")

	b0 := newBlock(nil, miner)
	b1 := newBlock(b0, miner, pay(outpoint(b0.Txs[0], 0), 50*Coin, alice, 30*Coin, miner))
	t2a := pay(outpoint(b1.Txs[1], 0), 30*Coin, bob, 25*Coin, alice)
	t2b := pay(outpoint(t2a, 0), 25*Coin, carol, 5*Coin, bob) // spends t2a, in the same block
	b2 := newBlock(b1, miner, t2a, t2b)

	fmt.Println("=== blocks 0-2, one write transaction each ===")
	for _, b := range []*Block{b0, b1, b2} {
		check(ApplyBlock(db, b))
		fmt.Printf("  height %d  %d tx   %v\n", b.Header.Height, len(b.Txs), ReadState(db))
	}
	fmt.Println("  Block 2's second transaction spends an output its first one created.")
	fmt.Println("  That works because a write transaction reads its own writes.")

	carolsCoin := outpoint(t2b, 0)
	t3a := pay(carolsCoin, 5*Coin, alice, 2*Coin, carol)
	ghost := Outpoint{sha256.Sum256([]byte("a coin nobody ever created")), 0}
	bad := newBlock(b2, miner, t3a, pay(ghost, 1000*Coin, bob, 1000*Coin, bob))

	fmt.Println("\n=== block 3: its second transaction spends a coin that never existed ===")
	before := ReadState(db)
	calls := 0
	err = db.Update(func(tx *bolt.Tx) error {
		calls++
		return applyIn(tx, bad, true)
	})
	after := ReadState(db)
	fmt.Printf("  db.Update returned   %v\n", err)
	fmt.Printf("  errors.Is(err, ErrMissingInput) = %v, and the closure ran %d time\n", errors.Is(err, ErrMissingInput), calls)
	fmt.Printf("  before  %v\n", before)
	fmt.Printf("  after   %v\n", after)
	fmt.Printf("  unchanged: %v\n", before == after)
	fmt.Printf("  carol's coin, which tx 1 deleted, is back:   %v\n", hasOutput(db, carolsCoin))
	fmt.Printf("  alice's output, which tx 1 inserted, is gone: %v\n", !hasOutput(db, outpoint(t3a, 0)))
	fmt.Println("  When tx 2 failed, tx 1 had already deleted an input and inserted an")
	fmt.Println("  output. The error rolled both back, and bbolt did not try again.")

	fmt.Println("\n=== block 3 again: this time the code panics halfway ===")
	before = ReadState(db)
	msg := func() (msg string) {
		defer func() { msg = fmt.Sprint(recover()) }()
		_ = db.Update(func(tx *bolt.Tx) error {
			utxo := tx.Bucket(bktUTXO)
			if err := utxo.Delete(utxoKey(carolsCoin)); err != nil {
				return err
			}
			if err := utxo.Put(utxoKey(outpoint(t3a, 0)), encodeOut(t3a.Outputs[0])); err != nil {
				return err
			}
			var parsed []TxOutput
			_ = parsed[3] // a decoding bug, further down the block
			return nil
		})
		return ""
	}()
	after = ReadState(db)
	fmt.Printf("  recovered: %s\n", msg)
	fmt.Printf("  unchanged: %v\n", before == after)
	fmt.Println("  db.Update rolls back in a deferred function, so a panic releases the")
	fmt.Println("  writer lock and discards the half-applied block on its way up.")

	fmt.Println("\n=== the ORDER of writes inside the transaction does not matter ===")
	copyPath := filepath.Join(dir, "copy.db")
	check(db.View(func(tx *bolt.Tx) error { return tx.CopyFile(copyPath, 0o600) }))
	db2, err := bolt.Open(copyPath, 0o600, opts)
	check(err)
	defer db2.Close()
	good := newBlock(b2, miner, t3a)
	check(ApplyBlock(db, good))
	check(ApplyBlockTipFirst(db2, good))
	s1, s2 := ReadState(db), ReadState(db2)
	fmt.Printf("  utxo, block, then tip   %v\n", s1)
	fmt.Printf("  tip, block, then utxo   %v\n", s2)
	fmt.Printf("  identical: %v\n", s1 == s2)
	fmt.Println("  Nobody can observe the middle of a transaction, so there is no")
	fmt.Println("  'safe order' to get right. Atomicity is the property; order is not.")

	fmt.Println("\n=== two blocks race for height 4 ===")
	a4, b4 := newBlock(good, pkh("pool-a")), newBlock(good, pkh("pool-b"))
	racePath := filepath.Join(dir, "race.db")
	check(db.View(func(tx *bolt.Tx) error { return tx.CopyFile(racePath, 0o600) }))
	db3, err := bolt.Open(racePath, 0o600, opts)
	check(err)
	defer db3.Close()

	connected, refused := race(func(b *Block) error { return ApplyBlock(db, b) }, a4, b4)
	fmt.Printf("  tip checked inside db.Update     connected %d, refused %d\n", connected, refused)
	fmt.Printf("    both pools' rewards in the UTXO set: %v\n",
		hasOutput(db, outpoint(a4.Txs[0], 0)) && hasOutput(db, outpoint(b4.Txs[0], 0)))

	var read sync.WaitGroup
	read.Add(2)
	bothRead := func() { read.Done(); read.Wait() } // each waits until both have read the tip
	connected, refused = race(func(b *Block) error { return CheckThenApply(db3, b, bothRead) }, a4, b4)
	fmt.Printf("  tip checked, then db.Update      connected %d, refused %d\n", connected, refused)
	fmt.Printf("    both pools' rewards in the UTXO set: %v\n",
		hasOutput(db3, outpoint(a4.Txs[0], 0)) && hasOutput(db3, outpoint(b4.Txs[0], 0)))
	fmt.Println()
	fmt.Println("  bbolt runs one writer at a time, so two Updates never conflict: the")
	fmt.Println("  second simply waits, then runs against the first one's result. That")
	fmt.Println("  is why the check INSIDE the closure is enough. But nothing detects a")
	fmt.Println("  stale read made OUTSIDE it — no conflict error, no retry. Both blocks")
	fmt.Println("  connected at height 4, and the node now holds coins from two blocks")
	fmt.Println("  that can never both be in one chain.")
	fmt.Println("  (Badger is the opposite: optimistic transactions that fail with")
	fmt.Println("  ErrConflict and must be retried. Know which kind your store is.)")
}
```

**Output:**

```
=== blocks 0-2, one write transaction each ===
  height 0  1 tx   tip 0,  1 utxos, fingerprint 8f610785
  height 1  2 tx   tip 1,  3 utxos, fingerprint e6f0f23d
  height 2  3 tx   tip 2,  6 utxos, fingerprint 51acbb92
  Block 2's second transaction spends an output its first one created.
  That works because a write transaction reads its own writes.

=== block 3: its second transaction spends a coin that never existed ===
  db.Update returned   tx 2: input is not in the UTXO set
  errors.Is(err, ErrMissingInput) = true, and the closure ran 1 time
  before  tip 2,  6 utxos, fingerprint 51acbb92
  after   tip 2,  6 utxos, fingerprint 51acbb92
  unchanged: true
  carol's coin, which tx 1 deleted, is back:   true
  alice's output, which tx 1 inserted, is gone: true
  When tx 2 failed, tx 1 had already deleted an input and inserted an
  output. The error rolled both back, and bbolt did not try again.

=== block 3 again: this time the code panics halfway ===
  recovered: runtime error: index out of range [3] with length 0
  unchanged: true
  db.Update rolls back in a deferred function, so a panic releases the
  writer lock and discards the half-applied block on its way up.

=== the ORDER of writes inside the transaction does not matter ===
  utxo, block, then tip   tip 3,  8 utxos, fingerprint 7a0d60a4
  tip, block, then utxo   tip 3,  8 utxos, fingerprint 7a0d60a4
  identical: true
  Nobody can observe the middle of a transaction, so there is no
  'safe order' to get right. Atomicity is the property; order is not.

=== two blocks race for height 4 ===
  tip checked inside db.Update     connected 1, refused 1
    both pools' rewards in the UTXO set: false
  tip checked, then db.Update      connected 2, refused 0
    both pools' rewards in the UTXO set: true

  bbolt runs one writer at a time, so two Updates never conflict: the
  second simply waits, then runs against the first one's result. That
  is why the check INSIDE the closure is enough. But nothing detects a
  stale read made OUTSIDE it — no conflict error, no retry. Both blocks
  connected at height 4, and the node now holds coins from two blocks
  that can never both be in one chain.
  (Badger is the opposite: optimistic transactions that fail with
  ErrConflict and must be retried. Know which kind your store is.)
```

---

## 10. Three write transactions instead of one

`🟡 medium` · *Atomic writes*

The same work as example 9 — the block, the UTXO delta, the tip pointer — split into three `db.Update` calls, which is what code looks like when every function "does its own transaction". The writer is killed after every possible step, the node restarts, and a startup check compares the tip with the marker the UTXO set keeps about itself. Two orderings, two different fatal kill points, and one writer with none.

**Steps:**

1. Connect blocks 0–4 atomically, and prepare block 5 (pays bob) and block 6 (spends bob's coin).
2. Connect block 5 with a split writer, killed after zero, one, two and three steps — a fresh copy of the database each time.
3. After each kill, restart: run the startup check, then finish block 5 and connect block 6.
4. Repeat with the tip moved before the UTXO set instead of after it.
5. Repeat with all three steps in one `db.Update`.

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
// Three write transactions instead of one.
//
// Example 9 connected a block in one db.Update. This example splits the same
// work into three — the block, the UTXO delta, the tip pointer — which is what
// the code looks like when each piece belongs to a different function and
// every function "does its own transaction".
//
// Then it kills the writer between every pair of steps, restarts the node,
// and asks two questions:
//
//   1. Does the startup check notice anything?
//   2. Can the node carry on: finish the block it was writing, then the next?
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key          value
//   -------   ----------   ---------------------------------------------------
//   blocks    hash         version [1] | header [92] | txs
//   heights   height BE    hash
//   utxo      txid|vout    value [8] | pkh [20]
//   meta      "tip"        hash of the chain's tip
//             "utxo-tip"   hash of the block the UTXO set reflects. Written in
//                          the SAME transaction as the UTXO changes, always —
//                          the state carries its own marker.
// ---------------------------------------------------------------------------

const (
	HeaderSize    = 92
	BlockRecordV1 = 0x01
	Coin          = int64(100_000_000)
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
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrInconsistent = errors.New("tip and UTXO set disagree")
	errKilled       = errors.New("process killed")
)

// ------------------------------------------------- blocks & txs (08, 10)

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
	Value      int64
	PubKeyHash [20]byte
}

type Transaction struct {
	Data    []byte // coinbase height
	Inputs  []Outpoint
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	b := binary.BigEndian.AppendUint32(nil, uint32(len(t.Data)))
	b = append(b, t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = append(b, in.TxID[:]...)
		b = binary.BigEndian.AppendUint32(b, in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = append(b, o.PubKeyHash[:]...)
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
	out := append([]byte{BlockRecordV1}, b.Header.Bytes()...)
	for _, t := range b.Txs {
		raw := t.Serialize()
		out = binary.BigEndian.AppendUint32(out, uint32(len(raw)))
		out = append(out, raw...)
	}
	return out
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

// ---------------------------------------------------------------- the steps

type Step int

const (
	StepBlock Step = iota // the block bytes and its height entry
	StepUTXO              // spent outputs out, new outputs in, the utxo-tip marker
	StepTip               // the tip pointer
)

func (s Step) String() string { return [...]string{"block", "utxo", "tip"}[s] }

func doStep(tx *bolt.Tx, b *Block, s Step) error {
	hash := b.Header.Hash()
	switch s {
	case StepBlock:
		if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
			return err
		}
		return tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:])
	case StepUTXO:
		utxo := tx.Bucket(bktUTXO)
		for i, t := range b.Txs {
			for _, in := range t.Inputs {
				if utxo.Get(utxoKey(in)) == nil {
					return fmt.Errorf("block %d tx %d: %w", b.Header.Height, i, ErrMissingInput)
				}
				if err := utxo.Delete(utxoKey(in)); err != nil {
					return err
				}
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				v := append(binary.BigEndian.AppendUint64(nil, uint64(o.Value)), o.PubKeyHash[:]...)
				if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), v); err != nil {
					return err
				}
			}
		}
		return tx.Bucket(bktMeta).Put(keyUTXOTip, hash[:])
	default:
		return tx.Bucket(bktMeta).Put(keyTip, hash[:])
	}
}

// A Writer connects a block. killAfter < 3 makes the process die after that
// many steps have completed.
type Writer func(db *bolt.DB, b *Block, killAfter int) error

// SplitWriter gives every step its own db.Update, in the order given.
func SplitWriter(order ...Step) Writer {
	return func(db *bolt.DB, b *Block, killAfter int) error {
		for i, s := range order {
			if i == killAfter {
				return errKilled
			}
			if err := db.Update(func(tx *bolt.Tx) error { return doStep(tx, b, s) }); err != nil {
				return err
			}
		}
		return nil
	}
}

// AtomicWriter does all three steps in one db.Update. Dying before the
// closure returns means nothing commits — which is exactly what a real crash
// leaves on disk too, because bbolt writes the meta page last (example 14
// kills a real process to prove it).
func AtomicWriter(db *bolt.DB, b *Block, killAfter int) error {
	return db.Update(func(tx *bolt.Tx) error {
		for i, s := range []Step{StepBlock, StepUTXO, StepTip} {
			if i == killAfter {
				return errKilled
			}
			if err := doStep(tx, b, s); err != nil {
				return err
			}
		}
		return nil
	})
}

// ------------------------------------------------------------------ startup

// StartupCheck compares two markers written by two different steps. If they
// disagree, a writer died between them and the state matches no block.
func StartupCheck(db *bolt.DB) (tip, utxoTip uint64, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		meta, blocks := tx.Bucket(bktMeta), tx.Bucket(bktBlocks)
		height := func(hash []byte) uint64 {
			return binary.BigEndian.Uint64(blocks.Get(hash)[1+84 : 1+92])
		}
		tip, utxoTip = height(meta.Get(keyTip)), height(meta.Get(keyUTXOTip))
		if !bytes.Equal(meta.Get(keyTip), meta.Get(keyUTXOTip)) {
			return ErrInconsistent
		}
		return nil
	})
	return tip, utxoTip, err
}

// Resume is a node starting up: finish the block it was writing if the tip
// says it is not connected, then connect the next one.
func Resume(db *bolt.DB, w Writer, pending, next *Block) error {
	var tip []byte
	if err := db.View(func(tx *bolt.Tx) error {
		tip = bytes.Clone(tx.Bucket(bktMeta).Get(keyTip))
		return nil
	}); err != nil {
		return err
	}
	if bytes.Equal(tip, pending.Header.PrevHash[:]) {
		if err := w(db, pending, 3); err != nil {
			return err
		}
	}
	return w(db, next, 3)
}

// ------------------------------------------------------------------ helpers

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func newBlock(parent *Block, txs ...*Transaction) *Block {
	h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Header.Hash()
		h.Height = parent.Header.Height + 1
		h.Timestamp = parent.Header.Timestamp + 600
	}
	cb := &Transaction{Data: be64(h.Height), Outputs: []TxOutput{{Value: 50 * Coin, PubKeyHash: pkh("miner")}}}
	h.MerkleRoot = sha256.Sum256(cb.Serialize()) // the real tree is in example 9; nothing here reads it
	return &Block{Header: h, Txs: append([]*Transaction{cb}, txs...)}
}

func pay(from Outpoint, value int64, to [20]byte, amount int64, change [20]byte) *Transaction {
	return &Transaction{
		Inputs:  []Outpoint{from},
		Outputs: []TxOutput{{Value: amount, PubKeyHash: to}, {Value: value - amount, PubKeyHash: change}},
	}
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "split-*")
	check(err)
	defer os.RemoveAll(dir)
	opts := &bolt.Options{Timeout: time.Second}
	open := func(path string) *bolt.DB {
		db, err := bolt.Open(path, 0o600, opts)
		check(err)
		return db
	}

	alice, bob, carol := pkh("alice"), pkh("bob"), pkh("carol")

	// Blocks 0-4 connected atomically, then the two blocks under test:
	// block 5 pays bob out of alice's coin; block 6 spends bob's new coin.
	var chain []*Block
	chain = append(chain, newBlock(nil))
	chain = append(chain, newBlock(chain[0], pay(Outpoint{chain[0].Txs[0].TxID(), 0}, 50*Coin, alice, 40*Coin, pkh("miner"))))
	for i := 2; i <= 4; i++ {
		chain = append(chain, newBlock(chain[i-1]))
	}
	b5 := newBlock(chain[4], pay(Outpoint{chain[1].Txs[1].TxID(), 0}, 40*Coin, bob, 15*Coin, alice))
	b6 := newBlock(b5, pay(Outpoint{b5.Txs[1].TxID(), 0}, 15*Coin, carol, 6*Coin, bob))

	basePath := filepath.Join(dir, "base.db")
	base := open(basePath)
	check(base.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucket(name); err != nil {
				return err
			}
		}
		return nil
	}))
	for _, b := range chain {
		check(AtomicWriter(base, b, 3))
	}
	fmt.Println("blocks 0-4 connected. Block 5 pays bob from alice's coin; block 6 spends bob's.")

	runs := 0
	run := func(title string, w Writer, labels []string) {
		fmt.Printf("\n=== %s ===\n", title)
		fmt.Printf("  %-26s %-4s %-9s %-14s %s\n", "killed after", "tip", "utxo-tip", "startup check", "then: finish block 5, connect block 6")
		for kill := 0; kill <= 3; kill++ {
			runs++
			path := filepath.Join(dir, fmt.Sprintf("run-%d.db", runs))
			check(base.View(func(tx *bolt.Tx) error { return tx.CopyFile(path, 0o600) }))

			db := open(path)
			if err := w(db, b5, kill); err != nil && !errors.Is(err, errKilled) {
				panic(err)
			}
			check(db.Close()) // the process is gone

			db = open(path) // and a new one starts
			tip, utxoTip, err := StartupCheck(db)
			verdict := "consistent"
			if errors.Is(err, ErrInconsistent) {
				verdict = "INCONSISTENT"
			} else {
				check(err)
			}
			outcome := "ok"
			if err := Resume(db, w, b5, b6); err != nil {
				outcome = err.Error()
			}
			check(db.Close())
			fmt.Printf("  %-26s %-4d %-9d %-14s %s\n", labels[kill], tip, utxoTip, verdict, outcome)
		}
	}

	run("block, utxo, tip: one db.Update each", SplitWriter(StepBlock, StepUTXO, StepTip),
		[]string{"nothing", "block", "block, utxo", "(not killed)"})
	run("block, tip, utxo: one db.Update each", SplitWriter(StepBlock, StepTip, StepUTXO),
		[]string{"nothing", "block", "block, tip", "(not killed)"})
	run("all three in one db.Update", AtomicWriter,
		[]string{"nothing committed", "block (rolled back)", "block, utxo (rolled back)", "(not killed)"})
	check(base.Close())

	fmt.Println()
	fmt.Println("  Split writers have one fatal kill point each, and they are different")
	fmt.Println("  points. Tip-last wedges on RE-APPLYING block 5: its input is already")
	fmt.Println("  gone. Tip-first wedges on block 6: bob's coin was never created. Both")
	fmt.Println("  nodes now reject valid blocks, forever, and neither crashed.")
	fmt.Println()
	fmt.Println("  The startup check turns that silent wedge into a loud refusal. It")
	fmt.Println("  works only because utxo-tip is written in the SAME transaction as the")
	fmt.Println("  UTXO changes: the state records which block it reflects. Bitcoin Core")
	fmt.Println("  keeps exactly this beside its coins database, and replays blocks on")
	fmt.Println("  startup when it disagrees with the block index.")
	fmt.Println()
	fmt.Println("  The atomic writer has no kill point that matters. That is the point.")
}
```

**Output:**

```
blocks 0-4 connected. Block 5 pays bob from alice's coin; block 6 spends bob's.

=== block, utxo, tip: one db.Update each ===
  killed after               tip  utxo-tip  startup check  then: finish block 5, connect block 6
  nothing                    4    4         consistent     ok
  block                      4    4         consistent     ok
  block, utxo                4    5         INCONSISTENT   block 5 tx 1: input is not in the UTXO set
  (not killed)               5    5         consistent     ok

=== block, tip, utxo: one db.Update each ===
  killed after               tip  utxo-tip  startup check  then: finish block 5, connect block 6
  nothing                    4    4         consistent     ok
  block                      4    4         consistent     ok
  block, tip                 5    4         INCONSISTENT   block 6 tx 1: input is not in the UTXO set
  (not killed)               5    5         consistent     ok

=== all three in one db.Update ===
  killed after               tip  utxo-tip  startup check  then: finish block 5, connect block 6
  nothing committed          4    4         consistent     ok
  block (rolled back)        4    4         consistent     ok
  block, utxo (rolled back)  4    4         consistent     ok
  (not killed)               5    5         consistent     ok

  Split writers have one fatal kill point each, and they are different
  points. Tip-last wedges on RE-APPLYING block 5: its input is already
  gone. Tip-first wedges on block 6: bob's coin was never created. Both
  nodes now reject valid blocks, forever, and neither crashed.

  The startup check turns that silent wedge into a loud refusal. It
  works only because utxo-tip is written in the SAME transaction as the
  UTXO changes: the state records which block it reflects. Bitcoin Core
  keeps exactly this beside its coins database, and replays blocks on
  startup when it disagrees with the block index.

  The atomic writer has no kill point that matters. That is the point.
```

---

## 11. Version bytes, a migration, and a measurement

`🟡 medium` · *Serialization on disk*

The format written today will be read by code written years from now. A version byte on every record lets a decoder tell formats apart; a schema version on the database tells the program a migration is pending — and lets it refuse a database from its own future. But first, the question usually answered without measuring: is compressing blocks worth it?

**Steps:**

1. Build 100 realistic blocks, and deflate one block, the whole stream, one header and one UTXO entry.
2. Write a 30-block chain with version 1 of the program, whose UTXO entries have no height and no coinbase flag.
3. Open it with version 2, which migrates in one write transaction, and kill the migration at entry 17.
4. Migrate again, spot-check a payment and a coinbase, and open the database once more.
5. Open the migrated database with the version-1 binary, and hand version 2 a record from version 3.

```go
package main

import (
	"bytes"
	"compress/flate"
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
// Version bytes, a migration, and a measurement.
//
// The format written today will be read by code written years from now. Two
// habits make that survivable:
//
//   - a VERSION BYTE on every record, so a decoder can tell formats apart
//     instead of misreading one as the other
//   - a SCHEMA VERSION for the whole database, so the program knows whether a
//     migration is pending, and refuses a database from its own future
//
// And one question usually answered without measuring: should records be
// compressed? This example measures first.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   blocks    hash          0x01 | header [92] | count [4] | len-prefixed txs
//   heights   height BE     hash
//   utxo      txid|vout     schema 1:  0x01 | value [8] | pkh [20]                          29
//                           schema 2:  0x02 | value [8] | height [8] | coinbase [1] | pkh [20]  38
//   meta      "schema"      one byte: the schema the whole database is in
// ---------------------------------------------------------------------------

const (
	HeaderSize    = 92
	BlockRecordV1 = 0x01
	Coin          = int64(100_000_000)

	EntryV1 = 0x01
	EntryV2 = 0x02
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktMeta    = []byte("meta")
	keySchema  = []byte("schema")
)

var (
	ErrNewerSchema    = errors.New("database schema is newer than this binary")
	ErrUnknownRecord  = errors.New("unknown record version")
	ErrNeedsMigration = errors.New("record is in an old format")
	ErrCorrupt        = errors.New("corrupt record")
	ErrTruncated      = errors.New("record truncated")
	errDied           = errors.New("process died mid-migration")
)

// ------------------------------------------------- blocks & txs (08, 10)

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

func (t *Transaction) Serialize() []byte {
	var b []byte
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = append(b, in.Prev.TxID[:]...)
		b = binary.BigEndian.AppendUint32(b, in.Prev.Index)
		b = binary.BigEndian.AppendUint32(b, in.Sequence)
		b = appendBytes(b, in.Signature)
		b = appendBytes(b, in.PubKey)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = appendBytes(b, o.PubKeyHash)
	}
	return b
}

func appendBytes(b, p []byte) []byte {
	b = binary.BigEndian.AppendUint32(b, uint32(len(p)))
	return append(b, p...)
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
	out := append([]byte{BlockRecordV1}, b.Header.Bytes()...)
	out = binary.BigEndian.AppendUint32(out, uint32(len(b.Txs)))
	for _, t := range b.Txs {
		out = appendBytes(out, t.Serialize())
	}
	return out
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

func (r *reader) u64() uint64 {
	if v := r.take(8); v != nil {
		return binary.BigEndian.Uint64(v)
	}
	return 0
}

func (r *reader) bytes() []byte { return bytes.Clone(r.take(int(r.u32()))) }

func DecodeBlock(p []byte) (*Block, error) {
	if len(p) == 0 || p[0] != BlockRecordV1 {
		return nil, ErrUnknownRecord
	}
	r := &reader{p: p[1:]}
	var h Header
	h.Version = r.u32()
	copy(h.PrevHash[:], r.take(32))
	copy(h.MerkleRoot[:], r.take(32))
	h.Timestamp = int64(r.u64())
	h.Bits = r.u32()
	h.Nonce = r.u32()
	h.Height = r.u64()
	b := &Block{Header: h}
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
			return nil, tr.err
		}
		b.Txs = append(b.Txs, t)
	}
	return b, r.err
}

// --------------------------------------------------------- the UTXO records

type Entry struct {
	Value    int64
	Height   uint64
	Coinbase bool
	PKH      [20]byte
}

func encodeV1(value int64, pkh []byte) []byte {
	return append(binary.BigEndian.AppendUint64([]byte{EntryV1}, uint64(value)), pkh...)
}

func encodeV2(e Entry) []byte {
	b := binary.BigEndian.AppendUint64([]byte{EntryV2}, uint64(e.Value))
	b = binary.BigEndian.AppendUint64(b, e.Height)
	if e.Coinbase {
		b = append(b, 1)
	} else {
		b = append(b, 0)
	}
	return append(b, e.PKH[:]...)
}

// decodeEntry is what the version-2 program reads everywhere. It dispatches on
// the version byte and never guesses from the length.
func decodeEntry(v []byte) (Entry, error) {
	if len(v) == 0 {
		return Entry{}, ErrCorrupt
	}
	var e Entry
	switch v[0] {
	case EntryV2:
		if len(v) != 38 {
			return Entry{}, ErrCorrupt
		}
		e.Value = int64(binary.BigEndian.Uint64(v[1:9]))
		e.Height = binary.BigEndian.Uint64(v[9:17])
		e.Coinbase = v[17] == 1
		copy(e.PKH[:], v[18:38])
		return e, nil
	case EntryV1:
		return Entry{}, ErrNeedsMigration
	default:
		return Entry{}, fmt.Errorf("%w: 0x%02x", ErrUnknownRecord, v[0])
	}
}

// ----------------------------------------------------- the two programs

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

// WriteChainV1 is version 1 of the program: blocks, heights, and UTXO
// entries that carry only a value and an owner.
func WriteChainV1(db *bolt.DB, chain []*Block) error {
	return db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktBlocks, bktHeights, bktUTXO, bktMeta} {
			if _, err := tx.CreateBucket(name); err != nil {
				return err
			}
		}
		if err := tx.Bucket(bktMeta).Put(keySchema, []byte{1}); err != nil {
			return err
		}
		utxo := tx.Bucket(bktUTXO)
		for _, b := range chain {
			hash := b.Header.Hash()
			if err := tx.Bucket(bktBlocks).Put(hash[:], b.Encode()); err != nil {
				return err
			}
			if err := tx.Bucket(bktHeights).Put(binary.BigEndian.AppendUint64(nil, b.Header.Height), hash[:]); err != nil {
				return err
			}
			for _, t := range b.Txs {
				for _, in := range t.Inputs {
					if err := utxo.Delete(utxoKey(in.Prev)); err != nil {
						return err
					}
				}
				id := t.TxID()
				for vout, o := range t.Outputs {
					if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), encodeV1(o.Value, o.PubKeyHash)); err != nil {
						return err
					}
				}
			}
		}
		return nil
	})
}

// Open is what a program does before touching anything else: read the
// schema, refuse the future, and migrate the past.
func Open(db *bolt.DB, supported byte, dieAfter int) (migrated int, err error) {
	var schema byte
	if err := db.View(func(tx *bolt.Tx) error {
		schema = tx.Bucket(bktMeta).Get(keySchema)[0]
		return nil
	}); err != nil {
		return 0, err
	}
	switch {
	case schema > supported:
		return 0, fmt.Errorf("%w: it is v%d, this binary understands up to v%d", ErrNewerSchema, schema, supported)
	case schema == supported:
		return 0, nil
	}
	return migrateV1toV2(db, dieAfter)
}

// migrateV1toV2 walks the chain forward and rewrites every entry that is
// still unspent with the height and coinbase flag of the block that created
// it — ALL in one write transaction, schema byte included. dieAfter >= 0
// simulates the process dying after that many entries.
func migrateV1toV2(db *bolt.DB, dieAfter int) (n int, err error) {
	err = db.Update(func(tx *bolt.Tx) error {
		utxo, blocks := tx.Bucket(bktUTXO), tx.Bucket(bktBlocks)
		// The cursor walks `heights`; the writes go to `utxo`. Never Put or
		// Delete in the bucket a cursor is walking — it can skip or repeat keys.
		c := tx.Bucket(bktHeights).Cursor()
		for k, hash := c.First(); k != nil; k, hash = c.Next() {
			b, err := DecodeBlock(blocks.Get(hash))
			if err != nil {
				return err
			}
			for i, t := range b.Txs {
				id := t.TxID()
				for vout := range t.Outputs {
					key := utxoKey(Outpoint{id, uint32(vout)})
					v := utxo.Get(key)
					if v == nil {
						continue // spent since
					}
					if len(v) != 29 || v[0] != EntryV1 {
						return ErrCorrupt
					}
					if n == dieAfter {
						return errDied
					}
					e := Entry{Value: int64(binary.BigEndian.Uint64(v[1:9])), Height: b.Header.Height, Coinbase: i == 0}
					copy(e.PKH[:], v[9:29])
					if err := utxo.Put(key, encodeV2(e)); err != nil {
						return err
					}
					n++
				}
			}
		}
		return tx.Bucket(bktMeta).Put(keySchema, []byte{2})
	})
	return n, err
}

func census(db *bolt.DB) (schema byte, byVersion map[byte]int) {
	byVersion = map[byte]int{}
	check(db.View(func(tx *bolt.Tx) error {
		schema = tx.Bucket(bktMeta).Get(keySchema)[0]
		return tx.Bucket(bktUTXO).ForEach(func(_, v []byte) error {
			byVersion[v[0]]++
			return nil
		})
	}))
	return schema, byVersion
}

// ------------------------------------------------------------------ helpers

// rng is SHA-256 over a counter: random-looking bytes, identical every run.
type rng struct{ n uint64 }

func (r *rng) bytes(n int) []byte {
	var out []byte
	for len(out) < n {
		r.n++
		s := sha256.Sum256(binary.BigEndian.AppendUint64(nil, r.n))
		out = append(out, s[:]...)
	}
	return out[:n]
}

func (r *rng) intn(n int) int { return int(binary.BigEndian.Uint64(r.bytes(8)) % uint64(n)) }

func pkh(name string) []byte {
	s := sha256.Sum256([]byte(name))
	return s[:20]
}

// signedInput looks like a real one: a 65-byte signature and a 33-byte key
// are indistinguishable from random bytes, and that is the point.
func signedInput(r *rng, prev Outpoint) TxInput {
	return TxInput{Prev: prev, Signature: r.bytes(65), PubKey: append([]byte{0x02}, r.bytes(32)...), Sequence: 0xfffffffd}
}

func coinbase(height uint64, to []byte) *Transaction {
	return &Transaction{
		Inputs:  []TxInput{{Prev: Outpoint{Index: 0xffffffff}, Signature: binary.BigEndian.AppendUint64(nil, height), Sequence: 0xffffffff}},
		Outputs: []TxOutput{{Value: 50 * Coin, PubKeyHash: to}},
	}
}

func header(parent *Block, txs []*Transaction) Header {
	h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Header.Hash()
		h.Height = parent.Header.Height + 1
		h.Timestamp = parent.Header.Timestamp + 600
	}
	var ids []byte
	for _, t := range txs {
		id := t.TxID()
		ids = append(ids, id[:]...)
	}
	h.MerkleRoot = sha256.Sum256(ids) // a flat commitment is enough here; lesson 05 has the tree
	return h
}

func deflated(p []byte) int {
	var b bytes.Buffer
	w, err := flate.NewWriter(&b, flate.DefaultCompression)
	check(err)
	_, err = w.Write(p)
	check(err)
	check(w.Close())
	return b.Len()
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	r := &rng{}

	// ----------------------------------------------------------------- 1
	fmt.Println("=== 1. should records be compressed? measure first ===")
	addrs := make([][]byte, 300) // a pool of 300 addresses, so some get reused
	for i := range addrs {
		addrs[i] = pkh(fmt.Sprintf("addr-%d", i))
	}
	var blocks []*Block
	var stream []byte
	for height := 0; height < 100; height++ {
		txs := []*Transaction{coinbase(uint64(height), addrs[r.intn(len(addrs))])}
		for i := 0; i < 25; i++ {
			t := &Transaction{}
			for n := 1 + r.intn(3); n > 0; n-- {
				var prev Outpoint
				copy(prev.TxID[:], r.bytes(32))
				prev.Index = uint32(r.intn(3))
				t.Inputs = append(t.Inputs, signedInput(r, prev))
			}
			for n := 2; n > 0; n-- {
				t.Outputs = append(t.Outputs, TxOutput{Value: int64(1+r.intn(2_000_000)) * 1000, PubKeyHash: addrs[r.intn(len(addrs))]})
			}
			txs = append(txs, t)
		}
		var parent *Block
		if height > 0 {
			parent = blocks[height-1]
		}
		b := &Block{Header: header(parent, txs), Txs: txs}
		blocks = append(blocks, b)
		stream = append(stream, b.Encode()...)
	}
	one := blocks[50].Encode()
	entry := encodeV2(Entry{Value: 12_345_000, Height: 50, PKH: [20]byte(addrs[7])})
	hdr := blocks[50].Header.Bytes()

	row := func(what string, raw, packed int) {
		fmt.Printf("  %-30s %9d  %9d   %3d%%\n", what, raw, packed, packed*100/raw)
	}
	fmt.Printf("  %-30s %9s  %9s   %s\n", "", "raw", "deflated", "size")
	row("one block (26 transactions)", len(one), deflated(one))
	row("100 blocks as one stream", len(stream), deflated(stream))
	row("one header", len(hdr), deflated(hdr))
	row("one UTXO entry", len(entry), deflated(entry))
	fmt.Println()
	fmt.Println("  Most of a block is txids, signatures and public keys, and those")
	fmt.Println("  are as close to random as bytes get — they are MEANT to be. The")
	fmt.Println("  small records a key-value store actually holds get LARGER. What")
	fmt.Println("  compression buys here is about 15% of the disk, paid for in CPU")
	fmt.Println("  on every read of every block. Bitcoin Core runs its LevelDB with")
	fmt.Println("  compression switched off. Measure yours with a benchmark before")
	fmt.Println("  adding any — and if disk is the problem, prune (example 13).")

	// ----------------------------------------------------------------- 2
	dir, err := os.MkdirTemp("", "schema-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()

	var chain []*Block
	for height := 0; height < 30; height++ {
		txs := []*Transaction{coinbase(uint64(height), pkh("miner"))}
		var parent *Block
		if height > 0 {
			parent = chain[height-1]
			prev := Outpoint{parent.Txs[0].TxID(), 0} // spend the previous coinbase
			who := []string{"alice", "bob", "carol"}[height%3]
			txs = append(txs, &Transaction{
				Inputs: []TxInput{signedInput(r, prev)},
				Outputs: []TxOutput{
					{Value: int64(height) * Coin, PubKeyHash: pkh(who)},
					{Value: (50-int64(height))*Coin - 1000, PubKeyHash: pkh("miner")},
				},
			})
		}
		chain = append(chain, &Block{Header: header(parent, txs), Txs: txs})
	}
	check(WriteChainV1(db, chain))
	schema, versions := census(db)
	fmt.Println("\n=== 2. a database written by version 1 of the program ===")
	fmt.Printf("  %d blocks, schema %d, UTXO entries by record version: 0x01 × %d\n", len(chain), schema, versions[EntryV1])
	fmt.Println("  Version 2 needs each entry's height and coinbase flag, to enforce")
	fmt.Println("  coinbase maturity (lesson 10). Version 1 never stored them.")

	var sample []byte
	check(db.View(func(tx *bolt.Tx) error {
		sample = bytes.Clone(tx.Bucket(bktUTXO).Get(utxoKey(Outpoint{chain[29].Txs[0].TxID(), 0})))
		return nil
	}))
	_, err = decodeEntry(sample)
	fmt.Printf("  the v2 decoder, handed a v1 record: %v\n", err)

	// ----------------------------------------------------------------- 3
	fmt.Println("\n=== 3. version 2 opens it, and dies halfway through migrating ===")
	n, err := Open(db, 2, 17)
	schema, versions = census(db)
	fmt.Printf("  attempt 1: %v (after rewriting %d entries)\n", err, n)
	fmt.Printf("  schema %d; entries 0x01 × %d, 0x02 × %d   <- nothing half-migrated\n", schema, versions[EntryV1], versions[EntryV2])
	n, err = Open(db, 2, -1)
	check(err)
	schema, versions = census(db)
	fmt.Printf("  attempt 2: migrated %d entries in one write transaction\n", n)
	fmt.Printf("  schema %d; entries 0x01 × %d, 0x02 × %d\n", schema, versions[EntryV1], versions[EntryV2])
	check(db.View(func(tx *bolt.Tx) error {
		for _, s := range []struct {
			what string
			op   Outpoint
		}{
			{"block 7's payment  ", Outpoint{chain[7].Txs[1].TxID(), 0}},
			{"block 29's coinbase", Outpoint{chain[29].Txs[0].TxID(), 0}},
		} {
			e, err := decodeEntry(tx.Bucket(bktUTXO).Get(utxoKey(s.op)))
			if err != nil {
				return err
			}
			fmt.Printf("  %s: %d.%08d, height %d, coinbase %v\n", s.what, e.Value/Coin, e.Value%Coin, e.Height, e.Coinbase)
		}
		return nil
	}))
	n, err = Open(db, 2, -1)
	fmt.Printf("  opened again: migrated %d, err %v — the schema byte says it is done\n", n, err)

	// ----------------------------------------------------------------- 4
	fmt.Println("\n=== 4. meeting the future ===")
	_, err = Open(db, 1, -1)
	fmt.Printf("  the v1 binary, after the upgrade:  %v\n", err)
	_, err = decodeEntry([]byte{0x03, 0, 0, 0})
	fmt.Printf("  the v2 binary, meeting 0x03:       %v\n", err)
	fmt.Println()
	fmt.Println("  Refusing is the feature. A v1 binary that 'mostly works' on a v2")
	fmt.Println("  database reads 38-byte records as 29-byte ones and writes v1 records")
	fmt.Println("  back into a v2 set. Downgrades happen: a rollback after a bad")
	fmt.Println("  release, an old container image, a second machine not yet updated.")
	fmt.Println("  go-ethereum keeps a DatabaseVersion key for exactly this reason.")
}
```

**Output:**

```
=== 1. should records be compressed? measure first ===
                                       raw   deflated   size
  one block (26 transactions)         9689       8520    87%
  100 blocks as one stream          952256     801079    84%
  one header                            92         99   107%
  one UTXO entry                        38         45   118%

  Most of a block is txids, signatures and public keys, and those
  are as close to random as bytes get — they are MEANT to be. The
  small records a key-value store actually holds get LARGER. What
  compression buys here is about 15% of the disk, paid for in CPU
  on every read of every block. Bitcoin Core runs its LevelDB with
  compression switched off. Measure yours with a benchmark before
  adding any — and if disk is the problem, prune (example 13).

=== 2. a database written by version 1 of the program ===
  30 blocks, schema 1, UTXO entries by record version: 0x01 × 59
  Version 2 needs each entry's height and coinbase flag, to enforce
  coinbase maturity (lesson 10). Version 1 never stored them.
  the v2 decoder, handed a v1 record: record is in an old format

=== 3. version 2 opens it, and dies halfway through migrating ===
  attempt 1: process died mid-migration (after rewriting 17 entries)
  schema 1; entries 0x01 × 59, 0x02 × 0   <- nothing half-migrated
  attempt 2: migrated 59 entries in one write transaction
  schema 2; entries 0x01 × 0, 0x02 × 59
  block 7's payment  : 7.00000000, height 7, coinbase false
  block 29's coinbase: 50.00000000, height 29, coinbase true
  opened again: migrated 0, err <nil> — the schema byte says it is done

=== 4. meeting the future ===
  the v1 binary, after the upgrade:  database schema is newer than this binary: it is v2, this binary understands up to v1
  the v2 binary, meeting 0x03:       unknown record version: 0x03

  Refusing is the feature. A v1 binary that 'mostly works' on a v2
  database reads 38-byte records as 29-byte ones and writes v1 records
  back into a v2 set. Downgrades happen: a rollback after a bad
  release, an old container image, a second machine not yet updated.
  go-ethereum keeps a DatabaseVersion key for exactly this reason.
```

---

## 12. Applying the same block twice

`🟡 medium` · *Crash safety*

A node will apply some block twice: a restart replays from the last checkpoint, two peers deliver the same block, a reorg reconnects one. So applying a block must be idempotent. A small balance index is kept two ways — one that trusts it will never see a replay, one that checks inside the transaction — and three blocks are replayed into both. Then `db.Batch`, the one bbolt API that runs your closure more than once on purpose.

**Steps:**

1. Apply blocks 0–5 once into two databases, one naive and one guarded, and audit both.
2. Replay blocks 3–5 into both, as a restarting node would.
3. Audit again, comparing every running total with the coins that address actually holds.
4. Send two competing blocks through `db.Batch` together, with the "announce to peers" side effect inside the closure.
5. Move the side effect to after `Batch` returns, and count again.

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
	"sync"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// Applying the same block twice.
//
// A block WILL be applied twice. A node commits block 812, crashes before it
// records that it did, and replays from 810 on restart. Two peers deliver the
// same block a millisecond apart. An operator re-runs an import. Lesson 14's
// reorgs disconnect and reconnect.
//
// So applying a block must be IDEMPOTENT: doing it twice leaves exactly the
// state doing it once did. This example keeps a small per-address balance
// index two ways, replays three blocks into each, and then shows the one
// bbolt API that runs your function twice on purpose.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   heights   height BE      hash
//   utxo      txid | vout    value [8] | pkh [20]
//   balances  pkh [20]       total [8]   — a running total, for fast lookups
//   meta      "tip"          hash
// ---------------------------------------------------------------------------

const (
	HeaderSize = 92
	Coin       = int64(100_000_000)
)

var (
	bktHeights  = []byte("heights")
	bktUTXO     = []byte("utxo")
	bktBalances = []byte("balances")
	bktMeta     = []byte("meta")
	keyTip      = []byte("tip")
)

var (
	ErrNotOnTip     = errors.New("parent is not the current tip")
	ErrMissingInput = errors.New("input is not in the UTXO set")
)

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

func (h Header) Hash() [32]byte {
	b := make([]byte, 0, HeaderSize)
	b = binary.BigEndian.AppendUint32(b, h.Version)
	b = append(b, h.PrevHash[:]...)
	b = append(b, h.MerkleRoot[:]...)
	b = binary.BigEndian.AppendUint64(b, uint64(h.Timestamp))
	b = binary.BigEndian.AppendUint32(b, h.Bits)
	b = binary.BigEndian.AppendUint32(b, h.Nonce)
	b = binary.BigEndian.AppendUint64(b, h.Height)
	f := sha256.Sum256(b)
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

func (t *Transaction) TxID() [32]byte {
	b := append(binary.BigEndian.AppendUint32(nil, uint32(len(t.Data))), t.Data...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = binary.BigEndian.AppendUint32(append(b, in.TxID[:]...), in.Index)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = append(binary.BigEndian.AppendUint64(b, uint64(o.Value)), o.PKH[:]...)
	}
	f := sha256.Sum256(b)
	return sha256.Sum256(f[:])
}

type Block struct {
	Header Header
	Txs    []*Transaction
}

// ---------------------------------------------------------------- the store

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

func addBalance(b *bolt.Bucket, pkh [20]byte, delta int64) error {
	var total int64
	if v := b.Get(pkh[:]); v != nil {
		total = int64(binary.BigEndian.Uint64(v))
	}
	return b.Put(bytes.Clone(pkh[:]), be64(uint64(total+delta)))
}

func initSchema(db *bolt.DB) error {
	return db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktHeights, bktUTXO, bktBalances, bktMeta} {
			if _, err := tx.CreateBucketIfNotExists(name); err != nil {
				return err
			}
		}
		return nil
	})
}

// applyNaive trusts the caller never to send a block twice. Every line of it
// looks reasonable in isolation.
func applyNaive(tx *bolt.Tx, b *Block) error {
	utxo, bal := tx.Bucket(bktUTXO), tx.Bucket(bktBalances)
	for _, t := range b.Txs {
		for _, in := range t.Inputs {
			if v := utxo.Get(utxoKey(in)); v != nil { // "already gone? nothing to do"
				var pkh [20]byte
				copy(pkh[:], v[8:28])
				if err := addBalance(bal, pkh, -int64(binary.BigEndian.Uint64(v[:8]))); err != nil {
					return err
				}
			}
			if err := utxo.Delete(utxoKey(in)); err != nil { // a missing key is not an error
				return err
			}
		}
		id := t.TxID()
		for vout, o := range t.Outputs {
			if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), append(be64(uint64(o.Value)), o.PKH[:]...)); err != nil {
				return err
			}
			if err := addBalance(bal, o.PKH, o.Value); err != nil { // += is not idempotent
				return err
			}
		}
	}
	hash := b.Header.Hash()
	if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
		return err
	}
	return tx.Bucket(bktMeta).Put(keyTip, hash[:])
}

// applyGuarded asks "is this exact block already connected?" INSIDE the
// transaction that would connect it, and answers yes with success. Having
// ruled replays out, it can afford to be strict about everything else.
func applyGuarded(tx *bolt.Tx, b *Block) (applied bool, err error) {
	hash := b.Header.Hash()
	if bytes.Equal(tx.Bucket(bktHeights).Get(be64(b.Header.Height)), hash[:]) {
		return false, nil // already connected: a replay is a no-op, not an error
	}
	if tip := tx.Bucket(bktMeta).Get(keyTip); b.Header.Height > 0 && !bytes.Equal(tip, b.Header.PrevHash[:]) {
		return false, ErrNotOnTip
	}
	utxo := tx.Bucket(bktUTXO)
	for i, t := range b.Txs {
		for _, in := range t.Inputs {
			if utxo.Get(utxoKey(in)) == nil {
				return false, fmt.Errorf("tx %d: %w", i, ErrMissingInput) // strict: a real gap is a real error
			}
		}
	}
	return true, applyNaive(tx, b) // the same writes, now that they are known to run once
}

// ------------------------------------------------------------- inspection

func fingerprint(db *bolt.DB) (fp [32]byte) {
	check(db.View(func(tx *bolt.Tx) error {
		h := sha256.New()
		err := tx.ForEach(func(name []byte, b *bolt.Bucket) error {
			h.Write(name)
			return b.ForEach(func(k, v []byte) error {
				h.Write(k)
				h.Write(v)
				return nil
			})
		})
		copy(fp[:], h.Sum(nil))
		return err
	}))
	return fp
}

// audit compares every running total with the sum of that address's coins.
func audit(db *bolt.DB, names map[[20]byte]string) (wrong []string) {
	check(db.View(func(tx *bolt.Tx) error {
		sums := map[[20]byte]int64{}
		if err := tx.Bucket(bktUTXO).ForEach(func(_, v []byte) error {
			sums[[20]byte(v[8:28])] += int64(binary.BigEndian.Uint64(v[:8]))
			return nil
		}); err != nil {
			return err
		}
		return tx.Bucket(bktBalances).ForEach(func(k, v []byte) error {
			if kept := int64(binary.BigEndian.Uint64(v)); kept != sums[[20]byte(k)] {
				wrong = append(wrong, fmt.Sprintf("%s: index says %s, coins add up to %s",
					names[[20]byte(k)], btc(kept), btc(sums[[20]byte(k)])))
			}
			return nil
		})
	}))
	return wrong
}

// ------------------------------------------------------------------ helpers

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func newBlock(parent *Block, miner [20]byte, txs ...*Transaction) *Block {
	h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Header.Hash()
		h.Height = parent.Header.Height + 1
		h.Timestamp = parent.Header.Timestamp + 600
	}
	cb := &Transaction{Data: be64(h.Height), Outputs: []TxOutput{{50 * Coin, miner}}}
	all := append([]*Transaction{cb}, txs...)
	h.MerkleRoot = all[len(all)-1].TxID() // stand-in: nothing here checks the tree
	return &Block{Header: h, Txs: all}
}

func pay(from Outpoint, value int64, to [20]byte, amount int64, change [20]byte) *Transaction {
	return &Transaction{Inputs: []Outpoint{from}, Outputs: []TxOutput{{amount, to}, {value - amount, change}}}
}

func btc(sat int64) string {
	sign := ""
	if sat < 0 {
		sign, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", sign, sat/Coin, sat%Coin)
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "idempotent-*")
	check(err)
	defer os.RemoveAll(dir)
	opts := &bolt.Options{Timeout: time.Second}
	open := func(name string) *bolt.DB {
		db, err := bolt.Open(filepath.Join(dir, name), 0o600, opts)
		check(err)
		check(initSchema(db))
		return db
	}

	miner, alice, bob, carol := pkh("miner"), pkh("alice"), pkh("bob"), pkh("carol")
	names := map[[20]byte]string{miner: "miner", alice: "alice", bob: "bob", carol: "carol"}

	var chain []*Block
	chain = append(chain, newBlock(nil, miner))
	chain = append(chain, newBlock(chain[0], miner, pay(Outpoint{chain[0].Txs[0].TxID(), 0}, 50*Coin, alice, 40*Coin, miner)))
	chain = append(chain, newBlock(chain[1], miner))
	chain = append(chain, newBlock(chain[2], miner, pay(Outpoint{chain[1].Txs[1].TxID(), 0}, 40*Coin, bob, 25*Coin, alice)))
	chain = append(chain, newBlock(chain[3], miner, pay(Outpoint{chain[3].Txs[1].TxID(), 0}, 25*Coin, carol, 10*Coin, bob)))
	chain = append(chain, newBlock(chain[4], miner))

	naive, guarded := open("naive.db"), open("guarded.db")
	defer naive.Close()
	defer guarded.Close()
	for _, b := range chain {
		check(naive.Update(func(tx *bolt.Tx) error { return applyNaive(tx, b) }))
		check(guarded.Update(func(tx *bolt.Tx) error { _, err := applyGuarded(tx, b); return err }))
	}
	fpNaive, fpGuarded := fingerprint(naive), fingerprint(guarded)
	fmt.Println("=== blocks 0-5 applied once, into two databases ===")
	fmt.Printf("  naive    fingerprint %x   audit: %d problems\n", fpNaive[:4], len(audit(naive, names)))
	fmt.Printf("  guarded  fingerprint %x   audit: %d problems\n", fpGuarded[:4], len(audit(guarded, names)))

	fmt.Println("\n=== the node restarts and replays from height 3 ===")
	for _, b := range chain[3:] {
		errN := naive.Update(func(tx *bolt.Tx) error { return applyNaive(tx, b) })
		var applied bool
		errG := guarded.Update(func(tx *bolt.Tx) (err error) { applied, err = applyGuarded(tx, b); return err })
		fmt.Printf("  block %d   naive: err=%v   guarded: err=%v applied=%v\n", b.Header.Height, errN, errG, applied)
	}
	fmt.Println()
	fmt.Printf("  naive    unchanged by the replay: %v\n", fingerprint(naive) == fpNaive)
	for _, w := range audit(naive, names) {
		fmt.Printf("           %s\n", w)
	}
	fmt.Printf("  guarded  unchanged by the replay: %v\n", fingerprint(guarded) == fpGuarded)
	fmt.Println()
	fmt.Println("  Nothing failed. The naive version re-credited every output of blocks")
	fmt.Println("  3-5 and skipped the debits, because those inputs were 'already gone'.")
	fmt.Println("  Puts are idempotent; deleting a missing key is silently fine; a +=")
	fmt.Println("  is neither. The guard makes replay a no-op, which then lets the")
	fmt.Println("  strict input check stay strict. (Or keep no running totals at all:")
	fmt.Println("  derive the balance from the address index, example 8, and there is")
	fmt.Println("  nothing to double.)")

	fmt.Println("\n=== db.Batch runs your function more than once ===")
	a6, b6 := newBlock(chain[5], pkh("pool-a")), newBlock(chain[5], pkh("pool-b"))
	for _, fixed := range []bool{false, true} {
		path := filepath.Join(dir, fmt.Sprintf("batch-%v.db", fixed))
		check(guarded.View(func(tx *bolt.Tx) error { return tx.CopyFile(path, 0o600) }))
		db, err := bolt.Open(path, 0o600, opts)
		check(err)
		db.MaxBatchSize = 2          // the batch runs when both calls have arrived...
		db.MaxBatchDelay = time.Hour // ...and never on a timer

		var mu sync.Mutex
		var announced, runs int
		connect := func(b *Block) error {
			var applied bool
			err := db.Batch(func(tx *bolt.Tx) (err error) {
				mu.Lock()
				runs++
				mu.Unlock()
				if applied, err = applyGuarded(tx, b); err != nil {
					return err
				}
				if !fixed {
					mu.Lock()
					announced++ // "tell peers about the block" — from inside the closure
					mu.Unlock()
				}
				return nil
			})
			if err == nil && applied && fixed {
				mu.Lock()
				announced++ // after Batch has returned nil: the commit really happened
				mu.Unlock()
			}
			return err
		}

		var wg sync.WaitGroup
		errs := make([]error, 2)
		for i, b := range []*Block{a6, b6} {
			wg.Add(1)
			go func() {
				defer wg.Done()
				errs[i] = connect(b)
			}()
		}
		wg.Wait()
		connected := 0
		for _, err := range errs {
			if err == nil {
				connected++
			} else if !errors.Is(err, ErrNotOnTip) {
				panic(err)
			}
		}
		label := "announce inside the closure"
		if fixed {
			label = "announce after Batch returns"
		}
		fmt.Printf("  %-29s  closures run %d, blocks connected %d, announcements %d\n", label, runs, connected, announced)
		check(db.Close())
	}
	fmt.Println()
	fmt.Println("  Two competing blocks landed in one batch. The second failed its tip")
	fmt.Println("  check, so bbolt rolled the whole batch back, re-ran the first on its")
	fmt.Println("  own, and re-ran the second solo to hand back its error. Four runs for")
	fmt.Println("  two calls. Batch's doc says it plainly: fn 'may be called multiple")
	fmt.Println("  times, regardless of whether it returns error or not'.")
	fmt.Println()
	fmt.Println("  So: nothing inside a transaction closure may have an effect outside")
	fmt.Println("  the database. Not a message, not a metric, not a cache write. Do it")
	fmt.Println("  after the call returns nil — true for Update, and doubly for Batch.")
}
```

**Output:**

```
=== blocks 0-5 applied once, into two databases ===
  naive    fingerprint c335b601   audit: 0 problems
  guarded  fingerprint c335b601   audit: 0 problems

=== the node restarts and replays from height 3 ===
  block 3   naive: err=<nil>   guarded: err=<nil> applied=false
  block 4   naive: err=<nil>   guarded: err=<nil> applied=false
  block 5   naive: err=<nil>   guarded: err=<nil> applied=false

  naive    unchanged by the replay: false
           alice: index says 30.00000000, coins add up to 15.00000000
           carol: index says 20.00000000, coins add up to 10.00000000
           bob: index says 30.00000000, coins add up to 15.00000000
           miner: index says 410.00000000, coins add up to 260.00000000
  guarded  unchanged by the replay: true

  Nothing failed. The naive version re-credited every output of blocks
  3-5 and skipped the debits, because those inputs were 'already gone'.
  Puts are idempotent; deleting a missing key is silently fine; a +=
  is neither. The guard makes replay a no-op, which then lets the
  strict input check stay strict. (Or keep no running totals at all:
  derive the balance from the address index, example 8, and there is
  nothing to double.)

=== db.Batch runs your function more than once ===
  announce inside the closure    closures run 4, blocks connected 1, announcements 2
  announce after Batch returns   closures run 4, blocks connected 1, announcements 1

  Two competing blocks landed in one batch. The second failed its tip
  check, so bbolt rolled the whole batch back, re-ran the first on its
  own, and re-ran the second solo to hand back its error. Four runs for
  two calls. Batch's doc says it plainly: fn 'may be called multiple
  times, regardless of whether it returns error or not'.

  So: nothing inside a transaction closure may have an effect outside
  the database. Not a message, not a metric, not a cache write. Do it
  after the call returns nil — true for Update, and doubly for Batch.
```

---

## 13. Pruning: what a node can still answer

`🟡 medium` · *Pruning and snapshots*

A node needs the UTXO set to validate and the headers to know which chain it is on; old block bodies and a transaction index only answer questions about the past. An archive node is asked six questions, pruned to its last 20 bodies with its index dropped, and asked the same six again. Then the file on disk, which does not get any smaller when you delete from it.

**Steps:**

1. Build a 120-block archive node with headers, bodies, a UTXO set and a transaction index.
2. Ask it six questions, from "alice's balance now" to "alice's balance at height 50".
3. Prune the bodies below height 100 in resumable chunks, and drop the transaction index.
4. Ask the same six questions, and compare bucket sizes and capabilities side by side.
5. Compare the file size before pruning, after pruning, and after `bolt.Compact`.

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
// Pruning: what a node can still answer.
//
// To validate the next block a node needs the UTXO set, and to know which
// chain it is on it needs the headers. It needs no old block body for either.
// Everything else — old bodies, a transaction index — exists to answer
// questions about the PAST, usually on someone else's behalf.
//
// This example builds an archive node, asks it six questions, prunes it to
// the last 20 bodies, and asks again. Then it looks at the file on disk,
// which did not get any smaller.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key               value
//   -------   ---------------   ----------------------------------------------
//   headers   hash              version [1] | header [92]          kept forever
//   bodies    hash              count [4] | transactions           PRUNABLE
//   heights   height BE         hash
//   utxo      txid | vout       value [8] | pkh [20]
//   txindex   txid              block hash                         archive only
//   meta      "tip"             hash
//             "pruned-below"    height BE: no bodies below this, on purpose
// ---------------------------------------------------------------------------

const (
	HeaderSize     = 92
	HeaderRecordV1 = 0x01
	Coin           = int64(100_000_000)
	PageSize       = 4096 // pinned, so the file sizes below match on every machine

	Blocks     = 120
	TxPerBlock = 20
	KeepBodies = 20 // Bitcoin Core's floor is 288: two days of blocks
)

var (
	bktHeaders = []byte("headers")
	bktBodies  = []byte("bodies")
	bktHeights = []byte("heights")
	bktUTXO    = []byte("utxo")
	bktTxIndex = []byte("txindex")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
	keyPruned  = []byte("pruned-below")
)

var (
	ErrPruned       = errors.New("block body pruned")
	ErrNotFound     = errors.New("not found")
	ErrNoIndex      = errors.New("this node keeps no transaction index")
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrBrokenLink   = errors.New("header chain broken")
	ErrTruncated    = errors.New("record truncated")
)

// ------------------------------------------------- blocks & txs (08, 10)

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

type TxInput struct {
	Prev      Outpoint
	Signature []byte
	PubKey    []byte
}

type TxOutput struct {
	Value int64
	PKH   [20]byte
}

type Transaction struct {
	Data    []byte
	Inputs  []TxInput
	Outputs []TxOutput
}

func appendBytes(b, p []byte) []byte {
	return append(binary.BigEndian.AppendUint32(b, uint32(len(p))), p...)
}

func (t *Transaction) Serialize() []byte {
	b := appendBytes(nil, t.Data)
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b = append(b, in.Prev.TxID[:]...)
		b = binary.BigEndian.AppendUint32(b, in.Prev.Index)
		b = appendBytes(b, in.Signature)
		b = appendBytes(b, in.PubKey)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		b = binary.BigEndian.AppendUint64(b, uint64(o.Value))
		b = append(b, o.PKH[:]...)
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

// ------------------------------------------------------------- the records

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

func (r *reader) u64() uint64 {
	if v := r.take(8); v != nil {
		return binary.BigEndian.Uint64(v)
	}
	return 0
}

func (r *reader) bytes() []byte { return bytes.Clone(r.take(int(r.u32()))) }

func encodeBody(txs []*Transaction) []byte {
	b := binary.BigEndian.AppendUint32(nil, uint32(len(txs)))
	for _, t := range txs {
		b = append(b, t.Serialize()...)
	}
	return b
}

func decodeBody(p []byte) ([]*Transaction, error) {
	r := &reader{p: p}
	var txs []*Transaction
	for n := r.u32(); n > 0 && r.err == nil; n-- {
		t := &Transaction{Data: r.bytes()}
		for m := r.u32(); m > 0 && r.err == nil; m-- {
			var in TxInput
			copy(in.Prev.TxID[:], r.take(32))
			in.Prev.Index = r.u32()
			in.Signature = r.bytes()
			in.PubKey = r.bytes()
			t.Inputs = append(t.Inputs, in)
		}
		for m := r.u32(); m > 0 && r.err == nil; m-- {
			var o TxOutput
			o.Value = int64(r.u64())
			copy(o.PKH[:], r.take(20))
			t.Outputs = append(t.Outputs, o)
		}
		txs = append(txs, t)
	}
	return txs, r.err
}

func decodeHeader(v []byte) (Header, error) {
	if len(v) != 1+HeaderSize || v[0] != HeaderRecordV1 {
		return Header{}, fmt.Errorf("%w: header record", ErrTruncated)
	}
	r := &reader{p: v[1:]}
	var h Header
	h.Version = r.u32()
	copy(h.PrevHash[:], r.take(32))
	copy(h.MerkleRoot[:], r.take(32))
	h.Timestamp = int64(r.u64())
	h.Bits = r.u32()
	h.Nonce = r.u32()
	h.Height = r.u64()
	return h, r.err
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func utxoKey(op Outpoint) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(op.TxID[:]), op.Index)
}

// ---------------------------------------------------------------- the store

func initSchema(db *bolt.DB) error {
	return db.Update(func(tx *bolt.Tx) error {
		for _, name := range [][]byte{bktHeaders, bktBodies, bktHeights, bktUTXO, bktTxIndex, bktMeta} {
			if _, err := tx.CreateBucket(name); err != nil {
				return err
			}
		}
		return nil
	})
}

func connect(db *bolt.DB, b *Block) error {
	hash := b.Header.Hash()
	return db.Update(func(tx *bolt.Tx) error {
		utxo, idx := tx.Bucket(bktUTXO), tx.Bucket(bktTxIndex)
		for i, t := range b.Txs {
			for _, in := range t.Inputs {
				if utxo.Get(utxoKey(in.Prev)) == nil {
					return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
				}
				if err := utxo.Delete(utxoKey(in.Prev)); err != nil {
					return err
				}
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				v := append(be64(uint64(o.Value)), o.PKH[:]...)
				if err := utxo.Put(utxoKey(Outpoint{id, uint32(vout)}), v); err != nil {
					return err
				}
			}
			if idx != nil {
				if err := idx.Put(id[:], hash[:]); err != nil {
					return err
				}
			}
		}
		if err := tx.Bucket(bktHeaders).Put(hash[:], append([]byte{HeaderRecordV1}, b.Header.Bytes()...)); err != nil {
			return err
		}
		if err := tx.Bucket(bktBodies).Put(hash[:], encodeBody(b.Txs)); err != nil {
			return err
		}
		if err := tx.Bucket(bktHeights).Put(be64(b.Header.Height), hash[:]); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, hash[:])
	})
}

// Prune deletes bodies below tip+1-keep, `chunk` heights per write
// transaction. Each chunk moves pruned-below in the same transaction as its
// deletes, so a crash between chunks leaves a node that knows exactly what
// it still has, and the next call carries on from there.
func Prune(db *bolt.DB, keep, chunk uint64) (commits int, err error) {
	for {
		done := false
		err := db.Update(func(tx *bolt.Tx) error {
			tip, err := decodeHeader(tx.Bucket(bktHeaders).Get(tx.Bucket(bktMeta).Get(keyTip)))
			if err != nil {
				return err
			}
			from, limit := prunedBelow(tx), uint64(0)
			if tip.Height+1 > keep {
				limit = tip.Height + 1 - keep
			}
			if from >= limit {
				done = true
				return nil
			}
			to := min(from+chunk, limit)
			for h := from; h < to; h++ {
				if err := tx.Bucket(bktBodies).Delete(tx.Bucket(bktHeights).Get(be64(h))); err != nil {
					return err
				}
			}
			return tx.Bucket(bktMeta).Put(keyPruned, be64(to))
		})
		if err != nil || done {
			return commits, err
		}
		commits++
	}
}

// ---------------------------------------------------------------- questions

func prunedBelow(tx *bolt.Tx) uint64 {
	if v := tx.Bucket(bktMeta).Get(keyPruned); v != nil {
		return binary.BigEndian.Uint64(v)
	}
	return 0
}

func Balance(tx *bolt.Tx, who [20]byte) (sum int64) {
	_ = tx.Bucket(bktUTXO).ForEach(func(_, v []byte) error {
		if [20]byte(v[8:28]) == who {
			sum += int64(binary.BigEndian.Uint64(v[:8]))
		}
		return nil
	})
	return sum
}

func CheckInputs(tx *bolt.Tx, b *Block) error {
	for i, t := range b.Txs {
		for _, in := range t.Inputs {
			if tx.Bucket(bktUTXO).Get(utxoKey(in.Prev)) == nil {
				return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
			}
		}
	}
	return nil
}

func VerifyHeaders(tx *bolt.Tx) (links int, err error) {
	c := tx.Bucket(bktHeights).Cursor()
	var prev [32]byte
	for k, hash := c.First(); k != nil; k, hash = c.Next() {
		h, err := decodeHeader(tx.Bucket(bktHeaders).Get(hash))
		if err != nil {
			return links, err
		}
		if h.Height > 0 {
			if h.PrevHash != prev {
				return links, ErrBrokenLink
			}
			links++
		}
		prev = h.Hash()
	}
	return links, nil
}

// BlockAt tells "pruned on purpose" apart from "missing": a peer asking for
// block 10 should hear "I don't keep it", not "it doesn't exist".
func BlockAt(tx *bolt.Tx, height uint64) ([]*Transaction, error) {
	hash := tx.Bucket(bktHeights).Get(be64(height))
	if hash == nil {
		return nil, fmt.Errorf("%w: height %d", ErrNotFound, height)
	}
	body := tx.Bucket(bktBodies).Get(hash)
	if body == nil {
		if below := prunedBelow(tx); height < below {
			return nil, fmt.Errorf("%w: this node keeps bodies from height %d", ErrPruned, below)
		}
		return nil, fmt.Errorf("%w: body at height %d — that is corruption, not policy", ErrNotFound, height)
	}
	return decodeBody(body)
}

func TxByID(tx *bolt.Tx, id [32]byte) (uint64, error) {
	idx := tx.Bucket(bktTxIndex)
	if idx == nil {
		return 0, ErrNoIndex
	}
	hash := idx.Get(id[:])
	if hash == nil {
		return 0, ErrNotFound
	}
	h, err := decodeHeader(tx.Bucket(bktHeaders).Get(hash))
	return h.Height, err
}

// BalanceAt replays bodies from genesis. Without a separate history index,
// that is the only way to answer a question about the past.
func BalanceAt(tx *bolt.Tx, who [20]byte, height uint64) (int64, error) {
	coins := map[Outpoint]TxOutput{}
	for h := uint64(0); h <= height; h++ {
		txs, err := BlockAt(tx, h)
		if err != nil {
			return 0, err
		}
		for _, t := range txs {
			for _, in := range t.Inputs {
				delete(coins, in.Prev)
			}
			id := t.TxID()
			for vout, o := range t.Outputs {
				coins[Outpoint{id, uint32(vout)}] = o
			}
		}
	}
	var sum int64
	for _, o := range coins {
		if o.PKH == who {
			sum += o.Value
		}
	}
	return sum, nil
}

// ------------------------------------------------------------------ helpers

type rng struct{ n uint64 }

func (r *rng) bytes(n int) []byte {
	var out []byte
	for len(out) < n {
		r.n++
		s := sha256.Sum256(binary.BigEndian.AppendUint64(nil, r.n))
		out = append(out, s[:]...)
	}
	return out[:n]
}

func (r *rng) intn(n int) int { return int(binary.BigEndian.Uint64(r.bytes(8)) % uint64(n)) }

func pkh(name string) (p [20]byte) {
	s := sha256.Sum256([]byte(name))
	copy(p[:], s[:20])
	return p
}

func btc(sat int64) string { return fmt.Sprintf("%d.%08d", sat/Coin, sat%Coin) }

func logicalSizes(db *bolt.DB) map[string]int {
	sizes := map[string]int{}
	check(db.View(func(tx *bolt.Tx) error {
		return tx.ForEach(func(name []byte, b *bolt.Bucket) error {
			return b.ForEach(func(k, v []byte) error {
				sizes[string(name)] += len(k) + len(v)
				return nil
			})
		})
	}))
	return sizes
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
	dir, err := os.MkdirTemp("", "prune-*")
	check(err)
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "chain.db")
	opts := &bolt.Options{Timeout: time.Second, PageSize: PageSize}
	db, err := bolt.Open(path, 0o600, opts)
	check(err)
	defer db.Close()
	check(initSchema(db))

	// ------------------------------------------------ an archive node's chain
	r := &rng{}
	miner, alice := pkh("miner"), pkh("alice")
	people := []string{"alice", "bob", "carol", "dave", "erin", "frank"}
	type coin struct {
		op  Outpoint
		out TxOutput
	}
	var pool []coin
	var parent *Block
	var block10 *Block
	makeBlock := func(nTx int) *Block {
		h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
		if parent != nil {
			h.PrevHash, h.Height, h.Timestamp = parent.Header.Hash(), parent.Header.Height+1, parent.Header.Timestamp+600
		}
		txs := []*Transaction{{Data: be64(h.Height), Outputs: []TxOutput{{50 * Coin, miner}}}}
		for i := 0; i < nTx && len(pool) > 0; i++ {
			j := r.intn(len(pool))
			c := pool[j]
			pool[j], pool = pool[len(pool)-1], pool[:len(pool)-1]
			if c.out.Value < 100_000 {
				continue
			}
			amount := c.out.Value / int64(2+r.intn(8))
			txs = append(txs, &Transaction{
				Inputs: []TxInput{{Prev: c.op, Signature: r.bytes(65), PubKey: append([]byte{0x02}, r.bytes(32)...)}},
				Outputs: []TxOutput{
					{amount, pkh(people[r.intn(len(people))])},
					{c.out.Value - amount - 1000, c.out.PKH},
				},
			})
		}
		h.MerkleRoot = sha256.Sum256(encodeBody(txs)) // a flat commitment; lesson 05 has the tree
		return &Block{Header: h, Txs: txs}
	}
	for height := 0; height < Blocks; height++ {
		b := makeBlock(TxPerBlock)
		check(connect(db, b))
		for _, t := range b.Txs { // new coins become spendable from the next block
			id := t.TxID()
			for vout, o := range t.Outputs {
				pool = append(pool, coin{Outpoint{id, uint32(vout)}, o})
			}
		}
		if height == 10 {
			block10 = b
		}
		parent = b
	}
	next := makeBlock(5) // block 120: not connected, just checked
	wanted := block10.Txs[3].TxID()

	ask := func(title string) {
		fmt.Printf("\n=== six questions for the %s ===\n", title)
		check(db.View(func(tx *bolt.Tx) error {
			answer := func(q string, a any, err error) {
				if err != nil {
					a = err
				}
				fmt.Printf("  %-38s %v\n", q, a)
			}
			answer("alice's balance now", btc(Balance(tx, alice)), nil)
			answer("are block 120's inputs unspent?", "yes", CheckInputs(tx, next))
			links, err := VerifyHeaders(tx)
			answer("header chain intact from genesis?", fmt.Sprintf("yes, %d links", links), err)
			txs, err := BlockAt(tx, 10)
			answer("send block 10 to a syncing peer", fmt.Sprintf("%d transactions", len(txs)), err)
			h, err := TxByID(tx, wanted)
			answer(fmt.Sprintf("which block holds tx %x…?", wanted[:4]), fmt.Sprintf("height %d", h), err)
			bal, err := BalanceAt(tx, alice, 50)
			answer("alice's balance at height 50", btc(bal), err)
			return nil
		}))
	}

	ask("archive node")
	before := logicalSizes(db)
	sizeBefore := fileSize(path)

	fmt.Printf("\n=== prune to the last %d bodies, and drop the transaction index ===\n", KeepBodies)
	commits, err := Prune(db, KeepBodies, 25)
	check(err)
	check(db.Update(func(tx *bolt.Tx) error { return tx.DeleteBucket(bktTxIndex) }))
	fmt.Printf("  %d write transactions of 25 heights each, pruned-below moving with each\n", commits)
	commits, err = Prune(db, KeepBodies, 25)
	check(err)
	fmt.Printf("  run it again: %d more — resumable, and idempotent\n", commits)

	ask("pruned node")
	check(db.View(func(tx *bolt.Tx) error {
		txs, err := BlockAt(tx, 110)
		check(err)
		fmt.Printf("  %-38s %d transactions\n", "(send block 110 instead)", len(txs))
		return nil
	}))
	after := logicalSizes(db)

	fmt.Println("\n=== the trade ===")
	fmt.Printf("  %-9s %9s %9s\n", "bucket", "archive", "pruned")
	var tb, ta int
	for _, name := range []string{"headers", "heights", "utxo", "meta", "bodies", "txindex"} {
		fmt.Printf("  %-9s %9d %9d\n", name, before[name], after[name])
		tb += before[name]
		ta += after[name]
	}
	fmt.Printf("  %-9s %9d %9d   bytes of keys and values (%d%%)\n", "total", tb, ta, ta*100/tb)
	fmt.Println()
	fmt.Println("                      archive    pruned")
	fmt.Println("  validate new blocks   yes        yes")
	fmt.Println("  current balances      yes        yes")
	fmt.Println("  verify the headers    yes        yes")
	fmt.Println("  serve old blocks      yes        last 20 only")
	fmt.Println("  find a tx by id       yes        no")
	fmt.Println("  balance in the past   yes        no")
	fmt.Println("  reorg deeper than 20  yes        no — needs the old bodies (lesson 14)")

	fmt.Println("\n=== and the file ===")
	fmt.Printf("  before pruning     %8d bytes\n", sizeBefore)
	fmt.Printf("  after pruning      %8d bytes   (the freed pages are still inside it)\n", fileSize(path))
	compacted := filepath.Join(dir, "compacted.db")
	dst, err := bolt.Open(compacted, 0o600, opts)
	check(err)
	check(bolt.Compact(dst, db, 1<<20))
	check(dst.Close())
	fmt.Printf("  after bolt.Compact %8d bytes   (a new file, written from scratch)\n", fileSize(compacted))
	fmt.Println()
	fmt.Println("  bbolt never hands pages back to the OS. Pruned pages go on its")
	fmt.Println("  freelist and are reused by later writes, so the file stops growing")
	fmt.Println("  but does not shrink. Getting the disk back means copying the live")
	fmt.Println("  data into a new file — offline, with room for both copies. LSM")
	fmt.Println("  stores (LevelDB, Pebble) reclaim space in background compaction")
	fmt.Println("  instead, which is one reason big nodes use them.")
}
```

**Output:**

```

=== six questions for the archive node ===
  alice's balance now                    202.51084922
  are block 120's inputs unspent?        yes
  header chain intact from genesis?      yes, 119 links
  send block 10 to a syncing peer        21 transactions
  which block holds tx ac5556c7…?        height 10
  alice's balance at height 50           123.63215448

=== prune to the last 20 bodies, and drop the transaction index ===
  4 write transactions of 25 heights each, pruned-below moving with each
  run it again: 0 more — resumable, and idempotent

=== six questions for the pruned node ===
  alice's balance now                    202.51084922
  are block 120's inputs unspent?        yes
  header chain intact from genesis?      yes, 119 links
  send block 10 to a syncing peer        block body pruned: this node keeps bodies from height 100
  which block holds tx ac5556c7…?        this node keeps no transaction index
  alice's balance at height 50           block body pruned: this node keeps bodies from height 100
  (send block 110 instead)               20 transactions

=== the trade ===
  bucket      archive    pruned
  headers       15000     15000
  heights        4800      4800
  utxo         143360    143360
  meta             35        55
  bodies       455280     77280
  txindex      143360         0
  total        761835    240495   bytes of keys and values (31%)

                      archive    pruned
  validate new blocks   yes        yes
  current balances      yes        yes
  verify the headers    yes        yes
  serve old blocks      yes        last 20 only
  find a tx by id       yes        no
  balance in the past   yes        no
  reorg deeper than 20  yes        no — needs the old bodies (lesson 14)

=== and the file ===
  before pruning      2097152 bytes
  after pruning       2097152 bytes   (the freed pages are still inside it)
  after bolt.Compact   524288 bytes   (a new file, written from scratch)

  bbolt never hands pages back to the OS. Pruned pages go on its
  freelist and are reused by later writes, so the file stops growing
  but does not shrink. Getting the disk back means copying the live
  data into a new file — offline, with room for both copies. LSM
  stores (LevelDB, Pebble) reclaim space in background compaction
  instead, which is one reason big nodes use them.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)
