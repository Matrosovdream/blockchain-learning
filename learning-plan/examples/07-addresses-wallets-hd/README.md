# Step 07 — Addresses, Encodings & HD Wallets · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                                # first time only
go get github.com/ethereum/go-ethereum@latest
go get github.com/tyler-smith/go-bip39@latest
go get github.com/btcsuite/btcd@v0.24.2            # pin: v0.26+ split into /v2 modules
go get github.com/btcsuite/btcd/btcutil@latest
# paste the example into main.go, then:
go run .
```

No chain and no node. Everything uses the **published Hardhat/anvil test mnemonic**
`test test … junk`, so all output reproduces exactly — and no real key is ever printed.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | address derivation, EIP-55, validation, address formats |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | BIP-39, BIP-32, derivation paths, watch-only wallets |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | the xpub leak, CREATE/CREATE2, vanity grinding, keystores |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. Public key to address, by hand](1-easy.md#1-public-key-to-address-by-hand)
- [2. EIP-55: computing the checksum](1-easy.md#2-eip-55-computing-the-checksum)
- [3. EIP-55: validating what a user pasted](1-easy.md#3-eip-55-validating-what-a-user-pasted)
- [4. Every 20-byte value is an address](1-easy.md#4-every-20-byte-value-is-an-address)
- [5. One key, five address formats](1-easy.md#5-one-key-five-address-formats)

### 🟡 [Medium](2-medium.md)

- [6. BIP-39: entropy becomes words](2-medium.md#6-bip-39-entropy-becomes-words)
- [7. BIP-39: words become a seed](2-medium.md#7-bip-39-words-become-a-seed)
- [8. BIP-32: the master key and the chain code](2-medium.md#8-bip-32-the-master-key-and-the-chain-code)
- [9. Deriving the accounts you have been using](2-medium.md#9-deriving-the-accounts-you-have-been-using)
- [10. Watch-only: addresses from an xpub](2-medium.md#10-watch-only-addresses-from-an-xpub)
- [11. Hardened and non-hardened derivation](2-medium.md#11-hardened-and-non-hardened-derivation)
- [12. One mnemonic, four Bitcoin address types](2-medium.md#12-one-mnemonic-four-bitcoin-address-types)
- [13. The wrong path gives a valid, empty wallet](2-medium.md#13-the-wrong-path-gives-a-valid-empty-wallet)

### 🔴 [Hard](3-hard.md)

- [14. The xpub leak](3-hard.md#14-the-xpub-leak)
- [15. CREATE: the address of your next deployment](3-hard.md#15-create-the-address-of-your-next-deployment)
- [16. CREATE2: an address before the contract](3-hard.md#16-create2-an-address-before-the-contract)
- [17. Vanity addresses, and how Profanity lost $160M](3-hard.md#17-vanity-addresses-and-how-profanity-lost-160m)
- [18. Storing a key: keystore v3](3-hard.md#18-storing-a-key-keystore-v3)

## Where the course loops back on itself

**[9. Deriving the accounts you have been using](2-medium.md#9-deriving-the-accounts-you-have-been-using)**
derives `m/44'/60'/0'/0/x` from the test mnemonic and produces the exact ten anvil accounts from
[lesson 02](../02-environment-setup/), including the private key you have been signing with since
[lesson 06](../06-keys-signatures/). Nothing was arbitrary — it was all one tree.

**[14. The xpub leak](3-hard.md#14-the-xpub-leak)** is the one to run twice. An xpub plus a single
non-hardened child private key recovers the parent key by subtraction — and then every sibling.
It is why BIP-44 hardens the first three levels.

## The arc

1–4 — how an Ethereum address is derived, and the checksum that is all that protects it.  
5 — the same key as five different addresses, and why the formats are not convertible.  
6–7 — entropy becomes words, words become a seed, and the passphrase that silently forks it.  
8–11 — the HD tree: master key, chain code, paths, watch-only xpubs, hardening.  
12–13 — the two ways a correct seed phrase shows you an empty wallet.  
14 — what non-hardened derivation actually costs.  
15–16 — addresses that belong to contracts, including ones that do not exist yet.  
17–18 — grinding a pretty address, and where the key ends up living.

---

*Lesson: [../../07-addresses-wallets-hd.md](../../07-addresses-wallets-hd.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
