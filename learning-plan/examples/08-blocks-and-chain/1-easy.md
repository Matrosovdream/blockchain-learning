# Step 08 — Blocks & the Chain · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. The header, field by field

`🟢 easy` · *The header*

Seven fixed-width fields, in a fixed order, big-endian, with no padding and no length prefixes. That is the entire design, and it is chosen so that byte 4 is *always* the first byte of `PrevHash` — on every machine, in every Go version.

**Steps:**

1. Define the `Header` struct with only comparable, fixed-size field types.
2. Write `Bytes()` by hand, field by field, in a documented order.
3. Print the offset and value of every field.
4. Note there is nothing optional anywhere — that is what makes the hash reproducible.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// Header is everything a node needs to validate a block's place in the chain.
// Fixed-size fields only: no strings, no slices, no maps. That is what makes
// the serialization below deterministic (lesson 04, topic 6).
type Header struct {
	Version    uint32   // format version, so the layout can change later
	PrevHash   [32]byte // the previous header's hash — the "chain" in blockchain
	MerkleRoot [32]byte // commits to the body (lesson 05)
	Timestamp  int64    // Unix seconds, not a time.Time (example 12)
	Bits       uint32   // the difficulty target, compact form (lesson 09)
	Nonce      uint32   // what a miner grinds (lesson 09)
	Height     uint64   // convenient, but NOT authoritative (lesson 14)
}

// HeaderSize is the exact byte length of a serialized header.
const HeaderSize = 4 + 32 + 32 + 8 + 4 + 4 + 8

// Bytes writes the header in a fixed field order, big-endian, with no padding.
// Every node must produce these exact bytes or the hashes will not match.
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

func main() {
	var prev, root [32]byte
	copy(root[:], []byte("a merkle root goes here.........."))

	h := Header{
		Version:    1,
		PrevHash:   prev,
		MerkleRoot: root,
		Timestamp:  1231006505, // fixed, so this example reproduces
		Bits:       0x1d00ffff,
		Nonce:      2083236893,
		Height:     0,
	}

	raw := h.Bytes()
	fmt.Printf("header is %d bytes (constant, always)\n\n", len(raw))

	// Walk the layout field by field.
	fmt.Printf("%-12s %-6s %s\n", "field", "bytes", "value")
	fmt.Printf("%-12s %-6d %x\n", "Version", 4, raw[0:4])
	fmt.Printf("%-12s %-6d %x\n", "PrevHash", 32, raw[4:36])
	fmt.Printf("%-12s %-6d %x\n", "MerkleRoot", 32, raw[36:68])
	fmt.Printf("%-12s %-6d %x\n", "Timestamp", 8, raw[68:76])
	fmt.Printf("%-12s %-6d %x\n", "Bits", 4, raw[76:80])
	fmt.Printf("%-12s %-6d %x\n", "Nonce", 4, raw[80:84])
	fmt.Printf("%-12s %-6d %x\n", "Height", 8, raw[84:92])

	fmt.Printf("\nfull encoding\n  %s\n", hex.EncodeToString(raw))

	// Everything is fixed-width, so the offsets never move.
	fmt.Println("\nno length prefixes, no optional fields, no padding —")
	fmt.Println("byte 4 is ALWAYS the first byte of PrevHash.")
	fmt.Println("that is what makes the hash reproducible on every machine.")
}
```

**Output:**

```
header is 92 bytes (constant, always)

field        bytes  value
Version      4      00000001
PrevHash     32     0000000000000000000000000000000000000000000000000000000000000000
MerkleRoot   32     61206d65726b6c6520726f6f7420676f657320686572652e2e2e2e2e2e2e2e2e
Timestamp    8      00000000495fab29
Bits         4      1d00ffff
Nonce        4      7c2bac1d
Height       8      0000000000000000

full encoding
  00000001000000000000000000000000000000000000000000000000000000000000000061206d65726b6c6520726f6f7420676f657320686572652e2e2e2e2e2e2e2e2e00000000495fab291d00ffff7c2bac1d0000000000000000

no length prefixes, no optional fields, no padding —
byte 4 is ALWAYS the first byte of PrevHash.
that is what makes the hash reproducible on every machine.
```

---

## 2. Hashing a header

`🟢 easy` · *The header*

The block's identity is the hash of its header. There is no id field and no sequence number the block carries — the content names it, exactly as in lesson 04. Change any field and the hash changes completely.

**Steps:**

1. Double-SHA256 the header bytes, as Bitcoin does.
2. Confirm the hash is stable across calls.
3. Mutate each field in turn and watch every hash change.
4. Note `Hash()` takes a value receiver, so it cannot accidentally modify the block.

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

// Hash is the block's identity. Double SHA-256, as Bitcoin does (lesson 04).
func (h Header) Hash() [32]byte {
	first := sha256.Sum256(h.Bytes())
	return sha256.Sum256(first[:])
}

func main() {
	h := Header{Version: 1, Timestamp: 1231006505, Bits: 0x1d00ffff, Nonce: 2083236893}

	fmt.Printf("hash %s\n", hashHex(h.Hash()))

	// The hash is a function of the bytes, so it is deterministic.
	a, b := h.Hash(), h.Hash()
	fmt.Printf("stable across calls: %v\n", a == b)

	// Change ANY field and the hash changes completely (lesson 04, avalanche).
	fmt.Println("\nchanging one field at a time:")
	variants := []struct {
		name   string
		mutate func(Header) Header
	}{
		{"Nonce +1", func(x Header) Header { x.Nonce++; return x }},
		{"Timestamp +1", func(x Header) Header { x.Timestamp++; return x }},
		{"Height +1", func(x Header) Header { x.Height++; return x }},
		{"PrevHash byte 0", func(x Header) Header { x.PrevHash[0] = 1; return x }},
	}
	base := h.Hash()
	for _, v := range variants {
		got := v.mutate(h).Hash()
		fmt.Printf("  %-16s %s  differs: %v\n",
			v.name, hex.EncodeToString(got[:8]), got != base)
	}

	// The header is a VALUE, so mutating a copy leaves the original alone
	// (lesson 03, arrays copy). That property is why Hash() takes a value
	// receiver and cannot accidentally change the block.
	fmt.Printf("\noriginal unchanged: %v\n", h.Hash() == base)

	fmt.Println("\nthe block's identity IS this hash. There is no id field,")
	fmt.Println("no sequence number the block carries — the content names it")
	fmt.Println("(lesson 04, 'the hash is the name').")
}

// hashHex is a small convenience: a [32]byte returned from a function is not
// addressable, so h.Hash()[:] does not compile (lesson 03, arrays vs slices).
func hashHex(h [32]byte) string { return hex.EncodeToString(h[:]) }
```

**Output:**

```
hash 8e49a8b2e90ec5f378fe0352fd05f4cd7b196b09452469996ac66d3efbc5dfea
stable across calls: true

changing one field at a time:
  Nonce +1         73e7c7ca154f32ff  differs: true
  Timestamp +1     c358c7677e7bcb00  differs: true
  Height +1        237c9d14cbd4a183  differs: true
  PrevHash byte 0  f0af0d988a3781cf  differs: true

original unchanged: true

the block's identity IS this hash. There is no id field,
no sequence number the block carries — the content names it
(lesson 04, 'the hash is the name').
```

---

## 3. The genesis block

`🟢 easy` · *Genesis*

The first block has no parent, so nothing can validate it. Every node simply has to agree on it byte-for-byte — two nodes with different genesis blocks are on different networks and will never agree on anything.

**Steps:**

1. Build a genesis block with an all-zero `PrevHash`.
2. Confirm `Genesis()` is deterministic.
3. Pin its hash in a constant, the way a test would.
4. Read why Bitcoin's genesis carries a newspaper headline.

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
	first := sha256.Sum256(h.Bytes())
	return sha256.Sum256(first[:])
}

type Block struct {
	Header Header
	Body   []string // stand-in for transactions until lesson 10
}

// Genesis is hardcoded. It has no parent, so nothing can validate it —
// every node simply has to agree on it, byte for byte.
func Genesis() Block {
	var zero [32]byte
	body := []string{"The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"}

	// A single-item Merkle root is just the hash of that item (lesson 05).
	leaf := sha256.Sum256([]byte(body[0]))
	root := sha256.Sum256(leaf[:])

	return Block{
		Header: Header{
			Version:    1,
			PrevHash:   zero, // no parent
			MerkleRoot: root,
			Timestamp:  1231006505,
			Bits:       0x1d00ffff,
			Nonce:      2083236893,
			Height:     0,
		},
		Body: body,
	}
}

func main() {
	g := Genesis()

	fmt.Printf("genesis hash   %s\n", hashHex(g.Header.Hash()))
	fmt.Printf("prev hash      %s\n", hex.EncodeToString(g.Header.PrevHash[:]))
	fmt.Printf("  ^ all zeros: there is nothing before it\n")
	fmt.Printf("height         %d\n", g.Header.Height)
	fmt.Printf("body           %q\n", g.Body[0])

	// Determinism is the whole point. Two calls, same bytes.
	fmt.Printf("\nGenesis() is deterministic: %v\n", Genesis().Header.Hash() == g.Header.Hash())

	// This is what a test pins. If someone changes a field, the test fails
	// loudly instead of the network splitting quietly.
	const want = "0b01791f2ff379496bf3599d2f88adf827bcb3bac5cbd4a2806e55de11ab727c"
	got := hashHex(g.Header.Hash())
	fmt.Printf("\npinned in a test: %v\n", got == want)
	if got != want {
		fmt.Printf("  (this build produces %s — update the pin deliberately, never casually)\n", got)
	}

	fmt.Println("\nwhy genesis is special")
	fmt.Println("  it has no parent, so the 'parent must exist' rule cannot apply")
	fmt.Println("  it is not mined by anyone — it is written into the source")
	fmt.Println("  two nodes with different genesis blocks are on different networks,")
	fmt.Println("  and will simply never agree on anything")

	fmt.Println("\nBitcoin's genesis coinbase carries a newspaper headline, which")
	fmt.Println("proves the chain was not started before that date. Ethereum's")
	fmt.Println("genesis instead carries an allocation of pre-funded accounts.")
}

// hashHex is a small convenience: a [32]byte returned from a function is not
// addressable, so h.Hash()[:] does not compile (lesson 03, arrays vs slices).
func hashHex(h [32]byte) string { return hex.EncodeToString(h[:]) }
```

**Output:**

```
genesis hash   0b01791f2ff379496bf3599d2f88adf827bcb3bac5cbd4a2806e55de11ab727c
prev hash      0000000000000000000000000000000000000000000000000000000000000000
  ^ all zeros: there is nothing before it
height         0
body           "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"

Genesis() is deterministic: true

pinned in a test: true

why genesis is special
  it has no parent, so the 'parent must exist' rule cannot apply
  it is not mined by anyone — it is written into the source
  two nodes with different genesis blocks are on different networks,
  and will simply never agree on anything

Bitcoin's genesis coinbase carries a newspaper headline, which
proves the chain was not started before that date. Ethereum's
genesis instead carries an allocation of pre-funded accounts.
```

---

## 4. Chaining three blocks

`🟢 easy` · *Linking*

Each header stores the previous header's hash. That is the whole of "chain" — not a slice, not a pointer, but a link made of content. Walking backwards is just following `PrevHash` until you reach the block that has none.

**Steps:**

1. Build genesis, then three blocks each pointing at its parent's hash.
2. Print the prev/hash columns and see them line up.
3. Index the blocks by hash and walk from the tip back to genesis.
4. Note the order cannot be rearranged without every later hash changing.

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
	first := sha256.Sum256(h.Bytes())
	return sha256.Sum256(first[:])
}

func merkleRoot(items []string) [32]byte {
	if len(items) == 0 {
		return sha256.Sum256(nil)
	}
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = sha256.Sum256([]byte(it))
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

func main() {
	// Genesis: no parent, so PrevHash is zero.
	var zero [32]byte
	chain := []Block{{
		Header: Header{Version: 1, PrevHash: zero, MerkleRoot: merkleRoot([]string{"genesis"}),
			Timestamp: 1231006505, Height: 0},
		Body: []string{"genesis"},
	}}

	// Each subsequent block points at the previous header's hash.
	for i := 1; i <= 3; i++ {
		body := []string{fmt.Sprintf("tx-%d-a", i), fmt.Sprintf("tx-%d-b", i)}
		prev := chain[i-1].Header
		chain = append(chain, Block{
			Header: Header{
				Version:    1,
				PrevHash:   prev.Hash(), // <- the link
				MerkleRoot: merkleRoot(body),
				Timestamp:  prev.Timestamp + 600, // ten minutes later
				Height:     prev.Height + 1,
			},
			Body: body,
		})
	}

	fmt.Printf("%-8s %-18s %-18s %s\n", "height", "prev", "hash", "body")
	for _, b := range chain {
		fmt.Printf("%-8d %-18s %-18s %v\n",
			b.Header.Height, short(b.Header.PrevHash), short(b.Header.Hash()), b.Body)
	}

	// Walking the chain backwards is just following PrevHash.
	fmt.Println("\nwalking back from the tip:")
	byHash := map[[32]byte]Block{}
	for _, b := range chain {
		byHash[b.Header.Hash()] = b
	}
	cur := chain[len(chain)-1]
	for {
		fmt.Printf("  height %d  %s\n", cur.Header.Height, short(cur.Header.Hash()))
		parent, ok := byHash[cur.Header.PrevHash]
		if !ok {
			fmt.Println("  (no parent — this is genesis)")
			break
		}
		cur = parent
	}

	fmt.Println("\nthe 'chain' is not a slice or a pointer — it is the hashes.")
	fmt.Println("each header names its parent by content, so the order cannot be")
	fmt.Println("rearranged without every subsequent hash changing (example 8).")
}
```

**Output:**

```
height   prev               hash               body
0        0000000000000000   3816663bc4af88f8   [genesis]
1        3816663bc4af88f8   70121f9cd5ee0103   [tx-1-a tx-1-b]
2        70121f9cd5ee0103   d857090c0403ee7b   [tx-2-a tx-2-b]
3        d857090c0403ee7b   ef9c46b25d2f30ef   [tx-3-a tx-3-b]

walking back from the tip:
  height 3  ef9c46b25d2f30ef
  height 2  d857090c0403ee7b
  height 1  70121f9cd5ee0103
  height 0  3816663bc4af88f8
  (no parent — this is genesis)

the 'chain' is not a slice or a pointer — it is the hashes.
each header names its parent by content, so the order cannot be
rearranged without every subsequent hash changing (example 8).
```

---

## 5. Header vs body: what a light client downloads

`🟢 easy` · *Header vs body*

A header is 92 bytes whatever the block contains. For 800,000 blocks that is 73 MB of headers against 600 GB of full blocks — and the Merkle root still pins the body down completely, so nothing can be swapped without detection.

**Steps:**

1. Build a 3,000-transaction block and compare header and body sizes.
2. Work out what syncing 800,000 blocks costs each way.
3. Change one transaction of 3,000 and watch the Merkle root move.
4. This split is what makes headers-first sync (lesson 13) and SPV (lesson 64) possible.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
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

func main() {
	// A realistic block body: a few thousand transactions of a few hundred
	// bytes each. The header stays 92 bytes regardless.
	const txCount = 3000
	const avgTxBytes = 250

	txs := make([]string, txCount)
	bodyBytes := 0
	for i := range txs {
		txs[i] = fmt.Sprintf("transaction-%d", i)
		bodyBytes += avgTxBytes
	}

	h := Header{Version: 1, MerkleRoot: merkleRoot(txs), Timestamp: 1231006505, Height: 1}

	fmt.Printf("block with %d transactions\n", txCount)
	fmt.Printf("  header %6d bytes\n", len(h.Bytes()))
	fmt.Printf("  body   %6d bytes (~%d KB)\n", bodyBytes, bodyBytes/1024)
	fmt.Printf("  ratio  the body is %dx the header\n", bodyBytes/len(h.Bytes()))

	// A light client syncs headers only.
	const blocks = 800_000
	fmt.Printf("\nsyncing %d blocks\n", blocks)
	fmt.Printf("  headers only  %8d MB\n", blocks*len(h.Bytes())/1_000_000)
	fmt.Printf("  full blocks   %8d MB\n", blocks*bodyBytes/1_000_000)

	fmt.Println("\nand the header still pins the body down completely:")
	fmt.Printf("  merkle root %x...\n", h.MerkleRoot[:8])

	// Swap one transaction and the root moves, so the header no longer matches.
	tampered := append([]string(nil), txs...)
	tampered[1500] = "transaction-1500-MODIFIED"
	tamperedRoot := merkleRoot(tampered)
	fmt.Printf("  after changing one of %d transactions:\n", txCount)
	fmt.Printf("  merkle root %x...\n", tamperedRoot[:8])
	fmt.Printf("  matches the header: %v\n", tamperedRoot == h.MerkleRoot)

	fmt.Println("\nso a light client can:")
	fmt.Println("  - verify the chain's structure from headers alone")
	fmt.Println("  - ask for ONE transaction plus a Merkle proof (lesson 05, example 17)")
	fmt.Println("  - detect any substitution, without ever downloading the body")
	fmt.Println("\nthis split is what makes headers-first sync possible (lesson 13)")
	fmt.Println("and SPV wallets practical (lesson 64).")
}
```

**Output:**

```
block with 3000 transactions
  header     92 bytes
  body   750000 bytes (~732 KB)
  ratio  the body is 8152x the header

syncing 800000 blocks
  headers only        73 MB
  full blocks     600000 MB

and the header still pins the body down completely:
  merkle root 58ef46694895b3ae...
  after changing one of 3000 transactions:
  merkle root 3efa165d22b263c3...
  matches the header: false

so a light client can:
  - verify the chain's structure from headers alone
  - ask for ONE transaction plus a Merkle proof (lesson 05, example 17)
  - detect any substitution, without ever downloading the body

this split is what makes headers-first sync possible (lesson 13)
and SPV wallets practical (lesson 64).
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
