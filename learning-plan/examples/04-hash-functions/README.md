# Step 04 — Cryptographic Hash Functions · Examples

A library of **20 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

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

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 20 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 | the API, avalanche, streaming, Keccak vs SHA-3, entropy |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–14 | Bitcoin's hashes, length extension, canonical preimages, commitments |
| 🔴 Hard | [3-hard.md](3-hard.md) | 15–20 | a hashing contract, Merkle roots, content addressing, performance |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. Your first hash](1-easy.md#1-your-first-hash)
- [2. Avalanche](1-easy.md#2-avalanche)
- [3. The hash.Hash interface](1-easy.md#3-the-hashhash-interface)
- [4. Streaming a large input](1-easy.md#4-streaming-a-large-input)
- [5. SHA-256, SHA3-256 and Keccak-256](1-easy.md#5-sha-256-sha3-256-and-keccak-256)
- [6. One-way is not unguessable](1-easy.md#6-one-way-is-not-unguessable)

### 🟡 [Medium](2-medium.md)

- [7. HASH256 and HASH160](2-medium.md#7-hash256-and-hash160)
- [8. Breaking a naive MAC by length extension](2-medium.md#8-breaking-a-naive-mac-by-length-extension)
- [9. HMAC, and the sponge alternative](2-medium.md#9-hmac-and-the-sponge-alternative)
- [10. Nondeterminism, and where it hides](2-medium.md#10-nondeterminism-and-where-it-hides)
- [11. Canonical preimages and domain separation](2-medium.md#11-canonical-preimages-and-domain-separation)
- [12. Commit and reveal](2-medium.md#12-commit-and-reveal)
- [13. The nonce is not optional](2-medium.md#13-the-nonce-is-not-optional)
- [14. Truncated hashes and the birthday bound](2-medium.md#14-truncated-hashes-and-the-birthday-bound)

### 🔴 [Hard](3-hard.md)

- [15. One hashing rule for many types](3-hard.md#15-one-hashing-rule-for-many-types)
- [16. A domain-separated Merkle root](3-hard.md#16-a-domain-separated-merkle-root)
- [17. A content-addressed blob store](3-hard.md#17-a-content-addressed-blob-store)
- [18. A tamper-evident log](3-hard.md#18-a-tamper-evident-log)
- [19. Allocation-free hashing in a mining loop](3-hard.md#19-allocation-free-hashing-in-a-mining-loop)
- [20. SHA-256 vs Keccak-256](3-hard.md#20-sha-256-vs-keccak-256)

## The one to run first

**[8. Breaking a naive MAC by length extension](2-medium.md#8-breaking-a-naive-mac-by-length-extension)**
is a working forgery, not a description of one. It reconstructs a SHA-256 hasher from a published
digest, appends the attacker's data, and produces a tag the server accepts — without ever learning
the secret. If you have ever written `sha256(secret + message)` and thought it was a MAC, run it.

## The arc

1–4 — what a hash is, how it behaves, and the Go API's two sharp edges.  
5 — which hash, and the Keccak/SHA-3 trap that costs Ethereum developers a day each.  
6 — one-way does not mean unguessable.  
7–9 — Bitcoin's double hashing, and the attack that explains why it exists.  
10–11 — why two nodes disagree, and the canonical preimage that fixes it.  
12–14 — commitments, the nonce that makes them hide, and what truncation costs.  
15–18 — one hashing rule, Merkle roots, content addressing, tamper-evident logs.  
19–20 — making it fast, and why Keccak is not.

---

*Lesson: [../../04-hash-functions.md](../../04-hash-functions.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
