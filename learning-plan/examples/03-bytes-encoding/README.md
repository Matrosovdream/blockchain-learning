# Step 03 — Bytes, Hex, Big Integers & Encoding · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

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

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 | hex, arrays vs slices, endianness, wei, `SetString` |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–13 | hexutil, EVM words, the genesis hash, aliasing, decimals |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | unit types, base58check, uint256, constant-time |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. Hex round-trip, and the odd-length error](1-easy.md#1-hex-round-trip-and-the-odd-length-error)
- [2. Arrays copy, slices alias](1-easy.md#2-arrays-copy-slices-alias)
- [3. Owning your bytes](1-easy.md#3-owning-your-bytes)
- [4. Big-endian vs little-endian](1-easy.md#4-big-endian-vs-little-endian)
- [5. Ether, wei, and why uint64 is not enough](1-easy.md#5-ether-wei-and-why-uint64-is-not-enough)
- [6. SetString takes a base](1-easy.md#6-setstring-takes-a-base)

### 🟡 [Medium](2-medium.md)

- [7. Three ways to decode hex, and which to trust](2-medium.md#7-three-ways-to-decode-hex-and-which-to-trust)
- [8. QUANTITY vs DATA](2-medium.md#8-quantity-vs-data)
- [9. The 32-byte EVM word](2-medium.md#9-the-32-byte-evm-word)
- [10. Bitcoin's genesis hash, byte by byte](2-medium.md#10-bitcoins-genesis-hash-byte-by-byte)
- [11. The big.Int aliasing trap](2-medium.md#11-the-bigint-aliasing-trap)
- [12. Decimals are per token](2-medium.md#12-decimals-are-per-token)
- [13. Parsing user input without losing money](2-medium.md#13-parsing-user-input-without-losing-money)

### 🔴 [Hard](3-hard.md)

- [14. A Wei type the compiler enforces](3-hard.md#14-a-wei-type-the-compiler-enforces)
- [15. base58 from scratch](3-hard.md#15-base58-from-scratch)
- [16. base58check, and catching a typo](3-hard.md#16-base58check-and-catching-a-typo)
- [17. big.Int vs uint256](3-hard.md#17-bigint-vs-uint256)
- [18. Constant-time comparison](3-hard.md#18-constant-time-comparison)

## Two that check their own answers

**[10. Bitcoin's genesis hash](2-medium.md#10-bitcoins-genesis-hash-byte-by-byte)** rebuilds the
80-byte genesis header from its fields, hashes it, and compares the result against the published
block hash. If any byte order were wrong, it would print `false`.

**[16. base58check](3-hard.md#16-base58check-and-catching-a-typo)** decodes the real genesis
coinbase address, re-encodes it, and asserts it gets the identical string back — then corrupts it
on purpose to prove the checksum works.

Both are worth running before you trust anything you write in this area.

## The arc

1–3 — bytes: hex in and out, why arrays copy and slices alias, and whose memory you are holding.  
4 — endianness, and the silent wrong answer it gives you.  
5–6 — why every on-chain value needs `big.Int`, and the two ways `SetString` betrays you.  
7–9 — the real hex APIs, JSON-RPC's two conventions, and the 32-byte EVM word.  
10–11 — endianness for real, and the aliasing trap that corrupts a balance.  
12–14 — decimals, safe parsing, and types that make the compiler catch unit errors.  
15–16 — base58 and base58check, built from scratch against a real address.  
17–18 — when to reach for `uint256`, and when equality has to be constant-time.

---

*Lesson: [../../03-bytes-encoding.md](../../03-bytes-encoding.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
