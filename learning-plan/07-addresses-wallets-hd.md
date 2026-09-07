# 07 — Addresses, Encodings & HD Wallets

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#07-addresses-encodings-hd-wallets)

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations *(final lesson)* |
| **Prerequisites** | [06](06-keys-signatures.md) |
| **Unlocks** | 11 |
| **Examples** | [18](examples/07-addresses-wallets-hd/) (🟢 5 · 🟡 8 · 🔴 5) |

*Ethereum address derivation, EIP-55 checksums, base58check, BIP-39 mnemonics, BIP-32/44 derivation.*

The last lesson of Part 2, and the one where the cryptography becomes a **wallet**. You have a key
pair; this lesson turns it into an address people can send to, a phrase you can write down, and a
tree of billions of keys derived from that one phrase.

It also closes a loop. The test key you have been signing with since lesson 06, and the ten funded
accounts from lesson 02, are all derived here from twelve published words — and example 9 shows the
derivation producing exactly those addresses.

## Goals

- Derive an Ethereum address from a public key by hand, in Go.
- Implement and verify the EIP-55 mixed-case checksum.
- Generate a BIP-39 mnemonic and derive keys along a BIP-44 path.
- Explain xpub/xprv, hardened derivation, and the xpub-plus-child-key leak.

## Concepts

### 1. Ethereum address derivation

Four steps, and one trap:

```go
pub := crypto.FromECDSAPub(&key.PublicKey)   // 65 bytes: 0x04 ‖ X ‖ Y
coords := pub[1:]                            // DROP the 0x04 prefix
h := crypto.Keccak256(coords)                // Keccak-256, not SHA3-256
addr := h[12:]                               // the last 20 bytes
```

The trap is step 2. Hashing all 65 bytes gives a valid-looking, completely different address, with no
error at any point. Example
[1](examples/07-addresses-wallets-hd/1-easy.md#1-public-key-to-address-by-hand) prints both so you can
see how similar they look.

Note the asymmetry with Bitcoin: Ethereum hashes the **uncompressed** coordinates, Bitcoin hashes the
**compressed** key. Different input, different result — the two addresses for one key are unrelated
and not convertible.

The property that matters most is this: **an address is derived, never registered.** There is no
allocation table, no list of "real" addresses. All 2¹⁶⁰ of them exist and hold zero until something is
sent. Which means a wrong-but-valid address accepts your funds silently — no bounce, no error, no
recovery. Example
[4](examples/07-addresses-wallets-hd/1-easy.md#4-every-20-byte-value-is-an-address) makes the point,
and shows that `common.HexToAddress` will happily pad short input and truncate long input from the
left.

### 2. EIP-55 checksums

Since nothing else protects a mistyped address, Ethereum added a checksum that changes no bytes at
all — it lives entirely in the **letter case** of the hex:

> Hash the lowercase hex string (as ASCII, not as bytes). Uppercase hex letter *i* whenever nibble
> *i* of that hash is ≥ 8.

```go
lower := hex.EncodeToString(addr)
hash := crypto.Keccak256([]byte(lower))
for i := range out {
    if out[i] >= 'a' && out[i] <= 'f' && nibble(hash, i) >= 8 {
        out[i] = upper(out[i])
    }
}
```

Because only the case changes, an all-lowercase address remains valid — which is the point, and also
the weakness. Example
[2](examples/07-addresses-wallets-hd/1-easy.md#2-eip-55-computing-the-checksum) implements it and
walks the first eight characters showing the decision for each.

Validation is recomputation. Example
[3](examples/07-addresses-wallets-hd/1-easy.md#3-eip-55-validating-what-a-user-pasted) is the function
to run on anything a user typed, and it treats the awkward case explicitly:

| Input | Result |
|---|---|
| Correct mixed case | valid |
| **All lowercase** | **no checksum to verify** — not "valid" |
| One character changed | checksum mismatch |
| Wrong length / no prefix / not hex | structural errors |

The protection is good but not total: a single wrong character is caught about 99.986% of the time.
Two things to remember. `common.IsHexAddress` does **not** check the checksum. And
`common.HexToAddress` discards the input casing and recomputes EIP-55 on output — so a typo'd address
comes back looking impeccable. **Parsing is not validation.**

### 3. Bitcoin address formats

Bitcoin took the opposite approach: a new, self-describing format per script type.

```
HASH160 = RIPEMD160(SHA256(compressed pubkey))     // 20 bytes, lesson 04
address = base58check(version ‖ HASH160)           // lesson 03
```

| Prefix | Type | Encoding |
|---|---|---|
| `1…` | P2PKH | base58check, version `0x00` |
| `3…` | P2SH | base58check, version `0x05` |
| `m…`/`n…` | P2PKH testnet | base58check, version `0x6f` |
| `bc1q…` | P2WPKH | bech32, witness v0 |
| `bc1p…` | P2TR (Taproot) | bech32m, witness v1 |

Example [5](examples/07-addresses-wallets-hd/1-easy.md#5-one-key-five-address-formats) builds several
from one key. The leading character is simply the version byte showing through base58.

**bech32 exists because base58check had real problems**: mixed case is error-prone to read aloud,
QR codes are about 20% larger for mixed-case alphanumeric, and a 4-byte checksum can detect an error
but not locate it. bech32 is lowercase-only, QR-efficient, and its BCH code can point at *which*
character is wrong. The encoding itself is in [36](36-bitcoin-deep-dive.md).

The trade-off is worth naming. Ethereum's single format means every address looks alike and you
cannot tell what one *does* before sending. Bitcoin's self-describing formats tell you, at the cost
of five formats and the migration pain in topic 7.

### 4. BIP-39 mnemonics

Writing down 32 bytes of hex is a recipe for transcription errors. BIP-39 encodes the same entropy as
words:

```
entropy (128 bits) + checksum (4 bits) = 132 bits
132 / 11 bits per word = 12 words, each indexing a fixed 2048-word list
```

The checksum is the first ENT/32 bits of `SHA-256(entropy)`. Example
[6](examples/07-addresses-wallets-hd/2-medium.md#6-bip-39-entropy-becomes-words) computes it by hand
— and then measures how much protection it actually gives, which is less than most people assume:

- A word **not in the list** is always rejected.
- A wrong word that **is** in the list slips through about **1 time in 16**, because 12 words carry
  only 4 checksum bits. A 24-word phrase gives 8 bits — 1 in 256. Still not a guarantee.

So verify a restored wallet by checking a known address, not by trusting the checksum.

The words are then stretched into the actual seed:

```
seed = PBKDF2-HMAC-SHA512(mnemonic, "mnemonic" + passphrase, 2048 iterations, 64 bytes)
```

Note where the passphrase goes: into the **salt**. Example
[7](examples/07-addresses-wallets-hd/2-medium.md#7-bip-39-words-become-a-seed) derives the seed both
via the library and via `pbkdf2.Key` directly, and then adds a passphrase to show a completely
different seed.

That optional passphrase is the "25th word", and it cuts both ways. It gives plausible deniability and
means a stolen phrase alone is not enough. But **any** passphrase is valid — there is no wrong one,
only a different wallet. A typo produces a valid, empty wallet and no error ever, and the passphrase
is rarely on the metal backup next to the phrase.

Why only 2048 iterations, when modern KDFs use far more? Because the stretching protects the
*passphrase*, which may be weak. The mnemonic itself already carries 128+ bits and needs no help.

The wordlist is engineered for humans: 2048 words so each carries exactly 11 bits, unique first four
letters, no confusable pairs, nothing under three letters. Language variants exist and are **not**
interchangeable.

### 5. BIP-32 hierarchical deterministic keys

One seed, a tree of billions of keys. The master key is a single HMAC:

```go
mac := hmac.New(sha512.New, []byte("Bitcoin seed"))
mac.Write(seed)
I := mac.Sum(nil)
masterKey, chainCode := I[:32], I[32:]
```

Two secrets come out, not one. The **chain code** is what makes derivation both deterministic and
private: children are derived as `HMAC-SHA512(chainCode, parentKey ‖ index)`, so knowing one child
key tells you nothing about its siblings — you would need the chain code to walk sideways. Example
[8](examples/07-addresses-wallets-hd/2-medium.md#8-bip-32-the-master-key-and-the-chain-code) computes
the master key by hand and confirms the library agrees.

An **extended key** packages all of it into 78 base58check-encoded bytes:

| Bytes | Field |
|---|---|
| 4 | version (`xprv`/`xpub`, per network) |
| 1 | depth |
| 4 | parent fingerprint |
| 4 | child index |
| 32 | chain code |
| 33 | key — `0x00 ‖ privkey`, or the compressed pubkey |

`Neuter()` strips the private key and keeps everything else, giving an **xpub**: enough to derive
every receiving address, not enough to spend. Example
[10](examples/07-addresses-wallets-hd/2-medium.md#10-watch-only-addresses-from-an-xpub) derives five
addresses from an xpub alone with no secret in the process. This is precisely how a deposit system
works ([58](58-deposit-detection.md)): signing keys offline, an xpub on the web server, a fresh
address per user forever, and a full server compromise leaks addresses rather than funds.

### 6. Hardened vs non-hardened derivation

The 32-bit index space splits at 2³¹, and the two halves derive differently:

```
normal   (i < 2³¹):  HMAC(chainCode, parentPUBKEY  ‖ index)
hardened (i ≥ 2³¹):  HMAC(chainCode, 0x00 ‖ parentPRIVKEY ‖ index)
```

Normal derivation uses only public material, so an xpub can do it — that is what makes watch-only
wallets possible. Hardened derivation needs the private key, so an xpub cannot. Example
[11](examples/07-addresses-wallets-hd/2-medium.md#11-hardened-and-non-hardened-derivation) shows an
xpub walking `m/0` and refusing `m/0'`.

**The cost of the convenience is a genuine key-recovery attack.** Non-hardened derivation is:

```
childPriv = (IL + parentPriv) mod n     where IL = HMAC(chainCode, parentPubKey ‖ index)[:32]
```

An xpub contains *both* the parent public key and the chain code. So anyone holding it can compute
`IL` for any index. If they also obtain **one** non-hardened child private key — from a sweep script,
a test fixture, an exported key for a single address — they simply rearrange:

```
parentPriv = (childPriv − IL) mod n
```

Example [14](examples/07-addresses-wallets-hd/3-hard.md#14-the-xpub-leak) performs it, recovers the
parent key exactly, and then derives every sibling.

This is why BIP-44 hardens the first three levels. The leak stops at the account node: an attacker
gets one account, not the master key and not your other accounts. **Treat an xpub as secret**, and
never export a single child private key from a node whose xpub you have shared.

### 7. BIP-44 and friends

```
m / purpose' / coin_type' / account' / change / index
m /    44'   /     60'    /    0'    /   0    /   x     ← standard Ethereum
```

| Level | Meaning |
|---|---|
| `purpose'` | 44 = BIP-44. Also 49 (P2SH-SegWit), 84 (native SegWit), 86 (Taproot) |
| `coin_type'` | SLIP-44: 0 = Bitcoin, 60 = Ethereum, 501 = Solana, 118 = Cosmos |
| `account'` | a user-visible wallet; hardened, so accounts are isolated |
| `change` | 0 = receive, 1 = internal change (Bitcoin) |
| `index` | 0, 1, 2 … one per address |

The first three are hardened and the last two are not — exactly the split topic 6 argues for.

Example [9](examples/07-addresses-wallets-hd/2-medium.md#9-deriving-the-accounts-you-have-been-using)
derives `m/44'/60'/0'/0/x` from the published test mnemonic and produces the ten anvil accounts from
[02](02-environment-setup.md), including the key you have signed with since
[06](06-keys-signatures.md).

**A wrong path is the most common "my funds are gone" report**, and it is completely silent. Example
[13](examples/07-addresses-wallets-hd/2-medium.md#13-the-wrong-path-gives-a-valid-empty-wallet)
derives seven plausible paths from one phrase and gets seven valid wallets, six of which have never
held anything. On Bitcoin it is worse, because `purpose'` selects the *address type*: example
[12](examples/07-addresses-wallets-hd/2-medium.md#12-one-mnemonic-four-bitcoin-address-types) shows
one mnemonic producing legacy, wrapped-SegWit, native-SegWit and Taproot wallets that share no coins.

Nothing errors, because every path is legitimate. So: **record the derivation path with the backup,
and verify a known address on restore.** Note also that a wrong path and a wrong passphrase (topic 4)
fail identically — both give a valid empty wallet.

### 8. Contract and vanity addresses

Contracts get addresses too, and neither is chosen freely.

**CREATE** — derived from the deployer and their nonce:

```
address = keccak256(rlp([sender, nonce]))[12:]
```

Predictable, which is useful — deploy from a fresh account and you get the same address on every
chain. And fragile: any other transaction from that account first shifts the nonce and therefore the
address. Example
[15](examples/07-addresses-wallets-hd/3-hard.md#15-create-the-address-of-your-next-deployment)
computes it for eight nonces, and nonce 0 gives the address anvil deploys to first.

**CREATE2** — replaces the nonce with a salt and a hash of the init code:

```
address = keccak256(0xff ‖ sender ‖ salt ‖ keccak256(initCode))[12:]
```

Every input is chosen, so the address is computable **before the contract exists**. That enables
counterfactual deployment: hand a user an address today, deploy on first use, and let them pay for it
— which is how smart-account wallets work ([47](47-account-abstraction.md)).

It also creates a trap worth internalising: **an address with no code today can have code tomorrow.**
A check like `addr.code.length == 0` does not mean "this is an EOA", it means "no code right now".
Example [16](examples/07-addresses-wallets-hd/3-hard.md#16-create2-an-address-before-the-contract)
covers it, including the historical SELFDESTRUCT-and-redeploy variant that EIP-6780 restricted.

**Vanity addresses** are ground by brute force, and the cost is exponential in the prefix: 16ⁿ
attempts for n hex characters. Example
[17](examples/07-addresses-wallets-hd/3-hard.md#17-vanity-addresses-and-how-profanity-lost-160m)
grinds up to four characters and tabulates the rest.

The cautionary tale is **Profanity** (2022). It seeded its search with a **32-bit** random value and
then walked deterministically, so the entire reachable keyspace was about 4 billion keys. Given a
Profanity-generated address, an attacker could search that space in hours. Wintermute lost roughly
$160M this way. The curve was fine; the entropy was not — the same lesson as
[06](06-keys-signatures.md) topic 4. If you want a vanity address safely, grind a **CREATE2 salt**
instead: no private key is involved at all.

### 9. Storing keys

The Web3 **keystore v3** format is what geth, MetaMask and most tooling write:

```
derivedKey = scrypt(passphrase, salt, n, r, p, 32)
ciphertext = AES-128-CTR(derivedKey[0:16], iv, privateKey)
mac        = keccak256(derivedKey[16:32] ‖ ciphertext)
```

The MAC distinguishes a wrong passphrase from a corrupted file. Example
[18](examples/07-addresses-wallets-hd/3-hard.md#18-storing-a-key-keystore-v3) writes one, inspects the
JSON, and unlocks it correctly and incorrectly.

**The passphrase is the entire security**, and scrypt's memory-hardness is what stands between a
stolen file and the key. Standard parameters (N=262144, r=8) cost about a second and ~256 MB per
attempt, which is deliberately hostile to GPU and ASIC brute-forcing. The lesson's example uses light
parameters purely for speed — never for a real key. The format is covered in full in
[44](44-symmetric-crypto-at-rest.md).

Storage tiers, in increasing order of safety:

1. A variable in your process — this course only.
2. A keystore file plus a passphrase.
3. An OS keychain or secret manager.
4. A **hardware wallet** — the key never leaves the device; you send a hash and receive a signature.
5. An **HSM or MPC service** — [32](32-key-management-signing.md), [43](43-multisig-mpc-threshold.md).

And behind all of it sits the seed phrase, which needs its own plan: metal backup, geographic
separation, a Shamir split for large sums ([43](43-multisig-mpc-threshold.md)), and an inheritance
procedure someone else can actually follow. That last one is the most commonly skipped and the most
expensive to skip.

> **The rule this course enforces:** test keys only, always labelled. Every key in these lessons is
> the published anvil key or the published test mnemonic, and every example says so.

## Exercises

Write these in `practice/07-addresses-wallets-hd/`.

1. **Address derivation, both ways.** Derive an address by hand and with `PubkeyToAddress`, and
   table-test them against five known keypairs. Then deliberately include the `0x04` prefix and write
   a test asserting the result is *different* — so the mistake can never come back silently.
2. **EIP-55 round trip.** Implement encode and validate. Test: correct mixed case, all-lowercase,
   all-uppercase, one character re-cased, and one character replaced. Each must produce a distinct
   outcome.
3. **Measure the checksum.** For 10,000 random addresses, flip one random character and count how
   often EIP-55 catches it. Compare your figure with 99.986%.
4. **BIP-39 by hand.** Implement entropy → mnemonic → seed without `go-bip39`, checking against the
   official BIP-39 test vectors. Then measure how often a single substituted word passes, for 12-word
   and 24-word phrases.
5. **Derive the anvil accounts.** From the test mnemonic, derive `m/44'/60'/0'/0/0..9` and assert they
   equal the ten addresses in lesson 02. Then derive the same indices under `m/44'/60'/x'/0/0`
   (Ledger's old layout) and note they are entirely different.
6. **Build a watch-only deriver.** Take an xpub, derive 100 addresses, and expose them over HTTP. Then
   write a test proving the process holds no private key — grep your own memory dump if you want to
   be thorough.
7. **Reproduce the xpub leak.** Perform example 14 with your own seed. Then repeat it with the child
   index *hardened* and confirm the attack fails. Write down, in one sentence, exactly which BIP-44
   level stops it.
8. **CREATE2 in practice.** Compute a CREATE2 address in Go, deploy the same init code with that salt
   on `anvil`, and confirm the deployed address matches your prediction. Then grind a salt for a
   two-character vanity prefix.

## Best Practices & Pitfalls

- **Strip the `0x04` prefix before hashing a public key.**
  *Why:* including it produces a valid-looking, entirely different address, with no error anywhere.
  The funds are simply gone.
- **Validate EIP-55 on anything a user typed or pasted — and treat all-lowercase as unchecked.**
  *Why:* an address is derived, not registered, so a wrong-but-valid one accepts your transfer
  silently. `IsHexAddress` does not check the checksum, and `HexToAddress` recomputes it on output, so
  a typo comes back looking perfect.
- **Record the derivation path alongside the seed phrase.**
  *Why:* the same phrase under a different path is a different, valid, empty wallet. This is the most
  common "my funds are gone" report, and nothing errors to warn you.
- **Treat an xpub as secret.**
  *Why:* it reveals every address you will ever derive (a privacy leak) and, combined with one
  non-hardened child private key, it gives up the parent key outright (example 14).
- **Never export a single child private key from a node whose xpub you have shared.**
  *Why:* that is the exact precondition for the leak. If you must expose one key, harden the level
  above it.
- **Harden `purpose'`, `coin_type'` and `account'`; leave `change` and `index` normal.**
  *Why:* this is BIP-44's design and it bounds the blast radius of a leak to a single account while
  keeping watch-only derivation possible.
- **Verify a restored wallet against a known address, not against the checksum.**
  *Why:* a 12-word phrase has only 4 checksum bits, so a substituted real word passes about 1 time in
  16. The checksum catches typos, not substitutions.
- **Do not assume a passphrase is remembered or backed up.**
  *Why:* any passphrase is valid, so a wrong one silently yields an empty wallet. If you use one, back
  it up separately and test the restore.
- **`code.length == 0` does not mean "this is an EOA".**
  *Why:* CREATE2 lets someone deploy at a known address later — including one you have already
  approved or funded.
- **Never trust a vanity generator's entropy.**
  *Why:* Profanity's 32-bit seed cost Wintermute ~$160M. If you want a pattern, grind a CREATE2 salt,
  where no private key is at stake.
- **Never write a real private key to a log, a terminal or an error string.**
  *Why:* it only takes once. Use a redacting type ([02](02-environment-setup.md)) and, for anything
  that matters, a signer that never hands the key out at all ([32](32-key-management-signing.md)).

## Checklist

- [ ] I can derive an Ethereum address from a public key by hand and say why the `0x04` prefix is dropped.
- [ ] I can implement and validate an EIP-55 checksum, and explain what an all-lowercase address means.
- [ ] I know that parsing an address is not validating it.
- [ ] I can name the common Bitcoin address prefixes and what distinguishes them.
- [ ] I can explain how BIP-39 turns entropy into words, and how weak the checksum really is.
- [ ] I know where the BIP-39 passphrase goes and why a wrong one fails silently.
- [ ] I can compute a BIP-32 master key and explain what the chain code is for.
- [ ] I can derive `m/44'/60'/0'/0/x` and explain every level of the path.
- [ ] I can build a watch-only wallet from an xpub.
- [ ] I can perform the xpub leak and explain which BIP-44 level bounds it.
- [ ] I can compute CREATE and CREATE2 addresses and explain the counterfactual trap.
- [ ] I can describe the keystore v3 format and name the five storage tiers.

## Resources

**Specifications**

- EIP-55 — address checksum: https://eips.ethereum.org/EIPS/eip-55
- BIP-32 — HD wallets: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP-39 — mnemonics: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
- BIP-44 — derivation paths: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- BIP-49 / BIP-84 / BIP-86 — SegWit and Taproot paths: https://github.com/bitcoin/bips
- SLIP-44 — registered coin types: https://github.com/satoshilabs/slips/blob/master/slip-0044.md
- EIP-1014 — CREATE2: https://eips.ethereum.org/EIPS/eip-1014
- Web3 Secret Storage (keystore v3): https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/

**Go**

- go-ethereum `crypto`: https://pkg.go.dev/github.com/ethereum/go-ethereum/crypto
- go-ethereum `accounts/keystore`: https://pkg.go.dev/github.com/ethereum/go-ethereum/accounts/keystore
- `hdkeychain`: https://pkg.go.dev/github.com/btcsuite/btcd/btcutil/hdkeychain
- `go-bip39`: https://pkg.go.dev/github.com/tyler-smith/go-bip39

**Background**

- The Profanity vanity-address vulnerability: https://blog.1inch.io/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool/
- Ethereum docs — accounts: https://ethereum.org/en/developers/docs/accounts/
- Ian Coleman's BIP-39 tool (read the source; do not paste a real phrase into it): https://github.com/iancoleman/bip39

---

**Examples:** [`examples/07-addresses-wallets-hd/`](examples/07-addresses-wallets-hd/) — **18 runnable
Go programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard), all using the published anvil test mnemonic so
every output reproduces exactly. Example 9 derives the ten accounts you met in lesson 02; example 14
recovers a parent private key from an xpub and one leaked child.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
