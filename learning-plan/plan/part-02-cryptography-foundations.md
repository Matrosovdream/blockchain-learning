# Part 2 — Cryptography Foundations

The four primitives every chain is built from: hashes, Merkle trees, signatures and key derivation. You implement each one in Go before you ever see a block.

**Core spine.** Lessons 04–07 · 74 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 04 | [Cryptographic Hash Functions](#04-cryptographic-hash-functions) | 03 | 20 |
| 05 | [Merkle Trees & Proofs](#05-merkle-trees-proofs) | 04 | 18 |
| 06 | [Keys & Digital Signatures (ECDSA on secp256k1)](#06-keys-digital-signatures-ecdsa-on-secp256k1) | 04 | 18 |
| 07 | [Addresses, Encodings & HD Wallets](#07-addresses-encodings-hd-wallets) | 06 | 18 |

---

## 04 — Cryptographic Hash Functions

**Lesson file:** [../04-hash-functions.md](../04-hash-functions.md) · **Examples folder:** `../examples/04-hash-functions/`

| | |
|---|---|
| Prerequisites | [03](../03-bytes-encoding.md) |
| Unlocks | 05, 06, 08, 44, 55 |
| Examples | **20** — 🟢 6 easy, 🟡 8 medium, 🔴 6 hard |
| Topics | 9 |

*SHA-256, Keccak-256, the five properties, commitments, and hashing structured data deterministically*

### Goals

- Compute SHA-256, SHA-3 and Keccak-256 in Go and know which chain uses which.
- State the five properties of a cryptographic hash and what breaks when each fails.
- Use a hash as a commitment and as an identifier.
- Serialize a struct deterministically before hashing it.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What a hash function is**

   - Arbitrary-length input → fixed-length output, deterministic and cheap to compute.
   - Not encryption (no key, no inverse) and not a checksum (CRC is not collision-resistant).
   - The Go interface: `hash.Hash`, `Write`/`Sum`, and why `Sum(nil)` is the idiom.
   - Streaming a large input through `io.Copy` into a hasher.

**2. The five properties**

   - Determinism — same input, same output, forever, on every machine.
   - Preimage resistance — given h, you cannot find x. This is what protects a commitment.
   - Second-preimage resistance — given x, you cannot find x' ≠ x with the same hash.
   - Collision resistance — you cannot find *any* pair colliding. This is what protects Merkle trees.
   - Avalanche — one flipped input bit changes ~half the output bits.

**3. SHA-256 vs SHA-3 vs Keccak-256**

   - SHA-256 (`crypto/sha256`): Bitcoin's workhorse, Merkle–Damgård construction.
   - Keccak won the SHA-3 competition; NIST then changed the padding before standardising.
   - **Ethereum uses original Keccak-256, not NIST SHA3-256** — different outputs, constant source of bugs.
   - In Go: `golang.org/x/crypto/sha3.NewLegacyKeccak256()` or `crypto.Keccak256` from go-ethereum.

**4. Double hashing**

   - Bitcoin's HASH256 = SHA256(SHA256(x)) and HASH160 = RIPEMD160(SHA256(x)).
   - The historical rationale: length-extension resistance and belt-and-braces.
   - Where each appears: block hashes, txids, addresses, Merkle nodes.
   - Implementing both in Go and checking against known vectors.

**5. Length-extension attacks**

   - Why Merkle–Damgård hashes leak enough state to append to an unknown message.
   - The naive-MAC bug: `H(secret‖message)` is forgeable; this is why HMAC exists.
   - Keccak/SHA-3's sponge construction is immune.
   - `crypto/hmac` in Go, and when you need it (webhooks, API auth — lesson 51).

**6. Hashing structured data**

   - Canonical serialization first: fixed field order, fixed widths, no maps.
   - Why `encoding/json` and `encoding/gob` are unsafe here (key order, type registration, version drift).
   - Domain separation: prefix distinct message types so a leaf can never be read as a node.
   - A `Hashable` interface pattern in Go, with a `preimage() []byte` method.

**7. Commitments**

   - `commit = H(value ‖ nonce)`; publish commit now, reveal later — hiding and binding.
   - Why the nonce is mandatory: a low-entropy value is brute-forced instantly.
   - Commit–reveal for on-chain randomness and sealed-bid auctions, and its last-mover problem.
   - The link forward: Pedersen and KZG commitments in lesson 39.

**8. Hash as identifier**

   - Block hashes, txids, content addressing (IPFS CIDs — lesson 55).
   - 'The hash *is* the name': self-verifying references.
   - Truncated hashes: 160-bit addresses, 4-byte function selectors, and their collision budget.
   - The birthday bound — 256-bit output ⇒ ~2^128 work to collide; 4 bytes ⇒ trivially collided.

**9. Performance**

   - Hardware SHA extensions, and why Keccak is slower in software.
   - Hashing in a mining loop (lesson 09): allocation-free `Write` into a reused hasher.
   - `sha3.NewLegacyKeccak256()` allocation cost and reuse via `Reset()`.
   - Benchmarking with `testing.B` and `ReportAllocs`.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using SHA3-256 where Ethereum wants Keccak-256 — everything verifies locally and fails on-chain.
- Hashing a Go map or a JSON blob and getting different roots on different runs.
- Committing without a nonce, so the commitment is guessable.
- Truncating a hash for an identifier and then relying on collision resistance.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 20).

**🟢 Easy — 6 examples** *(one concept in isolation)*

- SHA-256 a string, print hex, flip one bit and diff the output.
- Keccak-256 vs SHA3-256 of the same input — show they differ.
- Stream a 10 MB reader through a hasher with `io.Copy`.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Implement HASH160 and HASH256 and check against Bitcoin test vectors.
- Break a naive `H(secret‖msg)` MAC by length extension, then fix it with HMAC.
- Commit–reveal: publish `H(secret‖nonce)`, reveal, verify; then brute-force a nonce-less commitment.
- Hash the same struct via JSON vs a canonical encoder across 100 runs and count distinct roots.

**🔴 Hard — 6 examples** *(real-shaped, multi-concept programs)*

- A domain-separated hasher (`0x00` leaves, `0x01` nodes) used by lesson 05's Merkle tree.
- Benchmark SHA-256 vs Keccak-256 throughput and allocations with a reused hasher.
- A content-addressed blob store keyed by hash, detecting corruption on read.

### Packages & tools

`crypto/sha256`, `crypto/hmac`, `golang.org/x/crypto/sha3`, `golang.org/x/crypto/ripemd160`, `github.com/ethereum/go-ethereum/crypto`, `hash`, `io`

### Resources to cite

- NIST FIPS 180-4 (SHA-2): https://csrc.nist.gov/pubs/fips/180-4/upd1/final
- NIST FIPS 202 (SHA-3): https://csrc.nist.gov/pubs/fips/202/final
- Go — crypto/sha256: https://pkg.go.dev/crypto/sha256
- go-ethereum crypto package: https://pkg.go.dev/github.com/ethereum/go-ethereum/crypto

---

## 05 — Merkle Trees & Proofs

**Lesson file:** [../05-merkle-trees.md](../05-merkle-trees.md) · **Examples folder:** `../examples/05-merkle-trees/`

| | |
|---|---|
| Prerequisites | [04](../04-hash-functions.md) |
| Unlocks | 08, 17, 39 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 8 |

*building a tree, the root as a commitment, inclusion proofs, and the traps (odd leaves, second preimage)*

### Goals

- Build a Merkle tree over a list of items in Go.
- Generate and verify an inclusion proof without holding the whole list.
- Explain why a block header only needs one 32-byte root.
- Know the odd-leaf duplication bug and the second-preimage defence.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The problem**

   - Prove one item belongs to a set of a million without transmitting the million.
   - Hashing the concatenation works but proves nothing about a single element.
   - The trade you want: O(1) commitment, O(log n) proof, O(n) construction.
   - Where it shows up: block headers, light clients, airdrops, rollup withdrawals.

**2. Construction**

   - Hash each leaf, pair adjacent hashes, hash the pair, repeat to a single root.
   - The tree in Go: a `[][]byte` per level, or a flat array with index math.
   - Building bottom-up vs recursive top-down, and which is easier to prove from.
   - The root commits to both content and *order*.

**3. Inclusion proofs**

   - The sibling path: one hash per level, plus a left/right bit each.
   - Verification as a fold from leaf to root — 12 lines of Go.
   - Proof size for 1M leaves: 20 hashes = 640 bytes. Work the numbers out loud.
   - What a valid proof proves — and what it does not (nothing about absence).

**4. Odd numbers of leaves**

   - Bitcoin duplicates the last hash — and that created CVE-2012-2459 (block malleability).
   - Promotion (carry the odd node up) as used by many libraries.
   - Padding to a power of two with a fixed sentinel.
   - Pick one, document it, and never mix implementations.

**5. Second-preimage resistance**

   - The attack: an internal node's 64-byte preimage can be reinterpreted as two leaves.
   - The defence: domain separation — `H(0x00‖leaf)` and `H(0x01‖left‖right)`.
   - RFC 6962 (Certificate Transparency) as the canonical example.
   - Why libraries that omit it are usually still 'fine' — and why you should not rely on that.

**6. Sorted-pair trees**

   - OpenZeppelin's convention: sort each pair before hashing, so proofs carry no direction bits.
   - Why this is convenient on-chain and how it changes the Go implementation.
   - Building an airdrop allowlist that a Solidity `MerkleProof.verify` will accept.
   - The duplicate-leaf caveat in sorted trees.

**7. Variants you will meet**

   - Merkle Patricia Trie (lesson 17) — key/value, not just membership.
   - Sparse Merkle trees — proofs of *absence*, used by rollups and SMT-based state.
   - Merkle Mountain Ranges — append-only with cheap updates.
   - Verkle trees (lesson 66) — vector commitments, much smaller proofs.

**8. Merkle proofs in production**

   - Airdrop/allowlist claims — the most common real use.
   - SPV: a light client verifies a tx is in a block from the header alone (lesson 64).
   - Rollup withdrawal proofs and L1↔L2 messaging (lesson 67).
   - Multiproofs: proving several leaves at once with a shared path.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reusing an unsorted-tree proof against a sorted-pair verifier (or vice versa) — silent verification failure.
- Forgetting domain separation and accepting an internal node as a leaf.
- Copying Bitcoin's duplicate-last-hash rule into a system where it enables forgery.
- Assuming a Merkle root proves anything about data you did not also commit to (ordering, uniqueness).

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Build a tree over 4 leaves and print every level.
- Compute a root, change one leaf, and show the root changes.
- Verify a hand-built 2-leaf proof.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Generate a proof for leaf 2 of 8, verify it, then corrupt one sibling and show failure.
- Handle 5 leaves three ways (duplicate / promote / pad) and compare the three roots.
- Implement domain separation and demonstrate the second-preimage attack against a version without it.
- Compute proof sizes for 2^10, 2^20 and 2^24 leaves.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- A sorted-pair allowlist tree whose proofs a Solidity `MerkleProof.verify` accepts (verify with `cast`).
- A multiproof that proves 3 leaves with a shared sibling set.
- A sparse Merkle tree over a 256-bit key space with a proof of absence.

### Packages & tools

`crypto/sha256`, `sort`, `bytes`, `github.com/ethereum/go-ethereum/crypto`

### Resources to cite

- RFC 6962 (Certificate Transparency): https://datatracker.ietf.org/doc/html/rfc6962
- Bitcoin developer guide — Merkle trees: https://developer.bitcoin.org/reference/block_chain.html
- OpenZeppelin MerkleProof: https://docs.openzeppelin.com/contracts/5.x/api/utils#MerkleProof
- CVE-2012-2459 discussion: https://bitcointalk.org/?topic=102395

---

## 06 — Keys & Digital Signatures (ECDSA on secp256k1)

**Lesson file:** [../06-keys-signatures.md](../06-keys-signatures.md) · **Examples folder:** `../examples/06-keys-signatures/`

| | |
|---|---|
| Prerequisites | [04](../04-hash-functions.md) |
| Unlocks | 07, 10, 19, 42, 50 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*private/public keys, the curve, signing, verifying, public-key recovery, malleability and nonce disasters*

### Goals

- Generate a secp256k1 keypair in Go and derive the public key.
- Sign a message hash, verify it, and recover the public key from the signature.
- Explain `v`, `r`, `s` and why low-`s` is enforced.
- Explain why a reused or predictable nonce leaks the private key.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Asymmetric cryptography in one page**

   - A secret scalar you keep, a public point you publish, a proof you know the secret.
   - Signing vs encryption — this course only ever signs.
   - What a signature binds: a specific message, by a specific key. Nothing else.
   - What it does not prove: intent, freshness, or that the signer read anything.

**2. Elliptic curves, intuitively**

   - Points on y² = x³ + 7 over a finite field, plus a group law you never hand-implement.
   - Generator point G; a private key is a scalar d; the public key is P = d·G.
   - The discrete-log problem: computing d from P is infeasible — that is the whole security.
   - Curve order n, cofactor 1, and why a private key must be in [1, n-1].

**3. secp256k1 vs the alternatives**

   - secp256k1: Bitcoin, Ethereum. Koblitz curve, chosen for fast endomorphism-based multiplication.
   - NIST P-256: what Go's `crypto/ecdsa` gives you by default — **wrong curve** for these chains.
   - ed25519: Solana, Cosmos, SSH — different scheme (EdDSA), deterministic by construction.
   - BLS12-381: Ethereum consensus signatures (lesson 42) — aggregatable, slower.

**4. Key generation in Go**

   - `crypto.GenerateKey()` from go-ethereum returns an `*ecdsa.PrivateKey` on secp256k1.
   - `crypto.FromECDSA` / `crypto.ToECDSA` for the 32-byte serialization.
   - Entropy: `crypto/rand` only. `math/rand` here is a fund-losing bug.
   - Uncompressed (0x04‖X‖Y, 65 bytes) vs compressed (0x02/0x03‖X, 33 bytes) public keys.

**5. You sign a hash, never a message**

   - ECDSA takes a 32-byte digest; the message must be hashed first.
   - EIP-191: `\x19Ethereum Signed Message:\n` + len + message — why the prefix exists.
   - Domain separation prevents a signed 'login challenge' from being replayed as a transaction.
   - Forward link: EIP-712 typed data (lesson 50) is the structured version of this idea.

**6. ECDSA internals**

   - Pick nonce k, compute R = k·G, r = R.x, s = k⁻¹(z + r·d) mod n.
   - Verification recomputes R from (r, s, z, P) — no secret needed.
   - Public-key recovery: from (r, s, z) you can recover up to 4 candidate points; `v` selects one.
   - This is why Ethereum transactions have no `from` field — it is recovered.

**7. The go-ethereum signature API**

   - `crypto.Sign(hash, key)` → 65 bytes `r‖s‖v` where v ∈ {0,1}.
   - `crypto.Ecrecover(hash, sig)` → uncompressed pubkey bytes; `crypto.SigToPub` → `*ecdsa.PublicKey`.
   - `crypto.VerifySignature(pubkey, hash, sig[:64])` — note it takes 64 bytes, not 65.
   - The +27 offset: Ethereum transactions historically use v ∈ {27,28}; raw signing uses {0,1}.

**8. Malleability**

   - (r, s) and (r, n−s) are both valid signatures over the same message.
   - Consequence: signature bytes are not a unique identifier — Bitcoin's old txid malleability.
   - EIP-2 enforces low-`s` on Ethereum; BIP-62/BIP-146 on Bitcoin.
   - Practical rule: never key anything on a signature; key on the hash of the signed payload.

**9. Nonce catastrophes**

   - Reuse k across two signatures ⇒ d falls out with algebra. Show the derivation.
   - Sony PlayStation 3 (2010) and the Android SecureRandom Bitcoin thefts (2013).
   - RFC 6979 deterministic nonces: k = HMAC(d, z), removing the RNG from the critical path.
   - Biased nonces leak too — lattice attacks recover keys from a few bits per signature.

**10. What replaces ECDSA**

   - Schnorr/BIP-340 (Taproot): linear, so signatures aggregate (lesson 42).
   - BLS: aggregate thousands of signatures into one (Ethereum attestations).
   - Threshold/MPC signing: no single machine ever holds the key (lesson 43).
   - Post-quantum: why none of this survives a CRQC, and the current state of the art.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using Go's default P-256 curve instead of secp256k1 — valid signatures that no chain accepts.
- Signing a raw message instead of its hash, or forgetting the EIP-191 prefix.
- Passing a 65-byte signature to `VerifySignature`, which wants 64.
- Confusing v ∈ {0,1} (raw) with v ∈ {27,28} (transaction) — off-by-27 recovers the wrong address.
- Any RNG other than `crypto/rand`.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Generate a key; print the private key hex and both public key forms.
- Sign a Keccak hash and verify it.
- Flip one byte of the message and show verification fails.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Recover the public key from a signature and prove it equals the signer's.
- Show the +27 offset by recovering with the wrong v and getting a different address.
- Sign with the EIP-191 prefix and verify against what `cast wallet sign` produces.
- Compress and decompress a public key and confirm round-trip equality.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Demonstrate malleability: derive (r, n−s) and show a raw verifier accepts both.
- Recover a private key from two signatures that reused the same nonce k.
- Implement RFC 6979 deterministic k and show two signings of the same message are byte-identical.

### Packages & tools

`crypto/rand`, `crypto/ecdsa`, `crypto/elliptic`, `math/big`, `github.com/ethereum/go-ethereum/crypto`, `github.com/btcsuite/btcd/btcec/v2`

### Resources to cite

- SEC 2 (secp256k1 parameters): https://www.secg.org/sec2-v2.pdf
- RFC 6979 (deterministic ECDSA): https://datatracker.ietf.org/doc/html/rfc6979
- EIP-191 signed data standard: https://eips.ethereum.org/EIPS/eip-191
- EIP-2 (homestead, low-s): https://eips.ethereum.org/EIPS/eip-2

---

## 07 — Addresses, Encodings & HD Wallets

**Lesson file:** [../07-addresses-wallets-hd.md](../07-addresses-wallets-hd.md) · **Examples folder:** `../examples/07-addresses-wallets-hd/`

| | |
|---|---|
| Prerequisites | [06](../06-keys-signatures.md) |
| Unlocks | 11 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 9 |

*Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation*

### Goals

- Derive an Ethereum address from a public key by hand, in Go.
- Implement and verify the EIP-55 mixed-case checksum.
- Generate a BIP-39 mnemonic and derive keys along a BIP-44 path.
- Explain xpub/xprv, hardened derivation, and the xpub-plus-child-key leak.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Ethereum address derivation**

   - address = last 20 bytes of Keccak-256 of the 64-byte uncompressed pubkey (no 0x04 prefix).
   - Addresses are *derived*, never registered — every 20-byte value is a valid address.
   - `crypto.PubkeyToAddress` in Go, and doing it manually to prove you understand it.
   - Sending to a wrong-but-valid address is unrecoverable; this is why checksums exist.

**2. EIP-55 checksums**

   - Hash the lowercase hex address; uppercase hex letter i when nibble i of the hash ≥ 8.
   - ~99.986% chance of catching a single-character typo.
   - Verification: recompute and compare; wallets reject a mismatched mixed-case address.
   - All-lowercase addresses are legal and uncheckable — treat them with suspicion.

**3. Bitcoin address formats**

   - HASH160 = RIPEMD160(SHA256(pubkey)); version byte; base58check with a 4-byte double-SHA checksum.
   - P2PKH (`1...`), P2SH (`3...`), bech32 P2WPKH (`bc1q...`), bech32m P2TR (`bc1p...`).
   - Testnet/regtest version bytes and prefixes.
   - Why bech32 exists: case-insensitive, better error detection, QR-friendly.

**4. BIP-39 mnemonics**

   - Entropy (128–256 bits) + checksum bits → 12–24 words from a fixed 2048-word list.
   - Seed = PBKDF2-HMAC-SHA512(mnemonic, "mnemonic"+passphrase, 2048 iterations) → 512 bits.
   - The optional passphrase ('25th word') creates an entirely separate wallet — and a support nightmare.
   - Wordlist properties: first 4 letters unique, no confusable pairs; language variants.

**5. BIP-32 hierarchical deterministic keys**

   - Master key from seed via HMAC-SHA512("Bitcoin seed", seed) → key ‖ chain code.
   - Child derivation CKDpriv/CKDpub; the chain code is what makes it deterministic and unlinkable.
   - Extended keys: xprv/xpub serialization (version, depth, fingerprint, index, chain code, key).
   - Index space: 0..2³¹−1 normal, 2³¹..2³²−1 hardened.

**6. Hardened vs non-hardened derivation**

   - Non-hardened lets you derive child *public* keys from an xpub alone — great for watch-only wallets.
   - The leak: xpub + any non-hardened child *private* key ⇒ the parent private key. Show the algebra.
   - Therefore: harden account-level nodes, leave only the last two levels non-hardened.
   - Notation: `'` or `h` means hardened, e.g. `m/44'/60'/0'`.

**7. BIP-44 and friends**

   - `m / purpose' / coin_type' / account' / change / index`.
   - Ethereum: `m/44'/60'/0'/0/x`. Bitcoin legacy: `m/44'/0'/0'/0/x`.
   - BIP-49 (P2SH-SegWit, `49'`), BIP-84 (native SegWit, `84'`), BIP-86 (Taproot, `86'`).
   - SLIP-44 coin types, and why a wrong path silently produces a valid but empty wallet.

**8. Contract and vanity addresses**

   - CREATE: address = keccak(rlp([sender, nonce]))[12:] — predictable from the deployer's nonce.
   - CREATE2: keccak(0xff ‖ sender ‖ salt ‖ keccak(initcode))[12:] — counterfactual deployment.
   - Vanity address grinding, and the Profanity vulnerability that drained millions (2022).
   - Address ≠ account: an address with no code today can have code tomorrow.

**9. Storing keys**

   - Web3 keystore v3: scrypt/pbkdf2 KDF + AES-128-CTR + MAC; `accounts/keystore` in Go (lesson 44).
   - Hardware wallets: the key never leaves the device; the host sends a hash and gets a signature.
   - Seed phrase custody: metal backups, Shamir splits, and the inheritance problem.
   - The rule this course enforces: test keys only, always labelled.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Deriving from the 65-byte pubkey including the 0x04 prefix — produces a wrong address.
- Mixing derivation paths between wallets and concluding 'my funds are gone'.
- Exposing an xpub together with any non-hardened child private key.
- Trusting a lowercase address from user input without an EIP-55 check.
- Assuming a CREATE2 address is safe because it has no code yet.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Public key → Keccak → last 20 bytes → address; compare with `crypto.PubkeyToAddress`.
- Implement EIP-55 checksumming for a known address.
- Validate a mixed-case address and reject a corrupted one.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Generate a BIP-39 mnemonic, print entropy, words and seed.
- Derive `m/44'/60'/0'/0/0` and match what MetaMask shows for the same phrase.
- Derive 5 sequential receive addresses from an xpub with no private key.
- Build a P2PKH base58check address and a bech32 P2WPKH address from the same pubkey.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Demonstrate the xpub leak: recover a parent private key from an xpub and one non-hardened child key.
- Compute a CREATE and a CREATE2 address in Go and verify both against a deployment on `anvil`.
- Grind a 4-hex-character vanity address and report the attempts and time taken.

### Packages & tools

`github.com/ethereum/go-ethereum/crypto`, `github.com/tyler-smith/go-bip39`, `github.com/btcsuite/btcd/btcutil/hdkeychain`, `golang.org/x/crypto/pbkdf2`

### Resources to cite

- EIP-55 address checksum: https://eips.ethereum.org/EIPS/eip-55
- BIP-32 HD wallets: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP-39 mnemonics: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
- BIP-44 derivation paths: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- EIP-1014 CREATE2: https://eips.ethereum.org/EIPS/eip-1014

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
