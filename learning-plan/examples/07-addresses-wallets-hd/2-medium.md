# Step 07 — Addresses, Encodings & HD Wallets · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. BIP-39: entropy becomes words

`🟡 medium` · *BIP-39*

Entropy plus a short checksum, split into 11-bit groups, each indexing a 2048-word list. The surprise is how weak that checksum is: with 12 words it is **4 bits**, so a wrong-but-real word slips through about once in sixteen — and this example shows one that does.

**Steps:**

1. Turn 128 bits of entropy into 12 words and recover the entropy from them.
2. Compute the checksum by hand: the first ENT/32 bits of SHA-256(entropy).
3. Confirm a word outside the list is always rejected.
4. Substitute sixteen real words for word 0 and count how many pass — about 1 in 16 do.
5. Conclude: verify a restored wallet against a known address, not just the checksum.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"strings"

	bip39 "github.com/tyler-smith/go-bip39"
)

func main() {
	// Fixed entropy so this reproduces. In real use it comes from crypto/rand.
	entropy, _ := hex.DecodeString("df9bf37e6fcdf9bf37e6fcdf9bf37e3c")
	fmt.Printf("entropy (%d bytes = %d bits)\n  %s\n",
		len(entropy), len(entropy)*8, hex.EncodeToString(entropy))

	mnemonic, err := bip39.NewMnemonic(entropy)
	if err != nil {
		fmt.Println("mnemonic:", err)
		return
	}
	fmt.Printf("\nmnemonic (%d words)\n  %s\n", len(strings.Fields(mnemonic)), mnemonic)

	// The words are not arbitrary: entropy is split into 11-bit groups, each
	// indexing a fixed 2048-word list. 128 bits + 4 checksum bits = 132 = 12x11.
	fmt.Println("\nhow the words are chosen")
	fmt.Printf("  %d bits of entropy + %d checksum bits = %d bits\n",
		len(entropy)*8, len(entropy)*8/32, len(entropy)*8+len(entropy)*8/32)
	fmt.Printf("  %d bits / 11 bits per word = %d words\n",
		len(entropy)*8+len(entropy)*8/32, (len(entropy)*8+len(entropy)*8/32)/11)
	fmt.Println("  each 11-bit group indexes the 2048-word BIP-39 list")

	// The checksum: the first ENT/32 bits of SHA-256(entropy).
	sum := sha256.Sum256(entropy)
	checksumBits := len(entropy) * 8 / 32
	fmt.Printf("\nchecksum = first %d bits of sha256(entropy)\n", checksumBits)
	fmt.Printf("  sha256   %s\n", hex.EncodeToString(sum[:]))
	fmt.Printf("  first byte %08b, taking the top %d bits: %0*b\n",
		sum[0], checksumBits, checksumBits, sum[0]>>(8-checksumBits))

	// Round trip.
	back, _ := bip39.EntropyFromMnemonic(mnemonic)
	fmt.Printf("\nentropy recovered from the words: %v\n",
		hex.EncodeToString(back) == hex.EncodeToString(entropy))

	// A word that is not in the list at all is always rejected.
	words := strings.Fields(mnemonic)
	notAWord := append([]string{}, words...)
	notAWord[3] = "notaword"
	fmt.Printf("\na word that is not in the list:\n  %-10s valid: %v\n",
		"notaword", bip39.IsMnemonicValid(strings.Join(notAWord, " ")))

	// But a wrong word that IS in the list only fails if the checksum notices —
	// and with 12 words the checksum is just 4 bits, so it misses 1 time in 16.
	fmt.Println("\na wrong word that IS in the list — only 4 checksum bits protect you:")
	pass, total := 0, 0
	for _, w := range []string{"abandon", "ability", "able", "about", "above", "absent",
		"absorb", "abstract", "absurd", "abuse", "access", "accident", "account", "accuse",
		"achieve", "acid"} {
		cand := append([]string{}, words...)
		cand[0] = w
		ok := bip39.IsMnemonicValid(strings.Join(cand, " "))
		total++
		if ok {
			pass++
		}
		if w == "abandon" || w == "ability" {
			fmt.Printf("  %-10s valid: %v\n", w, ok)
		}
	}
	fmt.Printf("\n  across %d substitutions of word 0: %d passed, %d rejected\n",
		total, pass, total-pass)
	fmt.Printf("  expected pass rate 1/2^%d = 1 in %d\n", checksumBits, 1<<checksumBits)
	fmt.Println("\n  so the checksum catches a TYPO (a non-word) every time, but a")
	fmt.Println("  substituted real word slips through about once every 16 tries.")
	fmt.Println("  A 24-word phrase has 8 checksum bits: 1 in 256. Still not a")
	fmt.Println("  guarantee — verify a restored wallet by checking a known address.")

	// Wordlist design.
	fmt.Println("\nthe wordlist is designed for humans")
	fmt.Println("  2048 words, so each one carries exactly 11 bits")
	fmt.Println("  the first four letters are unique — you can type just those")
	fmt.Println("  no pairs that are easily confused, and no words under 3 letters")
	fmt.Println("  official lists exist for several languages; they are NOT interchangeable")

	// Entropy sizes.
	fmt.Println("\nsupported sizes")
	for _, bits := range []int{128, 160, 192, 224, 256} {
		fmt.Printf("  %3d bits -> %2d words (%d checksum bits)\n",
			bits, (bits+bits/32)/11, bits/32)
	}
}
```

**Output:**

```
entropy (16 bytes = 128 bits)
  df9bf37e6fcdf9bf37e6fcdf9bf37e3c

mnemonic (12 words)
  test test test test test test test test test test test junk

how the words are chosen
  128 bits of entropy + 4 checksum bits = 132 bits
  132 bits / 11 bits per word = 12 words
  each 11-bit group indexes the 2048-word BIP-39 list

checksum = first 4 bits of sha256(entropy)
  sha256   a13b27648ff45bcf53d78bba1170226871a88a2cae91754c0c1e31a087b169d3
  first byte 10100001, taking the top 4 bits: 1010

entropy recovered from the words: true

a word that is not in the list:
  notaword   valid: false

a wrong word that IS in the list — only 4 checksum bits protect you:
  abandon    valid: true
  ability    valid: false

  across 16 substitutions of word 0: 2 passed, 14 rejected
  expected pass rate 1/2^4 = 1 in 16

  so the checksum catches a TYPO (a non-word) every time, but a
  substituted real word slips through about once every 16 tries.
  A 24-word phrase has 8 checksum bits: 1 in 256. Still not a
  guarantee — verify a restored wallet by checking a known address.

the wordlist is designed for humans
  2048 words, so each one carries exactly 11 bits
  the first four letters are unique — you can type just those
  no pairs that are easily confused, and no words under 3 letters
  official lists exist for several languages; they are NOT interchangeable

supported sizes
  128 bits -> 12 words (4 checksum bits)
  160 bits -> 15 words (5 checksum bits)
  192 bits -> 18 words (6 checksum bits)
  224 bits -> 21 words (7 checksum bits)
  256 bits -> 24 words (8 checksum bits)
```

---

## 7. BIP-39: words become a seed

`🟡 medium` · *BIP-39*

The words are not the seed. They go through PBKDF2-HMAC-SHA512 with 2048 iterations and a salt of `"mnemonic" + passphrase`. That passphrase is the '25th word' — and because **any** passphrase is valid, a wrong one gives you a valid empty wallet and no error.

**Steps:**

1. Derive the seed with `bip39.NewSeed`, then again with `pbkdf2.Key` directly.
2. Add a passphrase and see a completely different seed.
3. Understand why 2048 iterations is low: it protects the passphrase, not the phrase.
4. Note the seed is chain-agnostic — the path decides which chain (example 9).

```go
package main

import (
	"encoding/hex"
	"fmt"

	"crypto/sha512"
	bip39 "github.com/tyler-smith/go-bip39"
	"golang.org/x/crypto/pbkdf2"
	"hash"
)

func main() {
	// The published Hardhat/anvil test mnemonic (lesson 02).
	const mnemonic = "test test test test test test test test test test test junk"
	fmt.Printf("mnemonic\n  %s\n", mnemonic)

	// The words are NOT the seed. They go through PBKDF2 with 2048 iterations.
	seed := bip39.NewSeed(mnemonic, "")
	fmt.Printf("\nseed (%d bytes)\n  %s\n", len(seed), hex.EncodeToString(seed))

	// Doing it by hand: PBKDF2-HMAC-SHA512, salt = "mnemonic" + passphrase.
	manual := pbkdf2.Key([]byte(mnemonic), []byte("mnemonic"), 2048, 64,
		func() hash.Hash { return sha512.New() })
	fmt.Printf("\nPBKDF2-HMAC-SHA512(mnemonic, \"mnemonic\"+passphrase, 2048, 64)\n")
	fmt.Printf("  matches bip39.NewSeed: %v\n", hex.EncodeToString(manual) == hex.EncodeToString(seed))

	// The passphrase is appended to the SALT, not the password. Any passphrase
	// is "valid" — there is no wrong one, only a different wallet.
	withPass := bip39.NewSeed(mnemonic, "my extra word")
	fmt.Printf("\nwith passphrase \"my extra word\"\n  %s\n", hex.EncodeToString(withPass))
	fmt.Printf("  same seed as before: %v\n",
		hex.EncodeToString(withPass) == hex.EncodeToString(seed))

	fmt.Println("\nthis is the '25th word', and it is a double-edged feature:")
	fmt.Println("  + plausible deniability — one phrase, many wallets")
	fmt.Println("  + a stolen phrase alone is not enough")
	fmt.Println("  - there is no way to tell a wrong passphrase from a different one.")
	fmt.Println("    You get a valid, empty wallet, and no error ever.")
	fmt.Println("  - the phrase is backed up on metal; the passphrase usually is not.")

	// Note what PBKDF2 protects here, and what it does not.
	fmt.Println("\nwhy 2048 iterations?")
	fmt.Println("  it slows brute-forcing a PASSPHRASE, which may be weak.")
	fmt.Println("  it does nothing for the mnemonic itself — 128 bits of entropy")
	fmt.Println("  is already out of reach, so no stretching is needed there.")
	fmt.Println("  2048 is low by modern standards precisely because the phrase is strong.")

	// One seed, every chain.
	fmt.Println("\nthe seed is chain-agnostic: the same 64 bytes derive Bitcoin,")
	fmt.Println("Ethereum, Solana and Cosmos keys. The path decides which (example 9).")
}
```

**Output:**

```
mnemonic
  test test test test test test test test test test test junk

seed (64 bytes)
  9dfc3c64c2f8bede1533b6a79f8570e5943e0b8fd1cf77107adf7b72cef42185d564a3aee24cab43f80e3c4538087d70fc824eabbad596a23c97b6ee8322ccc0

PBKDF2-HMAC-SHA512(mnemonic, "mnemonic"+passphrase, 2048, 64)
  matches bip39.NewSeed: true

with passphrase "my extra word"
  17dd579c961b79e147197e050a999d85ee2c923bcc081a67705afde2e97faf3c3d96f22de475a92547a728abcd4392f21086939e5ec308f25e884253a42860ff
  same seed as before: false

this is the '25th word', and it is a double-edged feature:
  + plausible deniability — one phrase, many wallets
  + a stolen phrase alone is not enough
  - there is no way to tell a wrong passphrase from a different one.
    You get a valid, empty wallet, and no error ever.
  - the phrase is backed up on metal; the passphrase usually is not.

why 2048 iterations?
  it slows brute-forcing a PASSPHRASE, which may be weak.
  it does nothing for the mnemonic itself — 128 bits of entropy
  is already out of reach, so no stretching is needed there.
  2048 is low by modern standards precisely because the phrase is strong.

the seed is chain-agnostic: the same 64 bytes derive Bitcoin,
Ethereum, Solana and Cosmos keys. The path decides which (example 9).
```

---

## 8. BIP-32: the master key and the chain code

`🟡 medium` · *BIP-32*

The master key is one HMAC-SHA512 of the seed, keyed by the literal string `"Bitcoin seed"`. It yields *two* secrets: the key and the **chain code**. Both are needed to derive children, which is why an extended key carries 64 bytes of secret rather than 32.

**Steps:**

1. Compute the master key and chain code by hand and check `hdkeychain` agrees.
2. Understand what the chain code buys: siblings stay unlinkable.
3. Print the xprv and xpub, and read the 78-byte extended-key layout.
4. Confirm an xpub cannot produce a private key.

```go
package main

import (
	"crypto/hmac"
	"crypto/sha512"
	"encoding/hex"
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	bip39 "github.com/tyler-smith/go-bip39"
)

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")

	// The master key is one HMAC-SHA512 of the seed, keyed by a fixed string.
	mac := hmac.New(sha512.New, []byte("Bitcoin seed"))
	mac.Write(seed)
	I := mac.Sum(nil)

	masterKey, chainCode := I[:32], I[32:]
	fmt.Printf("HMAC-SHA512(\"Bitcoin seed\", seed) = 64 bytes\n")
	fmt.Printf("  left  32 -> master private key %s\n", hex.EncodeToString(masterKey))
	fmt.Printf("  right 32 -> chain code         %s\n", hex.EncodeToString(chainCode))

	// The library agrees.
	master, err := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)
	if err != nil {
		fmt.Println("master:", err)
		return
	}
	priv, _ := master.ECPrivKey()
	fmt.Printf("\nhdkeychain agrees: %v\n",
		hex.EncodeToString(priv.Serialize()) == hex.EncodeToString(masterKey))

	// The chain code is the second secret. Without it you cannot derive
	// children — which is why an extended key is 64 bytes of secret, not 32.
	fmt.Println("\nthe chain code is what makes derivation deterministic AND private:")
	fmt.Println("  child = HMAC-SHA512(chainCode, parentKey ‖ index)")
	fmt.Println("  knowing a child key tells you nothing about its siblings,")
	fmt.Println("  because you would also need the chain code to walk sideways.")

	// Extended keys package all of it into one serialized string.
	fmt.Printf("\nxprv %s\n", master.String())
	xpub, _ := master.Neuter()
	fmt.Printf("xpub %s\n", xpub.String())

	fmt.Println("\nan extended key serializes 78 bytes, base58check-encoded:")
	fmt.Println("  4  version      (xprv/xpub, and per-network)")
	fmt.Println("  1  depth        (0 for the master)")
	fmt.Println("  4  parent fingerprint")
	fmt.Println("  4  child index")
	fmt.Println("  32 chain code")
	fmt.Println("  33 key          (0x00 ‖ privkey, or the compressed pubkey)")

	fmt.Printf("\ndepth %d, parent fingerprint %08x, index %d\n",
		master.Depth(), master.ParentFingerprint(), 0)

	// Neuter() strips the private key and keeps everything else.
	fmt.Printf("\nxpub can derive public keys: %v\n", !xpub.IsPrivate())
	if _, err := xpub.ECPrivKey(); err != nil {
		fmt.Printf("xpub cannot produce a private key: %v\n", err)
	}
}
```

**Output:**

```
HMAC-SHA512("Bitcoin seed", seed) = 64 bytes
  left  32 -> master private key ada5f7928ef8f684afa7b06929b5d1f653e486b52077ba9d798eded96c851ade
  right 32 -> chain code         a6001a33d7f033eb872b5e95587fe2053a2df53d546ba932637f2ab7a8cff940

hdkeychain agrees: true

the chain code is what makes derivation deterministic AND private:
  child = HMAC-SHA512(chainCode, parentKey ‖ index)
  knowing a child key tells you nothing about its siblings,
  because you would also need the chain code to walk sideways.

xprv xprv9s21ZrQH143K3iDNSJZcYeHRe33FQ98DS1wZR9xstVPN8zs77wpsghZt41M6s72keKC1gW3ePJMyL19duV524VbEqWBW6BL5fJ4FopSgNLz
xpub xpub661MyMwAqRbcGCHqYL6cunEAC4sjobr4oEsADYNVSpvM1oCFfV98EVtMuHVmKomD5EWqhYwUPCkQdrti7hUGbmxaoTGLSkzhtBmR5tk9Jtu

an extended key serializes 78 bytes, base58check-encoded:
  4  version      (xprv/xpub, and per-network)
  1  depth        (0 for the master)
  4  parent fingerprint
  4  child index
  32 chain code
  33 key          (0x00 ‖ privkey, or the compressed pubkey)

depth 0, parent fingerprint 00000000, index 0

xpub can derive public keys: true
xpub cannot produce a private key: unable to create private keys from a public extended key
```

---

## 9. Deriving the accounts you have been using

`🟡 medium` · *BIP-44*

`m / purpose' / coin_type' / account' / change / index`. Derive along `m/44'/60'/0'/0` from the published test mnemonic and you get the ten anvil accounts from lesson 02 — including the exact private key you have been signing with since lesson 06.

**Steps:**

1. Derive the account node, then ten sequential child addresses.
2. Compare them against lesson 02's list of funded anvil accounts.
3. Print account 0's private key and recognise it.
4. Read what each level of the path is actually for.

```go
package main

import (
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart // 0x80000000

// derive walks a path of raw indices from a starting key.
func derive(k *hdkeychain.ExtendedKey, path []uint32) (*hdkeychain.ExtendedKey, error) {
	var err error
	for _, i := range path {
		if k, err = k.Derive(i); err != nil {
			return nil, err
		}
	}
	return k, nil
}

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)

	// BIP-44:  m / purpose' / coin_type' / account' / change / index
	//          m /     44'   /     60'    /    0'    /   0    /   x
	//                          ^ SLIP-44 coin type 60 = Ethereum
	account, err := derive(master, []uint32{hardened + 44, hardened + 60, hardened + 0, 0})
	if err != nil {
		fmt.Println("derive:", err)
		return
	}
	fmt.Println("path m/44'/60'/0'/0")
	fmt.Printf("  depth %d\n\n", account.Depth())

	fmt.Printf("%-6s %s\n", "index", "address")
	for i := 0; i < 10; i++ {
		child, err := account.Derive(uint32(i))
		if err != nil {
			fmt.Println("child:", err)
			return
		}
		priv, _ := child.ECPrivKey()
		fmt.Printf("%-6d %s\n", i, crypto.PubkeyToAddress(priv.ToECDSA().PublicKey).Hex())
	}

	// These are exactly the ten accounts anvil funds (lesson 02, example 7) —
	// because anvil uses this mnemonic and this path.
	first, _ := account.Derive(0)
	fp, _ := first.ECPrivKey()
	fmt.Printf("\naccount 0 address  %s\n", crypto.PubkeyToAddress(fp.ToECDSA().PublicKey).Hex())
	fmt.Printf("account 0 key      %x\n", fp.Serialize())
	fmt.Println("  the key you have been signing with since lesson 06 —")
	fmt.Println("  it was derived from this phrase and this path all along.")

	// What each level is FOR.
	fmt.Println("\nwhat the levels mean")
	fmt.Println("  purpose'    44 = BIP-44. 49/84/86 select other address types (example 12)")
	fmt.Println("  coin_type'  SLIP-44: 0 = Bitcoin, 60 = Ethereum, 501 = Solana, 118 = Cosmos")
	fmt.Println("  account'    a user-visible wallet; hardened, so accounts are isolated")
	fmt.Println("  change      0 = receive addresses, 1 = internal change (Bitcoin)")
	fmt.Println("  index       0, 1, 2, ... one per address")

	fmt.Println("\nthe last two levels are NOT hardened, on purpose: that is what")
	fmt.Println("lets a watch-only server derive addresses without any private key")
	fmt.Println("(example 10) — at the cost of the leak in example 14.")
}
```

**Output:**

```
path m/44'/60'/0'/0
  depth 4

index  address
0      0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
1      0x70997970C51812dc3A010C7d01b50e0d17dc79C8
2      0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
3      0x90F79bf6EB2c4f870365E785982E1f101E93b906
4      0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65
5      0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc
6      0x976EA74026E726554dB657fA54763abd0C3a0aa9
7      0x14dC79964da2C08b23698B3D3cc7Ca32193d9955
8      0x23618e81E3f5cdF7f54C3d65f7FBc0aBf5B21E8f
9      0xa0Ee7A142d267C1f36714E4a8F75612F20a79720

account 0 address  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
account 0 key      ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
  the key you have been signing with since lesson 06 —
  it was derived from this phrase and this path all along.

what the levels mean
  purpose'    44 = BIP-44. 49/84/86 select other address types (example 12)
  coin_type'  SLIP-44: 0 = Bitcoin, 60 = Ethereum, 501 = Solana, 118 = Cosmos
  account'    a user-visible wallet; hardened, so accounts are isolated
  change      0 = receive addresses, 1 = internal change (Bitcoin)
  index       0, 1, 2, ... one per address

the last two levels are NOT hardened, on purpose: that is what
lets a watch-only server derive addresses without any private key
(example 10) — at the cost of the leak in example 14.
```

---

## 10. Watch-only: addresses from an xpub

`🟡 medium` · *BIP-32*

Neuter the account node and you get an xpub: enough to derive every receiving address, and not enough to spend anything. This is exactly how a deposit system works — the web server generates addresses forever while the signing keys stay offline.

**Steps:**

1. Derive to the change level, then `Neuter()` to strip the private key.
2. Parse the xpub string fresh and derive five addresses from it.
3. Confirm they match the private-side derivation, and that no private key can be extracted.
4. Note the privacy cost: an xpub reveals every address you will ever use.

```go
package main

import (
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)

	// The COLD side: derive down to the account's change level, then publish
	// only the extended PUBLIC key.
	k := master
	for _, i := range []uint32{hardened + 44, hardened + 60, hardened + 0, 0} {
		k, _ = k.Derive(i)
	}
	xpub, err := k.Neuter()
	if err != nil {
		fmt.Println("neuter:", err)
		return
	}
	fmt.Println("published to the server (contains NO private key):")
	fmt.Printf("  %s\n", xpub.String())

	// The HOT side: parse the xpub and derive addresses. No secret anywhere.
	watch, err := hdkeychain.NewKeyFromString(xpub.String())
	if err != nil {
		fmt.Println("parse:", err)
		return
	}
	fmt.Printf("\nis private: %v\n", watch.IsPrivate())

	fmt.Println("\naddresses derived from the xpub alone:")
	for i := 0; i < 5; i++ {
		child, _ := watch.Derive(uint32(i))
		pub, _ := child.ECPubKey()
		fmt.Printf("  %d  %s\n", i, crypto.PubkeyToAddress(*pub.ToECDSA()).Hex())
	}

	// They match what the private side produces.
	c0, _ := k.Derive(0)
	p0, _ := c0.ECPrivKey()
	w0, _ := watch.Derive(0)
	wp0, _ := w0.ECPubKey()
	fmt.Printf("\nsame as the private derivation: %v\n",
		crypto.PubkeyToAddress(*wp0.ToECDSA()) == crypto.PubkeyToAddress(p0.ToECDSA().PublicKey))

	// But it genuinely cannot sign.
	if _, err := watch.Derive(0); err == nil {
		c, _ := watch.Derive(0)
		if _, err := c.ECPrivKey(); err != nil {
			fmt.Printf("cannot extract a private key: %v\n", err)
		}
	}

	// This is how deposit systems work (lesson 58).
	fmt.Println("\nthis is exactly how a deposit system is built:")
	fmt.Println("  the signing keys live offline, or in an HSM")
	fmt.Println("  the public web server holds only an xpub")
	fmt.Println("  it generates a fresh deposit address per user, forever, with no secret")
	fmt.Println("  a full compromise of that server leaks addresses, not funds")

	// The catch, in one line.
	fmt.Println("\nthe catch: an xpub reveals EVERY address you will ever derive,")
	fmt.Println("so it is a privacy leak even though it is not a spending risk.")
	fmt.Println("And combined with one child private key it is much worse — example 14.")
}
```

**Output:**

```
published to the server (contains NO private key):
  xpub6DyUKdwoLWmUJ4Tn9Bbsdtx7B5Ws18mEN19e5HT52ikE53FiUheSQXrZUNPovqfyKmw4579A1Mm3GXXKM39N64uooBfJ4tNAzFsEbodRTx4

is private: false

addresses derived from the xpub alone:
  0  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
  1  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  2  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
  3  0x90F79bf6EB2c4f870365E785982E1f101E93b906
  4  0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65

same as the private derivation: true
cannot extract a private key: unable to create private keys from a public extended key

this is exactly how a deposit system is built:
  the signing keys live offline, or in an HSM
  the public web server holds only an xpub
  it generates a fresh deposit address per user, forever, with no secret
  a full compromise of that server leaks addresses, not funds

the catch: an xpub reveals EVERY address you will ever derive,
so it is a privacy leak even though it is not a spending risk.
And combined with one child private key it is much worse — example 14.
```

---

## 11. Hardened and non-hardened derivation

`🟡 medium` · *BIP-32*

The 32-bit index space splits at 2³¹. Below it, derivation mixes in the parent *public* key, so an xpub can do it. At or above, it mixes in the parent *private* key, so it cannot. That single difference is the whole trade-off, and BIP-44 resolves it per level.

**Steps:**

1. Derive `m/0` and `m/0'` and see two entirely unrelated keys.
2. Confirm an xpub can walk normal indices and refuses hardened ones.
3. Read the trade at each level: watch-only capability versus the example 14 leak.
4. See why BIP-44 hardens exactly the first three levels.

```go
package main

import (
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart // 2^31

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)

	fmt.Printf("the index space is 32 bits, split in half at 2^31 = %d\n", hardened)
	fmt.Println("  0 .. 2^31-1        normal   (written 0, 1, 2 ...)")
	fmt.Println("  2^31 .. 2^32-1     hardened (written 0', 1', 2' ... or 0h)")

	// Normal derivation mixes in the parent PUBLIC key, so it can be done from
	// an xpub. Hardened derivation mixes in the parent PRIVATE key, so it cannot.
	fmt.Println("\nthe difference is one input to the HMAC:")
	fmt.Println("  normal:   HMAC(chainCode, parentPUBKEY  ‖ index)")
	fmt.Println("  hardened: HMAC(chainCode, 0x00 ‖ parentPRIVKEY ‖ index)")

	normal, err := master.Derive(0)
	if err != nil {
		fmt.Println("normal:", err)
		return
	}
	hard, err := master.Derive(hardened + 0)
	if err != nil {
		fmt.Println("hardened:", err)
		return
	}
	np, _ := normal.ECPrivKey()
	hp, _ := hard.ECPrivKey()
	fmt.Printf("\nm/0   %s\n", crypto.PubkeyToAddress(np.ToECDSA().PublicKey).Hex())
	fmt.Printf("m/0'  %s\n", crypto.PubkeyToAddress(hp.ToECDSA().PublicKey).Hex())
	fmt.Println("  entirely unrelated keys — the apostrophe is not cosmetic")

	// From an xpub you can walk normal indices, and only normal indices.
	xpub, _ := master.Neuter()
	if _, err := xpub.Derive(0); err == nil {
		fmt.Printf("\nfrom the xpub, m/0  works:  yes\n")
	}
	if _, err := xpub.Derive(hardened + 0); err != nil {
		fmt.Printf("from the xpub, m/0' works:  no (%v)\n", err)
	}

	// So the choice per level is a trade.
	fmt.Println("\nthe trade at every level")
	fmt.Println("  normal   + a watch-only server can derive addresses (example 10)")
	fmt.Println("           - xpub + one child privkey reveals the parent (example 14)")
	fmt.Println("  hardened + that leak is impossible")
	fmt.Println("           - you must hold the private key to derive anything")

	// Which is why BIP-44 hardens exactly the first three levels.
	fmt.Println("\nBIP-44's answer:  m / 44' / 60' / 0' / 0 / x")
	fmt.Println("                       ^^^^^^^^^^^^^^^  hardened: a leak below")
	fmt.Println("                                        an account cannot climb past it")
	fmt.Println("                                   ^^^^^^^  normal: watch-only works")
	fmt.Println("\nso the blast radius of the example 14 leak is ONE account, not the wallet.")
}
```

**Output:**

```
the index space is 32 bits, split in half at 2^31 = 2147483648
  0 .. 2^31-1        normal   (written 0, 1, 2 ...)
  2^31 .. 2^32-1     hardened (written 0', 1', 2' ... or 0h)

the difference is one input to the HMAC:
  normal:   HMAC(chainCode, parentPUBKEY  ‖ index)
  hardened: HMAC(chainCode, 0x00 ‖ parentPRIVKEY ‖ index)

m/0   0x68ba87F622CD4f39800967f26CA545965dF4177d
m/0'  0x7a5cA0143EFA6dEafa06595E47b1a17B81791a44
  entirely unrelated keys — the apostrophe is not cosmetic

from the xpub, m/0  works:  yes
from the xpub, m/0' works:  no (cannot derive a hardened key from a public key)

the trade at every level
  normal   + a watch-only server can derive addresses (example 10)
           - xpub + one child privkey reveals the parent (example 14)
  hardened + that leak is impossible
           - you must hold the private key to derive anything

BIP-44's answer:  m / 44' / 60' / 0' / 0 / x
                       ^^^^^^^^^^^^^^^  hardened: a leak below
                                        an account cannot climb past it
                                   ^^^^^^^  normal: watch-only works

so the blast radius of the example 14 leak is ONE account, not the wallet.
```

---

## 12. One mnemonic, four Bitcoin address types

`🟡 medium` · *Address formats*

The `purpose` level selects the address type, so one mnemonic yields four separate Bitcoin wallets — legacy, wrapped SegWit, native SegWit and Taproot. Restore into a wallet that defaults to a different purpose and your coins appear to have vanished.

**Steps:**

1. Derive `m/44'`, `m/49'`, `m/84'` and `m/86'` from the same seed.
2. Build the matching address for each: P2PKH, P2SH-P2WPKH, P2WPKH and P2TR.
3. Note that all four are legitimate and none of them errors.
4. Read why this is the single most common 'my funds are gone' report.

```go
package main

import (
	"fmt"

	"github.com/btcsuite/btcd/btcutil"
	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/btcsuite/btcd/txscript"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart

func derive(k *hdkeychain.ExtendedKey, path ...uint32) *hdkeychain.ExtendedKey {
	for _, i := range path {
		k, _ = k.Derive(i)
	}
	return k
}

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)
	net := &chaincfg.MainNetParams

	// The `purpose` level selects the ADDRESS TYPE, and each type gets its own
	// branch of the tree so the same key is never reused across formats.
	specs := []struct {
		bip     uint32
		name    string
		example string
	}{
		{44, "BIP-44  P2PKH (legacy)", "1..."},
		{49, "BIP-49  P2SH-P2WPKH", "3..."},
		{84, "BIP-84  P2WPKH (native segwit)", "bc1q..."},
		{86, "BIP-86  P2TR (taproot)", "bc1p..."},
	}

	fmt.Println("one mnemonic, four Bitcoin address types")
	fmt.Printf("%-32s %-24s %s\n", "purpose", "path", "first address")
	for _, s := range specs {
		k := derive(master, hardened+s.bip, hardened+0, hardened+0, 0, 0)
		pub, err := k.ECPubKey()
		if err != nil {
			fmt.Printf("%-32s error %v\n", s.name, err)
			continue
		}
		path := fmt.Sprintf("m/%d'/0'/0'/0/0", s.bip)
		var addr string

		switch s.bip {
		case 44:
			a, _ := btcutil.NewAddressPubKeyHash(btcutil.Hash160(pub.SerializeCompressed()), net)
			addr = a.EncodeAddress()
		case 49:
			wpkh, _ := btcutil.NewAddressWitnessPubKeyHash(btcutil.Hash160(pub.SerializeCompressed()), net)
			script, _ := txscript.PayToAddrScript(wpkh)
			a, _ := btcutil.NewAddressScriptHash(script, net)
			addr = a.EncodeAddress()
		case 84:
			a, _ := btcutil.NewAddressWitnessPubKeyHash(btcutil.Hash160(pub.SerializeCompressed()), net)
			addr = a.EncodeAddress()
		case 86:
			tapKey := txscript.ComputeTaprootKeyNoScript(pub)
			a, _ := btcutil.NewAddressTaproot(schnorrSerialize(tapKey), net)
			addr = a.EncodeAddress()
		}
		fmt.Printf("%-32s %-24s %s\n", s.name, path, addr)
	}

	fmt.Println("\nall four control the SAME funds only if you use the same path.")
	fmt.Println("They are different keys, so they are different wallets.")

	fmt.Println("\nthis is the single most common 'my funds are gone' report:")
	fmt.Println("  restore a BIP-84 wallet into a BIP-44 wallet and you see an")
	fmt.Println("  empty, perfectly valid wallet. The coins are on the other branch.")
	fmt.Println("  Nothing errors, because both paths are legitimate.")

	fmt.Println("\nEthereum sidesteps this by having one address format, so almost")
	fmt.Println("everything uses m/44'/60'/0'/0/x — but Ledger Live historically")
	fmt.Println("used m/44'/60'/x'/0/0, which caused exactly the same confusion.")
}

// schnorrSerialize returns the 32-byte x-only form BIP-341 uses.
func schnorrSerialize(k interface{ SerializeCompressed() []byte }) []byte {
	return k.SerializeCompressed()[1:]
}
```

**Output:**

```
one mnemonic, four Bitcoin address types
purpose                          path                     first address
BIP-44  P2PKH (legacy)           m/44'/0'/0'/0/0          1Ei9UmLQv4o4UJTy5r5mnGFeC9auM3W5P1
BIP-49  P2SH-P2WPKH              m/49'/0'/0'/0/0          39sr5B8UAdxeoXbnpdw4frfxXwWwEChwzp
BIP-84  P2WPKH (native segwit)   m/84'/0'/0'/0/0          bc1q4qw42stdzjqs59xvlrlxr8526e3nunw7mp73te
BIP-86  P2TR (taproot)           m/86'/0'/0'/0/0          bc1pfzhx49qe6s5exppe5hqljg3n6587xk0w75xqr70pgdt7ygnfkssqxqjd9l

all four control the SAME funds only if you use the same path.
They are different keys, so they are different wallets.

this is the single most common 'my funds are gone' report:
  restore a BIP-84 wallet into a BIP-44 wallet and you see an
  empty, perfectly valid wallet. The coins are on the other branch.
  Nothing errors, because both paths are legitimate.

Ethereum sidesteps this by having one address format, so almost
everything uses m/44'/60'/0'/0/x — but Ledger Live historically
used m/44'/60'/x'/0/0, which caused exactly the same confusion.
```

---

## 13. The wrong path gives a valid, empty wallet

`🟡 medium` · *BIP-44*

Seven valid paths from one seed phrase, seven different wallets, no errors anywhere. Six of them have never held anything. This is what a wrong derivation path looks like from the inside — identical to a wrong passphrase, and equally silent.

**Steps:**

1. Derive seven plausible paths and print the address each produces.
2. Note that a path one level short is also perfectly valid.
3. Read the diagnosis procedure: record the address, not just the phrase.
4. Connect it to example 7 — a wrong passphrase fails exactly the same way.

```go
package main

import (
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart

func addrAt(master *hdkeychain.ExtendedKey, path []uint32) string {
	k := master
	var err error
	for _, i := range path {
		if k, err = k.Derive(i); err != nil {
			return "derivation error"
		}
	}
	priv, err := k.ECPrivKey()
	if err != nil {
		return "no private key"
	}
	return crypto.PubkeyToAddress(priv.ToECDSA().PublicKey).Hex()
}

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)

	// Every one of these is a legitimate path. Every one gives a valid,
	// completely different wallet. None of them errors.
	paths := []struct {
		label string
		path  []uint32
	}{
		{"m/44'/60'/0'/0/0   (standard Ethereum)", []uint32{hardened + 44, hardened + 60, hardened + 0, 0, 0}},
		{"m/44'/60'/0'/0/1   (next index)", []uint32{hardened + 44, hardened + 60, hardened + 0, 0, 1}},
		{"m/44'/60'/1'/0/0   (next account)", []uint32{hardened + 44, hardened + 60, hardened + 1, 0, 0}},
		{"m/44'/60'/0'/1/0   (change branch)", []uint32{hardened + 44, hardened + 60, hardened + 0, 1, 0}},
		{"m/44'/0'/0'/0/0    (Bitcoin coin type)", []uint32{hardened + 44, hardened + 0, hardened + 0, 0, 0}},
		{"m/44'/60'/0'/0     (one level short)", []uint32{hardened + 44, hardened + 60, hardened + 0, 0}},
		{"m/0'/0/0           (an old convention)", []uint32{hardened + 0, 0, 0}},
	}

	fmt.Println("same 12 words, seven paths:")
	fmt.Printf("%-40s %s\n", "path", "address")
	for _, p := range paths {
		fmt.Printf("%-40s %s\n", p.label, addrAt(master, p.path))
	}

	fmt.Println("\nnone of these failed. All seven are valid wallets with valid")
	fmt.Println("addresses, and six of them have never held anything.")

	fmt.Println("\nwhen this bites")
	fmt.Println("  you restore a seed phrase into a different wallet app")
	fmt.Println("  the new app uses a different default path")
	fmt.Println("  the balance shows zero and the phrase looks wrong")
	fmt.Println("  it is not wrong — you are looking at a different branch of the tree")

	fmt.Println("\nhow to diagnose it")
	fmt.Println("  1. write down the ADDRESS you expect, not just the phrase")
	fmt.Println("  2. on restore, check the address before trusting the balance")
	fmt.Println("  3. if it differs, try the common paths — most wallets let you choose")
	fmt.Println("  4. record the derivation path alongside the backup, always")

	fmt.Println("\nand the security consequence: this is why a wrong passphrase")
	fmt.Println("(example 7) is indistinguishable from a wrong path. Both give you")
	fmt.Println("a valid empty wallet and no error at all.")
}
```

**Output:**

```
same 12 words, seven paths:
path                                     address
m/44'/60'/0'/0/0   (standard Ethereum)   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
m/44'/60'/0'/0/1   (next index)          0x70997970C51812dc3A010C7d01b50e0d17dc79C8
m/44'/60'/1'/0/0   (next account)        0x8C8d35429F74ec245F8Ef2f4Fd1e551cFF97d650
m/44'/60'/0'/1/0   (change branch)       0x4b39F7b0624b9dB86AD293686bc38B903142dbBc
m/44'/0'/0'/0/0    (Bitcoin coin type)   0xE2Bd0016af6738548F37234744532CA93d0aDEa1
m/44'/60'/0'/0     (one level short)     0x1e59ce931B4CFea3fe4B875411e280e173cB7A9C
m/0'/0/0           (an old convention)   0xB4b458bCcd37C8b9Eb229acB65F40122E2C23511

none of these failed. All seven are valid wallets with valid
addresses, and six of them have never held anything.

when this bites
  you restore a seed phrase into a different wallet app
  the new app uses a different default path
  the balance shows zero and the phrase looks wrong
  it is not wrong — you are looking at a different branch of the tree

how to diagnose it
  1. write down the ADDRESS you expect, not just the phrase
  2. on restore, check the address before trusting the balance
  3. if it differs, try the common paths — most wallets let you choose
  4. record the derivation path alongside the backup, always

and the security consequence: this is why a wrong passphrase
(example 7) is indistinguishable from a wrong path. Both give you
a valid empty wallet and no error at all.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
