# Step 03 — Bytes, Hex, Big Integers & Encoding · 🟢 Easy

Examples **1–6**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get github.com/ethereum/go-ethereum@latest   # examples 7, 8, 9
go get github.com/holiman/uint256@latest        # example 17
# paste the example into main.go, then:
go run .
```

No chain, no node, no keys — these are all pure computation.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. Hex round-trip, and the odd-length error

`🟢 easy` · *Hex*

Two hex characters encode one byte, always. `encoding/hex` is strict about that and about what counts as a hex digit — including refusing the `0x` prefix, because `x` is not one.

**Steps:**

1. Encode 4 bytes and note the string is exactly twice as long.
2. Decode it back and confirm the round trip.
3. Try an odd-length string, a bad character, and a `0x`-prefixed string.
4. Note the `0x` failure: `encoding/hex` does not know about the prefix (see example 7).

```go
package main

import (
	"encoding/hex"
	"fmt"
)

func main() {
	// Two hex characters encode one byte. Always.
	raw := []byte{0xde, 0xad, 0xbe, 0xef}

	s := hex.EncodeToString(raw)
	fmt.Printf("bytes -> hex : % x  ->  %q  (%d bytes, %d chars)\n", raw, s, len(raw), len(s))

	back, err := hex.DecodeString(s)
	fmt.Printf("hex -> bytes : %q -> % x  err=%v\n", s, back, err)

	// An odd number of characters cannot be a whole number of bytes.
	_, err = hex.DecodeString("abc")
	fmt.Printf("\nodd length   : %v\n", err)

	// Non-hex characters are rejected too, with the offending position.
	_, err = hex.DecodeString("de.dbeef")
	fmt.Printf("bad character: %v\n", err)

	// encoding/hex does NOT understand the 0x prefix — 'x' is not a hex digit.
	_, err = hex.DecodeString("0xdeadbeef")
	fmt.Printf("with 0x      : %v\n", err)
	fmt.Println("\nstrip the prefix yourself, or use go-ethereum's hexutil (example 7)")
}
```

**Output:**

```
bytes -> hex : de ad be ef  ->  "deadbeef"  (4 bytes, 8 chars)
hex -> bytes : "deadbeef" -> de ad be ef  err=<nil>

odd length   : encoding/hex: odd length hex string
bad character: encoding/hex: invalid byte: U+002E '.'
with 0x      : encoding/hex: invalid byte: U+0078 'x'

strip the prefix yourself, or use go-ethereum's hexutil (example 7)
```

---

## 2. Arrays copy, slices alias

`🟢 easy` · *Bytes*

`[32]byte` is a value: assigning it copies all 32 bytes. `[]byte` is a header pointing at a backing array: assigning it shares. This is why go-ethereum defines `common.Hash` and `common.Address` as arrays — they are comparable, usable as map keys, and impossible to alias by accident.

**Steps:**

1. Copy a `[32]byte`, mutate the copy, and confirm the original is untouched.
2. Do the same with a `[]byte` and watch both change.
3. Compare two arrays with `==` and use one as a map key — neither works with a slice.

```go
package main

import "fmt"

func main() {
	// An array is a VALUE. Assigning it copies all 32 bytes.
	var hashA [32]byte
	hashA[0] = 0xaa
	hashB := hashA // full copy
	hashB[0] = 0xbb

	fmt.Println("array ([32]byte)")
	fmt.Printf("  hashA[0]=%#x hashB[0]=%#x  -> independent\n", hashA[0], hashB[0])

	// A slice is a HEADER pointing at a backing array. Assigning shares it.
	sliceA := []byte{0xaa, 0x00, 0x00}
	sliceB := sliceA // shares the same backing array
	sliceB[0] = 0xbb

	fmt.Println("\nslice ([]byte)")
	fmt.Printf("  sliceA[0]=%#x sliceB[0]=%#x  -> ALIASED\n", sliceA[0], sliceB[0])

	// This is why go-ethereum uses arrays for the types you store and compare.
	// Arrays are comparable with == and usable as map keys; slices are neither.
	var k1, k2 [32]byte
	k1[31], k2[31] = 1, 1
	fmt.Printf("\n[32]byte == [32]byte : %v\n", k1 == k2)

	seen := map[[32]byte]bool{k1: true}
	fmt.Printf("usable as a map key  : %v\n", seen[k2])
	fmt.Println("\n// []byte == []byte does not compile, and []byte cannot be a map key")
}
```

**Output:**

```
array ([32]byte)
  hashA[0]=0xaa hashB[0]=0xbb  -> independent

slice ([]byte)
  sliceA[0]=0xbb sliceB[0]=0xbb  -> ALIASED

[32]byte == [32]byte : true
usable as a map key  : true

// []byte == []byte does not compile, and []byte cannot be a map key
```

---

## 3. Owning your bytes

`🟢 easy` · *Bytes*

A `[]byte` a library hands back is not yours. Many decoders, hashers and readers reuse one internal buffer, so the slice you stored quietly changes on the next call. Copy anything you intend to keep.

**Steps:**

1. Call a buffer-reusing API twice and watch the first result mutate.
2. Copy with `append([]byte(nil), b...)` and confirm the copy is stable.
3. Note `Sum(nil)` allocates a fresh slice while `Sum(dst)` appends into `dst`.
4. Preallocate with `make([]byte, 0, n)` when you know the size.

```go
package main

import (
	"crypto/sha256"
	"fmt"
)

// buffer simulates a library that reuses one internal buffer across calls.
// Plenty of real decoders, hashers and readers do exactly this.
type buffer struct{ buf []byte }

func (b *buffer) Next(fill byte) []byte {
	if b.buf == nil {
		b.buf = make([]byte, 4)
	}
	for i := range b.buf {
		b.buf[i] = fill
	}
	return b.buf // the caller does NOT own this
}

func main() {
	var b buffer

	// Wrong: keep the slice the library handed back.
	first := b.Next(0x11)
	second := b.Next(0x22)
	fmt.Println("keeping the library's slice")
	fmt.Printf("  first  = % x   <- silently changed by the second call\n", first)
	fmt.Printf("  second = % x\n", second)

	// Right: copy into memory you own.
	var c buffer
	firstCopy := append([]byte(nil), c.Next(0x11)...)
	c.Next(0x22)
	fmt.Println("\ncopying it first")
	fmt.Printf("  firstCopy = % x   <- yours, and stable\n", firstCopy)

	// Sum(nil) allocates a fresh slice; Sum(dst) appends into dst. Know which you called.
	h := sha256.New()
	h.Write([]byte("block"))
	digest := h.Sum(nil)
	fmt.Printf("\nsha256 digest %d bytes, first 4: % x\n", len(digest), digest[:4])

	// Preallocate when you know the size: one allocation instead of log2(n) growth steps.
	out := make([]byte, 0, 32)
	for i := 0; i < 32; i++ {
		out = append(out, byte(i))
	}
	fmt.Printf("preallocated  len=%d cap=%d (no regrowth)\n", len(out), cap(out))
}
```

**Output:**

```
keeping the library's slice
  first  = 22 22 22 22   <- silently changed by the second call
  second = 22 22 22 22

copying it first
  firstCopy = 11 11 11 11   <- yours, and stable

sha256 digest 32 bytes, first 4: 49 6a ca 80
preallocated  len=32 cap=32 (no regrowth)
```

---

## 4. Big-endian vs little-endian

`🟢 easy` · *Endianness*

Endianness is which end of a number goes first on the wire. Ethereum is big-endian; Bitcoin serializes little-endian. Reading with the wrong decoder does not error — it returns a plausible, wrong number, which is why these bugs survive review.

**Steps:**

1. Serialize one `uint64` both ways and compare the hex.
2. Read each back with its matching decoder.
3. Read the big-endian bytes with the little-endian decoder and see silent nonsense.
4. Remember the rule of thumb: if a hex string looks backwards, it probably is.

```go
package main

import (
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

func main() {
	const n uint64 = 0x0123456789abcdef

	be := make([]byte, 8)
	binary.BigEndian.PutUint64(be, n)

	le := make([]byte, 8)
	binary.LittleEndian.PutUint64(le, n)

	fmt.Printf("value          %#016x\n", n)
	fmt.Printf("big-endian     %s   <- most significant byte first\n", hex.EncodeToString(be))
	fmt.Printf("little-endian  %s   <- least significant byte first\n", hex.EncodeToString(le))

	// Read them back with the matching decoder.
	fmt.Printf("\nBigEndian.Uint64(be)    = %#x\n", binary.BigEndian.Uint64(be))
	fmt.Printf("LittleEndian.Uint64(le) = %#x\n", binary.LittleEndian.Uint64(le))

	// Read with the wrong one and you get a plausible-looking wrong number.
	// Nothing errors. This is why endianness bugs survive code review.
	fmt.Printf("\nLittleEndian.Uint64(be) = %#x   <- wrong, and silent\n",
		binary.LittleEndian.Uint64(be))

	fmt.Println("\nEthereum is big-endian on the wire; Bitcoin serializes little-endian")
	fmt.Println("rule of thumb: if a hex string looks backwards, it probably is")
}
```

**Output:**

```
value          0x0123456789abcdef
big-endian     0123456789abcdef   <- most significant byte first
little-endian  efcdab8967452301   <- least significant byte first

BigEndian.Uint64(be)    = 0x123456789abcdef
LittleEndian.Uint64(le) = 0x123456789abcdef

LittleEndian.Uint64(be) = 0xefcdab8967452301   <- wrong, and silent

Ethereum is big-endian on the wire; Bitcoin serializes little-endian
rule of thumb: if a hex string looks backwards, it probably is
```

---

## 5. Ether, wei, and why uint64 is not enough

`🟢 easy` · *Big integers*

1 ether is 10¹⁸ wei. `uint64` tops out just above 18.44 ether, so it cannot hold a balance, a total supply, or any sum. Go catches constant overflow at compile time but wraps silently at runtime — and `float64` loses the low digits entirely.

**Steps:**

1. Print 1 ether in wei and compare against `math.MaxUint64`.
2. Multiply two `uint64` variables past the limit and watch it wrap with no panic.
3. Build 1.5 ether as `15 × 10¹⁷` — never `1.5 * 1e18` in floating point.
4. Convert back to a decimal string with `QuoRem`, then look at what `float64` does to `1e18 + 1`.

```go
package main

import (
	"fmt"
	"math"
	"math/big"
)

func main() {
	// 1 ether = 10^18 wei.
	oneEther := pow10(18)
	fmt.Printf("1 ether      = %s wei\n", oneEther)

	// uint64 tops out just above 18.44 ether. Fine for one small transfer,
	// useless for a balance, a total supply or any sum.
	fmt.Printf("max uint64   = %d wei\n", uint64(math.MaxUint64))
	fmt.Printf("             = about 18.44 ether\n")

	// 19 ether does not fit. As a CONSTANT the compiler catches it:
	//
	//	var n uint64 = 19 * 1_000_000_000_000_000_000
	//	// constant 19000000000000000000 overflows uint64
	//
	// But the same arithmetic on variables wraps at runtime, silently.
	var weiPerEther uint64 = 1_000_000_000_000_000_000
	var count uint64 = 19
	fmt.Printf("\n19 ether as uint64  = %d   <- wrapped, no panic\n", count*weiPerEther)
	fmt.Printf("19 ether as big.Int = %s   <- correct\n",
		new(big.Int).Mul(big.NewInt(19), oneEther))

	// 1.5 ETH. Never compute 1.5 * 1e18 in floating point.
	// Work in the smallest unit: 15 * 10^17.
	oneFive := new(big.Int).Mul(big.NewInt(15), pow10(17))
	fmt.Printf("\n1.5 ether    = %s wei\n", oneFive)

	// And back to a decimal string, by integer division and remainder.
	whole, frac := new(big.Int).QuoRem(oneFive, oneEther, new(big.Int))
	full := fmt.Sprintf("%s.%018s", whole, frac)
	fmt.Printf("back to ether = %s\n", full)
	fmt.Printf("trimmed       = %s\n", trim(full))

	// Why floats are banned from money paths.
	fmt.Printf("\nfloat64 0.1+0.2      = %.20f\n", 0.1+0.2)
	big1 := 1e18
	fmt.Printf("float64 1e18 + 1     = %.0f   <- the +1 vanished\n", big1+1)
}

func pow10(n int64) *big.Int {
	return new(big.Int).Exp(big.NewInt(10), big.NewInt(n), nil)
}

func trim(s string) string {
	for len(s) > 0 && s[len(s)-1] == '0' {
		s = s[:len(s)-1]
	}
	if len(s) > 0 && s[len(s)-1] == '.' {
		s = s[:len(s)-1]
	}
	return s
}
```

**Output:**

```
1 ether      = 1000000000000000000 wei
max uint64   = 18446744073709551615 wei
             = about 18.44 ether

19 ether as uint64  = 553255926290448384   <- wrapped, no panic
19 ether as big.Int = 19000000000000000000   <- correct

1.5 ether    = 1500000000000000000 wei
back to ether = 1.500000000000000000
trimmed       = 1.5

float64 0.1+0.2      = 0.29999999999999998890
float64 1e18 + 1     = 1000000000000000000   <- the +1 vanished
```

---

## 6. SetString takes a base

`🟢 easy` · *Big integers*

`SetString` takes a base, and passing the wrong one gives you a valid, wrong number rather than an error. It also signals failure with an `ok` boolean, not an `error` — ignore it and you are holding a nil `*big.Int` that panics on first use.

**Steps:**

1. Parse `"1000"` as base 10 and as base 16 and compare.
2. Use base 0 to infer from the `0x`, `0b` and `0o` prefixes.
3. Feed it garbage and check the `ok` return.
4. Convert back with `Text(base)`.

```go
package main

import (
	"fmt"
	"math/big"
)

func main() {
	const s = "1000"

	// SetString takes a BASE. Get it wrong and you get a valid, wrong number.
	dec, _ := new(big.Int).SetString(s, 10)
	hex, _ := new(big.Int).SetString(s, 16)
	fmt.Printf("%q base 10 = %s\n", s, dec)
	fmt.Printf("%q base 16 = %s   <- same input, different value\n", s, hex)

	// Base 0 infers from the prefix: 0x hex, 0b binary, 0o octal, else decimal.
	for _, in := range []string{"0x1000", "1000", "0b1010", "0o777"} {
		n, ok := new(big.Int).SetString(in, 0)
		fmt.Printf("base 0: %-8q -> %-6s ok=%v\n", in, n, ok)
	}

	// SetString returns ok=false rather than an error. Ignoring it leaves you
	// holding a nil *big.Int, and the next method call panics.
	bad, ok := new(big.Int).SetString("not a number", 10)
	fmt.Printf("\ninvalid input -> value=%v ok=%v\n", bad, ok)
	if !ok {
		fmt.Println("always check ok — a nil *big.Int panics on use")
	}

	// Text() is the inverse, and also takes a base.
	n := big.NewInt(4096)
	fmt.Printf("\n%s in base 10=%s base 16=%s base 2=%s\n",
		n, n.Text(10), n.Text(16), n.Text(2))
}
```

**Output:**

```
"1000" base 10 = 1000
"1000" base 16 = 4096   <- same input, different value
base 0: "0x1000" -> 4096   ok=true
base 0: "1000"   -> 1000   ok=true
base 0: "0b1010" -> 10     ok=true
base 0: "0o777"  -> 511    ok=true

invalid input -> value=<nil> ok=false
always check ok — a nil *big.Int panics on use

4096 in base 10=4096 base 16=1000 base 2=1000000000000
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
