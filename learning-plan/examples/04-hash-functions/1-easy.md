# Step 04 — Cryptographic Hash Functions · 🟢 Easy

Examples **1–6**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get golang.org/x/crypto@latest               # examples 5, 7, 9, 20
go get github.com/ethereum/go-ethereum@latest   # examples 5, 14
# paste the example into main.go, then:
go run .
```

No chain, no node, no keys. Examples 1–4, 6, 8, 10–13 and 15–19 need only the standard library.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. Your first hash

`🟢 easy` · *Basics*

A hash takes any amount of input and returns a fixed number of bytes — 32 for SHA-256, whether you feed it nothing or a gigabyte. It is deterministic: the same input gives the same output on every machine, forever. That is the property everything else is built on.

**Steps:**

1. Hash inputs of four different lengths and note the output size never changes.
2. Check the result for `"abc"` against the published FIPS 180-4 test vector.
3. Hash the same input twice and compare — `Sum256` returns `[32]byte`, so `==` works (lesson 03).

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	// Any input length, always 32 bytes out.
	inputs := []string{
		"",
		"a",
		"abc",
		"the quick brown fox jumps over the lazy dog, twice, at some length",
	}

	fmt.Printf("%-10s %-6s %s\n", "input len", "out", "sha256")
	for _, in := range inputs {
		sum := sha256.Sum256([]byte(in))
		fmt.Printf("%-10d %-6d %s\n", len(in), len(sum), hex.EncodeToString(sum[:]))
	}

	// The FIPS 180-4 test vector for "abc". If your implementation disagrees
	// with this, stop and find out why before building anything on it.
	const want = "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
	got := sha256.Sum256([]byte("abc"))
	fmt.Printf("\nmatches the published \"abc\" vector: %v\n", hex.EncodeToString(got[:]) == want)

	// Sum256 returns [32]byte — an array, so it is comparable and copies by
	// value (lesson 03). That is exactly what you want for an identifier.
	a := sha256.Sum256([]byte("abc"))
	b := sha256.Sum256([]byte("abc"))
	fmt.Printf("deterministic: %v (same input, same output, every time)\n", a == b)
}
```

**Output:**

```
input len  out    sha256
0          32     e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
1          32     ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb
3          32     ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
66         32     9c736d668f7d743d6a55fce345e6f79627984c4637f75e1958e9542350bad5ec

matches the published "abc" vector: true
deterministic: true (same input, same output, every time)
```

---

## 2. Avalanche

`🟢 easy` · *Properties*

Change one bit of the input and roughly half the output bits change. This is the avalanche property, and it is what makes a digest useless for guessing how close your input was — there is no gradient to climb.

**Steps:**

1. Hash a string, flip the lowest bit of its first byte, and hash again.
2. Count the differing output bits with `bits.OnesCount8` over the XOR.
3. Repeat for all eight bits of the first byte and average the result.
4. Note it lands near 128 of 256 — half — every time.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"math/bits"
)

// diffBits counts how many bits differ between two equal-length byte slices.
func diffBits(a, b []byte) int {
	n := 0
	for i := range a {
		n += bits.OnesCount8(a[i] ^ b[i])
	}
	return n
}

func main() {
	base := []byte("block 12345")

	orig := sha256.Sum256(base)
	fmt.Printf("input   %q\n", base)
	fmt.Printf("sha256  %s\n", hex.EncodeToString(orig[:]))

	// Flip a single bit of the input — the lowest bit of the first byte.
	flipped := append([]byte(nil), base...)
	flipped[0] ^= 0x01
	other := sha256.Sum256(flipped)

	fmt.Printf("\ninput   %q   (one bit changed)\n", flipped)
	fmt.Printf("sha256  %s\n", hex.EncodeToString(other[:]))

	d := diffBits(orig[:], other[:])
	fmt.Printf("\nbits changed in the output: %d of 256 (%.0f%%)\n", d, float64(d)/256*100)

	// Do it for every one of the first byte's 8 bits and look at the spread.
	fmt.Println("\nflipping each bit of byte 0:")
	total := 0
	for bit := 0; bit < 8; bit++ {
		f := append([]byte(nil), base...)
		f[bit/8] ^= 1 << (bit % 8)
		h := sha256.Sum256(f)
		n := diffBits(orig[:], h[:])
		total += n
		fmt.Printf("  bit %d -> %3d bits differ\n", bit, n)
	}
	fmt.Printf("\naverage %.1f — about half, which is the avalanche property\n", float64(total)/8)
	fmt.Println("a hash output tells you nothing about how close the input was")
}
```

**Output:**

```
input   "block 12345"
sha256  0a12748dfde88d87e58deab89aefb04acd1ed92a25b40dd52414b3d31c9162e7

input   "clock 12345"   (one bit changed)
sha256  e3456581cd7984b1cfe75bf25dc5e5d564b1fd6f57d3b2fe2823e46fa8627dd7

bits changed in the output: 126 of 256 (49%)

flipping each bit of byte 0:
  bit 0 -> 126 bits differ
  bit 1 -> 117 bits differ
  bit 2 -> 125 bits differ
  bit 3 -> 133 bits differ
  bit 4 -> 126 bits differ
  bit 5 -> 132 bits differ
  bit 6 -> 120 bits differ
  bit 7 -> 128 bits differ

average 125.9 — about half, which is the avalanche property
a hash output tells you nothing about how close the input was
```

---

## 3. The hash.Hash interface

`🟢 easy` · *The Go API*

`sha256.Sum256` is the convenience form. The `hash.Hash` interface is the real one: `Write` as many times as you like, then `Sum`. Two things surprise people — `Sum(dst)` *appends* to `dst` rather than hashing it, and `Sum` does not reset the hasher.

**Steps:**

1. Compare `Sum256` with `New` + `Write`, then split the input across two `Write` calls.
2. Call `Sum(prefix)` and see the prefix still there with the digest appended.
3. Write again after calling `Sum` and watch the stream continue rather than restart.
4. That is why `Sum(nil)` is the idiom: it means "just the digest".

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"hash"
)

func main() {
	// Two ways to the same answer.
	one := sha256.Sum256([]byte("hello world")) // convenience, returns [32]byte
	fmt.Printf("Sum256      %s\n", hex.EncodeToString(one[:]))

	h := sha256.New() // the streaming interface, returns []byte
	h.Write([]byte("hello world"))
	fmt.Printf("New+Write   %s\n", hex.EncodeToString(h.Sum(nil)))

	// Write is incremental: hashing in pieces gives the same result as one shot.
	h.Reset()
	h.Write([]byte("hello "))
	h.Write([]byte("world"))
	fmt.Printf("in 2 writes %s\n", hex.EncodeToString(h.Sum(nil)))

	// Sum(dst) APPENDS the digest to dst — it does not hash dst.
	// Sum(nil) is the idiom precisely because it means "give me just the digest".
	h.Reset()
	h.Write([]byte("hello world"))
	prefix := []byte{0xff, 0xee}
	appended := h.Sum(prefix)
	fmt.Printf("\nSum(prefix) %s\n", hex.EncodeToString(appended))
	fmt.Printf("            ^^^^ the prefix is still there; the digest follows\n")

	// Sum does NOT reset. Calling it twice gives the same answer, and writing
	// after it continues the same stream.
	h.Reset()
	h.Write([]byte("a"))
	first := hex.EncodeToString(h.Sum(nil))
	h.Write([]byte("b")) // continues "a" -> "ab"
	second := hex.EncodeToString(h.Sum(nil))
	ab := sha256.Sum256([]byte("ab"))
	fmt.Printf("\nafter Write(\"a\")  %s\n", first)
	fmt.Printf("after Write(\"b\")  %s\n", second)
	fmt.Printf("sha256(\"ab\")      %s  same: %v\n",
		hex.EncodeToString(ab[:]), second == hex.EncodeToString(ab[:]))

	// hash.Hash is an interface, so anything can be swapped in (example 5).
	var any hash.Hash = sha256.New()
	fmt.Printf("\nSize()=%d BlockSize()=%d\n", any.Size(), any.BlockSize())
}
```

**Output:**

```
Sum256      b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9
New+Write   b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9
in 2 writes b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9

Sum(prefix) ffeeb94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9
            ^^^^ the prefix is still there; the digest follows

after Write("a")  ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb
after Write("b")  fb8e20fc2e4c3f248c60c39bd652f3c1347298bb977b8b4d5903b85055620603
sha256("ab")      fb8e20fc2e4c3f248c60c39bd652f3c1347298bb977b8b4d5903b85055620603  same: true

Size()=32 BlockSize()=64
```

---

## 4. Streaming a large input

`🟢 easy` · *The Go API*

`hash.Hash` is an `io.Writer`, so `io.Copy` streams any reader through it in fixed-size chunks. Peak memory is the buffer, not the input — this is how you hash a file, an HTTP body or a block you are downloading.

**Steps:**

1. Stream 10 MiB through a hasher with `io.Copy` and never hold it all in memory.
2. Confirm the digest matches hashing the same bytes in one shot.
3. Use `io.MultiWriter` to hash and copy in a single pass.
4. Note the reader tracks an absolute offset — a per-call index would change the bytes.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"strings"
)

// repeatReader produces n bytes without ever holding them all in memory.
// pos tracks the ABSOLUTE offset in the stream — a per-call index would
// restart the pattern at every Read boundary and produce different bytes.
type repeatReader struct {
	chunk     []byte
	remaining int
	pos       int
}

func (r *repeatReader) Read(p []byte) (int, error) {
	if r.remaining == 0 {
		return 0, io.EOF
	}
	n := len(p)
	if n > r.remaining {
		n = r.remaining
	}
	for i := 0; i < n; i++ {
		p[i] = r.chunk[(r.pos+i)%len(r.chunk)]
	}
	r.pos += n
	r.remaining -= n
	return n, nil
}

func main() {
	const size = 10 << 20 // 10 MiB

	// io.Copy streams through the hasher in 32 KiB chunks. Peak memory is the
	// buffer, not the input — this is how you hash a file or an HTTP body.
	h := sha256.New()
	src := &repeatReader{chunk: []byte("blockchain"), remaining: size}
	n, err := io.Copy(h, src)
	fmt.Printf("hashed %d bytes (%d MiB), err=%v\n", n, n>>20, err)
	fmt.Printf("sha256 %s\n", hex.EncodeToString(h.Sum(nil)))

	// Same bytes, built in memory, to prove streaming changes nothing.
	buf := strings.Repeat("blockchain", size/10)
	one := sha256.Sum256([]byte(buf))
	fmt.Printf("\nsame as hashing it all at once: %v\n",
		hex.EncodeToString(one[:]) == hex.EncodeToString(h.Sum(nil)))

	// io.MultiWriter hashes and copies in one pass — useful when you are
	// downloading something and want its digest without a second read.
	h2 := sha256.New()
	var sink discard
	src2 := &repeatReader{chunk: []byte("blockchain"), remaining: size}
	if _, err := io.Copy(io.MultiWriter(h2, sink), src2); err != nil {
		fmt.Println("copy:", err)
		return
	}
	fmt.Printf("via MultiWriter, same digest:  %v\n",
		hex.EncodeToString(h2.Sum(nil)) == hex.EncodeToString(h.Sum(nil)))
}

type discard struct{}

func (discard) Write(p []byte) (int, error) { return len(p), nil }
```

**Output:**

```
hashed 10485760 bytes (10 MiB), err=<nil>
sha256 36b865df6b9516640f565883075b145e64e4c3ff44152649949b4f54e82c6952

same as hashing it all at once: true
via MultiWriter, same digest:  true
```

---

## 5. SHA-256, SHA3-256 and Keccak-256

`🟢 easy` · *Which hash*

Keccak won the SHA-3 competition, then NIST changed the padding before standardising it. Ethereum had already shipped the original. So **Ethereum's "sha3" is Keccak-256, not SHA3-256** — different functions, different outputs, and a constant source of bugs.

**Steps:**

1. Hash `"abc"` with SHA-256, SHA3-256 and Keccak-256 and compare all three.
2. Confirm SHA3-256 and Keccak-256 differ despite the similar names.
3. Check `crypto.Keccak256` from go-ethereum agrees with `sha3.NewLegacyKeccak256`.
4. Note `Keccak256Hash` returns a `common.Hash`, which is usually what you want to store.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"

	gcrypto "github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/sha3"
)

func main() {
	in := []byte("abc")

	s256 := sha256.Sum256(in)

	// NIST SHA3-256 — the standardised version, with the padding NIST changed.
	s3 := sha3.Sum256(in)

	// Original Keccak-256 — what Ethereum actually uses, everywhere.
	k := sha3.NewLegacyKeccak256()
	k.Write(in)
	keccak := k.Sum(nil)

	fmt.Printf("input %q\n\n", in)
	fmt.Printf("sha256      %s\n", hex.EncodeToString(s256[:]))
	fmt.Printf("sha3-256    %s\n", hex.EncodeToString(s3[:]))
	fmt.Printf("keccak-256  %s\n", hex.EncodeToString(keccak))

	fmt.Printf("\nsha3-256 == keccak-256 ? %v\n",
		hex.EncodeToString(s3[:]) == hex.EncodeToString(keccak))
	fmt.Println("  they differ by ONE padding byte in the spec, and that is enough")

	// go-ethereum's helper is the same thing, and it is what you will call.
	fmt.Printf("\ncrypto.Keccak256      %s\n", hex.EncodeToString(gcrypto.Keccak256(in)))
	fmt.Printf("matches sha3.NewLegacyKeccak256: %v\n",
		hex.EncodeToString(gcrypto.Keccak256(in)) == hex.EncodeToString(keccak))

	// Keccak256Hash returns a common.Hash ([32]byte), which is usually what
	// you want to store or compare (lesson 03).
	h := gcrypto.Keccak256Hash(in)
	fmt.Printf("Keccak256Hash         %s  (type %T)\n", h.Hex(), h)

	fmt.Println("\nEthereum says \"sha3\" in old docs and opcode names but means KECCAK-256.")
	fmt.Println("Use SHA3-256 by mistake and everything verifies locally and fails on-chain.")
}
```

**Output:**

```
input "abc"

sha256      ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
sha3-256    3a985da74fe225b2045c172d6bd390bd855f086e3e9d525b46bfe24511431532
keccak-256  4e03657aea45a94fc7d47ba826c8d667c0d1e6e33a64a036ec44f58fa12d6c45

sha3-256 == keccak-256 ? false
  they differ by ONE padding byte in the spec, and that is enough

crypto.Keccak256      4e03657aea45a94fc7d47ba826c8d667c0d1e6e33a64a036ec44f58fa12d6c45
matches sha3.NewLegacyKeccak256: true
Keccak256Hash         0x4e03657aea45a94fc7d47ba826c8d667c0d1e6e33a64a036ec44f58fa12d6c45  (type common.Hash)

Ethereum says "sha3" in old docs and opcode names but means KECCAK-256.
Use SHA3-256 by mistake and everything verifies locally and fails on-chain.
```

---

## 6. One-way is not unguessable

`🟢 easy` · *Properties*

Preimage resistance means there is no way to invert a digest. It does not mean the input is unguessable. If the input comes from a small set, an attacker enumerates the set — and the hash function has done nothing wrong.

**Steps:**

1. Publish the digest of a 4-digit PIN.
2. Recover it by hashing all 10,000 candidates.
3. Note where the weakness was: the input's entropy, not the hash.
4. See how a 128-bit nonce moves the same value permanently out of reach (example 12).

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	// Preimage resistance: given only the digest, you cannot compute the input.
	// There is no inverse function. But "cannot invert" is not "cannot guess".
	secret := "4271"
	sum := sha256.Sum256([]byte(secret))
	digest := hex.EncodeToString(sum[:])
	fmt.Printf("published digest: %s\n", digest)
	fmt.Println("the input is a 4-digit PIN. Nothing else is known.")

	// The search space is 10,000. That is not cryptography, that is a loop.
	tried := 0
	found := ""
	for i := 0; i < 10000; i++ {
		guess := fmt.Sprintf("%04d", i)
		tried++
		h := sha256.Sum256([]byte(guess))
		if hex.EncodeToString(h[:]) == digest {
			found = guess
			break
		}
	}
	fmt.Printf("\nrecovered %q after %d guesses\n", found, tried)

	// The hash was not broken. The INPUT had no entropy.
	fmt.Println("\nthe hash function did its job perfectly — the input was the weakness")
	fmt.Println("this is exactly why a commitment needs a random nonce (example 12)")

	// With a 128-bit nonce the same PIN becomes unguessable: 2^128 candidates.
	fmt.Printf("\n4-digit PIN alone      : 10,000 candidates\n")
	fmt.Printf("PIN + 128-bit nonce    : 10,000 x 2^128 candidates\n")
	fmt.Println("adding the nonce costs 16 bytes and moves it out of reach forever")
}
```

**Output:**

```
published digest: 0b94cd5e95afb17f118c0ed090fa2621bae55c692e2f7b13948c7d805985974e
the input is a 4-digit PIN. Nothing else is known.

recovered "4271" after 4272 guesses

the hash function did its job perfectly — the input was the weakness
this is exactly why a commitment needs a random nonce (example 12)

4-digit PIN alone      : 10,000 candidates
PIN + 128-bit nonce    : 10,000 x 2^128 candidates
adding the nonce costs 16 bytes and moves it out of reach forever
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
