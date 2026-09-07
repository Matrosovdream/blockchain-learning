# Step 09 — Proof of Work & Mining · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Target back to bits, canonically

`🟡 medium` · *Difficulty*

Encoding a target back into `bits` has a subtlety: the top mantissa bit is a sign flag, so a value with it set must be shifted and the exponent bumped. The encoding is also **not unique** — and a node must require the canonical spelling.

**Steps:**

1. Round-trip five real `bits` values through target and back.
2. Watch `0x03000001` come back as `0x01010000` — both decode to 1, but only one is canonical.
3. See the sign-bit rule shift a mantissa whose high bit is set.
4. Understand why non-canonical bits must be rejected: two spellings would mean two valid block hashes.

```go
package main

import (
	"fmt"
	"math/big"
)

func BitsToTarget(bits uint32) *big.Int {
	exp := bits >> 24
	mantissa := bits & 0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(mantissa)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(mantissa)), uint(8*(exp-3)))
}

// TargetToBits produces the CANONICAL compact form. The subtlety is the
// sign bit: if the top mantissa byte is >= 0x80 the value would be read as
// negative, so the mantissa is shifted right and the exponent incremented.
func TargetToBits(target *big.Int) uint32 {
	if target.Sign() == 0 {
		return 0
	}
	b := target.Bytes()
	exp := len(b)

	var mantissa uint32
	if exp <= 3 {
		// Left-align into three bytes.
		for i := 0; i < len(b); i++ {
			mantissa = mantissa<<8 | uint32(b[i])
		}
		mantissa <<= uint(8 * (3 - exp))
	} else {
		mantissa = uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
	}

	// The sign-bit rule.
	if mantissa&0x00800000 != 0 {
		mantissa >>= 8
		exp++
	}
	return uint32(exp)<<24 | mantissa
}

func main() {
	fmt.Printf("%-14s %-68s %-14s %s\n", "bits in", "target", "canonical bits", "already canonical")
	for _, bits := range []uint32{
		0x1d00ffff, // Bitcoin's max target
		0x1c0fffff,
		0x1b0404cb, // a real historical value
		0x1903a30c,
		0x170e2632,
		0x03000001, // a NON-canonical encoding of the value 1
	} {
		t := BitsToTarget(bits)
		back := TargetToBits(t)
		fmt.Printf("%#-14x %064x %#-14x %v\n", bits, t, back, back == bits)
	}

	// That last row is the point. 0x03000001 and 0x01010000 both decode to the
	// target 1, but only one of them is the canonical encoding.
	fmt.Println("\nthe last row is not a bug — it is the canonical form at work")
	fmt.Printf("  0x03000001 decodes to %s\n", BitsToTarget(0x03000001))
	fmt.Printf("  0x01010000 decodes to %s\n", BitsToTarget(0x01010000))
	fmt.Printf("  same target: %v\n",
		BitsToTarget(0x03000001).Cmp(BitsToTarget(0x01010000)) == 0)
	fmt.Println("  a node must reject the non-canonical spelling, or the same")
	fmt.Println("  difficulty could be written two ways — and two different block")
	fmt.Println("  hashes would both be valid.")

	// The sign-bit rule in action. A mantissa with the high bit set must be
	// shifted, because the compact format reserves that bit.
	fmt.Println("\nthe sign-bit rule")
	big1 := new(big.Int).Lsh(big.NewInt(0x80), 8*28) // top byte 0x80
	fmt.Printf("  target   %064x\n", big1)
	encoded := TargetToBits(big1)
	fmt.Printf("  encoded  %#x  (exponent %d, mantissa %#06x)\n",
		encoded, encoded>>24, encoded&0x007fffff)
	fmt.Printf("  decoded  %064x\n", BitsToTarget(encoded))
	fmt.Printf("  exact round trip: %v\n", BitsToTarget(encoded).Cmp(big1) == 0)
	fmt.Println("  the mantissa was shifted right and the exponent bumped, so")
	fmt.Println("  the top mantissa bit stays clear.")

	// The encoding is also LOSSY: nearby targets collapse together.
	fmt.Println("\nthe encoding is lossy")
	t := BitsToTarget(0x1b0404cb)
	nearby := new(big.Int).Add(t, big.NewInt(1))
	fmt.Printf("  target        %064x -> %#x\n", t, TargetToBits(t))
	fmt.Printf("  target + 1    %064x -> %#x\n", nearby, TargetToBits(nearby))
	fmt.Printf("  same bits: %v\n", TargetToBits(t) == TargetToBits(nearby))
	fmt.Println("  only 3 bytes of mantissa survive, so nearby targets collapse")
	fmt.Println("  onto the same encoding. Fine in practice: the target only ever")
	fmt.Println("  comes FROM a bits value in the first place (example 9).")

	fmt.Println("\nwhy this matters for consensus")
	fmt.Println("  a block carries `bits`, and the node recomputes what bits SHOULD")
	fmt.Println("  be from the retarget rule (example 9). If a non-canonical form")
	fmt.Println("  were accepted, two encodings of the same difficulty would both")
	fmt.Println("  be valid — and the block hash would differ. So: canonical only.")
}
```

**Output:**

```
bits in        target                                                               canonical bits already canonical
0x1d00ffff     00000000ffff0000000000000000000000000000000000000000000000000000 0x1d00ffff     true
0x1c0fffff     000000000fffff00000000000000000000000000000000000000000000000000 0x1c0fffff     true
0x1b0404cb     00000000000404cb000000000000000000000000000000000000000000000000 0x1b0404cb     true
0x1903a30c     0000000000000003a30c00000000000000000000000000000000000000000000 0x1903a30c     true
0x170e2632     0000000000000000000e26320000000000000000000000000000000000000000 0x170e2632     true
0x3000001      0000000000000000000000000000000000000000000000000000000000000001 0x1010000      false

the last row is not a bug — it is the canonical form at work
  0x03000001 decodes to 1
  0x01010000 decodes to 1
  same target: true
  a node must reject the non-canonical spelling, or the same
  difficulty could be written two ways — and two different block
  hashes would both be valid.

the sign-bit rule
  target   0000008000000000000000000000000000000000000000000000000000000000
  encoded  0x1e008000  (exponent 30, mantissa 0x008000)
  decoded  0000008000000000000000000000000000000000000000000000000000000000
  exact round trip: true
  the mantissa was shifted right and the exponent bumped, so
  the top mantissa bit stays clear.

the encoding is lossy
  target        00000000000404cb000000000000000000000000000000000000000000000000 -> 0x1b0404cb
  target + 1    00000000000404cb000000000000000000000000000000000000000000000001 -> 0x1b0404cb
  same bits: true
  only 3 bytes of mantissa survive, so nearby targets collapse
  onto the same encoding. Fine in practice: the target only ever
  comes FROM a bits value in the first place (example 9).

why this matters for consensus
  a block carries `bits`, and the node recomputes what bits SHOULD
  be from the retarget rule (example 9). If a non-canonical form
  were accepted, two encodings of the same difficulty would both
  be valid — and the block hash would differ. So: canonical only.
```

---

## 7. The allocation-free mining loop

`🟡 medium` · *Performance*

A miner runs the same loop billions of times, changing four bytes. Rebuilding the header each pass allocates; patching the nonce in place does not. The measurement uses `AllocsPerRun`, which is deterministic — unlike wall-clock timing.

**Steps:**

1. Write the naive loop that rebuilds and reallocates each attempt.
2. Write the tight one: one buffer, two reused hashers, one reused `big.Int`.
3. Confirm both find the same nonce.
4. Measure allocations per attempt — 1 versus 0.
5. Note the honest caveat: this does not make a Go miner competitive; it matters for *validating* millions of blocks (lesson 57).

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/big"
	"testing"
)

const headerSize = 92

// buildHeaderBytes lays out a header once. In the mining loop we then patch
// only the 4 nonce bytes in place — never rebuild the whole thing.
func buildHeaderBytes(version uint32, prev, root [32]byte, ts int64, bits uint32, height uint64) []byte {
	b := make([]byte, headerSize)
	binary.BigEndian.PutUint32(b[0:4], version)
	copy(b[4:36], prev[:])
	copy(b[36:68], root[:])
	binary.BigEndian.PutUint64(b[68:76], uint64(ts))
	binary.BigEndian.PutUint32(b[76:80], bits)
	// b[80:84] is the nonce — left for the loop to patch
	binary.BigEndian.PutUint64(b[84:92], height)
	return b
}

const noncePos = 80

// mineNaive rebuilds and reallocates on every attempt.
func mineNaive(version uint32, prev, root [32]byte, ts int64, bits uint32, height uint64,
	target *big.Int, max uint32) (uint32, bool) {
	for n := uint32(1); n < max; n++ {
		b := buildHeaderBytes(version, prev, root, ts, bits, height) // allocation
		binary.BigEndian.PutUint32(b[noncePos:noncePos+4], n)
		first := sha256.Sum256(b)
		second := sha256.Sum256(first[:])
		if new(big.Int).SetBytes(second[:]).Cmp(target) < 0 { // allocation
			return n, true
		}
	}
	return 0, false
}

// mineTight allocates nothing inside the loop: one buffer, two reused hashers,
// one reused big.Int, and a digest written back into a slice we own.
func mineTight(version uint32, prev, root [32]byte, ts int64, bits uint32, height uint64,
	target *big.Int, max uint32) (uint32, bool) {
	buf := buildHeaderBytes(version, prev, root, ts, bits, height)

	h1 := sha256.New()
	h2 := sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var n big.Int

	for nonce := uint32(1); nonce < max; nonce++ {
		binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], nonce)

		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])

		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])

		if n.SetBytes(d2).Cmp(target) < 0 {
			return nonce, true
		}
	}
	return 0, false
}

func main() {
	var prev, root [32]byte
	target := new(big.Int).Lsh(big.NewInt(1), 256-16)

	a, okA := mineNaive(1, prev, root, 1700000000, 0x1f00ffff, 1, target, 1<<24)
	b, okB := mineTight(1, prev, root, 1700000000, 0x1f00ffff, 1, target, 1<<24)

	fmt.Printf("naive found nonce %d (ok=%v)\n", a, okA)
	fmt.Printf("tight found nonce %d (ok=%v)\n", b, okB)
	fmt.Printf("same answer: %v\n", a == b && okA == okB)

	// AllocsPerRun is deterministic; wall-clock timing is not.
	naiveAllocs := testing.AllocsPerRun(200, func() {
		buf := buildHeaderBytes(1, prev, root, 1700000000, 0x1f00ffff, 1)
		binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], 1)
		first := sha256.Sum256(buf)
		second := sha256.Sum256(first[:])
		sink = new(big.Int).SetBytes(second[:]).Cmp(target)
	})

	buf := buildHeaderBytes(1, prev, root, 1700000000, 0x1f00ffff, 1)
	h1, h2 := sha256.New(), sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var nn big.Int
	tightAllocs := testing.AllocsPerRun(200, func() {
		binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], 1)
		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])
		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])
		sink = nn.SetBytes(d2).Cmp(target)
	})

	fmt.Printf("\nallocations per attempt\n")
	fmt.Printf("  naive  %.0f\n", naiveAllocs)
	fmt.Printf("  tight  %.0f\n", tightAllocs)

	fmt.Println("\nwhy this matters at scale")
	fmt.Println("  a real miner runs 10^12 attempts per second per machine.")
	fmt.Println("  one allocation per attempt is 10^12 allocations per second —")
	fmt.Println("  the GC would be the entire program.")

	fmt.Println("\nthe pattern, from lesson 04's example 19:")
	fmt.Println("  build the buffer ONCE, patch only the bytes that change,")
	fmt.Println("  Reset() the hasher, Sum() into a slice you already own,")
	fmt.Println("  and reuse the big.Int receiver rather than allocating one.")

	fmt.Println("\nand the honest caveat: none of this makes a Go miner competitive.")
	fmt.Println("real mining is ASICs. This matters because the same pattern applies")
	fmt.Println("to validating millions of blocks during sync (lesson 57).")
}

var sink int
```

**Output:**

```
naive found nonce 63021 (ok=true)
tight found nonce 63021 (ok=true)
same answer: true

allocations per attempt
  naive  1
  tight  0

why this matters at scale
  a real miner runs 10^12 attempts per second per machine.
  one allocation per attempt is 10^12 allocations per second —
  the GC would be the entire program.

the pattern, from lesson 04's example 19:
  build the buffer ONCE, patch only the bytes that change,
  Reset() the hasher, Sum() into a slice you already own,
  and reuse the big.Int receiver rather than allocating one.

and the honest caveat: none of this makes a Go miner competitive.
real mining is ASICs. This matters because the same pattern applies
to validating millions of blocks during sync (lesson 57).
```

---

## 8. When 4 billion nonces are not enough

`🟡 medium` · *Performance*

The header's nonce is a `uint32` — 4.3 billion values. At mainnet difficulty a miner exhausts that in under a millisecond, so it needs a second source of variation: an **extra nonce** in the coinbase, which changes the Merkle root and grants a fresh 2³² nonces.

**Steps:**

1. Work out where 2³² stops being enough.
2. Limit the nonce to 8 bits so exhaustion happens in front of you.
3. Loop the extra nonce, rebuild the Merkle root, and search again.
4. Read the four sources of search space, cheapest first.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/big"
)

// A header's Nonce is uint32: 4,294,967,296 possibilities. At real difficulty
// that is not nearly enough, so miners need a second source of variation.

const headerSize = 92

func headerBytes(root [32]byte, ts int64, nonce uint32) []byte {
	b := make([]byte, headerSize)
	binary.BigEndian.PutUint32(b[0:4], 1)
	copy(b[36:68], root[:])
	binary.BigEndian.PutUint64(b[68:76], uint64(ts))
	binary.BigEndian.PutUint32(b[80:84], nonce)
	return b
}

func hash(b []byte) [32]byte {
	f := sha256.Sum256(b)
	return sha256.Sum256(f[:])
}

// merkleRootWithExtraNonce stands in for the real mechanism: the extra nonce
// lives in the COINBASE transaction, so changing it changes the coinbase txid,
// which changes the Merkle root, which gives the miner a fresh 2^32 nonces.
func merkleRootWithExtraNonce(extra uint64) [32]byte {
	var b [8]byte
	binary.BigEndian.PutUint64(b[:], extra)
	return sha256.Sum256(append([]byte("coinbase|"), b[:]...))
}

func main() {
	// The arithmetic first.
	fmt.Println("how far does a uint32 nonce go?")
	fmt.Printf("  nonce space         %d (2^32)\n", uint64(1)<<32)
	for _, zb := range []uint{24, 32, 40, 79} {
		expected := new(big.Int).Lsh(big.NewInt(1), zb)
		enough := zb < 32
		fmt.Printf("  %2d-bit difficulty   ~%-26s enough? %v\n", zb, expected, enough)
	}
	fmt.Println("\n  Bitcoin mainnet sits near 2^79 hashes per block, so a miner")
	fmt.Println("  exhausts the whole nonce space in well under a millisecond.")

	// Now watch it happen, at a difficulty we can actually reach. The nonce
	// here is deliberately limited to 8 bits so exhaustion is quick.
	const nonceLimit = 1 << 8
	target := new(big.Int).Lsh(big.NewInt(1), 256-14)

	fmt.Printf("\nsearching with only %d nonces available, 14-bit target\n", nonceLimit)

	var val big.Int
	var extra uint64
	var totalHashes uint64
	var foundNonce uint32
	var found bool

	for ; extra < 1000; extra++ {
		root := merkleRootWithExtraNonce(extra)
		for nonce := uint32(0); nonce < nonceLimit; nonce++ {
			totalHashes++
			sum := hash(headerBytes(root, 1700000000, nonce))
			if val.SetBytes(sum[:]).Cmp(target) < 0 {
				foundNonce, found = nonce, true
				break
			}
		}
		if found {
			break
		}
	}

	fmt.Printf("  exhausted the nonce space %d times\n", extra)
	fmt.Printf("  found at extra-nonce %d, nonce %d\n", extra, foundNonce)
	fmt.Printf("  total hashes %d\n", totalHashes)

	root := merkleRootWithExtraNonce(extra)
	sum := hash(headerBytes(root, 1700000000, foundNonce))
	fmt.Printf("  hash %x\n", sum[:12])
	fmt.Printf("  valid: %v\n", new(big.Int).SetBytes(sum[:]).Cmp(target) < 0)

	// The three sources of variation, in the order miners use them.
	fmt.Println("\nwhere a miner finds more search space, cheapest first")
	fmt.Println("  1. Nonce         4.3 billion, free — just increment")
	fmt.Println("  2. extra nonce   in the coinbase; changes the Merkle root,")
	fmt.Println("                   so the whole Merkle path must be recomputed")
	fmt.Println("  3. Timestamp     can be rolled forward a little, within the")
	fmt.Println("                   bounds from lesson 08 (MTP and the 2h limit)")
	fmt.Println("  4. the transaction set itself — different txs, different root")

	fmt.Println("\nthis is why the coinbase has an arbitrary data field at all,")
	fmt.Println("and it is the same field that carried Satoshi's newspaper headline")
	fmt.Println("(lesson 08) and now carries miner tags and extra nonces.")
}
```

**Output:**

```
how far does a uint32 nonce go?
  nonce space         4294967296 (2^32)
  24-bit difficulty   ~16777216                   enough? true
  32-bit difficulty   ~4294967296                 enough? false
  40-bit difficulty   ~1099511627776              enough? false
  79-bit difficulty   ~604462909807314587353088   enough? false

  Bitcoin mainnet sits near 2^79 hashes per block, so a miner
  exhausts the whole nonce space in well under a millisecond.

searching with only 256 nonces available, 14-bit target
  exhausted the nonce space 59 times
  found at extra-nonce 59, nonce 161
  total hashes 15266
  hash 000242b413bb1611438ca047
  valid: true

where a miner finds more search space, cheapest first
  1. Nonce         4.3 billion, free — just increment
  2. extra nonce   in the coinbase; changes the Merkle root,
                   so the whole Merkle path must be recomputed
  3. Timestamp     can be rolled forward a little, within the
                   bounds from lesson 08 (MTP and the 2h limit)
  4. the transaction set itself — different txs, different root

this is why the coinbase has an arbitrary data field at all,
and it is the same field that carried Satoshi's newspaper headline
(lesson 08) and now carries miner tags and extra nonces.
```

---

## 9. Retargeting, integer-only

`🟡 medium` · *Retargeting*

Difficulty is adjusted to keep block times near target: `newTarget = oldTarget × actual / expected`, clamped to 4x either way. **Every operation is integer** — a float here could round differently on two machines and split the chain with no attacker involved.

**Steps:**

1. Implement the retarget with `big.Int`, clamping before the multiply.
2. Run six periods from 3.5 to 56 days and read the difficulty change.
3. Watch the clamp cap the adjustment at 4x and floor it at 1/4.
4. Note the operation order: multiply first, divide second, or you truncate away the precision.

```go
package main

import (
	"fmt"
	"math/big"
)

// Difficulty retargeting: keep the average block time near a target by
// adjusting how hard the puzzle is. Bitcoin does this every 2016 blocks.
//
// EVERY ARITHMETIC OPERATION HERE IS INTEGER. A float in consensus code means
// two platforms can compute two different targets and the chain splits.

const (
	targetSpacing  = 600                            // seconds per block
	retargetBlocks = 2016                           // blocks per period
	targetTimespan = targetSpacing * retargetBlocks // 1209600 = two weeks
)

func BitsToTarget(bits uint32) *big.Int {
	exp := bits >> 24
	mantissa := bits & 0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(mantissa)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(mantissa)), uint(8*(exp-3)))
}

func TargetToBits(target *big.Int) uint32 {
	if target.Sign() == 0 {
		return 0
	}
	b := target.Bytes()
	exp := len(b)
	var mantissa uint32
	if exp <= 3 {
		for i := 0; i < len(b); i++ {
			mantissa = mantissa<<8 | uint32(b[i])
		}
		mantissa <<= uint(8 * (3 - exp))
	} else {
		mantissa = uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
	}
	if mantissa&0x00800000 != 0 {
		mantissa >>= 8
		exp++
	}
	return uint32(exp)<<24 | mantissa
}

var maxTarget = BitsToTarget(0x1d00ffff)

// Retarget computes the next bits from the previous bits and how long the
// last period actually took.
//
//	newTarget = oldTarget * actualTimespan / targetTimespan
//
// Blocks came too fast  -> actual < target -> newTarget smaller -> harder.
// Blocks came too slow  -> actual > target -> newTarget larger  -> easier.
func Retarget(oldBits uint32, actualTimespan int64) uint32 {
	// Clamp BEFORE the multiply, so a single period can never move difficulty
	// by more than 4x in either direction.
	clamped := actualTimespan
	if clamped < targetTimespan/4 {
		clamped = targetTimespan / 4
	}
	if clamped > targetTimespan*4 {
		clamped = targetTimespan * 4
	}

	next := new(big.Int).Mul(BitsToTarget(oldBits), big.NewInt(clamped))
	next.Div(next, big.NewInt(targetTimespan))

	// Never easier than the maximum allowed target.
	if next.Cmp(maxTarget) > 0 {
		next.Set(maxTarget)
	}
	return TargetToBits(next)
}

func difficulty(bits uint32) *big.Int {
	return new(big.Int).Div(maxTarget, BitsToTarget(bits))
}

func main() {
	fmt.Printf("target spacing  %d seconds\n", targetSpacing)
	fmt.Printf("retarget every  %d blocks\n", retargetBlocks)
	fmt.Printf("target timespan %d seconds (%d days)\n\n", targetTimespan, targetTimespan/86400)

	start := uint32(0x1b0404cb)
	fmt.Printf("%-24s %-14s %-14s %-18s %s\n",
		"period took", "avg block", "old bits", "new bits", "difficulty change")

	for _, days := range []float64{14, 7, 21, 3.5, 56, 14} {
		actual := int64(days * 86400)
		next := Retarget(start, actual)
		before := difficulty(start)
		after := difficulty(next)

		// Integer ratio, expressed as a percentage — still no floats in the
		// consensus path, only in this display line.
		pct := new(big.Int).Mul(after, big.NewInt(100))
		pct.Div(pct, before)

		fmt.Printf("%-24s %-14s %#-14x %#-18x %s%%\n",
			fmt.Sprintf("%.1f days", days),
			fmt.Sprintf("%d s", actual/retargetBlocks),
			start, next, pct)
	}

	fmt.Println("\nreading the table")
	fmt.Println("  took 14 days  -> exactly on target, difficulty unchanged (100%)")
	fmt.Println("  took  7 days  -> blocks twice as fast, difficulty doubles")
	fmt.Println("  took 21 days  -> blocks too slow, difficulty drops to ~67%")
	fmt.Println("  took 3.5 days -> would be 4x, and the clamp caps it at 4x")
	fmt.Println("  took 56 days  -> would be 1/4, and the clamp floors it there (24% after\n                   the compact encoding rounds — example 6)")

	fmt.Println("\nwhy integer-only")
	fmt.Println("  a float division can round differently on different hardware,")
	fmt.Println("  compilers or optimisation levels. Two nodes computing two")
	fmt.Println("  different targets is a chain split with no attacker involved.")
	fmt.Println("  big.Int Mul-then-Div is exact and identical everywhere.")

	fmt.Println("\nnote the operation order: multiply FIRST, then divide.")
	fmt.Println("  dividing first would truncate away most of the precision")
	fmt.Println("  (lesson 03) — the same rule as any fixed-point arithmetic.")
}
```

**Output:**

```
target spacing  600 seconds
retarget every  2016 blocks
target timespan 1209600 seconds (14 days)

period took              avg block      old bits       new bits           difficulty change
14.0 days                600 s          0x1b0404cb     0x1b0404cb         100%
7.0 days                 300 s          0x1b0404cb     0x1b020265         200%
21.0 days                900 s          0x1b0404cb     0x1b060730         66%
3.5 days                 150 s          0x1b0404cb     0x1b010132         400%
56.0 days                2400 s         0x1b0404cb     0x1b10132c         24%
14.0 days                600 s          0x1b0404cb     0x1b0404cb         100%

reading the table
  took 14 days  -> exactly on target, difficulty unchanged (100%)
  took  7 days  -> blocks twice as fast, difficulty doubles
  took 21 days  -> blocks too slow, difficulty drops to ~67%
  took 3.5 days -> would be 4x, and the clamp caps it at 4x
  took 56 days  -> would be 1/4, and the clamp floors it there (24% after
                   the compact encoding rounds — example 6)

why integer-only
  a float division can round differently on different hardware,
  compilers or optimisation levels. Two nodes computing two
  different targets is a chain split with no attacker involved.
  big.Int Mul-then-Div is exact and identical everywhere.

note the operation order: multiply FIRST, then divide.
  dividing first would truncate away most of the precision
  (lesson 03) — the same rule as any fixed-point arithmetic.
```

---

## 10. The timewarp attack

`🟡 medium` · *Attacks*

The retarget reads timestamps from block headers, and miners choose those. Bitcoin measures the span using only the first and last block of a period — so an attacker who backdates everything except the last block makes the chain believe blocks are slow, and difficulty collapses.

**Steps:**

1. Establish the direction: a longer reported span means an easier target.
2. Run six periods of an inflated span and watch difficulty fall 4x each time.
3. Read the four defences, of which median-time-past (lesson 08) is the main one.
4. Note Bitcoin's known 2016-vs-2015 off-by-one, kept for compatibility.

```go
package main

import (
	"fmt"
	"math/big"
)

const (
	targetSpacing  = 600
	retargetBlocks = 2016
	targetTimespan = targetSpacing * retargetBlocks
)

func BitsToTarget(bits uint32) *big.Int {
	exp := bits >> 24
	m := bits & 0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

var maxTarget = BitsToTarget(0x1d00ffff)

func difficulty(t *big.Int) *big.Int { return new(big.Int).Div(maxTarget, t) }

// retargetTarget works in targets rather than bits, to keep the arithmetic
// visible. Clamped to 4x either way, exactly as in example 9.
func retargetTarget(old *big.Int, actualTimespan int64) *big.Int {
	c := actualTimespan
	if c < targetTimespan/4 {
		c = targetTimespan / 4
	}
	if c > targetTimespan*4 {
		c = targetTimespan * 4
	}
	next := new(big.Int).Mul(old, big.NewInt(c))
	next.Div(next, big.NewInt(targetTimespan))
	if next.Cmp(maxTarget) > 0 {
		next.Set(maxTarget)
	}
	return next
}

func main() {
	// The retarget uses the TIMESTAMPS in block headers, and miners choose
	// those. Lesson 08 bounded them — but Bitcoin's original rule measures the
	// span using only the FIRST and LAST block of each period, which leaves a
	// gap an attacker with majority hashrate can drive through.

	fmt.Println("the retarget direction, first")
	fmt.Println("  reported span SHORTER than 14 days -> 'blocks are fast' -> HARDER")
	fmt.Println("  reported span LONGER  than 14 days -> 'blocks are slow' -> EASIER")

	fmt.Println("\nhonest mining: the span really is two weeks")
	target := BitsToTarget(0x1b0404cb)
	fmt.Printf("  starting difficulty %s\n", difficulty(target))
	for period := 0; period < 3; period++ {
		target = retargetTarget(target, targetTimespan)
		fmt.Printf("  period %d  span %8ds  difficulty %s\n", period, targetTimespan, difficulty(target))
	}
	fmt.Println("  on target, so difficulty does not move.")

	// THE TIMEWARP. The attacker mines blocks quickly but stamps almost all of
	// them far in the past, making only the LAST block of each period carry a
	// current timestamp. The measured first-to-last span is then enormous.
	fmt.Println("\nthe timewarp: backdate every block except each period's last")
	fmt.Println("  real elapsed time per period: a few hours")
	fmt.Println("  reported first-to-last span : weeks")

	target = BitsToTarget(0x1b0404cb)
	fmt.Printf("\n  starting difficulty %s\n", difficulty(target))
	for period := 0; period < 6; period++ {
		reported := int64(targetTimespan * 8) // clamped to 4x
		target = retargetTarget(target, reported)
		fmt.Printf("  period %d  reported span %8ds  difficulty %s\n",
			period, reported, difficulty(target))
	}

	fmt.Println("\n  the clamp limits each period to a 4x drop, so six periods give")
	fmt.Println("  4^6 = 4096x easier — while the attacker mined them in hours.")
	fmt.Println("  from there they can produce blocks almost for free and rewrite")
	fmt.Println("  history at will.")

	// The defences.
	fmt.Println("\nwhat stops it")
	fmt.Println("  1. median-time-past (lesson 08): a block's timestamp must exceed")
	fmt.Println("     the median of the last 11, so timestamps cannot simply run")
	fmt.Println("     backwards — this is the main brake")
	fmt.Println("  2. the 2-hour future limit bounds the other direction")
	fmt.Println("  3. the 4x clamp bounds the damage per period")
	fmt.Println("  4. it needs majority hashrate anyway, at which point there are")
	fmt.Println("     simpler attacks available (example 15)")

	fmt.Println("\nBitcoin measures the span across 2016 blocks but only 2015")
	fmt.Println("intervals — a known off-by-one kept for compatibility. Testnet has")
	fmt.Println("seen real timewarp exploitation; mainnet has not, and later")
	fmt.Println("proposals tighten the rule.")
	fmt.Println("\nthe lesson: timestamp rules are consensus-critical, not bookkeeping.")
}
```

**Output:**

```
the retarget direction, first
  reported span SHORTER than 14 days -> 'blocks are fast' -> HARDER
  reported span LONGER  than 14 days -> 'blocks are slow' -> EASIER

honest mining: the span really is two weeks
  starting difficulty 16307
  period 0  span  1209600s  difficulty 16307
  period 1  span  1209600s  difficulty 16307
  period 2  span  1209600s  difficulty 16307
  on target, so difficulty does not move.

the timewarp: backdate every block except each period's last
  real elapsed time per period: a few hours
  reported first-to-last span : weeks

  starting difficulty 16307
  period 0  reported span  9676800s  difficulty 4076
  period 1  reported span  9676800s  difficulty 1019
  period 2  reported span  9676800s  difficulty 254
  period 3  reported span  9676800s  difficulty 63
  period 4  reported span  9676800s  difficulty 15
  period 5  reported span  9676800s  difficulty 3

  the clamp limits each period to a 4x drop, so six periods give
  4^6 = 4096x easier — while the attacker mined them in hours.
  from there they can produce blocks almost for free and rewrite
  history at will.

what stops it
  1. median-time-past (lesson 08): a block's timestamp must exceed
     the median of the last 11, so timestamps cannot simply run
     backwards — this is the main brake
  2. the 2-hour future limit bounds the other direction
  3. the 4x clamp bounds the damage per period
  4. it needs majority hashrate anyway, at which point there are
     simpler attacks available (example 15)

Bitcoin measures the span across 2016 blocks but only 2015
intervals — a known off-by-one kept for compatibility. Testnet has
seen real timewarp exploitation; mainnet has not, and later
proposals tighten the rule.

the lesson: timestamp rules are consensus-critical, not bookkeeping.
```

---

## 11. Block times are exponential

`🟡 medium` · *Statistics*

Block discovery is a Poisson process, so the gap between blocks is **exponentially distributed**. The consequences surprise nearly everyone: the median is well under the mean, most blocks arrive faster than target, and hour-long gaps are normal.

**Steps:**

1. Simulate 100,000 inter-block times with a seeded RNG.
2. Compare the mean, the median and the maximum.
3. Print the distribution as a histogram.
4. Verify memorylessness: of blocks already past 10 minutes, the same fraction again exceed 20.
5. Conclude why 'about 10 minutes' is a terrible basis for a timeout.

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sort"
)

// Block discovery is a Poisson process: every hash is an independent trial
// with a tiny success probability, so the WAIT between blocks is exponentially
// distributed. The consequences surprise almost everyone.

func main() {
	const target = 600.0 // seconds
	const n = 100_000

	rng := rand.New(rand.NewSource(42)) // seeded: this output reproduces

	times := make([]float64, n)
	for i := range times {
		// Inverse-transform sampling of an exponential with mean `target`.
		times[i] = -target * math.Log(1-rng.Float64())
	}

	var sum float64
	for _, t := range times {
		sum += t
	}
	mean := sum / n
	sort.Float64s(times)

	fmt.Printf("%d simulated blocks, target %.0fs\n\n", n, target)
	fmt.Printf("  mean    %6.1fs   (the target, as designed)\n", mean)
	fmt.Printf("  median  %6.1fs   (~0.69x the mean — NOT the same thing)\n", times[n/2])
	fmt.Printf("  min     %6.1fs\n", times[0])
	fmt.Printf("  max     %6.1fs   (%.1f minutes)\n", times[n-1], times[n-1]/60)

	// The distribution, as a histogram.
	fmt.Println("\ndistribution of inter-block times")
	buckets := []struct {
		lo, hi float64
		label  string
	}{
		{0, 60, "under 1 min"},
		{60, 300, "1-5 min"},
		{300, 600, "5-10 min"},
		{600, 1200, "10-20 min"},
		{1200, 1800, "20-30 min"},
		{1800, 3600, "30-60 min"},
		{3600, math.Inf(1), "over 1 hour"},
	}
	for _, b := range buckets {
		count := 0
		for _, t := range times {
			if t >= b.lo && t < b.hi {
				count++
			}
		}
		pct := float64(count) * 100 / n
		bar := ""
		for i := 0; i < int(pct); i++ {
			bar += "#"
		}
		fmt.Printf("  %-12s %5.1f%%  %s\n", b.label, pct, bar)
	}

	// The single most counter-intuitive fact.
	fmt.Println("\nthe surprising part")
	under := 0
	for _, t := range times {
		if t < target {
			under++
		}
	}
	fmt.Printf("  %.0f%% of blocks arrive FASTER than the 10-minute target\n",
		float64(under)*100/n)
	fmt.Println("  because the distribution has a long tail: many quick blocks")
	fmt.Println("  balanced by a few very slow ones. The MEAN is 10 minutes; the")
	fmt.Println("  MEDIAN is about 7.")

	// Memorylessness, stated concretely.
	fmt.Println("\nand it is memoryless")
	waited := 0
	thenAnother := 0
	for _, t := range times {
		if t > 600 {
			waited++
			if t > 1200 {
				thenAnother++
			}
		}
	}
	fmt.Printf("  of blocks that took over 10 minutes, %.0f%% took over 20\n",
		float64(thenAnother)*100/float64(waited))
	fmt.Printf("  which is the same ~%.0f%% as any block taking over 10 minutes.\n",
		float64(waited)*100/n)
	fmt.Println("  waiting does not make the next block more likely. There is no")
	fmt.Println("  'due' block — exactly as in example 1's memoryless search.")

	fmt.Println("\nwhy this matters in practice")
	fmt.Println("  - a 40-minute gap is normal, not evidence of an attack")
	fmt.Println("  - 'about 10 minutes' is a terrible basis for a timeout")
	fmt.Println("  - you cannot conclude anything about hashrate from one block")
	fmt.Println("  - and it is why safety is measured in CONFIRMATIONS, not")
	fmt.Println("    minutes (example 12)")
}
```

**Output:**

```
100000 simulated blocks, target 600s

  mean     600.2s   (the target, as designed)
  median   414.0s   (~0.69x the mean — NOT the same thing)
  min        0.0s
  max     6985.4s   (116.4 minutes)

distribution of inter-block times
  under 1 min    9.5%  #########
  1-5 min       30.0%  ##############################
  5-10 min      23.5%  #######################
  10-20 min     23.4%  #######################
  20-30 min      8.8%  ########
  30-60 min      4.6%  ####
  over 1 hour    0.2%  

the surprising part
  63% of blocks arrive FASTER than the 10-minute target
  because the distribution has a long tail: many quick blocks
  balanced by a few very slow ones. The MEAN is 10 minutes; the
  MEDIAN is about 7.

and it is memoryless
  of blocks that took over 10 minutes, 37% took over 20
  which is the same ~37% as any block taking over 10 minutes.
  waiting does not make the next block more likely. There is no
  'due' block — exactly as in example 1's memoryless search.

why this matters in practice
  - a 40-minute gap is normal, not evidence of an attack
  - 'about 10 minutes' is a terrible basis for a timeout
  - you cannot conclude anything about hashrate from one block
  - and it is why safety is measured in CONFIRMATIONS, not
    minutes (example 12)
```

---

## 12. Confirmations, not minutes

`🟡 medium` · *Statistics*

The Bitcoin whitepaper's section 11 calculation: the probability an attacker with q of the hashrate ever catches up from z blocks behind. It falls exponentially in z — and stops meaning anything at all once q reaches 50%.

**Steps:**

1. Implement the whitepaper's Poisson sum directly.
2. Tabulate q from 10% to 45% against z from 0 to 50.
3. Check the q=10%, z=6 cell against the published 0.0002428.
4. See that at q≥50% more confirmations buy delay, not safety.
5. Read why the choice of confirmation count is economic, not cryptographic.

```go
package main

import (
	"fmt"
	"math"
)

// Bitcoin whitepaper section 11. An attacker with q of the hashrate tries to
// catch up from z blocks behind. The probability they ever succeed falls
// exponentially in z — which is why safety is counted in confirmations.

// catchUpProbability is the whitepaper's calculation.
func catchUpProbability(q float64, z int) float64 {
	p := 1 - q
	if q >= p {
		return 1.0 // a majority attacker catches up with certainty, eventually
	}
	lambda := float64(z) * (q / p)

	sum := 1.0
	for k := 0; k <= z; k++ {
		// Poisson probability the attacker made k blocks while we made z.
		poisson := math.Exp(-lambda)
		for i := 1; i <= k; i++ {
			poisson *= lambda / float64(i)
		}
		sum -= poisson * (1 - math.Pow(q/p, float64(z-k)))
	}
	return sum
}

func main() {
	fmt.Println("probability an attacker with q hashrate ever catches up from z behind")
	fmt.Println("(Bitcoin whitepaper, section 11)")
	fmt.Println()

	qs := []float64{0.10, 0.20, 0.30, 0.40, 0.45}
	fmt.Printf("%-14s", "confirmations")
	for _, q := range qs {
		fmt.Printf("%12s", fmt.Sprintf("q=%.0f%%", q*100))
	}
	fmt.Println()

	for _, z := range []int{0, 1, 2, 3, 6, 10, 20, 50} {
		fmt.Printf("%-14d", z)
		for _, q := range qs {
			p := catchUpProbability(q, z)
			switch {
			case p > 0.001:
				fmt.Printf("%12s", fmt.Sprintf("%.4f", p))
			default:
				fmt.Printf("%12s", fmt.Sprintf("%.2e", p))
			}
		}
		fmt.Println()
	}

	fmt.Println("\nreading it")
	fmt.Println("  z=0  the transaction is unconfirmed: the attacker wins outright")
	fmt.Println("  z=6  the traditional Bitcoin standard — safe against a 10% attacker")
	fmt.Println("       to about 1 in 4000, and still fragile against 40%")
	fmt.Println("  the probability falls exponentially in z, and rises sharply as q")
	fmt.Println("  approaches 50%")

	// The point about 51%.
	fmt.Println("\nat q >= 50% the table stops meaning anything")
	fmt.Printf("  q=50%%  probability = %.1f at any depth\n", catchUpProbability(0.5, 100))
	fmt.Println("  a majority attacker catches up with certainty given enough time.")
	fmt.Println("  more confirmations buy delay, not safety (example 15).")

	// And the practical framing.
	fmt.Println("\nhow to actually choose a confirmation count")
	fmt.Println("  it is an ECONOMIC decision, not a cryptographic one:")
	fmt.Println("    what does an attack cost at this chain's hashrate?")
	fmt.Println("    what is this transaction worth?")
	fmt.Println("    how long will the counterparty tolerate waiting?")
	fmt.Println("  exchanges use different depths per chain and per amount —")
	fmt.Println("  a small chain may need hundreds where Bitcoin needs six.")

	fmt.Println("\nand note what confirmations are NOT")
	fmt.Println("  not a clock: six blocks might take 20 minutes or two hours")
	fmt.Println("  (example 11). Count blocks, never wall-clock time.")
	fmt.Println("\n  proof of stake replaces this probabilistic picture with explicit")
	fmt.Println("  economic finality (lesson 28) — a different guarantee entirely.")
}
```

**Output:**

```
probability an attacker with q hashrate ever catches up from z behind
(Bitcoin whitepaper, section 11)

confirmations        q=10%       q=20%       q=30%       q=40%       q=45%
0                   1.0000      1.0000      1.0000      1.0000      1.0000
1                   0.2046      0.4159      0.6277      0.8289      0.9198
2                   0.0510      0.2039      0.4457      0.7364      0.8777
3                   0.0132      0.1032      0.3246      0.6642      0.8444
6                 2.43e-04      0.0143      0.1321      0.5040      0.7661
10                1.24e-06      0.0011      0.0417      0.3600      0.6854
20                2.46e-12    1.74e-06      0.0025      0.1636      0.5366
50                7.32e-17    8.37e-15    5.90e-07      0.0172      0.2800

reading it
  z=0  the transaction is unconfirmed: the attacker wins outright
  z=6  the traditional Bitcoin standard — safe against a 10% attacker
       to about 1 in 4000, and still fragile against 40%
  the probability falls exponentially in z, and rises sharply as q
  approaches 50%

at q >= 50% the table stops meaning anything
  q=50%  probability = 1.0 at any depth
  a majority attacker catches up with certainty given enough time.
  more confirmations buy delay, not safety (example 15).

how to actually choose a confirmation count
  it is an ECONOMIC decision, not a cryptographic one:
    what does an attack cost at this chain's hashrate?
    what is this transaction worth?
    how long will the counterparty tolerate waiting?
  exchanges use different depths per chain and per amount —
  a small chain may need hundreds where Bitcoin needs six.

and note what confirmations are NOT
  not a clock: six blocks might take 20 minutes or two hours
  (example 11). Count blocks, never wall-clock time.

  proof of stake replaces this probabilistic picture with explicit
  economic finality (lesson 28) — a different guarantee entirely.
```

---

## 13. Hashrate, difficulty and the security budget

`🟡 medium` · *Economics*

Difficulty tells you hashes per block; hashes per block plus block time tells you network hashrate; and hashrate is bought with block rewards. So a chain's safety is proportional to **what it pays miners** — which is why small chains sharing an algorithm with big ones are cheap to attack.

**Steps:**

1. Convert difficulty to expected hashes per block and to hashrate.
2. See where the 2³² factor comes from.
3. Print Bitcoin's halving schedule and note that fees must eventually carry the whole budget.
4. Read the cost-of-attack framing, and the honest list of what PoW does and does not provide.

```go
package main

import (
	"fmt"
	"math/big"
)

func BitsToTarget(bits uint32) *big.Int {
	exp := bits >> 24
	m := bits & 0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

var maxTarget = BitsToTarget(0x1d00ffff)

func main() {
	// Difficulty tells you how many hashes a block costs on average:
	//
	//   expected hashes per block = 2^256 / (target + 1) ≈ difficulty * 2^32
	//
	// and from that plus the block time you get the network's hashrate.

	fmt.Printf("%-16s %-22s %-24s %s\n", "difficulty", "hashes per block", "hashrate at 600s", "")
	for _, d := range []int64{1, 16307, 1e9, 1e12, 1e14} {
		diff := big.NewInt(d)
		// hashes per block ≈ difficulty * 2^32
		perBlock := new(big.Int).Lsh(diff, 32)
		// hashrate = hashes per block / seconds per block
		rate := new(big.Int).Div(perBlock, big.NewInt(600))
		fmt.Printf("%-16d %-22s %-24s\n", d, sci(perBlock), sci(rate)+" H/s")
	}

	fmt.Println("\nthat 2^32 factor is where the compact target's max value comes")
	fmt.Println("from: difficulty 1 means a target of 0x00000000ffff..., which")
	fmt.Println("about 1 in 2^32 hashes beats.")

	// The security budget: what the network PAYS for that hashrate.
	fmt.Println("\nthe security budget")
	fmt.Println("  miners spend right up to what they earn, so:")
	fmt.Println("    hashrate  ~=  (block reward + fees) / cost per hash")
	fmt.Println("  which means security is bought with issuance, and issuance")
	fmt.Println("  halves every four years.")

	fmt.Println("\n  Bitcoin's subsidy schedule")
	subsidy := 50.0
	for era := 0; era < 8; era++ {
		year := 2009 + era*4
		fmt.Printf("    %d  %8.4f BTC/block\n", year, subsidy)
		subsidy /= 2
	}
	fmt.Println("    ...  eventually 0, and fees alone must pay for security")

	// Cost of attack.
	fmt.Println("\ncost of attack, the number that actually matters")
	fmt.Println("  to rewrite z blocks you must out-mine the network for that long.")
	fmt.Println("  roughly: attack cost ~= z * (block reward + fees) * (q / (1-q))")
	fmt.Println("  so a chain's safety is proportional to what it PAYS miners,")
	fmt.Println("  not to how clever its cryptography is.")

	fmt.Println("\n  which is why small proof-of-work chains are cheap to attack:")
	fmt.Println("    a chain paying $1,000/hour in rewards can be out-mined for")
	fmt.Println("    a few thousand dollars an hour on a hashrate rental market,")
	fmt.Println("    even though it uses the same SHA-256 as Bitcoin.")
	fmt.Println("    Ethereum Classic and Bitcoin Gold were both attacked this way")
	fmt.Println("    while sharing an algorithm with a far larger chain.")

	// And the honest summary of what PoW provides.
	fmt.Println("\nwhat proof of work actually provides")
	fmt.Println("  + Sybil resistance: influence costs money, so identities are not free")
	fmt.Println("  + an objective ordering anyone can verify without trusting anyone")
	fmt.Println("  - it says NOTHING about transaction validity — every node still")
	fmt.Println("    checks every rule (lesson 08). A miner with 100% hashrate still")
	fmt.Println("    cannot spend your coins or print money.")
	fmt.Println("  - and it is not 'security' in the abstract: it is a specific,")
	fmt.Println("    priced resistance to history being rewritten.")
}

// sci renders a big.Int in approximate scientific notation.
func sci(n *big.Int) string {
	s := n.String()
	if len(s) <= 4 {
		return s
	}
	return fmt.Sprintf("%s.%se%d", s[:1], s[1:3], len(s)-1)
}
```

**Output:**

```
difficulty       hashes per block       hashrate at 600s         
1                4.29e9                 7.15e6 H/s              
16307            7.00e13                1.16e11 H/s             
1000000000       4.29e18                7.15e15 H/s             
1000000000000    4.29e21                7.15e18 H/s             
100000000000000  4.29e23                7.15e20 H/s             

that 2^32 factor is where the compact target's max value comes
from: difficulty 1 means a target of 0x00000000ffff..., which
about 1 in 2^32 hashes beats.

the security budget
  miners spend right up to what they earn, so:
    hashrate  ~=  (block reward + fees) / cost per hash
  which means security is bought with issuance, and issuance
  halves every four years.

  Bitcoin's subsidy schedule
    2009   50.0000 BTC/block
    2013   25.0000 BTC/block
    2017   12.5000 BTC/block
    2021    6.2500 BTC/block
    2025    3.1250 BTC/block
    2029    1.5625 BTC/block
    2033    0.7812 BTC/block
    2037    0.3906 BTC/block
    ...  eventually 0, and fees alone must pay for security

cost of attack, the number that actually matters
  to rewrite z blocks you must out-mine the network for that long.
  roughly: attack cost ~= z * (block reward + fees) * (q / (1-q))
  so a chain's safety is proportional to what it PAYS miners,
  not to how clever its cryptography is.

  which is why small proof-of-work chains are cheap to attack:
    a chain paying $1,000/hour in rewards can be out-mined for
    a few thousand dollars an hour on a hashrate rental market,
    even though it uses the same SHA-256 as Bitcoin.
    Ethereum Classic and Bitcoin Gold were both attacked this way
    while sharing an algorithm with a far larger chain.

what proof of work actually provides
  + Sybil resistance: influence costs money, so identities are not free
  + an objective ordering anyone can verify without trusting anyone
  - it says NOTHING about transaction validity — every node still
    checks every rule (lesson 08). A miner with 100% hashrate still
    cannot spend your coins or print money.
  - and it is not 'security' in the abstract: it is a specific,
    priced resistance to history being rewritten.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
