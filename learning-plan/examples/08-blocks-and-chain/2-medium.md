# Step 08 — Blocks & the Chain · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Encode, decode, encode

`🟡 medium` · *Serialization*

The round-trip test is the one that catches a field-order mistake, and it is worth writing before anything that depends on the format. Note the decoder checks its length *first* — a decoder that trusts its input is a denial of service waiting to happen.

**Steps:**

1. Write `ParseHeader` as the exact inverse of `Bytes`.
2. Assert encode → decode → encode is byte-identical.
3. Note the struct is comparable with `==` because every field is.
4. Feed it wrong lengths and check the typed error.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"encoding/hex"
	"errors"
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

const HeaderSize = 92

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

var ErrShortHeader = errors.New("header too short")

// ParseHeader is the exact inverse of Bytes. It checks the length FIRST —
// a decoder that trusts its input is a denial-of-service waiting to happen
// (lesson 13).
func ParseHeader(b []byte) (Header, error) {
	var h Header
	if len(b) != HeaderSize {
		return h, fmt.Errorf("%w: got %d, want %d", ErrShortHeader, len(b), HeaderSize)
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

func main() {
	var prev, root [32]byte
	copy(prev[:], bytes.Repeat([]byte{0xab}, 32))
	copy(root[:], bytes.Repeat([]byte{0xcd}, 32))

	original := Header{
		Version: 1, PrevHash: prev, MerkleRoot: root,
		Timestamp: 1231006505, Bits: 0x1d00ffff, Nonce: 2083236893, Height: 42,
	}

	// encode -> decode -> encode, and compare the BYTES.
	first := original.Bytes()
	decoded, err := ParseHeader(first)
	if err != nil {
		fmt.Println("parse:", err)
		return
	}
	second := decoded.Bytes()

	fmt.Printf("encoded  %s\n", hex.EncodeToString(first)[:48]+"...")
	fmt.Printf("re-encoded %s\n", hex.EncodeToString(second)[:48]+"...")
	fmt.Printf("\nbyte-identical round trip: %v\n", bytes.Equal(first, second))
	fmt.Printf("struct equal too:          %v\n", decoded == original)

	// Struct equality works because every field is comparable — no slices,
	// no maps (lesson 03). That is a deliberate design choice.
	fmt.Println("\nHeader is comparable with == because every field is:")
	fmt.Println("  uint32, [32]byte, int64, uint64 — all comparable")
	fmt.Println("  a []byte field would break this, and break determinism too")

	// The decoder rejects anything that is not exactly the right length.
	fmt.Println("\nmalformed input:")
	for _, n := range []int{0, 91, 93} {
		_, err := ParseHeader(make([]byte, n))
		fmt.Printf("  %2d bytes -> %v (ErrShortHeader: %v)\n",
			n, err, errors.Is(err, ErrShortHeader))
	}

	// Round-tripping is the test that catches field-order mistakes: swap two
	// fields in Bytes but not in ParseHeader and this assertion fails.
	fmt.Println("\nthis round-trip test is what catches a field-order mistake.")
	fmt.Println("write it before you write anything that depends on the format.")
}
```

**Output:**

```
encoded  00000001abababababababababababababababababababab...
re-encoded 00000001abababababababababababababababababababab...

byte-identical round trip: true
struct equal too:          true

Header is comparable with == because every field is:
  uint32, [32]byte, int64, uint64 — all comparable
  a []byte field would break this, and break determinism too

malformed input:
   0 bytes -> header too short: got 0, want 92 (ErrShortHeader: true)
  91 bytes -> header too short: got 91, want 92 (ErrShortHeader: true)
  93 bytes -> header too short: got 93, want 92 (ErrShortHeader: true)

this round-trip test is what catches a field-order mistake.
write it before you write anything that depends on the format.
```

---

## 7. Why not JSON or gob

`🟡 medium` · *Serialization*

Two demonstrations of why reflection-based encoders cannot be used for anything you hash. `gob` encodes **field names**, so a rename changes the bytes. JSON emits struct fields in **declaration order**, so reordering them for readability changes every hash.

**Steps:**

1. Encode two structs differing only in a field name with `gob` and compare.
2. Encode two structs with reordered fields as JSON and compare.
3. Note JSON also has free choices in whitespace, escaping and number formatting.
4. Contrast with an explicit binary encoding that has none of those degrees of freedom.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/gob"
	"encoding/hex"
	"encoding/json"
	"fmt"
)

// Two structs with IDENTICAL data and identical field order.
// The only difference is a field NAME.
type HeaderV1 struct {
	Version   uint32
	Timestamp int64
	Height    uint64
}

type HeaderV2 struct {
	Version   uint32
	CreatedAt int64 // renamed from Timestamp — same type, same position
	Height    uint64
}

// And two structs with the same fields in a different ORDER.
type OrderA struct {
	Version uint32
	Height  uint64
}

type OrderB struct {
	Height  uint64
	Version uint32
}

func hashOf(b []byte) string {
	s := sha256.Sum256(b)
	return hex.EncodeToString(s[:8])
}

func main() {
	// ---- gob: carries type and FIELD NAMES ---------------------------------
	var g1, g2 bytes.Buffer
	gob.NewEncoder(&g1).Encode(HeaderV1{Version: 1, Timestamp: 1231006505, Height: 42})
	gob.NewEncoder(&g2).Encode(HeaderV2{Version: 1, CreatedAt: 1231006505, Height: 42})

	fmt.Println("gob — encodes the type description, including field names")
	fmt.Printf("  HeaderV1  %3d bytes  hash %s\n", g1.Len(), hashOf(g1.Bytes()))
	fmt.Printf("  HeaderV2  %3d bytes  hash %s\n", g2.Len(), hashOf(g2.Bytes()))
	fmt.Printf("  same data, one field RENAMED -> same bytes? %v\n",
		bytes.Equal(g1.Bytes(), g2.Bytes()))
	fmt.Println("  a rename is a refactor. With gob it is a chain split.")

	// ---- JSON: field order follows DECLARATION order ------------------------
	ja, _ := json.Marshal(OrderA{Version: 1, Height: 42})
	jb, _ := json.Marshal(OrderB{Version: 1, Height: 42})

	fmt.Println("\nJSON — emits struct fields in declaration order")
	fmt.Printf("  OrderA  %s\n", ja)
	fmt.Printf("  OrderB  %s\n", jb)
	fmt.Printf("  same data, fields REORDERED -> same bytes? %v\n", bytes.Equal(ja, jb))
	fmt.Println("  reordering fields for readability changes every hash.")

	// JSON has other freedoms too: whitespace, escaping, number formatting.
	compact := []byte(`{"Version":1,"Height":42}`)
	spaced := []byte(`{"Version": 1, "Height": 42}`)
	fmt.Printf("\n  and these are the same JSON value:\n    %s\n    %s\n", compact, spaced)
	fmt.Printf("  but hash to the same value: %v\n", hashOf(compact) == hashOf(spaced))

	// ---- explicit binary: none of those degrees of freedom -----------------
	explicit := func(version uint32, timestamp int64, height uint64) []byte {
		buf := new(bytes.Buffer)
		binary.Write(buf, binary.BigEndian, version)
		binary.Write(buf, binary.BigEndian, timestamp)
		binary.Write(buf, binary.BigEndian, height)
		return buf.Bytes()
	}
	e1 := explicit(1, 1231006505, 42)
	e2 := explicit(1, 1231006505, 42)
	fmt.Println("\nexplicit binary encoding")
	fmt.Printf("  %d bytes  %s\n", len(e1), hex.EncodeToString(e1))
	fmt.Printf("  deterministic: %v\n", bytes.Equal(e1, e2))
	fmt.Println("  no names, no order to choose, no whitespace, no escaping.")
	fmt.Println("  renaming a Go field changes nothing on the wire.")

	fmt.Println("\nthe rule from lesson 04, restated for blocks:")
	fmt.Println("  the serialization you HASH must be written by hand, field by")
	fmt.Println("  field, in a documented order. A reflection-based encoder ties")
	fmt.Println("  your consensus rules to your Go source code.")
}
```

**Output:**

```
gob — encodes the type description, including field names
  HeaderV1   73 bytes  hash 81ff4e13fec91307
  HeaderV2   74 bytes  hash d60a7d8c1bb71c38
  same data, one field RENAMED -> same bytes? false
  a rename is a refactor. With gob it is a chain split.

JSON — emits struct fields in declaration order
  OrderA  {"Version":1,"Height":42}
  OrderB  {"Height":42,"Version":1}
  same data, fields REORDERED -> same bytes? false
  reordering fields for readability changes every hash.

  and these are the same JSON value:
    {"Version":1,"Height":42}
    {"Version": 1, "Height": 42}
  but hash to the same value: false

explicit binary encoding
  20 bytes  0000000100000000495fab29000000000000002a
  deterministic: true
  no names, no order to choose, no whitespace, no escaping.
  renaming a Go field changes nothing on the wire.

the rule from lesson 04, restated for blocks:
  the serialization you HASH must be written by hand, field by
  field, in a documented order. A reflection-based encoder ties
  your consensus rules to your Go source code.
```

---

## 8. Tampering breaks every later block

`🟡 medium` · *Linking*

Edit one transaction and watch the damage propagate. Repairing the block's own Merkle root does not help — the break simply moves one block along, because the *next* block still stores the old hash. That is the tamper-evidence hash linking buys.

**Steps:**

1. Build a six-block chain and validate it.
2. Edit block 1's body and see the Merkle check fail.
3. Let the attacker repair the Merkle root, and see the failure move to block 2.
4. Compare every hash before and after to see how far the change reaches.

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

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func build(n int) []Block {
	var zero [32]byte
	chain := []Block{{
		Header: Header{Version: 1, PrevHash: zero, MerkleRoot: merkleRoot([]string{"genesis"}),
			Timestamp: 1231006505},
		Body: []string{"genesis"},
	}}
	for i := 1; i <= n; i++ {
		body := []string{fmt.Sprintf("alice->bob %d", i)}
		prev := chain[i-1].Header
		chain = append(chain, Block{
			Header: Header{Version: 1, PrevHash: prev.Hash(), MerkleRoot: merkleRoot(body),
				Timestamp: prev.Timestamp + 600, Height: prev.Height + 1},
			Body: body,
		})
	}
	return chain
}

// Validate reports the first block whose link or body commitment is broken.
func Validate(chain []Block) (int, error) {
	for i, b := range chain {
		if merkleRoot(b.Body) != b.Header.MerkleRoot {
			return i, fmt.Errorf("block %d: body does not match merkle root", i)
		}
		if i == 0 {
			continue
		}
		if b.Header.PrevHash != chain[i-1].Header.Hash() {
			return i, fmt.Errorf("block %d: prev hash does not match block %d", i, i-1)
		}
	}
	return -1, nil
}

func main() {
	chain := build(5)

	fmt.Printf("%-8s %-18s %-18s %s\n", "height", "prev", "hash", "body")
	for _, b := range chain {
		fmt.Printf("%-8d %-18s %-18s %v\n",
			b.Header.Height, short(b.Header.PrevHash), short(b.Header.Hash()), b.Body)
	}
	i, err := Validate(chain)
	fmt.Printf("\nvalid: %v (first failure at %d)\n", err == nil, i)

	// Rewrite history: change the amount in block 1's transaction.
	fmt.Println("\n--- tampering with block 1's body ---")
	chain[1].Body = []string{"alice->bob 1000000"}
	i, err = Validate(chain)
	fmt.Printf("valid: %v\n  %v\n", err == nil, err)

	// The attacker "fixes" the merkle root so block 1 is internally consistent.
	fmt.Println("\n--- attacker recomputes block 1's merkle root ---")
	chain[1].Header.MerkleRoot = merkleRoot(chain[1].Body)
	i, err = Validate(chain)
	fmt.Printf("valid: %v\n  %v\n", err == nil, err)
	fmt.Println("  block 1 is now self-consistent, but block 2 still points at")
	fmt.Println("  block 1's OLD hash. The break simply moved one block along.")

	// Show every downstream hash that changed.
	fresh := build(5)
	fmt.Println("\nhashes before and after the edit:")
	fmt.Printf("%-8s %-18s %-18s %s\n", "height", "original", "tampered", "changed")
	for j := range chain {
		o, t := fresh[j].Header.Hash(), chain[j].Header.Hash()
		fmt.Printf("%-8d %-18s %-18s %v\n", j, short(o), short(t), o != t)
	}

	// To make the chain valid again the attacker must redo EVERY later block.
	fmt.Println("\nto hide the edit the attacker must recompute block 1's hash,")
	fmt.Println("then block 2's PrevHash, then block 2's hash, and so on to the tip.")
	fmt.Println("with proof of work (lesson 09) each of those costs real money —")
	fmt.Println("which is what turns 'detectable' into 'infeasible'.")
}
```

**Output:**

```
height   prev               hash               body
0        0000000000000000   3816663bc4af88f8   [genesis]
1        3816663bc4af88f8   81126bb28c200112   [alice->bob 1]
2        81126bb28c200112   d8dbbcf64186e0b3   [alice->bob 2]
3        d8dbbcf64186e0b3   7c016e136e99d2d0   [alice->bob 3]
4        7c016e136e99d2d0   c03261b49de11f20   [alice->bob 4]
5        c03261b49de11f20   336609657d36e496   [alice->bob 5]

valid: true (first failure at -1)

--- tampering with block 1's body ---
valid: false
  block 1: body does not match merkle root

--- attacker recomputes block 1's merkle root ---
valid: false
  block 2: prev hash does not match block 1
  block 1 is now self-consistent, but block 2 still points at
  block 1's OLD hash. The break simply moved one block along.

hashes before and after the edit:
height   original           tampered           changed
0        3816663bc4af88f8   3816663bc4af88f8   false
1        81126bb28c200112   4e75346f085271bc   true
2        d8dbbcf64186e0b3   d8dbbcf64186e0b3   false
3        7c016e136e99d2d0   7c016e136e99d2d0   false
4        c03261b49de11f20   c03261b49de11f20   false
5        336609657d36e496   336609657d36e496   false

to hide the edit the attacker must recompute block 1's hash,
then block 2's PrevHash, then block 2's hash, and so on to the tip.
with proof of work (lesson 09) each of those costs real money —
which is what turns 'detectable' into 'infeasible'.
```

---

## 9. Height is convenient, not authoritative

`🟡 medium` · *Height*

Height is a number *inside* the header, so a block can claim any height it likes — nothing outside the block vouches for it. More importantly, two competing branches can share a height, so height can never answer "which chain is canonical?".

**Steps:**

1. Build a valid chain, then a block claiming height 9,999,999 with a correct link.
2. Build two different blocks at the same height from the same parent.
3. Understand why 100 easy blocks can be longer than 50 hard ones.
4. Read what height *is* good for: a parent check, an index, and scheduling.

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
	Version   uint32
	PrevHash  [32]byte
	Timestamp int64
	Bits      uint32
	Height    uint64
}

func (h Header) Bytes() []byte {
	buf := new(bytes.Buffer)
	binary.Write(buf, binary.BigEndian, h.Version)
	buf.Write(h.PrevHash[:])
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Bits)
	binary.Write(buf, binary.BigEndian, h.Height)
	return buf.Bytes()
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	// Height is stored in the header because it is convenient: it makes
	// "is this the next block?" a subtraction, and it indexes a database.
	var zero [32]byte
	g := Header{Version: 1, PrevHash: zero, Timestamp: 1000, Height: 0}
	b1 := Header{Version: 1, PrevHash: g.Hash(), Timestamp: 1600, Height: 1}
	b2 := Header{Version: 1, PrevHash: b1.Hash(), Timestamp: 2200, Height: 2}

	fmt.Println("a well-formed chain")
	for _, h := range []Header{g, b1, b2} {
		fmt.Printf("  height %d  prev %s  hash %s\n", h.Height, short(h.PrevHash), short(h.Hash()))
	}

	// But height is just a NUMBER IN THE HEADER. Nothing outside the block
	// vouches for it, so a block can claim any height it likes.
	liar := Header{Version: 1, PrevHash: b2.Hash(), Timestamp: 2800, Height: 9_999_999}
	fmt.Printf("\na block claiming height %d:\n", liar.Height)
	fmt.Printf("  prev hash correct: %v\n", liar.PrevHash == b2.Hash())
	fmt.Printf("  height plausible : %v\n", liar.Height == b2.Height+1)
	fmt.Println("  the LINK is sound; only the claimed height is nonsense.")
	fmt.Println("  so a node must check height against the parent, never trust it.")

	// The deeper point: two competing chains can have the same height.
	fmt.Println("\ntwo branches from the same parent, both height 2:")
	altB2 := Header{Version: 1, PrevHash: b1.Hash(), Timestamp: 2300, Height: 2}
	fmt.Printf("  branch A  %s\n", short(b2.Hash()))
	fmt.Printf("  branch B  %s\n", short(altB2.Hash()))
	fmt.Printf("  same height, same parent, different blocks: %v\n", b2.Hash() != altB2.Hash())

	fmt.Println("\nheight cannot answer 'which branch is canonical?'. It measures")
	fmt.Println("how many blocks there are, not how much effort they cost.")
	fmt.Println("\nunder variable difficulty a chain of 100 easy blocks can be LONGER")
	fmt.Println("than a chain of 50 hard ones, and much cheaper to produce. That is")
	fmt.Println("why fork choice uses ACCUMULATED WORK, not height (lesson 14).")

	fmt.Println("\nso what is height actually for?")
	fmt.Println("  - a cheap 'is this the next block' check against the parent")
	fmt.Println("  - a database index and a human-readable position")
	fmt.Println("  - scheduling consensus rule changes at a known point")
	fmt.Println("  it is a convenience, validated against the parent — never an authority.")
}
```

**Output:**

```
a well-formed chain
  height 0  prev 0000000000000000  hash e8448034e481baa9
  height 1  prev e8448034e481baa9  hash a66f985eae6d1809
  height 2  prev a66f985eae6d1809  hash 6038b7d53b0f0c0e

a block claiming height 9999999:
  prev hash correct: true
  height plausible : false
  the LINK is sound; only the claimed height is nonsense.
  so a node must check height against the parent, never trust it.

two branches from the same parent, both height 2:
  branch A  6038b7d53b0f0c0e
  branch B  7fcdb08d88b4040c
  same height, same parent, different blocks: true

height cannot answer 'which branch is canonical?'. It measures
how many blocks there are, not how much effort they cost.

under variable difficulty a chain of 100 easy blocks can be LONGER
than a chain of 50 hard ones, and much cheaper to produce. That is
why fork choice uses ACCUMULATED WORK, not height (lesson 14).

so what is height actually for?
  - a cheap 'is this the next block' check against the parent
  - a database index and a human-readable position
  - scheduling consensus rule changes at a known point
  it is a convenience, validated against the parent — never an authority.
```

---

## 10. Stateless and stateful validation

`🟡 medium` · *Validation*

Stateless checks need only the block and can run the moment it arrives. Stateful checks need the parent and must wait. Splitting them matters because a syncing node receives blocks out of order constantly — and the typed errors let the network layer decide what to *do*.

**Steps:**

1. Write the two passes separately, each returning wrapped sentinel errors.
2. Run eight one-mutation-away cases and see which pass catches each.
3. Match with `errors.Is` — an orphan gets queued, a bad version gets the peer dropped.
4. Note the network layer branches on the *kind* of failure, so it must be a value.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
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

// Typed errors, so the caller can decide what to DO. An orphan gets queued and
// its parent requested; a bad version gets the peer disconnected (lesson 13).
var (
	ErrBadVersion   = errors.New("unsupported version")
	ErrEmptyBody    = errors.New("block has no transactions")
	ErrBadMerkle    = errors.New("merkle root does not match body")
	ErrTooLarge     = errors.New("block exceeds size limit")
	ErrOrphan       = errors.New("parent not found")
	ErrBadHeight    = errors.New("height is not parent+1")
	ErrBadTimestamp = errors.New("timestamp not after parent")
)

const maxBodyLen = 4

// ValidateStateless needs nothing but the block itself. It can run the moment
// a block arrives, before you know whether you even have its parent.
func ValidateStateless(b Block) error {
	if b.Header.Version != 1 {
		return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version)
	}
	if len(b.Body) == 0 {
		return ErrEmptyBody
	}
	if len(b.Body) > maxBodyLen {
		return fmt.Errorf("%w: %d > %d", ErrTooLarge, len(b.Body), maxBodyLen)
	}
	if merkleRoot(b.Body) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	return nil
}

// ValidateStateful needs the parent, so it can only run once the chain has it.
func ValidateStateful(b Block, parent *Block) error {
	if parent == nil {
		return fmt.Errorf("%w: %x", ErrOrphan, b.Header.PrevHash[:4])
	}
	if b.Header.Height != parent.Header.Height+1 {
		return fmt.Errorf("%w: got %d, parent is %d", ErrBadHeight,
			b.Header.Height, parent.Header.Height)
	}
	if b.Header.Timestamp <= parent.Header.Timestamp {
		return fmt.Errorf("%w: %d <= %d", ErrBadTimestamp,
			b.Header.Timestamp, parent.Header.Timestamp)
	}
	return nil
}

func main() {
	var zero [32]byte
	genesis := Block{
		Header: Header{Version: 1, PrevHash: zero, MerkleRoot: merkleRoot([]string{"genesis"}),
			Timestamp: 1000, Height: 0},
		Body: []string{"genesis"},
	}

	good := func() Block {
		body := []string{"tx-a", "tx-b"}
		return Block{
			Header: Header{Version: 1, PrevHash: genesis.Header.Hash(),
				MerkleRoot: merkleRoot(body), Timestamp: 1600, Height: 1},
			Body: body,
		}
	}

	cases := []struct {
		name   string
		mutate func(Block) Block
		parent *Block
	}{
		{"valid", func(b Block) Block { return b }, &genesis},
		{"unsupported version", func(b Block) Block { b.Header.Version = 2; return b }, &genesis},
		{"empty body", func(b Block) Block { b.Body = nil; return b }, &genesis},
		{"body too large", func(b Block) Block {
			b.Body = []string{"a", "b", "c", "d", "e"}
			b.Header.MerkleRoot = merkleRoot(b.Body)
			return b
		}, &genesis},
		{"merkle mismatch", func(b Block) Block { b.Body = []string{"tx-a", "tx-X"}; return b }, &genesis},
		{"orphan", func(b Block) Block { return b }, nil},
		{"wrong height", func(b Block) Block { b.Header.Height = 7; return b }, &genesis},
		{"timestamp not after parent", func(b Block) Block { b.Header.Timestamp = 900; return b }, &genesis},
	}

	fmt.Printf("%-28s %-10s %s\n", "case", "pass", "error")
	for _, c := range cases {
		b := c.mutate(good())
		var err error
		stage := "stateless"
		if err = ValidateStateless(b); err == nil {
			stage = "stateful"
			err = ValidateStateful(b, c.parent)
		}
		msg := "-"
		if err != nil {
			msg = err.Error()
		}
		fmt.Printf("%-28s %-10s %s\n", c.name, stage, msg)
	}

	// Why the split matters: a node receives blocks out of order all the time.
	fmt.Println("\nwhy separate the two passes?")
	fmt.Println("  a node receives blocks OUT OF ORDER during sync. Stateless checks")
	fmt.Println("  run immediately and reject garbage before it costs you anything;")
	fmt.Println("  stateful checks wait until the parent arrives (lesson 13's orphan pool).")

	// And why typed errors, not strings.
	orphan := good()
	err := ValidateStateful(orphan, nil)
	fmt.Printf("\nerrors.Is(err, ErrOrphan)     = %v  -> queue it, request the parent\n",
		errors.Is(err, ErrOrphan))
	badVer := good()
	badVer.Header.Version = 99
	err = ValidateStateless(badVer)
	fmt.Printf("errors.Is(err, ErrBadVersion) = %v  -> drop it, maybe ban the peer\n",
		errors.Is(err, ErrBadVersion))
	fmt.Println("\nthe network layer branches on the KIND of failure, so the kind")
	fmt.Println("has to be a value — never a string you match on (lesson 02).")
}
```

**Output:**

```
case                         pass       error
valid                        stateful   -
unsupported version          stateless  unsupported version: 2
empty body                   stateless  block has no transactions
body too large               stateless  block exceeds size limit: 5 > 4
merkle mismatch              stateless  merkle root does not match body
orphan                       stateful   parent not found: 1c525a41
wrong height                 stateful   height is not parent+1: got 7, parent is 0
timestamp not after parent   stateful   timestamp not after parent: 900 <= 1000

why separate the two passes?
  a node receives blocks OUT OF ORDER during sync. Stateless checks
  run immediately and reject garbage before it costs you anything;
  stateful checks wait until the parent arrives (lesson 13's orphan pool).

errors.Is(err, ErrOrphan)     = true  -> queue it, request the parent
errors.Is(err, ErrBadVersion) = true  -> drop it, maybe ban the peer

the network layer branches on the KIND of failure, so the kind
has to be a value — never a string you match on (lesson 02).
```

---

## 11. The cached-hash bug

`🔴 medium` · *Traps*

Caching a block's hash is the obvious optimisation and it is a trap: mutate a field afterwards and the block reports a hash that is not its hash. Worse, it validates against *itself* consistently, so the bug only surfaces when a peer recomputes from the bytes.

**Steps:**

1. Cache a hash, mutate `Nonce`, and watch the stale value come back.
2. Try the `sync.Once` version and see it make the staleness permanent.
3. Seal the block on construction instead, returning header copies.
4. Read the three workable options, and why lesson 09's miner uses none of them.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"sync"
)

type Header struct {
	Version   uint32
	PrevHash  [32]byte
	Timestamp int64
	Nonce     uint32
	Height    uint64
}

func (h Header) Bytes() []byte {
	buf := new(bytes.Buffer)
	binary.Write(buf, binary.BigEndian, h.Version)
	buf.Write(h.PrevHash[:])
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Nonce)
	binary.Write(buf, binary.BigEndian, h.Height)
	return buf.Bytes()
}

func compute(h Header) [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

// ---------------------------------------------------------------------------
// The tempting optimisation: hashing is not free, and a block's hash is asked
// for constantly. So cache it.

type CachedBlock struct {
	Header Header
	cached *[32]byte
}

func (b *CachedBlock) Hash() [32]byte {
	if b.cached == nil {
		h := compute(b.Header)
		b.cached = &h
	}
	return *b.cached
}

// The sync.Once variant has exactly the same bug, with more ceremony.
type OnceBlock struct {
	Header Header
	once   sync.Once
	hash   [32]byte
}

func (b *OnceBlock) Hash() [32]byte {
	b.once.Do(func() { b.hash = compute(b.Header) })
	return b.hash
}

// The fix that actually works: never mutate a block after construction.
// Build it once, hash it once, and expose no setters.
type SealedBlock struct {
	header Header
	hash   [32]byte
}

func NewSealedBlock(h Header) SealedBlock {
	return SealedBlock{header: h, hash: compute(h)}
}

func (b SealedBlock) Header() Header { return b.header } // a copy
func (b SealedBlock) Hash() [32]byte { return b.hash }

func main() {
	h := Header{Version: 1, Timestamp: 1000, Nonce: 1, Height: 1}

	// --- the bug -----------------------------------------------------------
	cb := &CachedBlock{Header: h}
	before := cb.Hash()
	cb.Header.Nonce = 2 // a miner does exactly this, millions of times
	after := cb.Hash()

	fmt.Println("cached hash, then mutate a field")
	fmt.Printf("  before  %s\n", short(before))
	fmt.Printf("  after   %s\n", short(after))
	fmt.Printf("  hash changed after mutating Nonce? %v   <- it should have\n", before != after)
	fmt.Printf("  truth   %s\n", short(compute(cb.Header)))
	fmt.Println("  the block now reports a hash that is not its hash.")

	// sync.Once does not help.
	ob := &OnceBlock{Header: h}
	obBefore := ob.Hash()
	ob.Header.Nonce = 2
	fmt.Printf("\nsync.Once version: still stale: %v\n", ob.Hash() == obBefore)
	fmt.Println("  Once guarantees the computation runs once. That is the problem,")
	fmt.Println("  not the solution — it makes the staleness permanent.")

	// --- why it is so damaging ---------------------------------------------
	fmt.Println("\nwhy this one is nasty")
	fmt.Println("  the block validates against itself, because everything downstream")
	fmt.Println("  asks Hash() and gets the same wrong answer consistently.")
	fmt.Println("  it only fails when a PEER recomputes from the bytes — so it shows")
	fmt.Println("  up as 'that node is sending invalid blocks', far from the cause.")

	// --- the fix -----------------------------------------------------------
	sealed := NewSealedBlock(h)
	fmt.Printf("\nsealed block hash %s\n", short(sealed.Hash()))
	hdr := sealed.Header() // a COPY, because Header is a value type (lesson 03)
	hdr.Nonce = 2
	fmt.Printf("caller mutates the returned header: sealed hash still %s\n", short(sealed.Hash()))
	fmt.Printf("and it is still correct: %v\n", sealed.Hash() == compute(sealed.header))

	fmt.Println("\nthree workable options, in order of preference:")
	fmt.Println("  1. do not cache — hashing 92 bytes is ~1 microsecond")
	fmt.Println("  2. seal on construction, expose no mutators (above)")
	fmt.Println("  3. cache, but invalidate in every single mutator — and accept")
	fmt.Println("     that the next person to add a field will forget")

	fmt.Println("\nthe miner in lesson 09 mutates Nonce billions of times, so it")
	fmt.Println("does not use a block type at all: it patches 4 bytes in a reusable")
	fmt.Println("buffer and rehashes (lesson 04, example 19).")
}
```

**Output:**

```
cached hash, then mutate a field
  before  b8509a40c8c14e16
  after   b8509a40c8c14e16
  hash changed after mutating Nonce? false   <- it should have
  truth   f78817e30bda837e
  the block now reports a hash that is not its hash.

sync.Once version: still stale: true
  Once guarantees the computation runs once. That is the problem,
  not the solution — it makes the staleness permanent.

why this one is nasty
  the block validates against itself, because everything downstream
  asks Hash() and gets the same wrong answer consistently.
  it only fails when a PEER recomputes from the bytes — so it shows
  up as 'that node is sending invalid blocks', far from the cause.

sealed block hash b8509a40c8c14e16
caller mutates the returned header: sealed hash still b8509a40c8c14e16
and it is still correct: true

three workable options, in order of preference:
  1. do not cache — hashing 92 bytes is ~1 microsecond
  2. seal on construction, expose no mutators (above)
  3. cache, but invalidate in every single mutator — and accept
     that the next person to add a field will forget

the miner in lesson 09 mutates Nonce billions of times, so it
does not use a block type at all: it patches 4 bytes in a reusable
buffer and rehashes (lesson 04, example 19).
```

---

## 12. Timestamps are int64, not time.Time

`🟡 medium` · *Timestamps*

A `time.Time` from `time.Now()` carries a hidden monotonic reading, so two values representing the same instant are not `==`. That makes it unusable in anything you hash. Store `int64` Unix seconds and convert only at the edges.

**Steps:**

1. Compare `now == now.Round(0)` with `now.Equal(now.Round(0))`.
2. Encode the same instant three different ways and confirm identical bytes.
3. Convert back to a `time.Time` for display and back again.
4. Note Bitcoin's `uint32` overflows in 2106; `int64` costs four bytes and does not.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"time"
)

// The header stores Unix SECONDS, not a time.Time. This example is why.
type Header struct {
	Version   uint32
	Timestamp int64
	Height    uint64
}

func (h Header) Bytes() []byte {
	buf := new(bytes.Buffer)
	binary.Write(buf, binary.BigEndian, h.Version)
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Height)
	return buf.Bytes()
}

func main() {
	// A time.Time from time.Now() carries TWO clocks: a wall clock and an
	// opaque monotonic reading used for measuring elapsed time.
	now := time.Now()
	stripped := now.Round(0) // Round(0) strips the monotonic reading

	fmt.Println("time.Now() carries a hidden monotonic reading")
	fmt.Printf("  now == now.Round(0):        %v\n", now == stripped)
	fmt.Printf("  now.Equal(now.Round(0)):    %v\n", now.Equal(stripped))
	fmt.Println("  == compares the monotonic reading; Equal compares the instant.")
	fmt.Println("  two values that represent the SAME moment are not ==.")

	// That alone makes time.Time unusable as a hashed field: two nodes with
	// the same wall-clock instant produce different structs.
	fmt.Println("\nwhy that is fatal for a header")
	fmt.Println("  a struct containing time.Time is not reliably comparable")
	fmt.Println("  and any reflection-based encoder may or may not include the")
	fmt.Println("  monotonic part — so the bytes you hash depend on how the value")
	fmt.Println("  was constructed, not on what it means.")

	// Unix seconds have none of those problems.
	const fixed = 1231006505
	h1 := Header{Version: 1, Timestamp: fixed, Height: 0}
	h2 := Header{Version: 1, Timestamp: time.Unix(fixed, 0).Unix(), Height: 0}
	h3 := Header{Version: 1, Timestamp: time.Unix(fixed, 999999999).Unix(), Height: 0}

	fmt.Printf("\nint64 Unix seconds\n")
	fmt.Printf("  literal            %s\n", hex.EncodeToString(h1.Bytes()))
	fmt.Printf("  via time.Unix      %s\n", hex.EncodeToString(h2.Bytes()))
	fmt.Printf("  with 999ms dropped %s\n", hex.EncodeToString(h3.Bytes()))
	fmt.Printf("  all identical: %v\n",
		bytes.Equal(h1.Bytes(), h2.Bytes()) && bytes.Equal(h2.Bytes(), h3.Bytes()))

	// And it round-trips to a real time for display.
	t := time.Unix(h1.Timestamp, 0).UTC()
	fmt.Printf("\nfor display: %s\n", t.Format(time.RFC3339))
	fmt.Printf("back to the wire value: %d (unchanged: %v)\n", t.Unix(), t.Unix() == h1.Timestamp)

	// The rule.
	fmt.Println("\nthe rule")
	fmt.Println("  store int64 Unix seconds in anything you hash or serialize")
	fmt.Println("  convert to time.Time only at the edges, for display and arithmetic")
	fmt.Println("  if you must keep a time.Time in a struct, call .Round(0) first —")
	fmt.Println("  but prefer not to keep one at all")

	fmt.Println("\nBitcoin uses a uint32 of Unix seconds, which overflows in 2106.")
	fmt.Println("We use int64 because there is no reason not to.")
}
```

**Output:**

```
time.Now() carries a hidden monotonic reading
  now == now.Round(0):        false
  now.Equal(now.Round(0)):    true
  == compares the monotonic reading; Equal compares the instant.
  two values that represent the SAME moment are not ==.

why that is fatal for a header
  a struct containing time.Time is not reliably comparable
  and any reflection-based encoder may or may not include the
  monotonic part — so the bytes you hash depend on how the value
  was constructed, not on what it means.

int64 Unix seconds
  literal            0000000100000000495fab290000000000000000
  via time.Unix      0000000100000000495fab290000000000000000
  with 999ms dropped 0000000100000000495fab290000000000000000
  all identical: true

for display: 2009-01-03T18:15:05Z
back to the wire value: 1231006505 (unchanged: true)

the rule
  store int64 Unix seconds in anything you hash or serialize
  convert to time.Time only at the edges, for display and arithmetic
  if you must keep a time.Time in a struct, call .Round(0) first —
  but prefer not to keep one at all

Bitcoin uses a uint32 of Unix seconds, which overflows in 2106.
We use int64 because there is no reason not to.
```

---

## 13. Median-time-past

`🟡 medium` · *Timestamps*

A block timestamp is not a clock reading — miners choose it and have incentives to lie. The protocol bounds it on both sides: median-time-past below (which a single liar cannot move) and a couple of hours of drift above.

**Steps:**

1. Build an 11-block history where one miner's clock is wildly ahead.
2. Compute the median and watch it ignore the outlier entirely.
3. Test six candidate timestamps against both bounds.
4. Read why each bound exists — the lower one prevents the timewarp attack.

```go
package main

import (
	"errors"
	"fmt"
	"sort"
	"time"
)

// A block timestamp is not a clock reading. Miners choose it, and they have
// incentives to lie a little. So the protocol bounds it from both sides.
const (
	mtpWindow    = 11          // blocks used for median-time-past
	maxFutureSec = 2 * 60 * 60 // 2 hours of allowed clock drift
)

var (
	ErrTooOld    = errors.New("timestamp at or before median-time-past")
	ErrTooFuture = errors.New("timestamp too far in the future")
)

// medianTimePast is the median of the last 11 block timestamps. Using the
// median rather than the parent's timestamp means a single lying miner
// cannot move the lower bound.
func medianTimePast(timestamps []int64) int64 {
	n := mtpWindow
	if len(timestamps) < n {
		n = len(timestamps)
	}
	recent := append([]int64(nil), timestamps[len(timestamps)-n:]...)
	sort.Slice(recent, func(i, j int) bool { return recent[i] < recent[j] })
	return recent[len(recent)/2]
}

func checkTimestamp(ts int64, history []int64, now int64) error {
	if mtp := medianTimePast(history); ts <= mtp {
		return fmt.Errorf("%w: %d <= %d", ErrTooOld, ts, mtp)
	}
	if ts > now+maxFutureSec {
		return fmt.Errorf("%w: %d > %d", ErrTooFuture, ts, now+maxFutureSec)
	}
	return nil
}

func main() {
	// A chain whose last 11 timestamps mostly advance by ten minutes,
	// with one miner who set a wildly wrong clock.
	base := int64(1700000000)
	history := []int64{}
	for i := 0; i < 11; i++ {
		history = append(history, base+int64(i)*600)
	}
	history[7] = base + 100000 // one miner's clock is far ahead

	fmt.Println("last 11 block timestamps (one miner is lying):")
	for i, t := range history {
		note := ""
		if i == 7 {
			note = "  <- clock far ahead"
		}
		fmt.Printf("  %2d  %d  %s%s\n", i, t, time.Unix(t, 0).UTC().Format("15:04:05"), note)
	}

	mtp := medianTimePast(history)
	fmt.Printf("\nmedian-time-past  %d  (%s)\n", mtp, time.Unix(mtp, 0).UTC().Format("15:04:05"))
	fmt.Println("  the median ignores the outlier entirely — that is the point.")
	fmt.Printf("  the parent's timestamp is %d, which the liar could have set\n", history[10])

	now := base + 6000
	fmt.Printf("\nnode's own clock: %d\n\n", now)

	cases := []struct {
		name string
		ts   int64
	}{
		{"just after MTP", mtp + 1},
		{"exactly MTP", mtp},
		{"before MTP (backdated)", mtp - 3600},
		{"a bit ahead of now", now + 600},
		{"2 hours ahead", now + maxFutureSec},
		{"3 hours ahead", now + 3*3600},
	}
	fmt.Printf("%-26s %-12s %s\n", "candidate", "accepted", "reason")
	for _, c := range cases {
		err := checkTimestamp(c.ts, history, now)
		reason := "-"
		if err != nil {
			reason = err.Error()
		}
		fmt.Printf("%-26s %-12v %s\n", c.name, err == nil, reason)
	}

	fmt.Println("\nwhy both bounds exist")
	fmt.Println("  lower (MTP)  stops a miner backdating blocks to make the")
	fmt.Println("               difficulty retarget think blocks were slow —")
	fmt.Println("               the TIMEWARP attack (lesson 09)")
	fmt.Println("  upper (2h)   stops a miner dating blocks far in the future to")
	fmt.Println("               push difficulty down, and bounds clock disagreement")

	fmt.Println("\nnote what this is NOT: a timestamp is not evidence of when a")
	fmt.Println("block was made. It is a loosely-bounded value the protocol needs")
	fmt.Println("for difficulty and timelocks, and nothing more. Never use a block")
	fmt.Println("timestamp as a clock in a contract (lesson 27, 53).")
}
```

**Output:**

```
last 11 block timestamps (one miner is lying):
   0  1700000000  22:13:20
   1  1700000600  22:23:20
   2  1700001200  22:33:20
   3  1700001800  22:43:20
   4  1700002400  22:53:20
   5  1700003000  23:03:20
   6  1700003600  23:13:20
   7  1700100000  02:00:00  <- clock far ahead
   8  1700004800  23:33:20
   9  1700005400  23:43:20
  10  1700006000  23:53:20

median-time-past  1700003000  (23:03:20)
  the median ignores the outlier entirely — that is the point.
  the parent's timestamp is 1700006000, which the liar could have set

node's own clock: 1700006000

candidate                  accepted     reason
just after MTP             true         -
exactly MTP                false        timestamp at or before median-time-past: 1700003000 <= 1700003000
before MTP (backdated)     false        timestamp at or before median-time-past: 1699999400 <= 1700003000
a bit ahead of now         true         -
2 hours ahead              true         -
3 hours ahead              false        timestamp too far in the future: 1700016800 > 1700013200

why both bounds exist
  lower (MTP)  stops a miner backdating blocks to make the
               difficulty retarget think blocks were slow —
               the TIMEWARP attack (lesson 09)
  upper (2h)   stops a miner dating blocks far in the future to
               push difficulty down, and bounds clock disagreement

note what this is NOT: a timestamp is not evidence of when a
block was made. It is a loosely-bounded value the protocol needs
for difficulty and timelocks, and nothing more. Never use a block
timestamp as a clock in a contract (lesson 27, 53).
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
