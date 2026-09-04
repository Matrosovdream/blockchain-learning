# Step 06 — Keys & Digital Signatures · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. Malleability: r and n minus s

`🔴 hard` · *Malleability*

If `(r, s)` is a valid signature then so is `(r, n−s)` — the verification equation is symmetric in s. This example proves it with a hand-written verifier, then shows go-ethereum rejecting the flipped form because EIP-2 requires low-s. Note that `Ecrecover` accepts both.

**Steps:**

1. Implement the raw ECDSA verification equation with no policy checks.
2. Confirm the raw equation accepts both `(r, s)` and `(r, n−s)`.
3. Watch `Ecrecover` accept both and recover the same address.
4. Watch `VerifySignature` reject the flipped one, and check s against n/2 to see why.
5. Take away the rule: never key anything on signature bytes.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

var n = btcec.S256().N

// rawVerify is the textbook ECDSA verification equation, with no low-s check.
// This is what the mathematics says; the extra rule is a policy layer on top.
//
//	w  = s^-1 mod n
//	u1 = z*w mod n      u2 = r*w mod n
//	(x, _) = u1*G + u2*P
//	valid iff r == x mod n
func rawVerify(px, py, r, s, z *big.Int) bool {
	if r.Sign() <= 0 || r.Cmp(n) >= 0 || s.Sign() <= 0 || s.Cmp(n) >= 0 {
		return false
	}
	curve := btcec.S256()
	w := new(big.Int).ModInverse(s, n)
	u1 := new(big.Int).Mod(new(big.Int).Mul(z, w), n)
	u2 := new(big.Int).Mod(new(big.Int).Mul(r, w), n)

	x1, y1 := curve.ScalarBaseMult(u1.Bytes())
	x2, y2 := curve.ScalarMult(px, py, u2.Bytes())
	x, _ := curve.Add(x1, y1, x2, y2)

	return new(big.Int).Mod(x, n).Cmp(r) == 0
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	pub := crypto.FromECDSAPub(&key.PublicKey)

	hashBytes := crypto.Keccak256([]byte("transfer 100 to bob"))
	z := new(big.Int).SetBytes(hashBytes)
	sig, _ := crypto.Sign(hashBytes, key)

	r := new(big.Int).SetBytes(sig[0:32])
	s := new(big.Int).SetBytes(sig[32:64])
	sFlipped := new(big.Int).Sub(n, s)

	fmt.Printf("r        %s\n", r.Text(16))
	fmt.Printf("s        %s\n", s.Text(16))
	fmt.Printf("n - s    %s\n", sFlipped.Text(16))

	// The mathematics accepts BOTH. The verification equation is symmetric in s.
	fmt.Println("\nagainst the raw ECDSA equation (no policy checks):")
	fmt.Printf("  (r, s)     %v\n", rawVerify(key.PublicKey.X, key.PublicKey.Y, r, s, z))
	fmt.Printf("  (r, n-s)   %v   <- a second valid signature, same key, same message\n",
		rawVerify(key.PublicKey.X, key.PublicKey.Y, r, sFlipped, z))

	// Build the flipped signature in wire form. The recovery bit flips with s.
	malleable := make([]byte, 65)
	copy(malleable[0:32], sig[0:32])
	sFlipped.FillBytes(malleable[32:64])
	malleable[64] = sig[64] ^ 1

	fmt.Printf("\ntwo different byte sequences:\n  %s\n  %s\n",
		hex.EncodeToString(sig), hex.EncodeToString(malleable))

	// Recovery accepts both and returns the SAME signer.
	fmt.Println("\ncrypto.Ecrecover accepts both and recovers the same address:")
	for _, c := range []struct {
		name string
		sg   []byte
	}{{"(r, s)  ", sig}, {"(r, n-s)", malleable}} {
		if p, err := crypto.SigToPub(hashBytes, c.sg); err == nil {
			fmt.Printf("  %s %s\n", c.name, crypto.PubkeyToAddress(*p).Hex())
		} else {
			fmt.Printf("  %s error: %v\n", c.name, err)
		}
	}

	// But crypto.VerifySignature enforces EIP-2's low-s rule and refuses.
	fmt.Println("\ncrypto.VerifySignature enforces low-s and rejects the flipped one:")
	fmt.Printf("  (r, s)     %v\n", crypto.VerifySignature(pub, hashBytes, sig[:64]))
	fmt.Printf("  (r, n-s)   %v\n", crypto.VerifySignature(pub, hashBytes, malleable[:64]))

	halfN := new(big.Int).Rsh(n, 1)
	fmt.Printf("\n  n/2      %s\n", halfN.Text(16))
	fmt.Printf("  s   low  %v\n", s.Cmp(halfN) <= 0)
	fmt.Printf("  n-s low  %v\n", sFlipped.Cmp(halfN) <= 0)

	// Why it mattered.
	fmt.Println("\nwhy this was a real problem")
	fmt.Println("  a Bitcoin txid used to cover the signatures, so anyone relaying a")
	fmt.Println("  transaction could flip s and change its id. Services tracking")
	fmt.Println("  withdrawals by txid saw them 'vanish' and re-sent — a real loss.")
	fmt.Println("  BIP-62/BIP-146 made low-s a rule; SegWit removed signatures from")
	fmt.Println("  the txid preimage entirely (lesson 36). EIP-2 does the same on")
	fmt.Println("  Ethereum, which is why VerifySignature said false above.")

	fmt.Println("\nthe rule that outlives the fixes:")
	fmt.Println("  never key anything on signature bytes. Key on the hash of the")
	fmt.Println("  signed payload — no third party can change that.")
}
```

**Output:**

```
r        e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e
s        5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb40
n - s    a13122b043db5314142db97f9d15c79b1a093c34d2e606007c5896620f0a6601

against the raw ECDSA equation (no policy checks):
  (r, s)     true
  (r, n-s)   true   <- a second valid signature, same key, same message

two different byte sequences:
  e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb4000
  e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15ea13122b043db5314142db97f9d15c79b1a093c34d2e606007c5896620f0a660101

crypto.Ecrecover accepts both and recovers the same address:
  (r, s)   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
  (r, n-s) 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

crypto.VerifySignature enforces low-s and rejects the flipped one:
  (r, s)     true
  (r, n-s)   false

  n/2      7fffffffffffffffffffffffffffffff5d576e7357a4501ddfe92f46681b20a0
  s   low  true
  n-s low  false

why this was a real problem
  a Bitcoin txid used to cover the signatures, so anyone relaying a
  transaction could flip s and change its id. Services tracking
  withdrawals by txid saw them 'vanish' and re-sent — a real loss.
  BIP-62/BIP-146 made low-s a rule; SegWit removed signatures from
  the txid preimage entirely (lesson 36). EIP-2 does the same on
  Ethereum, which is why VerifySignature said false above.

the rule that outlives the fixes:
  never key anything on signature bytes. Key on the hash of the
  signed payload — no third party can change that.
```

---

## 15. Nonce reuse hands over the private key

`🔴 hard` · *Nonces*

The catastrophe. Sign two different messages with the same nonce and the private key falls out in four modular operations. This example does it — recovering the exact key it started with. No curve is broken; the algebra simply gives it away.

**Steps:**

1. Write a raw signer that takes a caller-supplied nonce (no real library allows this).
2. Sign two different messages with the same k, and notice r₁ == r₂ — the reuse is public.
3. Recover k from `(z₁ − z₂)/(s₁ − s₂)`, then d from `(s₁·k − z₁)/r`.
4. Confirm the recovered key equals the original.
5. Read the real incidents: PlayStation 3 in 2010, Android's SecureRandom in 2013.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

var n = btcec.S256().N

// signWithK is raw ECDSA with a CALLER-SUPPLIED nonce. No real library lets
// you do this, for the reason this example is about.
func signWithK(d, k, z *big.Int) (r, s *big.Int) {
	x, _ := btcec.S256().ScalarBaseMult(k.Bytes())
	r = new(big.Int).Mod(x, n)

	kInv := new(big.Int).ModInverse(k, n)
	s = new(big.Int).Mul(r, d)
	s.Add(s, z)
	s.Mul(s, kInv)
	s.Mod(s, n)
	return r, s
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	d := key.D

	// The victim signs two DIFFERENT messages and reuses the same nonce.
	k, _ := new(big.Int).SetString(
		"7b1c2f4e9a3d5068b1e4c7a09f2d6538e0b4a7c1d9f36285a4e0c7b1d5f39268", 16)

	z1 := new(big.Int).SetBytes(crypto.Keccak256([]byte("transfer 100 to bob")))
	z2 := new(big.Int).SetBytes(crypto.Keccak256([]byte("transfer 250 to carol")))

	r1, s1 := signWithK(d, k, z1)
	r2, s2 := signWithK(d, k, z2)

	fmt.Println("two signatures, same nonce k")
	fmt.Printf("  sig 1  r %s\n         s %s\n", r1.Text(16), s1.Text(16))
	fmt.Printf("  sig 2  r %s\n         s %s\n", r2.Text(16), s2.Text(16))
	fmt.Printf("\nnotice r1 == r2: %v\n", r1.Cmp(r2) == 0)
	fmt.Println("  r depends only on k, so a repeated r ANNOUNCES the reuse publicly.")

	// ---- the attack, from public data only ---------------------------------
	// s1 = k^-1 (z1 + r*d)  and  s2 = k^-1 (z2 + r*d)
	// s1 - s2 = k^-1 (z1 - z2)          =>  k = (z1 - z2) / (s1 - s2)
	// then  d = (s1*k - z1) / r
	fmt.Println("\nthe attacker has: r, s1, s2, z1, z2 — all public.")

	dz := new(big.Int).Sub(z1, z2)
	ds := new(big.Int).Sub(s1, s2)
	ds.Mod(ds, n)
	recoveredK := new(big.Int).Mul(dz, new(big.Int).ModInverse(ds, n))
	recoveredK.Mod(recoveredK, n)

	fmt.Printf("\n  k = (z1 - z2) / (s1 - s2) mod n\n    = %s\n", recoveredK.Text(16))
	fmt.Printf("  correct: %v\n", recoveredK.Cmp(k) == 0)

	recoveredD := new(big.Int).Mul(s1, recoveredK)
	recoveredD.Sub(recoveredD, z1)
	recoveredD.Mul(recoveredD, new(big.Int).ModInverse(r1, n))
	recoveredD.Mod(recoveredD, n)

	fmt.Printf("\n  d = (s1*k - z1) / r mod n\n    = %s\n", recoveredD.Text(16))
	fmt.Printf("  correct: %v\n", recoveredD.Cmp(d) == 0)

	fmt.Printf("\nPRIVATE KEY RECOVERED: %s\n", hex.EncodeToString(recoveredD.Bytes()))
	fmt.Printf("address now controlled: %s\n", crypto.PubkeyToAddress(key.PublicKey).Hex())
	fmt.Println("\nthat is four modular operations. No curve was broken.")

	fmt.Println("\nwhen this has happened for real")
	fmt.Println("  2010  Sony PlayStation 3: the ECDSA nonce was a constant. The code-")
	fmt.Println("        signing key was recovered and the console was fully unlocked.")
	fmt.Println("  2013  Android's SecureRandom was broken; Bitcoin wallets on affected")
	fmt.Println("        phones produced repeated nonces and were emptied.")
	fmt.Println("  ongoing  scanners watch every chain for repeated r values.")

	fmt.Println("\nand it is worse than 'do not repeat k': even a BIASED k leaks.")
	fmt.Println("lattice attacks recover a key from a few hundred signatures whose")
	fmt.Println("nonces share as little as a handful of known bits.")
	fmt.Println("\nthe fix is example 16: derive k deterministically from d and z.")
}
```

**Output:**

```
two signatures, same nonce k
  sig 1  r e0ed85050818fcee30177e93f5e4ccd0c5779ca9235f542c63d92ce6b3601d69
         s 646df943896fc457f402505bb79b221680e2a5a22d72e76175319037c36f53fd
  sig 2  r e0ed85050818fcee30177e93f5e4ccd0c5779ca9235f542c63d92ce6b3601d69
         s 5a499f60dfc827d1540dff9adb48ffaca92873850414f642778bebe1fa014787

notice r1 == r2: true
  r depends only on k, so a repeated r ANNOUNCES the reuse publicly.

the attacker has: r, s1, s2, z1, z2 — all public.

  k = (z1 - z2) / (s1 - s2) mod n
    = 7b1c2f4e9a3d5068b1e4c7a09f2d6538e0b4a7c1d9f36285a4e0c7b1d5f39268
  correct: true

  d = (s1*k - z1) / r mod n
    = ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
  correct: true

PRIVATE KEY RECOVERED: ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
address now controlled: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

that is four modular operations. No curve was broken.

when this has happened for real
  2010  Sony PlayStation 3: the ECDSA nonce was a constant. The code-
        signing key was recovered and the console was fully unlocked.
  2013  Android's SecureRandom was broken; Bitcoin wallets on affected
        phones produced repeated nonces and were emptied.
  ongoing  scanners watch every chain for repeated r values.

and it is worse than 'do not repeat k': even a BIASED k leaks.
lattice attacks recover a key from a few hundred signatures whose
nonces share as little as a handful of known bits.

the fix is example 16: derive k deterministically from d and z.
```

---

## 16. RFC 6979: deriving k from the key

`🔴 hard` · *Nonces*

The fix: derive the nonce deterministically from the private key and the message hash with HMAC, so there is no RNG in the signing path at all. This implements RFC 6979 from scratch — and lands on the exact nonce go-ethereum uses internally, which is how you know it is right.

**Steps:**

1. Implement the HMAC-SHA256 derivation: initialise V and K, mix in the key and hash, then iterate.
2. Confirm the same inputs always give the same k, and a different message gives a different one.
3. Compute r from your k and compare it with `crypto.Sign`'s r — they are identical.
4. Understand what this buys, and what it does not (a weak key is still weak).

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

var n = btcec.S256().N

// rfc6979k derives the ECDSA nonce deterministically from the private key and
// the message hash, using HMAC-SHA256. No RNG is involved at signing time, so
// the failure mode in example 15 becomes structurally impossible.
func rfc6979k(x *big.Int, h1 []byte) *big.Int {
	hlen := sha256.Size

	// int2octets(x) and bits2octets(h1), both 32 bytes for secp256k1.
	xb := make([]byte, 32)
	x.FillBytes(xb)
	z1 := new(big.Int).SetBytes(h1)
	if z1.Cmp(n) >= 0 {
		z1.Sub(z1, n)
	}
	hb := make([]byte, 32)
	z1.FillBytes(hb)

	mac := func(key, data []byte) []byte {
		m := hmac.New(sha256.New, key)
		m.Write(data)
		return m.Sum(nil)
	}

	v := make([]byte, hlen)
	for i := range v {
		v[i] = 0x01
	}
	k := make([]byte, hlen) // all zeros

	k = mac(k, concat(v, []byte{0x00}, xb, hb))
	v = mac(k, v)
	k = mac(k, concat(v, []byte{0x01}, xb, hb))
	v = mac(k, v)

	for {
		v = mac(k, v)
		cand := new(big.Int).SetBytes(v)
		if cand.Sign() > 0 && cand.Cmp(n) < 0 {
			return cand
		}
		k = mac(k, concat(v, []byte{0x00}))
		v = mac(k, v)
	}
}

func concat(parts ...[]byte) []byte {
	var out []byte
	for _, p := range parts {
		out = append(out, p...)
	}
	return out
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	hash := crypto.Keccak256([]byte("transfer 100 to bob"))

	// Derive k the RFC 6979 way.
	k := rfc6979k(key.D, hash)
	fmt.Printf("k = HMAC-SHA256(private key, message hash) ...\n  %s\n", k.Text(16))

	// Same inputs always give the same k.
	k2 := rfc6979k(key.D, hash)
	fmt.Printf("\nderiving it again gives the same k: %v\n", k.Cmp(k2) == 0)

	// A different message gives a completely different k.
	other := rfc6979k(key.D, crypto.Keccak256([]byte("transfer 250 to carol")))
	fmt.Printf("a different message gives a different k: %v\n", k.Cmp(other) != 0)
	fmt.Printf("  %s\n", other.Text(16))

	// Compute r from our k, and compare with what go-ethereum produced.
	x, _ := btcec.S256().ScalarBaseMult(k.Bytes())
	ourR := new(big.Int).Mod(x, n)

	sig, _ := crypto.Sign(hash, key)
	gethR := new(big.Int).SetBytes(sig[0:32])

	fmt.Printf("\nr from our RFC 6979 k   %s\n", ourR.Text(16))
	fmt.Printf("r from crypto.Sign      %s\n", gethR.Text(16))
	fmt.Printf("identical: %v\n", ourR.Cmp(gethR) == 0)
	fmt.Println("  go-ethereum already signs with RFC 6979 — this reimplements what")
	fmt.Println("  it does internally, and lands on the same nonce.")

	// So repeated signing is byte-identical, which is worth knowing.
	a, _ := crypto.Sign(hash, key)
	b, _ := crypto.Sign(hash, key)
	fmt.Printf("\nsigning the same message twice:\n  %s\n  %s\n  identical: %v\n",
		hex.EncodeToString(a)[:40]+"...", hex.EncodeToString(b)[:40]+"...",
		hex.EncodeToString(a) == hex.EncodeToString(b))

	fmt.Println("\nwhat this buys")
	fmt.Println("  no RNG in the signing path, so a broken RNG cannot leak the key")
	fmt.Println("  signatures are reproducible, which makes them testable")
	fmt.Println("  hardware wallets can be audited: same input, same output, always")

	fmt.Println("\nwhat it does NOT buy")
	fmt.Println("  nothing about the KEY's entropy — a weak key is still weak (example 12)")
	fmt.Println("  no protection against fault injection: glitch the chip mid-signature")
	fmt.Println("  and you can still get two signatures with a related nonce")
	fmt.Println("\nRFC 6979 is standard on Bitcoin and Ethereum. ed25519 bakes the same")
	fmt.Println("idea into the scheme itself (example 3).")
}
```

**Output:**

```
k = HMAC-SHA256(private key, message hash) ...
  fbc3afe5cc8466089f3a136fb3042f55f7911d0ba325be2613859dab521170f9

deriving it again gives the same k: true
a different message gives a different k: true
  c47b6c2f88e36c1ff62ca37115146d49f97a97b203c215d9b190641100222bd8

r from our RFC 6979 k   e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e
r from crypto.Sign      e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e
identical: true
  go-ethereum already signs with RFC 6979 — this reimplements what
  it does internally, and lands on the same nonce.

signing the same message twice:
  e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2...
  e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2...
  identical: true

what this buys
  no RNG in the signing path, so a broken RNG cannot leak the key
  signatures are reproducible, which makes them testable
  hardware wallets can be audited: same input, same output, always

what it does NOT buy
  nothing about the KEY's entropy — a weak key is still weak (example 12)
  no protection against fault injection: glitch the chip mid-signature
  and you can still get two signatures with a related nonce

RFC 6979 is standard on Bitcoin and Ethereum. ed25519 bakes the same
idea into the scheme itself (example 3).
```

---

## 17. Schnorr, and what linearity buys

`🔴 hard` · *Beyond ECDSA*

Schnorr signs with `s = k + H(R‖P‖m)·d` — the private key appears **linearly**, where ECDSA buries it under a modular inverse. That one structural difference is why Schnorr signatures and keys can be added together, and ECDSA's cannot.

**Steps:**

1. Sign the same hash with both schemes and compare sizes: 64 bytes versus 65.
2. Look at the x-only public key — BIP-340 drops Y entirely.
3. Compare the two signing equations and see where linearity comes from.
4. Read what aggregation buys, and why naive aggregation is broken by the rogue-key attack.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/btcsuite/btcd/btcec/v2/schnorr"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	keyBytes, _ := hex.DecodeString("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	btcKey, _ := btcec.PrivKeyFromBytes(keyBytes)
	ethKey, _ := crypto.ToECDSA(keyBytes)

	// BIP-340 signs a 32-byte message; use the same hash as the ECDSA examples.
	msg := crypto.Keccak256([]byte("transfer 100 to bob"))

	ecdsaSig, _ := crypto.Sign(msg, ethKey)
	schnorrSig, err := schnorr.Sign(btcKey, msg)
	if err != nil {
		fmt.Println("schnorr sign:", err)
		return
	}
	sSer := schnorrSig.Serialize()

	fmt.Printf("ECDSA   (%d bytes) %s\n", len(ecdsaSig), hex.EncodeToString(ecdsaSig))
	fmt.Printf("Schnorr (%d bytes) %s\n", len(sSer), hex.EncodeToString(sSer))
	fmt.Println("  Schnorr is r ‖ s with no recovery byte — 64 bytes, always")

	// Public keys are x-only: 32 bytes, with an implicit even Y.
	xonly := schnorr.SerializePubKey(btcKey.PubKey())
	fmt.Printf("\nx-only pubkey (%d bytes) %s\n", len(xonly), hex.EncodeToString(xonly))
	fmt.Printf("ECDSA uncompressed      (%d bytes)\n", len(crypto.FromECDSAPub(&ethKey.PublicKey)))
	fmt.Println("  BIP-340 drops the Y coordinate entirely: even Y is assumed,")
	fmt.Println("  and a key with odd Y is negated to make it so.")

	fmt.Printf("\nverifies: %v\n", schnorrSig.Verify(msg, btcKey.PubKey()))
	tampered := crypto.Keccak256([]byte("transfer 100 to eve"))
	fmt.Printf("against a tampered message: %v\n", schnorrSig.Verify(tampered, btcKey.PubKey()))

	// THE STRUCTURAL DIFFERENCE.
	fmt.Println("\nECDSA:    s = k^-1 * (z + r*d)      <- d is multiplied by r, then inverted")
	fmt.Println("Schnorr:  s = k + H(R‖P‖m) * d      <- d appears LINEARLY")
	fmt.Println()
	fmt.Println("that linearity is the whole point. Add two Schnorr signatures and")
	fmt.Println("you get a valid signature for the sum of the two keys:")
	fmt.Println("  s1 + s2 = (k1 + k2) + H(...)*(d1 + d2)")
	fmt.Println()
	fmt.Println("ECDSA cannot do that — the k^-1 term does not distribute.")

	fmt.Println("\nwhat aggregation buys")
	fmt.Println("  an n-of-n multisig looks like a single-key spend on-chain")
	fmt.Println("  one signature and one key instead of n of each: cheaper and private")
	fmt.Println("  batch verification of many signatures at once")

	fmt.Println("\nthe catch — naive aggregation is BROKEN")
	fmt.Println("  if I choose my key AFTER seeing yours, I can pick P2 = X - P1 and")
	fmt.Println("  sign for the aggregate alone. That is the rogue-key attack.")
	fmt.Println("  MuSig2 fixes it with key-aggregation coefficients (lesson 42).")

	fmt.Println("\nwhere Schnorr is used: Bitcoin Taproot key-path spends (BIP-340,")
	fmt.Println("lesson 36). Ethereum EOAs are still ECDSA; its consensus layer uses")
	fmt.Println("BLS, which aggregates non-interactively (lesson 42).")
}
```

**Output:**

```
ECDSA   (65 bytes) e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb4000
Schnorr (64 bytes) a7642b090a85a4099208e9520c5cb0fcc73eb03455f3cb386a74a59dd526553f2d40f7d98d10bb99e578807efb6e585ed6158fadd609173a9e83a8e9cfabca8d
  Schnorr is r ‖ s with no recovery byte — 64 bytes, always

x-only pubkey (32 bytes) 8318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75
ECDSA uncompressed      (65 bytes)
  BIP-340 drops the Y coordinate entirely: even Y is assumed,
  and a key with odd Y is negated to make it so.

verifies: true
against a tampered message: false

ECDSA:    s = k^-1 * (z + r*d)      <- d is multiplied by r, then inverted
Schnorr:  s = k + H(R‖P‖m) * d      <- d appears LINEARLY

that linearity is the whole point. Add two Schnorr signatures and
you get a valid signature for the sum of the two keys:
  s1 + s2 = (k1 + k2) + H(...)*(d1 + d2)

ECDSA cannot do that — the k^-1 term does not distribute.

what aggregation buys
  an n-of-n multisig looks like a single-key spend on-chain
  one signature and one key instead of n of each: cheaper and private
  batch verification of many signatures at once

the catch — naive aggregation is BROKEN
  if I choose my key AFTER seeing yours, I can pick P2 = X - P1 and
  sign for the aggregate alone. That is the rogue-key attack.
  MuSig2 fixes it with key-aggregation coefficients (lesson 42).

where Schnorr is used: Bitcoin Taproot key-path spends (BIP-340,
lesson 36). Ethereum EOAs are still ECDSA; its consensus layer uses
BLS, which aggregates non-interactively (lesson 42).
```

---

## 18. A complete signature verifier

`🔴 hard` · *In production*

Everything assembled: the function a real service runs on every inbound signature. Length, recovery-id normalisation, low-s, EIP-191 hashing, recover-then-compare — plus a server-issued nonce and expiry, because without those one signature is a permanent password.

**Steps:**

1. Order the checks cheapest first: shape, then v, then s, then the curve operation last.
2. Normalise v from 27/28, reject high-s, and hash with the EIP-191 prefix.
3. Recover and compare against the expected address — never trust recovery alone.
4. Bind the message to a server-issued nonce with a TTL, and reject replays.
5. Exercise every rejection path, including a correctly signed but expired challenge.

```go
package main

import (
	"errors"
	"fmt"
	"math/big"
	"strings"
	"time"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/accounts"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// Everything this lesson taught, assembled into the function a real service
// runs on every inbound signature (lesson 50 turns it into a login flow).

var (
	ErrBadLength  = errors.New("signature must be 65 bytes")
	ErrBadV       = errors.New("invalid recovery id")
	ErrHighS      = errors.New("signature has high s (EIP-2)")
	ErrRecover    = errors.New("could not recover a public key")
	ErrWrongSaddr = errors.New("recovered a different address")
	ErrExpired    = errors.New("challenge expired")
	ErrReplay     = errors.New("challenge already used")
)

var halfN = new(big.Int).Rsh(btcec.S256().N, 1)

// Verify performs every check in order, cheapest first.
func Verify(message string, sig []byte, expected common.Address) error {
	// 1. Shape.
	if len(sig) != 65 {
		return ErrBadLength
	}

	// 2. Normalise v. Wallets return 27/28; crypto wants 0/1.
	s := append([]byte(nil), sig...)
	if s[64] >= 27 {
		s[64] -= 27
	}
	if s[64] > 1 {
		return ErrBadV
	}

	// 3. Reject high-s. Ecrecover would happily accept it, giving a second
	//    valid signature for the same message (example 14).
	if new(big.Int).SetBytes(s[32:64]).Cmp(halfN) > 0 {
		return ErrHighS
	}

	// 4. Hash with the EIP-191 prefix, so this digest can never be a
	//    transaction (example 7).
	digest := accounts.TextHash([]byte(message))

	// 5. Recover, then COMPARE. Recovery alone is not verification (example 9).
	pub, err := crypto.SigToPub(digest, s)
	if err != nil {
		return ErrRecover
	}
	if crypto.PubkeyToAddress(*pub) != expected {
		return ErrWrongSaddr
	}
	return nil
}

// A challenge binds the signature to one session: a nonce the server issued,
// and an expiry. Without these, one signature is a permanent password.
type Challenge struct {
	Nonce   string
	Expires time.Time
}

type Server struct {
	issued map[string]Challenge
	used   map[string]bool
}

func NewServer() *Server {
	return &Server{issued: map[string]Challenge{}, used: map[string]bool{}}
}

func (s *Server) Issue(nonce string, ttl time.Duration, now time.Time) string {
	s.issued[nonce] = Challenge{Nonce: nonce, Expires: now.Add(ttl)}
	return fmt.Sprintf("example.com wants you to sign in.\nNonce: %s", nonce)
}

func (s *Server) Login(message string, sig []byte, claimed common.Address, now time.Time) error {
	nonce := strings.TrimPrefix(message[strings.LastIndex(message, "\n")+1:], "Nonce: ")
	ch, ok := s.issued[nonce]
	if !ok {
		return ErrReplay
	}
	if s.used[nonce] {
		return ErrReplay
	}
	if now.After(ch.Expires) {
		return ErrExpired
	}
	if err := Verify(message, sig, claimed); err != nil {
		return err
	}
	s.used[nonce] = true
	return nil
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	addr := crypto.PubkeyToAddress(key.PublicKey)
	now := time.Unix(1700000000, 0)

	srv := NewServer()
	msg := srv.Issue("a1b2c3d4e5f6", 5*time.Minute, now)
	fmt.Printf("challenge issued:\n  %q\n\n", msg)

	sig, _ := crypto.Sign(accounts.TextHash([]byte(msg)), key)

	fmt.Printf("%-38s %v\n", "honest login", srv.Login(msg, sig, addr, now))
	fmt.Printf("%-38s %v\n", "same signature replayed", srv.Login(msg, sig, addr, now))

	// Every individual check, exercised.
	fmt.Println("\nthe checks in Verify:")
	fmt.Printf("  %-34s %v\n", "wrong length", Verify(msg, sig[:64], addr))

	badV := append([]byte(nil), sig...)
	badV[64] = 9
	fmt.Printf("  %-34s %v\n", "invalid recovery id", Verify(msg, badV, addr))

	high := append([]byte(nil), sig...)
	sVal := new(big.Int).SetBytes(sig[32:64])
	new(big.Int).Sub(btcec.S256().N, sVal).FillBytes(high[32:64])
	high[64] ^= 1
	fmt.Printf("  %-34s %v\n", "high s (malleated)", Verify(msg, high, addr))

	other := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")
	fmt.Printf("  %-34s %v\n", "claimed by a different address", Verify(msg, sig, other))

	fmt.Printf("  %-34s %v\n", "wrong message", Verify("something else", sig, addr))

	// Expiry.
	msg2 := srv.Issue("ffffffffffff", 5*time.Minute, now)
	sig2, _ := crypto.Sign(accounts.TextHash([]byte(msg2)), key)
	late := now.Add(10 * time.Minute)
	fmt.Printf("\n%-38s %v\n", "signed correctly, but 10 min late", srv.Login(msg2, sig2, addr, late))

	fmt.Println("\nthe order matters: cheap structural checks first, the elliptic-curve")
	fmt.Println("operation last. Recovery costs ~100x a length comparison, and an")
	fmt.Println("unauthenticated endpoint is exactly where you do not want that.")
	fmt.Println("\nlesson 50 turns this into SIWE, adds EIP-1271 for smart accounts,")
	fmt.Println("and swaps the plain message for EIP-712 typed data.")
}
```

**Output:**

```
challenge issued:
  "example.com wants you to sign in.\nNonce: a1b2c3d4e5f6"

honest login                           <nil>
same signature replayed                challenge already used

the checks in Verify:
  wrong length                       signature must be 65 bytes
  invalid recovery id                invalid recovery id
  high s (malleated)                 signature has high s (EIP-2)
  claimed by a different address     recovered a different address
  wrong message                      recovered a different address

signed correctly, but 10 min late      challenge expired

the order matters: cheap structural checks first, the elliptic-curve
operation last. Recovery costs ~100x a length comparison, and an
unauthenticated endpoint is exactly where you do not want that.

lesson 50 turns this into SIWE, adds EIP-1271 for smart accounts,
and swaps the plain message for EIP-712 typed data.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
