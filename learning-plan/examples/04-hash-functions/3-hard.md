# Step 04 — Cryptographic Hash Functions · 🔴 Hard

Examples **15–20**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get golang.org/x/crypto@latest               # examples 5, 7, 9, 20
go get github.com/ethereum/go-ethereum@latest   # examples 5, 14
# paste the example into main.go, then:
go run .
```

No chain, no node, no keys. Examples 1–4, 6, 8, 10–13 and 15–19 need only the standard library.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 15. One hashing rule for many types

`🔴 hard` · *Determinism*

Once several types need hashing, the rule has to live in one place or it drifts. A `Hashable` interface exposing `preimage()` — rather than `Hash()` — means every type declares its bytes and the single `HashOf` decides how they are hashed.

**Steps:**

1. Define `Hashable` with an unexported `preimage() []byte` method.
2. Give each concrete type its own domain tag and a fixed-layout preimage.
3. Length-prefix the variable field so `"ab"` and `"abc"` cannot slide into each other.
4. Confirm a Mint and a Burn with identical fields do not collide.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// Hashable is the contract: every type that gets hashed must say exactly which
// bytes represent it. Putting preimage() in the interface — rather than Hash()
// — means the hashing rule lives in ONE place and cannot drift per type.
type Hashable interface {
	preimage() []byte
}

// Domain tags. One per concrete type, assigned once and never reused.
const (
	tagTransfer byte = 0x20
	tagMint     byte = 0x21
	tagBurn     byte = 0x22
)

// HashOf is the single hashing rule for the whole system.
func HashOf(h Hashable) [32]byte { return sha256.Sum256(h.preimage()) }

// appendBytes length-prefixes variable-length data. Without this,
// ("ab","c") and ("a","bc") would produce identical preimages.
func appendBytes(dst []byte, b []byte) []byte {
	dst = binary.BigEndian.AppendUint32(dst, uint32(len(b)))
	return append(dst, b...)
}

type Transfer struct {
	From, To [20]byte
	Amount   uint64
	Memo     string
}

func (t Transfer) preimage() []byte {
	b := []byte{tagTransfer}
	b = append(b, t.From[:]...)
	b = append(b, t.To[:]...)
	b = binary.BigEndian.AppendUint64(b, t.Amount)
	return appendBytes(b, []byte(t.Memo))
}

type Mint struct {
	To     [20]byte
	Amount uint64
}

func (m Mint) preimage() []byte {
	b := []byte{tagMint}
	b = append(b, m.To[:]...)
	return binary.BigEndian.AppendUint64(b, m.Amount)
}

type Burn struct {
	From   [20]byte
	Amount uint64
}

func (b Burn) preimage() []byte {
	out := []byte{tagBurn}
	out = append(out, b.From[:]...)
	return binary.BigEndian.AppendUint64(out, b.Amount)
}

func addr(b byte) [20]byte {
	var a [20]byte
	for i := range a {
		a[i] = b
	}
	return a
}

func main() {
	alice, bob := addr(0xa1), addr(0xb0)

	events := []Hashable{
		Mint{To: alice, Amount: 1000},
		Transfer{From: alice, To: bob, Amount: 250, Memo: "rent"},
		Burn{From: bob, Amount: 50},
	}

	for _, e := range events {
		h := HashOf(e)
		fmt.Printf("%-14T %s\n", e, hex.EncodeToString(h[:]))
	}

	// A Mint and a Burn with identical fields hash differently — the tag.
	m := Mint{To: alice, Amount: 1000}
	bn := Burn{From: alice, Amount: 1000}
	hm, hb := HashOf(m), HashOf(bn)
	fmt.Printf("\nMint and Burn with identical fields collide? %v\n", hm == hb)

	// Length prefixing stops adjacent variable-length fields from sliding.
	t1 := Transfer{From: alice, To: bob, Amount: 1, Memo: "ab"}
	t2 := Transfer{From: alice, To: bob, Amount: 1, Memo: "abc"}
	fmt.Printf("memo \"ab\" vs \"abc\" collide? %v\n", HashOf(t1) == HashOf(t2))

	fmt.Println("\none interface, one HashOf, one rule — new types cannot invent their own")
}
```

**Output:**

```
main.Mint      3ad37b9927672fa31fbacf61c47b7a022f6763375aab18884755fca70497d8a9
main.Transfer  69c2779dfa6809d39d993d5fac5552f1606cc3fbe3714cc38ae2abb5c3ed5a1b
main.Burn      3920de7f3f74186f170263a8b01d13d1ce5a3cb336d489ab3825c16d49ba8a9d

Mint and Burn with identical fields collide? false
memo "ab" vs "abc" collide? false

one interface, one HashOf, one rule — new types cannot invent their own
```

---

## 16. A domain-separated Merkle root

`🔴 hard` · *Merkle*

The hasher lesson 05 builds on. Two tags, `0x00` for leaves and `0x01` for nodes, and one fold from leaves to root. The root commits to both the contents and the order of every item underneath it.

**Steps:**

1. Implement `LeafHash` and `NodeHash` with distinct domain tags.
2. Fold four leaves into a root and print every level.
3. Edit one leaf, then swap two, and confirm the root moves both times.
4. Present an internal node's 64-byte preimage as a leaf and confirm the tags keep them apart.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// The hasher lesson 05 builds its Merkle tree on. Two tags, one job: make it
// impossible to read a leaf as an internal node, or vice versa.
const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func LeafHash(data []byte) [32]byte {
	return sha256.Sum256(append([]byte{tagLeaf}, data...))
}

func NodeHash(l, r [32]byte) [32]byte {
	buf := make([]byte, 0, 1+64)
	buf = append(buf, tagNode)
	buf = append(buf, l[:]...)
	buf = append(buf, r[:]...)
	return sha256.Sum256(buf)
}

// Root folds a list of leaves into a single commitment. Odd levels promote the
// last node rather than duplicating it (lesson 05 covers why that matters).
func Root(items [][]byte) [32]byte {
	if len(items) == 0 {
		return sha256.Sum256([]byte{tagLeaf}) // empty-tree constant
	}
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = LeafHash(it)
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i]) // promote
				continue
			}
			next = append(next, NodeHash(level[i], level[i+1]))
		}
		level = next
	}
	return level[0]
}

func main() {
	txs := [][]byte{[]byte("tx-a"), []byte("tx-b"), []byte("tx-c"), []byte("tx-d")}

	root := Root(txs)
	fmt.Printf("4 leaves -> root %s\n", hex.EncodeToString(root[:]))

	// Every level, for the shape of it.
	l := make([][32]byte, len(txs))
	for i, t := range txs {
		l[i] = LeafHash(t)
	}
	fmt.Println("\nlevel 0 (leaves)")
	for i, h := range l {
		fmt.Printf("  %d %s\n", i, hex.EncodeToString(h[:8]))
	}
	n0, n1 := NodeHash(l[0], l[1]), NodeHash(l[2], l[3])
	fmt.Println("level 1")
	fmt.Printf("  0 %s\n  1 %s\n", hex.EncodeToString(n0[:8]), hex.EncodeToString(n1[:8]))
	top := NodeHash(n0, n1)
	fmt.Printf("root %s\n", hex.EncodeToString(top[:8]))

	// Any change to any leaf moves the root.
	txs2 := [][]byte{[]byte("tx-a"), []byte("tx-B"), []byte("tx-c"), []byte("tx-d")}
	fmt.Printf("\none leaf edited -> root changes: %v\n", Root(txs) != Root(txs2))

	// Order is part of the commitment.
	txs3 := [][]byte{[]byte("tx-b"), []byte("tx-a"), []byte("tx-c"), []byte("tx-d")}
	fmt.Printf("two leaves swapped -> root changes: %v\n", Root(txs) != Root(txs3))

	// The attack the tags prevent: presenting an internal node's 64-byte
	// preimage as if it were a leaf.
	forged := append(append([]byte{}, l[0][:]...), l[1][:]...)
	fmt.Printf("\nleaf(n0's preimage) == n0 ? %v   <- domain separation holds\n",
		LeafHash(forged) == n0)

	fmt.Println("\nthis is the hasher lesson 05 builds proofs on")
}
```

**Output:**

```
4 leaves -> root 6e5818a6c1d68c5aa635dcb16b6aa1166c374ed812682c807d3cf0f7c4b854df

level 0 (leaves)
  0 24536162901e5df6
  1 efc4942ae88b4394
  2 a60238363fd02697
  3 7473871f4cd69271
level 1
  0 f847efba820f4560
  1 c72cc6b52d9fc77b
root 6e5818a6c1d68c5a

one leaf edited -> root changes: true
two leaves swapped -> root changes: true

leaf(n0's preimage) == n0 ? false   <- domain separation holds

this is the hasher lesson 05 builds proofs on
```

---

## 17. A content-addressed blob store

`🔴 hard` · *Identifiers*

In a content-addressed store the key *is* the hash of the value, so the name proves the content. Re-hashing on read turns storage corruption, a bad cache or a malicious mirror into the same detectable error. IPFS, git objects and every block hash work this way.

**Steps:**

1. Store blobs keyed by their own digest, copying the caller's slice (lesson 03).
2. Store identical content twice and get deduplication for free.
3. Flip one bit in the stored bytes and watch `Get` refuse to return them.
4. Distinguish `ErrNotFound` from `ErrCorrupted` — they mean very different things.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
)

// A content-addressed store: the key IS the hash of the value. You do not
// choose names; the data names itself. IPFS works this way (lesson 55), as do
// git objects and every block hash you will ever fetch.
type Store struct {
	blobs map[[32]byte][]byte
}

func NewStore() *Store { return &Store{blobs: map[[32]byte][]byte{}} }

// Put returns the address the caller must keep. Storing the same bytes twice
// is free and idempotent — deduplication falls out of the design.
func (s *Store) Put(data []byte) [32]byte {
	key := sha256.Sum256(data)
	if _, ok := s.blobs[key]; !ok {
		// Copy: the caller's slice is not ours to keep (lesson 03).
		s.blobs[key] = append([]byte(nil), data...)
	}
	return key
}

var (
	ErrNotFound  = errors.New("not found")
	ErrCorrupted = errors.New("content does not match its address")
)

// Get re-hashes on the way out. This is the property that makes content
// addressing self-verifying: you cannot be handed the wrong bytes silently,
// whether the storage is a disk, a cache or an untrusted peer.
func (s *Store) Get(key [32]byte) ([]byte, error) {
	data, ok := s.blobs[key]
	if !ok {
		return nil, ErrNotFound
	}
	if sha256.Sum256(data) != key {
		return nil, ErrCorrupted
	}
	return data, nil
}

func short(k [32]byte) string { return hex.EncodeToString(k[:6]) }

func main() {
	s := NewStore()

	a := s.Put([]byte("the genesis block"))
	b := s.Put([]byte("the second block"))
	fmt.Printf("stored  %s  %q\n", short(a), "the genesis block")
	fmt.Printf("stored  %s  %q\n", short(b), "the second block")

	// Same content, same address — deduplicated automatically.
	again := s.Put([]byte("the genesis block"))
	fmt.Printf("\nre-storing identical content gives the same key: %v\n", a == again)
	fmt.Printf("blobs held: %d (not 3)\n", len(s.blobs))

	got, err := s.Get(a)
	fmt.Printf("\nget %s -> %q err=%v\n", short(a), got, err)

	// Bit rot, a bad disk, a malicious mirror — all the same to the reader.
	s.blobs[a][4] ^= 0x01
	_, err = s.Get(a)
	fmt.Printf("after flipping one bit on disk:\n  err=%v  corrupted=%v\n",
		err, errors.Is(err, ErrCorrupted))

	// A key nobody stored.
	var missing [32]byte
	_, err = s.Get(missing)
	fmt.Printf("\nget an unknown key -> notFound=%v\n", errors.Is(err, ErrNotFound))

	fmt.Println("\nthe hash is the name, so the name proves the content.")
	fmt.Println("you can fetch a blob from an untrusted source and still be certain.")
}
```

**Output:**

```
stored  adefd71a6d99  "the genesis block"
stored  e1bff9cecce2  "the second block"

re-storing identical content gives the same key: true
blobs held: 2 (not 3)

get adefd71a6d99 -> "the genesis block" err=<nil>
after flipping one bit on disk:
  err=content does not match its address  corrupted=true

get an unknown key -> notFound=true

the hash is the name, so the name proves the content.
you can fetch a blob from an untrusted source and still be certain.
```

---

## 18. A tamper-evident log

`🔴 hard` · *Identifiers*

Chain the entries so each commits to the one before, and the latest hash becomes a commitment to the entire history. This is a blockchain with one transaction per block, and it is exactly what lesson 08 formalises.

**Steps:**

1. Append entries that each store the previous entry's hash.
2. Verify the chain and print the head commitment.
3. Tamper with an early entry and see verification fail at the *next* one.
4. Repair every forward link and confirm the head still does not match the published one.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

const tagEntry byte = 0x30

// Entry is one record in an append-only log. Each one commits to the entry
// before it, so the log can be verified end to end from the latest hash alone.
type Entry struct {
	Index uint64
	Data  string
	Prev  [32]byte
}

func (e Entry) Hash() [32]byte {
	b := []byte{tagEntry}
	b = binary.BigEndian.AppendUint64(b, e.Index)
	b = append(b, e.Prev[:]...)
	b = binary.BigEndian.AppendUint32(b, uint32(len(e.Data)))
	b = append(b, e.Data...)
	return sha256.Sum256(b)
}

type Log struct{ entries []Entry }

func (l *Log) Append(data string) [32]byte {
	var prev [32]byte
	if n := len(l.entries); n > 0 {
		prev = l.entries[n-1].Hash()
	}
	e := Entry{Index: uint64(len(l.entries)), Data: data, Prev: prev}
	l.entries = append(l.entries, e)
	return e.Hash()
}

// Verify walks the chain and returns the index of the first broken link.
func (l *Log) Verify() (int, bool) {
	var prev [32]byte
	for i, e := range l.entries {
		if e.Prev != prev || e.Index != uint64(i) {
			return i, false
		}
		prev = e.Hash()
	}
	return -1, true
}

func short(h [32]byte) string { return hex.EncodeToString(h[:6]) }

func main() {
	var l Log
	for _, d := range []string{
		"alice -> bob 30",
		"bob -> carol 10",
		"carol -> dave 5",
		"dave -> erin 2",
	} {
		l.Append(d)
	}

	fmt.Printf("%-3s %-14s %-14s %s\n", "#", "prev", "hash", "data")
	for _, e := range l.entries {
		fmt.Printf("%-3d %-14s %-14s %q\n", e.Index, short(e.Prev), short(e.Hash()), e.Data)
	}

	i, ok := l.Verify()
	fmt.Printf("\nintact: %v (first break at %d)\n", ok, i)

	// The head hash is a commitment to the ENTIRE history. Publish it anywhere
	// and nobody can rewrite the past without you noticing.
	head := l.entries[len(l.entries)-1].Hash()
	fmt.Printf("head commitment: %s\n", hex.EncodeToString(head[:]))

	// Now rewrite history: change entry 1's amount.
	fmt.Println("\ntampering: entry 1 becomes \"bob -> carol 10000\"")
	l.entries[1].Data = "bob -> carol 10000"

	i, ok = l.Verify()
	fmt.Printf("intact: %v (first break at %d)\n", ok, i)
	fmt.Println("  entry 2 still stores the OLD hash of entry 1, so the link fails")

	// An attacker who also repairs the forward links still cannot reproduce the
	// published head — that is the whole point.
	for j := 2; j < len(l.entries); j++ {
		l.entries[j].Prev = l.entries[j-1].Hash()
	}
	i, ok = l.Verify()
	newHead := l.entries[len(l.entries)-1].Hash()
	fmt.Printf("\nafter recomputing every later link: intact=%v (break=%d)\n", ok, i)
	fmt.Printf("head is now: %s\n", hex.EncodeToString(newHead[:]))
	fmt.Printf("matches the published head: %v   <- the forgery is still caught\n", newHead == head)

	fmt.Println("\nthis is a blockchain with one transaction per block (lesson 08)")
}
```

**Output:**

```
#   prev           hash           data
0   000000000000   4dbdeb4ce738   "alice -> bob 30"
1   4dbdeb4ce738   f39118c4c869   "bob -> carol 10"
2   f39118c4c869   3008a501cf2e   "carol -> dave 5"
3   3008a501cf2e   700a7b5f656b   "dave -> erin 2"

intact: true (first break at -1)
head commitment: 700a7b5f656bb8c79ba3f5e1baadfc513da19509675efa32410e2314b719da0f

tampering: entry 1 becomes "bob -> carol 10000"
intact: false (first break at 2)
  entry 2 still stores the OLD hash of entry 1, so the link fails

after recomputing every later link: intact=true (break=-1)
head is now: 8b199cece9645434f60977b348adc9e0e59c39dcbada4440f60d6e46faf1d955
matches the published head: false   <- the forgery is still caught

this is a blockchain with one transaction per block (lesson 08)
```

---

## 19. Allocation-free hashing in a mining loop

`🔴 hard` · *Performance*

A miner hashes the same header millions of times, changing four bytes. Allocating once per attempt is a billion allocations over a billion attempts. The pattern — reuse the buffer, `Reset` the hasher, `Sum` into a slice you own — is what lesson 09 depends on.

**Steps:**

1. Write a naive miner that builds a fresh buffer and digest each attempt.
2. Write a tight one that allocates nothing inside the loop.
3. Confirm both find the same nonce.
4. Measure allocations per attempt with `testing.AllocsPerRun` — 1 versus 0.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"testing"
)

// A miner hashes the same 80-byte header millions of times, changing only the
// nonce. Allocating anything inside that loop is the difference between a
// working miner and a slow one (lesson 09).

// mineNaive allocates on every attempt: a new slice, a new hasher, a new digest.
func mineNaive(header []byte, target byte, maxNonce uint32) (uint32, bool) {
	for n := uint32(0); n < maxNonce; n++ {
		buf := make([]byte, len(header)+4) // allocation
		copy(buf, header)                  //
		binary.BigEndian.PutUint32(buf[len(header):], n)
		sum := sha256.Sum256(buf) // fresh hasher each time
		if sum[0] < target {
			return n, true
		}
	}
	return 0, false
}

// mineTight allocates nothing. One buffer, one hasher, both reused; the digest
// is written back into a slice we already own.
func mineTight(header []byte, target byte, maxNonce uint32) (uint32, bool) {
	buf := make([]byte, len(header)+4)
	copy(buf, header)
	noncePos := len(header)

	h := sha256.New()
	digest := make([]byte, 0, sha256.Size)

	for n := uint32(0); n < maxNonce; n++ {
		binary.BigEndian.PutUint32(buf[noncePos:], n)
		h.Reset()
		h.Write(buf)
		digest = h.Sum(digest[:0]) // append into our own backing array
		if digest[0] < target {
			return n, true
		}
	}
	return 0, false
}

func main() {
	header := []byte("block header goes here, 80 bytes in the real thing")
	const target = 0x08 // first byte below 0x08: roughly 1 in 32

	n1, ok1 := mineNaive(header, target, 1<<20)
	n2, ok2 := mineTight(header, target, 1<<20)
	fmt.Printf("naive found nonce %d (ok=%v)\n", n1, ok1)
	fmt.Printf("tight found nonce %d (ok=%v)\n", n2, ok2)
	fmt.Printf("same answer: %v\n", n1 == n2 && ok1 == ok2)

	buf := make([]byte, len(header)+4)
	copy(buf, header)
	binary.BigEndian.PutUint32(buf[len(header):], n2)
	sum := sha256.Sum256(buf)
	fmt.Printf("digest %s (first byte %#02x < %#02x)\n",
		hex.EncodeToString(sum[:8]), sum[0], target)

	// Measure one attempt of each. AllocsPerRun is deterministic.
	fmt.Printf("\nallocations per attempt\n")
	bufN := make([]byte, len(header)+4)
	fmt.Printf("  naive  %.0f\n", testing.AllocsPerRun(1000, func() {
		b := make([]byte, len(header)+4)
		copy(b, header)
		binary.BigEndian.PutUint32(b[len(header):], 1)
		_ = sha256.Sum256(b)
	}))

	h := sha256.New()
	dg := make([]byte, 0, sha256.Size)
	copy(bufN, header)
	fmt.Printf("  tight  %.0f\n", testing.AllocsPerRun(1000, func() {
		binary.BigEndian.PutUint32(bufN[len(header):], 1)
		h.Reset()
		h.Write(bufN)
		dg = h.Sum(dg[:0])
	}))

	fmt.Println("\nover a billion attempts, one allocation each is a billion allocations")
	fmt.Println("the pattern: reuse the buffer, Reset the hasher, Sum into your own slice")
}
```

**Output:**

```
naive found nonce 25 (ok=true)
tight found nonce 25 (ok=true)
same answer: true
digest 048531d4241b065b (first byte 0x04 < 0x08)

allocations per attempt
  naive  1
  tight  0

over a billion attempts, one allocation each is a billion allocations
the pattern: reuse the buffer, Reset the hasher, Sum into your own slice
```

---

## 20. SHA-256 vs Keccak-256

`🔴 hard` · *Performance*

Both hashers are allocation-free when reused; the cost is elsewhere. SHA-256 has dedicated CPU instructions on modern x86-64 and arm64, and Keccak does not — so Keccak runs several times slower in software. Ethereum pays that on every `KECCAK256` opcode.

**Steps:**

1. Measure allocations for both with the hasher and output slice reused.
2. Measure again constructing a hasher per call, and when the digest escapes to the heap.
3. Compare throughput over 50 MiB — the boolean reproduces, the exact figures are yours to measure.
4. Understand why: silicon support for one, plain software for the other.

```go
package main

import (
	"crypto/sha256"
	"fmt"
	"testing"
	"time"

	"golang.org/x/crypto/sha3"
)

// sinkB keeps results alive so the optimiser cannot delete the work.
var sinkB []byte

func main() {
	data := make([]byte, 1<<20) // 1 MiB
	for i := range data {
		data[i] = byte(i)
	}

	hs := sha256.New()
	hk := sha3.NewLegacyKeccak256()
	buf := make([]byte, 0, 64)

	// Both are allocation-free when the hasher and the output slice are reused.
	fmt.Println("allocations per 1 MiB hash (hasher reused, Sum into our own slice)")
	fmt.Printf("  sha256      %.0f\n", testing.AllocsPerRun(20, func() {
		hs.Reset()
		hs.Write(data)
		buf = hs.Sum(buf[:0])
	}))
	fmt.Printf("  keccak256   %.0f\n", testing.AllocsPerRun(20, func() {
		hk.Reset()
		hk.Write(data)
		buf = hk.Sum(buf[:0])
	}))

	// Constructing a hasher per call is what actually costs you. sha256.Sum256
	// returns a [32]byte array, so an unused result stays on the stack; the
	// Keccak constructor heap-allocates its state.
	fmt.Println("\nallocations when a hasher is constructed per call")
	fmt.Printf("  sha256.Sum256 (array result, discarded)  %.0f\n", testing.AllocsPerRun(20, func() {
		_ = sha256.Sum256(data)
	}))
	fmt.Printf("  new keccak hasher each time              %.0f\n", testing.AllocsPerRun(20, func() {
		h := sha3.NewLegacyKeccak256()
		h.Write(data)
		sinkB = h.Sum(nil)
	}))
	fmt.Printf("  sha256 digest kept (escapes to heap)     %.0f\n", testing.AllocsPerRun(20, func() {
		hs.Reset()
		hs.Write(data)
		sinkB = hs.Sum(nil)
	}))

	// Throughput. Exact numbers are hardware-specific, so compare the shape
	// rather than printing figures that will not reproduce on your machine.
	const rounds = 50
	t0 := time.Now()
	for i := 0; i < rounds; i++ {
		hs.Reset()
		hs.Write(data)
		buf = hs.Sum(buf[:0])
	}
	dSHA := time.Since(t0)

	t0 = time.Now()
	for i := 0; i < rounds; i++ {
		hk.Reset()
		hk.Write(data)
		buf = hk.Sum(buf[:0])
	}
	dKec := time.Since(t0)

	fmt.Printf("\nhashing %d MiB with each\n", rounds)
	fmt.Printf("  sha256 is faster than keccak256: %v\n", dSHA < dKec)
	fmt.Printf("  (run it yourself for the exact figures on your hardware)\n")

	fmt.Println("\nwhy: modern x86-64 and arm64 have SHA-256 instructions in silicon.")
	fmt.Println("Keccak has no such support, so it runs in plain software.")
	fmt.Println("Ethereum pays that cost on every Keccak-256 in the EVM (lesson 18).")
}
```

**Output:**

```
allocations per 1 MiB hash (hasher reused, Sum into our own slice)
  sha256      0
  keccak256   0

allocations when a hasher is constructed per call
  sha256.Sum256 (array result, discarded)  0
  new keccak hasher each time              1
  sha256 digest kept (escapes to heap)     1

hashing 50 MiB with each
  sha256 is faster than keccak256: true
  (run it yourself for the exact figures on your hardware)

why: modern x86-64 and arm64 have SHA-256 instructions in silicon.
Keccak has no such support, so it runs in plain software.
Ethereum pays that cost on every Keccak-256 in the EVM (lesson 18).
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
