# Part 10 — Cryptography, Deeper

The signature schemes and key-protection machinery the core plan only previewed. Take any time after [06](../06-keys-signatures.md); [43](../43-multisig-mpc-threshold.md) pairs with [32](../32-key-management-signing.md).

**Extension.** Beyond the core 01–41 spine. Lessons 42–44 · 45 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 42 | [Schnorr, BLS & Aggregate Signatures](#42-schnorr-bls-aggregate-signatures) | 06 | 15 |
| 43 | [Multisig, MPC & Threshold Signing](#43-multisig-mpc-threshold-signing) | 32, 42 | 14 |
| 44 | [Symmetric Crypto, KDFs & Encryption at Rest](#44-symmetric-crypto-kdfs-encryption-at-rest) | 04 | 16 |

---

## 42 — Schnorr, BLS & Aggregate Signatures

**Lesson file:** [../42-schnorr-bls-aggregate.md](../42-schnorr-bls-aggregate.md) · **Examples folder:** `../examples/42-schnorr-bls-aggregate/`

| | |
|---|---|
| Prerequisites | [06](../06-keys-signatures.md) |
| Unlocks | 43 |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 8 |

*the signature schemes that replaced ECDSA — linearity, aggregation, and where each is used*

### Goals

- Implement and verify a BIP-340 Schnorr signature in Go.
- Explain why Schnorr's linearity enables key and signature aggregation.
- Use BLS signatures and aggregate many into one.
- Say precisely which chain uses which scheme and why.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why ECDSA was replaced**

   - ECDSA is non-linear, so signatures cannot be combined.
   - It is malleable (lesson 06), it needs a nonce with catastrophic failure modes, and it has no security proof under standard assumptions.
   - Schnorr is simpler, provably secure, non-malleable and linear.
   - It was patented until 2008 — that is the only reason Bitcoin did not start with it.

**2. Schnorr signatures**

   - s = k + H(R ‖ P ‖ m)·d, signature = (R, s). Verify: s·G == R + H(R‖P‖m)·P.
   - Linearity: sums of keys and sums of signatures still verify. This is the whole point.
   - Implement sign and verify from scratch over secp256k1 with `btcec`.
   - Compare the code with your ECDSA implementation from lesson 06.

**3. BIP-340**

   - 64-byte signatures, x-only public keys (32 bytes), implicit even-Y.
   - Tagged hashes: `H_tag(m) = SHA256(SHA256(tag) ‖ SHA256(tag) ‖ m)` — domain separation done right.
   - Deterministic nonce derivation with auxiliary randomness.
   - The batch-verification speedup, and why it matters for full nodes.

**4. Key and signature aggregation**

   - Naive aggregation is broken: the rogue-key attack, explained and demonstrated.
   - MuSig2: two rounds, nonce commitments, and key-aggregation coefficients that stop rogue keys.
   - What aggregation buys on Bitcoin: an n-of-n multisig looks like a single-key spend.
   - Privacy and fee benefits in Taproot (lesson 36).

**5. BLS signatures**

   - Pairing-based: e(σ, G₂) == e(H(m), P). Signatures are a single curve point.
   - **Non-interactive aggregation**: add signatures together, no coordination round.
   - The trade: pairings are slow, and verification of an aggregate over *distinct* messages is expensive.
   - The rogue-key defence here: proof of possession, which Ethereum requires at deposit.

**6. BLS12-381**

   - The curve Ethereum's consensus layer uses; two groups G₁ (48-byte) and G₂ (96-byte).
   - Ethereum's choice: public keys in G₁, signatures in G₂.
   - In Go: `gnark-crypto`'s `ecc/bls12-381`, or `github.com/herumi/bls-eth-go-binary`.
   - EIP-2537 precompiles making BLS verification cheap on-chain (lesson 65).

**7. Where each scheme lives**

   - ECDSA/secp256k1: Ethereum EOAs, Bitcoin legacy/SegWit.
   - Schnorr/secp256k1: Bitcoin Taproot key-path spends.
   - BLS12-381: Ethereum consensus attestations and the deposit contract.
   - ed25519: Solana, Cosmos, NEAR, most non-EVM chains.
   - A one-page table you will refer back to constantly.

**8. Threshold and ring signatures**

   - Threshold Schnorr (FROST) — t-of-n producing one indistinguishable signature (lesson 43).
   - Ring signatures and their use in Monero, in one paragraph.
   - Adaptor signatures and atomic swaps, in one paragraph.
   - Where to read further without going down a rabbit hole.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Naive key aggregation without MuSig's coefficients — the rogue-key attack steals the funds.
- Reusing a nonce in Schnorr; the same catastrophe as ECDSA, with simpler algebra.
- Mixing up G₁ and G₂ in BLS and getting a valid-looking but unverifiable signature.
- Accepting a BLS public key without a proof of possession.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Sign and verify a BIP-340 Schnorr signature with `btcec`.
- Implement a tagged hash and check it against the BIP-340 test vectors.
- Show that (R, s) is 64 bytes vs ECDSA's 65.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Implement Schnorr sign/verify from scratch and check against the BIP-340 vectors.
- Demonstrate linearity: aggregate two keys and two signatures naively and verify.
- Demonstrate the rogue-key attack against that naive aggregation.
- Sign and verify with BLS12-381 using `gnark-crypto`.
- Aggregate 100 BLS signatures over the same message into one and verify it.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Implement a two-round MuSig2 signing session between two parties and verify the result as a single key.
- Batch-verify 1000 Schnorr signatures and benchmark against verifying them individually.
- Verify an aggregate BLS signature over *distinct* messages and measure the cost difference.

### Packages & tools

`github.com/btcsuite/btcd/btcec/v2/schnorr`, `github.com/consensys/gnark-crypto/ecc/bls12-381`, `crypto/sha256`, `math/big`

### Resources to cite

- BIP-340 (Schnorr): https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- MuSig2 paper: https://eprint.iacr.org/2020/1261
- BLS signatures IETF draft: https://datatracker.ietf.org/doc/draft-irtf-cfrg-bls-signature/
- EIP-2537 (BLS12-381 precompiles): https://eips.ethereum.org/EIPS/eip-2537
- gnark-crypto: https://github.com/Consensys/gnark-crypto

---

## 43 — Multisig, MPC & Threshold Signing

**Lesson file:** [../43-multisig-mpc-threshold.md](../43-multisig-mpc-threshold.md) · **Examples folder:** `../examples/43-multisig-mpc-threshold/`

| | |
|---|---|
| Prerequisites | [32](../32-key-management-signing.md), [42](../42-schnorr-bls-aggregate.md) |
| Unlocks | — |
| Examples | **14** — 🟢 4 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 8 |

*removing the single point of failure — on-chain multisig, Shamir sharing, and threshold signatures*

### Goals

- Compare on-chain multisig, secret sharing and threshold signing on their real trade-offs.
- Implement Shamir's Secret Sharing in Go.
- Explain how a t-of-n threshold signature is produced without ever reconstructing the key.
- Interact with a Safe multisig from Go.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The problem**

   - One key = one point of catastrophic, irreversible failure.
   - Three different answers: split the *authorization* (multisig), split the *key at rest* (Shamir), split the *signing* (MPC).
   - They solve different problems and are frequently confused.
   - Which to use for: a treasury, an automated hot wallet, a cold backup.

**2. On-chain multisig**

   - A contract that executes only with m-of-n owner approvals. Safe (formerly Gnosis Safe) is the standard.
   - Transparent, auditable, and enforceable by the chain — but it costs gas and is chain-specific.
   - The Safe execution flow: `getTransactionHash`, off-chain EIP-712 signatures, `execTransaction`.
   - Owner management, threshold changes, and modules/guards as extension points.
   - Interacting from Go: build the hash, collect signatures, sort by owner address (required!), execute.

**3. Shamir's Secret Sharing**

   - Split a secret into n shares; any t reconstruct it, t−1 reveal nothing (information-theoretically).
   - The math: a degree t−1 polynomial over a finite field; the secret is f(0); shares are points.
   - Lagrange interpolation to reconstruct. Implement both directions in Go.
   - **The weakness**: reconstruction puts the whole key on one machine, at one moment.
   - SLIP-39 as the standardised mnemonic form.

**4. Threshold signatures (MPC/TSS)**

   - t-of-n parties jointly produce one signature; the private key **never exists** anywhere, ever.
   - Distributed key generation (DKG): each party ends with a share and nobody has the whole.
   - The output is an ordinary signature — indistinguishable on-chain, so it works on any chain.
   - Proactive resharing: refresh shares periodically so an attacker must compromise t within one window.

**5. Threshold ECDSA vs threshold Schnorr**

   - ECDSA's non-linearity makes threshold signing hard: GG18/GG20, CGGMP, multi-round protocols.
   - Known implementation vulnerabilities in threshold ECDSA — this is expert territory.
   - Threshold Schnorr (FROST) is far simpler because Schnorr is linear (lesson 42).
   - Practical advice: use an audited library, never roll your own.

**6. Comparing the three**

   - On-chain multisig: transparent, gas cost, chain-specific, policy on-chain.
   - Shamir: cheap, chain-agnostic, but has a reconstruction moment.
   - MPC: chain-agnostic, no reconstruction, but complex and needs a trusted implementation.
   - A decision table by use case, and the common hybrid (MPC hot, multisig treasury, Shamir cold backup).

**7. Operational design**

   - Signer diversity: different people, machines, locations, and cloud providers.
   - Ceremony design for key generation, and the audit trail.
   - Recovery: what happens when a signer is lost, compromised, or leaves the company.
   - Approval workflows, timelocks and the human circuit breaker (lesson 32).

**8. Go implementations**

   - `hashicorp/vault/shamir` or a from-scratch implementation for learning.
   - MPC libraries: `bnb-chain/tss-lib`, `taurusgroup/multi-party-sig` — read before you trust.
   - Safe's API and contracts from Go via generated bindings.
   - Testing: property tests that any t shares reconstruct and any t−1 do not.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reconstructing a Shamir secret on an internet-facing machine.
- Unsorted owner signatures in a Safe `execTransaction` call — it reverts, confusingly.
- Implementing threshold ECDSA yourself; the published attacks are subtle and real.
- Storing t shares in the same trust domain, which reduces it to a single key.
- No resharing schedule, so share compromise accumulates forever.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 14).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Read a Safe's owners and threshold from Go.
- Split a 32-byte secret into 5 Shamir shares.
- Reconstruct the secret from any 3 of them.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Prove that 2 shares reveal nothing: show the reconstructed value is uniformly distributed.
- Build a Safe transaction hash and sign it with two owner keys.
- Sort signatures by owner address and execute the Safe transaction on `anvil`.
- Property-test Shamir: for random (t, n), any t shares reconstruct and any t−1 fail.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Run a FROST-style threshold Schnorr signing between 2 of 3 parties and verify the single output signature.
- Implement proactive resharing: refresh all shares without changing the secret.
- Compare on-chain gas for a 3-of-5 Safe execution against a single-key transaction.

### Packages & tools

`github.com/hashicorp/vault/shamir`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `crypto/rand`, `math/big`

### Resources to cite

- Shamir (1979) How to Share a Secret: https://dl.acm.org/doi/10.1145/359168.359176
- SLIP-39: https://github.com/satoshilabs/slips/blob/master/slip-0039.md
- FROST: https://eprint.iacr.org/2020/852
- Safe contracts: https://docs.safe.global/advanced/smart-account-overview
- GG20 threshold ECDSA: https://eprint.iacr.org/2020/540

---

## 44 — Symmetric Crypto, KDFs & Encryption at Rest

**Lesson file:** [../44-symmetric-crypto-at-rest.md](../44-symmetric-crypto-at-rest.md) · **Examples folder:** `../examples/44-symmetric-crypto-at-rest/`

| | |
|---|---|
| Prerequisites | [04](../04-hash-functions.md) |
| Unlocks | — |
| Examples | **16** — 🟢 4 easy, 🟡 8 medium, 🔴 4 hard |
| Topics | 10 |

*AES-GCM, ChaCha20, argon2/scrypt, envelope encryption, and the keystore format decoded*

### Goals

- Encrypt and authenticate data correctly in Go with AEAD.
- Choose and parameterise a password KDF.
- Implement and decode a Web3 keystore v3 file by hand.
- Design envelope encryption with a KMS.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Symmetric vs asymmetric**

   - This course has only signed so far; now you encrypt. Different goal, different tools.
   - Confidentiality vs integrity vs authenticity — and why you always need all three.
   - Where encryption at rest belongs in a chain service: keystores, backups, PII, session data.
   - What encryption does *not* solve: an attacker with your running process.

**2. AEAD is the only acceptable default**

   - Authenticated Encryption with Associated Data: ciphertext plus a tag that detects tampering.
   - AES-256-GCM (`crypto/aes` + `cipher.NewGCM`) and ChaCha20-Poly1305 (`x/crypto/chacha20poly1305`).
   - Which to pick: AES-GCM when the CPU has AES-NI, ChaCha20 otherwise (mobile, embedded).
   - Never CBC, never ECB, never encrypt-without-MAC. Unauthenticated encryption is a vulnerability.

**3. Nonces**

   - GCM nonces are 12 bytes and **must never repeat** under the same key — reuse breaks confidentiality *and* authenticity.
   - Random nonces are safe up to ~2³² messages per key; counters are safe if you can guarantee no reuse.
   - Store the nonce alongside the ciphertext; it is not secret.
   - Key rotation as the answer to nonce exhaustion.

**4. Associated data**

   - Bind ciphertext to its context: record id, version, tenant — authenticated but not encrypted.
   - Prevents an attacker moving a valid ciphertext to a different row.
   - A concrete example: binding an encrypted key blob to its address.
   - Forgetting AAD is a silent, common design flaw.

**5. Password-based KDFs**

   - scrypt (memory-hard, used by keystore v3), argon2id (the modern recommendation), PBKDF2 (legacy, weak).
   - Parameters: argon2id time/memory/threads; scrypt N/r/p. Benchmark on *your* hardware and target ~250ms.
   - Salts: 16+ random bytes, unique per secret, stored in the clear.
   - Never use a plain hash (SHA-256) for a password. Never.

**6. HKDF and key hierarchies**

   - Derive many purpose-specific keys from one master with HKDF-Expand and an `info` label.
   - Domain separation so an encryption key is never also a MAC key.
   - `golang.org/x/crypto/hkdf` in Go.
   - Versioned key derivation so you can rotate without re-deriving everything.

**7. The Web3 keystore v3 file, decoded**

   - The JSON fields: `crypto.kdf`, `kdfparams`, `cipher`, `cipherparams.iv`, `ciphertext`, `mac`.
   - derivedKey = scrypt(password, salt, N, r, p, 32); encKey = derivedKey[0:16]; macKey = derivedKey[16:32].
   - ciphertext = AES-128-CTR(encKey, iv, privateKey); mac = keccak256(macKey ‖ ciphertext).
   - Implement decryption from scratch in Go and unlock a real keystore file with it.
   - Note the design weakness: CTR + MAC-then-encrypt-order quirks; modern code would use AEAD.

**8. Envelope encryption**

   - Generate a random data key, encrypt data with it, encrypt the data key with a KMS master key.
   - Store the wrapped data key next to the ciphertext; decrypt requires a KMS call.
   - Why: bounded KMS calls, cheap rotation, and no plaintext master key anywhere.
   - Implement it in Go against a KMS interface with a local fake for tests.

**9. Key rotation and secret lifecycle**

   - Versioned key ids in the ciphertext header; decrypt with any version, encrypt with the newest.
   - Rewrap on read vs a background migration.
   - Deletion as crypto-shredding: destroy the key, the data is gone.
   - Auditing decryptions — every unwrap is an event worth logging.

**10. Go pitfalls**

   - `crypto/rand` only, and always check its error.
   - Comparing MACs with `hmac.Equal`/`subtle.ConstantTimeCompare`, never `==`.
   - Not being able to reliably zero memory in Go — accept it and reduce exposure time instead.
   - `defer` on a decrypted buffer does not guarantee anything; keep secrets in scope as briefly as possible.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reusing a GCM nonce — the single most destructive mistake in symmetric crypto.
- Encrypting without authenticating (CBC, CTR alone).
- Using SHA-256 or a low-iteration PBKDF2 for a password.
- Comparing MACs with `bytes.Equal` and leaking timing.
- Copy-pasting KDF parameters from a blog post without benchmarking your own hardware.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Encrypt and decrypt a message with AES-256-GCM.
- Derive a key from a password with argon2id and time it.
- Generate a random salt and nonce correctly with `crypto/rand`.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Show that a tampered ciphertext fails to decrypt (the tag catches it).
- Bind associated data and show that changing it breaks decryption.
- Demonstrate GCM nonce reuse leaking the XOR of two plaintexts.
- Benchmark scrypt and argon2id parameter sets to hit a 250ms target.
- Derive three purpose-separated keys from one master with HKDF.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Decrypt a real Web3 keystore v3 file from scratch, without `accounts/keystore`.
- Write a keystore v3 file yourself and unlock it with geth to prove compatibility.
- Implement envelope encryption with a KMS interface, a local fake, and key-version rotation.

### Packages & tools

`crypto/aes`, `crypto/cipher`, `crypto/rand`, `crypto/subtle`, `crypto/hmac`, `golang.org/x/crypto/argon2`, `golang.org/x/crypto/scrypt`, `golang.org/x/crypto/hkdf`, `golang.org/x/crypto/chacha20poly1305`

### Resources to cite

- Go crypto/cipher (AEAD): https://pkg.go.dev/crypto/cipher#AEAD
- RFC 9106 (Argon2): https://datatracker.ietf.org/doc/html/rfc9106
- RFC 5869 (HKDF): https://datatracker.ietf.org/doc/html/rfc5869
- Web3 Secret Storage: https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/
- NIST SP 800-38D (GCM): https://csrc.nist.gov/pubs/sp/800/38/d/final

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
