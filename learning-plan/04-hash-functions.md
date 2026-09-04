# 04 — Cryptographic Hash Functions

> **Status:** ✅ written. Examples: 20/20 built and run.
> **Spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#04-cryptographic-hash-functions)

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [03](03-bytes-encoding.md) |
| **Unlocks** | 05, 06, 08, 44, 55 |
| **Examples** | [20](examples/04-hash-functions/) (🟢 6 · 🟡 8 · 🔴 6) |

*SHA-256, Keccak-256, the five properties, commitments, and hashing structured data deterministically.*

The first of the four cryptographic primitives. Hashing is the one you will use most and think about
least — which is exactly why the mistakes in it are so durable. Two of them, wrong-hash-function and
non-canonical-input, account for a remarkable share of the time people lose in this field.

## Goals

- Compute SHA-256, SHA-3 and Keccak-256 in Go and know which chain uses which.
- State the five properties of a cryptographic hash and what breaks when each fails.
- Use a hash as a commitment and as an identifier.
- Serialize a struct deterministically before hashing it.

## Concepts

### 1. What a hash function is

A cryptographic hash takes input of any length and returns a fixed number of bytes — 32 for SHA-256,
whether you feed it nothing or a gigabyte. It is deterministic, cheap to compute, and has no inverse.

Be clear about what it is **not**:

- **Not encryption.** There is no key and no decrypt. Information is destroyed, not hidden — a
  gigabyte compressed to 32 bytes cannot be recovered even in principle.
- **Not a checksum.** CRC32 detects accidental corruption and is trivial to forge deliberately. A
  cryptographic hash resists an adversary, which is a completely different bar.

Go exposes two shapes, and example
[3](examples/04-hash-functions/1-easy.md#3-the-hashhash-interface) covers both:

```go
sum := sha256.Sum256(data)     // convenience: returns [32]byte

h := sha256.New()              // the hash.Hash interface
h.Write(part1)                 // Write as many times as you like
h.Write(part2)
digest := h.Sum(nil)           // returns []byte
```

Two things in that interface surprise people. **`Sum(dst)` appends the digest to `dst`** — it does
not hash `dst`. That is why `Sum(nil)` is the idiom: it means "just give me the digest". And **`Sum`
does not reset the hasher**, so writing again after calling it continues the same stream rather than
starting a new one. Call `Reset()` when you want to start over.

Because `hash.Hash` is an `io.Writer`, `io.Copy` streams anything through it. Example
[4](examples/04-hash-functions/1-easy.md#4-streaming-a-large-input) hashes 10 MiB without ever
holding it in memory — which is how you hash a file, an HTTP response body, or a block you are
downloading:

```go
h := sha256.New()
if _, err := io.Copy(h, reader); err != nil { ... }
digest := h.Sum(nil)
```

### 2. The five properties

**Determinism.** Same input, same output, on every machine, forever. Without this, nodes cannot
agree, and topic 6 is entirely about the ways Go can break it for you.

**Preimage resistance.** Given a digest, you cannot find an input that produces it. This is what
protects a commitment — but note carefully what it does *not* promise. It says you cannot *invert*
the function; it says nothing about *guessing*. Example
[6](examples/04-hash-functions/1-easy.md#6-one-way-is-not-unguessable) recovers a hashed 4-digit PIN
in 4,272 tries. The hash was not broken; the input had no entropy.

**Second-preimage resistance.** Given a specific input, you cannot find a different one with the same
digest. This is what makes a hash usable as a fingerprint for a specific document.

**Collision resistance.** You cannot find *any* two inputs that collide. This is a strictly stronger
requirement, and it is what Merkle trees ([05](05-merkle-trees.md)) depend on — an attacker gets to
choose *both* sides. MD5 and SHA-1 are broken here and fine at preimage resistance, which is why "is
it broken?" is not a yes/no question.

**Avalanche.** One flipped input bit changes about half the output bits. Example
[2](examples/04-hash-functions/1-easy.md#2-avalanche) measures it: flipping each bit of the first
byte changes 117–133 of 256 output bits, averaging 125.9. The consequence is that a digest offers no
gradient — you cannot tell you are getting "warmer", which is what makes brute force the only attack.

### 3. SHA-256 vs SHA-3 vs Keccak-256

This is the single most common source of wasted hours in Ethereum development, and it comes from an
accident of history.

Keccak won the NIST SHA-3 competition in 2012. NIST then **changed the padding rule** before
finalising the standard in 2015. Ethereum had already shipped the original. So:

> **Ethereum's "sha3" is original Keccak-256, not NIST SHA3-256.** Same author, same core, different
> padding byte, completely different outputs.

The opcode is called `SHA3`, older documentation says "SHA-3", and Solidity's function is
`keccak256()`. Example [5](examples/04-hash-functions/1-easy.md#5-sha-256-sha3-256-and-keccak-256)
prints all three of `"abc"`:

```
sha256      ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
sha3-256    3a985da74fe225b2045c172d6bd390bd855f086e3e9d525b46bfe24511431532
keccak-256  4e03657aea45a94fc7d47ba826c8d667c0d1e6e33a64a036ec44f58fa12d6c45
```

In Go:

```go
sha256.Sum256(data)                  // Bitcoin, and general use
sha3.Sum256(data)                    // NIST SHA3-256 — almost never what you want here
sha3.NewLegacyKeccak256()            // original Keccak-256 — note "Legacy" in the name
crypto.Keccak256(data)               // go-ethereum's helper, returns []byte
crypto.Keccak256Hash(data)           // returns common.Hash ([32]byte) — usually preferable
```

Use SHA3-256 by mistake and everything verifies perfectly in your tests and fails against the chain.
The two families also differ structurally, which matters in topic 5: SHA-256 is Merkle–Damgård,
Keccak is a sponge.

### 4. Double hashing

Bitcoin never hashes once:

```go
func hash256(b []byte) []byte {          // block hashes, txids, Merkle nodes, base58check
    first := sha256.Sum256(b)
    second := sha256.Sum256(first[:])
    return second[:]
}

func hash160(b []byte) []byte {          // addresses (lessons 07, 36)
    first := sha256.Sum256(b)
    r := ripemd160.New()
    r.Write(first[:])
    return r.Sum(nil)
}
```

`HASH160` produces 20 bytes rather than 32 — shorter to type and transcribe, and 160 bits is still
far beyond reach. `HASH256` is SHA-256 twice.

The usual explanation for double hashing is "defence in depth", and the concrete benefit is
length-extension resistance: **the outer hash consumes a fixed 32 bytes, so there is no message to
extend**. Whether Satoshi had that specifically in mind is unclear, but it is the property it buys.
Example [7](examples/04-hash-functions/2-medium.md#7-hash256-and-hash160) implements both and checks
RIPEMD-160 against its published test vector.

### 5. Length-extension attacks

SHA-256, SHA-1 and MD5 are Merkle–Damgård constructions: they process fixed-size blocks, carrying an
internal state forward, and **the final digest is that internal state**. Which means anyone holding
the digest can resume hashing from it.

The consequence is a real, exploitable bug in a construction that looks entirely reasonable:

```go
func naiveMAC(secret, msg []byte) []byte {   // DO NOT DO THIS
    sum := sha256.Sum256(append(secret, msg...))
    return sum[:]
}
```

Given `H(secret‖msg)` and the *length* of the secret — often guessable, often published — an attacker
can compute a valid tag for `msg ‖ padding ‖ anything`, without ever learning the secret. Example
[8](examples/04-hash-functions/2-medium.md#8-breaking-a-naive-mac-by-length-extension) does exactly
that, reconstructing a hasher from the published digest via `encoding.BinaryUnmarshaler` and
appending `&to=attacker`. The server recomputes the MAC with the real secret and accepts it.

**Run that example.** It is the difference between knowing the phrase "length extension" and knowing
why you must never hand-roll a MAC.

The fix is HMAC, which nests two keyed hashes so the output is not a resumable state:

```go
mac := hmac.New(sha256.New, secret)
mac.Write(msg)
tag := mac.Sum(nil)
hmac.Equal(tag, expected)   // constant time — lesson 03, topic 9
```

Keccak's sponge construction is structurally immune: the digest is only part of the state, so there
is nothing to resume from. That is a genuine advantage — and still not a reason to hand-roll.
Example [9](examples/04-hash-functions/2-medium.md#9-hmac-and-the-sponge-alternative) shows both, and
the same HMAC construction verifies webhook signatures in
[51](51-wallet-integration-backends.md) and API requests in [32](32-key-management-signing.md).

### 6. Hashing structured data

A hash function takes bytes. The moment you hash anything richer than a byte string, you have made a
serialization decision — and if that decision is not canonical, two honest nodes compute different
digests and the network splits.

Go offers several convenient ways to get this wrong, and example
[10](examples/04-hash-functions/2-medium.md#10-nondeterminism-and-where-it-hides) demonstrates them:

- **Ranging over a map.** Go randomises map iteration order. Hashing a map by ranging over it gives a
  different digest almost every run. Sort the keys first.
- **`encoding/json` on a struct.** JSON *does* sort map keys, so hashing a marshalled map is stable
  within one Go version — but struct fields are emitted in **declaration order**, so reordering
  fields for readability silently changes your commitment. More broadly, JSON is not a canonical
  format at all: whitespace, escaping and number formatting are free choices.
- **`encoding/gob`.** Carries type information, depends on registration order, and is explicitly not
  stable across type changes.

The fix is an explicit preimage — bytes you constructed deliberately, in a documented order:

```go
func (a Account) preimage() []byte {
    buf := make([]byte, 0, 1+20+8+8)
    buf = append(buf, tagAccount)                        // 1. domain tag
    buf = append(buf, a.Address[:]...)                   // 2. fixed order
    buf = binary.BigEndian.AppendUint64(buf, a.Nonce)    // 3. fixed width, big-endian
    buf = binary.BigEndian.AppendUint64(buf, a.Balance)
    return buf
}
```

Five rules, from example
[11](examples/04-hash-functions/2-medium.md#11-canonical-preimages-and-domain-separation):

1. One domain tag per message type.
2. Fixed field order.
3. Fixed-width, big-endian integers ([03](03-bytes-encoding.md)).
4. Length-prefix anything variable — otherwise `("ab","c")` and `("a","bc")` produce identical bytes.
5. Never a map, never JSON, never gob.

**Domain separation** — rule 1 — deserves its own emphasis. Tagging each message type means a digest
of one kind can never be reinterpreted as another. The canonical case is a Merkle tree: without
distinct leaf and node tags, an internal node's 64-byte preimage can be presented as a leaf. That is
the Merkle second-preimage attack, and [05](05-merkle-trees.md) is where it bites.

Once several types need hashing, put the rule in one place. Example
[15](examples/04-hash-functions/3-hard.md#15-one-hashing-rule-for-many-types) uses an interface that
exposes the *preimage* rather than the hash, so no type can invent its own hashing:

```go
type Hashable interface{ preimage() []byte }

func HashOf(h Hashable) [32]byte { return sha256.Sum256(h.preimage()) }
```

### 7. Commitments

A commitment lets you publish a promise now and prove later what you promised:

```go
commitment := H(value ‖ nonce)     // publish this
// ... later ...
// reveal (value, nonce); anyone recomputes and checks
```

It needs two properties:

- **Hiding** — the commitment reveals nothing about the value. This comes from the **nonce**.
- **Binding** — you cannot later open it as a different value. This comes from collision resistance.

**The nonce is not optional.** Without it, hiding depends entirely on the value being unguessable —
and in practice values come from small, publicly known sets. Example
[13](examples/04-hash-functions/2-medium.md#13-the-nonce-is-not-optional) recovers a sealed auction
bid by enumerating every whole-dollar amount under 100,000. Thirty-two bytes from `crypto/rand` turns
that search from a loop into an impossibility. Example
[12](examples/04-hash-functions/2-medium.md#12-commit-and-reveal) does it properly, including a
domain tag and constant-time verification.

Commit–reveal is used for sealed-bid auctions and on-chain randomness, and it has a flaw no
cryptography fixes: **the last-mover problem**. Whoever reveals last has already seen everyone else's
values and can simply refuse to open — aborting the round if the outcome does not suit them. Real
designs add a forfeited deposit, a reveal deadline, or replace the scheme with a VRF
([53](53-oracles-randomness.md)).

Hash-based commitments are the simplest of a larger family. Pedersen commitments add homomorphism
(you can add commitments), and KZG commitments let you open a polynomial at any point in constant
size — the basis of EIP-4844 blobs. Both are in [39](39-zero-knowledge-proofs.md).

### 8. Hash as identifier

If content determines its hash, the hash can *be* the name. This idea shows up everywhere:

- **Block hashes and txids** — a block is identified by the hash of its header ([08](08-blocks-and-chain.md)).
- **Content addressing** — IPFS CIDs, git objects ([55](55-decentralized-storage.md)).
- **Merkle roots** — one 32-byte name for an entire set ([05](05-merkle-trees.md)).

The property that makes this powerful is that the name **proves** the content. Fetch a blob from an
untrusted mirror, hash it, compare: you cannot be handed the wrong bytes silently. Example
[17](examples/04-hash-functions/3-hard.md#17-a-content-addressed-blob-store) builds a store that
re-hashes on read and so turns bit rot, a bad cache and a malicious peer into the same detectable
error. Example [18](examples/04-hash-functions/3-hard.md#18-a-tamper-evident-log) chains entries so
the head hash commits to the entire history — which is a blockchain with one transaction per block.

**Truncation spends security.** Ethereum addresses are the last 20 bytes of a Keccak digest; ABI
function selectors are the first 4. The birthday bound says a collision takes roughly 2^(n/2) work:

| Output | Work to find *any* collision |
|---|---|
| 32 bits (4-byte selector) | ~2¹⁶ — about 65,000 hashes |
| 160 bits (address) | ~2⁸⁰ |
| 256 bits | ~2¹²⁸ |

Example [14](examples/04-hash-functions/2-medium.md#14-truncated-hashes-and-the-birthday-bound)
brute-forces a real 32-bit collision in under 59,000 hashes. So a 4-byte selector is an
**identifier, not a security boundary** — selector collisions are real and have been used against
proxy contracts ([46](46-upgradeable-contracts.md)). A 160-bit address at 2⁸⁰ is comfortable today
but is not the 2¹²⁸ people assume.

### 9. Performance

Hashing is usually not your bottleneck — except in the two places where it is completely dominant:
mining ([09](09-proof-of-work.md)) and bulk ingestion ([57](57-high-throughput-ingestion.md)).

**SHA-256 is several times faster than Keccak-256 in software**, because modern x86-64 and arm64 have
SHA-256 instructions in silicon and nothing equivalent for Keccak. Ethereum pays that cost on every
`KECCAK256` opcode, which is part of why it is priced at 30 gas plus 6 per word
([18](18-evm.md)). Example [20](examples/04-hash-functions/3-hard.md#20-sha-256-vs-keccak-256)
measures both on your own hardware.

The allocation pattern matters more than the algorithm choice, because it is the part you control.
Inside a loop, reuse everything:

```go
h := sha256.New()
digest := make([]byte, 0, sha256.Size)
for n := uint32(0); n < max; n++ {
    binary.BigEndian.PutUint32(buf[noncePos:], n)
    h.Reset()
    h.Write(buf)
    digest = h.Sum(digest[:0])   // append into a slice we already own
    ...
}
```

Example [19](examples/04-hash-functions/3-hard.md#19-allocation-free-hashing-in-a-mining-loop)
measures the naive version at 1 allocation per attempt and the tight one at 0. Over a billion
attempts that is a billion allocations you did not need. Measure with `testing.AllocsPerRun` (which
is deterministic) rather than guessing, and with `testing.B` plus `ReportAllocs` for throughput.

## Exercises

Write these in `practice/04-hash-functions/`.

1. **Check yourself against the standards.** Write a table-driven test covering SHA-256, SHA3-256,
   Keccak-256 and RIPEMD-160 against published test vectors for `""`, `"abc"` and one long input.
   Include the empty string — it catches padding bugs.
2. **Measure avalanche properly.** Flip every one of the first 64 input bits in turn and record how
   many output bits change. Report the mean and the range. Then do the same for CRC32 and explain the
   difference in one sentence.
3. **Mount the attack.** Reproduce example 8 from scratch: implement `mdPadding`, reconstruct a
   hasher from a digest, and forge a tag. Then vary the secret length from 8 to 40 bytes and confirm
   the attack works whenever you guess the length correctly — and fails when you do not.
4. **Fix it three ways.** Implement `H(secret‖msg)`, `H(msg‖secret)` and `hmac.New`. Write a test
   that forges the first. Explain in comments why the second resists length extension but is still
   not a MAC.
5. **Build a canonical encoder.** Define three message types with a `Hashable` interface. Write a
   fuzz test (`testing.F`) asserting that two structurally different values never produce the same
   preimage. Deliberately remove the length prefix from one variable field and let the fuzzer find
   the collision.
6. **A commitment scheme with tests.** Implement `Commit`/`Verify` with a 32-byte nonce. Test:
   honest reveal passes, wrong value fails, wrong nonce fails, and — with the nonce removed — a
   dictionary of 100,000 candidates recovers the value.
7. **A tamper-evident log.** Extend example 18: add `Proof(i int)` returning the entries needed to
   verify entry `i` against the head, and `VerifyProof`. This is a linear Merkle proof, and
   [05](05-merkle-trees.md) makes it logarithmic.
8. **Make it fast.** Write a benchmark for one hash of a 64-byte header. Optimise until
   `ReportAllocs` shows 0 allocations per operation. Then benchmark SHA-256 against Keccak-256 and
   record the ratio on your hardware.

## Best Practices & Pitfalls

- **Use Keccak-256 for Ethereum, never NIST SHA3-256.**
  *Why:* the names are nearly identical and the outputs are completely different. Your tests pass and
  the chain rejects everything. In Go the function is `sha3.NewLegacyKeccak256()` or
  `crypto.Keccak256` — if the word "Legacy" is not there, you have the wrong one.
- **Never hash a Go map by ranging over it, or a JSON blob.**
  *Why:* map iteration is randomised and JSON is not canonical. Two honest nodes compute different
  digests and disagree about the chain. Build an explicit preimage with a fixed field order.
- **Domain-separate every message type you hash.**
  *Why:* without a tag, a digest of one kind can be reinterpreted as another — which is exactly the
  Merkle second-preimage attack in [05](05-merkle-trees.md). One byte per type buys immunity.
- **Length-prefix variable-length fields.**
  *Why:* `("ab","c")` and `("a","bc")` concatenate identically. This is the same collision class as
  `abi.encodePacked` with adjacent dynamic types ([23](23-abi-encoding.md)), and it has caused real
  losses.
- **Never hand-roll a MAC. Use `hmac.New`.**
  *Why:* `H(secret‖msg)` on SHA-256 is forgeable by anyone who sees one tag — example 8 does it.
  The standard library's version is one line.
- **Compare digests with `hmac.Equal` or `subtle.ConstantTimeCompare` when they are secret.**
  *Why:* early-exit comparison leaks the matching prefix through timing ([03](03-bytes-encoding.md)).
  Plain `bytes.Equal` is correct for public data like block hashes.
- **A commitment without a random nonce is not hiding.**
  *Why:* preimage resistance protects against inversion, not enumeration. Auction bids, votes and
  PINs all come from small sets. 32 bytes from `crypto/rand`, every time.
- **Treat truncated hashes as identifiers, not security boundaries.**
  *Why:* the birthday bound is 2^(n/2). A 4-byte selector collides in ~65,000 tries — trivially, on a
  laptop, as example 14 shows.
- **Reuse the hasher and the output buffer inside a loop.**
  *Why:* `Reset()` plus `Sum(dst[:0])` is allocation-free. In a mining loop or a bulk indexer that is
  the difference between fine and unusable.
- **Remember `Sum` appends and does not reset.**
  *Why:* `h.Sum(existing)` returns `existing‖digest`, and writing after `Sum` continues the same
  stream. Both produce silently wrong digests rather than errors.

## Checklist

- [ ] I can compute SHA-256, SHA3-256, Keccak-256, HASH256 and HASH160 in Go.
- [ ] I know Ethereum means Keccak-256 when it says "sha3", and which Go function gives me that.
- [ ] I can state the five properties and say what each one protects.
- [ ] I can explain why preimage resistance does not make a low-entropy input safe.
- [ ] I understand why `Sum(dst)` appends and why `Sum` does not reset.
- [ ] I can stream a large input through a hasher with `io.Copy`.
- [ ] I can explain a length-extension attack and why HMAC prevents it.
- [ ] I can build a canonical preimage: domain tag, fixed order, fixed widths, length prefixes.
- [ ] I can implement commit–reveal, and explain why the nonce and the last-mover problem both matter.
- [ ] I can state the birthday bound and what a 4-byte selector is worth.
- [ ] I can write an allocation-free hashing loop and prove it with `testing.AllocsPerRun`.

## Resources

**Standards**

- NIST FIPS 180-4 (SHA-2): https://csrc.nist.gov/pubs/fips/180-4/upd1/final
- NIST FIPS 202 (SHA-3): https://csrc.nist.gov/pubs/fips/202/final
- RFC 2104 (HMAC): https://datatracker.ietf.org/doc/html/rfc2104
- The Keccak team on SHA-3 vs Keccak: https://keccak.team/keccak.html

**Go**

- `crypto/sha256`: https://pkg.go.dev/crypto/sha256
- `crypto/hmac`: https://pkg.go.dev/crypto/hmac
- `golang.org/x/crypto/sha3`: https://pkg.go.dev/golang.org/x/crypto/sha3
- `hash.Hash`: https://pkg.go.dev/hash#Hash

**Ethereum & Bitcoin**

- go-ethereum `crypto` package: https://pkg.go.dev/github.com/ethereum/go-ethereum/crypto
- Ethereum — cryptography overview: https://ethereum.org/en/developers/docs/
- Bitcoin developer reference — block hashing: https://developer.bitcoin.org/reference/block_chain.html

---

**Examples:** [`examples/04-hash-functions/`](examples/04-hash-functions/) — **20 runnable Go
programs** (🟢 6 easy · 🟡 8 medium · 🔴 6 hard). No chain, no node, no keys. Example 8 is a working
length-extension forgery against a naive MAC — if you have ever written `sha256(secret + message)`,
run that one first.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
