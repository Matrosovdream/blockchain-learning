# Step 06 — Keys & Digital Signatures · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

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

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | keys, the curve, choosing one, signing, verifying |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | key forms, EIP-191, r/s/v, recovery, encodings, entropy |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | malleability, nonce reuse, RFC 6979, Schnorr, a real verifier |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. A private key is one number](1-easy.md#1-a-private-key-is-one-number)
- [2. The curve, and why d to P is one-way](1-easy.md#2-the-curve-and-why-d-to-p-is-one-way)
- [3. Three curves, three schemes](1-easy.md#3-three-curves-three-schemes)
- [4. Sign and verify](1-easy.md#4-sign-and-verify)
- [5. Change anything, and it fails](1-easy.md#5-change-anything-and-it-fails)

### 🟡 [Medium](2-medium.md)

- [6. Compressed and uncompressed public keys](2-medium.md#6-compressed-and-uncompressed-public-keys)
- [7. Sign the hash, and prefix it](2-medium.md#7-sign-the-hash-and-prefix-it)
- [8. Inside a signature: r, s, v](2-medium.md#8-inside-a-signature-r-s-v)
- [9. Recovery: why there is no from field](2-medium.md#9-recovery-why-there-is-no-from-field)
- [10. The +27 offset](2-medium.md#10-the-27-offset)
- [11. Two encodings: compact and DER](2-medium.md#11-two-encodings-compact-and-der)
- [12. Entropy is the whole game](2-medium.md#12-entropy-is-the-whole-game)
- [13. Not every 32 bytes is a key](2-medium.md#13-not-every-32-bytes-is-a-key)

### 🔴 [Hard](3-hard.md)

- [14. Malleability: r and n minus s](3-hard.md#14-malleability-r-and-n-minus-s)
- [15. Nonce reuse hands over the private key](3-hard.md#15-nonce-reuse-hands-over-the-private-key)
- [16. RFC 6979: deriving k from the key](3-hard.md#16-rfc-6979-deriving-k-from-the-key)
- [17. Schnorr, and what linearity buys](3-hard.md#17-schnorr-and-what-linearity-buys)
- [18. A complete signature verifier](3-hard.md#18-a-complete-signature-verifier)

## The two that matter most

**[15. Nonce reuse hands over the private key](3-hard.md#15-nonce-reuse-hands-over-the-private-key)**
— not a description, a recovery. Two signatures sharing a nonce, four modular operations, and the
exact private key comes back out. This is what broke the PlayStation 3 and emptied Android Bitcoin
wallets in 2013.

**[16. RFC 6979](3-hard.md#16-rfc-6979-deriving-k-from-the-key)** — the fix, implemented from
scratch. It derives the nonce with HMAC and lands on the *exact* value go-ethereum uses
internally, which is how you know the implementation is right.

## The arc

1–3 — a key is a number, a public key is a point, and the curve you pick matters.  
4–5 — sign, verify, and see what breaks it.  
6–8 — key encodings, why you prefix before signing, and what is inside a signature.  
9–11 — recovery and the `v` byte, plus the two wire encodings.  
12–13 — entropy is the real weak point, and not every 32 bytes is a key.  
14 — the same signature has two valid forms, and what that cost Bitcoin.  
15–16 — the nonce catastrophe, and the deterministic derivation that ends it.  
17–18 — what replaces ECDSA, and the verifier you would actually ship.

---

*Lesson: [../../06-keys-signatures.md](../../06-keys-signatures.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
