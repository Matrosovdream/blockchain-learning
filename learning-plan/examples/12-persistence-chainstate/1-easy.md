# Step 12 — Persistence & Chain State · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
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
exactly. Where a file size is printed the page size is pinned to 4096 bytes, because bbolt
otherwise uses the operating system's — 16384 on Apple Silicon. Example 5 needs no dependencies.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. A block on disk

`🟢 easy` · *Why a key-value store*

Everything a node asks of its storage is one of four things: write a block once, read it by hash, walk blocks by height, scan unspent outputs by prefix. None of them is a query, and all of them are a key-value store's job description. One bbolt file, one bucket, the block hash as the key and lesson 08's encoding behind a version byte as the value — then the two bbolt behaviours that bite first.

**Steps:**

1. Build a two-block chain from lesson 10's transactions, and lay block 1 out as a key and a value.
2. Store both blocks in a `blocks` bucket, close the file, and look at its size in pages.
3. Reopen it, fetch block 1 by hash, and check the decoded header still hashes to its key.
4. Ask for a hash that was never stored, and turn bbolt's `nil` into a typed error.
5. Open the file a second time while it is open, and watch `Options.Timeout` save the process.

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
// A block on disk.
//
// Everything a node asks of its storage is one of four things:
//
//   1. write a block once, and never change it
//   2. read a block by its hash
//   3. walk blocks by height                       (examples 2, 3, 6)
//   4. scan unspent outputs by prefix              (example 8)
//
// No joins, no ad-hoc queries, nothing a SQL planner could help with. That
// is a key-value store's job description. This example does the first two
// with bbolt: one file, one bucket, the block hash as the key and the
// block's deterministic encoding (lesson 08) as the value.
// ===========================================================================

const (
	HeaderSize    = 92
	BlockRecordV1 = 0x01 // a version byte on every record (example 11)
	Coin          = int64(100_000_000)

	// bbolt uses the OS page size by default: 4096 on most Linux machines,
	// 16384 on Apple Silicon. Pinned so the sizes below match everywhere.
	PageSize = 4096
)

var bucketBlocks = []byte("blocks")

var (
	ErrNotFound  = errors.New("block not found")
	ErrNoBucket  = errors.New("no blocks bucket: database not initialised")
	ErrVersion   = errors.New("unknown record version")
	ErrTruncated = errors.New("record truncated")
	ErrTrailing  = errors.New("trailing bytes after record")
)

// ---------------------------------------------------- header & txs (08, 10)

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

// ------------------------------------------------------ the record on disk

type Block struct {
	Header Header
	Txs    []*Transaction
}

// Encode is the storage format: the SAME bytes that get hashed, behind a
// version byte. One format for hashing and for disk means there is no
// second encoder to drift out of sync with the first.
func (b *Block) Encode() []byte {
	out := []byte{BlockRecordV1}
	out = append(out, b.Header.Bytes()...)
	out = binary.BigEndian.AppendUint32(out, uint32(len(b.Txs)))
	for _, t := range b.Txs {
		out = appendBytes(out, t.Serialize())
	}
	return out
}

// reader walks a record and remembers the first error, so the decoder does
// not have to check after every field. Every []byte it hands out is a COPY:
// the input may be memory that belongs to the database (example 4).
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
	b := &Block{Header: decodeHeader(r)}
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

func open(path string) (*bolt.DB, error) {
	return bolt.Open(path, 0o600, &bolt.Options{
		Timeout:  100 * time.Millisecond, // never wait forever for the file lock
		PageSize: PageSize,
	})
}

func putBlock(db *bolt.DB, b *Block) error {
	hash := b.Header.Hash()
	return db.Update(func(tx *bolt.Tx) error {
		bk, err := tx.CreateBucketIfNotExists(bucketBlocks)
		if err != nil {
			return err
		}
		return bk.Put(hash[:], b.Encode())
	})
}

func getBlock(db *bolt.DB, hash [32]byte) (*Block, error) {
	var blk *Block
	err := db.View(func(tx *bolt.Tx) error {
		bk := tx.Bucket(bucketBlocks)
		if bk == nil {
			return ErrNoBucket
		}
		v := bk.Get(hash[:])
		if v == nil {
			return fmt.Errorf("%w: %x…", ErrNotFound, hash[:4])
		}
		var err error
		blk, err = DecodeBlock(v) // decoded into fresh memory before the tx ends
		return err
	})
	return blk, err
}

// ------------------------------------------------------------------ helpers

func newBlock(parent *Header, txs ...*Transaction) *Block {
	h := Header{Version: 1, MerkleRoot: MerkleRoot(txs), Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Hash()
		h.Height = parent.Height + 1
		h.Timestamp = parent.Timestamp + 600
	}
	// No proof of work: storage does not care, and lesson 09 already did it.
	return &Block{Header: h, Txs: txs}
}

// addr stands in for HASH160(pubkey) (lesson 07). Storage never looks inside.
func addr(name string) []byte {
	s := sha256.Sum256([]byte(name))
	return s[:20]
}

// fake returns n deterministic bytes: stand-in signatures and keys.
func fake(n int, label string) []byte {
	var out []byte
	for i := 0; len(out) < n; i++ {
		s := sha256.Sum256([]byte(fmt.Sprintf("%s-%d", label, i)))
		out = append(out, s[:]...)
	}
	return out[:n]
}

func btc(sat int64) string { return fmt.Sprintf("%d.%08d", sat/Coin, sat%Coin) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "chain-*")
	check(err)
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "chain.db")

	miner, alice := addr("miner"), addr("alice")
	genesis := newBlock(nil, NewCoinbase(0, miner, 50*Coin))
	b1 := newBlock(&genesis.Header,
		NewCoinbase(1, miner, 50*Coin+1000),
		&Transaction{
			Inputs: []TxInput{{
				Prev:      Outpoint{genesis.Txs[0].TxID(), 0},
				Signature: fake(65, "sig"), // stand-ins: storage never verifies them
				PubKey:    fake(33, "pub"),
				Sequence:  0xfffffffd,
			}},
			Outputs: []TxOutput{
				{Value: 30 * Coin, PubKeyHash: alice},
				{Value: 20*Coin - 1000, PubKeyHash: miner},
			},
		})

	hash := b1.Header.Hash()
	value := b1.Encode()
	body := len(value) - 1 - HeaderSize - 4

	fmt.Println("=== one block, as a key and a value ===")
	fmt.Printf("  key    block hash     %3d bytes  %x\n", len(hash), hash)
	fmt.Printf("  value  version byte   %3d byte   0x%02x\n", 1, value[0])
	fmt.Printf("         header         %3d bytes  height %d\n", HeaderSize, b1.Header.Height)
	fmt.Printf("         tx count       %3d bytes  %d\n", 4, len(b1.Txs))
	fmt.Printf("         transactions   %3d bytes  each behind a length prefix\n", body)
	fmt.Printf("         total          %3d bytes\n", len(value))

	fmt.Println("\n=== write both blocks, close the file ===")
	db, err := open(path)
	check(err)
	check(putBlock(db, genesis))
	check(putBlock(db, b1))
	check(db.Close())
	fi, err := os.Stat(path)
	check(err)
	fmt.Printf("  %s: %d bytes = %d pages of %d\n", filepath.Base(path), fi.Size(), fi.Size()/PageSize, PageSize)
	fmt.Println("  The process that wrote them is finished with the file. The blocks")
	fmt.Println("  are in it, and nothing else is needed to read them back.")

	fmt.Println("\n=== reopen, read block 1 back by its hash ===")
	db, err = open(path)
	check(err)
	defer db.Close()
	got, err := getBlock(db, hash)
	check(err)
	fmt.Printf("  decoded: height %d, %d transactions, pays alice %s\n",
		got.Header.Height, len(got.Txs), btc(got.Txs[1].Outputs[0].Value))
	fmt.Printf("  re-encoding equals the stored bytes : %v\n", bytes.Equal(got.Encode(), value))
	fmt.Printf("  hash of the decoded header          : %x\n", got.Header.Hash())
	fmt.Printf("  equals the key it was stored under  : %v\n", got.Header.Hash() == hash)
	fmt.Println("  The key is not an arbitrary id. It is a hash of the value, so a record")
	fmt.Println("  that has rotted on disk is caught the moment anyone reads it.")

	fmt.Println("\n=== a hash that was never stored ===")
	missing := sha256.Sum256([]byte("no such block"))
	_, err = getBlock(db, missing)
	fmt.Printf("  getBlock: %v\n", err)
	fmt.Printf("  errors.Is(err, ErrNotFound): %v\n", errors.Is(err, ErrNotFound))
	fmt.Println("  bbolt's Get returns nil, not an error, for a missing key. Turn that")
	fmt.Println("  into a typed error once, at the storage boundary, so no caller ever")
	fmt.Println("  decodes a nil slice into an empty block.")

	fmt.Println("\n=== a second process tries to open the same file ===")
	_, err = open(path)
	fmt.Printf("  bolt.Open: %v\n", err)
	fmt.Println("  bbolt holds an exclusive lock on the file while it is open. Without")
	fmt.Println("  Options.Timeout that Open blocks forever — which is what 'my node")
	fmt.Println("  hangs at startup' usually is: the old process has not exited yet.")
}
```

**Output:**

```
=== one block, as a key and a value ===
  key    block hash      32 bytes  3b1493298fa2190571473da86d52b0feb7879fe128f9401a68606339a9188e89
  value  version byte     1 byte   0x01
         header          92 bytes  height 1
         tx count         4 bytes  2
         transactions   322 bytes  each behind a length prefix
         total          419 bytes

=== write both blocks, close the file ===
  chain.db: 32768 bytes = 8 pages of 4096
  The process that wrote them is finished with the file. The blocks
  are in it, and nothing else is needed to read them back.

=== reopen, read block 1 back by its hash ===
  decoded: height 1, 2 transactions, pays alice 30.00000000
  re-encoding equals the stored bytes : true
  hash of the decoded header          : 3b1493298fa2190571473da86d52b0feb7879fe128f9401a68606339a9188e89
  equals the key it was stored under  : true
  The key is not an arbitrary id. It is a hash of the value, so a record
  that has rotted on disk is caught the moment anyone reads it.

=== a hash that was never stored ===
  getBlock: block not found: cef1d353…
  errors.Is(err, ErrNotFound): true
  bbolt's Get returns nil, not an error, for a missing key. Turn that
  into a typed error once, at the storage boundary, so no caller ever
  decodes a nil slice into an empty block.

=== a second process tries to open the same file ===
  bolt.Open: timeout
  bbolt holds an exclusive lock on the file while it is open. Without
  Options.Timeout that Open blocks forever — which is what 'my node
  hangs at startup' usually is: the old process has not exited yet.
```

---

## 2. Heights as keys

`🟢 easy` · *Key-space design*

An ordered key-value store sorts keys as byte strings and nothing else, so the bytes chosen for a number decide whether key order is numeric order. The same ten heights go into three buckets — as decimal strings, as little-endian and as big-endian `uint64` — and the same cursor walks and range scans run over each. Only one of them is right, and the other two are wrong at different heights.

**Steps:**

1. Insert ten heights, out of order, into one bucket per encoding.
2. Walk each bucket with a cursor and compare the order with numeric order.
3. Ask each bucket for `Last()`, which is the question "what is the highest height?".
4. Run two range scans, 8..11 and 250..257, against all three.
5. Read where each one breaks: decimal at 10, little-endian at 256, big-endian never.

```go
package main

import (
	"encoding/binary"
	"fmt"
	"os"
	"path/filepath"
	"slices"
	"strconv"
	"time"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// Heights as keys.
//
// bbolt, LevelDB, Pebble — every ordered key-value store sorts keys as BYTE
// STRINGS, lexicographically, and nothing else. So the bytes you choose for
// a number decide whether "walk the keys in order" means "walk the heights
// in order".
//
// The same ten heights under three encodings, three buckets, the same scans.
// ===========================================================================

type encoding struct {
	name   string
	bucket []byte
	key    func(uint64) []byte
}

var encodings = []encoding{
	{"decimal string", []byte("decimal"), func(h uint64) []byte {
		return []byte(strconv.FormatUint(h, 10))
	}},
	{"little-endian uint64", []byte("little"), func(h uint64) []byte {
		return binary.LittleEndian.AppendUint64(nil, h)
	}},
	{"big-endian uint64", []byte("big"), func(h uint64) []byte {
		return binary.BigEndian.AppendUint64(nil, h)
	}},
}

// Inserted out of order on purpose: a B+tree keeps keys sorted, not arrivals.
var heights = []uint64{9, 10, 2, 255, 1, 256, 11, 0, 257, 8}

// all walks a bucket in key order. The VALUE is always the height as 8
// big-endian bytes, so every bucket can say what it holds.
func all(b *bolt.Bucket) []uint64 {
	var out []uint64
	c := b.Cursor()
	for k, v := c.First(); k != nil; k, v = c.Next() {
		out = append(out, height(v))
	}
	return out
}

// scan is how every "blocks from X to Y" query is written: seek to the first
// key, walk forward, stop once past the end.
func scan(b *bolt.Bucket, from []byte, to uint64) []uint64 {
	var out []uint64
	c := b.Cursor()
	for k, v := c.Seek(from); k != nil; k, v = c.Next() {
		h := height(v)
		if h > to {
			break
		}
		out = append(out, h)
	}
	return out
}

func height(v []byte) uint64 { return binary.BigEndian.Uint64(v) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "heights-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "index.db"), 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()

	check(db.Update(func(tx *bolt.Tx) error {
		for _, e := range encodings {
			b, err := tx.CreateBucket(e.bucket)
			if err != nil {
				return err
			}
			for _, h := range heights {
				if err := b.Put(e.key(h), binary.BigEndian.AppendUint64(nil, h)); err != nil {
					return err
				}
			}
		}
		return nil
	}))

	fmt.Printf("inserted in this order: %v\n", heights)

	check(db.View(func(tx *bolt.Tx) error {
		for _, e := range encodings {
			b := tx.Bucket(e.bucket)
			order := all(b)
			_, last := b.Cursor().Last()

			fmt.Printf("\n=== %s ===\n", e.name)
			fmt.Printf("  key for 10      %x\n", e.key(10))
			fmt.Printf("  key for 256     %x\n", e.key(256))
			fmt.Printf("  cursor order    %v\n", order)
			fmt.Printf("  numeric order?  %v\n", slices.IsSorted(order))
			fmt.Printf("  Last()          %d   <- \"the highest height\"\n", height(last))
			fmt.Printf("  range 8..11     %v\n", scan(b, e.key(8), 11))
			fmt.Printf("  range 250..257  %v\n", scan(b, e.key(250), 257))
		}
		return nil
	}))

	fmt.Println("\n=== what went wrong, and where ===")
	fmt.Println("  decimal       \"10\" < \"9\" because '1' < '9'. Last() answers 9, and a")
	fmt.Println("                range scan both misses blocks and includes wrong ones.")
	fmt.Println("                Invisible below height 10, which is exactly the part")
	fmt.Println("                of the chain every test covers.")
	fmt.Println("  little-endian The least significant byte is compared first, so 256")
	fmt.Println("                (00 01 …) sorts next to 0. Correct up to 255, then")
	fmt.Println("                wrong forever — a bug that waits for block 256.")
	fmt.Println("  big-endian    Most significant byte first: byte order IS numeric")
	fmt.Println("                order, for every uint64. Fixed width is what makes")
	fmt.Println("                it work — a varint sorts no better than decimal.")
	fmt.Println()
	fmt.Println("  go-ethereum keys its headers as \"h\" + number (8 bytes, big-endian)")
	fmt.Println("  + hash, so every header at one height sits together, in order.")
}
```

**Output:**

```
inserted in this order: [9 10 2 255 1 256 11 0 257 8]

=== decimal string ===
  key for 10      3130
  key for 256     323536
  cursor order    [0 1 10 11 2 255 256 257 8 9]
  numeric order?  false
  Last()          9   <- "the highest height"
  range 8..11     [8 9]
  range 250..257  [255 256 257 8 9]

=== little-endian uint64 ===
  key for 10      0a00000000000000
  key for 256     0001000000000000
  cursor order    [0 256 1 257 2 8 9 10 11 255]
  numeric order?  false
  Last()          255   <- "the highest height"
  range 8..11     [8 9 10 11]
  range 250..257  [255]

=== big-endian uint64 ===
  key for 10      000000000000000a
  key for 256     0000000000000100
  cursor order    [0 1 2 8 9 10 11 255 256 257]
  numeric order?  true
  Last()          257   <- "the highest height"
  range 8..11     [8 9 10 11]
  range 250..257  [255 256 257]

=== what went wrong, and where ===
  decimal       "10" < "9" because '1' < '9'. Last() answers 9, and a
                range scan both misses blocks and includes wrong ones.
                Invisible below height 10, which is exactly the part
                of the chain every test covers.
  little-endian The least significant byte is compared first, so 256
                (00 01 …) sorts next to 0. Correct up to 255, then
                wrong forever — a bug that waits for block 256.
  big-endian    Most significant byte first: byte order IS numeric
                order, for every uint64. Fixed width is what makes
                it work — a varint sorts no better than decimal.

  go-ethereum keys its headers as "h" + number (8 bytes, big-endian)
  + hash, so every header at one height sits together, in order.
```

---

## 3. The height index and the tip pointer

`🟢 easy` · *Key-space design*

A node rarely knows a block's hash; it asks for the tip, or for a height. That takes two more pieces of schema — a height index and a tip pointer — and a distinction that matters from here to lesson 14: the blocks a node has *stored* are not the chain it is *on*. The tip pointer is a commit marker, not a summary of what is in the file.

**Steps:**

1. Write the schema as a comment block, before the code that uses it.
2. Connect five headers, each with its height entry and the tip pointer, in one write transaction apiece.
3. Store two more without connecting them: a stale sibling of block 3, and a block 5 still being validated.
4. Answer `Tip()` and `AtHeight(3)` in two lookups each.
5. Derive the tip from "the highest block stored" instead, and watch it come out wrong.

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
// The height index and the tip pointer.
//
// Example 1 could fetch a block if you already knew its hash. A node almost
// never does. It asks "what is the tip?" and "what is block 812,000?", so
// the schema needs two more things: an index from height to hash, and one
// key that names the block the rest of the state reflects.
// ===========================================================================

// ---------------------------------------------------------------- the schema
//
//   bucket    key               value
//   -------   ---------------   ------------------------------------------
//   blocks    hash      [32]    version byte + header                [93]
//   heights   height [8] BE     hash of the CANONICAL block there    [32]
//   meta      "tip"             hash of the block the state reflects [32]
//
//   `blocks` holds every block we have: stale branches, blocks received but
//   not yet validated. `heights` and `meta` describe only the chain we are
//   ON, and they change together, in one write transaction, or not at all.
//
// Write this comment before the code. In six months it is the only
// documentation the bytes will have.
// ---------------------------------------------------------------------------

const (
	HeaderSize     = 92
	HeaderRecordV1 = 0x01
)

var (
	bktBlocks  = []byte("blocks")
	bktHeights = []byte("heights")
	bktMeta    = []byte("meta")
	keyTip     = []byte("tip")
)

var (
	ErrNotFound = errors.New("not found")
	ErrNotOnTip = errors.New("parent is not the current tip")
	ErrCorrupt  = errors.New("corrupt header record")
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

func encodeHeader(h Header) []byte { return append([]byte{HeaderRecordV1}, h.Bytes()...) }

// decodeHeader copies every field into a Header value. Arrays copy; nothing
// in the result points back into the database's memory.
func decodeHeader(v []byte) (Header, error) {
	if len(v) != 1+HeaderSize {
		return Header{}, fmt.Errorf("%w: %d bytes", ErrCorrupt, len(v))
	}
	if v[0] != HeaderRecordV1 {
		return Header{}, fmt.Errorf("%w: version 0x%02x", ErrCorrupt, v[0])
	}
	p := v[1:]
	var h Header
	h.Version = binary.BigEndian.Uint32(p[0:4])
	copy(h.PrevHash[:], p[4:36])
	copy(h.MerkleRoot[:], p[36:68])
	h.Timestamp = int64(binary.BigEndian.Uint64(p[68:76]))
	h.Bits = binary.BigEndian.Uint32(p[76:80])
	h.Nonce = binary.BigEndian.Uint32(p[80:84])
	h.Height = binary.BigEndian.Uint64(p[84:92])
	return h, nil
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

// ---------------------------------------------------------------- the store

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

// store writes a block WITHOUT making it part of the chain.
func store(tx *bolt.Tx, h Header) error {
	hash := h.Hash()
	return tx.Bucket(bktBlocks).Put(hash[:], encodeHeader(h))
}

// connect makes h the new tip: the block, its height entry and the tip
// pointer, in ONE write transaction.
func connect(db *bolt.DB, h Header) error {
	hash := h.Hash()
	return db.Update(func(tx *bolt.Tx) error {
		tip := tx.Bucket(bktMeta).Get(keyTip)
		if h.Height > 0 && !bytes.Equal(tip, h.PrevHash[:]) {
			return ErrNotOnTip // lesson 14 decides what to do with it instead
		}
		if err := store(tx, h); err != nil {
			return err
		}
		if err := tx.Bucket(bktHeights).Put(be64(h.Height), hash[:]); err != nil {
			return err
		}
		return tx.Bucket(bktMeta).Put(keyTip, hash[:])
	})
}

func headerByHash(tx *bolt.Tx, hash []byte) (Header, error) {
	v := tx.Bucket(bktBlocks).Get(hash)
	if v == nil {
		return Header{}, fmt.Errorf("%w: block %x…", ErrNotFound, hash[:4])
	}
	return decodeHeader(v)
}

// Tip: two lookups, whatever the height of the chain.
func Tip(db *bolt.DB) (h Header, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		hash := tx.Bucket(bktMeta).Get(keyTip)
		if hash == nil {
			return fmt.Errorf("%w: empty chain", ErrNotFound)
		}
		h, err = headerByHash(tx, hash)
		return err
	})
	return h, err
}

// AtHeight: two lookups, whatever the height asked for.
func AtHeight(db *bolt.DB, height uint64) (h Header, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		hash := tx.Bucket(bktHeights).Get(be64(height))
		if hash == nil {
			return fmt.Errorf("%w: nothing canonical at height %d", ErrNotFound, height)
		}
		h, err = headerByHash(tx, hash)
		return err
	})
	return h, err
}

// highestStored is the tempting wrong answer to "what is the tip?".
func highestStored(db *bolt.DB) (best Header, scanned int, err error) {
	err = db.View(func(tx *bolt.Tx) error {
		return tx.Bucket(bktBlocks).ForEach(func(_, v []byte) error {
			h, err := decodeHeader(v)
			if err != nil {
				return err
			}
			scanned++
			if h.Height > best.Height {
				best = h
			}
			return nil
		})
	})
	return best, scanned, err
}

// ------------------------------------------------------------------ helpers

func mkHeader(parent *Header, branch string) Header {
	h := Header{Version: 1, Timestamp: 1700000000, Bits: 0x2000ffff}
	if parent != nil {
		h.PrevHash = parent.Hash()
		h.Height = parent.Height + 1
		h.Timestamp = parent.Timestamp + 600
	}
	h.MerkleRoot = sha256.Sum256([]byte(fmt.Sprintf("%s-%d", branch, h.Height)))
	return h
}

func short(h [32]byte) string { return fmt.Sprintf("%x", h[:6]) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "index-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{Timeout: time.Second})
	check(err)
	defer db.Close()
	check(initSchema(db))

	fmt.Println("=== five blocks connected, one write transaction each ===")
	var chain []Header
	for i := 0; i < 5; i++ {
		var parent *Header
		if i > 0 {
			parent = &chain[i-1]
		}
		h := mkHeader(parent, "main")
		check(connect(db, h))
		chain = append(chain, h)
		fmt.Printf("  height %d  %s   heights[%x] -> %s\n", h.Height, short(h.Hash()), be64(h.Height), short(h.Hash()))
	}

	fmt.Println("\n=== two more blocks arrive: stored, not connected ===")
	stale := mkHeader(&chain[2], "stale") // a sibling of block 3: someone else's branch
	next := mkHeader(&chain[4], "main")   // block 5: its body is still being validated
	check(db.Update(func(tx *bolt.Tx) error {
		if err := store(tx, stale); err != nil {
			return err
		}
		return store(tx, next)
	}))
	fmt.Printf("  stale sibling of block 3   %s\n", short(stale.Hash()))
	fmt.Printf("  block 5, not yet validated %s\n", short(next.Hash()))
	fmt.Printf("  connect(stale sibling):    %v\n", connect(db, stale))

	fmt.Println("\n=== asking the store ===")
	tip, err := Tip(db)
	check(err)
	fmt.Printf("  Tip()        height %d  %s   meta, then blocks\n", tip.Height, short(tip.Hash()))
	at3, err := AtHeight(db, 3)
	check(err)
	fmt.Printf("  AtHeight(3)  height %d  %s   heights, then blocks\n", at3.Height, short(at3.Hash()))
	fmt.Printf("               the canonical block 3, not the sibling: %v\n", at3.Hash() == chain[3].Hash())
	_, err = AtHeight(db, 5)
	fmt.Printf("  AtHeight(5)  %v\n", err)

	var nBlocks, nHeights int
	check(db.View(func(tx *bolt.Tx) error {
		nBlocks = tx.Bucket(bktBlocks).Stats().KeyN
		nHeights = tx.Bucket(bktHeights).Stats().KeyN
		return nil
	}))
	fmt.Printf("\n  blocks stored   : %d\n", nBlocks)
	fmt.Printf("  heights indexed : %d\n", nHeights)
	fmt.Println("  Two lookups for the tip, two for any height — at height 5 or at")
	fmt.Println("  height 900,000. Neither depends on how many blocks are stored.")

	fmt.Println("\n=== the tempting wrong answer ===")
	best, scanned, err := highestStored(db)
	check(err)
	fmt.Printf("  highest block stored: height %d %s, found by decoding all %d blocks\n",
		best.Height, short(best.Hash()), scanned)
	fmt.Printf("  is it the tip?        %v\n", best.Hash() == tip.Hash())
	fmt.Println()
	fmt.Println("  'The highest block I have' and 'the block my state reflects' are")
	fmt.Println("  different questions. Block 5 is stored and not connected; a stale")
	fmt.Println("  sibling is stored at height 3 and never will be. Deriving the tip")
	fmt.Println("  from what is stored gets it wrong in both directions, and costs a")
	fmt.Println("  full scan to do it.")
	fmt.Println()
	fmt.Println("  The tip pointer is the COMMIT MARKER: the one key that says which")
	fmt.Println("  block every other piece of state corresponds to. Bitcoin Core keeps")
	fmt.Println("  exactly this beside its UTXO set — the best-block key in chainstate.")
}
```

**Output:**

```
=== five blocks connected, one write transaction each ===
  height 0  db7b39104eb2   heights[0000000000000000] -> db7b39104eb2
  height 1  b84d9c39420a   heights[0000000000000001] -> b84d9c39420a
  height 2  d1cbd5ab23f3   heights[0000000000000002] -> d1cbd5ab23f3
  height 3  4c1ff434982a   heights[0000000000000003] -> 4c1ff434982a
  height 4  fe25f19c884a   heights[0000000000000004] -> fe25f19c884a

=== two more blocks arrive: stored, not connected ===
  stale sibling of block 3   7cda1b6fffe5
  block 5, not yet validated f16950075172
  connect(stale sibling):    parent is not the current tip

=== asking the store ===
  Tip()        height 4  fe25f19c884a   meta, then blocks
  AtHeight(3)  height 3  4c1ff434982a   heights, then blocks
               the canonical block 3, not the sibling: true
  AtHeight(5)  not found: nothing canonical at height 5

  blocks stored   : 7
  heights indexed : 5
  Two lookups for the tip, two for any height — at height 5 or at
  height 900,000. Neither depends on how many blocks are stored.

=== the tempting wrong answer ===
  highest block stored: height 5 f16950075172, found by decoding all 7 blocks
  is it the tip?        false

  'The highest block I have' and 'the block my state reflects' are
  different questions. Block 5 is stored and not connected; a stale
  sibling is stored at height 3 and never will be. Deriving the tip
  from what is stored gets it wrong in both directions, and costs a
  full scan to do it.

  The tip pointer is the COMMIT MARKER: the one key that says which
  block every other piece of state corresponds to. Bitcoin Core keeps
  exactly this beside its UTXO set — the best-block key in chainstate.
```

---

## 4. Copying a value out of a transaction

`🟢 easy` · *Iterators and cursors*

bbolt memory-maps its file and `Get` returns a slice that points straight into the map — not a copy of the record, the record. That slice is valid only until the transaction ends, and the compiler cannot tell you when you have kept it longer. This example makes the ownership visible, shows which decoders quietly keep a pointer into the map, and then shows `Put` borrowing *your* memory in exactly the same way.

**Steps:**

1. Store a UTXO entry among a hundred others, so the bucket lives on pages of its own.
2. Ask whether the slice `Get` returned points inside bbolt's memory map.
3. Decode it three ways and check which results still point there.
4. Read the kept values after the transaction has ended, and see nothing wrong — yet.
5. Reuse one buffer for three `Put`s, and watch every value come back as the last one.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"os"
	"path/filepath"
	"time"
	"unsafe"

	bolt "go.etcd.io/bbolt"
)

// ===========================================================================
// Copying a value out of a transaction.
//
// bbolt does not copy data out of the file. It memory-maps the file and
// hands you slices that point straight into the map — zero-copy reads are
// most of why it is fast. The price is written in Get's doc comment:
//
//     The returned value is only valid for the life of the transaction.
//     The returned memory is owned by bbolt and must never be modified.
//
// The compiler checks neither sentence. This example makes the ownership
// visible, shows which decoders keep a pointer into the map, and then shows
// the same rule running the other way, on Put.
// ===========================================================================

const (
	RecordV1 = 0x01
	Coin     = int64(100_000_000)
)

var (
	bktUTXO    = []byte("utxo")
	bktHeights = []byte("heights")
)

// UTXOEntry is lesson 10's entry. On disk:
//
//	version [1] | value [8] | height [8] | coinbase [1] | len [4] | pkh [len]
type UTXOEntry struct {
	Value      int64
	Height     uint64
	Coinbase   bool
	PubKeyHash []byte
}

func encodeEntry(e UTXOEntry) []byte {
	b := []byte{RecordV1}
	b = binary.BigEndian.AppendUint64(b, uint64(e.Value))
	b = binary.BigEndian.AppendUint64(b, e.Height)
	if e.Coinbase {
		b = append(b, 1)
	} else {
		b = append(b, 0)
	}
	b = binary.BigEndian.AppendUint32(b, uint32(len(e.PubKeyHash)))
	return append(b, e.PubKeyHash...)
}

// decodeAliased looks like a decoder and is a bug. The integers are copied,
// but PubKeyHash is a window onto the database's memory, not a value.
func decodeAliased(v []byte) UTXOEntry {
	return UTXOEntry{
		Value:      int64(binary.BigEndian.Uint64(v[1:9])),
		Height:     binary.BigEndian.Uint64(v[9:17]),
		Coinbase:   v[17] == 1,
		PubKeyHash: v[22:],
	}
}

// decodeCopied is the same decoder with one call added.
func decodeCopied(v []byte) UTXOEntry {
	return UTXOEntry{
		Value:      int64(binary.BigEndian.Uint64(v[1:9])),
		Height:     binary.BigEndian.Uint64(v[9:17]),
		Coinbase:   v[17] == 1,
		PubKeyHash: bytes.Clone(v[22:]),
	}
}

// inMap reports whether p points into the database file's memory map. It
// peeks at bbolt's internals with unsafe and exists only to make the point
// visible. Never ship anything like it.
func inMap(db *bolt.DB, tx *bolt.Tx, p []byte) bool {
	if len(p) == 0 {
		return false
	}
	start := db.Info().Data
	addr := uintptr(unsafe.Pointer(unsafe.SliceData(p)))
	return addr >= start && addr < start+uintptr(tx.Size())
}

func outpointKey(txid [32]byte, index uint32) []byte {
	return binary.BigEndian.AppendUint32(bytes.Clone(txid[:]), index)
}

func be64(n uint64) []byte { return binary.BigEndian.AppendUint64(nil, n) }

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	dir, err := os.MkdirTemp("", "copy-*")
	check(err)
	defer os.RemoveAll(dir)
	db, err := bolt.Open(filepath.Join(dir, "chain.db"), 0o600, &bolt.Options{Timeout: time.Second, PageSize: 4096})
	check(err)
	defer db.Close()

	key := outpointKey(sha256.Sum256([]byte("some transaction")), 0)
	alice := bytes.Repeat([]byte{0xa1}, 20)
	check(db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucket(bktUTXO)
		if err != nil {
			return err
		}
		// A hundred other outputs first. A bucket holding only a few bytes is
		// stored INLINE inside its parent's page, and bbolt copies inline values
		// that are misaligned — which would hide everything below. No real UTXO
		// set is that small.
		for i := 0; i < 100; i++ {
			other := outpointKey(sha256.Sum256([]byte(fmt.Sprintf("other-%d", i))), 0)
			e := UTXOEntry{Value: int64(i+1) * Coin, Height: 3, PubKeyHash: bytes.Repeat([]byte{0xbb}, 20)}
			if err := b.Put(other, encodeEntry(e)); err != nil {
				return err
			}
		}
		return b.Put(key, encodeEntry(UTXOEntry{Value: 50 * Coin, Height: 7, Coinbase: true, PubKeyHash: alice}))
	}))

	var raw []byte
	var aliased, copied UTXOEntry
	check(db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket(bktUTXO).Get(key)

		fmt.Println("=== 1. what Get hands you ===")
		fmt.Printf("  a %d-byte slice. Inside bbolt's memory map of the file: %v\n", len(v), inMap(db, tx, v))
		fmt.Println("  Not a copy of the record. The record.")

		raw = v
		aliased = decodeAliased(v)
		copied = decodeCopied(v)

		fmt.Println("\n=== 2. what each decoder keeps hold of ===")
		fmt.Printf("  %-40s -> in the map: %v\n", "keep v itself", inMap(db, tx, raw))
		fmt.Printf("  %-40s -> in the map: %v\n", "e.PubKeyHash = v[22:]", inMap(db, tx, aliased.PubKeyHash))
		fmt.Printf("  %-40s -> in the map: %v\n", "e.PubKeyHash = bytes.Clone(v[22:])", inMap(db, tx, copied.PubKeyHash))
		fmt.Printf("  %-40s -> copied by the assignment\n", "e.Value = int64(Uint64(v[1:9]))")
		fmt.Printf("  %-40s -> copied by the assignment\n", "copy(h.PrevHash[:], v[4:36])")
		fmt.Println("  Integers and arrays copy when you assign them. Slices never do:")
		fmt.Println("  slicing a slice makes a second window onto the same memory.")
		return nil
	}))

	fmt.Println("\n=== 3. the transaction is over, and nothing looks wrong ===")
	fmt.Printf("  raw value       %d\n", int64(binary.BigEndian.Uint64(raw[1:9])))
	fmt.Printf("  aliased pkh     %x…\n", aliased.PubKeyHash[:4])
	fmt.Printf("  copied pkh      %x…\n", copied.PubKeyHash[:4])
	fmt.Println("  All three still read correctly, because nothing has written to the")
	fmt.Println("  file since. That is exactly why this bug passes its tests: it needs")
	fmt.Println("  a LATER write to reuse the page, and example 7 supplies one.")

	fmt.Println("\n=== 4. the rule runs the other way on Put ===")
	buf := make([]byte, 8) // one buffer, reused for every value: an ordinary "optimisation"
	check(db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucket(bktHeights)
		if err != nil {
			return err
		}
		for h := uint64(1); h <= 3; h++ {
			binary.BigEndian.PutUint64(buf, h*100)
			if err := b.Put(be64(h), buf); err != nil {
				return err
			}
		}
		return nil
	}))
	printHeights(db, "reused buffer")

	check(db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket(bktHeights)
		for h := uint64(1); h <= 3; h++ {
			if err := b.Put(be64(h), be64(h*100)); err != nil { // a fresh slice per Put
				return err
			}
		}
		return nil
	}))
	printHeights(db, "fresh slice each")
	fmt.Println("  Put's doc: \"Supplied value must remain valid for the life of the")
	fmt.Println("  transaction.\" bbolt copies the key but keeps a reference to the")
	fmt.Println("  value until commit, when it finally writes the page — by which")
	fmt.Println("  time the buffer holds the last thing you put in it.")

	fmt.Println("\n=== the rule, both directions ===")
	fmt.Println("  Anything bbolt gives you is borrowed until the transaction ends.")
	fmt.Println("  Anything you give bbolt is borrowed until the transaction ends.")
	fmt.Println("  Copy at the boundary — bytes.Clone out of Get, a fresh slice into")
	fmt.Println("  Put — and nothing above the storage layer ever has to know.")
}

func printHeights(db *bolt.DB, label string) {
	check(db.View(func(tx *bolt.Tx) error {
		fmt.Printf("  %-17s", label)
		c := tx.Bucket(bktHeights).Cursor()
		for k, v := c.First(); k != nil; k, v = c.Next() {
			fmt.Printf("  height %d -> %d", binary.BigEndian.Uint64(k), binary.BigEndian.Uint64(v))
		}
		fmt.Println()
		return nil
	}))
}
```

**Output:**

```
=== 1. what Get hands you ===
  a 42-byte slice. Inside bbolt's memory map of the file: true
  Not a copy of the record. The record.

=== 2. what each decoder keeps hold of ===
  keep v itself                            -> in the map: true
  e.PubKeyHash = v[22:]                    -> in the map: true
  e.PubKeyHash = bytes.Clone(v[22:])       -> in the map: false
  e.Value = int64(Uint64(v[1:9]))          -> copied by the assignment
  copy(h.PrevHash[:], v[4:36])             -> copied by the assignment
  Integers and arrays copy when you assign them. Slices never do:
  slicing a slice makes a second window onto the same memory.

=== 3. the transaction is over, and nothing looks wrong ===
  raw value       5000000000
  aliased pkh     a1a1a1a1…
  copied pkh      a1a1a1a1…
  All three still read correctly, because nothing has written to the
  file since. That is exactly why this bug passes its tests: it needs
  a LATER write to reuse the page, and example 7 supplies one.

=== 4. the rule runs the other way on Put ===
  reused buffer      height 1 -> 300  height 2 -> 300  height 3 -> 300
  fresh slice each   height 1 -> 100  height 2 -> 200  height 3 -> 300
  Put's doc: "Supplied value must remain valid for the life of the
  transaction." bbolt copies the key but keeps a reference to the
  value until commit, when it finally writes the page — by which
  time the buffer holds the last thing you put in it.

=== the rule, both directions ===
  Anything bbolt gives you is borrowed until the transaction ends.
  Anything you give bbolt is borrowed until the transaction ends.
  Copy at the boundary — bytes.Clone out of Get, a fresh slice into
  Put — and nothing above the storage layer ever has to know.
```

---

## 5. Why not gob

`🟢 easy` · *Serialization on disk*

`encoding/gob` round-trips any Go value in one line, and it is the wrong format for anything that outlives the process that wrote it. Five failures, each demonstrated beside lesson 08's fixed-layout encoder doing the same job: names stored in every record, a rename that silently loses money, zero values that are never written, bytes that depend on the previous record, and maps that encode differently every time.

**Steps:**

1. Encode one UTXO entry with gob and with lesson 08's encoder, and compare what is in the bytes.
2. Rename a field and decode the old bytes into the new type.
3. Decode a normal output, then a zero-value output, into the same reused struct.
4. Encode one value twice on one encoder, and try to decode the second copy by itself.
5. Encode a value holding a map a hundred times and ask whether the bytes ever agree.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"encoding/gob"
	"errors"
	"fmt"
)

// ===========================================================================
// Why not gob.
//
// encoding/gob round-trips any Go value in one line, and it is the wrong
// format for anything that outlives the process that wrote it. Five ways it
// goes wrong on disk, each demonstrated beside the lesson-08 encoder doing
// the same job.
//
// No database needed: a key-value store keeps whatever bytes you hand it,
// and every one of these problems is in the bytes.
// ===========================================================================

const Coin = int64(100_000_000)

// OutputV1 is a UTXO entry as version 1 of the program wrote it...
type OutputV1 struct {
	Value      int64
	PubKeyHash []byte
}

// ...and OutputV2 is how version 2 reads it, after a rename in code review.
type OutputV2 struct {
	Amount     int64
	PubKeyHash []byte
}

var errShort = errors.New("record truncated")

// gobRecord encodes ONE value with its own encoder — which is what storing it
// under its own key forces, since every value must decode without the others.
func gobRecord(v any) []byte {
	var b bytes.Buffer
	if err := gob.NewEncoder(&b).Encode(v); err != nil {
		panic(err)
	}
	return b.Bytes()
}

func gobDecode(p []byte, v any) error {
	return gob.NewDecoder(bytes.NewReader(p)).Decode(v)
}

// encodeOutput is the lesson-08 way: fixed order, fixed widths, big-endian,
// a length prefix on the variable part. No names, no state between records.
func encodeOutput(value int64, pkh []byte) []byte {
	b := binary.BigEndian.AppendUint64(nil, uint64(value))
	b = binary.BigEndian.AppendUint32(b, uint32(len(pkh)))
	return append(b, pkh...)
}

// decodeOutput assigns EVERY field, every time.
func decodeOutput(p []byte, o *OutputV2) error {
	if len(p) < 12 || len(p) != 12+int(binary.BigEndian.Uint32(p[8:12])) {
		return errShort
	}
	o.Amount = int64(binary.BigEndian.Uint64(p[:8]))
	o.PubKeyHash = bytes.Clone(p[12:])
	return nil
}

func ascii(p []byte) string {
	out := make([]byte, len(p))
	for i, c := range p {
		out[i] = '.'
		if c >= 0x20 && c < 0x7f {
			out[i] = c
		}
	}
	return string(out)
}

func main() {
	alice := bytes.Repeat([]byte{0xa1}, 20)
	g := gobRecord(OutputV1{Value: 50 * Coin, PubKeyHash: alice})
	d := encodeOutput(50*Coin, alice)

	fmt.Println("=== 1. the field names are in every record ===")
	fmt.Printf("  gob        %3d bytes  %s\n", len(g), ascii(g))
	fmt.Printf("  lesson 08  %3d bytes\n", len(d))
	extra := int64(len(g) - len(d))
	total := extra * 100_000_000 // at a hundred million UTXOs
	fmt.Printf("  %d extra bytes × 100 million UTXOs = %d.%d GB of type descriptions\n",
		extra, total/1_000_000_000, total/100_000_000%10)

	fmt.Println("\n=== 2. rename a field, lose the money ===")
	var v2 OutputV2
	err := gobDecode(g, &v2)
	fmt.Printf("  gob        V1 bytes into V2: err=%v  Amount=%d\n", err, v2.Amount)
	var w2 OutputV2
	err = decodeOutput(d, &w2)
	fmt.Printf("  lesson 08  V1 bytes into V2: err=%v  Amount=%d\n", err, w2.Amount)
	fmt.Println("  gob matches fields by NAME and quietly skips what it cannot place.")
	fmt.Println("  Nothing fails. Fifty coins read back as zero. The position-based")
	fmt.Println("  format never stored a name, so a rename cannot touch it.")

	fmt.Println("\n=== 3. a zero is never written ===")
	// A zero-value output is ordinary: Bitcoin's OP_RETURN data outputs.
	dataOut := OutputV1{Value: 0, PubKeyHash: []byte("data")}
	var e OutputV1 // one struct, reused across a cursor loop — the normal Go idiom
	for i, rec := range [][]byte{g, gobRecord(dataOut)} {
		if err := gobDecode(rec, &e); err != nil {
			panic(err)
		}
		fmt.Printf("  gob        record %d into the reused struct: Value=%d\n", i+1, e.Value)
	}
	var f OutputV2
	for i, rec := range [][]byte{d, encodeOutput(dataOut.Value, dataOut.PubKeyHash)} {
		if err := decodeOutput(rec, &f); err != nil {
			panic(err)
		}
		fmt.Printf("  lesson 08  record %d into the reused struct: Value=%d\n", i+1, f.Amount)
	}
	fmt.Println("  gob omits zero-valued fields and never clears the destination, so")
	fmt.Println("  a 0-value output decodes as the 50 coins before it.")

	fmt.Println("\n=== 4. the bytes depend on what the encoder sent before ===")
	var stream bytes.Buffer
	enc := gob.NewEncoder(&stream)
	same := OutputV1{Value: 50 * Coin, PubKeyHash: alice}
	if err := enc.Encode(same); err != nil {
		panic(err)
	}
	first := stream.Len()
	if err := enc.Encode(same); err != nil {
		panic(err)
	}
	second := stream.Len() - first
	fmt.Printf("  same value, same encoder: %d bytes, then %d bytes\n", first, second)
	var lone OutputV1
	err = gobDecode(stream.Bytes()[first:], &lone)
	fmt.Printf("  decode the second record on its own: %v\n", err)
	fmt.Println("  A gob stream is a conversation, not a sequence of records. Store")
	fmt.Println("  the second value under its own key and it cannot be read.")

	fmt.Println("\n=== 5. maps come out in whatever order they iterate ===")
	type Balances struct{ ByAddr map[string]int64 }
	bal := Balances{map[string]int64{"alice": 30, "bob": 20, "carol": 5, "dave": 1, "erin": 7}}
	distinct := map[string]bool{}
	for i := 0; i < 100; i++ {
		distinct[string(gobRecord(bal))] = true
	}
	fmt.Printf("  one value encoded 100 times, always the same bytes: %v\n", len(distinct) == 1)
	fmt.Println("  Harmless in a cache. Fatal in anything you hash, compare or dedupe.")

	fmt.Println("\n=== and the one that is not in the bytes ===")
	fmt.Println("  gob has exactly one implementation, in Go. A block explorer in")
	fmt.Println("  Rust, an analytics job in Python, or your own node rewritten in")
	fmt.Println("  five years cannot read the database. The lesson-08 format fits in")
	fmt.Println("  a comment block, and anything with a byte buffer can parse it.")
}
```

**Output:**

```
=== 1. the field names are in every record ===
  gob         80 bytes  ......OutputV1........Value.....PubKeyHash..... .....T..........................
  lesson 08   32 bytes
  48 extra bytes × 100 million UTXOs = 4.8 GB of type descriptions

=== 2. rename a field, lose the money ===
  gob        V1 bytes into V2: err=<nil>  Amount=0
  lesson 08  V1 bytes into V2: err=<nil>  Amount=5000000000
  gob matches fields by NAME and quietly skips what it cannot place.
  Nothing fails. Fifty coins read back as zero. The position-based
  format never stored a name, so a rename cannot touch it.

=== 3. a zero is never written ===
  gob        record 1 into the reused struct: Value=5000000000
  gob        record 2 into the reused struct: Value=5000000000
  lesson 08  record 1 into the reused struct: Value=5000000000
  lesson 08  record 2 into the reused struct: Value=0
  gob omits zero-valued fields and never clears the destination, so
  a 0-value output decodes as the 50 coins before it.

=== 4. the bytes depend on what the encoder sent before ===
  same value, same encoder: 80 bytes, then 33 bytes
  decode the second record on its own: gob: unknown type id or corrupted data
  A gob stream is a conversation, not a sequence of records. Store
  the second value under its own key and it cannot be read.

=== 5. maps come out in whatever order they iterate ===
  one value encoded 100 times, always the same bytes: false
  Harmless in a cache. Fatal in anything you hash, compare or dedupe.

=== and the one that is not in the bytes ===
  gob has exactly one implementation, in Go. A block explorer in
  Rust, an analytics job in Python, or your own node rewritten in
  five years cannot read the database. The lesson-08 format fits in
  a comment block, and anything with a byte buffer can parse it.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)
