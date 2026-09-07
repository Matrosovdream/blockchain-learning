# Step 09 — Proof of Work & Mining · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch       # first time only
# paste the example into main.go, then:
go run .
```

**Standard library only.** Simulations use a seeded RNG and mining reports *hash counts* rather than
elapsed time, so every output reproduces exactly.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. The puzzle

`🟢 easy` · *The puzzle*

Find a nonce such that the header's hash, read as a 256-bit number, is below a target. There is no cleverer method than trying — the search is **memoryless**, so no attempt tells you anything about the next. That is what makes hashrate translate directly into a share of blocks.

**Steps:**

1. Set a target with 16 leading zero bits: about 1 in 65,536 headers will beat it.
2. Grind the nonce until the hash compares below it.
3. Note the asymmetry — tens of thousands of hashes to find, one to check.
4. Read why memorylessness is the property that makes the lottery fair.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"math/big"
)

// The header from lesson 08, unchanged.
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

func main() {
	// The puzzle: find a Nonce such that Hash(header) < target.
	// A target with 16 leading zero BITS: 1 in 2^16 headers will satisfy it.
	target := new(big.Int).Lsh(big.NewInt(1), 256-16)

	fmt.Printf("target  %064x\n", target)
	fmt.Println("we need a header whose hash, read as a 256-bit number, is SMALLER.")

	h := Header{Version: 1, Timestamp: 1700000000, Height: 1}

	// Grinding: change the nonce, rehash, compare. There is no cleverer way —
	// the search is memoryless, so no attempt teaches you anything about the next.
	var attempts uint64
	for {
		h.Nonce++
		attempts++
		sum := h.Hash()
		if new(big.Int).SetBytes(sum[:]).Cmp(target) < 0 {
			break
		}
	}

	sum := h.Hash()
	fmt.Printf("\nfound after %d attempts\n", attempts)
	fmt.Printf("nonce   %d\n", h.Nonce)
	fmt.Printf("hash    %s\n", hex.EncodeToString(sum[:]))
	fmt.Printf("        ^^^^ the leading zeros are the visible sign of the work\n")

	// The asymmetry that makes it useful.
	fmt.Println("\nthe asymmetry")
	fmt.Printf("  finding it   %d hashes\n", attempts)
	fmt.Printf("  checking it  1 hash\n")
	fmt.Printf("  ratio        %dx\n", attempts)
	fmt.Println("\nthat is what lets every node police every miner cheaply.")

	// And no progress is stored between attempts.
	fmt.Println("\nthe search is MEMORYLESS: each nonce is an independent coin flip.")
	fmt.Println("a miner who has tried a billion nonces is no closer than one who")
	fmt.Println("just started. That is what makes the lottery fair, and it is why")
	fmt.Println("hashrate translates directly into a share of blocks.")
}
```

**Output:**

```
target  0001000000000000000000000000000000000000000000000000000000000000
we need a header whose hash, read as a 256-bit number, is SMALLER.

found after 57516 attempts
nonce   57516
hash    000018b9e794a1e64c28e2397dd1de66ae0aa5f6074b55dc02a41d2065cadb29
        ^^^^ the leading zeros are the visible sign of the work

the asymmetry
  finding it   57516 hashes
  checking it  1 hash
  ratio        57516x

that is what lets every node police every miner cheaply.

the search is MEMORYLESS: each nonce is an independent coin flip.
a miner who has tried a billion nonces is no closer than one who
just started. That is what makes the lottery fair, and it is why
hashrate translates directly into a share of blocks.
```

---

## 2. Verification is one hash

`🟢 easy` · *The puzzle*

Verification is a single hash and a single comparison. Every field is inside that hash, so the work is bound to one exact header — a miner cannot reuse it, and cannot change anything afterwards.

**Steps:**

1. Verify a mined header with `CheckPoW`.
2. Try changing the nonce, timestamp, height and Merkle root in turn.
3. Watch every one invalidate the proof.
4. Contrast with a signature: one says *who* authorised, the other says *what it cost*.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"math/big"
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

// CheckPoW is the ENTIRE verification. One hash, one comparison.
func CheckPoW(h Header, target *big.Int) bool {
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(target) < 0
}

func main() {
	target := new(big.Int).Lsh(big.NewInt(1), 256-16)

	// A miner did the work (example 1) and published this header.
	mined := Header{Version: 1, Timestamp: 1700000000, Height: 1, Nonce: 57516}
	sum := mined.Hash()

	fmt.Println("a block arrives from the network")
	fmt.Printf("  nonce %d\n", mined.Nonce)
	fmt.Printf("  hash  %s\n", hex.EncodeToString(sum[:]))
	fmt.Printf("\nverifying: %v\n", CheckPoW(mined, target))
	fmt.Println("  one hash. That is all a node has to do.")

	// The miner cannot lie about the work: change anything and the hash moves.
	fmt.Println("\ncan the miner cheat?")
	for _, tc := range []struct {
		name string
		f    func(Header) Header
	}{
		{"claim a different nonce", func(x Header) Header { x.Nonce = 1; return x }},
		{"change the timestamp", func(x Header) Header { x.Timestamp++; return x }},
		{"change the height", func(x Header) Header { x.Height = 999; return x }},
		{"swap the merkle root", func(x Header) Header { x.MerkleRoot[0] = 1; return x }},
	} {
		fmt.Printf("  %-26s -> valid PoW: %v\n", tc.name, CheckPoW(tc.f(mined), target))
	}

	fmt.Println("\nevery field is inside the hash, so the work is bound to this")
	fmt.Println("EXACT header. A miner cannot reuse it for a different block.")

	// And the check is objective: no opinion, no trust, no committee.
	fmt.Println("\nwhat makes this different from a signature (lesson 06):")
	fmt.Println("  a signature says WHO authorised something")
	fmt.Println("  proof of work says HOW MUCH IT COST to produce this")
	fmt.Println("  the second needs no identity, no registry and no trust —")
	fmt.Println("  which is exactly why it works in an open network.")
}
```

**Output:**

```
a block arrives from the network
  nonce 57516
  hash  000018b9e794a1e64c28e2397dd1de66ae0aa5f6074b55dc02a41d2065cadb29

verifying: true
  one hash. That is all a node has to do.

can the miner cheat?
  claim a different nonce    -> valid PoW: false
  change the timestamp       -> valid PoW: false
  change the height          -> valid PoW: false
  swap the merkle root       -> valid PoW: false

every field is inside the hash, so the work is bound to this
EXACT header. A miner cannot reuse it for a different block.

what makes this different from a signature (lesson 06):
  a signature says WHO authorised something
  proof of work says HOW MUCH IT COST to produce this
  the second needs no identity, no registry and no trust —
  which is exactly why it works in an open network.
```

---

## 3. Compact bits to a 256-bit target

`🟢 easy` · *Difficulty*

The target is a 256-bit number, but the header has only four bytes for it. Bitcoin packs it as a base-256 float: one exponent byte and three mantissa bytes. Difficulty is then just a ratio against the easiest allowed target.

**Steps:**

1. Decode `0x1d00ffff` into its exponent and mantissa.
2. Expand it to the full 256-bit target with `big.Int`.
3. Tabulate several real historical `bits` values and their leading zero counts.
4. Compute difficulty as `max_target / target`.

```go
package main

import (
	"fmt"
	"math/big"
)

// Bitcoin packs a 256-bit target into 4 bytes: one exponent byte and three
// mantissa bytes. It is a base-256 floating-point number.
//
//	target = mantissa * 256^(exponent-3)
func BitsToTarget(bits uint32) *big.Int {
	exponent := bits >> 24
	mantissa := bits & 0x007fffff // the top mantissa bit is a sign flag

	if exponent <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(mantissa)), uint(8*(3-exponent)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(mantissa)), uint(8*(exponent-3)))
}

func main() {
	// The genesis difficulty, and the easiest target Bitcoin allows.
	const genesisBits = 0x1d00ffff

	target := BitsToTarget(genesisBits)
	fmt.Printf("bits     %#08x\n", uint32(genesisBits))
	fmt.Printf("  exponent %#02x = %d\n", genesisBits>>24, genesisBits>>24)
	fmt.Printf("  mantissa %#06x = %d\n", genesisBits&0x007fffff, genesisBits&0x007fffff)
	fmt.Printf("\ntarget = %d * 256^(%d-3)\n", genesisBits&0x007fffff, genesisBits>>24)
	fmt.Printf("       = %064x\n", target)

	// The leading zeros in the target are what a valid hash must beat.
	fmt.Printf("\nthat target has %d leading zero bits\n", 256-target.BitLen())

	// A few more, to see the encoding vary.
	fmt.Printf("\n%-12s %-8s %-10s %s\n", "bits", "exponent", "mantissa", "leading zero bits")
	for _, b := range []uint32{0x1d00ffff, 0x1c0fffff, 0x1b0404cb, 0x1903a30c, 0x170e2632} {
		t := BitsToTarget(b)
		fmt.Printf("%#-12x %-8d %#-10x %d\n", b, b>>24, b&0x007fffff, 256-t.BitLen())
	}

	// Difficulty is a RATIO against the easiest allowed target.
	maxTarget := BitsToTarget(0x1d00ffff)
	fmt.Println("\ndifficulty = max_target / target")
	for _, b := range []uint32{0x1d00ffff, 0x1c0fffff, 0x1b0404cb, 0x170e2632} {
		d := new(big.Int).Div(maxTarget, BitsToTarget(b))
		fmt.Printf("  bits %#x -> difficulty %s\n", b, d)
	}

	// Why a compact form at all: it keeps the header small and fixed-size.
	fmt.Println("\nwhy pack it into 4 bytes?")
	fmt.Println("  the header is fixed-size (lesson 08). A full 32-byte target")
	fmt.Println("  would add 28 bytes to every header ever made, to express a")
	fmt.Println("  number that only ever needs about 3 significant digits.")

	// The gotcha.
	fmt.Println("\nthe gotcha: the encoding is lossy and not unique. Several bits")
	fmt.Println("values can describe nearly the same target, so Bitcoin requires")
	fmt.Println("the canonical form — and rejects a block whose bits are not it.")
}
```

**Output:**

```
bits     0x1d00ffff
  exponent 0x1d = 29
  mantissa 0x00ffff = 65535

target = 65535 * 256^(29-3)
       = 00000000ffff0000000000000000000000000000000000000000000000000000

that target has 32 leading zero bits

bits         exponent mantissa   leading zero bits
0x1d00ffff   29       0xffff     32
0x1c0fffff   28       0xfffff    36
0x1b0404cb   27       0x404cb    45
0x1903a30c   25       0x3a30c    62
0x170e2632   23       0xe2632    76

difficulty = max_target / target
  bits 0x1d00ffff -> difficulty 1
  bits 0x1c0fffff -> difficulty 15
  bits 0x1b0404cb -> difficulty 16307
  bits 0x170e2632 -> difficulty 19893045048575

why pack it into 4 bytes?
  the header is fixed-size (lesson 08). A full 32-byte target
  would add 28 bytes to every header ever made, to express a
  number that only ever needs about 3 significant digits.

the gotcha: the encoding is lossy and not unique. Several bits
values can describe nearly the same target, so Bitcoin requires
the canonical form — and rejects a block whose bits are not it.
```

---

## 4. Each bit doubles the work

`🟢 easy` · *Difficulty*

Each extra bit of difficulty doubles the expected work. A *single* mining run tells you almost nothing though — the counts scatter enormously — so this averages 30 independent runs at each level to show the law emerging.

**Steps:**

1. Mine 30 different headers at each of five difficulties.
2. Compare the mean against 2ⁿ and watch the ratio sit near 1.
3. Note each row adds two bits, so the ratio between rows is about 4x.
4. Extrapolate to Bitcoin's ~2⁷⁹ hashes per block.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/big"
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

// mine returns the winning nonce and the number of hashes it took.
// Attempts, not elapsed time: the count is deterministic, the clock is not.
func mine(h Header, zeroBits uint) (uint32, uint64) {
	target := new(big.Int).Lsh(big.NewInt(1), 256-zeroBits)
	var n big.Int
	for nonce := uint32(1); ; nonce++ {
		h.Nonce = nonce
		sum := h.Hash()
		if n.SetBytes(sum[:]).Cmp(target) < 0 {
			return nonce, uint64(nonce)
		}
	}
}

func main() {
	// A SINGLE mining run tells you almost nothing: the counts scatter
	// enormously. Averaging many independent runs is what shows the law.
	const trials = 30

	fmt.Printf("mean hashes over %d runs at each difficulty\n\n", trials)
	fmt.Printf("%-8s %-14s %-14s %-10s %s\n",
		"bits", "expected 2^n", "mean actual", "mean/2^n", "ratio to previous")
	var prevMean float64
	for _, zb := range []uint{8, 10, 12, 14, 16} {
		var total uint64
		for t := 0; t < trials; t++ {
			// A different header each trial: an independent experiment.
			h := Header{Version: 1, Timestamp: 1700000000 + int64(t), Height: 1}
			_, attempts := mine(h, zb)
			total += attempts
		}
		mean := float64(total) / trials
		expected := float64(uint64(1) << zb)
		ratio := "-"
		if prevMean > 0 {
			ratio = fmt.Sprintf("%.2fx", mean/prevMean)
		}
		fmt.Printf("%-8d %-14.0f %-14.0f %-10.2f %s\n", zb, expected, mean, mean/expected, ratio)
		prevMean = mean
	}

	fmt.Println("\neach extra bit of difficulty DOUBLES the expected work, and the")
	fmt.Println("means land near 2^n. Note the 'ratio to previous' column is close")
	fmt.Println("to 4x because each row adds TWO bits.")
	fmt.Println("\na single run scatters wildly — that is example 12's point, and it")
	fmt.Println("is why you can never conclude anything from one block's timing.")

	// What that means at real difficulty.
	fmt.Println("\nextrapolating to Bitcoin")
	fmt.Println("  mainnet difficulty needs roughly 2^79 hashes per block")
	fmt.Println("  at 1 billion hashes/second that is about 19 million years")
	fmt.Println("  the whole network does it every 10 minutes, which tells you")
	fmt.Println("  how much hardware is pointed at it")

	// And the honest framing.
	fmt.Println("\nnote what 'difficulty' is NOT: it is not a measure of security")
	fmt.Println("in the abstract. It is a knob the protocol turns to keep block")
	fmt.Println("times near target as hashrate changes (example 10).")
}
```

**Output:**

```
mean hashes over 30 runs at each difficulty

bits     expected 2^n   mean actual    mean/2^n   ratio to previous
8        256            174            0.68       -
10       1024           876            0.86       5.03x
12       4096           4137           1.01       4.73x
14       16384          14004          0.85       3.38x
16       65536          58065          0.89       4.15x

each extra bit of difficulty DOUBLES the expected work, and the
means land near 2^n. Note the 'ratio to previous' column is close
to 4x because each row adds TWO bits.

a single run scatters wildly — that is example 12's point, and it
is why you can never conclude anything from one block's timing.

extrapolating to Bitcoin
  mainnet difficulty needs roughly 2^79 hashes per block
  at 1 billion hashes/second that is about 19 million years
  the whole network does it every 10 minutes, which tells you
  how much hardware is pointed at it

note what 'difficulty' is NOT: it is not a measure of security
in the abstract. It is a knob the protocol turns to keep block
times near target as hashrate changes (example 10).
```

---

## 5. Leading zeros is a picture, not the rule

`🟢 easy` · *Difficulty*

"Find a hash with N leading zeros" is how proof of work is always explained, and it is not the rule. The rule is a 256-bit integer comparison, and the two differ whenever the target is not an exact power of two — which in practice it never is.

**Steps:**

1. Take two hashes with the *same* leading-zero count on opposite sides of the target.
2. See that counting zeros cannot tell them apart, and the comparison can.
3. Understand why a hex-prefix test could only ever express 16x difficulty jumps.
4. Take away the one line that is always correct: `SetBytes(hash).Cmp(target) < 0`.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"math/big"
	"strings"
)

// leadingZeroBits counts the zero bits before the first 1.
func leadingZeroBits(b [32]byte) int {
	return 256 - new(big.Int).SetBytes(b[:]).BitLen()
}

func main() {
	// "Find a hash with N leading zeros" is the usual explanation of proof of
	// work. It is a good picture and it is not the rule. The rule is a 256-bit
	// integer comparison, and the two differ whenever the target is not an
	// exact power of two — which, in practice, it never is.

	// Bitcoin's easiest target. Note it is 0x00000000ffff..., not 0x0000000100...
	maxTarget, _ := new(big.Int).SetString(
		"00000000ffff0000000000000000000000000000000000000000000000000000", 16)
	fmt.Printf("Bitcoin's max target\n  %064x\n", maxTarget)
	fmt.Printf("  leading zero bits: %d\n", 256-maxTarget.BitLen())

	// Two hashes with the SAME number of leading zero bits, on opposite sides
	// of that target.
	below, _ := new(big.Int).SetString(
		"00000000aaaa0000000000000000000000000000000000000000000000000000", 16)
	above, _ := new(big.Int).SetString(
		"00000000ffff0000000000000000000000000000000000000000000000000001", 16)

	fmt.Printf("\ntwo candidate hashes, both with %d leading zero bits:\n", 256-below.BitLen())
	fmt.Printf("  A %064x\n", below)
	fmt.Printf("  B %064x\n", above)
	fmt.Printf("\n  A < target: %v\n", below.Cmp(maxTarget) < 0)
	fmt.Printf("  B < target: %v   <- same zero count, still invalid\n", above.Cmp(maxTarget) < 0)
	fmt.Println("\n  counting zeros cannot tell these apart. The comparison can.")

	// A prefix test cannot express a target at all.
	fmt.Println("\nwhy `strings.HasPrefix(hex, \"0000\")` is not a substitute")
	fmt.Println("  a prefix test can only express targets that are exact powers")
	fmt.Println("  of 16 — so difficulty could only ever move in 4-bit jumps,")
	fmt.Println("  meaning 16x at a time. Retargeting (example 10) needs to make")
	fmt.Println("  adjustments of a few percent.")

	// Show real hashes and both verdicts, at a difficulty we can actually reach.
	easy := new(big.Int).Lsh(big.NewInt(1), 256-12) // 12 zero bits
	fmt.Printf("\nreal hashes against a 12-zero-bit target:\n")
	fmt.Printf("%-8s %-20s %-8s %s\n", "input", "hash (first 10 bytes)", "zeros", "< target")
	found := 0
	for i := 0; found < 5; i++ {
		sum := sha256.Sum256([]byte(fmt.Sprintf("input-%d", i)))
		z := leadingZeroBits(sum)
		if z >= 10 {
			fmt.Printf("%-8d %-20s %-8d %v\n",
				i, hex.EncodeToString(sum[:10]), z, new(big.Int).SetBytes(sum[:]).Cmp(easy) < 0)
			found++
		}
	}

	// And the practical rule.
	fmt.Println("\nthe rule, always:")
	fmt.Println("  new(big.Int).SetBytes(hash[:]).Cmp(target) < 0")
	fmt.Println("\nnever a hex prefix, never a zero count, never a string compare.")
	fmt.Printf("  (%q would be a prefix test — it works for toy demos and\n",
		strings.Repeat("0", 4))
	fmt.Println("   silently expresses the wrong rule the moment difficulty is real.)")
}
```

**Output:**

```
Bitcoin's max target
  00000000ffff0000000000000000000000000000000000000000000000000000
  leading zero bits: 32

two candidate hashes, both with 32 leading zero bits:
  A 00000000aaaa0000000000000000000000000000000000000000000000000000
  B 00000000ffff0000000000000000000000000000000000000000000000000001

  A < target: true
  B < target: false   <- same zero count, still invalid

  counting zeros cannot tell these apart. The comparison can.

why `strings.HasPrefix(hex, "0000")` is not a substitute
  a prefix test can only express targets that are exact powers
  of 16 — so difficulty could only ever move in 4-bit jumps,
  meaning 16x at a time. Retargeting (example 10) needs to make
  adjustments of a few percent.

real hashes against a 12-zero-bit target:
input    hash (first 10 bytes) zeros    < target
1562     00118b0d690b15c899bb 11       false
4317     0026ed07957e62094b48 10       false
4830     0003da3b7dd21934adec 14       true
5316     0001f60dbd6f29be8814 15       true
8908     00247703960055cd536f 10       false

the rule, always:
  new(big.Int).SetBytes(hash[:]).Cmp(target) < 0

never a hex prefix, never a zero count, never a string compare.
  ("0000" would be a prefix test — it works for toy demos and
   silently expresses the wrong rule the moment difficulty is real.)
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
