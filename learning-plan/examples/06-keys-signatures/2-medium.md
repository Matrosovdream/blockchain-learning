# Step 06 — Keys & Digital Signatures · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Compressed and uncompressed public keys

`🟡 medium` · *Keys*

A public key is a point, and the curve equation has exactly two solutions for Y given X. So you can drop Y entirely and store a one-byte parity hint instead — 33 bytes rather than 65. Bitcoin cares because it stores millions of them; Ethereum hashes the uncompressed form, so the choice is made for you.

**Steps:**

1. Print both forms and their prefixes.
2. Work out which parity your key has and check it against the prefix byte.
3. Decompress and confirm you recover the same point.
4. Compute the other root, `p − Y`, and confirm the two sum to the field prime.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	uncompressed := crypto.FromECDSAPub(&key.PublicKey)
	compressed := crypto.CompressPubkey(&key.PublicKey)

	fmt.Printf("uncompressed (%d bytes)\n  %s\n", len(uncompressed), hex.EncodeToString(uncompressed))
	fmt.Printf("  0x04 ‖ X (32) ‖ Y (32)\n")
	fmt.Printf("\ncompressed   (%d bytes)\n  %s\n", len(compressed), hex.EncodeToString(compressed))
	fmt.Printf("  0x%02x ‖ X (32)   — Y is recomputed from the curve equation\n", compressed[0])

	// Why it works: y^2 = x^3 + 7 has exactly two solutions for y, and they sum
	// to p. One is even, one is odd. The prefix says which one to take.
	fmt.Printf("\nY is %s, so the prefix is 0x%02x\n",
		map[bool]string{true: "even", false: "odd"}[key.PublicKey.Y.Bit(0) == 0],
		compressed[0])
	fmt.Println("  0x02 = even Y, 0x03 = odd Y")

	// Decompress and confirm we get the same point back.
	back, err := crypto.DecompressPubkey(compressed)
	if err != nil {
		fmt.Println("decompress:", err)
		return
	}
	fmt.Printf("\ndecompressed matches original: %v\n",
		back.X.Cmp(key.PublicKey.X) == 0 && back.Y.Cmp(key.PublicKey.Y) == 0)

	// The other root, for illustration: p - Y.
	other := new(big.Int).Sub(btcec.S256().P, key.PublicKey.Y)
	fmt.Printf("\nthe two possible Y values for this X:\n  %s\n  %s\n",
		hex.EncodeToString(key.PublicKey.Y.Bytes()), hex.EncodeToString(other.Bytes()))
	fmt.Printf("they sum to p: %v\n",
		new(big.Int).Add(key.PublicKey.Y, other).Cmp(btcec.S256().P) == 0)

	// Which form to use where.
	fmt.Println("\nwhere each form appears")
	fmt.Println("  uncompressed  Ethereum address derivation (lesson 07), ecrecover output")
	fmt.Println("  compressed    Bitcoin addresses since ~2012, BIP-32 xpubs, P2WPKH")
	fmt.Println("\nBitcoin cares because 32 bytes per key adds up across millions of UTXOs.")
	fmt.Println("Ethereum hashes the uncompressed form, so the choice is fixed for you.")
}
```

**Output:**

```
uncompressed (65 bytes)
  048318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed753547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5
  0x04 ‖ X (32) ‖ Y (32)

compressed   (33 bytes)
  038318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75
  0x03 ‖ X (32)   — Y is recomputed from the curve equation

Y is odd, so the prefix is 0x03
  0x02 = even Y, 0x03 = odd Y

decompressed matches original: true

the two possible Y values for this X:
  3547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5
  cab80ee3579699b90d0c534f71cefe95053dc19cf3a2ee0a609e010984f2d18a
they sum to p: true

where each form appears
  uncompressed  Ethereum address derivation (lesson 07), ecrecover output
  compressed    Bitcoin addresses since ~2012, BIP-32 xpubs, P2WPKH

Bitcoin cares because 32 bytes per key adds up across millions of UTXOs.
Ethereum hashes the uncompressed form, so the choice is fixed for you.
```

---

## 7. Sign the hash, and prefix it

`🟡 medium` · *Domain separation*

A 32-byte digest carries no information about what it was a digest *of*. Sign a bare hash and you may have signed a transaction you never saw. EIP-191 prefixes the message with a byte that a valid RLP transaction can never start with, so the two digests can never collide.

**Steps:**

1. Sign a bare Keccak hash and note what an attacker could substitute.
2. Build the EIP-191 digest with `accounts.TextHash`, then again by hand.
3. Understand why the prefix byte is `0x19` specifically.
4. Recover the signer's address — the server side of a login flow (lesson 50).

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/accounts"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	message := []byte("Log in to example.com")

	// The naive approach: hash the message and sign it.
	rawHash := crypto.Keccak256(message)
	rawSig, _ := crypto.Sign(rawHash, key)
	fmt.Printf("raw keccak of the message\n  hash %s\n  sig  %s\n",
		hex.EncodeToString(rawHash), hex.EncodeToString(rawSig)[:32]+"...")

	// THE PROBLEM: that hash could be anything. A 32-byte digest carries no
	// information about what it was a digest OF. If a site tricks you into
	// signing "a login challenge" that is really the hash of a transaction,
	// your signature authorises the transaction.
	fmt.Println("\nthe danger: a signature over a bare hash is a signature over ANY")
	fmt.Println("preimage of that hash — including a transaction you never saw.")

	// EIP-191 fixes it with a prefix that a transaction can never have.
	prefixed := accounts.TextHash(message)
	fmt.Printf("\nEIP-191 personal_sign\n  prefix \"\\x19Ethereum Signed Message:\\n%d\"\n", len(message))
	fmt.Printf("  hash   %s\n", hex.EncodeToString(prefixed))

	// Build it by hand to see there is no magic.
	manual := crypto.Keccak256([]byte(fmt.Sprintf("\x19Ethereum Signed Message:\n%d%s", len(message), message)))
	fmt.Printf("  built by hand matches accounts.TextHash: %v\n",
		hex.EncodeToString(manual) == hex.EncodeToString(prefixed))

	// Why 0x19: an RLP-encoded transaction can never start with that byte, so
	// a personal_sign digest can never collide with a transaction digest.
	fmt.Println("\nwhy 0x19: a valid RLP transaction never starts with 0x19, so this")
	fmt.Println("digest can never be mistaken for a transaction (lesson 17, 19).")

	sig, _ := crypto.Sign(prefixed, key)
	fmt.Printf("\nsigned    %s\n", hex.EncodeToString(sig))

	// Recover the signer, which is what a server does on login (lesson 50).
	recovered, err := crypto.SigToPub(prefixed, sig)
	if err != nil {
		fmt.Println("recover:", err)
		return
	}
	fmt.Printf("recovered %s\n", crypto.PubkeyToAddress(*recovered).Hex())
	fmt.Printf("expected  %s\n", crypto.PubkeyToAddress(key.PublicKey).Hex())

	// The same signature does NOT verify against the unprefixed hash.
	fmt.Printf("\nchecked against the unprefixed hash: %v\n",
		crypto.VerifySignature(crypto.FromECDSAPub(&key.PublicKey), rawHash, sig[:64]))
	fmt.Println("\ndomain separation, exactly as in lesson 04 — one byte of context")
	fmt.Println("that stops a signature meaning something you did not intend.")
	fmt.Println("EIP-712 (lesson 50) is the structured version of the same idea.")
}
```

**Output:**

```
raw keccak of the message
  hash 5eb57c4bbe90c19503fe3730d292bf0ef052595bc448fcc15ae0d5d90e2cd150
  sig  296bf14ad54facb8139e731fa2d7391a...

the danger: a signature over a bare hash is a signature over ANY
preimage of that hash — including a transaction you never saw.

EIP-191 personal_sign
  prefix "\x19Ethereum Signed Message:\n21"
  hash   3502fcf77cf75bfe63597f483e36de0a9a1fe024e041bfdf2a4866c84a736aa9
  built by hand matches accounts.TextHash: true

why 0x19: a valid RLP transaction never starts with 0x19, so this
digest can never be mistaken for a transaction (lesson 17, 19).

signed    b77fe48adcbb4ec672f5309dcc4aa1fa27daf6efa8f5fd03dbb0841b65e31c2e5f576ad266e63d4bb755fdc074fa6c0580e4d4fdcab964e72784702c78d8ad0600
recovered 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
expected  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

checked against the unprefixed hash: false

domain separation, exactly as in lesson 04 — one byte of context
that stops a signature meaning something you did not intend.
EIP-712 (lesson 50) is the structured version of the same idea.
```

---

## 8. Inside a signature: r, s, v

`🟡 medium` · *Signatures*

A signature is two numbers and a hint. `r` comes from the nonce, `s` mixes in the hash and the private key, and `v` says which candidate point to use when recovering. Both r and s must lie in [1, n−1], and EIP-2 additionally requires s in the lower half.

**Steps:**

1. Split a signature into r, s and v and print the signing equations.
2. Confirm both values are inside the group order.
3. Check the low-s rule against n/2.
4. Note what is *not* in the signature: the nonce, and the private key.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	hash := crypto.Keccak256([]byte("transfer 100 to bob"))
	sig, _ := crypto.Sign(hash, key)

	r := new(big.Int).SetBytes(sig[0:32])
	s := new(big.Int).SetBytes(sig[32:64])
	v := sig[64]
	n := btcec.S256().N

	fmt.Println("a signature is two numbers and a hint")
	fmt.Printf("  r %s\n", hex.EncodeToString(sig[0:32]))
	fmt.Printf("  s %s\n", hex.EncodeToString(sig[32:64]))
	fmt.Printf("  v %d\n", v)

	// What the signing equations produced.
	fmt.Println("\nwhere they come from")
	fmt.Println("  pick a random nonce k")
	fmt.Println("  R = k*G          r = R.x mod n")
	fmt.Println("  s = k^-1 * (z + r*d) mod n     (z is the hash, d the private key)")
	fmt.Println("  v records which of the candidate R points to use when recovering")

	// Both r and s must be in [1, n-1]. Zero is invalid.
	fmt.Printf("\nboth r and s live in [1, n-1]\n")
	fmt.Printf("  n     %s\n", n.Text(16))
	fmt.Printf("  r < n %v      s < n %v\n", r.Cmp(n) < 0, s.Cmp(n) < 0)
	fmt.Printf("  r > 0 %v      s > 0 %v\n", r.Sign() > 0, s.Sign() > 0)

	// EIP-2 requires s to be in the LOWER half. go-ethereum produces low-s.
	halfN := new(big.Int).Rsh(n, 1)
	fmt.Printf("\nEIP-2 requires s <= n/2 (the 'low-s' rule)\n")
	fmt.Printf("  n/2   %s\n", halfN.Text(16))
	fmt.Printf("  s     %s\n", s.Text(16))
	fmt.Printf("  low-s %v\n", s.Cmp(halfN) <= 0)
	fmt.Println("  example 14 shows what the other half would let you do")

	// The k that produced this signature is NOT recoverable from the signature.
	// If it were, so would be the private key (example 15).
	fmt.Println("\nnote what is NOT in the signature: the nonce k, and the private key.")
	fmt.Println("recovering either from (r, s) is exactly as hard as breaking the curve —")
	fmt.Println("unless you reuse k, in which case it takes one line of algebra.")
}
```

**Output:**

```
a signature is two numbers and a hint
  r e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e
  s 5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb40
  v 0

where they come from
  pick a random nonce k
  R = k*G          r = R.x mod n
  s = k^-1 * (z + r*d) mod n     (z is the hash, d the private key)
  v records which of the candidate R points to use when recovering

both r and s live in [1, n-1]
  n     fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
  r < n true      s < n true
  r > 0 true      s > 0 true

EIP-2 requires s <= n/2 (the 'low-s' rule)
  n/2   7fffffffffffffffffffffffffffffff5d576e7357a4501ddfe92f46681b20a0
  s     5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb40
  low-s true
  example 14 shows what the other half would let you do

note what is NOT in the signature: the nonce k, and the private key.
recovering either from (r, s) is exactly as hard as breaking the curve —
unless you reuse k, in which case it takes one line of algebra.
```

---

## 9. Recovery: why there is no from field

`🟡 medium` · *Recovery*

ECDSA has a property most signature schemes lack: given the signature and the hash, you can recover the public key. That is why an Ethereum transaction has no `from` field — the sender is derived, never declared, so it cannot be forged by lying in a field.

**Steps:**

1. Recover the public key with `Ecrecover` and compare it to the signer's.
2. Use `SigToPub` and derive the address.
3. Tamper with the signature and watch recovery return a *different* address rather than an error.
4. Conclude: recovery is not verification. Recover, then compare against who you expected.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	signer := crypto.PubkeyToAddress(key.PublicKey)

	hash := crypto.Keccak256([]byte("transfer 100 to bob"))
	sig, _ := crypto.Sign(hash, key)

	fmt.Printf("signer   %s\n", signer.Hex())
	fmt.Printf("hash     %s\n", hex.EncodeToString(hash))
	fmt.Printf("sig      %s\n", hex.EncodeToString(sig))

	// ECDSA has a property most signature schemes do not: from (r, s, v) and
	// the hash you can RECOVER the public key. No public key is transmitted.
	pubBytes, err := crypto.Ecrecover(hash, sig)
	if err != nil {
		fmt.Println("ecrecover:", err)
		return
	}
	fmt.Printf("\nEcrecover -> %d-byte pubkey\n  %s\n", len(pubBytes), hex.EncodeToString(pubBytes))
	fmt.Printf("matches the signer's public key: %v\n",
		hex.EncodeToString(pubBytes) == hex.EncodeToString(crypto.FromECDSAPub(&key.PublicKey)))

	// SigToPub does the same and hands back a typed key.
	pub, _ := crypto.SigToPub(hash, sig)
	fmt.Printf("\nSigToPub -> address %s\n", crypto.PubkeyToAddress(*pub).Hex())
	fmt.Printf("equals the signer: %v\n", crypto.PubkeyToAddress(*pub) == signer)

	// THIS IS WHY an Ethereum transaction has no 'from' field. The sender is
	// derived, not declared — so it cannot be forged by lying in a field.
	fmt.Println("\nthis is why an Ethereum transaction has no 'from' field:")
	fmt.Println("  the sender is RECOVERED from the signature, never transmitted.")
	fmt.Println("  you cannot claim to be someone else, because there is nothing to claim in.")

	// The catch: recovery always returns SOMETHING. A garbage signature gives a
	// garbage address, not an error. Always compare against who you expected.
	garbage := append([]byte(nil), sig...)
	garbage[5] ^= 0xff
	if other, err := crypto.SigToPub(hash, garbage); err == nil {
		fmt.Printf("\na tampered signature recovers a DIFFERENT address, not an error:\n  %s\n",
			crypto.PubkeyToAddress(*other).Hex())
	} else {
		fmt.Printf("\na tampered signature failed to recover: %v\n", err)
	}
	fmt.Println("  so recovery is not verification. Recover, then compare to who you expected.")
}
```

**Output:**

```
signer   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
hash     92747a70e6add69d7367fd87fc8b47878b6627717081c2bcd7f54afbb4672181
sig      e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb4000

Ecrecover -> 65-byte pubkey
  048318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed753547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5
matches the signer's public key: true

SigToPub -> address 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
equals the signer: true

this is why an Ethereum transaction has no 'from' field:
  the sender is RECOVERED from the signature, never transmitted.
  you cannot claim to be someone else, because there is nothing to claim in.

a tampered signature recovers a DIFFERENT address, not an error:
  0x609a80072bB7A2244f84978B7e4929ea0DD49a86
  so recovery is not verification. Recover, then compare to who you expected.
```

---

## 10. The +27 offset

`🟡 medium` · *Recovery*

`crypto.Sign` returns v ∈ {0,1}. Wallets and `eth_sign` return 27/28, and post-EIP-155 transactions use something else again. Getting this wrong either errors loudly or — worse — recovers a perfectly valid but completely wrong address.

**Steps:**

1. Add 27 to v the way a wallet would, and watch `Ecrecover` reject it.
2. Normalise back to {0,1} and recover the correct signer.
3. Flip the recovery bit and see a different address come back, with no error at all.
4. Compare the two signatures: identical except the final byte.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	signer := crypto.PubkeyToAddress(key.PublicKey)

	hash := crypto.Keccak256([]byte("hello"))
	sig, _ := crypto.Sign(hash, key)

	fmt.Printf("crypto.Sign returns v = %d\n", sig[64])
	fmt.Println("  raw recovery id: 0 or 1")

	// Ethereum transactions historically carry v = 27 or 28 (and, after
	// EIP-155, 35 + 2*chainId + parity — see lesson 19). Wallets and
	// eth_sign return 27/28 too. go-ethereum's crypto package uses 0/1.
	ethStyle := append([]byte(nil), sig...)
	ethStyle[64] += 27
	fmt.Printf("\nwhat a wallet / eth_sign hands you: v = %d\n", ethStyle[64])

	// Feeding 27 straight into Ecrecover fails — it wants 0/1.
	if _, err := crypto.Ecrecover(hash, ethStyle); err != nil {
		fmt.Printf("Ecrecover with v=%d -> error: %v\n", ethStyle[64], err)
	}

	// Normalise before recovering. This three-line dance appears in every
	// signature-verifying service you will write (lesson 50).
	normalised := append([]byte(nil), ethStyle...)
	if normalised[64] >= 27 {
		normalised[64] -= 27
	}
	pub, err := crypto.SigToPub(hash, normalised)
	if err != nil {
		fmt.Println("recover:", err)
		return
	}
	fmt.Printf("after subtracting 27 -> %s\n", crypto.PubkeyToAddress(*pub).Hex())
	fmt.Printf("correct signer:         %v\n", crypto.PubkeyToAddress(*pub) == signer)

	// And the dangerous case: flipping the recovery id does NOT error. It
	// recovers a different, entirely valid-looking address.
	flipped := append([]byte(nil), sig...)
	flipped[64] ^= 1
	if other, err := crypto.SigToPub(hash, flipped); err == nil {
		fmt.Printf("\nwith the wrong recovery bit (v=%d):\n", flipped[64])
		fmt.Printf("  recovers %s\n", crypto.PubkeyToAddress(*other).Hex())
		fmt.Printf("  which is NOT the signer: %v\n", crypto.PubkeyToAddress(*other) != signer)
		fmt.Println("  no error, no warning — just the wrong answer.")
	}

	fmt.Printf("\nsignature bytes are identical apart from the last byte:\n")
	fmt.Printf("  %s..%02x\n", hex.EncodeToString(sig[:8]), sig[64])
	fmt.Printf("  %s..%02x\n", hex.EncodeToString(flipped[:8]), flipped[64])

	fmt.Println("\nrule: normalise v to {0,1} before Ecrecover, and always compare")
	fmt.Println("the recovered address against the one you expected.")
}
```

**Output:**

```
crypto.Sign returns v = 0
  raw recovery id: 0 or 1

what a wallet / eth_sign hands you: v = 27
Ecrecover with v=27 -> error: invalid signature recovery id
after subtracting 27 -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
correct signer:         true

with the wrong recovery bit (v=1):
  recovers 0x14Bc8035B965c8b90F584E3A3faB6805b9AA5c61
  which is NOT the signer: true
  no error, no warning — just the wrong answer.

signature bytes are identical apart from the last byte:
  73eebf81a6111366..00
  73eebf81a6111366..01

rule: normalise v to {0,1} before Ecrecover, and always compare
the recovered address against the one you expected.
```

---

## 11. Two encodings: compact and DER

`🟡 medium` · *Encodings*

Ethereum uses a fixed 65-byte compact form that carries the recovery id. Bitcoin uses variable-length ASN.1 DER, which is 70–72 bytes and needs the public key supplied separately. DER's flexibility caused real problems.

**Steps:**

1. Sign the same hash with both libraries.
2. Walk the DER structure byte by byte: SEQUENCE, two INTEGERs, and their lengths.
3. Understand why a leading `0x00` appears — ASN.1 integers are signed.
4. Confirm both encodings carry the same r and s.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	btcecdsa "github.com/btcsuite/btcd/btcec/v2/ecdsa"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	keyBytes, _ := hex.DecodeString("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	ethKey, _ := crypto.ToECDSA(keyBytes)
	btcKey, _ := btcec.PrivKeyFromBytes(keyBytes)

	hash := crypto.Keccak256([]byte("transfer 100 to bob"))

	// --- Ethereum: 65-byte compact ------------------------------------------
	compact, _ := crypto.Sign(hash, ethKey)
	fmt.Printf("compact (Ethereum), %d bytes\n  %s\n", len(compact), hex.EncodeToString(compact))
	fmt.Println("  r (32) ‖ s (32) ‖ v (1)")
	fmt.Println("  fixed length, and carries the recovery id — so no public key needed")

	// --- Bitcoin: DER -------------------------------------------------------
	derSig := btcecdsa.Sign(btcKey, hash)
	der := derSig.Serialize()
	fmt.Printf("\nDER (Bitcoin), %d bytes\n  %s\n", len(der), hex.EncodeToString(der))

	// Walk the DER structure by hand — it is ASN.1, and it is fiddly.
	fmt.Println("\n  DER layout")
	fmt.Printf("    %02x       SEQUENCE\n", der[0])
	fmt.Printf("    %02x       total length %d\n", der[1], der[1])
	fmt.Printf("    %02x       INTEGER\n", der[2])
	fmt.Printf("    %02x       r length %d\n", der[3], der[3])
	rEnd := 4 + int(der[3])
	fmt.Printf("    %s  r\n", hex.EncodeToString(der[4:rEnd]))
	fmt.Printf("    %02x       INTEGER\n", der[rEnd])
	fmt.Printf("    %02x       s length %d\n", der[rEnd+1], der[rEnd+1])
	fmt.Printf("    %s  s\n", hex.EncodeToString(der[rEnd+2:]))

	// DER is VARIABLE length: a leading 0x00 is added when the high bit is set,
	// because ASN.1 integers are signed. That is why signatures are 70-72 bytes.
	fmt.Println("\n  DER integers are signed, so a value with the high bit set gets")
	fmt.Println("  a 0x00 prefix. Signatures are therefore 70, 71 or 72 bytes.")

	// Both encode the same two numbers.
	rC := new(big.Int).SetBytes(compact[0:32])
	sC := new(big.Int).SetBytes(compact[32:64])
	rD := new(big.Int).SetBytes(der[4:rEnd])
	sD := new(big.Int).SetBytes(der[rEnd+2:])
	fmt.Printf("\nsame r in both encodings: %v\n", rC.Cmp(rD) == 0)
	fmt.Printf("same s in both encodings: %v\n", sC.Cmp(sD) == 0)

	// The historical cost of DER's flexibility: multiple byte encodings of the
	// same signature were accepted, which made Bitcoin txids malleable.
	fmt.Println("\nDER's flexibility was a real problem: several byte sequences could")
	fmt.Println("encode the same signature, so a third party could re-encode a")
	fmt.Println("transaction and change its txid. BIP-66 made strict DER a consensus")
	fmt.Println("rule in 2015; SegWit removed the issue entirely (lesson 36).")
}
```

**Output:**

```
compact (Ethereum), 65 bytes
  e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb4000
  r (32) ‖ s (32) ‖ v (1)
  fixed length, and carries the recovery id — so no public key needed

DER (Bitcoin), 71 bytes
  3045022100e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e02205ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb40

  DER layout
    30       SEQUENCE
    45       total length 69
    02       INTEGER
    21       r length 33
    00e0cb3e6fd5d6ee6be5f8462a3aaa82424d5efff2877bd2c891cd06e02ab2c15e  r
    02       INTEGER
    20       s length 32
    5ecedd4fbc24acebebd2468062ea3863a0a5a0b1dc629a3b4379c82ac12bdb40  s

  DER integers are signed, so a value with the high bit set gets
  a 0x00 prefix. Signatures are therefore 70, 71 or 72 bytes.

same r in both encodings: true
same s in both encodings: true

DER's flexibility was a real problem: several byte sequences could
encode the same signature, so a third party could re-encode a
transaction and change its txid. BIP-66 made strict DER a consensus
rule in 2015; SegWit removed the issue entirely (lesson 36).
```

---

## 12. Entropy is the whole game

`🟡 medium` · *Entropy*

The curve is not the weak point; the randomness is. `math/rand` with a fixed seed produces the same "key" on every machine forever, and a passphrase-derived key has been precomputed by someone else already. Both have emptied real wallets.

**Steps:**

1. Generate a key properly with `crypto.GenerateKey` — printing only its shape, never its value.
2. Seed `math/rand` with 42 and print the resulting address: identical on every run.
3. Derive a key from `keccak256("password")` — the classic brainwallet.
4. Compare the entropy of each approach against the 2²⁵⁶ key space.

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	mrand "math/rand"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// The right way. crypto/rand reads from the OS entropy source.
	good, err := crypto.GenerateKey()
	if err != nil {
		fmt.Println("generate:", err)
		return
	}
	// Print only its SHAPE, never its value — a real key must not reach a log
	// or a terminal (lesson 02's Secret type).
	fmt.Printf("crypto.GenerateKey -> %d-byte key, valid: %v\n",
		len(crypto.FromECDSA(good)), good.D.Sign() > 0)
	fmt.Println("  different on every run — as it must be")

	buf := make([]byte, 32)
	if _, err := rand.Read(buf); err != nil {
		fmt.Println("rand:", err)
		return
	}
	fmt.Printf("crypto/rand direct -> %d bytes of entropy\n", len(buf))

	// THE BUG. math/rand with a fixed seed produces the same "key" every time,
	// on every machine, forever.
	r := mrand.New(mrand.NewSource(42))
	weak := make([]byte, 32)
	r.Read(weak)
	weakKey, _ := crypto.ToECDSA(weak)
	fmt.Printf("\nmath/rand seeded with 42:\n")
	fmt.Printf("  private key %s\n", hex.EncodeToString(weak))
	fmt.Printf("  address     %s\n", crypto.PubkeyToAddress(weakKey.PublicKey).Hex())
	fmt.Println("  identical on every run, on every machine. Anyone can regenerate it.")

	// The other classic: deriving a key from a passphrase ("brainwallet").
	brain := crypto.Keccak256([]byte("password"))
	brainKey, _ := crypto.ToECDSA(brain)
	fmt.Printf("\nkeccak256(\"password\") as a key:\n")
	fmt.Printf("  address %s\n", crypto.PubkeyToAddress(brainKey.PublicKey).Hex())
	fmt.Println("  every brainwallet address has been precomputed and is swept")
	fmt.Println("  within seconds of receiving funds. This is not hypothetical.")

	// How much entropy is actually needed.
	fmt.Println("\nthe numbers")
	fmt.Println("  secp256k1 key space   ~2^256")
	fmt.Println("  a 12-word BIP-39 seed  2^128  (lesson 07) — comfortable")
	fmt.Println("  a strong passphrase    2^40   — brute-forced on a laptop")
	fmt.Println("  math/rand seed         2^63   — and usually seeded predictably")

	// Real incidents.
	fmt.Println("\nwhat went wrong in practice")
	fmt.Println("  2013  Android SecureRandom was broken; Bitcoin wallets on affected")
	fmt.Println("        devices reused nonces, and keys were recovered (example 15)")
	fmt.Println("  2018  'blockchain bandit' swept thousands of ETH from addresses whose")
	fmt.Println("        keys were small integers or weak-RNG output")

	fmt.Println("\nrule: crypto/rand, or a KDF over real entropy. Never math/rand,")
	fmt.Println("never a passphrase, never time.Now() as a seed.")
}
```

**Output:**

```
crypto.GenerateKey -> 32-byte key, valid: true
  different on every run — as it must be
crypto/rand direct -> 32 bytes of entropy

math/rand seeded with 42:
  private key 538c7f96b164bf1b97bb9f4bb472e89f5b1484f25209c9d9343e92ba09dd9d52
  address     0xb78Ee9B75b0946f11533d9B7004E0bE2e7973560
  identical on every run, on every machine. Anyone can regenerate it.

keccak256("password") as a key:
  address 0x9d39856F91822ff0BDc2e234BB0D40124a201677
  every brainwallet address has been precomputed and is swept
  within seconds of receiving funds. This is not hypothetical.

the numbers
  secp256k1 key space   ~2^256
  a 12-word BIP-39 seed  2^128  (lesson 07) — comfortable
  a strong passphrase    2^40   — brute-forced on a laptop
  math/rand seed         2^63   — and usually seeded predictably

what went wrong in practice
  2013  Android SecureRandom was broken; Bitcoin wallets on affected
        devices reused nonces, and keys were recovered (example 15)
  2018  'blockchain bandit' swept thousands of ETH from addresses whose
        keys were small integers or weak-RNG output

rule: crypto/rand, or a KDF over real entropy. Never math/rand,
never a passphrase, never time.Now() as a seed.
```

---

## 13. Not every 32 bytes is a key

`🟡 medium` · *Keys*

A private key is a scalar in [1, n−1], so zero and anything at or above the group order are invalid. The invalid fraction is about 1 in 2¹²⁸, which is why generate-and-check never actually loops — and why BIP-32 still has to specify what to do when it does.

**Steps:**

1. Feed zero, one, n−1, n, n+1 and all-0xff to `crypto.ToECDSA` and read the errors.
2. Compute how many 32-byte values are out of range.
3. Note BIP-32's fallback rule for a derived child key that lands out of range.
4. Then see that key = 1 is perfectly *valid* — and its public key is literally G.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	n := btcec.S256().N

	// A private key is a scalar in [1, n-1]. Not every 32-byte string qualifies.
	cases := []struct {
		name string
		val  *big.Int
	}{
		{"zero", big.NewInt(0)},
		{"one", big.NewInt(1)},
		{"n-1 (the largest valid key)", new(big.Int).Sub(n, big.NewInt(1))},
		{"n (the group order)", new(big.Int).Set(n)},
		{"n+1", new(big.Int).Add(n, big.NewInt(1))},
		{"all 0xff (2^256-1)", new(big.Int).Sub(new(big.Int).Lsh(big.NewInt(1), 256), big.NewInt(1))},
	}

	fmt.Printf("%-30s %-8s %s\n", "candidate", "in range", "crypto.ToECDSA")
	for _, c := range cases {
		inRange := c.val.Sign() > 0 && c.val.Cmp(n) < 0
		b := make([]byte, 32)
		c.val.FillBytes(b)
		_, err := crypto.ToECDSA(b)
		status := "accepted"
		if err != nil {
			status = "rejected: " + err.Error()
		}
		fmt.Printf("%-30s %-8v %s\n", c.name, inRange, status)
	}

	// How much of the 2^256 space is invalid? Almost none — n is very close to
	// 2^256, so rejection sampling terminates on the first try essentially always.
	max := new(big.Int).Lsh(big.NewInt(1), 256)
	invalid := new(big.Int).Sub(max, n)
	fmt.Printf("\n2^256                 %s\n", max.Text(16))
	fmt.Printf("n                     %s\n", n.Text(16))
	fmt.Printf("values >= n           %s\n", invalid.Text(16))
	fmt.Println("  about 1 in 2^128 random 32-byte strings is out of range,")
	fmt.Println("  which is why 'generate and check' never loops in practice.")

	// The practical consequence for HD wallets (lesson 07): BIP-32 specifies
	// what to do if a derived key lands out of range — skip to the next index.
	fmt.Println("\nBIP-32 (lesson 07) specifies the fallback anyway: if a derived")
	fmt.Println("child key is zero or >= n, skip that index and use the next one.")
	fmt.Println("You will never see it happen, and the spec still has to say it.")

	// And a reminder that a valid key is not a SAFE key (example 12).
	small, _ := crypto.ToECDSA(func() []byte { b := make([]byte, 32); b[31] = 1; return b }())
	fmt.Printf("\nkey = 1 is perfectly valid: address %s\n",
		crypto.PubkeyToAddress(small.PublicKey).Hex())
	fmt.Printf("  its public key is literally G: %v\n",
		small.PublicKey.X.Cmp(btcec.S256().Gx) == 0)
	fmt.Println("  valid, and swept the instant anything lands there.")
	fmt.Printf("  (private key %s)\n", hex.EncodeToString(crypto.FromECDSA(small)))
}
```

**Output:**

```
candidate                      in range crypto.ToECDSA
zero                           false    rejected: invalid private key, zero or negative
one                            true     accepted
n-1 (the largest valid key)    true     accepted
n (the group order)            false    rejected: invalid private key, >=N
n+1                            false    rejected: invalid private key, >=N
all 0xff (2^256-1)             false    rejected: invalid private key, >=N

2^256                 10000000000000000000000000000000000000000000000000000000000000000
n                     fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
values >= n           14551231950b75fc4402da1732fc9bebf
  about 1 in 2^128 random 32-byte strings is out of range,
  which is why 'generate and check' never loops in practice.

BIP-32 (lesson 07) specifies the fallback anyway: if a derived
child key is zero or >= n, skip that index and use the next one.
You will never see it happen, and the spec still has to say it.

key = 1 is perfectly valid: address 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf
  its public key is literally G: true
  valid, and swept the instant anything lands there.
  (private key 0000000000000000000000000000000000000000000000000000000000000001)
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
