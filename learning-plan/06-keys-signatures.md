# 06 — Keys & Digital Signatures (ECDSA on secp256k1)

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#06-keys-digital-signatures-ecdsa-on-secp256k1)

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [04](04-hash-functions.md) |
| **Unlocks** | 07, 10, 19, 42, 50 |
| **Examples** | [18](examples/06-keys-signatures/) (🟢 5 · 🟡 8 · 🔴 5) |

*Private/public keys, the curve, signing, verifying, public-key recovery, malleability and nonce disasters.*

Hashing gave you integrity. Signatures give you **authorization** — the ability to prove you and only
you approved something. This is the lesson where mistakes stop being theoretical: a reused nonce hands
over your key in four lines of algebra, and this lesson does exactly that to a key and shows you the
result.

## Goals

- Generate a secp256k1 keypair in Go and derive the public key.
- Sign a message hash, verify it, and recover the public key from the signature.
- Explain `v`, `r`, `s` and why low-`s` is enforced.
- Explain why a reused or predictable nonce leaks the private key.

## Concepts

### 1. Asymmetric cryptography in one page

You hold a secret number. You publish something derived from it. You can then produce proofs that only
the holder of the secret could produce, and anyone can check them without learning the secret.

That is the whole idea, and it has two applications: **encryption** (anyone can send you a message only
you can read) and **signing** (only you can produce a mark anyone can verify). **This course only ever
signs.** No lesson here encrypts with a public key.

Be precise about what a signature establishes:

> Someone holding this private key produced this signature over this exact message.

That is all. It does **not** prove:

- **Intent.** A signature over a hash proves nothing about whether the signer understood it. This is
  the entire premise of wallet-drainer phishing, and why EIP-191 and EIP-712 exist (topic 5).
- **Freshness.** A signature is valid forever unless the message contains a nonce and an expiry. One
  signature is otherwise a permanent password.
- **Uniqueness.** The same message can be signed twice, and — as topic 8 shows — a single signature
  has two valid byte encodings.

Example [5](examples/06-keys-signatures/1-easy.md#5-change-anything-and-it-fails) demonstrates the
binding, and example
[18](examples/06-keys-signatures/3-hard.md#18-a-complete-signature-verifier) shows what you have to add
around it to get something safe.

### 2. Elliptic curves, intuitively

secp256k1 is the set of points satisfying `y² = x³ + 7` over a finite field of prime order `p`, plus a
"point at infinity". There is a rule for adding two points that produces a third point on the curve.
You will never implement it — you will call a library — but you should know the shape of it.

From addition you get multiplication by repetition: `d·G` means adding the generator point `G` to
itself `d` times (efficiently, by doubling). And that gives you the key pair:

```go
x, y := curve.ScalarBaseMult(key.D.Bytes())   // P = d·G
```

Example [2](examples/06-keys-signatures/1-easy.md#2-the-curve-and-why-d-to-p-is-one-way) computes it
directly and confirms it matches the library's public key, then verifies both G and P satisfy the curve
equation.

**The discrete logarithm problem** is the whole security: given `P` and `G`, finding `d` has no known
shortcut. Multiplying takes about 256 doublings — microseconds. Reversing takes about 2¹²⁸ operations.
Everything in the next five lessons rests on that gap.

Two parameters worth knowing:

- **n**, the order of `G`'s group. A private key must be in `[1, n−1]`. Example
  [13](examples/06-keys-signatures/2-medium.md#13-not-every-32-bytes-is-a-key) feeds zero, `n`, `n+1`
  and all-`0xff` to `crypto.ToECDSA` and reads the errors.
- **The cofactor is 1**, which means every point on the curve is in `G`'s group. That spares you the
  small-subgroup checks some other curves require.

### 3. secp256k1 vs the alternatives

| Curve | Scheme | Used by |
|---|---|---|
| **secp256k1** | ECDSA | Bitcoin, Ethereum EOAs |
| P-256 (secp256r1) | ECDSA | TLS, passkeys/WebAuthn |
| ed25519 | EdDSA | Solana, Cosmos, SSH |
| BLS12-381 | BLS | Ethereum consensus ([42](42-schnorr-bls-aggregate.md)) |

secp256k1 is a Koblitz curve, chosen partly because its structure allows a fast endomorphism-based
multiplication, and partly because its parameters were generated in a visibly non-arbitrary way — a
point of some significance given the suspicion around NIST's curve constants.

**Go's standard library does not include secp256k1.** `crypto/elliptic` gives you the NIST curves, so
`elliptic.P256()` or a bare `ecdsa.GenerateKey` in chain code is a bug that compiles, runs, and
produces a perfectly valid keypair no chain will ever recognise. Example
[3](examples/06-keys-signatures/1-easy.md#3-three-curves-three-schemes) takes the same 32-byte secret
onto both curves and shows two entirely different public keys.

ed25519 deserves a note because it is a different **scheme**, not just a different curve. It is
deterministic by construction, so the nonce catastrophe in topic 9 is structurally impossible on
Solana and Cosmos.

### 4. Key generation in Go

```go
key, err := crypto.GenerateKey()          // secp256k1, from crypto/rand
b := crypto.FromECDSA(key)                // the 32-byte scalar
key2, err := crypto.ToECDSA(b)            // and back, with validation
key3, err := crypto.HexToECDSA(hexString) // from a hex string
```

Public keys have two serializations, and example
[6](examples/06-keys-signatures/2-medium.md#6-compressed-and-uncompressed-public-keys) covers both:

- **Uncompressed**, 65 bytes: `0x04 ‖ X ‖ Y`. What Ethereum hashes to make an address
  ([07](07-addresses-wallets-hd.md)), and what `Ecrecover` returns.
- **Compressed**, 33 bytes: `0x02` or `0x03` ‖ X, where the prefix records whether Y is even or odd.
  The curve equation has exactly two solutions for Y and they sum to `p`, so one bit is enough to pick
  the right one. Bitcoin uses this everywhere.

**Entropy is the actual weak point.** Not the curve, not the algorithm — the randomness. Example
[12](examples/06-keys-signatures/2-medium.md#12-entropy-is-the-whole-game) seeds `math/rand` with 42
and prints an address that is identical on every machine forever, then derives a key from
`keccak256("password")` — the classic "brainwallet", every one of which has been precomputed and is
swept within seconds of receiving funds. Use `crypto/rand` or a KDF over real entropy. Never
`math/rand`, never a passphrase, never `time.Now()` as a seed.

### 5. You sign a hash, never a message

ECDSA operates on a 32-byte integer. The message must be hashed first — which creates a problem:

> A 32-byte digest carries no information about what it was a digest **of**.

If a site persuades you to sign "a login challenge" that is actually the hash of a transaction, your
signature authorises the transaction. **EIP-191** fixes this by prefixing:

```
keccak256("\x19Ethereum Signed Message:\n" + len(message) + message)
```

The `0x19` byte is chosen because a valid RLP-encoded transaction can never begin with it
([17](17-rlp-merkle-patricia-trie.md)), so a `personal_sign` digest can never collide with a
transaction digest. In Go it is `accounts.TextHash(msg)`, and example
[7](examples/06-keys-signatures/2-medium.md#7-sign-the-hash-and-prefix-it) also builds it by hand to
show there is no magic in it.

This is domain separation, exactly as in [04](04-hash-functions.md) — one byte of context that stops a
signature meaning something you did not intend. EIP-712 ([50](50-offchain-signatures-siwe.md)) is the
structured version: instead of an opaque string, the user sees typed fields, and the domain includes
the chain id and the verifying contract.

### 6. ECDSA internals

Signing, with `z` the message hash and `d` the private key:

```
pick a nonce k, uniformly at random in [1, n-1]
R = k·G                     r = R.x mod n
s = k⁻¹ · (z + r·d) mod n
signature = (r, s)
```

Verification recomputes `R` from `(r, s, z, P)` and checks its x-coordinate equals `r`. No secret is
involved. Example [14](examples/06-keys-signatures/3-hard.md#14-malleability-r-and-n-minus-s)
implements the verification equation from scratch, which is worth reading once:

```go
w  := new(big.Int).ModInverse(s, n)
u1 := new(big.Int).Mod(new(big.Int).Mul(z, w), n)
u2 := new(big.Int).Mod(new(big.Int).Mul(r, w), n)
x1, y1 := curve.ScalarBaseMult(u1.Bytes())
x2, y2 := curve.ScalarMult(px, py, u2.Bytes())
x, _ := curve.Add(x1, y1, x2, y2)
valid := new(big.Int).Mod(x, n).Cmp(r) == 0
```

**Public-key recovery** is the property that shapes Ethereum. From `(r, s, z)` you can reconstruct up
to four candidate public keys; one extra bit — `v` — selects the right one. So the signature carries
the signer's identity implicitly:

> **This is why an Ethereum transaction has no `from` field.** The sender is *recovered*, never
> declared, so it cannot be forged by lying in a field.

Example [9](examples/06-keys-signatures/2-medium.md#9-recovery-why-there-is-no-from-field) does it,
and makes the crucial follow-up: recovery on a tampered signature returns a **different valid-looking
address**, not an error. Recovery is not verification. Recover, then compare against who you expected.

### 7. The go-ethereum signature API

```go
sig, err := crypto.Sign(hash, key)              // 65 bytes: r ‖ s ‖ v, v ∈ {0,1}
pubBytes, err := crypto.Ecrecover(hash, sig)    // 65-byte uncompressed pubkey
pub, err := crypto.SigToPub(hash, sig)          // *ecdsa.PublicKey
ok := crypto.VerifySignature(pubkey, hash, sig[:64])   // note: 64 bytes, not 65
```

Three sharp edges, all of which fail **silently**:

1. **`VerifySignature` takes 64 bytes.** Pass all 65 and it returns `false` — not an error. Example
   [4](examples/06-keys-signatures/1-easy.md#4-sign-and-verify) shows it.
2. **The +27 offset.** `crypto.Sign` returns `v ∈ {0,1}`. Wallets and `eth_sign` return 27/28, and
   post-EIP-155 transactions encode the chain id in it ([19](19-transaction-types.md)). Feeding 27 to
   `Ecrecover` errors; feeding the *wrong parity* recovers a different address with no complaint at
   all. Example [10](examples/06-keys-signatures/2-medium.md#10-the-27-offset) shows both.
3. **Recovery always returns something.** See topic 6.

There are also two wire encodings, covered in example
[11](examples/06-keys-signatures/2-medium.md#11-two-encodings-compact-and-der): Ethereum's fixed
65-byte compact form, and Bitcoin's variable-length ASN.1 DER (70–72 bytes, no recovery id). DER's
flexibility let several byte sequences encode the same signature, which contributed to Bitcoin's txid
malleability until BIP-66 made strict DER a consensus rule.

### 8. Malleability

If `(r, s)` is a valid signature, so is `(r, n−s)`. The verification equation is symmetric in `s`, so
both satisfy it — same key, same message, different bytes.

Example 14 proves this with the hand-written verifier from topic 6, then shows what the libraries
actually do:

| | `(r, s)` | `(r, n−s)` |
|---|---|---|
| Raw ECDSA equation | ✅ valid | ✅ valid |
| `crypto.Ecrecover` | accepts, correct address | accepts, **same** address |
| `crypto.VerifySignature` | ✅ | ❌ rejected (low-s rule) |

The consequence used to be serious. A Bitcoin txid covered the signatures, so anyone relaying a
transaction could flip `s`, change its id, and break every service tracking that transaction by id —
some of which re-sent, and lost money. **BIP-62/BIP-146** made low-s a rule; SegWit removed signatures
from the txid preimage entirely ([36](36-bitcoin-deep-dive.md)). **EIP-2** enforces low-s on Ethereum,
which is why `VerifySignature` says false above, and go-ethereum's signer always produces low-s.

The rule that outlives all of those fixes:

> **Never key anything on signature bytes.** Key on the hash of the signed payload, which no third
> party can change.

### 9. Nonce catastrophes

The nonce `k` must be **secret, unique and unbiased**. Break any of those and the private key follows.

Sign two different messages with the same `k`:

```
s₁ = k⁻¹(z₁ + r·d)        s₂ = k⁻¹(z₂ + r·d)
s₁ − s₂ = k⁻¹(z₁ − z₂)    ⇒   k = (z₁ − z₂) / (s₁ − s₂)
                          ⇒   d = (s₁·k − z₁) / r
```

Four modular operations, using only public values. Example
[15](examples/06-keys-signatures/3-hard.md#15-nonce-reuse-hands-over-the-private-key) performs the
whole attack and recovers the exact private key it started with. Note that the reuse is
**self-announcing**: `r` depends only on `k`, so a repeated `r` is visible to anyone watching the
chain — and scanners do watch.

This has happened, repeatedly:

- **2010 — Sony PlayStation 3.** The ECDSA nonce was a *constant*. The console's code-signing key was
  recovered and the platform was fully unlocked.
- **2013 — Android SecureRandom.** A flaw made the RNG repeat. Bitcoin wallets on affected devices
  produced colliding nonces and were emptied.

And it is worse than "do not repeat `k`": a **biased** nonce leaks too. Lattice attacks recover a key
from a few hundred signatures whose nonces share as few as a handful of known bits.

**RFC 6979** removes the RNG from the signing path entirely by deriving `k` deterministically:

```
k = HMAC-SHA256(private key, message hash)   // iterated, per the spec
```

Same key and message always give the same `k`, and different messages give unrelated ones. Example
[16](examples/06-keys-signatures/3-hard.md#16-rfc-6979-deriving-k-from-the-key) implements it from
scratch and lands on the **exact nonce go-ethereum uses internally** — which is how you know the
implementation is right. It also means signing is reproducible, which makes it testable and lets
hardware wallets be audited.

What RFC 6979 does *not* fix: a weak **key** is still weak (topic 4), and fault injection during
signing can still produce related nonces.

### 10. What replaces ECDSA

ECDSA has awkward properties — malleability, a fragile nonce, no aggregation — that newer schemes
avoid.

**Schnorr (BIP-340)** signs with `s = k + H(R‖P‖m)·d`. The private key appears **linearly**, where
ECDSA buries it under a modular inverse. That single structural difference means signatures and keys
can be *added*: `s₁ + s₂` is a valid signature for `d₁ + d₂`. Example
[17](examples/06-keys-signatures/3-hard.md#17-schnorr-and-what-linearity-buys) signs with both and
compares — 64 bytes versus 65, x-only public keys, no recovery byte.

Aggregation means an n-of-n multisig looks like a single-key spend on-chain: cheaper and more private.
The catch is that **naive aggregation is broken** — if I choose my key after seeing yours, I can pick
`P₂ = X − P₁` and sign for the aggregate alone. That is the rogue-key attack, and MuSig2 fixes it with
key-aggregation coefficients ([42](42-schnorr-bls-aggregate.md)). Bitcoin Taproot uses Schnorr for
key-path spends ([36](36-bitcoin-deep-dive.md)).

**BLS** goes further: signatures aggregate **non-interactively**, so thousands compress into one. That
is what makes Ethereum's consensus layer work, where every validator attests every epoch
([28](28-proof-of-stake.md)). The cost is pairing operations, which are slow.

**Threshold signing (MPC/TSS)** splits the key so no machine ever holds it
([43](43-multisig-mpc-threshold.md)).

**Post-quantum**: none of this survives a cryptographically relevant quantum computer. Shor's algorithm
solves discrete logarithms, so every key whose public key is known is at risk — which on a blockchain
means every address that has ever *spent*. Hash-based signatures survive; the migration is unsolved
and is an active area of work.

## Exercises

Write these in `practice/06-keys-signatures/`.

1. **Key round trips.** Generate a key, serialize it, reload it, and confirm the address matches. Do
   the same for compressed and uncompressed public keys, and write a table-driven test for the
   invalid-key cases from example 13.
2. **Derive the public key yourself.** Use `ScalarBaseMult` to compute `d·G` and confirm it matches
   the library. Then verify both G and your public key satisfy `y² = x³ + 7 (mod p)`.
3. **Sign and verify, three ways.** Sign a message and verify it with `VerifySignature`, by
   recovering with `SigToPub`, and with your own implementation of the verification equation. All
   three must agree.
4. **Break your own verifier.** Write tests for: 65 bytes passed to `VerifySignature`, v of 27
   unnormalised, the wrong recovery bit, a flipped `s`, and a signature from a different key. Each
   should fail for a *distinct* reason.
5. **The EIP-191 prefix.** Implement `TextHash` yourself and check it against
   `accounts.TextHash` for empty, ASCII and multi-byte UTF-8 messages. Then explain in a comment what
   goes wrong if you use `len(string)` instead of `len([]byte)`.
6. **Recover the key.** Reproduce example 15 from scratch with your own key. Then extend it: given
   only two signatures with a repeated `r`, detect the reuse automatically and recover `d` without
   being told which is which.
7. **RFC 6979 against test vectors.** Implement the derivation and check it against the vectors in
   the RFC's appendix. Then confirm your `r` matches `crypto.Sign`'s for ten different messages.
8. **A verifier worth shipping.** Extend example 18: add a rate limit per address, structured logging
   that never prints the signature, and a benchmark showing how much cheaper the structural checks
   are than the recovery. Decide what your nonce TTL should be and write down why.

## Best Practices & Pitfalls

- **Use secp256k1, not Go's default curve.**
  *Why:* `elliptic.P256()` and a bare `ecdsa.GenerateKey` compile fine and produce valid keys that no
  chain recognises. If your chain code imports `crypto/elliptic` for key generation, it is wrong.
- **`crypto/rand` only — never `math/rand`, a passphrase, or a timestamp seed.**
  *Why:* the curve is not the weak point, the entropy is. Brainwallet addresses are precomputed and
  swept within seconds, and a fixed `math/rand` seed produces the same key on every machine forever.
- **Never let a caller supply the nonce, and never write your own signer.**
  *Why:* two signatures sharing a nonce give up the private key in four operations (example 15). Use
  a library that implements RFC 6979 — go-ethereum does.
- **Hash before signing, and prefix before hashing.**
  *Why:* a bare 32-byte digest could be the hash of anything, including a transaction you never saw.
  EIP-191's `0x19` prefix makes that collision impossible.
- **Pass 64 bytes to `VerifySignature`, not 65.**
  *Why:* it returns `false` rather than erroring, so the bug looks like a bad signature and you go
  hunting in the wrong place.
- **Normalise `v` to {0,1} before recovering, and check the result.**
  *Why:* 27 makes `Ecrecover` error; the wrong parity makes it return a valid but **wrong** address
  with no warning at all.
- **Recovery is not verification.**
  *Why:* `SigToPub` succeeds on garbage. Always compare the recovered address against the one you
  expected — the comparison is the verification.
- **Reject high-`s` signatures at your boundary.**
  *Why:* `Ecrecover` accepts both `(r, s)` and `(r, n−s)`, so without an explicit check one logical
  approval has two valid byte encodings — and any dedupe keyed on signature bytes fails.
- **Never key anything on signature bytes.**
  *Why:* malleability means the bytes are not a stable identifier. Key on the hash of the signed
  payload.
- **A signature alone is not authorization.**
  *Why:* it never expires and can be replayed forever. Bind it to a server-issued nonce, an expiry, a
  chain id and a verifying contract — example 18 and [50](50-offchain-signatures-siwe.md).

## Checklist

- [ ] I can generate a secp256k1 keypair in Go and derive its address.
- [ ] I can explain why `d·G` is easy and recovering `d` is not.
- [ ] I know which curve and scheme each major chain uses.
- [ ] I can convert between compressed and uncompressed public keys and explain the prefix byte.
- [ ] I always hash before signing, and I know why EIP-191 prefixes the hash.
- [ ] I can write out the ECDSA signing and verification equations.
- [ ] I can explain why an Ethereum transaction has no `from` field.
- [ ] I know the three silent failure modes in go-ethereum's signature API.
- [ ] I can demonstrate signature malleability and explain what low-`s` fixes.
- [ ] I can recover a private key from two signatures sharing a nonce.
- [ ] I can explain what RFC 6979 buys and what it does not.
- [ ] I can say why Schnorr aggregates and ECDSA does not.

## Resources

**Specifications**

- SEC 2 — secp256k1 parameters: https://www.secg.org/sec2-v2.pdf
- RFC 6979 — deterministic ECDSA: https://datatracker.ietf.org/doc/html/rfc6979
- EIP-191 — signed data standard: https://eips.ethereum.org/EIPS/eip-191
- EIP-2 — low-`s` enforcement: https://eips.ethereum.org/EIPS/eip-2
- BIP-340 — Schnorr signatures: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- BIP-66 — strict DER: https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki

**Go**

- go-ethereum `crypto`: https://pkg.go.dev/github.com/ethereum/go-ethereum/crypto
- `btcec/v2`: https://pkg.go.dev/github.com/btcsuite/btcd/btcec/v2
- `crypto/ecdsa`: https://pkg.go.dev/crypto/ecdsa
- `crypto/ed25519`: https://pkg.go.dev/crypto/ed25519

**Incidents & background**

- fail0verflow, PS3 ECDSA nonce reuse (27C3): https://media.ccc.de/v/27c3-4087-en-console_hacking_2010
- Android SecureRandom advisory (2013): https://bitcoin.org/en/alert/2013-08-11-android
- Ethereum docs — accounts and keys: https://ethereum.org/en/developers/docs/accounts/

---

**Examples:** [`examples/06-keys-signatures/`](examples/06-keys-signatures/) — **18 runnable Go
programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard), all using the published anvil test key so every
output reproduces exactly. Example 15 recovers a private key from two signatures that share a nonce;
example 16 implements RFC 6979 and lands on the same nonce go-ethereum uses internally.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
