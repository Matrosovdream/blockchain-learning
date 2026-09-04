# Step 04 — Cryptographic Hash Functions · 🟡 Medium

Examples **7–14**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 7. HASH256 and HASH160

`🟡 medium` · *Bitcoin*

Bitcoin never hashes once. `HASH256` is SHA-256 applied twice and produces block hashes, txids and Merkle nodes. `HASH160` is RIPEMD-160 over SHA-256 and produces addresses — 20 bytes instead of 32, shorter to type and still 160 bits.

**Steps:**

1. Implement both and check RIPEMD-160(`"abc"`) against its published vector.
2. Compare the output sizes: 32 bytes for HASH256, 20 for HASH160.
3. Note the real motivation for double hashing — the outer hash eats a fixed 32 bytes, so there is nothing to extend (example 8).
4. Learn where each one appears in Bitcoin.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"

	"golang.org/x/crypto/ripemd160"
)

// HASH256 = SHA256(SHA256(x)). Bitcoin uses it for block hashes, txids and
// Merkle nodes.
func hash256(b []byte) []byte {
	first := sha256.Sum256(b)
	second := sha256.Sum256(first[:])
	return second[:]
}

// HASH160 = RIPEMD160(SHA256(x)). Bitcoin uses it for addresses: 20 bytes
// instead of 32, which is shorter to type and still 160 bits of security.
func hash160(b []byte) []byte {
	first := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(first[:])
	return r.Sum(nil)
}

func main() {
	in := []byte("abc")

	s := sha256.Sum256(in)
	fmt.Printf("sha256(\"abc\")     %s\n", hex.EncodeToString(s[:]))
	fmt.Printf("hash256(\"abc\")    %s  (%d bytes)\n", hex.EncodeToString(hash256(in)), len(hash256(in)))
	fmt.Printf("hash160(\"abc\")    %s  (%d bytes)\n", hex.EncodeToString(hash160(in)), len(hash160(in)))

	// Published test vectors. If these do not match, something is wrong.
	r := ripemd160.New()
	r.Write(in)
	fmt.Printf("\nripemd160(\"abc\")  %s\n", hex.EncodeToString(r.Sum(nil)))
	fmt.Printf("matches published vector: %v\n",
		hex.EncodeToString(r.Sum(nil)) == "8eb208f7e05d987a9b044a8e98c6b087f15a0bfc")

	// Double hashing is not twice the security. Its real motivation was
	// length-extension resistance (example 8): the outer SHA-256 hashes a
	// fixed 32-byte input, so there is no message to extend.
	fmt.Println("\nwhy double-hash?")
	fmt.Println("  the outer hash consumes a fixed 32 bytes, so nothing can be appended")
	fmt.Println("  it was belt-and-braces in 2008; SHA-256 has held up fine on its own")

	// Where each one shows up.
	fmt.Println("\nin Bitcoin")
	fmt.Println("  hash256  block header hash, txid, merkle nodes, base58check checksum")
	fmt.Println("  hash160  P2PKH and P2WPKH addresses (lesson 07, 36)")
}
```

**Output:**

```
sha256("abc")     ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
hash256("abc")    4f8b42c22dd3729b519ba6f68d2da7cc5b2d606d05daed5ad5128cc03e6c6358  (32 bytes)
hash160("abc")    bb1be98c142444d7a56aa3981c3942a978e4dc33  (20 bytes)

ripemd160("abc")  8eb208f7e05d987a9b044a8e98c6b087f15a0bfc
matches published vector: true

why double-hash?
  the outer hash consumes a fixed 32 bytes, so nothing can be appended
  it was belt-and-braces in 2008; SHA-256 has held up fine on its own

in Bitcoin
  hash256  block header hash, txid, merkle nodes, base58check checksum
  hash160  P2PKH and P2WPKH addresses (lesson 07, 36)
```

---

## 8. Breaking a naive MAC by length extension

`🟡 medium` · *Length extension*

A working forgery, not a description of one. SHA-256 is Merkle–Damgård, which means **the digest is the internal state**. Given `H(secret‖msg)` and the length of the secret, anyone can resume hashing and append whatever they like — without ever learning the secret.

**Steps:**

1. Build the naive MAC `H(secret‖msg)` that looks reasonable and is not.
2. Reconstruct a hasher from the published digest using `encoding.BinaryUnmarshaler`.
3. Compute the glue padding SHA-256 would have appended, then write the attacker's extension.
4. Watch the server recompute the MAC with the real secret and accept the forgery.

```go
package main

import (
	"crypto/sha256"
	"encoding"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// naiveMAC is the bug: authenticating by hashing the secret in front of the
// message. It looks reasonable and it is forgeable.
func naiveMAC(secret, msg []byte) []byte {
	sum := sha256.Sum256(append(append([]byte{}, secret...), msg...))
	return sum[:]
}

// mdPadding returns the padding SHA-256 appends to a message of n bytes:
// 0x80, then zeros, then the bit length as a big-endian uint64.
func mdPadding(n int) []byte {
	pad := []byte{0x80}
	for (n+len(pad))%64 != 56 {
		pad = append(pad, 0)
	}
	var l [8]byte
	binary.BigEndian.PutUint64(l[:], uint64(n)*8)
	return append(pad, l[:]...)
}

// resume rebuilds a SHA-256 hasher positioned exactly where a digest left off.
// It needs the digest and the TOTAL number of bytes already absorbed
// (including padding) — but not the secret.
func resume(digest []byte, absorbed int) (h interface {
	Write([]byte) (int, error)
	Sum([]byte) []byte
}, err error) {
	state := make([]byte, 0, 108)
	state = append(state, "sha\x03"...) // crypto/sha256 state magic
	state = append(state, digest...)    // h[0..7], big-endian
	state = append(state, make([]byte, 64)...)
	state = binary.BigEndian.AppendUint64(state, uint64(absorbed))

	hh := sha256.New()
	if err := hh.(encoding.BinaryUnmarshaler).UnmarshalBinary(state); err != nil {
		return nil, err
	}
	return hh, nil
}

func main() {
	// The server knows this. The attacker does not — and never learns it.
	secret := []byte("server-side-secret-key")
	msg := []byte("amount=10&to=alice")

	tag := naiveMAC(secret, msg)
	fmt.Println("what the attacker sees")
	fmt.Printf("  message %q\n", msg)
	fmt.Printf("  tag     %s\n", hex.EncodeToString(tag))
	fmt.Printf("  (and guesses the secret is %d bytes long)\n", len(secret))

	// The attack. Absorbed = secret+message+padding, all multiples of 64.
	absorbed := len(secret) + len(msg)
	glue := mdPadding(absorbed)

	h, err := resume(tag, absorbed+len(glue))
	if err != nil {
		fmt.Println("resume:", err)
		return
	}
	extension := []byte("&to=attacker")
	h.Write(extension)
	forgedTag := h.Sum(nil)

	// The forged message is the original, plus the glue padding, plus whatever
	// the attacker wanted to append.
	forgedMsg := append(append(append([]byte{}, msg...), glue...), extension...)

	fmt.Println("\nthe forgery")
	fmt.Printf("  message %q\n", forgedMsg)
	fmt.Printf("  tag     %s\n", hex.EncodeToString(forgedTag))

	// The server verifies with the real secret. It has no idea.
	real := naiveMAC(secret, forgedMsg)
	fmt.Printf("\nserver recomputes: %s\n", hex.EncodeToString(real))
	fmt.Printf("forgery accepted : %v   <- without ever knowing the secret\n",
		hex.EncodeToString(real) == hex.EncodeToString(forgedTag))

	fmt.Println("\nSHA-256 is Merkle-Damgard: the digest IS the internal state, so anyone")
	fmt.Println("can resume hashing from it. Example 9 shows the fix.")
}
```

**Output:**

```
what the attacker sees
  message "amount=10&to=alice"
  tag     c7ea6b3887e699dc9424f667e7957a9f4c5a24e336e7cf3276ef95145174aa7e
  (and guesses the secret is 22 bytes long)

the forgery
  message "amount=10&to=alice\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01@&to=attacker"
  tag     b03164b45dac35d61cea1c8de3f751602a1b447efe98deabddb526f9ce98c294

server recomputes: b03164b45dac35d61cea1c8de3f751602a1b447efe98deabddb526f9ce98c294
forgery accepted : true   <- without ever knowing the secret

SHA-256 is Merkle-Damgard: the digest IS the internal state, so anyone
can resume hashing from it. Example 9 shows the fix.
```

---

## 9. HMAC, and the sponge alternative

`🟡 medium` · *Length extension*

Two fixes. HMAC nests two keyed hashes so the output is not a resumable state — it is the answer, and it is in the standard library. Keccak's sponge construction is structurally immune, but "immune by construction" is a fragile thing to rely on. Use HMAC.

**Steps:**

1. Build and verify an HMAC-SHA256 tag, comparing with `hmac.Equal` (constant time — lesson 03).
2. Confirm a tampered message fails.
3. Hash `secret‖msg` with Keccak and note why the sponge is not extendable.
4. See the same construction used for a webhook signature (lesson 51).

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"

	"golang.org/x/crypto/sha3"
)

func main() {
	secret := []byte("server-side-secret-key")
	msg := []byte("amount=10&to=alice")

	// --- fix 1: HMAC ---------------------------------------------------------
	// HMAC nests two hashes with keyed padding, so the digest is not a resumable
	// internal state. There is nothing for a length-extension attack to grab.
	mac := hmac.New(sha256.New, secret)
	mac.Write(msg)
	tag := mac.Sum(nil)
	fmt.Printf("HMAC-SHA256      %s\n", hex.EncodeToString(tag))

	// Verification always uses hmac.Equal — constant time (lesson 03).
	mac.Reset()
	mac.Write(msg)
	fmt.Printf("verifies         %v\n", hmac.Equal(tag, mac.Sum(nil)))

	mac.Reset()
	mac.Write([]byte("amount=10&to=attacker"))
	fmt.Printf("tampered message %v\n", hmac.Equal(tag, mac.Sum(nil)))

	// --- fix 2: use a sponge -------------------------------------------------
	// Keccak/SHA-3 absorb and squeeze; the digest is only part of the state, so
	// H(secret||msg) is not extendable in the first place.
	k := sha3.NewLegacyKeccak256()
	k.Write(secret)
	k.Write(msg)
	fmt.Printf("\nkeccak(secret||msg) %s\n", hex.EncodeToString(k.Sum(nil)))
	fmt.Println("  structurally immune to length extension — but still use HMAC,")
	fmt.Println("  because 'immune by construction' is a fragile thing to rely on")

	// --- the general rule ----------------------------------------------------
	fmt.Println("\nnever hand-roll a MAC:")
	fmt.Println("  H(secret || msg)   forgeable on SHA-256 (example 8)")
	fmt.Println("  H(msg || secret)   weaker against collisions, still not a MAC")
	fmt.Println("  hmac.New(...)      the answer, in the standard library")

	// This is exactly what verifies a webhook signature (lesson 51).
	body := []byte(`{"event":"transfer","amount":"1000000"}`)
	whmac := hmac.New(sha256.New, []byte("webhook-signing-key")) // TEST key only
	whmac.Write(body)
	fmt.Printf("\nwebhook signature  sha256=%s\n", hex.EncodeToString(whmac.Sum(nil)))
}
```

**Output:**

```
HMAC-SHA256      6c03788fda4a8277bf5f070d23409f9763b2b140e44ce8225f5d6e186bd08126
verifies         true
tampered message false

keccak(secret||msg) 11c84150d6a8cac32a9aee46cfb3f90c572d0b81b18a9ef164b3f202227adb53
  structurally immune to length extension — but still use HMAC,
  because 'immune by construction' is a fragile thing to rely on

never hand-roll a MAC:
  H(secret || msg)   forgeable on SHA-256 (example 8)
  H(msg || secret)   weaker against collisions, still not a MAC
  hmac.New(...)      the answer, in the standard library

webhook signature  sha256=4ca38e61bed713fc4db34808ea746d4d4c6f850dd2d01cf2d15e7f802ba844cd
```

---

## 10. Nondeterminism, and where it hides

`🟡 medium` · *Determinism*

Two nodes must compute the same hash from the same data, so every input to a hash has to be canonical. Go's map iteration is randomised — a genuine consensus bug. And while `encoding/json` *does* sort map keys, it emits struct fields in declaration order, so reordering fields silently changes your commitment.

**Steps:**

1. Hash a map by ranging over it 100 times and check whether the digests agree.
2. Sort the keys first and check again.
3. Marshal two structs with identical data but reordered fields, and compare their hashes.
4. Conclude: JSON is not a canonical format, so never hash it.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"slices"
)

// hashByRange folds a map by iterating it directly. Go randomises map order,
// so this produces a different digest almost every time it runs.
func hashByRange(m map[string]int64) string {
	h := sha256.New()
	for k, v := range m {
		fmt.Fprintf(h, "%s=%d;", k, v)
	}
	return hex.EncodeToString(h.Sum(nil))
}

// hashSorted fixes the order first. Same map, same digest, forever.
func hashSorted(m map[string]int64) string {
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	slices.Sort(keys)

	h := sha256.New()
	for _, k := range keys {
		fmt.Fprintf(h, "%s=%d;", k, m[k])
	}
	return hex.EncodeToString(h.Sum(nil))
}

// Two versions of the same struct, fields reordered. encoding/json emits
// fields in DECLARATION order, so these serialize differently.
type AccountV1 struct {
	Address string `json:"address"`
	Balance int64  `json:"balance"`
}

type AccountV2 struct {
	Balance int64  `json:"balance"`
	Address string `json:"address"`
}

func main() {
	balances := map[string]int64{
		"alice": 10, "bob": 20, "carol": 30, "dave": 40,
		"erin": 50, "frank": 60, "grace": 70, "heidi": 80,
	}

	seen := map[string]bool{}
	for i := 0; i < 100; i++ {
		seen[hashByRange(balances)] = true
	}
	fmt.Printf("map iteration, 100 runs -> all digests identical? %v\n", len(seen) == 1)

	sortedSeen := map[string]bool{}
	var stable string
	for i := 0; i < 100; i++ {
		stable = hashSorted(balances)
		sortedSeen[stable] = true
	}
	fmt.Printf("sorted keys,   100 runs -> all digests identical? %v\n", len(sortedSeen) == 1)
	fmt.Printf("stable digest: %s\n", stable[:32])

	// JSON of a MAP is fine — encoding/json sorts map keys.
	// JSON of a STRUCT follows declaration order, which is a version-drift trap.
	j1, _ := json.Marshal(AccountV1{Address: "0xabc", Balance: 100})
	j2, _ := json.Marshal(AccountV2{Address: "0xabc", Balance: 100})
	h1 := sha256.Sum256(j1)
	h2 := sha256.Sum256(j2)

	fmt.Printf("\nsame data, fields declared in a different order:\n")
	fmt.Printf("  v1 %s\n", j1)
	fmt.Printf("  v2 %s\n", j2)
	fmt.Printf("  same hash? %v   <- reordering fields changed the commitment\n",
		hex.EncodeToString(h1[:]) == hex.EncodeToString(h2[:]))

	fmt.Println("\nJSON is not a canonical format: whitespace, escaping and number")
	fmt.Println("formatting are all free choices. Never hash it. Example 11 does it right.")
}
```

**Output:**

```
map iteration, 100 runs -> all digests identical? false
sorted keys,   100 runs -> all digests identical? true
stable digest: f007d7fb59d6be436d443cbb74c65f51

same data, fields declared in a different order:
  v1 {"address":"0xabc","balance":100}
  v2 {"balance":100,"address":"0xabc"}
  same hash? false   <- reordering fields changed the commitment

JSON is not a canonical format: whitespace, escaping and number
formatting are all free choices. Never hash it. Example 11 does it right.
```

---

## 11. Canonical preimages and domain separation

`🟡 medium` · *Determinism*

The fix is an explicit preimage: one domain tag, fixed field order, fixed-width big-endian integers, and a length prefix on anything variable. The tags are what stop a leaf being reinterpreted as an internal node — the Merkle second-preimage attack lesson 05 defends against.

**Steps:**

1. Write `preimage()` that emits a tag, a 20-byte address, and two big-endian `uint64`s.
2. Confirm changing any field moves the hash.
3. Build a forged "leaf" whose contents are exactly `left‖right` and confirm it does not collide with the node hash.
4. Learn the five rules at the bottom — they apply to everything you hash from here on.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// Domain separation tags. Every distinct kind of thing you hash gets its own
// prefix, so a digest of one kind can never be reinterpreted as another.
const (
	tagLeaf    byte = 0x00
	tagNode    byte = 0x01
	tagAccount byte = 0x02
)

type Account struct {
	Address [20]byte
	Nonce   uint64
	Balance uint64
}

// preimage builds the EXACT bytes that get hashed: fixed order, fixed widths,
// big-endian, no maps, no strings of variable length without a length prefix.
func (a Account) preimage() []byte {
	buf := make([]byte, 0, 1+20+8+8)
	buf = append(buf, tagAccount)
	buf = append(buf, a.Address[:]...)
	buf = binary.BigEndian.AppendUint64(buf, a.Nonce)
	buf = binary.BigEndian.AppendUint64(buf, a.Balance)
	return buf
}

func (a Account) Hash() []byte {
	sum := sha256.Sum256(a.preimage())
	return sum[:]
}

func hashLeaf(data []byte) []byte {
	sum := sha256.Sum256(append([]byte{tagLeaf}, data...))
	return sum[:]
}

func hashNode(l, r []byte) []byte {
	buf := append([]byte{tagNode}, l...)
	sum := sha256.Sum256(append(buf, r...))
	return sum[:]
}

func main() {
	var addr [20]byte
	copy(addr[:], []byte("\xf3\x9f\xd6\xe5\x1a\xad\x88\xf6\xf4\xce\x6a\xb8\x82\x72\x79\xcf\xff\xb9\x22\x66"))
	a := Account{Address: addr, Nonce: 3, Balance: 1_000_000}

	fmt.Printf("preimage (%d bytes)\n  %x\n", len(a.preimage()), a.preimage())
	fmt.Printf("  ^^ tag, then 20-byte address, then two big-endian uint64s\n")
	fmt.Printf("hash %s\n", hex.EncodeToString(a.Hash()))

	// Every field change moves the hash, and nothing else does.
	b := a
	b.Nonce = 4
	fmt.Printf("\nnonce 3 -> 4 changes the hash: %v\n",
		hex.EncodeToString(a.Hash()) != hex.EncodeToString(b.Hash()))

	// Why the tags matter. Without domain separation, a 64-byte leaf could be
	// reinterpreted as two 32-byte child hashes — the Merkle second-preimage
	// attack that lesson 05 defends against.
	left := hashLeaf([]byte("tx-a"))
	right := hashLeaf([]byte("tx-b"))
	parent := hashNode(left, right)

	// The attacker's forged "leaf" whose contents are exactly left||right.
	forged := append(append([]byte{}, left...), right...)
	fmt.Printf("\nnode hash        %s\n", hex.EncodeToString(parent))
	fmt.Printf("leaf(left||right) %s\n", hex.EncodeToString(hashLeaf(forged)))
	fmt.Printf("collide? %v   <- the 0x00/0x01 tags keep them apart\n",
		hex.EncodeToString(parent) == hex.EncodeToString(hashLeaf(forged)))

	fmt.Println("\nrules for hashing a struct:")
	fmt.Println("  1. one tag per message type      2. fixed field order")
	fmt.Println("  3. fixed-width, big-endian ints  4. length-prefix anything variable")
	fmt.Println("  5. never a map, never JSON, never gob")
}
```

**Output:**

```
preimage (37 bytes)
  02f39fd6e51aad88f6f4ce6ab8827279cfffb92266000000000000000300000000000f4240
  ^^ tag, then 20-byte address, then two big-endian uint64s
hash 82e42877358fe2e8a5b54f7c64b6404215a2115e4cc084a52b4fd57fb4bed9b3

nonce 3 -> 4 changes the hash: true

node hash        f847efba820f456048ec00c49c6bb214b15d78520d25c660c20d53ae9c3f7192
leaf(left||right) f5bbd0e998f6a913cf285bfc6b8b72a6e902533f05a40b454aed65b2eb716611
collide? false   <- the 0x00/0x01 tags keep them apart

rules for hashing a struct:
  1. one tag per message type      2. fixed field order
  3. fixed-width, big-endian ints  4. length-prefix anything variable
  5. never a map, never JSON, never gob
```

---

## 12. Commit and reveal

`🟡 medium` · *Commitments*

`H(value‖nonce)` lets you publish a promise now and prove it later. It needs two properties: **hiding**, so the commitment reveals nothing (that is what the nonce buys), and **binding**, so you cannot open it as something else (that is collision resistance).

**Steps:**

1. Commit to a sealed bid with a 32-byte nonce and a domain tag.
2. Verify an honest reveal with `subtle.ConstantTimeCompare`.
3. Try to reveal a different value, then try swapping the nonce — both fail.
4. Commit to three different bids and see that the digests reveal nothing about their order.

```go
package main

import (
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/hex"
	"fmt"
)

// Commit builds H(value || nonce). The nonce is what makes the commitment
// HIDING: without it, a guessable value is trivially recovered (example 13).
func Commit(value string, nonce []byte) []byte {
	h := sha256.New()
	h.Write([]byte{0x10}) // domain separation tag for "commitment"
	h.Write([]byte(value))
	h.Write(nonce)
	return h.Sum(nil)
}

// Verify checks a revealed (value, nonce) against a published commitment.
func Verify(commitment []byte, value string, nonce []byte) bool {
	return subtle.ConstantTimeCompare(commitment, Commit(value, nonce)) == 1
}

func main() {
	// In production this is crypto/rand. Fixed here so the output reproduces.
	nonce, _ := hex.DecodeString(
		"9f2c4a1e7b3d508614a9c0e2f7581d3b46e0a7c9128d5f3e6b04a1c8d2f9e7b5")

	// Proof that a real nonce is one line and 32 bytes of entropy.
	live := make([]byte, 32)
	if _, err := rand.Read(live); err != nil {
		fmt.Println("rand:", err)
		return
	}
	fmt.Printf("a fresh nonce is %d bytes from crypto/rand\n\n", len(live))

	// --- phase 1: commit ----------------------------------------------------
	const bid = "1200"
	c := Commit(bid, nonce)
	fmt.Println("phase 1 — publish the commitment (the bid stays secret)")
	fmt.Printf("  commitment %s\n", hex.EncodeToString(c))

	// --- phase 2: reveal ----------------------------------------------------
	fmt.Println("\nphase 2 — reveal the value and the nonce")
	fmt.Printf("  honest reveal (%q)      -> %v\n", bid, Verify(c, bid, nonce))

	// BINDING: you cannot reveal a different value for the same commitment.
	fmt.Printf("  lying about the bid (\"9999\") -> %v\n", Verify(c, "9999", nonce))

	// Nor can you swap the nonce to make a different value fit.
	otherNonce, _ := hex.DecodeString(
		"0000000000000000000000000000000000000000000000000000000000000001")
	fmt.Printf("  swapping the nonce           -> %v\n", Verify(c, bid, otherNonce))

	// HIDING: two different bids produce commitments that reveal nothing about
	// their relative size, or even whether they are close.
	fmt.Println("\nhiding — the commitment leaks nothing about the value")
	for _, v := range []string{"1200", "1201", "9999"} {
		fmt.Printf("  bid %-6s %s\n", v, hex.EncodeToString(Commit(v, nonce))[:32])
	}

	fmt.Println("\ntwo properties, both needed:")
	fmt.Println("  hiding  — the commitment reveals nothing (needs the random nonce)")
	fmt.Println("  binding — you cannot change your mind (needs collision resistance)")
}
```

**Output:**

```
a fresh nonce is 32 bytes from crypto/rand

phase 1 — publish the commitment (the bid stays secret)
  commitment 8fdb9072282e49d39b016559945fdc01669d321272b7896e9e68f347c8bf8283

phase 2 — reveal the value and the nonce
  honest reveal ("1200")      -> true
  lying about the bid ("9999") -> false
  swapping the nonce           -> false

hiding — the commitment leaks nothing about the value
  bid 1200   8fdb9072282e49d39b016559945fdc01
  bid 1201   3f529cf5b369dcd9867460efb4cb2f26
  bid 9999   979ec3a7f44b1c071f753de4892891df

two properties, both needed:
  hiding  — the commitment reveals nothing (needs the random nonce)
  binding — you cannot change your mind (needs collision resistance)
```

---

## 13. The nonce is not optional

`🟡 medium` · *Commitments*

Drop the nonce and the commitment stops hiding anything, because bids come from a small, publicly known set. This is example 6's lesson applied to a real design — and it also shows the flaw a nonce does *not* fix.

**Steps:**

1. Commit to an auction bid with no nonce.
2. Recover it by enumerating every whole-dollar bid under 100,000.
3. See how 32 bytes of entropy turns that from a loop into an impossibility.
4. Read the last-mover problem: whoever reveals last has seen everything and can walk away.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// A commitment with no nonce. The author thought "it is hashed, so it is secret".
func commitNoNonce(bid int) string {
	sum := sha256.Sum256([]byte(fmt.Sprintf("%d", bid)))
	return hex.EncodeToString(sum[:])
}

func main() {
	// A sealed-bid auction. Bids are whole dollars, somewhere under 100,000.
	// That is the entire search space, and it is tiny.
	secretBid := 42750
	published := commitNoNonce(secretBid)

	fmt.Println("sealed-bid auction, commitments published on-chain")
	fmt.Printf("  victim's commitment %s\n", published)
	fmt.Println("  bids are whole dollars below 100,000 — that is public knowledge")

	// Enumerate. No cryptography is broken; the input space is just small.
	tried := 0
	recovered := -1
	for b := 0; b < 100000; b++ {
		tried++
		if commitNoNonce(b) == published {
			recovered = b
			break
		}
	}
	fmt.Printf("\n  recovered bid: %d after %d guesses\n", recovered, tried)
	fmt.Println("  a rival now bids one dollar more, and wins")

	// Add 32 bytes of entropy and the same attack needs 2^256 work.
	fmt.Println("\nwith a 32-byte nonce")
	fmt.Printf("  search space goes from 10^5 to 10^5 x 2^256\n")
	fmt.Println("  the attack does not get harder — it becomes impossible")

	// The remaining weakness is not cryptographic: whoever reveals LAST sees
	// everyone else's values first and can simply refuse to open.
	fmt.Println("\nwhat a nonce does NOT fix — the last-mover problem:")
	fmt.Println("  the final revealer has seen every other bid and can walk away")
	fmt.Println("  real designs add a deposit forfeited on non-reveal, a reveal")
	fmt.Println("  deadline, or a VRF instead of commit-reveal (lesson 53)")
}
```

**Output:**

```
sealed-bid auction, commitments published on-chain
  victim's commitment 43ac3535f90e477226e919d8e6996468e0f9424f618f18163f7be4b589dd2945
  bids are whole dollars below 100,000 — that is public knowledge

  recovered bid: 42750 after 42751 guesses
  a rival now bids one dollar more, and wins

with a 32-byte nonce
  search space goes from 10^5 to 10^5 x 2^256
  the attack does not get harder — it becomes impossible

what a nonce does NOT fix — the last-mover problem:
  the final revealer has seen every other bid and can walk away
  real designs add a deposit forfeited on non-reveal, a reveal
  deadline, or a VRF instead of commit-reveal (lesson 53)
```

---

## 14. Truncated hashes and the birthday bound

`🟡 medium` · *Identifiers*

Truncating a hash is normal — Ethereum addresses are 20 bytes of a Keccak digest and ABI selectors are 4. But every truncation spends security: the birthday bound puts a collision at roughly 2^(n/2) work, so 4 bytes falls in about 65,000 tries.

**Steps:**

1. Compute the `transfer(address,uint256)` selector and check it equals the well-known `0xa9059cbb`.
2. Tabulate collision work against output size.
3. Brute-force an actual 32-bit collision and confirm the full digests still differ.
4. Conclude that selectors are identifiers, not security boundaries (lesson 46).

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// Truncating a hash is normal and useful: Ethereum addresses are the last
	// 20 bytes of a Keccak digest, and ABI function selectors are the first 4.
	sel := crypto.Keccak256([]byte("transfer(address,uint256)"))[:4]
	fmt.Printf("selector for transfer(address,uint256): %s\n", hex.EncodeToString(sel))
	fmt.Printf("matches the well-known value 0xa9059cbb: %v\n",
		hex.EncodeToString(sel) == "a9059cbb")

	// But every truncation spends security. The birthday bound says a collision
	// takes about 2^(n/2) work for an n-bit output.
	fmt.Println("\noutput size vs work to find ANY collision")
	for _, bits := range []int{32, 64, 128, 160, 256} {
		fmt.Printf("  %3d bits -> about 2^%-3d operations\n", bits, bits/2)
	}

	// 32 bits means about 2^16 = 65,536 tries. That is a loop, not an attack.
	fmt.Println("\ncolliding a 4-byte (32-bit) truncated hash by brute force:")
	seen := map[uint32]uint64{}
	var a, b uint64
	var collided uint32
	tries := 0
	for i := uint64(0); ; i++ {
		tries++
		var buf [8]byte
		binary.BigEndian.PutUint64(buf[:], i)
		sum := sha256.Sum256(buf[:])
		t := binary.BigEndian.Uint32(sum[:4])
		if prev, ok := seen[t]; ok {
			a, b, collided = prev, i, t
			break
		}
		seen[t] = i
	}
	fmt.Printf("  inputs %d and %d both truncate to %08x\n", a, b, collided)
	fmt.Printf("  found after %d hashes\n", tries)

	// Confirm it.
	var ba, bb [8]byte
	binary.BigEndian.PutUint64(ba[:], a)
	binary.BigEndian.PutUint64(bb[:], b)
	ha, hb := sha256.Sum256(ba[:]), sha256.Sum256(bb[:])
	fmt.Printf("  %s...\n  %s...\n", hex.EncodeToString(ha[:6]), hex.EncodeToString(hb[:6]))
	fmt.Printf("  full digests differ: %v (only the first 4 bytes match)\n", ha != hb)

	fmt.Println("\nso: 4-byte selectors are an identifier, NOT a security boundary.")
	fmt.Println("Selector collisions are real and have been used against proxies (lesson 46).")
	fmt.Println("160-bit addresses still need 2^80 work — fine, but no longer 2^128.")
}
```

**Output:**

```
selector for transfer(address,uint256): a9059cbb
matches the well-known value 0xa9059cbb: true

output size vs work to find ANY collision
   32 bits -> about 2^16  operations
   64 bits -> about 2^32  operations
  128 bits -> about 2^64  operations
  160 bits -> about 2^80  operations
  256 bits -> about 2^128 operations

colliding a 4-byte (32-bit) truncated hash by brute force:
  inputs 50799 and 58968 both truncate to 7357966a
  found after 58969 hashes
  7357966af283...
  7357966a93b4...
  full digests differ: true (only the first 4 bytes match)

so: 4-byte selectors are an identifier, NOT a security boundary.
Selector collisions are real and have been used against proxies (lesson 46).
160-bit addresses still need 2^80 work — fine, but no longer 2^128.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
