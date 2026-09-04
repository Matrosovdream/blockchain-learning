# Step 06 — Keys & Digital Signatures · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                                  # first time only
go get github.com/ethereum/go-ethereum@latest
go get github.com/btcsuite/btcd/btcec/v2@latest      # examples 2, 3, 6, 8, 11, 13-17
# paste the example into main.go, then:
go run .
```

No chain and no node. Every example uses the **published anvil test key**
`0xac0974be…ff80`, so all output is reproducible — and no real key is ever printed.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. A private key is one number

`🟢 easy` · *Keys*

A private key is a single 256-bit number. The public key is a point on a curve, derived from it. Everything else in this lesson is machinery around those two facts. The key used throughout is anvil's account-0 key — published in every Foundry tutorial, and never anything real.

**Steps:**

1. Load the test key and print it as bytes and as a decimal integer.
2. Print the public key's X and Y coordinates separately.
3. Serialize it uncompressed: `0x04 ‖ X ‖ Y`.
4. Derive the address, and round-trip the private key through its byte form.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// The anvil / hardhat account-0 key. PUBLIC TEST KEY — the private key is
	// published in every Foundry tutorial. Never a real one, not even here.
	const testKeyHex = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

	key, err := crypto.HexToECDSA(testKeyHex)
	if err != nil {
		fmt.Println("bad key:", err)
		return
	}

	// A private key is one number: a 256-bit scalar, and nothing else.
	fmt.Printf("private key (a scalar, 32 bytes)\n  %s\n",
		hex.EncodeToString(crypto.FromECDSA(key)))
	fmt.Printf("  as a decimal integer:\n  %s\n", key.D)

	// The public key is a POINT on the curve: an (x, y) pair, 32 bytes each.
	fmt.Printf("\npublic key (a point)\n  X %s\n  Y %s\n",
		hex.EncodeToString(key.PublicKey.X.Bytes()),
		hex.EncodeToString(key.PublicKey.Y.Bytes()))

	// Serialized uncompressed: 0x04 marker, then X, then Y.
	pub := crypto.FromECDSAPub(&key.PublicKey)
	fmt.Printf("\nuncompressed (%d bytes)\n  %s\n", len(pub), hex.EncodeToString(pub))
	fmt.Printf("  prefix %#02x means 'uncompressed, both coordinates follow'\n", pub[0])

	// And the address is derived from it — lesson 07 does this properly.
	fmt.Printf("\naddress %s\n", crypto.PubkeyToAddress(key.PublicKey).Hex())

	// Round-trip the private key through its byte form.
	back, err := crypto.ToECDSA(crypto.FromECDSA(key))
	fmt.Printf("\nround-trips through bytes: %v (err=%v)\n", back.D.Cmp(key.D) == 0, err)

	fmt.Println("\none secret number in; one public point, one address out.")
	fmt.Println("the whole of the next four lessons rests on that being one-way.")
}
```

**Output:**

```
private key (a scalar, 32 bytes)
  ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
  as a decimal integer:
  77814517325470205911140941194401928579557062014761831930645393041380819009408

public key (a point)
  X 8318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75
  Y 3547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5

uncompressed (65 bytes)
  048318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed753547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5
  prefix 0x04 means 'uncompressed, both coordinates follow'

address 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

round-trips through bytes: true (err=<nil>)

one secret number in; one public point, one address out.
the whole of the next four lessons rests on that being one-way.
```

---

## 2. The curve, and why d to P is one-way

`🟢 easy` · *Curves*

secp256k1 is the curve `y² = x³ + 7` over a 256-bit prime field. A private key is a scalar `d`; the public key is `d·G`. Multiplying is fast, reversing is infeasible — and that asymmetry is the entire security of every key in this course.

**Steps:**

1. Print the curve parameters: field prime, group order, generator.
2. Verify G satisfies the curve equation with `math/big`.
3. Compute `d·G` directly with `ScalarBaseMult` and confirm it matches the library's public key.
4. Check the key is in the valid range [1, n−1].

```go
package main

import (
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	curve := btcec.S256()
	p := curve.P // the prime field modulus
	n := curve.N // the order of the generator's group

	fmt.Println("secp256k1 parameters")
	fmt.Printf("  p (field prime) %x\n", p)
	fmt.Printf("  n (group order) %x\n", n)
	fmt.Printf("  Gx              %x\n", curve.Gx)
	fmt.Printf("  Gy              %x\n", curve.Gy)
	// The cofactor is 1 (SEC 2), which means every point on the curve is in
	// the generator's group — no small-subgroup checks needed, unlike some curves.
	fmt.Printf("  cofactor        %d (every curve point is in G's group)\n", 1)

	// The curve is y^2 = x^3 + 7 (mod p). Check the generator satisfies it.
	lhs := new(big.Int).Exp(curve.Gy, big.NewInt(2), p)
	rhs := new(big.Int).Exp(curve.Gx, big.NewInt(3), p)
	rhs.Add(rhs, big.NewInt(7))
	rhs.Mod(rhs, p)
	fmt.Printf("\nG satisfies y^2 = x^3 + 7 (mod p): %v\n", lhs.Cmp(rhs) == 0)

	// A public key is literally d*G — scalar multiplication of the generator.
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	x, y := curve.ScalarBaseMult(key.D.Bytes())

	fmt.Printf("\nd*G computed directly:\n  X %x\n  Y %x\n", x, y)
	fmt.Printf("matches the library's public key: %v\n",
		x.Cmp(key.PublicKey.X) == 0 && y.Cmp(key.PublicKey.Y) == 0)

	// The public key is on the curve too — every valid point is.
	lhs = new(big.Int).Exp(y, big.NewInt(2), p)
	rhs = new(big.Int).Exp(x, big.NewInt(3), p)
	rhs.Add(rhs, big.NewInt(7))
	rhs.Mod(rhs, p)
	fmt.Printf("public key is on the curve: %v\n", lhs.Cmp(rhs) == 0)

	// Multiplication one way is easy; going back is the discrete-log problem.
	fmt.Println("\nd -> d*G costs about 256 point doublings: microseconds.")
	fmt.Println("d*G -> d has no known shortcut: about 2^128 operations.")
	fmt.Println("that asymmetry is the entire security of every key in this course.")

	// A private key must be in [1, n-1]. Zero and n are not keys.
	fmt.Printf("\nvalid key range: 1 .. n-1\n")
	fmt.Printf("  n-1 = %x\n", new(big.Int).Sub(n, big.NewInt(1)))
	fmt.Printf("  our key is in range: %v\n",
		key.D.Sign() > 0 && key.D.Cmp(n) < 0)
}
```

**Output:**

```
secp256k1 parameters
  p (field prime) fffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f
  n (group order) fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
  Gx              79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798
  Gy              483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8
  cofactor        1 (every curve point is in G's group)

G satisfies y^2 = x^3 + 7 (mod p): true

d*G computed directly:
  X 8318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75
  Y 3547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5
matches the library's public key: true
public key is on the curve: true

d -> d*G costs about 256 point doublings: microseconds.
d*G -> d has no known shortcut: about 2^128 operations.
that asymmetry is the entire security of every key in this course.

valid key range: 1 .. n-1
  n-1 = fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364140
  our key is in range: true
```

---

## 3. Three curves, three schemes

`🟢 easy` · *Curves*

Go's standard library gives you the NIST curves; secp256k1 is not among them. That is why every Ethereum program imports go-ethereum or btcec — and why `elliptic.P256()` in chain code is a bug that compiles, runs, and produces keys no chain recognises.

**Steps:**

1. Compare secp256k1, P-256, ed25519 and BLS12-381 and where each is used.
2. Confirm secp256k1 and P-256 have different field primes.
3. Take the same 32-byte secret onto both curves and see two different public keys.
4. Note ed25519 is a different *scheme* — deterministic by construction, with no nonce to leak.

```go
package main

import (
	"crypto/ecdsa"
	"crypto/ed25519"
	"crypto/elliptic"
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// Go's crypto/elliptic gives you the NIST curves. secp256k1 is NOT among
	// them — that is why every Ethereum program imports go-ethereum or btcec.
	fmt.Println("curve                bits  where it is used")
	fmt.Printf("  %-18s %-5d %s\n", "secp256k1", btcec.S256().BitSize, "Bitcoin, Ethereum EOAs")
	fmt.Printf("  %-18s %-5d %s\n", "P-256 (secp256r1)", elliptic.P256().Params().BitSize,
		"TLS, passkeys/WebAuthn")
	fmt.Printf("  %-18s %-5d %s\n", "ed25519", 255, "Solana, Cosmos, SSH")
	fmt.Printf("  %-18s %-5d %s\n", "BLS12-381", 381, "Ethereum consensus (lesson 42)")

	// The two 256-bit curves are entirely different groups.
	fmt.Printf("\nsecp256k1 p %x\n", btcec.S256().P)
	fmt.Printf("P-256     p %x\n", elliptic.P256().Params().P)
	fmt.Printf("same curve? %v\n", btcec.S256().P.Cmp(elliptic.P256().Params().P) == 0)

	// THE TRAP: this compiles, runs, and produces a perfectly valid keypair
	// that no blockchain will ever recognise.
	wrong := &ecdsa.PrivateKey{
		PublicKey: ecdsa.PublicKey{Curve: elliptic.P256()},
		D:         new(big.Int).SetBytes(mustHex("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")),
	}
	wrong.PublicKey.X, wrong.PublicKey.Y = elliptic.P256().ScalarBaseMult(wrong.D.Bytes())

	right, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	fmt.Println("\nsame 32-byte secret, two curves:")
	fmt.Printf("  on secp256k1 X %s\n", hex.EncodeToString(right.PublicKey.X.Bytes())[:32]+"...")
	fmt.Printf("  on P-256     X %s\n", hex.EncodeToString(wrong.PublicKey.X.Bytes())[:32]+"...")
	fmt.Println("  completely different public keys, so completely different addresses")

	// ed25519 is a different SCHEME, not just a different curve.
	seed := make([]byte, ed25519.SeedSize) // fixed all-zero seed, for reproducibility
	edKey := ed25519.NewKeyFromSeed(seed)
	fmt.Printf("\ned25519 public key (%d bytes) %s\n",
		len(edKey.Public().(ed25519.PublicKey)),
		hex.EncodeToString(edKey.Public().(ed25519.PublicKey)))
	fmt.Println("  EdDSA, not ECDSA: deterministic by construction, no nonce to leak")
	fmt.Println("  (which makes examples 15 and 16 impossible on Solana — by design)")

	fmt.Println("\nrule: on Ethereum and Bitcoin, if your code says elliptic.P256()")
	fmt.Println("or crypto/ecdsa.GenerateKey, it is on the wrong curve.")
}

func mustHex(s string) []byte {
	b, err := hex.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return b
}
```

**Output:**

```
curve                bits  where it is used
  secp256k1          256   Bitcoin, Ethereum EOAs
  P-256 (secp256r1)  256   TLS, passkeys/WebAuthn
  ed25519            255   Solana, Cosmos, SSH
  BLS12-381          381   Ethereum consensus (lesson 42)

secp256k1 p fffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f
P-256     p ffffffff00000001000000000000000000000000ffffffffffffffffffffffff
same curve? false

same 32-byte secret, two curves:
  on secp256k1 X 8318535b54105d4a7aae60c08fc45f96...
  on P-256     X a43b66d1eaee03f07d64920491f8b348...
  completely different public keys, so completely different addresses

ed25519 public key (32 bytes) 3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29
  EdDSA, not ECDSA: deterministic by construction, no nonce to leak
  (which makes examples 15 and 16 impossible on Solana — by design)

rule: on Ethereum and Bitcoin, if your code says elliptic.P256()
or crypto/ecdsa.GenerateKey, it is on the wrong curve.
```

---

## 4. Sign and verify

`🟢 easy` · *Signing*

Sign a digest, get 65 bytes back: `r ‖ s ‖ v`. Two API sharp edges appear immediately — `VerifySignature` wants the 64-byte form and returns `false` (not an error) for 65, and go-ethereum's signing is deterministic, so the same message always gives identical bytes.

**Steps:**

1. Hash the message with Keccak-256 first — ECDSA takes a digest, never a message.
2. Sign, and split the result into r, s and v.
3. Verify with `sig[:64]`, then with all 65 bytes and watch it silently fail.
4. Sign twice and confirm the bytes are identical (example 16 explains why).

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	// ECDSA signs a 32-byte DIGEST, never a message. Hash first (lesson 04).
	message := []byte("transfer 100 tokens to bob")
	hash := crypto.Keccak256(message)
	fmt.Printf("message %q\n", message)
	fmt.Printf("keccak  %s (%d bytes)\n", hex.EncodeToString(hash), len(hash))

	// Sign. go-ethereum returns 65 bytes: r ‖ s ‖ v.
	sig, err := crypto.Sign(hash, key)
	if err != nil {
		fmt.Println("sign:", err)
		return
	}
	fmt.Printf("\nsignature (%d bytes)\n  %s\n", len(sig), hex.EncodeToString(sig))
	fmt.Printf("  r %s\n", hex.EncodeToString(sig[0:32]))
	fmt.Printf("  s %s\n", hex.EncodeToString(sig[32:64]))
	fmt.Printf("  v %d\n", sig[64])

	// Verify. Note VerifySignature wants the 64-byte r‖s, NOT the 65-byte form.
	pub := crypto.FromECDSAPub(&key.PublicKey)
	fmt.Printf("\nVerifySignature(pub, hash, sig[:64]): %v\n",
		crypto.VerifySignature(pub, hash, sig[:64]))

	// Passing all 65 bytes silently fails — it does not error, it returns false.
	fmt.Printf("passing the full 65 bytes:            %v  <- silently false\n",
		crypto.VerifySignature(pub, hash, sig))

	// go-ethereum's signing is RFC 6979 deterministic, so the same message and
	// key always produce byte-identical signatures (example 16).
	again, _ := crypto.Sign(hash, key)
	fmt.Printf("\nsigning twice gives identical bytes: %v\n",
		hex.EncodeToString(sig) == hex.EncodeToString(again))
}
```

**Output:**

```
message "transfer 100 tokens to bob"
keccak  7bc80568d872c65bddc70aa6f216569aa1542bca0d2c571d64c279805c358f9a (32 bytes)

signature (65 bytes)
  96865784536b1135c3b0f44acae4de867124ad2714d74b3b65024cb62b82d1a7469914610337fbaafc4f80007f8f85235affae57988a13144c75f6d6943ae92100
  r 96865784536b1135c3b0f44acae4de867124ad2714d74b3b65024cb62b82d1a7
  s 469914610337fbaafc4f80007f8f85235affae57988a13144c75f6d6943ae921
  v 0

VerifySignature(pub, hash, sig[:64]): true
passing the full 65 bytes:            false  <- silently false

signing twice gives identical bytes: true
```

---

## 5. Change anything, and it fails

`🟢 easy` · *Signing*

A signature binds one message to one key. Change the message, the signature, or the key you check against, and verification fails. What it does *not* tell you is when it was signed, why, or whether the signer understood it — those remain your problem.

**Steps:**

1. Verify an honest signature.
2. Change `bob` to `eve` and watch the hash change completely (lesson 04's avalanche).
3. Flip one bit of the signature itself.
4. Verify against a different public key.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	pub := crypto.FromECDSAPub(&key.PublicKey)

	message := []byte("send 100 to bob")
	hash := crypto.Keccak256(message)
	sig, _ := crypto.Sign(hash, key)

	fmt.Printf("original %q\n", message)
	fmt.Printf("verifies: %v\n", crypto.VerifySignature(pub, hash, sig[:64]))

	// Change one character: "bob" -> "eve".
	tampered := []byte("send 100 to eve")
	tamperedHash := crypto.Keccak256(tampered)
	fmt.Printf("\ntampered %q\n", tampered)
	fmt.Printf("hash changes completely (lesson 04's avalanche):\n")
	fmt.Printf("  %s\n  %s\n", hex.EncodeToString(hash[:16]), hex.EncodeToString(tamperedHash[:16]))
	fmt.Printf("same signature against the new hash: %v\n",
		crypto.VerifySignature(pub, tamperedHash, sig[:64]))

	// Change one bit of the signature instead.
	bad := append([]byte(nil), sig...)
	bad[10] ^= 0x01
	fmt.Printf("\nsignature with one bit flipped: %v\n",
		crypto.VerifySignature(pub, hash, bad[:64]))

	// Or verify against somebody else's public key.
	other, _ := crypto.HexToECDSA("59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d")
	fmt.Printf("verified against a different key: %v\n",
		crypto.VerifySignature(crypto.FromECDSAPub(&other.PublicKey), hash, sig[:64]))

	fmt.Println("\na signature binds ONE message to ONE key. Change either and it fails.")
	fmt.Println("what it does not tell you: when it was signed, why, or whether the")
	fmt.Println("signer understood what they were signing. Those are your problem.")
}
```

**Output:**

```
original "send 100 to bob"
verifies: true

tampered "send 100 to eve"
hash changes completely (lesson 04's avalanche):
  e79e6310b9e2d1ba6ac845a4d9a362ab
  20eace8d20acc93c3af7e79a5096fa92
same signature against the new hash: false

signature with one bit flipped: false
verified against a different key: false

a signature binds ONE message to ONE key. Change either and it fails.
what it does not tell you: when it was signed, why, or whether the
signer understood what they were signing. Those are your problem.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
