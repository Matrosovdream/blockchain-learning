# Step 03 — Bytes, Hex, Big Integers & Encoding · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. A Wei type the compiler enforces

`🔴 hard` · *Units*

The underlying representation of wei and gwei is identical, so nothing but a type stops you passing one where the other is expected — an error of 10⁹. Named types move that check to compile time, and the constructors also stop internal `*big.Int` state escaping (example 11).

**Steps:**

1. Define `Wei` and `Gwei` as distinct types wrapping a `*big.Int`.
2. Make conversion an explicit method, never implicit.
3. Return new values from arithmetic so no caller can mutate internal state.
4. Note the commented-out line: passing `Gwei` to a function wanting `Wei` will not compile.

```go
package main

import (
	"fmt"
	"math/big"
)

// Named types stop the compiler from letting you mix units.
// The underlying representation is identical; the type is the point.
type (
	Wei  struct{ v *big.Int }
	Gwei struct{ v *big.Int }
)

func NewWei(i int64) Wei   { return Wei{big.NewInt(i)} }
func NewGwei(i int64) Gwei { return Gwei{big.NewInt(i)} }

// Conversions are explicit, one-way functions — never implicit.
func (g Gwei) ToWei() Wei { return Wei{new(big.Int).Mul(g.v, big.NewInt(1e9))} }

// Add returns a NEW value. Nothing here hands out a pointer to internal state.
func (w Wei) Add(o Wei) Wei { return Wei{new(big.Int).Add(w.v, o.v)} }
func (w Wei) Cmp(o Wei) int { return w.v.Cmp(o.v) }

func (w Wei) String() string {
	unit := new(big.Int).Exp(big.NewInt(10), big.NewInt(18), nil)
	whole, frac := new(big.Int).QuoRem(w.v, unit, new(big.Int))
	return fmt.Sprintf("%s.%018s ETH", whole, frac)
}

func (g Gwei) String() string { return g.v.String() + " gwei" }

// A gas calculation that cannot be given the wrong units.
func TxFee(gasUsed uint64, price Wei) Wei {
	return Wei{new(big.Int).Mul(new(big.Int).SetUint64(gasUsed), price.v)}
}

func main() {
	baseFee := NewGwei(30)
	tip := NewGwei(2)

	// Gwei + Gwei needs an explicit trip through Wei — the compiler insists.
	perGas := baseFee.ToWei().Add(tip.ToWei())
	fmt.Printf("base fee   %s\n", baseFee)
	fmt.Printf("tip        %s\n", tip)
	fmt.Printf("per gas    %s\n", perGas)

	fee := TxFee(21000, perGas)
	fmt.Printf("\n21000 gas  %s\n", fee)

	budget := NewWei(1).Add(NewWei(0)) // 1 wei, deliberately tiny
	fmt.Printf("budget     %s\n", budget)
	fmt.Printf("affordable %v\n", fee.Cmp(budget) <= 0)

	// The bug this design prevents:
	//
	//	TxFee(21000, baseFee)   // compile error: Gwei is not Wei
	//
	// With raw *big.Int everywhere, that line compiles and you underpay by 10^9.
	fmt.Println("\nTxFee(21000, baseFee) does not compile — Gwei is not Wei")
	fmt.Println("with raw *big.Int it would compile, and be wrong by a factor of 10^9")
}
```

**Output:**

```
base fee   30 gwei
tip        2 gwei
per gas    0.000000032000000000 ETH

21000 gas  0.000672000000000000 ETH
budget     0.000000000000000001 ETH
affordable false

TxFee(21000, baseFee) does not compile — Gwei is not Wei
with raw *big.Int it would compile, and be wrong by a factor of 10^9
```

---

## 15. base58 from scratch

`🔴 hard` · *Encodings*

base58 is base64 with the confusable characters removed — no `0`, `O`, `I` or `l` — because addresses get read aloud and retyped. Encoding is repeated division by 58 with `big.Int`, plus one wrinkle: leading zero bytes carry no numeric value and must be handled separately.

**Steps:**

1. Divide by 58 with `QuoRem`, collecting digits least-significant first, then reverse.
2. Handle leading zero bytes as literal `'1'` characters.
3. Decode by multiplying back up, and reject characters outside the alphabet.
4. Confirm leading zeros survive a round trip — a Bitcoin address depends on it.

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"math/big"
)

// The base58 alphabet deliberately omits 0 O I l — the characters humans
// confuse when reading an address off a screen or a piece of paper.
const alphabet = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"

var radix = big.NewInt(58)

func Encode(input []byte) string {
	// Leading zero bytes carry no value, so treat them separately: each one
	// becomes a literal '1' (the digit for zero in this alphabet).
	zeros := 0
	for zeros < len(input) && input[zeros] == 0 {
		zeros++
	}

	num := new(big.Int).SetBytes(input)
	mod := new(big.Int)
	var out []byte
	for num.Sign() > 0 {
		num.QuoRem(num, radix, mod)
		out = append(out, alphabet[mod.Int64()])
	}
	for i := 0; i < zeros; i++ {
		out = append(out, alphabet[0])
	}
	// Digits came out least-significant first.
	for i, j := 0, len(out)-1; i < j; i, j = i+1, j-1 {
		out[i], out[j] = out[j], out[i]
	}
	return string(out)
}

var ErrBadChar = errors.New("character not in base58 alphabet")

func Decode(s string) ([]byte, error) {
	num := new(big.Int)
	for i := 0; i < len(s); i++ {
		idx := bytes.IndexByte([]byte(alphabet), s[i])
		if idx < 0 {
			return nil, fmt.Errorf("%w: %q at %d", ErrBadChar, s[i], i)
		}
		num.Mul(num, radix)
		num.Add(num, big.NewInt(int64(idx)))
	}
	zeros := 0
	for zeros < len(s) && s[zeros] == alphabet[0] {
		zeros++
	}
	return append(make([]byte, zeros), num.Bytes()...), nil
}

func main() {
	fmt.Printf("alphabet (%d chars): %s\n", len(alphabet), alphabet)
	fmt.Println("missing on purpose: 0 (zero) O (oh) I (eye) l (ell)")

	cases := [][]byte{
		[]byte("hello"),
		{0x00, 0x01, 0x02},
		{0x00, 0x00, 0xff},
	}
	fmt.Printf("\n%-14s %-14s %s\n", "bytes", "base58", "round-trips")
	for _, in := range cases {
		enc := Encode(in)
		back, err := Decode(enc)
		fmt.Printf("%-14x %-14s %v\n", in, enc, err == nil && bytes.Equal(back, in))
	}

	// Leading zeros must survive, or a Bitcoin address loses its version byte.
	in := []byte{0x00, 0x00, 0x00, 0x41}
	enc := Encode(in)
	back, _ := Decode(enc)
	fmt.Printf("\nleading zeros: % x -> %q -> % x (preserved: %v)\n",
		in, enc, back, bytes.Equal(in, back))

	_, err := Decode("hello0world")
	fmt.Printf("\nrejects bad input: %v\n", err)
}
```

**Output:**

```
alphabet (58 chars): 123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
missing on purpose: 0 (zero) O (oh) I (eye) l (ell)

bytes          base58         round-trips
68656c6c6f     Cn8eVZg        true
000102         15T            true
0000ff         115Q           true

leading zeros: 00 00 00 41 -> "11128" -> 00 00 00 41 (preserved: true)

rejects bad input: character not in base58 alphabet: 'l' at 2
```

---

## 16. base58check, and catching a typo

`🔴 hard` · *Encodings*

base58check wraps a payload with a version byte and four checksum bytes taken from a double SHA-256. That is what makes a mistyped Bitcoin address fail immediately instead of sending funds into the void. The program decodes the real genesis address and then corrupts it on purpose.

**Steps:**

1. Decode `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa` into its version byte and 20-byte hash160.
2. Re-encode and confirm you get the identical string back.
3. Corrupt one character at three positions and confirm the checksum catches each.
4. Four checksum bytes means roughly a 1-in-2³² chance of an undetected typo.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"errors"
	"fmt"
	"math/big"
)

const alphabet = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"

var radix = big.NewInt(58)

func b58encode(input []byte) string {
	zeros := 0
	for zeros < len(input) && input[zeros] == 0 {
		zeros++
	}
	num, mod := new(big.Int).SetBytes(input), new(big.Int)
	var out []byte
	for num.Sign() > 0 {
		num.QuoRem(num, radix, mod)
		out = append(out, alphabet[mod.Int64()])
	}
	for i := 0; i < zeros; i++ {
		out = append(out, alphabet[0])
	}
	for i, j := 0, len(out)-1; i < j; i, j = i+1, j-1 {
		out[i], out[j] = out[j], out[i]
	}
	return string(out)
}

func b58decode(s string) ([]byte, error) {
	num := new(big.Int)
	for i := 0; i < len(s); i++ {
		idx := bytes.IndexByte([]byte(alphabet), s[i])
		if idx < 0 {
			return nil, fmt.Errorf("bad character %q at %d", s[i], i)
		}
		num.Mul(num, radix)
		num.Add(num, big.NewInt(int64(idx)))
	}
	zeros := 0
	for zeros < len(s) && s[zeros] == alphabet[0] {
		zeros++
	}
	return append(make([]byte, zeros), num.Bytes()...), nil
}

// checksum is the first 4 bytes of double SHA-256 — Bitcoin's typo detector.
func checksum(b []byte) []byte {
	first := sha256.Sum256(b)
	second := sha256.Sum256(first[:])
	return second[:4]
}

// CheckEncode: version byte ‖ payload ‖ checksum(version‖payload), then base58.
func CheckEncode(version byte, payload []byte) string {
	body := append([]byte{version}, payload...)
	return b58encode(append(body, checksum(body)...))
}

var ErrBadChecksum = errors.New("checksum mismatch — address is corrupted")

func CheckDecode(s string) (version byte, payload []byte, err error) {
	raw, err := b58decode(s)
	if err != nil {
		return 0, nil, err
	}
	if len(raw) < 5 {
		return 0, nil, errors.New("too short")
	}
	body, want := raw[:len(raw)-4], raw[len(raw)-4:]
	if got := checksum(body); !bytes.Equal(got, want) {
		return 0, nil, fmt.Errorf("%w: have %x, want %x", ErrBadChecksum, want, got)
	}
	return body[0], body[1:], nil
}

func main() {
	// Bitcoin's genesis coinbase address — the most public address in existence.
	const genesis = "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"

	version, hash160, err := CheckDecode(genesis)
	if err != nil {
		fmt.Println("decode failed:", err)
		return
	}
	fmt.Printf("address  %s\n", genesis)
	fmt.Printf("version  0x%02x  (0x00 = mainnet P2PKH, hence the leading '1')\n", version)
	fmt.Printf("hash160  %x  (%d bytes)\n", hash160, len(hash160))

	// Re-encode and confirm we get the identical string back.
	again := CheckEncode(version, hash160)
	fmt.Printf("re-encoded matches: %v\n", again == genesis)

	// Now corrupt one character and watch the checksum catch it.
	fmt.Println("\ncorrupting one character at a time:")
	for _, i := range []int{1, 10, len(genesis) - 1} {
		bad := []byte(genesis)
		if bad[i] == 'a' {
			bad[i] = 'b'
		} else {
			bad[i] = 'a'
		}
		_, _, err := CheckDecode(string(bad))
		fmt.Printf("  pos %-2d %q -> caught=%v\n", i, string(bad), errors.Is(err, ErrBadChecksum))
	}

	// 4 checksum bytes = 1 in 2^32 chance a random typo slips through.
	fmt.Printf("\n4 checksum bytes -> ~1 in %d chance of an undetected typo\n", uint64(1)<<32)
	fmt.Println("this is the same idea as EIP-55 for Ethereum addresses (lesson 07)")
}
```

**Output:**

```
address  1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
version  0x00  (0x00 = mainnet P2PKH, hence the leading '1')
hash160  62e907b15cbf27d5425399ebf6f0fb50ebb88f18  (20 bytes)
re-encoded matches: true

corrupting one character at a time:
  pos 1  "1a1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa" -> caught=true
  pos 10 "1A1zP1eP5Qaefi2DMPTfTL5SLmv7DivfNa" -> caught=true
  pos 33 "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNb" -> caught=true

4 checksum bytes -> ~1 in 4294967296 chance of an undetected typo
this is the same idea as EIP-55 for Ethereum addresses (lesson 07)
```

---

## 17. big.Int vs uint256

`🔴 hard` · *Big integers*

`uint256.Int` is four `uint64`s of plain data; `big.Int` is a struct wrapping a heap slice. The allocation numbers below are measured with `testing.AllocsPerRun`, which is deterministic. They also behave differently at the boundary: uint256 wraps like the EVM, big.Int just grows.

**Steps:**

1. Measure allocations for fresh and reused receivers of both types.
2. Measure a local value of each — uint256 stays on the stack, big.Int's slice does not.
3. Add 1 to 2²⁵⁶−1 in both and compare: uint256 wraps to 0 and flags it, big.Int becomes 257 bits.
4. Convert an oversized `big.Int` down with `FromBig` and check the overflow flag.

```go
package main

import (
	"fmt"
	"math/big"
	"testing"

	"github.com/holiman/uint256"
)

// Sinks keep the results alive so the optimiser cannot delete the work.
var (
	sinkB *big.Int
	sinkU *uint256.Int
)

func main() {
	// Same value, two representations.
	b, _ := new(big.Int).SetString("1000000000000000000", 10)
	u, _ := uint256.FromDecimal("1000000000000000000")
	fmt.Printf("big.Int  %s  (struct + heap slice, arbitrary precision)\n", b)
	fmt.Printf("uint256  %s  (exactly 4 x uint64, always 256 bits)\n", u.Dec())

	// --- allocations --------------------------------------------------------
	// AllocsPerRun is deterministic, unlike wall-clock timing. The sinks stop
	// the compiler optimising the work away.
	x, y := big.NewInt(12345), big.NewInt(67890)
	ux, uy := uint256.NewInt(12345), uint256.NewInt(67890)
	zb, zu := new(big.Int), new(uint256.Int)

	fmt.Printf("\nallocations per Mul\n")
	fmt.Printf("  big.Int, fresh receiver : %.0f   (the Int, plus its backing []Word)\n",
		testing.AllocsPerRun(1000, func() { sinkB = new(big.Int).Mul(x, y) }))
	fmt.Printf("  uint256, fresh receiver : %.0f   (just the struct — no backing slice)\n",
		testing.AllocsPerRun(1000, func() { sinkU = new(uint256.Int).Mul(ux, uy) }))
	fmt.Printf("  big.Int, reused receiver: %.0f   (capacity already there)\n",
		testing.AllocsPerRun(1000, func() { zb.Mul(x, y); sinkB = zb }))
	fmt.Printf("  uint256, reused receiver: %.0f\n",
		testing.AllocsPerRun(1000, func() { zu.Mul(ux, uy); sinkU = zu }))

	// The real difference: a uint256.Int is 4 words of plain data, so a local
	// value never leaves the stack. A big.Int always carries a slice.
	fmt.Printf("\nas a local value (no pointer escaping)\n")
	fmt.Printf("  var t big.Int; t.Mul(x,y)      : %.0f   (the []Word still escapes)\n",
		testing.AllocsPerRun(1000, func() {
			var t big.Int
			t.Mul(x, y)
			_ = t
		}))
	fmt.Printf("  var t uint256.Int; t.Mul(x,y)  : %.0f   (entirely on the stack)\n",
		testing.AllocsPerRun(1000, func() {
			var t uint256.Int
			t.Mul(ux, uy)
			_ = t
		}))

	// --- overflow behaviour -------------------------------------------------
	// The EVM wraps at 2^256. big.Int grows instead, which is usually what YOU
	// want in Go — but it means big.Int does not model the EVM faithfully.
	maxU := new(uint256.Int).Not(uint256.NewInt(0))
	one := uint256.NewInt(1)

	wrapped := new(uint256.Int).Add(maxU, one)
	_, overflow := new(uint256.Int).AddOverflow(maxU, one)
	fmt.Printf("\n2^256-1 + 1\n")
	fmt.Printf("  uint256          %s   (overflow flag: %v)\n", wrapped.Dec(), overflow)

	maxB := new(big.Int).Sub(new(big.Int).Lsh(big.NewInt(1), 256), big.NewInt(1))
	grown := new(big.Int).Add(maxB, big.NewInt(1))
	fmt.Printf("  big.Int          %s\n", grown)
	fmt.Printf("  big.Int bit len  %d   <- silently became a 257-bit number\n", grown.BitLen())

	// Converting back down truncates. Check, do not assume.
	shrunk, ok := uint256.FromBig(grown)
	fmt.Printf("\nuint256.FromBig(2^256) -> %s, overflowed=%v\n", shrunk.Dec(), ok)

	fmt.Println("\nuse uint256 in interpreters and hot loops (lesson 18, 57)")
	fmt.Println("use big.Int in APIs and anywhere readability wins")
}
```

**Output:**

```
big.Int  1000000000000000000  (struct + heap slice, arbitrary precision)
uint256  1000000000000000000  (exactly 4 x uint64, always 256 bits)

allocations per Mul
  big.Int, fresh receiver : 2   (the Int, plus its backing []Word)
  uint256, fresh receiver : 1   (just the struct — no backing slice)
  big.Int, reused receiver: 0   (capacity already there)
  uint256, reused receiver: 0

as a local value (no pointer escaping)
  var t big.Int; t.Mul(x,y)      : 1   (the []Word still escapes)
  var t uint256.Int; t.Mul(x,y)  : 0   (entirely on the stack)

2^256-1 + 1
  uint256          0   (overflow flag: true)
  big.Int          115792089237316195423570985008687907853269984665640564039457584007913129639936
  big.Int bit len  257   <- silently became a 257-bit number

uint256.FromBig(2^256) -> 0, overflowed=true

use uint256 in interpreters and hot loops (lesson 18, 57)
use big.Int in APIs and anywhere readability wins
```

---

## 18. Constant-time comparison

`🔴 hard` · *Timing*

`bytes.Equal` stops at the first differing byte, so the time it takes reveals how much of a guess was correct. The counter here stands in for that timing. Use constant-time comparison for secrets, MACs and tokens — and plain equality for public data, where there is nothing to leak.

**Steps:**

1. Compare a secret against three guesses with an early-exit comparator, counting byte comparisons.
2. Watch the count climb as the guess gets closer — that is the side channel.
3. Run the same guesses through a constant-time comparator and see a flat count.
4. Use `subtle.ConstantTimeCompare` and `hmac.Equal` rather than hand-rolling either.

```go
package main

import (
	"bytes"
	"crypto/hmac"
	"crypto/sha256"
	"crypto/subtle"
	"fmt"
)

// naiveEqual is what bytes.Equal does underneath: stop at the first difference.
// The counter stands in for the time an attacker measures.
func naiveEqual(a, b []byte) (equal bool, comparisons int) {
	if len(a) != len(b) {
		return false, 0 // length alone already leaked
	}
	for i := range a {
		comparisons++
		if a[i] != b[i] {
			return false, comparisons
		}
	}
	return true, comparisons
}

// ctEqual always inspects every byte, whatever it finds.
func ctEqual(a, b []byte) (equal bool, comparisons int) {
	if len(a) != len(b) {
		return false, 0
	}
	var diff byte
	for i := range a {
		comparisons++
		diff |= a[i] ^ b[i] // accumulate, never branch
	}
	return diff == 0, comparisons
}

func main() {
	secret := []byte("s3cr3t-api-token")

	// An attacker guesses one byte at a time. With an early-exit comparison,
	// the work done tells them exactly how much of their guess was right.
	guesses := [][]byte{
		[]byte("x3cr3t-api-token"),
		[]byte("s3cr3t-api-tokex"),
		[]byte("s3cr3t-api-token"),
	}

	fmt.Printf("%-20s %-7s %-12s %s\n", "guess", "equal", "naive work", "constant-time work")
	for _, g := range guesses {
		eq1, n1 := naiveEqual(secret, g)
		eq2, n2 := ctEqual(secret, g)
		fmt.Printf("%-20q %-7v %-12d %d\n", string(g), eq1, n1, n2)
		_ = eq2
	}
	fmt.Println("\nthe naive column climbs as the guess gets closer — that is the leak.")
	fmt.Println("measure it as elapsed time and you can recover a secret byte by byte.")

	// The standard library gives you this without hand-rolling it.
	fmt.Printf("\nsubtle.ConstantTimeCompare(secret, secret)   = %d\n",
		subtle.ConstantTimeCompare(secret, []byte("s3cr3t-api-token")))
	fmt.Printf("subtle.ConstantTimeCompare(secret, guess)   = %d\n",
		subtle.ConstantTimeCompare(secret, []byte("s3cr3t-api-tokex")))

	// The usual real-world case: verifying a webhook signature (lesson 51).
	key := []byte("webhook-signing-key") // TEST value only
	body := []byte(`{"event":"transfer","amount":"1000000"}`)
	mac := hmac.New(sha256.New, key)
	mac.Write(body)
	want := mac.Sum(nil)

	mac.Reset()
	mac.Write(body)
	got := mac.Sum(nil)

	fmt.Printf("\nhmac.Equal(want, got) = %v   <- always use this, not bytes.Equal\n",
		hmac.Equal(want, got))

	// And where it genuinely does not matter: public data.
	blockHash := sha256.Sum256([]byte("block 12345"))
	fmt.Printf("\nbytes.Equal on a public block hash = %v   (fine — nothing to leak)\n",
		bytes.Equal(blockHash[:], blockHash[:]))
	fmt.Println("\nconstant-time for secrets, MACs and tokens; plain equality for public bytes")
}
```

**Output:**

```
guess                equal   naive work   constant-time work
"x3cr3t-api-token"   false   1            16
"s3cr3t-api-tokex"   false   16           16
"s3cr3t-api-token"   true    16           16

the naive column climbs as the guess gets closer — that is the leak.
measure it as elapsed time and you can recover a secret byte by byte.

subtle.ConstantTimeCompare(secret, secret)   = 1
subtle.ConstantTimeCompare(secret, guess)   = 0

hmac.Equal(want, got) = true   <- always use this, not bytes.Equal

bytes.Equal on a public block hash = true   (fine — nothing to leak)

constant-time for secrets, MACs and tokens; plain equality for public bytes
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
