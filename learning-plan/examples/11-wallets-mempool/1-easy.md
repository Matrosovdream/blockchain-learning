# Step 11 — Wallets, Fees & the Mempool · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                                # first time only
go get github.com/ethereum/go-ethereum@latest      # secp256k1 sign/verify
go get golang.org/x/crypto@latest                  # scrypt, RIPEMD-160
go get github.com/btcsuite/btcd@v0.24.2            # pin: v0.26+ split into /v2 modules
go get github.com/btcsuite/btcd/btcutil@latest     # BIP-32, example 1 only
go get github.com/tyler-smith/go-bip39@latest      # example 1 only
# paste the example into main.go, then:
go run .
```

No chain and no node. Every key comes from the **published Hardhat/anvil test mnemonic**, every
simulation is seeded, and nothing is timed — so all output reproduces exactly. Examples 3 and 11–17
need no dependencies at all.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. What a wallet actually holds

`🟢 easy` · *Wallets*

A wallet holds no coins. It holds **keys** and a **view** — which unspent outputs those keys can open. The coins live in the UTXO set, which belongs to the network. Lose the view and you rescan; lose the keys and the money stays perfectly visible to everyone, forever.

**Steps:**

1. Derive an account key from the published test mnemonic and neuter it into a watch-only xpub.
2. Print the receive and change branches side by side — two chains, `m/44'/60'/0'/0/i` and `.../1/i`.
3. Confirm the watch-only wallet derives the same 50 addresses and can sign none of them.
4. Rescan with the gap limit and watch index 40 stay invisible.
5. Read why that is behind most 'my funds vanished after restore' reports.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"fmt"

	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	bip39 "github.com/tyler-smith/go-bip39"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// A wallet holds no coins. It holds two things:
//
//   1. KEYS      — the ability to produce signatures
//   2. A VIEW    — which unspent outputs on the chain those keys can open
//
// The coins are in the UTXO set (lesson 10), which belongs to the network.
// Lose the view and you rescan; lose the keys and the coins are gone forever
// while remaining perfectly visible to everyone.
//
// This example builds the address book half: derived addresses, the gap
// limit, and the watch-only wallet that can see but not spend.
// ===========================================================================

// The published Hardhat/anvil test mnemonic. TEST ONLY.
const mnemonic = "test test test test test test test test test test test junk"

// Wallet is the spending side: it can derive private keys.
type Wallet struct {
	acct *hdkeychain.ExtendedKey // m/44'/60'/0'
}

// WatchOnly is the same wallet with the private half removed. It can derive
// every address, recognise every payment, and sign nothing.
type WatchOnly struct {
	acct *hdkeychain.ExtendedKey // the xpub
}

func open() (*Wallet, *WatchOnly) {
	seed := bip39.NewSeed(mnemonic, "")
	master, err := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)
	if err != nil {
		panic(err)
	}
	acct := master
	for _, i := range []uint32{
		hdkeychain.HardenedKeyStart + 44,
		hdkeychain.HardenedKeyStart + 60,
		hdkeychain.HardenedKeyStart + 0,
	} {
		if acct, err = acct.Derive(i); err != nil {
			panic(err)
		}
	}
	pub, err := acct.Neuter() // strip the private half
	if err != nil {
		panic(err)
	}
	return &Wallet{acct}, &WatchOnly{pub}
}

// address derives m/<chain>/<index> under the account key. chain 0 is the
// receive chain, chain 1 is the CHANGE chain — a separate branch so change
// addresses never collide with ones you have handed out.
func derive(k *hdkeychain.ExtendedKey, chain, index uint32) []byte {
	c, err := k.Derive(chain)
	if err != nil {
		panic(err)
	}
	leaf, err := c.Derive(index)
	if err != nil {
		panic(err)
	}
	pub, err := leaf.ECPubKey()
	if err != nil {
		panic(err)
	}
	return hash160(pub.SerializeCompressed())
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func (w *Wallet) Address(chain, index uint32) []byte    { return derive(w.acct, chain, index) }
func (v *WatchOnly) Address(chain, index uint32) []byte { return derive(v.acct, chain, index) }

func (w *Wallet) CanSign() bool    { return w.acct.IsPrivate() }
func (v *WatchOnly) CanSign() bool { return v.acct.IsPrivate() }

// ------------------------------------------------------------- the gap limit

// A wallet does not know how many addresses it has used — the chain does.
// Scan forward until GapLimit consecutive addresses have never been paid,
// then stop. Hand out addresses beyond that gap and a restored wallet will
// not find the money.
const GapLimit = 20

func scan(v *WatchOnly, seen func(pkh []byte) bool) (used []uint32, scanned int) {
	gap := 0
	for i := uint32(0); gap < GapLimit; i++ {
		scanned++
		if seen(v.Address(0, i)) {
			used = append(used, i)
			gap = 0
		} else {
			gap++
		}
	}
	return used, scanned
}

func main() {
	w, v := open()

	fmt.Println("=== one account key, two objects ===")
	fmt.Printf("  wallet     can sign: %v\n", w.CanSign())
	fmt.Printf("  watch-only can sign: %v\n", v.CanSign())
	fmt.Println("  Both derive the SAME addresses — that is the whole point of")
	fmt.Println("  BIP-32's public derivation (lesson 07).")

	fmt.Println("\n=== the address book ===")
	fmt.Printf("  %-6s %-44s %s\n", "index", "receive  m/44'/60'/0'/0/i", "change  m/44'/60'/0'/1/i")
	for i := uint32(0); i < 4; i++ {
		fmt.Printf("  %-6d %-44x %x\n", i, w.Address(0, i), w.Address(1, i))
	}
	fmt.Println("\n  Two branches. Change never lands on an address you gave out,")
	fmt.Println("  which keeps the payment and the change distinguishable to YOU")
	fmt.Println("  and, ideally, to nobody else (lesson 10, example 15).")

	fmt.Println("\n=== the watch-only wallet derives the same book ===")
	same := true
	for i := uint32(0); i < 50; i++ {
		if !bytes.Equal(w.Address(0, i), v.Address(0, i)) {
			same = false
		}
	}
	fmt.Printf("  first 50 receive addresses identical: %v\n", same)
	fmt.Println("  This is what a custody team runs on the internet-facing box:")
	fmt.Println("  it can watch deposits, build unsigned transactions and reconcile")
	fmt.Println("  balances, and an attacker who takes the whole machine gets no")
	fmt.Println("  ability to move anything. The signing key stays offline.")

	fmt.Println("\n=== rescanning, and the gap limit ===")
	// Pretend the chain has paid these indices. Index 3 is inside the gap;
	// index 40 is beyond it.
	paid := map[string]bool{}
	for _, i := range []uint32{0, 1, 3, 40} {
		paid[fmt.Sprintf("%x", v.Address(0, i))] = true
	}
	used, scanned := scan(v, func(pkh []byte) bool { return paid[fmt.Sprintf("%x", pkh)] })
	fmt.Printf("  chain has paid indices : 0, 1, 3, 40\n")
	fmt.Printf("  rescan found           : %v\n", used)
	fmt.Printf("  addresses derived      : %d (stopped after %d unused in a row)\n", scanned, GapLimit)
	fmt.Println()
	fmt.Println("  Index 40 is invisible. Twenty consecutive unused addresses is")
	fmt.Println("  the agreed stopping point (BIP-44), and a wallet that hands out")
	fmt.Println("  address 40 without using 4..39 has made money that a restored")
	fmt.Println("  wallet will never find. Every 'my funds vanished after restore'")
	fmt.Println("  report is either this or a wrong derivation path (lesson 07).")

	fmt.Println("\n=== what the wallet still needs ===")
	fmt.Println("  Keys: example 2 puts them on disk, encrypted.")
	fmt.Println("  A view: the UTXO set filtered to these addresses, which is what")
	fmt.Println("  example 6 spends from.")
	fmt.Println()
	fmt.Println("  Modelled as an interface, so the signer can be a file, a")
	fmt.Println("  hardware device or a remote HSM without the spend path caring:")
	fmt.Println()
	fmt.Println("      type Signer interface {")
	fmt.Println("          PubKey(chain, index uint32) []byte")
	fmt.Println("          Sign(chain, index uint32, hash []byte) ([]byte, error)")
	fmt.Println("      }")
}
```

**Output:**

```
=== one account key, two objects ===
  wallet     can sign: true
  watch-only can sign: false
  Both derive the SAME addresses — that is the whole point of
  BIP-32's public derivation (lesson 07).

=== the address book ===
  index  receive  m/44'/60'/0'/0/i                    change  m/44'/60'/0'/1/i
  0      a55476015c13afb8afb92160329a8cde976f1f2e     5b648efe5f2a3e93a3aa294ba362d332798d1a86
  1      c4841b8f5e6f10a38dc5e672e183ffd9be0cd12f     f8c0d6526b1c133e11f254f9b68a49b7783e6452
  2      84a3b88b8d059d50fe0cdf82f3dca0397f0255c1     8e4e2c338fdf3548dd0db2a532571cf2d1796abd
  3      245289ae1d4ab1b27e13e44e77a3ce0ebb2f445c     10be72bdfe27548c044a3f3dcb2305d1d90f32ec

  Two branches. Change never lands on an address you gave out,
  which keeps the payment and the change distinguishable to YOU
  and, ideally, to nobody else (lesson 10, example 15).

=== the watch-only wallet derives the same book ===
  first 50 receive addresses identical: true
  This is what a custody team runs on the internet-facing box:
  it can watch deposits, build unsigned transactions and reconcile
  balances, and an attacker who takes the whole machine gets no
  ability to move anything. The signing key stays offline.

=== rescanning, and the gap limit ===
  chain has paid indices : 0, 1, 3, 40
  rescan found           : [0 1 3]
  addresses derived      : 24 (stopped after 20 unused in a row)

  Index 40 is invisible. Twenty consecutive unused addresses is
  the agreed stopping point (BIP-44), and a wallet that hands out
  address 40 without using 4..39 has made money that a restored
  wallet will never find. Every 'my funds vanished after restore'
  report is either this or a wrong derivation path (lesson 07).

=== what the wallet still needs ===
  Keys: example 2 puts them on disk, encrypted.
  A view: the UTXO set filtered to these addresses, which is what
  example 6 spends from.

  Modelled as an interface, so the signer can be a file, a
  hardware device or a remote HSM without the spend path caring:

      type Signer interface {
          PubKey(chain, index uint32) []byte
          Sign(chain, index uint32, hash []byte) ([]byte, error)
      }
```

---

## 2. A key on disk

`🟢 easy` · *Keys*

Putting a key on disk. Four things have to be right and three of them are not cryptography: derive slowly from the password, seal with an AEAD, write atomically with the right permissions, and never let the secret reach a log line.

**Steps:**

1. Write a `Secret` type that redacts under `%v`, `%s`, `%x`, `%+v` and `json.Marshal`.
2. Seal a key with scrypt at N=2¹⁸ and AES-256-GCM, authenticating the address as additional data.
3. Write it atomically: temp file, chmod 0600, fsync, rename, fsync the directory.
4. Reload it and re-derive the same address.
5. Try a wrong password, a flipped bit and a relabelled address — and see all three fail identically.

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"path/filepath"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
	"golang.org/x/crypto/scrypt"
)

// ===========================================================================
// Putting a key on disk. Four things have to be right, and three of them are
// not cryptography:
//
//   1. derive a key from the password SLOWLY (scrypt), so a stolen file is
//      not a stolen wallet
//   2. seal with an AEAD (AES-GCM), so a modified file fails loudly
//   3. write it atomically and with the right permissions, so a crash or a
//      curious process cannot get at it
//   4. never let the secret reach a log line
//
// The format below is a simplified Web3 Secret Storage (keystore v3) — the
// same shape geth writes. Lesson 44 does the cryptography properly.
// ===========================================================================

// ------------------------------------------------------------- 4. redaction

// Secret wraps bytes that must never be printed. It satisfies fmt.Stringer,
// fmt.Formatter and json.Marshaler, which between them cover %v, %s, %x,
// %+v, log.Print and encoding/json.
type Secret []byte

func (s Secret) String() string { return "<redacted>" }

func (s Secret) Format(f fmt.State, verb rune) {
	// Without this, %x would happily print the key.
	fmt.Fprint(f, "<redacted>")
}

func (s Secret) MarshalJSON() ([]byte, error) { return []byte(`"<redacted>"`), nil }

// Reveal is the only way out, and it is deliberately ugly to type.
func (s Secret) Reveal() []byte { return []byte(s) }

// --------------------------------------------------------- the file format

type KDFParams struct {
	N     int    `json:"n"`
	R     int    `json:"r"`
	P     int    `json:"p"`
	DKLen int    `json:"dklen"`
	Salt  string `json:"salt"`
}

type Crypto struct {
	Cipher     string    `json:"cipher"`
	Nonce      string    `json:"nonce"`
	Ciphertext string    `json:"ciphertext"`
	KDF        string    `json:"kdf"`
	KDFParams  KDFParams `json:"kdfparams"`
}

type Keystore struct {
	Version int    `json:"version"`
	Address string `json:"address"` // so you can identify the file without unlocking it
	Crypto  Crypto `json:"crypto"`
}

// scrypt at N=2^18 costs about 256 MB and a second of CPU. That is the point:
// it is charged to every guess an attacker makes, and paid once by you.
var params = KDFParams{N: 1 << 18, R: 8, P: 1, DKLen: 32}

var ErrBadPassword = errors.New("wrong password or corrupted file")

func Seal(key Secret, address []byte, password []byte) (*Keystore, error) {
	salt := make([]byte, 32)
	if _, err := rand.Read(salt); err != nil {
		return nil, err
	}
	dk, err := scrypt.Key(password, salt, params.N, params.R, params.P, params.DKLen)
	if err != nil {
		return nil, err
	}
	block, err := aes.NewCipher(dk)
	if err != nil {
		return nil, err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	nonce := make([]byte, gcm.NonceSize())
	if _, err := rand.Read(nonce); err != nil {
		return nil, err
	}
	// The address is authenticated as additional data, so a file cannot be
	// relabelled to point at a different account.
	ct := gcm.Seal(nil, nonce, key.Reveal(), address)

	p := params
	p.Salt = hex.EncodeToString(salt)
	return &Keystore{
		Version: 3,
		Address: hex.EncodeToString(address),
		Crypto: Crypto{
			Cipher: "aes-256-gcm", Nonce: hex.EncodeToString(nonce),
			Ciphertext: hex.EncodeToString(ct), KDF: "scrypt", KDFParams: p,
		},
	}, nil
}

func (k *Keystore) Open(password []byte) (Secret, error) {
	salt, err := hex.DecodeString(k.Crypto.KDFParams.Salt)
	if err != nil {
		return nil, err
	}
	p := k.Crypto.KDFParams
	dk, err := scrypt.Key(password, salt, p.N, p.R, p.P, p.DKLen)
	if err != nil {
		return nil, err
	}
	block, err := aes.NewCipher(dk)
	if err != nil {
		return nil, err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	nonce, _ := hex.DecodeString(k.Crypto.Nonce)
	ct, _ := hex.DecodeString(k.Crypto.Ciphertext)
	addr, _ := hex.DecodeString(k.Address)

	pt, err := gcm.Open(nil, nonce, ct, addr)
	if err != nil {
		return nil, ErrBadPassword // GCM says "authentication failed"; say something useful
	}
	return Secret(pt), nil
}

// ------------------------------------------------------- 3. writing it down

// WriteAtomic is the pattern for any file you cannot afford to find truncated:
// write a temp file in the SAME directory, fsync it, then rename over the
// target. Rename within a directory is atomic, so a reader sees the old file
// or the new one and never a half-written one.
func WriteAtomic(path string, data []byte, mode os.FileMode) error {
	dir := filepath.Dir(path)
	f, err := os.CreateTemp(dir, ".keystore-*.tmp")
	if err != nil {
		return err
	}
	tmp := f.Name()
	defer os.Remove(tmp) // no-op once the rename succeeds

	if err := f.Chmod(mode); err != nil { // before any bytes are in it
		f.Close()
		return err
	}
	if _, err := f.Write(data); err != nil {
		f.Close()
		return err
	}
	if err := f.Sync(); err != nil { // the bytes, not just the page cache
		f.Close()
		return err
	}
	if err := f.Close(); err != nil {
		return err
	}
	if err := os.Rename(tmp, path); err != nil {
		return err
	}
	// The rename itself needs to be durable, which means fsyncing the
	// DIRECTORY. Skipping this is the classic "the file is empty after a
	// power cut" bug.
	d, err := os.Open(dir)
	if err != nil {
		return err
	}
	defer d.Close()
	return d.Sync()
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func main() {
	// TEST ONLY — the published anvil account 0 key.
	priv, err := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	if err != nil {
		panic(err)
	}
	key := Secret(crypto.FromECDSA(priv))
	addr := hash160(crypto.CompressPubkey(&priv.PublicKey))
	password := []byte("correct horse battery staple")

	fmt.Println("=== 4. the key never reaches a log line ===")
	fmt.Printf("  fmt.Println      : %v\n", key)
	fmt.Printf("  %%s               : %s\n", key)
	fmt.Printf("  %%x               : %x\n", key)
	fmt.Printf("  %%+v in a struct  : %+v\n", struct {
		Name string
		Key  Secret
	}{"wallet", key})
	j, _ := json.Marshal(map[string]any{"key": key})
	fmt.Printf("  json.Marshal     : %s\n", j)
	fmt.Printf("  len(key.Reveal()): %d bytes — still there when you ask for it\n", len(key.Reveal()))

	fmt.Println("\n=== 1 & 2. sealing ===")
	fmt.Printf("  kdf     scrypt N=2^%d r=%d p=%d dklen=%d  (~256 MB, ~1s per guess)\n",
		18, params.R, params.P, params.DKLen)
	fmt.Println("  cipher  aes-256-gcm, with the address as additional data")
	ks, err := Seal(key, addr, password)
	if err != nil {
		panic(err)
	}
	fmt.Printf("  address %s\n", ks.Address)
	fmt.Printf("  salt    %d bytes (random per file)\n", len(ks.Crypto.KDFParams.Salt)/2)
	fmt.Printf("  nonce   %d bytes (random per file)\n", len(ks.Crypto.Nonce)/2)
	fmt.Printf("  cipher  %d bytes = 32 key + 16 GCM tag\n", len(ks.Crypto.Ciphertext)/2)

	fmt.Println("\n=== 3. writing it down ===")
	dir, err := os.MkdirTemp("", "keystore")
	if err != nil {
		panic(err)
	}
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "UTC--wallet.json")

	blob, err := json.MarshalIndent(ks, "", "  ")
	if err != nil {
		panic(err)
	}
	if err := WriteAtomic(path, blob, 0o600); err != nil {
		panic(err)
	}
	st, err := os.Stat(path)
	if err != nil {
		panic(err)
	}
	fmt.Printf("  wrote %d bytes, mode %v\n", st.Size(), st.Mode().Perm())
	fmt.Println("  temp file -> chmod -> write -> fsync -> rename -> fsync(dir)")
	leftovers, _ := filepath.Glob(filepath.Join(dir, ".keystore-*"))
	fmt.Printf("  temp files left behind: %d\n", len(leftovers))

	fmt.Println("\n=== reloading ===")
	raw, err := os.ReadFile(path)
	if err != nil {
		panic(err)
	}
	var back Keystore
	if err := json.Unmarshal(raw, &back); err != nil {
		panic(err)
	}
	got, err := back.Open(password)
	if err != nil {
		panic(err)
	}
	reloaded, err := crypto.ToECDSA(got.Reveal())
	if err != nil {
		panic(err)
	}
	reAddr := hash160(crypto.CompressPubkey(&reloaded.PublicKey))
	fmt.Printf("  unlocked, re-derived address: %x\n", reAddr)
	fmt.Printf("  matches the original:         %v\n", ks.Address == hex.EncodeToString(reAddr))

	fmt.Println("\n=== the failure modes ===")
	_, err = back.Open([]byte("hunter2"))
	fmt.Printf("  wrong password        : %v\n", err)

	tampered := back
	ctb, _ := hex.DecodeString(back.Crypto.Ciphertext)
	ctb[3] ^= 1
	tampered.Crypto.Ciphertext = hex.EncodeToString(ctb)
	_, err = tampered.Open(password)
	fmt.Printf("  one flipped bit       : %v\n", err)

	relabelled := back
	relabelled.Address = hex.EncodeToString(make([]byte, 20))
	_, err = relabelled.Open(password)
	fmt.Printf("  address field changed : %v\n", err)
	fmt.Println()
	fmt.Println("  All three fail the same way, and that is the value of an AEAD:")
	fmt.Println("  a wrong password and a corrupted file are indistinguishable, so")
	fmt.Println("  there is no oracle to probe. Plain AES-CTR would have decrypted")
	fmt.Println("  the tampered file into 32 bytes of garbage — a perfectly valid")
	fmt.Println("  private key for an account holding nothing.")

	fmt.Println("\n=== what this is not ===")
	fmt.Println("  This protects a key AT REST. It does nothing about a key in")
	fmt.Println("  memory, in a swap file, in a core dump, or in the shell history")
	fmt.Println("  of whoever typed the password. Lessons 32 and 44 cover those,")
	fmt.Println("  and the honest answer for real money is a device that never")
	fmt.Println("  exports the key at all.")
}
```

**Output:**

```
=== 4. the key never reaches a log line ===
  fmt.Println      : <redacted>
  %s               : <redacted>
  %x               : <redacted>
  %+v in a struct  : {Name:wallet Key:<redacted>}
  json.Marshal     : {"key":"\u003credacted\u003e"}
  len(key.Reveal()): 32 bytes — still there when you ask for it

=== 1 & 2. sealing ===
  kdf     scrypt N=2^18 r=8 p=1 dklen=32  (~256 MB, ~1s per guess)
  cipher  aes-256-gcm, with the address as additional data
  address a55476015c13afb8afb92160329a8cde976f1f2e
  salt    32 bytes (random per file)
  nonce   12 bytes (random per file)
  cipher  48 bytes = 32 key + 16 GCM tag

=== 3. writing it down ===
  wrote 475 bytes, mode -rw-------
  temp file -> chmod -> write -> fsync -> rename -> fsync(dir)
  temp files left behind: 0

=== reloading ===
  unlocked, re-derived address: a55476015c13afb8afb92160329a8cde976f1f2e
  matches the original:         true

=== the failure modes ===
  wrong password        : wrong password or corrupted file
  one flipped bit       : wrong password or corrupted file
  address field changed : wrong password or corrupted file

  All three fail the same way, and that is the value of an AEAD:
  a wrong password and a corrupted file are indistinguishable, so
  there is no oracle to probe. Plain AES-CTR would have decrypted
  the tampered file into 32 bytes of garbage — a perfectly valid
  private key for an account holding nothing.

=== what this is not ===
  This protects a key AT REST. It does nothing about a key in
  memory, in a swap file, in a core dump, or in the shell history
  of whoever typed the password. Lessons 32 and 44 cover those,
  and the honest answer for real money is a device that never
  exports the key at all.
```

---

## 3. Fee rate, not fee

`🟢 easy` · *Fees*

A fee is not a price, it is a **bid for space**. Blocks are capped by size, so a miner is solving 'most fees per byte'. The number that decides whether you get mined is the fee RATE, and the absolute fee is almost irrelevant.

**Steps:**

1. Price five transaction shapes and see an input cost about twice what an output costs.
2. Sort a mempool by absolute fee, then by rate, and watch the biggest payer fall to second-last.
3. Fill a block both ways and compare the revenue.
4. Do the same arithmetic in gas and gwei, and see it is the same arithmetic.
5. Take away the two mistakes: picking a fee instead of a rate, and estimating before signing.

```go
package main

import (
	"fmt"
	"sort"
)

// ===========================================================================
// A fee is not a price. It is a BID for space in the next block.
//
// Blocks are capped by size, not by transaction count, so a miner filling one
// is solving "most fees per byte", not "biggest fees". Which means the number
// that decides whether you get mined is the fee RATE — satoshi per vbyte,
// gwei per gas — and the absolute fee is almost irrelevant.
// ===========================================================================

// A P2WPKH vsize model, in vbytes. Good enough to budget with, and the same
// shape as the one lesson 10 used for coin selection.
const (
	overheadVSize = 11
	inputVSize    = 68
	outputVSize   = 31
)

func VSize(nIn, nOut int) int64 {
	return int64(overheadVSize + nIn*inputVSize + nOut*outputVSize)
}

// Fee is what you pay: rate times size. Always compute it this way round —
// choose a rate, measure the size, multiply. Never pick a round number.
func Fee(vsize, rate int64) int64 { return vsize * rate }

type Tx struct {
	Name  string
	Fee   int64
	VSize int64
}

func (t Tx) Rate() float64 { return float64(t.Fee) / float64(t.VSize) }

// fill is what a miner does: walk the list in order, take anything that
// still fits. The ORDER is the entire strategy.
func fill(list []Tx, capacity int64) (fees, used int64, names []string) {
	for _, t := range list {
		if used+t.VSize > capacity {
			continue
		}
		used += t.VSize
		fees += t.Fee
		names = append(names, t.Name)
	}
	return
}

func table(list []Tx) {
	fmt.Printf("  %-18s %-9s %-12s %s\n", "transaction", "vbytes", "fee (sat)", "sat/vB")
	for _, t := range list {
		fmt.Printf("  %-18s %-9d %-12d %.1f\n", t.Name, t.VSize, t.Fee, t.Rate())
	}
}

func main() {
	fmt.Println("=== the size of a transaction is its shape ===")
	fmt.Printf("  %-38s %-8s %s\n", "shape", "vbytes", "fee at 20 sat/vB")
	for _, s := range []struct {
		name      string
		nIn, nOut int
	}{
		{"1 input, 1 output (sweep)", 1, 1},
		{"1 input, 2 outputs (payment + change)", 1, 2},
		{"2 inputs, 2 outputs", 2, 2},
		{"10 inputs, 2 outputs (consolidating)", 10, 2},
		{"1 input, 100 outputs (batch payout)", 1, 100},
	} {
		v := VSize(s.nIn, s.nOut)
		fmt.Printf("  %-38s %-8d %d sat\n", s.name, v, Fee(v, 20))
	}
	fmt.Println("\n  An input costs about twice what an output costs, because it")
	fmt.Println("  carries a signature and a public key. That is why consolidating")
	fmt.Println("  is expensive and batching is cheap (lesson 10, example 14).")

	// A mempool. Note the exchange batch pays by far the largest fee.
	pool := []Tx{
		{"exchange batch", 40_000, VSize(1, 60)},
		{"urgent payment", 30_000, VSize(2, 2)},
		{"merchant payout", 29_000, VSize(2, 20)},
		{"swap deposit", 15_000, VSize(3, 2)},
		{"wallet refill", 12_000, VSize(2, 2)},
		{"someone's coffee", 10_000, VSize(1, 2)},
		{"a lazy default", 2_260, VSize(1, 2)},
	}

	byFee := append([]Tx(nil), pool...)
	sort.SliceStable(byFee, func(i, j int) bool { return byFee[i].Fee > byFee[j].Fee })
	byRate := append([]Tx(nil), pool...)
	sort.SliceStable(byRate, func(i, j int) bool { return byRate[i].Rate() > byRate[j].Rate() })

	fmt.Println("\n=== sorted by absolute fee ===")
	table(byFee)
	fmt.Println("\n  The exchange batch looks like the best customer in the pool.")

	fmt.Println("\n=== sorted by fee rate ===")
	table(byRate)
	fmt.Println("\n  It is now second from last. Someone's coffee outbids it while")
	fmt.Println("  paying a quarter as much, because it asks for a fourteenth of")
	fmt.Println("  the space.")

	fmt.Println("\n=== what a miner collects ===")
	const capacity = 2000 // vbytes. Bitcoin's real cap is 4,000,000 weight units.
	fmt.Printf("  block space: %d vbytes\n\n", capacity)
	var best int64
	for _, c := range []struct {
		label string
		list  []Tx
	}{{"greedy by absolute fee", byFee}, {"greedy by fee rate", byRate}} {
		fees, used, names := fill(c.list, capacity)
		if fees > best {
			best = fees
		}
		fmt.Printf("  %-24s %6d sat from %d vbytes\n", c.label, fees, used)
		fmt.Printf("  %-24s %v\n\n", "", names)
	}
	feeFirst, _, _ := fill(byFee, capacity)
	fmt.Printf("  Same transactions, same block, %.1fx the revenue. Every miner on\n",
		float64(best)/float64(feeFirst))
	fmt.Println("  every chain sorts by rate, so every wallet has to bid by rate.")

	fmt.Println("\n=== the same idea on Ethereum ===")
	fmt.Println("  Blocks are capped in GAS rather than bytes, and the bid is priced")
	fmt.Println("  in gwei per gas. The arithmetic is identical:")
	fmt.Println()
	fmt.Printf("    %-26s %-10s %s\n", "operation", "gas", "cost at 30 gwei")
	for _, g := range []struct {
		name string
		gas  int64
	}{
		{"plain ETH transfer", 21_000},
		{"ERC-20 transfer", 65_000},
		{"Uniswap v2 swap", 150_000},
		{"deploying a contract", 1_200_000},
	} {
		gwei := g.gas * 30
		fmt.Printf("    %-26s %-10d 0.%09d ETH\n", g.name, g.gas, gwei)
	}
	fmt.Println()
	fmt.Println("  A transfer and a swap paying the same absolute fee are not the")
	fmt.Println("  same bid: the swap is asking for seven times as much block.")

	fmt.Println("\n=== the two mistakes ===")
	fmt.Println("  1. Picking a fee. 'I'll pay 5000 sat' is a good rate on a small")
	fmt.Println("     transaction and a terrible one on a big one. Pick a RATE and")
	fmt.Println("     let the size decide the fee.")
	fmt.Println("  2. Estimating the size before signing. Signatures are most of an")
	fmt.Println("     input, and an estimate that omits them under-pays by more than")
	fmt.Println("     half — example 4.")
}
```

**Output:**

```
=== the size of a transaction is its shape ===
  shape                                  vbytes   fee at 20 sat/vB
  1 input, 1 output (sweep)              110      2200 sat
  1 input, 2 outputs (payment + change)  141      2820 sat
  2 inputs, 2 outputs                    209      4180 sat
  10 inputs, 2 outputs (consolidating)   753      15060 sat
  1 input, 100 outputs (batch payout)    3179     63580 sat

  An input costs about twice what an output costs, because it
  carries a signature and a public key. That is why consolidating
  is expensive and batching is cheap (lesson 10, example 14).

=== sorted by absolute fee ===
  transaction        vbytes    fee (sat)    sat/vB
  exchange batch     1939      40000        20.6
  urgent payment     209       30000        143.5
  merchant payout    767       29000        37.8
  swap deposit       277       15000        54.2
  wallet refill      209       12000        57.4
  someone's coffee   141       10000        70.9
  a lazy default     141       2260         16.0

  The exchange batch looks like the best customer in the pool.

=== sorted by fee rate ===
  transaction        vbytes    fee (sat)    sat/vB
  urgent payment     209       30000        143.5
  someone's coffee   141       10000        70.9
  wallet refill      209       12000        57.4
  swap deposit       277       15000        54.2
  merchant payout    767       29000        37.8
  exchange batch     1939      40000        20.6
  a lazy default     141       2260         16.0

  It is now second from last. Someone's coffee outbids it while
  paying a quarter as much, because it asks for a fourteenth of
  the space.

=== what a miner collects ===
  block space: 2000 vbytes

  greedy by absolute fee    40000 sat from 1939 vbytes
                           [exchange batch]

  greedy by fee rate        98260 sat from 1744 vbytes
                           [urgent payment someone's coffee wallet refill swap deposit merchant payout a lazy default]

  Same transactions, same block, 2.5x the revenue. Every miner on
  every chain sorts by rate, so every wallet has to bid by rate.

=== the same idea on Ethereum ===
  Blocks are capped in GAS rather than bytes, and the bid is priced
  in gwei per gas. The arithmetic is identical:

    operation                  gas        cost at 30 gwei
    plain ETH transfer         21000      0.000630000 ETH
    ERC-20 transfer            65000      0.001950000 ETH
    Uniswap v2 swap            150000     0.004500000 ETH
    deploying a contract       1200000    0.036000000 ETH

  A transfer and a swap paying the same absolute fee are not the
  same bid: the swap is asking for seven times as much block.

=== the two mistakes ===
  1. Picking a fee. 'I'll pay 5000 sat' is a good rate on a small
     transaction and a terrible one on a big one. Pick a RATE and
     let the size decide the fee.
  2. Estimating the size before signing. Signatures are most of an
     input, and an estimate that omits them under-pays by more than
     half — example 4.
```

---

## 4. Estimating a size that does not exist yet

`🟢 easy` · *Fees*

The fee depends on the size, the size depends on the signatures, and the signatures cannot exist until the fee is decided. So a wallet has to estimate the SIGNED size of a transaction that is not signed yet — and estimating the unsigned size under-pays by more than half.

**Steps:**

1. Measure one transaction before and after signing, and see what the signatures add.
2. Pay for the unsigned size at 20 sat/byte and watch the achieved rate come out at 9.
3. Run the estimator across six shapes and confirm it is exact.
4. Read why a real Bitcoin estimate never is: DER pads r about half the time.
5. Work through weight units and the segwit discount, and why a P2WPKH input costs 68 vbytes and its legacy equivalent 148.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// The fee depends on the size. The size depends on the signatures. And the
// signatures cannot be made until the fee is decided, because the fee decides
// the change output and the change output is signed over (lesson 10).
//
// So a wallet has to ESTIMATE the signed size before it exists. Estimate it
// as the unsigned size and you under-pay by more than half.
// ===========================================================================
// --------------------------------------------------------- the transaction

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint
	Signature []byte
	PubKey    []byte
}

type TxOutput struct {
	Value      int64
	PubKeyHash []byte
}

type Transaction struct {
	Inputs  []TxInput
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	var b bytes.Buffer
	binary.Write(&b, binary.BigEndian, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b.Write(in.Prev.TxID[:])
		binary.Write(&b, binary.BigEndian, in.Prev.Index)
		writeBytes(&b, in.Signature)
		writeBytes(&b, in.PubKey)
	}
	binary.Write(&b, binary.BigEndian, uint32(len(t.Outputs)))
	for _, out := range t.Outputs {
		binary.Write(&b, binary.BigEndian, out.Value)
		writeBytes(&b, out.PubKeyHash)
	}
	return b.Bytes()
}

func writeBytes(b *bytes.Buffer, p []byte) {
	binary.Write(b, binary.BigEndian, uint32(len(p)))
	b.Write(p)
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

func (t *Transaction) TrimmedCopy() *Transaction {
	ins := make([]TxInput, len(t.Inputs))
	for i, in := range t.Inputs {
		ins[i] = TxInput{Prev: in.Prev}
	}
	outs := make([]TxOutput, len(t.Outputs))
	for i, o := range t.Outputs {
		outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
	}
	return &Transaction{Inputs: ins, Outputs: outs}
}

func (t *Transaction) SigHash(i int, prevPubKeyHash []byte) [32]byte {
	c := t.TrimmedCopy()
	c.Inputs[i].PubKey = prevPubKeyHash
	return c.TxID()
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

// ------------------------------------------------------------- the estimate

// Our inputs carry a 65-byte signature and a 33-byte compressed public key,
// plus the two 4-byte length prefixes the serializer writes.
const (
	sigBytes    = 65
	pubKeyBytes = 33
	perInputSig = sigBytes + pubKeyBytes
)

// EstimateSize returns what the transaction WILL serialize to once signed,
// computed from a transaction that is not signed yet.
func EstimateSize(t *Transaction) int {
	return len(t.Serialize()) + len(t.Inputs)*perInputSig
}

// -------------------------------------------------------------------- setup

const aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY

func key() *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(aliceKey)
	if err != nil {
		panic(err)
	}
	return k
}

func pkh(k *ecdsa.PrivateKey) []byte { return hash160(crypto.CompressPubkey(&k.PublicKey)) }

func build(k *ecdsa.PrivateKey, nIn, nOut int) *Transaction {
	t := &Transaction{}
	for i := 0; i < nIn; i++ {
		var id [32]byte
		id[0] = byte(0xa0 + i)
		t.Inputs = append(t.Inputs, TxInput{Prev: Outpoint{id, 0}})
	}
	for i := 0; i < nOut; i++ {
		t.Outputs = append(t.Outputs, TxOutput{Value: 1000, PubKeyHash: pkh(k)})
	}
	return t
}

func sign(t *Transaction, k *ecdsa.PrivateKey) {
	pub := crypto.CompressPubkey(&k.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, pkh(k))
		sig, err := crypto.Sign(h[:], k)
		if err != nil {
			panic(err)
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
}

func main() {
	k := key()

	fmt.Println("=== one transaction, before and after signing ===")
	t := build(k, 2, 2)
	unsigned := len(t.Serialize())
	estimate := EstimateSize(t)
	sign(t, k)
	actual := len(t.Serialize())

	fmt.Printf("  unsigned serialization : %d bytes\n", unsigned)
	fmt.Printf("  estimate               : %d bytes\n", estimate)
	fmt.Printf("  actual, once signed    : %d bytes\n", actual)
	fmt.Printf("  the signatures added   : %d bytes (%d per input)\n",
		actual-unsigned, (actual-unsigned)/len(t.Inputs))

	fmt.Println("\n=== what estimating too early costs you ===")
	const targetRate = 20 // sat/vB
	fmt.Printf("  target rate: %d sat/vB\n\n", targetRate)
	fmt.Printf("  %-14s %-10s %-12s %-12s %s\n",
		"basis", "size", "fee paid", "real size", "rate achieved")
	for _, c := range []struct {
		name string
		size int
	}{
		{"unsigned", unsigned},
		{"estimated", estimate},
	} {
		fee := int64(c.size) * targetRate
		rate := float64(fee) / float64(actual)
		fmt.Printf("  %-14s %-10d %-12d %-12d %.1f sat/vB\n", c.name, c.size, fee, actual, rate)
	}
	fmt.Println("\n  Paying for the unsigned size buys less than half the rate you")
	fmt.Println("  asked for. At a busy moment that is the difference between the")
	fmt.Println("  next block and next week — and the transaction is already signed,")
	fmt.Println("  so the only fix is to replace it (example 14).")

	fmt.Println("\n=== the estimator, across shapes ===")
	fmt.Printf("  %-8s %-8s %-11s %-9s %-9s %s\n",
		"inputs", "outputs", "unsigned", "estimate", "actual", "error")
	for _, s := range []struct{ in, out int }{
		{1, 1}, {1, 2}, {2, 2}, {5, 2}, {10, 2}, {1, 50},
	} {
		u := build(k, s.in, s.out)
		un, est := len(u.Serialize()), EstimateSize(u)
		sign(u, k)
		act := len(u.Serialize())
		fmt.Printf("  %-8d %-8d %-11d %-9d %-9d %+d\n", s.in, s.out, un, est, act, est-act)
	}
	fmt.Println("\n  Exact, because our signatures are fixed at 65 bytes. Bitcoin's")
	fmt.Println("  are not, and that is the next problem.")

	fmt.Println("\n=== why real estimates are never exact ===")
	fmt.Println("  A Bitcoin signature is DER-encoded, and DER stores r and s as")
	fmt.Println("  SIGNED integers. A 32-byte value whose top bit is set needs a")
	fmt.Println("  0x00 pad in front of it — 33 bytes. Low-s enforcement (lesson 06)")
	fmt.Println("  keeps s below n/2, so s never needs the pad; r needs it about half")
	fmt.Println("  the time. And roughly one value in 256 starts with a zero byte and")
	fmt.Println("  shrinks to 31. So a signature is 71 or 72 bytes, occasionally 70.")
	fmt.Println()
	fmt.Println("  Two consequences:")
	fmt.Println("    - estimate the WORST case (72), or you land under your target")
	fmt.Println("      rate whenever you get unlucky")
	fmt.Println("    - the difference is real money at scale, so some wallets GRIND:")
	fmt.Println("      re-sign with a different nonce until the signature is short.")
	fmt.Println("      Never do this by making the nonce non-deterministic — RFC 6979")
	fmt.Println("      exists for a reason (lesson 06). Grind the change amount or")
	fmt.Println("      the locktime instead, and re-sign deterministically.")

	fmt.Println("\n=== vbytes, weight, and the segwit discount ===")
	fmt.Println("  Since segwit, Bitcoin's cap is 4,000,000 WEIGHT units, and:")
	fmt.Println()
	fmt.Println("      weight = 4 x (base bytes) + 1 x (witness bytes)")
	fmt.Println("      vsize  = ceil(weight / 4)")
	fmt.Println()
	fmt.Println("  Signatures live in the witness, so they are charged a QUARTER of")
	fmt.Println("  what they used to be. That is the whole reason a P2WPKH input")
	fmt.Println("  costs about 68 vbytes while its legacy equivalent costs 148.")
	fmt.Println()
	fmt.Printf("    %-26s %-8s %-10s %s\n", "input type", "base", "witness", "vbytes")
	for _, r := range []struct {
		name          string
		base, witness int
	}{
		{"P2PKH (legacy)", 148, 0},
		{"P2SH-P2WPKH (wrapped)", 64, 108},
		{"P2WPKH (native segwit)", 41, 108},
		{"P2TR (taproot, key path)", 41, 66},
	} {
		weight := 4*r.base + r.witness
		fmt.Printf("    %-26s %-8d %-10d %d\n", r.name, r.base, r.witness, (weight+3)/4)
	}
	fmt.Println()
	fmt.Println("  A fee estimator that does not know which script types it is")
	fmt.Println("  spending is guessing by a factor of two. Ours knows because there")
	fmt.Println("  is only one type; a real one takes the type from the UTXO.")

	fmt.Println("\n=== the rule ===")
	fmt.Println("  Estimate the SIGNED size, from the shape, before choosing the")
	fmt.Println("  change amount. Then sign. Then assert the real size is within a")
	fmt.Println("  byte or two of the estimate — a cheap check that catches every")
	fmt.Println("  future change to your serialization format.")
	final := len(t.Serialize())
	fmt.Printf("\n  assertion on the transaction above: |%d - %d| = %d\n",
		estimate, final, abs(estimate-final))
}

func abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}
```

**Output:**

```
=== one transaction, before and after signing ===
  unsigned serialization : 160 bytes
  estimate               : 356 bytes
  actual, once signed    : 356 bytes
  the signatures added   : 196 bytes (98 per input)

=== what estimating too early costs you ===
  target rate: 20 sat/vB

  basis          size       fee paid     real size    rate achieved
  unsigned       160        3200         356          9.0 sat/vB
  estimated      356        7120         356          20.0 sat/vB

  Paying for the unsigned size buys less than half the rate you
  asked for. At a busy moment that is the difference between the
  next block and next week — and the transaction is already signed,
  so the only fix is to replace it (example 14).

=== the estimator, across shapes ===
  inputs   outputs  unsigned    estimate  actual    error
  1        1        84          182       182       +0
  1        2        116         214       214       +0
  2        2        160         356       356       +0
  5        2        292         782       782       +0
  10       2        512         1492      1492      +0
  1        50       1652        1750      1750      +0

  Exact, because our signatures are fixed at 65 bytes. Bitcoin's
  are not, and that is the next problem.

=== why real estimates are never exact ===
  A Bitcoin signature is DER-encoded, and DER stores r and s as
  SIGNED integers. A 32-byte value whose top bit is set needs a
  0x00 pad in front of it — 33 bytes. Low-s enforcement (lesson 06)
  keeps s below n/2, so s never needs the pad; r needs it about half
  the time. And roughly one value in 256 starts with a zero byte and
  shrinks to 31. So a signature is 71 or 72 bytes, occasionally 70.

  Two consequences:
    - estimate the WORST case (72), or you land under your target
      rate whenever you get unlucky
    - the difference is real money at scale, so some wallets GRIND:
      re-sign with a different nonce until the signature is short.
      Never do this by making the nonce non-deterministic — RFC 6979
      exists for a reason (lesson 06). Grind the change amount or
      the locktime instead, and re-sign deterministically.

=== vbytes, weight, and the segwit discount ===
  Since segwit, Bitcoin's cap is 4,000,000 WEIGHT units, and:

      weight = 4 x (base bytes) + 1 x (witness bytes)
      vsize  = ceil(weight / 4)

  Signatures live in the witness, so they are charged a QUARTER of
  what they used to be. That is the whole reason a P2WPKH input
  costs about 68 vbytes while its legacy equivalent costs 148.

    input type                 base     witness    vbytes
    P2PKH (legacy)             148      0          148
    P2SH-P2WPKH (wrapped)      64       108        91
    P2WPKH (native segwit)     41       108        68
    P2TR (taproot, key path)   41       66         58

  A fee estimator that does not know which script types it is
  spending is guessing by a factor of two. Ours knows because there
  is only one type; a real one takes the type from the UTXO.

=== the rule ===
  Estimate the SIGNED size, from the shape, before choosing the
  change amount. Then sign. Then assert the real size is within a
  byte or two of the estimate — a cheap check that catches every
  future change to your serialization format.

  assertion on the transaction above: |356 - 356| = 0
```

---

## 5. A mempool, ordered by what a miner wants

`🟢 easy` · *The mempool*

The mempool is one node's private waiting room: validated but unconfirmed, never consensus, gone on restart. Two indexes, because there are two questions — 'do I already have this?' and 'what should I mine next?'

**Steps:**

1. Build `map[TxID]*Entry` plus an order by fee rate.
2. Compare rates with integer cross-multiplication, so no two nodes disagree over a rounding.
3. Print the pool with a running cumulative-bytes column and see where the next block's cut line falls.
4. Re-add a transaction and watch the id index reject it in one lookup.
5. Mine three, remove them by txid, and read the list of everything this structure still lacks.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"sort"

	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// The mempool: transactions a node has validated but not yet seen in a block.
//
// It is NOT consensus. It is one node's private waiting room — every node has
// a different one, none of them agree, and none of it survives a restart.
// Two transactions in it can conflict; only a block decides.
//
// Two indexes, because there are two questions:
//   "do I already have this?"   -> map[TxID]*Entry, O(1)
//   "what should I mine next?"  -> an order by FEE RATE
// ===========================================================================
// --------------------------------------------------------- the transaction

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint
	Signature []byte
	PubKey    []byte
}

type TxOutput struct {
	Value      int64
	PubKeyHash []byte
}

type Transaction struct {
	Inputs  []TxInput
	Outputs []TxOutput
}

func (t *Transaction) Serialize() []byte {
	var b bytes.Buffer
	binary.Write(&b, binary.BigEndian, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b.Write(in.Prev.TxID[:])
		binary.Write(&b, binary.BigEndian, in.Prev.Index)
		writeBytes(&b, in.Signature)
		writeBytes(&b, in.PubKey)
	}
	binary.Write(&b, binary.BigEndian, uint32(len(t.Outputs)))
	for _, out := range t.Outputs {
		binary.Write(&b, binary.BigEndian, out.Value)
		writeBytes(&b, out.PubKeyHash)
	}
	return b.Bytes()
}

func writeBytes(b *bytes.Buffer, p []byte) {
	binary.Write(b, binary.BigEndian, uint32(len(p)))
	b.Write(p)
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

func (t *Transaction) TrimmedCopy() *Transaction {
	ins := make([]TxInput, len(t.Inputs))
	for i, in := range t.Inputs {
		ins[i] = TxInput{Prev: in.Prev}
	}
	outs := make([]TxOutput, len(t.Outputs))
	for i, o := range t.Outputs {
		outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
	}
	return &Transaction{Inputs: ins, Outputs: outs}
}

func (t *Transaction) SigHash(i int, prevPubKeyHash []byte) [32]byte {
	c := t.TrimmedCopy()
	c.Inputs[i].PubKey = prevPubKeyHash
	return c.TxID()
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

// ------------------------------------------------------------- the mempool

type Entry struct {
	Tx    *Transaction
	Name  string // for the demo; a real node has no name for a transaction
	VSize int64
	Fee   int64
	Added int64 // sequence number, for expiry and tie-breaking
}

// Rate is the only number that matters for selection. Integer arithmetic:
// compare a.Fee*b.VSize against b.Fee*a.VSize rather than dividing, so two
// nodes never disagree because of a rounding difference.
func (e *Entry) Less(o *Entry) bool {
	l, r := e.Fee*o.VSize, o.Fee*e.VSize
	if l != r {
		return l < r
	}
	return e.Added > o.Added // older first among equals
}

func (e *Entry) RateString() string {
	whole := e.Fee / e.VSize
	frac := (e.Fee % e.VSize) * 10 / e.VSize
	return fmt.Sprintf("%d.%d", whole, frac)
}

type Mempool struct {
	byID  map[[32]byte]*Entry
	bytes int64
	seq   int64
}

func NewMempool() *Mempool { return &Mempool{byID: map[[32]byte]*Entry{}} }

var ErrDuplicate = errors.New("already in the pool")

func (m *Mempool) Add(name string, t *Transaction, vsize, fee int64) error {
	id := t.TxID()
	if _, ok := m.byID[id]; ok {
		return ErrDuplicate
	}
	m.seq++
	m.byID[id] = &Entry{Tx: t, Name: name, VSize: vsize, Fee: fee, Added: m.seq}
	m.bytes += vsize
	return nil
}

func (m *Mempool) Has(id [32]byte) bool { return m.byID[id] != nil }

func (m *Mempool) Remove(id [32]byte) {
	if e, ok := m.byID[id]; ok {
		m.bytes -= e.VSize
		delete(m.byID, id)
	}
}

// ByRate returns the pool best-first. This is the order a miner walks
// (example 12) and the reverse of the order an evictor walks (example 13).
func (m *Mempool) ByRate() []*Entry {
	out := make([]*Entry, 0, len(m.byID))
	for _, e := range m.byID {
		out = append(out, e)
	}
	// A total order, so the output does not depend on map iteration.
	sort.Slice(out, func(i, j int) bool { return out[j].Less(out[i]) })
	return out
}

func (m *Mempool) Len() int     { return len(m.byID) }
func (m *Mempool) Bytes() int64 { return m.bytes }

// -------------------------------------------------------------- scaffolding

const (
	overheadVSize = 11
	inputVSize    = 68
	outputVSize   = 31
)

func vsize(nIn, nOut int) int64 {
	return int64(overheadVSize + nIn*inputVSize + nOut*outputVSize)
}

func addr(name string) []byte { return hash160([]byte(name)) }

// tx builds a distinct transaction; the id only has to be unique and real.
func tx(tag byte, nIn, nOut int) *Transaction {
	t := &Transaction{}
	for i := 0; i < nIn; i++ {
		var id [32]byte
		id[0], id[1] = tag, byte(i)
		t.Inputs = append(t.Inputs, TxInput{Prev: Outpoint{id, 0},
			Signature: make([]byte, 65), PubKey: make([]byte, 33)})
	}
	for i := 0; i < nOut; i++ {
		t.Outputs = append(t.Outputs, TxOutput{Value: 1000, PubKeyHash: addr("dest")})
	}
	return t
}

func dump(m *Mempool, capacity int64) {
	fmt.Printf("  %-18s %-14s %-8s %-10s %-9s %s\n",
		"transaction", "txid", "vbytes", "fee (sat)", "sat/vB", "cumulative vB")
	var cum int64
	cut := false
	for _, e := range m.ByRate() {
		cum += e.VSize
		id := e.Tx.TxID()
		mark := ""
		if capacity > 0 && cum > capacity && !cut {
			mark = "  <- next block ends above here"
			cut = true
		}
		fmt.Printf("  %-18s %-14x %-8d %-10d %-9s %d%s\n",
			e.Name, id[:6], e.VSize, e.Fee, e.RateString(), cum, mark)
	}
}

func main() {
	m := NewMempool()

	type in struct {
		name      string
		tag       byte
		nIn, nOut int
		fee       int64
	}
	for _, a := range []in{
		{"exchange batch", 0xe0, 1, 60, 40_000},
		{"urgent payment", 0xa1, 2, 2, 30_000},
		{"merchant payout", 0xb2, 2, 20, 29_000},
		{"swap deposit", 0xc3, 3, 2, 15_000},
		{"someone's coffee", 0xd4, 1, 2, 10_000},
		{"a lazy default", 0xf5, 1, 2, 2_260},
	} {
		t := tx(a.tag, a.nIn, a.nOut)
		if err := m.Add(a.name, t, vsize(a.nIn, a.nOut), a.fee); err != nil {
			fmt.Println("  rejected:", err)
		}
	}

	fmt.Printf("=== the pool: %d transactions, %d vbytes ===\n", m.Len(), m.Bytes())
	dump(m, 0)

	fmt.Println("\n=== the same pool, with a 1000-vbyte block in mind ===")
	dump(m, 1000)
	fmt.Println("\n  That cut line is the whole game. Everything above it goes in")
	fmt.Println("  the next block; everything below waits, and competes against")
	fmt.Println("  whatever arrives in the meantime.")

	fmt.Println("\n=== the two indexes, doing their two jobs ===")
	dup := tx(0xa1, 2, 2)
	fmt.Printf("  do I have this already?  %v  (O(1) map lookup)\n", m.Has(dup.TxID()))
	fmt.Printf("  adding it again:         %v\n", m.Add("urgent payment", dup, vsize(2, 2), 30_000))
	fmt.Println("  Without the id index, a node would re-validate and re-relay")
	fmt.Println("  every transaction every time a peer announced it.")

	fmt.Println("\n=== a block arrives ===")
	mined := m.ByRate()[:3]
	for _, e := range mined {
		m.Remove(e.Tx.TxID())
	}
	fmt.Printf("  3 transactions mined; pool is now %d transactions, %d vbytes\n", m.Len(), m.Bytes())
	dump(m, 0)
	fmt.Println("\n  Note what removal needs: the txid. Which is why a node indexes")
	fmt.Println("  by txid even though it never SELECTS by txid.")

	fmt.Println("\n=== what this structure is missing ===")
	fmt.Println("  - admission checks: is it valid, are its inputs unspent, does it")
	fmt.Println("    conflict with something already here (example 9)")
	fmt.Println("  - policy on top of consensus: minimum rate, size limits (example 10)")
	fmt.Println("  - a cap and an eviction rule, or it grows until the node dies")
	fmt.Println("    (example 13)")
	fmt.Println("  - replacement, so a stuck transaction can be bumped (example 14)")
	fmt.Println("  - ancestor tracking, so a child can pay for its parent (example 16)")
	fmt.Println("  - a lock, because the network fills it while the miner reads it")
	fmt.Println("    (example 17)")
}
```

**Output:**

```
=== the pool: 6 transactions, 3474 vbytes ===
  transaction        txid           vbytes   fee (sat)  sat/vB    cumulative vB
  urgent payment     454ac9ba98ca   209      30000      143.5     209
  someone's coffee   c9985b758fe1   141      10000      70.9      350
  swap deposit       9efcc7a0ec6e   277      15000      54.1      627
  merchant payout    4f45fcaec1e6   767      29000      37.8      1394
  exchange batch     a83f9adabcf5   1939     40000      20.6      3333
  a lazy default     5f5cb2222637   141      2260       16.0      3474

=== the same pool, with a 1000-vbyte block in mind ===
  transaction        txid           vbytes   fee (sat)  sat/vB    cumulative vB
  urgent payment     454ac9ba98ca   209      30000      143.5     209
  someone's coffee   c9985b758fe1   141      10000      70.9      350
  swap deposit       9efcc7a0ec6e   277      15000      54.1      627
  merchant payout    4f45fcaec1e6   767      29000      37.8      1394  <- next block ends above here
  exchange batch     a83f9adabcf5   1939     40000      20.6      3333
  a lazy default     5f5cb2222637   141      2260       16.0      3474

  That cut line is the whole game. Everything above it goes in
  the next block; everything below waits, and competes against
  whatever arrives in the meantime.

=== the two indexes, doing their two jobs ===
  do I have this already?  true  (O(1) map lookup)
  adding it again:         already in the pool
  Without the id index, a node would re-validate and re-relay
  every transaction every time a peer announced it.

=== a block arrives ===
  3 transactions mined; pool is now 3 transactions, 2847 vbytes
  transaction        txid           vbytes   fee (sat)  sat/vB    cumulative vB
  merchant payout    4f45fcaec1e6   767      29000      37.8      767
  exchange batch     a83f9adabcf5   1939     40000      20.6      2706
  a lazy default     5f5cb2222637   141      2260       16.0      2847

  Note what removal needs: the txid. Which is why a node indexes
  by txid even though it never SELECTS by txid.

=== what this structure is missing ===
  - admission checks: is it valid, are its inputs unspent, does it
    conflict with something already here (example 9)
  - policy on top of consensus: minimum rate, size limits (example 10)
  - a cap and an eviction rule, or it grows until the node dies
    (example 13)
  - replacement, so a stuck transaction can be bumped (example 14)
  - ancestor tracking, so a child can pay for its parent (example 16)
  - a lock, because the network fills it while the miner reads it
    (example 17)
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
