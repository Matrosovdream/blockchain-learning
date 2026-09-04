# 03 — Bytes, Hex, Big Integers & Encoding

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-01-foundations.md](plan/part-01-foundations.md#03-bytes-hex-big-integers-encoding)

| | |
|---|---|
| **Part** | Part 1 — Foundations |
| **Prerequisites** | [02](02-environment-setup.md) |
| **Unlocks** | 04 |
| **Examples** | [18](examples/03-bytes-encoding/) (🟢 6 · 🟡 7 · 🔴 5) |

*The data primitives — `[]byte`, endianness, hex, base58, bech32, `big.Int`, fixed-size arrays.*

This is the least glamorous lesson in the course and the one whose absence hurts most. Every later
lesson assumes you can move fluently between bytes, hex and integers without thinking about it. Get
these habits now and the rest of the course is about ideas; skip them and you will spend it debugging
byte order.

## Goals

- Move fluently between `[]byte`, hex strings and integers in Go.
- Explain endianness and pick the right one when serializing.
- Use `math/big`/`uint256` for values that overflow `uint64` — which is most of them.
- Know why `[32]byte` and `[]byte` are different types and where each is used.

## Concepts

### 1. Everything is bytes

A block, a transaction, an address, a signature, a hash — on the wire they are all just bytes. Go
gives you two ways to hold them, and the difference matters more here than in most Go code.

**`[]byte` is a slice**: a header (pointer, length, capacity) pointing at a backing array. It can be
`nil`, it can grow, and assigning it **shares** the backing array.

**`[N]byte` is an array**: a value of exactly N bytes. Assigning it **copies** all N. It is
comparable with `==`, and it can be a map key.

That last property is why go-ethereum defines its core identifiers as arrays:

```go
type Hash    [32]byte   // common.Hash
type Address [20]byte   // common.Address
```

You can compare them directly, use them as map keys, and store them without worrying that someone
else holds a pointer into the same memory. With `[]byte` none of that is true — `==` does not
compile, map keys are not allowed, and two variables can silently be the same bytes.

Example [2](examples/03-bytes-encoding/1-easy.md#2-arrays-copy-slices-alias) shows the difference in
eight lines. It is worth internalising, because the aliasing bug looks like this:

```go
hash := hasher.Sum(nil)   // a fresh slice — fine
store[key] = hash         // but if Sum had reused a buffer, this now changes underneath you
```

Which brings us to the harder version of the same problem: **a `[]byte` a library hands you is not
yours**. Plenty of decoders, hashers and readers reuse one internal buffer across calls, so the slice
you carefully stored quietly becomes something else on the next call. Example
[3](examples/03-bytes-encoding/1-easy.md#3-owning-your-bytes) demonstrates it and then fixes it:

```go
mine := append([]byte(nil), theirs...)   // or: mine := make([]byte, len(theirs)); copy(mine, theirs)
```

The everyday toolbox: `bytes.Equal` for comparison (not `==`, which does not compile for slices),
`bytes.Compare` when you need ordering, `copy` to take ownership, and `make([]byte, 0, n)` to
preallocate when you know the size.

### 2. Hex encoding

Two hex characters encode one byte. Always. A 32-byte hash is 64 hex characters; a 20-byte address
is 40. If your string length is not exactly twice your byte length, something is wrong.

The standard library's `encoding/hex` is strict, and example
[1](examples/03-bytes-encoding/1-easy.md#1-hex-round-trip-and-the-odd-length-error) walks its three
failure modes:

```go
hex.EncodeToString([]byte{0xde, 0xad})  // "dead"
hex.DecodeString("dead")                // [0xde 0xad], nil
hex.DecodeString("abc")                 // error: odd length hex string
hex.DecodeString("0xdead")              // error: invalid byte: U+0078 'x'
```

That last one surprises people. **`encoding/hex` does not know about the `0x` prefix** — `x` is
simply not a hex digit.

The `0x` prefix is a convention, and JSON-RPC uses it universally so that a hex string is never
mistaken for a decimal one. go-ethereum gives you three decoders with very different attitudes, and
example [7](examples/03-bytes-encoding/2-medium.md#7-three-ways-to-decode-hex-and-which-to-trust)
puts them side by side:

| Function | Prefix | On bad input |
|---|---|---|
| `common.FromHex` | optional | never errors; left-pads odd input |
| `common.Hex2Bytes` | must be absent | never errors; returns empty for prefixed input |
| `hexutil.Decode` | **required** | typed errors you can match with `errors.Is` |

The rule: forgiving helpers for your own hardcoded constants, `hexutil.Decode` for anything that
crossed a process boundary.

`hexutil` also encodes the distinction JSON-RPC draws between two kinds of hex, which example
[8](examples/03-bytes-encoding/2-medium.md#8-quantity-vs-data) makes concrete:

- **QUANTITY** — a number. Minimal digits, **no leading zeros**. `0x0`, `0x41`, `0x400`.
- **DATA** — a byte blob. Always an **even** number of digits, leading zeros preserved. `0x0041`.

So `"0x41"` is 65 as a quantity and one byte as data, and `"0x041"` is a valid quantity string in
appearance but is rejected by geth for its leading zero. Use `EncodeUint64`/`DecodeUint64` and
`DecodeBig` for quantities, `Encode`/`Decode` for data.

### 3. Endianness

Endianness is simply which end of a multi-byte number goes first. Big-endian puts the most
significant byte first, which is how humans write numbers. Little-endian puts the least significant
byte first.

**Ethereum is big-endian on the wire. Bitcoin serializes little-endian.** You will work with both.

```go
be := make([]byte, 8)
binary.BigEndian.PutUint64(be, 0x0123456789abcdef)     // 0123456789abcdef
le := make([]byte, 8)
binary.LittleEndian.PutUint64(le, 0x0123456789abcdef)  // efcdab8967452301
```

The dangerous part, shown in example
[4](examples/03-bytes-encoding/1-easy.md#4-big-endian-vs-little-endian), is that reading with the
wrong decoder **does not error**. It returns a perfectly plausible wrong number. Nothing crashes,
nothing logs, and the bug survives review.

Bitcoin adds a second twist that catches everyone: **a transaction id is displayed reversed from the
byte order it is hashed and stored in**. The same 32 bytes have an "internal" order and a "display"
order, and explorers show the display one. Example
[10](examples/03-bytes-encoding/2-medium.md#10-bitcoins-genesis-hash-byte-by-byte) rebuilds the
genesis block header field by field, hashes it, reverses the result, and checks it against the
published genesis hash — so if any byte order were wrong, it would say so:

```
internal  6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000
display   000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
```

Those leading zeros are the proof of work ([09](09-proof-of-work.md)) — and they are only at the
*front* in display order.

> **Rule of thumb:** if a hex string looks backwards — familiar bytes in an unfamiliar order — it
> probably is. Reverse it before assuming the data is wrong.

### 4. Big integers

1 ether is 10¹⁸ wei. That single fact drives every arithmetic decision in this course.

`uint64` maxes out at 18,446,744,073,709,551,615 — just over **18.44 ether**. Fine for one small
transfer; useless for a balance, a total supply, or any sum. And Go's overflow behaviour here is
asymmetric in a way worth knowing: constant expressions are caught by the compiler, but the same
arithmetic on *variables* wraps silently at runtime. Example
[5](examples/03-bytes-encoding/1-easy.md#5-ether-wei-and-why-uint64-is-not-enough) shows 19 ether
wrapping to 553255926290448384 with no panic and no error.

So on-chain values are `*big.Int`. The API you need:

```go
n := big.NewInt(1000)
n, ok := new(big.Int).SetString("1000000000000000000", 10)  // note the base
z := new(big.Int).Add(x, y)      // z = x + y
z.Mul(z, y)                       // z = z * y
whole, frac := new(big.Int).QuoRem(wei, unit, new(big.Int))
n.Cmp(m)                          // -1, 0, +1 — there is no ==
n.Text(16)                        // to a string in any base
```

Two traps live in there.

**`SetString` takes a base, and reports failure with an `ok` boolean rather than an `error`.** Pass
the wrong base and `"1000"` becomes 4096 instead of 1000 — a valid, wrong number. Ignore `ok` and you
are holding a `nil *big.Int` that panics on first use. Base `0` infers from the `0x`/`0b`/`0o`
prefix, which is often what you want. Example
[6](examples/03-bytes-encoding/1-easy.md#6-setstring-takes-a-base) covers both.

**`big.Int` methods write into the receiver.** `z.Add(x, y)` means `z = x + y`, not "return `x + y`".
Use an operand as the receiver and you destroy it — along with anything else holding that pointer.
Example [11](examples/03-bytes-encoding/2-medium.md#11-the-bigint-aliasing-trap) works through the
three severities: overwriting a local, handing out internal state from a getter, and corrupting a
package-level "constant" for the life of the process:

```go
func (b *Balance) Wei() *big.Int     { return b.wei }                    // BUG: caller can mutate it
func (b *Balance) WeiSafe() *big.Int { return new(big.Int).Set(b.wei) }  // a copy
```

Three rules that prevent all of it: **fresh receiver for results**, **`Set()` to copy**, and **never
share a `*big.Int` across goroutines** — it is not safe for concurrent use, even for reads, because
any method might reallocate.

`big.Rat` exists for exact rational arithmetic, and it is the right tool when you genuinely need a
ratio — a price, a percentage — rather than a display string. And `float64` is never the right tool
for any of it: `0.1 + 0.2` is `0.29999999999999998890`, and `1e18 + 1` is `1e18`.

### 5. uint256

The EVM's word is 256 bits, and `github.com/holiman/uint256` models it exactly: four `uint64`s of
plain data, wrapping arithmetic, no heap allocation.

The allocation difference is real but narrower than people assume, and example
[17](examples/03-bytes-encoding/3-hard.md#17-bigint-vs-uint256) measures it with
`testing.AllocsPerRun` rather than guessing:

| Pattern | `big.Int` | `uint256` |
|---|---|---|
| Fresh receiver (`new(T).Mul(x, y)`) | 2 | 1 |
| Reused receiver | 0 | 0 |
| **Local value (`var t T; t.Mul(x, y)`)** | **1** | **0** |

That last row is the real story. A `uint256.Int` is 32 bytes of data, so a local never leaves the
stack. A `big.Int` always carries a slice, which escapes. In an interpreter loop
([18](18-evm.md)) or a hot decode path ([57](57-high-throughput-ingestion.md)) that is the
difference that matters; everywhere else, reach for `big.Int` and keep the code readable.

The behavioural difference matters more than the allocations. **The EVM wraps at 2²⁵⁶; `big.Int`
grows.** Adding 1 to 2²⁵⁶−1 gives 0 in `uint256` (with an overflow flag available via
`AddOverflow`) and a 257-bit number in `big.Int`. So `big.Int` does *not* faithfully model EVM
arithmetic — which is usually what you want in Go, and is a bug when you are writing an interpreter.
Converting down with `uint256.FromBig` truncates, and returns a flag saying so. Check it.

### 6. Fixed-width serialization

The EVM has one unit: a 32-byte word. Everything the ABI encodes is padded to it — numbers and
addresses on the **left**, `bytesN` on the **right**:

```go
raw := make([]byte, 8)
binary.BigEndian.PutUint64(raw, 1234)
word := common.LeftPadBytes(raw, 32)
// 00000000000000000000000000000000000000000000000000000000000004d2

common.RightPadBytes([]byte("hello"), 32)
// 68656c6c6f000000000000000000000000000000000000000000000000000000
```

`common.TrimLeftZeroes` goes back the other way. Example
[9](examples/03-bytes-encoding/2-medium.md#9-the-32-byte-evm-word) builds two words — an address and
an amount — which is precisely what an ABI-encoded function call looks like
([23](23-abi-encoding.md)).

**Do not reach for `binary.Write` on a struct to build these.** It has no field tags, no control over
padding, and it silently depends on declaration order, so reordering fields for readability changes
your bytes. Write the layout explicitly, one field at a time, in a documented order.

That discipline matters beyond the ABI, because **deterministic encoding is a hard requirement before
hashing**. Two nodes that serialize the same struct differently compute different hashes and disagree
about the chain. Lesson [04](04-hash-functions.md) makes this rule explicit and
[08](08-blocks-and-chain.md) depends on it completely.

### 7. Address and hash encodings

Hex is fine for machines. Addresses get read aloud, retyped, and written on paper, so they use
alphabets designed for humans.

- **base64** — dense, but includes `+`, `/`, and both cases of every letter. Rare on-chain.
- **base58** — base64 minus the confusable characters. Bitcoin and Solana.
- **base58check** — base58 over `version ‖ payload ‖ checksum`. Bitcoin's legacy addresses.
- **bech32 / bech32m** — SegWit and Taproot addresses (`bc1q…`, `bc1p…`), with a BCH error-detecting
  code that can also locate the mistyped character. Specified in
  [BIP-173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki); the details are in
  [36](36-bitcoin-deep-dive.md).

base58 drops exactly four characters — `0` (zero), `O` (capital o), `I` (capital i) and `l`
(lowercase L) — because those are the pairs humans confuse. Encoding is repeated division by 58 using
`big.Int`, with one wrinkle: leading zero bytes carry no numeric value, so they are handled
separately and become literal `1` characters. Example
[15](examples/03-bytes-encoding/3-hard.md#15-base58-from-scratch) implements both directions.

Checksums are a UX feature: they turn "funds sent into the void" into "invalid address". base58check
appends the first four bytes of a double SHA-256, which means roughly a 1-in-2³² chance a random typo
slips through. Example
[16](examples/03-bytes-encoding/3-hard.md#16-base58check-and-catching-a-typo) decodes Bitcoin's
genesis coinbase address, re-encodes it to confirm the round trip, then corrupts single characters
and watches the checksum catch every one.

Ethereum took a different route: addresses are plain hex, and the checksum is smuggled into the
*letter case* of that hex — EIP-55, covered in [07](07-addresses-wallets-hd.md). Same purpose,
cleverer packaging.

### 8. Units and money formatting

Three rules, and they are not negotiable.

**Compute in the smallest unit, always.** wei for Ethereum, satoshi for Bitcoin, and the token's own
base unit for everything else. Convert for display at the very last moment, and never convert back.

**Decimals are a property of the token, not a constant.** ETH is 18. USDC and USDT are 6. WBTC is 8.
Some tokens use 0. Hardcoding 18 for a 6-decimal token is wrong by a factor of a trillion — example
[12](examples/03-bytes-encoding/2-medium.md#12-decimals-are-per-token) shows 2500 USDC rendering as
`0.0000000025`. Read `decimals()` from the contract ([26](26-erc-standards.md)) and store it
alongside the amount.

**Format with integer arithmetic.** `QuoRem` gives you the whole part and the remainder; pad the
remainder to `decimals` digits and trim trailing zeros:

```go
unit := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(decimals)), nil)
whole, frac := new(big.Int).QuoRem(raw, unit, new(big.Int))
s := fmt.Sprintf("%s.%0*s", whole, decimals, frac)
s = strings.TrimSuffix(strings.TrimRight(s, "0"), ".")
```

Parsing user input runs the same machinery backwards, with one critical addition: **if the input has
more decimal places than the token supports, reject it**. Do not round, do not truncate. Silently
turning `0.0000001` USDC into `0` is how users lose money without anyone noticing. Example
[13](examples/03-bytes-encoding/2-medium.md#13-parsing-user-input-without-losing-money) is a complete
parser with typed errors for every failure mode.

Finally, give units a type. `Wei` and `Gwei` have identical representations, so nothing but the type
system stops you passing one where the other is expected — an error of 10⁹. Example
[14](examples/03-bytes-encoding/3-hard.md#14-a-wei-type-the-compiler-enforces) makes the compiler
catch it:

```go
type Wei  struct{ v *big.Int }
type Gwei struct{ v *big.Int }

func TxFee(gasUsed uint64, price Wei) Wei { ... }
// TxFee(21000, baseFee)  // does not compile: Gwei is not Wei
```

The wrapper also stops internal `*big.Int` state escaping, which closes the aliasing hole from
topic 4 at the same time.

### 9. Constant-time comparison

`bytes.Equal` stops at the first differing byte. That is the obvious optimisation and it is a side
channel: the time the comparison takes reveals **how much of the input was correct**. An attacker
guessing a token one byte at a time gets feedback on every attempt, and recovers the secret in
linear rather than exponential time.

Example [18](examples/03-bytes-encoding/3-hard.md#18-constant-time-comparison) makes the leak
visible by counting byte comparisons instead of measuring nanoseconds:

```
guess                equal   naive work   constant-time work
"x3cr3t-api-token"   false   1            16
"s3cr3t-api-tokex"   false   16           16
```

The fix accumulates differences instead of branching, and the standard library does it for you:

```go
subtle.ConstantTimeCompare(a, b)   // 1 if equal, 0 otherwise
hmac.Equal(mac1, mac2)             // for MACs specifically
```

Use them for **secrets, MAC/signature verification, API keys and session tokens** — you will need
this for webhook signatures in [51](51-wallet-integration-backends.md) and signing-service
authentication in [32](32-key-management-signing.md).

Use plain `bytes.Equal` for **public data** — block hashes, transaction ids, addresses. There is
nothing to leak, and constant-time comparison is slower for no benefit. Note also that neither
function hides *length*: they return early if the lengths differ, so a length difference always
leaks. Where that matters, compare fixed-size values.

## Exercises

Write these in `practice/03-bytes-encoding/`.

1. **Hex round-trip, three ways.** Write `mustHex(s string) []byte` that accepts input with or
   without a `0x` prefix, rejects odd lengths, and panics with a clear message otherwise. Then write
   the inverse. Test both against `hexutil.Decode` on ten inputs including the edge cases.
2. **Prove the aliasing bug.** Write a `Ledger` with a `Balance(addr) *big.Int` method that returns
   internal state. Write a test that calls it, mutates the result, and asserts the ledger is
   corrupted. Then fix the method and assert the test now fails to corrupt anything.
3. **Endianness both ways.** Serialize a `uint32` and a `uint64` in both orders. Then take a Bitcoin
   txid in display order, convert it to internal order, and back. Write a table-driven test.
4. **Rebuild the genesis header.** Reproduce example 10 from memory: assemble the 80-byte header,
   double-SHA256 it, reverse, and assert it equals the known genesis hash. Then change one field by
   one and confirm the hash no longer matches.
5. **A money formatter.** Write `Format(raw *big.Int, decimals int) string` and
   `Parse(s string, decimals int) (*big.Int, error)` such that `Parse(Format(x, d), d) == x` for all
   `x`. Fuzz it with `testing.F` across decimals 0, 2, 6, 8 and 18. Fix whatever the fuzzer finds.
6. **base58check.** Implement encode and decode. Verify against
   `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa`. Then write a test that corrupts every character position in
   turn and asserts the checksum catches all of them.
7. **Measure it yourself.** Use `testing.AllocsPerRun` to compare `big.Int` and `uint256` for a loop
   of 1000 multiply-accumulates. Then add `-benchmem` benchmarks. Write down which one you would
   use for an EVM interpreter and which for a REST API, and why.
8. **Find the leak.** Write a `checkToken(provided, expected string) bool` using `==`. Instrument it
   to count byte comparisons. Show that the count reveals the shared prefix length, then fix it with
   `subtle.ConstantTimeCompare` and show the count is now flat.

## Best Practices & Pitfalls

- **Never put `float64` anywhere near a money path.**
  *Why:* `float64` has 53 bits of mantissa and wei needs 60+. The precision does not go missing
  loudly — it silently mints or burns value, and the chain records the result faithfully forever.
  `math/big` or `uint256`, always.
- **Use a fresh receiver for `big.Int` results, and `Set()` to copy.**
  *Why:* `z.Add(x, y)` writes into `z`. Passing an operand as the receiver destroys it, and returning
  internal state from a getter lets any caller corrupt your balance. A package-level `*big.Int`
  "constant" mutated once stays wrong for the life of the process.
- **Never share a `*big.Int` across goroutines, even for reads.**
  *Why:* it is not safe for concurrent use. Methods can reallocate the backing slice, so a read
  racing a write is a genuine data race, not a stale value.
- **Always pass `SetString`'s base, and always check its `ok`.**
  *Why:* the wrong base gives a valid wrong number — `"1000"` in base 16 is 4096. And ignoring `ok`
  leaves you holding `nil`, which panics on the next method call rather than at the parse site.
- **Copy any `[]byte` you did not allocate before storing it.**
  *Why:* libraries reuse buffers. The hash you saved changes on the next call to the same hasher, and
  the corruption appears far from its cause.
- **Match the hex decoder to the trust level of the input.**
  *Why:* `common.FromHex` and `Hex2Bytes` never error — they left-pad odd input and return empty for
  an unexpected prefix. That is fine for your own constants and dangerous for anything external. Use
  `hexutil.Decode` on input.
- **Keep QUANTITY and DATA straight.**
  *Why:* JSON-RPC numbers have no leading zeros and blobs are byte-aligned. `"0x41"` is 65 or one
  byte depending which rule applies, and geth rejects `"0x041"` as a quantity.
- **Read `decimals()` from the token; never assume 18.**
  *Why:* USDC and USDT use 6, WBTC uses 8. Assuming 18 is an error of 10¹², and it is one of the most
  common real production bugs in this space.
- **Reject over-precise input rather than rounding it.**
  *Why:* truncating `0.0000001` USDC to `0` loses the user's money silently. An error message costs
  nothing.
- **Constant-time comparison for secrets, plain equality for public bytes.**
  *Why:* early-exit comparison leaks the matched prefix length through timing. It costs nothing to
  use `subtle.ConstantTimeCompare` on a token, and nothing is gained by using it on a block hash.

## Checklist

- [ ] I can convert between `[]byte`, hex strings and integers without looking anything up.
- [ ] I can explain why `common.Hash` is `[32]byte` and not `[]byte`.
- [ ] I know that a `[]byte` from a library may be reused, and I copy before storing.
- [ ] I can serialize a number big-endian and little-endian, and I know which chain uses which.
- [ ] I can explain why a Bitcoin txid displays reversed from how it is hashed.
- [ ] I use `*big.Int` for every on-chain value and never `float64`.
- [ ] I can state the `big.Int` receiver rule and the three ways it goes wrong.
- [ ] I know when `uint256` earns its place and when `big.Int` is the better choice.
- [ ] I can left-pad a value into a 32-byte EVM word and trim it back.
- [ ] I can implement base58check and explain what its four checksum bytes buy.
- [ ] I format and parse token amounts with integer arithmetic, honouring per-token decimals.
- [ ] I use constant-time comparison for secrets and plain equality for public data.

## Resources

**Standard library**

- Go — `math/big`: https://pkg.go.dev/math/big
- Go — `encoding/binary`: https://pkg.go.dev/encoding/binary
- Go — `encoding/hex`: https://pkg.go.dev/encoding/hex
- Go — `crypto/subtle`: https://pkg.go.dev/crypto/subtle

**Ecosystem**

- `hexutil` (go-ethereum): https://pkg.go.dev/github.com/ethereum/go-ethereum/common/hexutil
- `common` (go-ethereum): https://pkg.go.dev/github.com/ethereum/go-ethereum/common
- `uint256`: https://pkg.go.dev/github.com/holiman/uint256

**Encodings**

- JSON-RPC hex encoding rules: https://ethereum.org/en/developers/docs/apis/json-rpc/#hex-encoding
- Base58Check encoding: https://en.bitcoin.it/wiki/Base58Check_encoding
- BIP-173 (bech32): https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
- BIP-350 (bech32m): https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki

---

**Examples:** [`examples/03-bytes-encoding/`](examples/03-bytes-encoding/) — **18 runnable Go
programs** (🟢 6 easy · 🟡 7 medium · 🔴 5 hard). No chain, no node, no keys — all pure computation.
Two of them check their own answers against real Bitcoin data: example 10 rebuilds the genesis block
header and asserts its hash, and example 16 round-trips the genesis coinbase address through
base58check.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
