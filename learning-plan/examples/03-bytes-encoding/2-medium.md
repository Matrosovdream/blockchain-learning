# Step 03 — Bytes, Hex, Big Integers & Encoding · 🟡 Medium

Examples **7–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 7. Three ways to decode hex, and which to trust

`🟡 medium` · *Hex*

go-ethereum offers three hex decoders with very different attitudes. Two are forgiving and never return an error, which is fine for your own constants and dangerous for anything from outside the process. Use `hexutil.Decode` on input.

**Steps:**

1. Run a prefixed, an unprefixed and an odd-length string through `common.FromHex`.
2. Watch `common.Hex2Bytes` return empty for a `0x`-prefixed string, silently.
3. Use `hexutil.Decode` and match its typed errors with `errors.Is`.
4. Note `"0x"` decodes successfully to zero bytes, while `""` is an error.

```go
package main

import (
	"errors"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
)

func main() {
	const withPrefix = "0xdeadbeef"
	const noPrefix = "deadbeef"

	// common.FromHex is forgiving: it strips 0x if present, and pads odd input.
	fmt.Println("common.FromHex (forgiving, never errors)")
	fmt.Printf("  %-12q -> % x\n", withPrefix, common.FromHex(withPrefix))
	fmt.Printf("  %-12q -> % x\n", noPrefix, common.FromHex(noPrefix))
	fmt.Printf("  %-12q -> % x   <- odd length, silently left-padded\n", "abc", common.FromHex("abc"))

	// common.Hex2Bytes assumes no prefix and no error path at all.
	fmt.Println("\ncommon.Hex2Bytes (assumes no prefix, no errors)")
	fmt.Printf("  %-12q -> % x\n", noPrefix, common.Hex2Bytes(noPrefix))
	fmt.Printf("  %-12q -> % x   <- garbage in, empty out, no complaint\n",
		withPrefix, common.Hex2Bytes(withPrefix))

	// hexutil.Decode validates. Use this on anything from outside your process.
	fmt.Println("\nhexutil.Decode (validates, returns typed errors)")
	b, err := hexutil.Decode(withPrefix)
	fmt.Printf("  %-12q -> % x err=%v\n", withPrefix, b, err)

	for _, in := range []string{noPrefix, "0xabc", "0x", ""} {
		_, err := hexutil.Decode(in)
		fmt.Printf("  %-12q -> err=%v (MissingPrefix=%v OddLength=%v)\n", in, err,
			errors.Is(err, hexutil.ErrMissingPrefix), errors.Is(err, hexutil.ErrOddLength))
	}

	fmt.Println("\nrule: forgiving helpers for your own constants, hexutil.Decode for input")
}
```

**Output:**

```
common.FromHex (forgiving, never errors)
  "0xdeadbeef" -> de ad be ef
  "deadbeef"   -> de ad be ef
  "abc"        -> 0a bc   <- odd length, silently left-padded

common.Hex2Bytes (assumes no prefix, no errors)
  "deadbeef"   -> de ad be ef
  "0xdeadbeef" ->    <- garbage in, empty out, no complaint

hexutil.Decode (validates, returns typed errors)
  "0xdeadbeef" -> de ad be ef err=<nil>
  "deadbeef"   -> err=hex string without 0x prefix (MissingPrefix=true OddLength=false)
  "0xabc"      -> err=hex string of odd length (MissingPrefix=false OddLength=true)
  "0x"         -> err=<nil> (MissingPrefix=false OddLength=false)
  ""           -> err=empty hex string (MissingPrefix=false OddLength=false)

rule: forgiving helpers for your own constants, hexutil.Decode for input
```

---

## 8. QUANTITY vs DATA

`🟡 medium` · *Hex*

JSON-RPC has two hex conventions and mixing them is a real bug. A **QUANTITY** is a number: minimal digits, no leading zeros. **DATA** is a byte blob: always byte-aligned, leading zeros preserved. The same string means different things under each.

**Steps:**

1. Encode numbers with `EncodeUint64` and blobs with `Encode`, and compare the shapes.
2. Decode `"0x41"` both ways — 65 as a quantity, one byte as data.
3. Watch `"0x041"` be rejected as a quantity for its leading zero.
4. Use `DecodeBig` for values above 64 bits.

```go
package main

import (
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common/hexutil"
)

func main() {
	// JSON-RPC has TWO hex conventions and mixing them up is a real bug.
	//
	//   QUANTITY - a number.  Minimal digits, no leading zeros. "0x41", "0x0".
	//   DATA     - a byte blob. Always an even number of digits. "0x0041".
	fmt.Println("QUANTITY (numbers): minimal, no leading zeros")
	for _, n := range []uint64{0, 65, 1024, 1_000_000_000} {
		fmt.Printf("  %-12d -> %s\n", n, hexutil.EncodeUint64(n))
	}

	fmt.Println("\nDATA (byte blobs): byte-aligned, leading zeros preserved")
	for _, b := range [][]byte{{}, {0x00, 0x41}, {0xde, 0xad}} {
		fmt.Printf("  % -8x -> %q\n", b, hexutil.Encode(b))
	}

	// The same string means different things under the two rules.
	q, err := hexutil.DecodeUint64("0x41")
	fmt.Printf("\nas QUANTITY: \"0x41\" -> %d  err=%v\n", q, err)
	d, err := hexutil.Decode("0x41")
	fmt.Printf("as DATA    : \"0x41\" -> % x  err=%v\n", d, err)

	// A quantity with a leading zero is invalid, and geth rejects it.
	_, err = hexutil.DecodeUint64("0x041")
	fmt.Printf("\nquantity \"0x041\" -> err=%v\n", err)

	// Big values use DecodeBig; anything over 256 bits is refused.
	v, err := hexutil.DecodeBig("0xde0b6b3a7640000")
	fmt.Printf("\nDecodeBig(\"0xde0b6b3a7640000\") = %s wei (%s ether)\n",
		v, new(big.Int).Div(v, big.NewInt(1e18)))
}
```

**Output:**

```
QUANTITY (numbers): minimal, no leading zeros
  0            -> 0x0
  65           -> 0x41
  1024         -> 0x400
  1000000000   -> 0x3b9aca00

DATA (byte blobs): byte-aligned, leading zeros preserved
           -> "0x"
  00 41    -> "0x0041"
  de ad    -> "0xdead"

as QUANTITY: "0x41" -> 65  err=<nil>
as DATA    : "0x41" -> 41  err=<nil>

quantity "0x041" -> err=hex number with leading zero digits

DecodeBig("0xde0b6b3a7640000") = 1000000000000000000 wei (1 ether)
```

---

## 9. The 32-byte EVM word

`🟡 medium` · *Serialization*

The EVM's universal unit is a 32-byte word. Numbers and addresses are padded on the **left**, `bytesN` on the **right**. Build these layouts explicitly, one field at a time — `binary.Write` on a struct depends on declaration order and gives you no control over padding.

**Steps:**

1. Serialize a `uint64` big-endian and left-pad it to a full word.
2. Right-pad a short byte string the way `bytesN` is encoded.
3. Trim the padding back off and recover the number.
4. Build two words — an address and an amount — which is exactly what lesson 23 formalises.

```go
package main

import (
	"encoding/binary"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
)

func main() {
	// The EVM's universal unit is a 32-byte word. Everything the ABI encodes is
	// padded to it: numbers and addresses on the LEFT, bytesN on the RIGHT.
	const n uint64 = 1234

	raw := make([]byte, 8)
	binary.BigEndian.PutUint64(raw, n)
	fmt.Printf("uint64 %d as 8 bytes : % x\n", n, raw)

	word := common.LeftPadBytes(raw, 32)
	fmt.Printf("left-padded to a word: %x\n", word)
	fmt.Printf("                       ^ 24 zero bytes, then the value\n")

	// bytesN goes the other way: the data first, zeros after.
	tag := []byte("hello")
	fmt.Printf("\nbytes5 %q right-padded: %x\n", tag, common.RightPadBytes(tag, 32))

	// Round-trip: strip the padding back off.
	fmt.Printf("\ntrimmed back         : % x -> %d\n",
		common.TrimLeftZeroes(word), binary.BigEndian.Uint64(word[24:]))

	// Do NOT reach for binary.Write on a struct to build these.
	// It has no field tags, no control over padding, and silently depends on
	// declaration order — three ways to change the bytes without meaning to.
	// Write the layout explicitly, one field at a time, in a documented order.
	fmt.Println("\nbuilding two words explicitly (an ABI call's arguments):")
	addr := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266")
	amount := make([]byte, 8)
	binary.BigEndian.PutUint64(amount, 1_000_000)

	args := make([]byte, 0, 64)
	args = append(args, common.LeftPadBytes(addr.Bytes(), 32)...)
	args = append(args, common.LeftPadBytes(amount, 32)...)
	for i := 0; i < len(args); i += 32 {
		fmt.Printf("  word %d: %x\n", i/32, args[i:i+32])
	}
	fmt.Println("\nthat is exactly what lesson 23 formalises as ABI encoding")
}
```

**Output:**

```
uint64 1234 as 8 bytes : 00 00 00 00 00 00 04 d2
left-padded to a word: 00000000000000000000000000000000000000000000000000000000000004d2
                       ^ 24 zero bytes, then the value

bytes5 "hello" right-padded: 68656c6c6f000000000000000000000000000000000000000000000000000000

trimmed back         : 04 d2 -> 1234

building two words explicitly (an ABI call's arguments):
  word 0: 000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266
  word 1: 00000000000000000000000000000000000000000000000000000000000f4240

that is exactly what lesson 23 formalises as ABI encoding
```

---

## 10. Bitcoin's genesis hash, byte by byte

`🟡 medium` · *Endianness*

Bitcoin's genesis block header, rebuilt field by field and hashed. It exercises everything at once: little-endian integers, hashes stored in internal order, and the reversal explorers apply for display. The program checks its own result against the known genesis hash.

**Steps:**

1. Assemble the 80-byte header: version, prev block, merkle root, time, bits, nonce.
2. Reverse the merkle root from display order into the internal order the header stores.
3. Double-SHA256 it, then reverse the result to get the familiar leading zeros.
4. Compare against the published genesis hash — the program asserts it for you.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// hash256 is Bitcoin's double SHA-256.
func hash256(b []byte) []byte {
	first := sha256.Sum256(b)
	second := sha256.Sum256(first[:])
	return second[:]
}

// reverse returns a reversed copy — internal byte order to display order.
func reverse(b []byte) []byte {
	out := make([]byte, len(b))
	for i := range b {
		out[i] = b[len(b)-1-i]
	}
	return out
}

func main() {
	// Rebuild Bitcoin's genesis block header, field by field, exactly as it is
	// serialized: 80 bytes, little-endian numbers, hashes in internal order.
	header := make([]byte, 0, 80)

	var u32 [4]byte
	binary.LittleEndian.PutUint32(u32[:], 1) // version
	header = append(header, u32[:]...)

	header = append(header, make([]byte, 32)...) // prev block: all zeros

	// The merkle root, as it appears in explorers (display order) — so we must
	// reverse it to get the internal order the header actually stores.
	merkleDisplay, _ := hex.DecodeString(
		"4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b")
	header = append(header, reverse(merkleDisplay)...)

	binary.LittleEndian.PutUint32(u32[:], 1231006505) // timestamp
	header = append(header, u32[:]...)
	binary.LittleEndian.PutUint32(u32[:], 0x1d00ffff) // bits
	header = append(header, u32[:]...)
	binary.LittleEndian.PutUint32(u32[:], 2083236893) // nonce
	header = append(header, u32[:]...)

	fmt.Printf("header is %d bytes\n", len(header))
	fmt.Printf("raw       %x\n", header)

	sum := hash256(header)
	fmt.Printf("\ninternal  %s   <- what the hash function returns\n", hex.EncodeToString(sum))
	fmt.Printf("display   %s   <- reversed, what an explorer shows\n", hex.EncodeToString(reverse(sum)))

	const want = "000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f"
	fmt.Printf("\nmatches the known genesis hash: %v\n", hex.EncodeToString(reverse(sum)) == want)
	fmt.Println("\nsame 32 bytes, two orders — get this wrong and nothing matches anything")
}
```

**Output:**

```
header is 80 bytes
raw       0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a29ab5f49ffff001d1dac2b7c

internal  6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000   <- what the hash function returns
display   000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f   <- reversed, what an explorer shows

matches the known genesis hash: true

same 32 bytes, two orders — get this wrong and nothing matches anything
```

---

## 11. The big.Int aliasing trap

`🟡 medium` · *Big integers*

`big.Int` methods write into the **receiver** and return it: `z.Add(x, y)` means `z = x + y`. Use an operand as the receiver and you destroy it, along with anything else holding that pointer — including package-level values you thought were constants.

**Steps:**

1. Compare `new(big.Int).Add(x, y)` with `a.Add(a, b)`.
2. Return an internal `*big.Int` from a getter and let the caller corrupt the balance.
3. Return `new(big.Int).Set(...)` instead and confirm the balance survives.
4. Corrupt a package-level 'constant' to see the worst version of the bug.

```go
package main

import (
	"fmt"
	"math/big"
)

// Balance looks harmless. It is not: it hands out a pointer to its own state.
type Balance struct{ wei *big.Int }

func (b *Balance) Wei() *big.Int     { return b.wei }                   // BUG: callers can mutate it
func (b *Balance) WeiSafe() *big.Int { return new(big.Int).Set(b.wei) } // a copy

func main() {
	// big.Int methods write into the RECEIVER and return it.
	// z.Add(x, y) means "z = x + y", not "return x + y".
	x := big.NewInt(10)
	y := big.NewInt(3)

	z := new(big.Int).Add(x, y) // correct: a fresh receiver
	fmt.Printf("new(big.Int).Add(x,y): x=%s y=%s z=%s\n", x, y, z)

	// Using an operand as the receiver silently destroys it.
	a := big.NewInt(10)
	b := big.NewInt(3)
	a.Add(a, b) // a is now 13 — and anything else holding a sees 13
	fmt.Printf("a.Add(a,b)          : a=%s b=%s   <- a was overwritten\n", a, b)

	// The same trap, one layer up.
	bal := &Balance{wei: big.NewInt(1000)}
	stolen := bal.Wei()
	stolen.SetInt64(0) // caller "just wanted to compute something"
	fmt.Printf("\nafter caller touched Wei(): balance=%s\n", bal.wei)

	bal2 := &Balance{wei: big.NewInt(1000)}
	copy2 := bal2.WeiSafe()
	copy2.SetInt64(0)
	fmt.Printf("after caller touched WeiSafe(): balance=%s\n", bal2.wei)

	// Package-level constants are the worst version of this, because the damage
	// is permanent for the life of the process.
	oneEther := big.NewInt(1e18)
	total := oneEther // NOT a copy — same pointer
	total.Mul(total, big.NewInt(5))
	fmt.Printf("\n'constant' oneEther is now %s   <- corrupted for everyone\n", oneEther)

	fmt.Println("\nrules: fresh receiver for results, Set() to copy, never share a *big.Int")
}
```

**Output:**

```
new(big.Int).Add(x,y): x=10 y=3 z=13
a.Add(a,b)          : a=13 b=3   <- a was overwritten

after caller touched Wei(): balance=0
after caller touched WeiSafe(): balance=1000

'constant' oneEther is now 5000000000000000000   <- corrupted for everyone

rules: fresh receiver for results, Set() to copy, never share a *big.Int
```

---

## 12. Decimals are per token

`🟡 medium` · *Units*

Token decimals are a property of the token, not a constant. USDC and USDT use 6, WBTC uses 8, ETH uses 18. The same raw integer means wildly different amounts depending on which — and hardcoding 18 for a 6-decimal token is wrong by a factor of a trillion.

**Steps:**

1. Format one raw value as five different tokens and compare.
2. Format 2500 USDC with decimals 6 and again with 18 to see the size of the error.
3. Do all of it with `QuoRem` and string padding — no floats anywhere.
4. Use `big.Rat` when you need an exact ratio rather than a display string.

```go
package main

import (
	"fmt"
	"math/big"
	"strings"
)

// Token decimals are per token and are NOT always 18.
type Token struct {
	Symbol   string
	Decimals int
}

// Format renders a raw on-chain amount for humans. Integer arithmetic only.
func (t Token) Format(raw *big.Int) string {
	unit := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(t.Decimals)), nil)
	whole, frac := new(big.Int).QuoRem(raw, unit, new(big.Int))
	if t.Decimals == 0 {
		return whole.String()
	}
	s := fmt.Sprintf("%s.%0*s", whole, t.Decimals, frac)
	s = strings.TrimRight(s, "0")
	return strings.TrimSuffix(s, ".")
}

func main() {
	tokens := []Token{
		{"ETH", 18}, {"USDC", 6}, {"USDT", 6}, {"WBTC", 8}, {"GUSD", 2},
	}

	// The SAME raw integer means wildly different amounts per token.
	raw, _ := new(big.Int).SetString("1500000000", 10)
	fmt.Printf("raw on-chain value: %s\n\n", raw)
	fmt.Printf("%-6s %-9s %s\n", "token", "decimals", "displays as")
	for _, t := range tokens {
		fmt.Printf("%-6s %-9d %s\n", t.Symbol, t.Decimals, t.Format(raw))
	}

	// Hardcoding 18 for a 6-decimal token is off by a factor of a trillion.
	usdc := Token{"USDC", 6}
	wrong := Token{"USDC", 18}
	amount, _ := new(big.Int).SetString("2500000000", 10) // 2500 USDC
	fmt.Printf("\n2500 USDC raw = %s\n", amount)
	fmt.Printf("  with decimals=6  -> %s USDC   (correct)\n", usdc.Format(amount))
	fmt.Printf("  with decimals=18 -> %s USDC   (off by 10^12)\n", wrong.Format(amount))

	// big.Rat when you need an exact ratio rather than a display string.
	price := new(big.Rat).SetFrac(amount, new(big.Int).Exp(big.NewInt(10), big.NewInt(6), nil))
	fmt.Printf("\nas an exact big.Rat: %s (%s)\n", price.RatString(), price.FloatString(2))

	fmt.Println("\nalways read decimals() from the contract — never assume 18 (lesson 26)")
}
```

**Output:**

```
raw on-chain value: 1500000000

token  decimals  displays as
ETH    18        0.0000000015
USDC   6         1500
USDT   6         1500
WBTC   8         15
GUSD   2         15000000

2500 USDC raw = 2500000000
  with decimals=6  -> 2500 USDC   (correct)
  with decimals=18 -> 0.0000000025 USDC   (off by 10^12)

as an exact big.Rat: 2500 (2500.00)

always read decimals() from the contract — never assume 18 (lesson 26)
```

---

## 13. Parsing user input without losing money

`🟡 medium` · *Units*

User input arrives as `"1.5"` and has to become an exact integer. The critical rule: if the input has more decimal places than the token supports, **reject it**. Silently truncating `0.0000001` USDC to zero is how users lose money and nobody notices.

**Steps:**

1. Split on the decimal point and right-pad the fraction to exactly `decimals` digits.
2. Reject empty, negative, malformed and non-numeric input with distinct sentinel errors.
3. Reject anything more precise than the token supports, rather than rounding it.
4. Note there is no `float64` anywhere in the parser.

```go
package main

import (
	"errors"
	"fmt"
	"math/big"
	"strings"
)

var (
	ErrEmpty        = errors.New("empty amount")
	ErrNotANumber   = errors.New("not a number")
	ErrTooPrecise   = errors.New("more decimal places than the token supports")
	ErrNegative     = errors.New("negative amount")
	ErrMultipleDots = errors.New("malformed decimal")
)

// ParseAmount turns human input ("1.5") into a raw on-chain integer.
// It never uses float64, and it refuses input it cannot represent exactly.
func ParseAmount(s string, decimals int) (*big.Int, error) {
	s = strings.TrimSpace(s)
	if s == "" {
		return nil, ErrEmpty
	}
	if strings.HasPrefix(s, "-") {
		return nil, ErrNegative
	}
	parts := strings.Split(s, ".")
	if len(parts) > 2 {
		return nil, ErrMultipleDots
	}
	whole, frac := parts[0], ""
	if len(parts) == 2 {
		frac = parts[1]
	}
	if whole == "" {
		whole = "0"
	}
	// This is the check that stops a user losing money to silent truncation.
	if len(frac) > decimals {
		return nil, fmt.Errorf("%w: got %d, token supports %d", ErrTooPrecise, len(frac), decimals)
	}
	// Right-pad the fraction to exactly `decimals` digits, then it is just an integer.
	frac += strings.Repeat("0", decimals-len(frac))

	n, ok := new(big.Int).SetString(whole+frac, 10)
	if !ok {
		return nil, ErrNotANumber
	}
	return n, nil
}

func main() {
	const decimals = 6 // USDC

	inputs := []string{"1", "1.5", "0.000001", "1.000000", "  2.25  ", "", "1.2.3", "-5", "abc", "0.0000001"}

	fmt.Printf("parsing as USDC (%d decimals)\n\n", decimals)
	fmt.Printf("%-14s %-22s %s\n", "input", "raw", "note")
	for _, in := range inputs {
		n, err := ParseAmount(in, decimals)
		if err != nil {
			fmt.Printf("%-14q %-22s %v\n", in, "-", err)
			continue
		}
		fmt.Printf("%-14q %-22s ok\n", in, n)
	}

	fmt.Println("\n0.0000001 is rejected rather than truncated to 0 — refuse, never round")
}
```

**Output:**

```
parsing as USDC (6 decimals)

input          raw                    note
"1"            1000000                ok
"1.5"          1500000                ok
"0.000001"     1                      ok
"1.000000"     1000000                ok
"  2.25  "     2250000                ok
""             -                      empty amount
"1.2.3"        -                      malformed decimal
"-5"           -                      negative amount
"abc"          -                      not a number
"0.0000001"    -                      more decimal places than the token supports: got 7, token supports 6

0.0000001 is rejected rather than truncated to 0 — refuse, never round
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
