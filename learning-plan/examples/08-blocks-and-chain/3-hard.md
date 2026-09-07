# Step 08 — Blocks & the Chain · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch       # first time only
# paste the example into main.go, then:
go run .
```

**Standard library only** — no chain, no node, no dependencies. Every timestamp is a fixed
constant, so all output reproduces exactly.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. A chainBuilder for the next seven lessons

`🔴 hard` · *Testing*

The fixture the next seven lessons depend on. The point of a builder is that **valid chains are the default** and invalidity is something a test asks for explicitly — so the interesting line of each test is the only line.

**Steps:**

1. Write `newChain().AddN(5)` producing a correct-by-construction chain.
2. Add `Corrupt(i, fn)` so a test can request one specific defect.
3. Add `Fork(n)` for the branch tests lesson 14 will need constantly.
4. Run four table-driven corruptions, each one line long.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
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
	buf := bytes.NewBuffer(make([]byte, 0, 92))
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

func merkleRoot(txs []string) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256(nil)
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		level[i] = sha256.Sum256([]byte(t))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			r := i + 1
			if r == len(level) {
				r = i
			}
			next = append(next, sha256.Sum256(append(append([]byte{}, level[i][:]...), level[r][:]...)))
		}
		level = next
	}
	return level[0]
}

type Block struct {
	Header Header
	Body   []string
}

// ===========================================================================
// chainBuilder: the test fixture the next seven lessons reuse.
//
// The point of a builder is that valid chains are the DEFAULT and invalidity
// is something you ask for explicitly. Without it, every test starts with
// thirty lines of setup and the interesting line is invisible.
// ===========================================================================

type chainBuilder struct {
	blocks    []Block
	timestamp int64
	spacing   int64
}

func newChain() *chainBuilder {
	b := &chainBuilder{timestamp: 1700000000, spacing: 600}
	var zero [32]byte
	body := []string{"genesis"}
	b.blocks = []Block{{
		Header: Header{Version: 1, PrevHash: zero, MerkleRoot: merkleRoot(body),
			Timestamp: b.timestamp, Bits: 0x1f00ffff, Height: 0},
		Body: body,
	}}
	return b
}

// Add appends a valid block. Every field is derived from the parent, so the
// result is correct by construction.
func (b *chainBuilder) Add(txs ...string) *chainBuilder {
	if len(txs) == 0 {
		txs = []string{fmt.Sprintf("tx-%d", len(b.blocks))}
	}
	parent := b.blocks[len(b.blocks)-1].Header
	b.blocks = append(b.blocks, Block{
		Header: Header{
			Version: 1, PrevHash: parent.Hash(), MerkleRoot: merkleRoot(txs),
			Timestamp: parent.Timestamp + b.spacing, Bits: parent.Bits,
			Height: parent.Height + 1,
		},
		Body: txs,
	})
	return b
}

// AddN is the common case: n blocks of filler.
func (b *chainBuilder) AddN(n int) *chainBuilder {
	for i := 0; i < n; i++ {
		b.Add()
	}
	return b
}

// Corrupt applies a deliberate mutation to one block, WITHOUT repairing the
// links. This is how a test asks for a specific kind of invalid chain.
func (b *chainBuilder) Corrupt(i int, f func(*Block)) *chainBuilder {
	f(&b.blocks[i])
	return b
}

// Fork returns a second builder sharing the first n blocks — the setup every
// reorg test in lesson 14 will need.
func (b *chainBuilder) Fork(n int) *chainBuilder {
	f := &chainBuilder{timestamp: b.timestamp, spacing: b.spacing}
	f.blocks = append([]Block(nil), b.blocks[:n]...)
	return f
}

func (b *chainBuilder) Chain() []Block { return b.blocks }
func (b *chainBuilder) Tip() Block     { return b.blocks[len(b.blocks)-1] }

func Validate(chain []Block) error {
	for i, blk := range chain {
		if merkleRoot(blk.Body) != blk.Header.MerkleRoot {
			return fmt.Errorf("block %d: merkle mismatch", i)
		}
		if i == 0 {
			continue
		}
		if blk.Header.PrevHash != chain[i-1].Header.Hash() {
			return fmt.Errorf("block %d: broken link", i)
		}
		if blk.Header.Height != chain[i-1].Header.Height+1 {
			return fmt.Errorf("block %d: bad height", i)
		}
	}
	return nil
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	// One line for a valid five-block chain.
	c := newChain().AddN(5)
	fmt.Printf("built %d blocks, valid: %v\n", len(c.Chain()), Validate(c.Chain()) == nil)
	fmt.Printf("tip: height %d  %s\n", c.Tip().Header.Height, short(c.Tip().Header.Hash()))

	// Named transactions when the test cares about them.
	c2 := newChain().Add("alice->bob 30").Add("bob->carol 10")
	fmt.Printf("\nwith named txs, valid: %v\n", Validate(c2.Chain()) == nil)
	for _, b := range c2.Chain() {
		fmt.Printf("  %d %v\n", b.Header.Height, b.Body)
	}

	// And the interesting line is now the only line.
	fmt.Println("\ntable-driven invalidity, one mutation each:")
	cases := []struct {
		name   string
		mutate func(*Block)
	}{
		{"body edited", func(b *Block) { b.Body = []string{"forged"} }},
		{"height wrong", func(b *Block) { b.Header.Height = 99 }},
		{"prev hash zeroed", func(b *Block) { b.Header.PrevHash = [32]byte{} }},
		{"timestamp changed", func(b *Block) { b.Header.Timestamp += 1 }},
	}
	for _, tc := range cases {
		bad := newChain().AddN(5).Corrupt(2, tc.mutate)
		err := Validate(bad.Chain())
		fmt.Printf("  %-20s -> %v\n", tc.name, err)
	}

	// Forking, which lesson 14 needs constantly.
	base := newChain().AddN(3)
	branchA := base.Fork(3).Add("A-1").Add("A-2")
	branchB := base.Fork(3).Add("B-1")
	fmt.Printf("\nfork at height 2:\n")
	fmt.Printf("  branch A  %d blocks, tip %s, valid %v\n",
		len(branchA.Chain()), short(branchA.Tip().Header.Hash()), Validate(branchA.Chain()) == nil)
	fmt.Printf("  branch B  %d blocks, tip %s, valid %v\n",
		len(branchB.Chain()), short(branchB.Tip().Header.Hash()), Validate(branchB.Chain()) == nil)
	fmt.Printf("  shared parent at height 2: %v\n",
		branchA.Chain()[2].Header.Hash() == branchB.Chain()[2].Header.Hash())

	fmt.Println("\nthis fixture is the thing to carry into lessons 09-15.")
	fmt.Println("every test there starts 'given a valid chain, when X is wrong...'")
}
```

**Output:**

```
built 6 blocks, valid: true
tip: height 5  cb802d649291e74d

with named txs, valid: true
  0 [genesis]
  1 [alice->bob 30]
  2 [bob->carol 10]

table-driven invalidity, one mutation each:
  body edited          -> block 2: merkle mismatch
  height wrong         -> block 2: bad height
  prev hash zeroed     -> block 2: broken link
  timestamp changed    -> block 3: broken link

fork at height 2:
  branch A  5 blocks, tip 9078eb3d6119bcef, valid true
  branch B  4 blocks, tip d0c601e2c4a5149c, valid true
  shared parent at height 2: true

this fixture is the thing to carry into lessons 09-15.
every test there starts 'given a valid chain, when X is wrong...'
```

---

## 15. A Chain that validates before it mutates

`🔴 hard` · *Validation*

A `Chain` type indexed by hash — the only stable identifier. The rule that matters: `Append` validates completely and *then* mutates. A validator that mutates as it goes is how you end up with a half-applied block and a corrupt database (lesson 12).

**Steps:**

1. Index blocks by hash, keeping the canonical order separately.
2. Run stateless checks, then duplicate, orphan, height, and median-time-past.
3. Exercise all eight rejection paths with `errors.Is`.
4. Confirm the chain is untouched by every rejection.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"sort"
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
	buf := bytes.NewBuffer(make([]byte, 0, 92))
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

func merkleRoot(txs []string) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256(nil)
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		level[i] = sha256.Sum256([]byte(t))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			r := i + 1
			if r == len(level) {
				r = i
			}
			next = append(next, sha256.Sum256(append(append([]byte{}, level[i][:]...), level[r][:]...)))
		}
		level = next
	}
	return level[0]
}

type Block struct {
	Header Header
	Body   []string
}

var (
	ErrBadVersion = errors.New("unsupported version")
	ErrEmptyBody  = errors.New("empty body")
	ErrBadMerkle  = errors.New("merkle root mismatch")
	ErrOrphan     = errors.New("parent not found")
	ErrBadHeight  = errors.New("height is not parent+1")
	ErrTooOld     = errors.New("timestamp at or before median-time-past")
	ErrDuplicate  = errors.New("block already known")
)

// Chain holds blocks indexed by hash, which is the only stable identifier
// (example 9: height is a claim, the hash is the block).
type Chain struct {
	byHash map[[32]byte]Block
	order  [][32]byte // the canonical sequence, genesis first
}

func NewChain(genesis Block) *Chain {
	c := &Chain{byHash: map[[32]byte]Block{}}
	h := genesis.Header.Hash()
	c.byHash[h] = genesis
	c.order = append(c.order, h)
	return c
}

func (c *Chain) Tip() Block { return c.byHash[c.order[len(c.order)-1]] }
func (c *Chain) Len() int   { return len(c.order) }

func (c *Chain) medianTimePast() int64 {
	const window = 11
	n := window
	if len(c.order) < n {
		n = len(c.order)
	}
	ts := make([]int64, 0, n)
	for _, h := range c.order[len(c.order)-n:] {
		ts = append(ts, c.byHash[h].Header.Timestamp)
	}
	sort.Slice(ts, func(i, j int) bool { return ts[i] < ts[j] })
	return ts[len(ts)/2]
}

// stateless checks need nothing but the block.
func stateless(b Block) error {
	if b.Header.Version != 1 {
		return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version)
	}
	if len(b.Body) == 0 {
		return ErrEmptyBody
	}
	if merkleRoot(b.Body) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	return nil
}

// Append runs both passes and only then mutates the chain. A validator that
// mutates before it finishes checking is how you corrupt state (lesson 12).
func (c *Chain) Append(b Block) error {
	if err := stateless(b); err != nil {
		return err
	}
	h := b.Header.Hash()
	if _, seen := c.byHash[h]; seen {
		return ErrDuplicate
	}
	parent, ok := c.byHash[b.Header.PrevHash]
	if !ok {
		return fmt.Errorf("%w: %x", ErrOrphan, b.Header.PrevHash[:4])
	}
	if b.Header.Height != parent.Header.Height+1 {
		return fmt.Errorf("%w: got %d, parent %d", ErrBadHeight,
			b.Header.Height, parent.Header.Height)
	}
	if mtp := c.medianTimePast(); b.Header.Timestamp <= mtp {
		return fmt.Errorf("%w: %d <= %d", ErrTooOld, b.Header.Timestamp, mtp)
	}
	c.byHash[h] = b
	c.order = append(c.order, h)
	return nil
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func genesis() Block {
	var zero [32]byte
	body := []string{"genesis"}
	return Block{Header: Header{Version: 1, PrevHash: zero, MerkleRoot: merkleRoot(body),
		Timestamp: 1700000000, Bits: 0x1f00ffff, Height: 0}, Body: body}
}

func next(parent Header, txs []string, offset int64) Block {
	return Block{
		Header: Header{Version: 1, PrevHash: parent.Hash(), MerkleRoot: merkleRoot(txs),
			Timestamp: parent.Timestamp + offset, Bits: parent.Bits, Height: parent.Height + 1},
		Body: txs,
	}
}

func main() {
	c := NewChain(genesis())

	// Build a healthy chain.
	for i := 1; i <= 4; i++ {
		b := next(c.Tip().Header, []string{fmt.Sprintf("tx-%d", i)}, 600)
		if err := c.Append(b); err != nil {
			fmt.Println("unexpected:", err)
			return
		}
	}
	fmt.Printf("chain length %d, tip height %d (%s)\n\n",
		c.Len(), c.Tip().Header.Height, short(c.Tip().Header.Hash()))

	// Now every rejection path, each one mutation away from valid.
	tip := c.Tip().Header
	cases := []struct {
		name  string
		build func() Block
		want  error
	}{
		{"valid", func() Block { return next(tip, []string{"ok"}, 600) }, nil},
		{"bad version", func() Block {
			b := next(tip, []string{"x"}, 600)
			b.Header.Version = 2
			return b
		}, ErrBadVersion},
		{"empty body", func() Block {
			b := next(tip, []string{"x"}, 600)
			b.Body = nil
			return b
		}, ErrEmptyBody},
		{"merkle mismatch", func() Block {
			b := next(tip, []string{"x"}, 600)
			b.Body = []string{"y"}
			return b
		}, ErrBadMerkle},
		{"orphan", func() Block {
			b := next(tip, []string{"x"}, 600)
			b.Header.PrevHash = [32]byte{9}
			return b
		}, ErrOrphan},
		{"wrong height", func() Block {
			b := next(tip, []string{"x"}, 600)
			b.Header.Height = 42
			return b
		}, ErrBadHeight},
		{"backdated", func() Block { return next(tip, []string{"x"}, -5000) }, ErrTooOld},
		{"duplicate", func() Block { return c.Tip() }, ErrDuplicate},
	}

	fmt.Printf("%-18s %-10s %s\n", "case", "matches", "error")
	for _, tc := range cases {
		err := c.Append(tc.build())
		match := errors.Is(err, tc.want) || (tc.want == nil && err == nil)
		msg := "-"
		if err != nil {
			msg = err.Error()
		}
		fmt.Printf("%-18s %-10v %s\n", tc.name, match, msg)
	}

	fmt.Printf("\nchain length after all that: %d\n", c.Len())
	fmt.Println("  only the one valid block was appended; every rejection left")
	fmt.Println("  the chain untouched, because Append validates before it mutates.")

	fmt.Println("\nwhat lesson 09 adds to this: a Bits->target check and the proof")
	fmt.Println("of work that makes producing a valid block expensive.")
	fmt.Println("what lesson 14 adds: keeping REJECTED-as-orphan blocks, tracking")
	fmt.Println("branches, and choosing between them by accumulated work.")
}
```

**Output:**

```
chain length 5, tip height 4 (1044d12200dba3e5)

case               matches    error
valid              true       -
bad version        true       unsupported version: 2
empty body         true       empty body
merkle mismatch    true       merkle root mismatch
orphan             true       parent not found: 09000000
wrong height       true       height is not parent+1: got 42, parent 4
backdated          true       timestamp at or before median-time-past: 1699997400 <= 1700001800
duplicate          true       block already known

chain length after all that: 6
  only the one valid block was appended; every rejection left
  the chain untouched, because Append validates before it mutates.

what lesson 09 adds to this: a Bits->target check and the proof
of work that makes producing a valid block expensive.
what lesson 14 adds: keeping REJECTED-as-orphan blocks, tracking
branches, and choosing between them by accumulated work.
```

---

## 16. Fuzzing the decoder

`🔴 hard` · *Testing*

A decoder is your most exposed surface — it runs on bytes a hostile peer chose. This one contains the classic bug: a length prefix read from the wire and passed straight to `make`. A peer sends four bytes and you allocate four billion entries.

**Steps:**

1. Write the naive decoder and mark the two places it trusts the input.
2. Write the safe one, bounding every count against the bytes actually remaining.
3. Hand-craft the 4-billion-transaction claim and the 2 GB length prefix.
4. Run 20,000 deterministic mutations and confirm zero panics — that is the bar.
5. See the same thing as a real `testing.F` target for `go test -fuzz`.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"errors"
	"fmt"
	"math/rand"
)

const HeaderSize = 92

// ---------------------------------------------------------------------------
// A block on the wire: a fixed header, then a count, then that many
// length-prefixed transactions. The decoder below has a real bug.
// ---------------------------------------------------------------------------

type Block struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
	Body       []string
}

func (b Block) Encode() []byte {
	buf := new(bytes.Buffer)
	binary.Write(buf, binary.BigEndian, b.Version)
	buf.Write(b.PrevHash[:])
	buf.Write(b.MerkleRoot[:])
	binary.Write(buf, binary.BigEndian, b.Timestamp)
	binary.Write(buf, binary.BigEndian, b.Bits)
	binary.Write(buf, binary.BigEndian, b.Nonce)
	binary.Write(buf, binary.BigEndian, b.Height)
	binary.Write(buf, binary.BigEndian, uint32(len(b.Body)))
	for _, tx := range b.Body {
		binary.Write(buf, binary.BigEndian, uint32(len(tx)))
		buf.WriteString(tx)
	}
	return buf.Bytes()
}

var ErrMalformed = errors.New("malformed block")

// DecodeNaive trusts the counts it is sent. Two bugs live here.
func DecodeNaive(data []byte) (b Block, err error) {
	if len(data) < HeaderSize+4 {
		return b, ErrMalformed
	}
	r := bytes.NewReader(data)
	binary.Read(r, binary.BigEndian, &b.Version)
	r.Read(b.PrevHash[:])
	r.Read(b.MerkleRoot[:])
	binary.Read(r, binary.BigEndian, &b.Timestamp)
	binary.Read(r, binary.BigEndian, &b.Bits)
	binary.Read(r, binary.BigEndian, &b.Nonce)
	binary.Read(r, binary.BigEndian, &b.Height)

	var count uint32
	binary.Read(r, binary.BigEndian, &count)

	// BUG 1: `count` came from the wire. A peer sends 0xffffffff and we try
	// to allocate a 4-billion-entry slice.
	body := make([]string, 0, count)

	for i := uint32(0); i < count; i++ {
		var n uint32
		if err := binary.Read(r, binary.BigEndian, &n); err != nil {
			return b, ErrMalformed
		}
		// BUG 2: same problem, per transaction.
		tx := make([]byte, n)
		if _, err := r.Read(tx); err != nil {
			return b, ErrMalformed
		}
		body = append(body, string(tx))
	}
	b.Body = body
	return b, nil
}

// DecodeSafe bounds everything against what is actually left to read.
func DecodeSafe(data []byte) (b Block, err error) {
	const maxBodyCount = 10_000
	const maxTxLen = 1 << 20

	if len(data) < HeaderSize+4 {
		return b, fmt.Errorf("%w: too short", ErrMalformed)
	}
	r := bytes.NewReader(data)
	binary.Read(r, binary.BigEndian, &b.Version)
	r.Read(b.PrevHash[:])
	r.Read(b.MerkleRoot[:])
	binary.Read(r, binary.BigEndian, &b.Timestamp)
	binary.Read(r, binary.BigEndian, &b.Bits)
	binary.Read(r, binary.BigEndian, &b.Nonce)
	binary.Read(r, binary.BigEndian, &b.Height)

	var count uint32
	binary.Read(r, binary.BigEndian, &count)

	// A count is only plausible if the bytes to support it are present:
	// each entry needs at least a 4-byte length prefix.
	if count > maxBodyCount || uint64(count)*4 > uint64(r.Len()) {
		return b, fmt.Errorf("%w: implausible tx count %d with %d bytes left",
			ErrMalformed, count, r.Len())
	}
	body := make([]string, 0, count)

	for i := uint32(0); i < count; i++ {
		var n uint32
		if err := binary.Read(r, binary.BigEndian, &n); err != nil {
			return b, fmt.Errorf("%w: truncated length", ErrMalformed)
		}
		if n > maxTxLen || uint64(n) > uint64(r.Len()) {
			return b, fmt.Errorf("%w: tx length %d exceeds %d remaining", ErrMalformed, n, r.Len())
		}
		tx := make([]byte, n)
		if _, err := readFull(r, tx); err != nil {
			return b, fmt.Errorf("%w: truncated tx", ErrMalformed)
		}
		body = append(body, string(tx))
	}
	if r.Len() != 0 {
		return b, fmt.Errorf("%w: %d trailing bytes", ErrMalformed, r.Len())
	}
	b.Body = body
	return b, nil
}

func readFull(r *bytes.Reader, p []byte) (int, error) {
	n := 0
	for n < len(p) {
		m, err := r.Read(p[n:])
		n += m
		if err != nil {
			return n, err
		}
	}
	return n, nil
}

// try runs a decoder and reports a panic as a failure rather than crashing.
func try(name string, f func() error) (ok bool, note string) {
	defer func() {
		if r := recover(); r != nil {
			ok, note = false, fmt.Sprintf("PANIC: %v", r)
		}
	}()
	if err := f(); err != nil {
		return true, "rejected"
	}
	return true, "accepted"
}

func main() {
	valid := Block{Version: 1, Timestamp: 1700000000, Height: 1,
		Body: []string{"tx-a", "tx-b", "tx-c"}}
	encoded := valid.Encode()
	fmt.Printf("a valid block encodes to %d bytes\n", len(encoded))

	got, err := DecodeSafe(encoded)
	fmt.Printf("round trip: err=%v body=%v\n\n", err, got.Body)

	// ---- the hand-written attack ------------------------------------------
	// Take a valid block and claim it has 4 billion transactions.
	evil := append([]byte(nil), encoded...)
	binary.BigEndian.PutUint32(evil[HeaderSize:HeaderSize+4], 0xffffffff)

	fmt.Println("a peer claims the block has 4,294,967,295 transactions")
	fmt.Printf("  the message is still only %d bytes\n", len(evil))
	_, err = DecodeSafe(evil)
	fmt.Printf("  DecodeSafe:  %v\n", err)
	fmt.Println("  DecodeNaive: would allocate a 4-billion-entry slice here")
	fmt.Println("  (not run — it would exhaust memory on this machine)")

	// A length prefix that overruns the buffer.
	evil2 := append([]byte(nil), encoded...)
	binary.BigEndian.PutUint32(evil2[HeaderSize:HeaderSize+4], 1)
	binary.BigEndian.PutUint32(evil2[HeaderSize+4:HeaderSize+8], 0x7fffffff)
	_, err = DecodeSafe(evil2[:HeaderSize+8])
	fmt.Printf("\none tx claiming 2GB of length:\n  DecodeSafe: %v\n", err)

	// ---- the fuzz loop -----------------------------------------------------
	// Deterministic mutations of a valid encoding. This is what
	// `go test -fuzz` automates; the principle is identical.
	fmt.Println("\nfuzzing: 20000 deterministic mutations of a valid block")
	rng := rand.New(rand.NewSource(1))
	var accepted, rejected, panicked int
	for i := 0; i < 20000; i++ {
		m := append([]byte(nil), encoded...)
		switch rng.Intn(3) {
		case 0: // flip a byte
			m[rng.Intn(len(m))] ^= byte(1 << rng.Intn(8))
		case 1: // truncate
			m = m[:rng.Intn(len(m))]
		case 2: // extend
			m = append(m, byte(rng.Intn(256)))
		}
		ok, note := try("safe", func() error { _, e := DecodeSafe(m); return e })
		switch {
		case !ok:
			panicked++
		case note == "accepted":
			accepted++
		default:
			rejected++
		}
	}
	fmt.Printf("  accepted %d   rejected %d   PANICKED %d\n", accepted, rejected, panicked)
	fmt.Println("  a decoder that never panics on hostile input is the bar.")
	fmt.Println("  accepting some mutations is fine — a flipped nonce is still a")
	fmt.Println("  well-formed block. It fails VALIDATION (example 15), not parsing.")

	fmt.Println("\nin a real test file this is:")
	fmt.Println("    func FuzzDecode(f *testing.F) {")
	fmt.Println("        f.Add(validEncoding)")
	fmt.Println("        f.Fuzz(func(t *testing.T, data []byte) { DecodeSafe(data) })")
	fmt.Println("    }")
	fmt.Println("  run with: go test -fuzz=FuzzDecode")
	fmt.Println("  lesson 34 makes this part of the suite.")
}
```

**Output:**

```
a valid block encodes to 120 bytes
round trip: err=<nil> body=[tx-a tx-b tx-c]

a peer claims the block has 4,294,967,295 transactions
  the message is still only 120 bytes
  DecodeSafe:  malformed block: implausible tx count 4294967295 with 24 bytes left
  DecodeNaive: would allocate a 4-billion-entry slice here
  (not run — it would exhaust memory on this machine)

one tx claiming 2GB of length:
  DecodeSafe: malformed block: tx length 2147483647 exceeds 0 remaining

fuzzing: 20000 deterministic mutations of a valid block
  accepted 5773   rejected 14227   PANICKED 0
  a decoder that never panics on hostile input is the bar.
  accepting some mutations is fine — a flipped nonce is still a
  well-formed block. It fails VALIDATION (example 15), not parsing.

in a real test file this is:
    func FuzzDecode(f *testing.F) {
        f.Add(validEncoding)
        f.Fuzz(func(t *testing.T, data []byte) { DecodeSafe(data) })
    }
  run with: go test -fuzz=FuzzDecode
  lesson 34 makes this part of the suite.
```

---

## 17. Changing the format without splitting the chain

`🔴 hard` · *Versioning*

The `Version` field exists so the layout can change without every old block becoming unparseable. The discipline: read the version first, encode by the header's *own* version, and never change the bytes of a block that already exists.

**Steps:**

1. Add a field that only exists from version 2 onward.
2. Encode by the header's own version, so v1 bytes are unchanged.
3. Confirm setting the new field on a v1 header changes nothing.
4. Enforce activation at a known height, and reject unknown versions outright.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
)

// The Version field exists so the header layout can change without every old
// block becoming unparseable. The rule: a node must be able to hash and
// validate EVERY block ever made, including ones from before the change.

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64

	// Added in version 2. Absent from v1 blocks, and absent from their bytes.
	ExtraNonce uint64
}

var ErrUnknownVersion = errors.New("unknown header version")

// Bytes encodes according to the header's OWN version, not the current one.
func (h Header) Bytes() []byte {
	buf := bytes.NewBuffer(make([]byte, 0, 100))
	binary.Write(buf, binary.BigEndian, h.Version)
	buf.Write(h.PrevHash[:])
	buf.Write(h.MerkleRoot[:])
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Bits)
	binary.Write(buf, binary.BigEndian, h.Nonce)
	binary.Write(buf, binary.BigEndian, h.Height)
	if h.Version >= 2 {
		binary.Write(buf, binary.BigEndian, h.ExtraNonce)
	}
	return buf.Bytes()
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

func ParseHeader(b []byte) (Header, error) {
	var h Header
	if len(b) < 4 {
		return h, ErrUnknownVersion
	}
	// Read the version FIRST, then decide the layout.
	version := binary.BigEndian.Uint32(b[:4])
	want := 92
	if version >= 2 {
		want = 100
	}
	if version == 0 || version > 2 {
		return h, fmt.Errorf("%w: %d", ErrUnknownVersion, version)
	}
	if len(b) != want {
		return h, fmt.Errorf("version %d header must be %d bytes, got %d", version, want, len(b))
	}
	r := bytes.NewReader(b)
	binary.Read(r, binary.BigEndian, &h.Version)
	r.Read(h.PrevHash[:])
	r.Read(h.MerkleRoot[:])
	binary.Read(r, binary.BigEndian, &h.Timestamp)
	binary.Read(r, binary.BigEndian, &h.Bits)
	binary.Read(r, binary.BigEndian, &h.Nonce)
	binary.Read(r, binary.BigEndian, &h.Height)
	if version >= 2 {
		binary.Read(r, binary.BigEndian, &h.ExtraNonce)
	}
	return h, nil
}

// ActivationHeight is where the new version becomes MANDATORY. Everything
// below it must still be accepted in the old format, forever.
const ActivationHeight = 100

func checkVersion(h Header) error {
	switch {
	case h.Height < ActivationHeight && h.Version != 1:
		return fmt.Errorf("height %d requires version 1, got %d", h.Height, h.Version)
	case h.Height >= ActivationHeight && h.Version < 2:
		return fmt.Errorf("height %d requires version 2+, got %d", h.Height, h.Version)
	}
	return nil
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	v1 := Header{Version: 1, Timestamp: 1700000000, Height: 50}
	v2 := Header{Version: 2, Timestamp: 1700060000, Height: 150, ExtraNonce: 7}

	fmt.Printf("v1 header  %3d bytes  hash %s\n", len(v1.Bytes()), short(v1.Hash()))
	fmt.Printf("v2 header  %3d bytes  hash %s\n", len(v2.Bytes()), short(v2.Hash()))

	// Both round-trip, each under its own layout.
	for _, h := range []Header{v1, v2} {
		back, err := ParseHeader(h.Bytes())
		fmt.Printf("\nv%d round trip: err=%v, identical: %v\n", h.Version, err, back == h)
	}

	// A v1 header's bytes and hash are UNCHANGED by the existence of v2.
	// That is the whole requirement: old blocks keep their identity.
	fmt.Printf("\nv1 hash is unaffected by the new field: %v\n",
		short(v1.Hash()) == short(Header{Version: 1, Timestamp: 1700000000, Height: 50}.Hash()))
	withGarbage := v1
	withGarbage.ExtraNonce = 999999 // set, but never encoded for v1
	fmt.Printf("setting ExtraNonce on a v1 header changes nothing: %v\n",
		withGarbage.Hash() == v1.Hash())

	// Activation: the version becomes mandatory at a known height.
	fmt.Printf("\nversion 2 activates at height %d\n\n", ActivationHeight)
	cases := []Header{
		{Version: 1, Height: 99},
		{Version: 2, Height: 99},
		{Version: 1, Height: 100},
		{Version: 2, Height: 100},
	}
	fmt.Printf("%-10s %-8s %s\n", "version", "height", "accepted")
	for _, h := range cases {
		err := checkVersion(h)
		msg := "yes"
		if err != nil {
			msg = err.Error()
		}
		fmt.Printf("%-10d %-8d %s\n", h.Version, h.Height, msg)
	}

	// Unknown versions.
	fmt.Println("\nan unknown version:")
	future := make([]byte, 92)
	binary.BigEndian.PutUint32(future[:4], 99)
	_, err := ParseHeader(future)
	fmt.Printf("  %v (ErrUnknownVersion: %v)\n", err, errors.Is(err, ErrUnknownVersion))
	fmt.Println("  an old node CANNOT validate a v99 block, so it must reject it")
	fmt.Println("  rather than guess — that rejection is what a hard fork IS.")

	fmt.Println("\nthe discipline")
	fmt.Println("  1. version first in the byte layout, so it can always be read")
	fmt.Println("  2. encode by the header's OWN version, never the newest")
	fmt.Println("  3. old blocks keep their exact bytes and hashes, forever")
	fmt.Println("  4. activate at a height every node agrees on in advance")
	fmt.Println("\nget this wrong and old blocks rehash differently, which means")
	fmt.Println("the whole chain fails to validate — a self-inflicted fork.")
}
```

**Output:**

```
v1 header   92 bytes  hash 9727ec22d94b7541
v2 header  100 bytes  hash a6854a3f6c928e10

v1 round trip: err=<nil>, identical: true

v2 round trip: err=<nil>, identical: true

v1 hash is unaffected by the new field: true
setting ExtraNonce on a v1 header changes nothing: true

version 2 activates at height 100

version    height   accepted
1          99       yes
2          99       height 99 requires version 1, got 2
1          100      height 100 requires version 2+, got 1
2          100      yes

an unknown version:
  unknown header version: 99 (ErrUnknownVersion: true)
  an old node CANNOT validate a v99 block, so it must reject it
  rather than guess — that rejection is what a hard fork IS.

the discipline
  1. version first in the byte layout, so it can always be read
  2. encode by the header's OWN version, never the newest
  3. old blocks keep their exact bytes and hashes, forever
  4. activate at a height every node agrees on in advance

get this wrong and old blocks rehash differently, which means
the whole chain fails to validate — a self-inflicted fork.
```

---

## 18. The whole thing, assembled

`🔴 hard` · *Assembly*

Everything from this lesson in one file: header, Merkle root with domain separation, genesis, both validation passes, and a `Chain` with an injectable clock. This is the exact code lesson 09 extends with proof of work.

**Steps:**

1. Read it top to bottom — it is the shape the next seven lessons grow.
2. Note the injectable clock: deterministic tests start now, not in lesson 34.
3. Note `NextBlock`, which is where lesson 09 will grind the nonce.
4. Read the closing list of what each following lesson changes about this code.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"sort"
)

// ===========================================================================
// Everything from lesson 08, assembled. This is the starting point lesson 09
// extends with proof of work, and lesson 10 with real transactions.
//
// In practice this lives in a package (internal/chain); it is one file here
// because every example must be runnable on its own.
// ===========================================================================

// ---------------------------------------------------------------- header ---

const (
	HeaderVersion = 1
	HeaderSize    = 92
	MaxBodyLen    = 4096
	MTPWindow     = 11
	MaxFutureSec  = 2 * 60 * 60
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

var ErrBadHeaderSize = errors.New("wrong header size")

func ParseHeader(b []byte) (Header, error) {
	var h Header
	if len(b) != HeaderSize {
		return h, fmt.Errorf("%w: got %d, want %d", ErrBadHeaderSize, len(b), HeaderSize)
	}
	r := bytes.NewReader(b)
	binary.Read(r, binary.BigEndian, &h.Version)
	r.Read(h.PrevHash[:])
	r.Read(h.MerkleRoot[:])
	binary.Read(r, binary.BigEndian, &h.Timestamp)
	binary.Read(r, binary.BigEndian, &h.Bits)
	binary.Read(r, binary.BigEndian, &h.Nonce)
	binary.Read(r, binary.BigEndian, &h.Height)
	return h, nil
}

// ------------------------------------------------------------------ body ---

const (
	tagLeaf byte = 0x00 // domain separation, lesson 05
	tagNode byte = 0x01
)

func MerkleRoot(txs []string) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256([]byte{tagLeaf})
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		level[i] = sha256.Sum256(append([]byte{tagLeaf}, t...))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i]) // promote, never duplicate (lesson 05)
				continue
			}
			buf := append([]byte{tagNode}, level[i][:]...)
			next = append(next, sha256.Sum256(append(buf, level[i+1][:]...)))
		}
		level = next
	}
	return level[0]
}

type Block struct {
	Header Header
	Body   []string
}

// ------------------------------------------------------------ validation ---

var (
	ErrBadVersion = errors.New("unsupported version")
	ErrEmptyBody  = errors.New("empty body")
	ErrTooLarge   = errors.New("body too large")
	ErrBadMerkle  = errors.New("merkle root mismatch")
	ErrDuplicate  = errors.New("already known")
	ErrOrphan     = errors.New("parent not found")
	ErrBadHeight  = errors.New("height is not parent+1")
	ErrTooOld     = errors.New("timestamp at or before median-time-past")
	ErrTooFuture  = errors.New("timestamp too far in the future")
)

// CheckStateless needs only the block, so it can run on arrival.
func CheckStateless(b Block) error {
	if b.Header.Version != HeaderVersion {
		return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version)
	}
	if len(b.Body) == 0 {
		return ErrEmptyBody
	}
	if len(b.Body) > MaxBodyLen {
		return fmt.Errorf("%w: %d", ErrTooLarge, len(b.Body))
	}
	if MerkleRoot(b.Body) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	return nil
	// lesson 09 adds: proof-of-work check against Bits
}

// ----------------------------------------------------------------- chain ---

type Chain struct {
	byHash map[[32]byte]Block
	order  [][32]byte
	now    func() int64 // injectable, so tests are deterministic (lesson 34)
}

func Genesis() Block {
	var zero [32]byte
	body := []string{"The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"}
	return Block{
		Header: Header{
			Version: HeaderVersion, PrevHash: zero, MerkleRoot: MerkleRoot(body),
			Timestamp: 1231006505, Bits: 0x1f00ffff, Nonce: 0, Height: 0,
		},
		Body: body,
	}
}

func NewChain(now func() int64) *Chain {
	g := Genesis()
	c := &Chain{byHash: map[[32]byte]Block{}, now: now}
	h := g.Header.Hash()
	c.byHash[h] = g
	c.order = append(c.order, h)
	return c
}

func (c *Chain) Len() int   { return len(c.order) }
func (c *Chain) Tip() Block { return c.byHash[c.order[len(c.order)-1]] }
func (c *Chain) Get(h [32]byte) (Block, bool) {
	b, ok := c.byHash[h]
	return b, ok
}

func (c *Chain) MedianTimePast() int64 {
	n := MTPWindow
	if len(c.order) < n {
		n = len(c.order)
	}
	ts := make([]int64, 0, n)
	for _, h := range c.order[len(c.order)-n:] {
		ts = append(ts, c.byHash[h].Header.Timestamp)
	}
	sort.Slice(ts, func(i, j int) bool { return ts[i] < ts[j] })
	return ts[len(ts)/2]
}

// Append validates fully, then mutates. Never the other way round.
func (c *Chain) Append(b Block) error {
	if err := CheckStateless(b); err != nil {
		return err
	}
	h := b.Header.Hash()
	if _, ok := c.byHash[h]; ok {
		return ErrDuplicate
	}
	parent, ok := c.byHash[b.Header.PrevHash]
	if !ok {
		return fmt.Errorf("%w: %x", ErrOrphan, b.Header.PrevHash[:4])
	}
	if b.Header.Height != parent.Header.Height+1 {
		return fmt.Errorf("%w: %d after %d", ErrBadHeight, b.Header.Height, parent.Header.Height)
	}
	if mtp := c.MedianTimePast(); b.Header.Timestamp <= mtp {
		return fmt.Errorf("%w: %d <= %d", ErrTooOld, b.Header.Timestamp, mtp)
	}
	if limit := c.now() + MaxFutureSec; b.Header.Timestamp > limit {
		return fmt.Errorf("%w: %d > %d", ErrTooFuture, b.Header.Timestamp, limit)
	}
	c.byHash[h] = b
	c.order = append(c.order, h)
	return nil
	// lesson 14 replaces this linear chain with a block tree and fork choice
}

// NextBlock builds a valid successor to the tip — the miner's starting point.
func (c *Chain) NextBlock(txs []string) Block {
	p := c.Tip().Header
	return Block{
		Header: Header{
			Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot(txs),
			Timestamp: p.Timestamp + 600, Bits: p.Bits, Height: p.Height + 1,
		},
		Body: txs,
	}
	// lesson 09 adds: grind Nonce until Hash() is below the target
}

// ------------------------------------------------------------------ demo ---

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	// A fixed clock, so this output reproduces (lesson 34's rule, early).
	clock := int64(1231006505 + 100*600)
	c := NewChain(func() int64 { return clock })

	fmt.Printf("genesis  height %d  hash %s\n", c.Tip().Header.Height, short(c.Tip().Header.Hash()))
	fmt.Printf("         merkle %s\n\n", short(c.Tip().Header.MerkleRoot))

	for i := 1; i <= 5; i++ {
		b := c.NextBlock([]string{fmt.Sprintf("alice->bob %d", i), fmt.Sprintf("bob->carol %d", i)})
		if err := c.Append(b); err != nil {
			fmt.Println("append:", err)
			return
		}
	}

	fmt.Printf("%-8s %-18s %-18s %s\n", "height", "prev", "hash", "body")
	for _, h := range c.order {
		b := c.byHash[h]
		fmt.Printf("%-8d %-18s %-18s %v\n",
			b.Header.Height, short(b.Header.PrevHash), short(h), b.Body)
	}

	fmt.Printf("\nchain length %d, MTP %d\n", c.Len(), c.MedianTimePast())

	// Serialization round trip on the tip.
	raw := c.Tip().Header.Bytes()
	back, err := ParseHeader(raw)
	fmt.Printf("header round trip: %d bytes, err=%v, identical=%v\n",
		len(raw), err, back == c.Tip().Header)

	// A few rejections, to show the rules are live.
	fmt.Println("\nrejections:")
	bad := c.NextBlock([]string{"x"})
	bad.Header.Height = 99
	fmt.Printf("  wrong height : %v\n", c.Append(bad))

	orphan := c.NextBlock([]string{"x"})
	orphan.Header.PrevHash = [32]byte{7}
	fmt.Printf("  orphan       : %v\n", c.Append(orphan))

	future := c.NextBlock([]string{"x"})
	future.Header.Timestamp = clock + 10*3600
	fmt.Printf("  far future   : %v\n", c.Append(future))

	fmt.Printf("\nchain unchanged: %d blocks\n", c.Len())

	fmt.Println("\n--- what the next lessons add to exactly this code ---")
	fmt.Println("  09  Bits becomes a real target; NextBlock grinds Nonce to meet it")
	fmt.Println("  10  Body becomes []*Transaction with inputs, outputs and signatures")
	fmt.Println("  11  a mempool feeds NextBlock, and a wallet builds the transactions")
	fmt.Println("  12  byHash moves to disk, and Append becomes one atomic write")
	fmt.Println("  13  blocks arrive from peers, out of order, and ErrOrphan starts to matter")
	fmt.Println("  14  order becomes a TREE, and the tip is chosen by accumulated work")
	fmt.Println("  15  the UTXO set gives way to accounts, nonces and a state root")
}
```

**Output:**

```
genesis  height 0  hash a5238f69ca5a63f5
         merkle eee126eef2e24891

height   prev               hash               body
0        0000000000000000   a5238f69ca5a63f5   [The Times 03/Jan/2009 Chancellor on brink of second bailout for banks]
1        a5238f69ca5a63f5   b9761072f3c7e695   [alice->bob 1 bob->carol 1]
2        b9761072f3c7e695   2cdcf93dd58b1e35   [alice->bob 2 bob->carol 2]
3        2cdcf93dd58b1e35   7b28d6fee4171c65   [alice->bob 3 bob->carol 3]
4        7b28d6fee4171c65   3bd0904f9db408b3   [alice->bob 4 bob->carol 4]
5        3bd0904f9db408b3   521a067f5f3af540   [alice->bob 5 bob->carol 5]

chain length 6, MTP 1231008305
header round trip: 92 bytes, err=<nil>, identical=true

rejections:
  wrong height : height is not parent+1: 99 after 5
  orphan       : parent not found: 07000000
  far future   : timestamp too far in the future: 1231102505 > 1231073705

chain unchanged: 6 blocks

--- what the next lessons add to exactly this code ---
  09  Bits becomes a real target; NextBlock grinds Nonce to meet it
  10  Body becomes []*Transaction with inputs, outputs and signatures
  11  a mempool feeds NextBlock, and a wallet builds the transactions
  12  byHash moves to disk, and Append becomes one atomic write
  13  blocks arrive from peers, out of order, and ErrOrphan starts to matter
  14  order becomes a TREE, and the tip is chosen by accumulated work
  15  the UTXO set gives way to accounts, nonces and a state root
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
