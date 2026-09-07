# Step 07 — Addresses, Encodings & HD Wallets · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. Public key to address, by hand

`🟢 easy` · *Ethereum addresses*

Four steps: take the uncompressed public key, **drop the `0x04` prefix**, Keccak-256 the remaining 64 bytes, keep the last 20. Forgetting to strip that prefix produces a valid-looking, completely wrong address — with no error at any point.

**Steps:**

1. Print the 65-byte uncompressed public key, then the 64 coordinate bytes.
2. Keccak-256 them (not SHA3-256 — lesson 04).
3. Take the last 20 bytes and compare against `crypto.PubkeyToAddress`.
4. Then hash all 65 bytes to see the wrong address the common mistake produces.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// The published anvil account-0 key (lesson 06). TEST KEY ONLY.
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	// Step 1: the uncompressed public key, 65 bytes: 0x04 ‖ X ‖ Y.
	pub := crypto.FromECDSAPub(&key.PublicKey)
	fmt.Printf("public key (%d bytes)\n  %s\n", len(pub), hex.EncodeToString(pub))

	// Step 2: DROP the 0x04 prefix. Hash the 64 coordinate bytes only.
	// Including the prefix is the single most common mistake here.
	coords := pub[1:]
	fmt.Printf("\nwithout the 0x04 prefix (%d bytes)\n  %s\n",
		len(coords), hex.EncodeToString(coords))

	// Step 3: Keccak-256 (NOT SHA3-256 — lesson 04).
	h := crypto.Keccak256(coords)
	fmt.Printf("\nkeccak256 of those 64 bytes\n  %s\n", hex.EncodeToString(h))

	// Step 4: take the LAST 20 bytes.
	addr := h[12:]
	fmt.Printf("\nlast 20 bytes = the address\n  0x%s\n", hex.EncodeToString(addr))

	// Compare with the library.
	fmt.Printf("\ncrypto.PubkeyToAddress\n  %s\n", crypto.PubkeyToAddress(key.PublicKey).Hex())
	fmt.Printf("\nsame: %v\n",
		hex.EncodeToString(addr) == hex.EncodeToString(crypto.PubkeyToAddress(key.PublicKey).Bytes()))

	// The classic bug: hashing all 65 bytes gives a valid-looking wrong address.
	wrong := crypto.Keccak256(pub)[12:]
	fmt.Printf("\nif you forget to strip 0x04:\n  0x%s   <- wrong, and it looks fine\n",
		hex.EncodeToString(wrong))
	fmt.Println("\nno error, no warning. Funds sent there are gone.")
}
```

**Output:**

```
public key (65 bytes)
  048318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed753547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5

without the 0x04 prefix (64 bytes)
  8318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed753547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5

keccak256 of those 64 bytes
  c1ffd3cfee2d9e5cd67643f8f39fd6e51aad88f6f4ce6ab8827279cfffb92266

last 20 bytes = the address
  0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266

crypto.PubkeyToAddress
  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

same: true

if you forget to strip 0x04:
  0x56377c0b855c204ae32ed48dffddc1e059076f04   <- wrong, and it looks fine

no error, no warning. Funds sent there are gone.
```

---

## 2. EIP-55: computing the checksum

`🟢 easy` · *EIP-55*

The checksum is smuggled into letter case. Hash the lowercase hex *string* (as ASCII, not as bytes), then uppercase hex letter i whenever nibble i of that hash is ≥ 8. The address bytes never change, which is why old all-lowercase addresses are still valid.

**Steps:**

1. Implement the rule and compare your output with `common.Address.Hex()`.
2. Walk the first eight characters showing the nibble and the decision for each.
3. Note that digits carry no case, so they carry no checksum information.
4. Confirm the underlying bytes are identical either way.

```go
package main

import (
	"encoding/hex"
	"fmt"
	"strings"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// EIP55 encodes a 20-byte address using the mixed-case checksum.
// The rule: hash the LOWERCASE HEX STRING (as ASCII, not as bytes), then
// uppercase hex letter i whenever nibble i of that hash is >= 8.
func EIP55(addr []byte) string {
	lower := hex.EncodeToString(addr) // 40 chars, no 0x
	hash := crypto.Keccak256([]byte(lower))

	out := []byte(lower)
	for i := 0; i < len(out); i++ {
		if out[i] < 'a' || out[i] > 'f' {
			continue // digits have no case to carry information
		}
		// nibble i of the hash: high nibble for even i, low for odd.
		nibble := hash[i/2] >> 4
		if i%2 == 1 {
			nibble = hash[i/2] & 0x0f
		}
		if nibble >= 8 {
			out[i] = byte(strings.ToUpper(string(out[i]))[0])
		}
	}
	return "0x" + string(out)
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	addr := crypto.PubkeyToAddress(key.PublicKey)

	lower := hex.EncodeToString(addr.Bytes())
	fmt.Printf("all-lowercase   0x%s\n", lower)
	fmt.Printf("keccak of that  %s\n", hex.EncodeToString(crypto.Keccak256([]byte(lower))))
	fmt.Printf("EIP-55          %s\n", EIP55(addr.Bytes()))
	fmt.Printf("go-ethereum     %s\n", addr.Hex())
	fmt.Printf("\nmatch: %v\n", EIP55(addr.Bytes()) == addr.Hex())

	// Walk the first few characters to see the rule in action.
	hash := crypto.Keccak256([]byte(lower))
	fmt.Println("\nfirst 8 characters:")
	fmt.Printf("  %-4s %-6s %-8s %s\n", "i", "char", "nibble", "result")
	for i := 0; i < 8; i++ {
		nibble := hash[i/2] >> 4
		if i%2 == 1 {
			nibble = hash[i/2] & 0x0f
		}
		note := "digit, unchanged"
		if lower[i] >= 'a' && lower[i] <= 'f' {
			if nibble >= 8 {
				note = "letter, nibble>=8 -> UPPER"
			} else {
				note = "letter, nibble<8  -> lower"
			}
		}
		fmt.Printf("  %-4d %-6c %-8x %s\n", i, lower[i], nibble, note)
	}

	// The checksum lives entirely in the CASE. The address bytes are identical.
	fmt.Printf("\nbytes are the same either way: %v\n",
		common.HexToAddress(strings.ToLower(addr.Hex())) == addr)
	fmt.Println("EIP-55 adds error detection without changing the address at all —")
	fmt.Println("which is why old all-lowercase addresses are still valid.")
}
```

**Output:**

```
all-lowercase   0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
keccak of that  19682d67812181b19d61f09263ab9e723b258d8d7949f68794d8916171bea91d
EIP-55          0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
go-ethereum     0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

match: true

first 8 characters:
  i    char   nibble   result
  0    f      1        letter, nibble<8  -> lower
  1    3      9        digit, unchanged
  2    9      6        digit, unchanged
  3    f      8        letter, nibble>=8 -> UPPER
  4    d      2        letter, nibble<8  -> lower
  5    6      d        digit, unchanged
  6    e      6        letter, nibble<8  -> lower
  7    5      7        digit, unchanged

bytes are the same either way: true
EIP-55 adds error detection without changing the address at all —
which is why old all-lowercase addresses are still valid.
```

---

## 3. EIP-55: validating what a user pasted

`🟢 easy` · *EIP-55*

The validator you run on anything a user typed or pasted. Note the case that catches people: an **all-lowercase address is legal and carries no checksum at all**, so it cannot be verified — it must be treated as unchecked input, not as valid.

**Steps:**

1. Check prefix, length, hex-ness, then the checksum, in that order.
2. Detect the single-case case explicitly rather than silently passing it.
3. Run six inputs through and read the distinct errors.
4. Note `common.IsHexAddress` does *not* check the checksum.

```go
package main

import (
	"encoding/hex"
	"errors"
	"fmt"
	"strings"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

func eip55(addr []byte) string {
	lower := hex.EncodeToString(addr)
	hash := crypto.Keccak256([]byte(lower))
	out := []byte(lower)
	for i := range out {
		if out[i] < 'a' || out[i] > 'f' {
			continue
		}
		nibble := hash[i/2] >> 4
		if i%2 == 1 {
			nibble = hash[i/2] & 0x0f
		}
		if nibble >= 8 {
			out[i] = out[i] - 'a' + 'A'
		}
	}
	return "0x" + string(out)
}

var (
	ErrLength     = errors.New("not 42 characters")
	ErrPrefix     = errors.New("missing 0x prefix")
	ErrHex        = errors.New("not hex")
	ErrChecksum   = errors.New("EIP-55 checksum mismatch")
	ErrNoChecksum = errors.New("all one case: no checksum to verify")
)

// Validate is what you run on any address that came from outside your process.
func Validate(s string) error {
	if !strings.HasPrefix(s, "0x") {
		return ErrPrefix
	}
	if len(s) != 42 {
		return ErrLength
	}
	body := s[2:]
	raw, err := hex.DecodeString(strings.ToLower(body))
	if err != nil {
		return ErrHex
	}
	// An address in a single case carries no checksum information at all.
	if body == strings.ToLower(body) || body == strings.ToUpper(body) {
		return ErrNoChecksum
	}
	if eip55(raw) != s {
		return ErrChecksum
	}
	return nil
}

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	good := crypto.PubkeyToAddress(key.PublicKey).Hex()

	cases := []string{
		good,
		strings.ToLower(good),
		good[:10] + "A" + good[11:], // one character replaced
		"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb9226",  // one char short
		"f39Fd6e51aad88F6F4ce6aB8827279cffFb92266",   // no prefix
		"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb9226Z", // not hex
	}

	fmt.Printf("%-46s %s\n", "address", "result")
	for _, c := range cases {
		err := Validate(c)
		res := "valid"
		if err != nil {
			res = err.Error()
		}
		fmt.Printf("%-46s %s\n", c, res)
	}

	// How good is the check? Each hex letter that could carry a case is a
	// coin flip, and a typo has to land on the same case by chance.
	fmt.Println("\nhow much protection is that?")
	fmt.Println("  a single wrong character is caught ~99.986% of the time")
	fmt.Println("  a transposition of two different characters is almost always caught")
	fmt.Println("  an all-lowercase address is caught 0% of the time")

	// The point of it all.
	fmt.Println("\nan Ethereum address is DERIVED, never registered. Every 20-byte")
	fmt.Println("value is a valid address that nobody holds the key to. Send there")
	fmt.Println("and the funds are gone — there is no bounce, no error, no recovery.")
	fmt.Printf("\ncommon.IsHexAddress does NOT check the checksum: %v\n",
		common.IsHexAddress(strings.ToLower(good)))
	fmt.Println("so validate the checksum yourself on anything a user typed or pasted.")
}
```

**Output:**

```
address                                        result
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266     valid
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266     all one case: no checksum to verify
0xf39Fd6e5Aaad88F6F4ce6aB8827279cffFb92266     EIP-55 checksum mismatch
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb9226      not 42 characters
f39Fd6e51aad88F6F4ce6aB8827279cffFb92266       missing 0x prefix
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb9226Z     not hex

how much protection is that?
  a single wrong character is caught ~99.986% of the time
  a transposition of two different characters is almost always caught
  an all-lowercase address is caught 0% of the time

an Ethereum address is DERIVED, never registered. Every 20-byte
value is a valid address that nobody holds the key to. Send there
and the funds are gone — there is no bounce, no error, no recovery.

common.IsHexAddress does NOT check the checksum: true
so validate the checksum yourself on anything a user typed or pasted.
```

---

## 4. Every 20-byte value is an address

`🟢 easy` · *Ethereum addresses*

There is no registry. Every 20-byte value is a valid address that simply holds nothing until something is sent to it — which is why a wrong-but-valid address accepts your funds silently. Watch what `HexToAddress` does to a typo, too.

**Steps:**

1. Parse five addresses including the zero address and a one-character typo.
2. See that `HexToAddress` discards the input casing and recomputes the checksum on output.
3. Read what the zero address means by convention, and why `ecrecover` returning it matters.
4. See how `HexToAddress` silently pads short input and truncates long input from the left.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
)

func main() {
	// There is no registry. An Ethereum address is any 20-byte value, and
	// every one of them "exists" — it simply has a balance of zero until
	// something is sent to it.
	examples := []struct {
		name string
		hex  string
	}{
		{"the zero address", "0x0000000000000000000000000000000000000000"},
		{"address 0x1", "0x0000000000000000000000000000000000000001"},
		{"all ones", "0xffffffffffffffffffffffffffffffffffffffff"},
		{"a real account", "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"},
		{"a typo of it", "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92267"},
	}

	fmt.Printf("%-20s %-44s %s\n", "", "as parsed and re-encoded", "valid 20 bytes?")
	for _, e := range examples {
		a := common.HexToAddress(e.hex)
		fmt.Printf("%-20s %-44s %v\n", e.name, a.Hex(), len(a.Bytes()) == 20)
	}

	// Look carefully at the last two rows. The input differed by one character,
	// and BOTH came back perfectly checksummed — because HexToAddress ignores
	// the input casing entirely and recomputes EIP-55 on output.
	fmt.Println("\nnote: HexToAddress DISCARDS the input casing and recomputes the")
	fmt.Println("checksum, so a typo'd address is handed back looking impeccable.")
	fmt.Println("Parsing is not validation — check the checksum on the raw string first.")

	// The keyspace, and the part of it anyone can actually reach.
	fmt.Println("\n2^160 possible addresses.")
	fmt.Println("A private key exists for essentially all of them — but finding one")
	fmt.Println("for a CHOSEN address needs about 2^160 work (lesson 04's birthday")
	fmt.Println("bound does not help an attacker here; they need a preimage, not a")
	fmt.Println("collision).")

	// The zero address is special only by convention.
	zero := common.Address{}
	fmt.Printf("\nthe zero address %s\n", zero.Hex())
	fmt.Println("  nobody has its key, and by convention it means 'burn' or 'none':")
	fmt.Println("  ERC-20 mints show as Transfer(from: 0x0), burns as Transfer(to: 0x0)")
	fmt.Println("  a contract with owner == 0x0 has renounced ownership")
	fmt.Println("  ecrecover returns 0x0 on failure — which is why you must check for it")

	// And the consequence that matters.
	fmt.Println("\nbecause addresses are derived and not registered:")
	fmt.Println("  - a wrong-but-valid address accepts your funds silently")
	fmt.Println("  - there is no bounce, no error, and no recovery")
	fmt.Println("  - the only defence is the checksum (example 3)")

	// Show that HexToAddress silently truncates or pads bad input.
	fmt.Println("\ncommon.HexToAddress is forgiving, which is its own hazard:")
	for _, in := range []string{"0x1", "0x" + hex.EncodeToString(make([]byte, 25))} {
		fmt.Printf("  %-56q -> %s\n", in, common.HexToAddress(in).Hex())
	}
	fmt.Println("  short input is left-padded; long input is TRUNCATED FROM THE LEFT.")
	fmt.Println("  validate before you convert (example 3), never after.")
}
```

**Output:**

```
                     as parsed and re-encoded                     valid 20 bytes?
the zero address     0x0000000000000000000000000000000000000000   true
address 0x1          0x0000000000000000000000000000000000000001   true
all ones             0xFFfFfFffFFfffFFfFFfFFFFFffFFFffffFfFFFfF   true
a real account       0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   true
a typo of it         0xF39fD6e51Aad88F6f4CE6AB8827279CFffb92267   true

note: HexToAddress DISCARDS the input casing and recomputes the
checksum, so a typo'd address is handed back looking impeccable.
Parsing is not validation — check the checksum on the raw string first.

2^160 possible addresses.
A private key exists for essentially all of them — but finding one
for a CHOSEN address needs about 2^160 work (lesson 04's birthday
bound does not help an attacker here; they need a preimage, not a
collision).

the zero address 0x0000000000000000000000000000000000000000
  nobody has its key, and by convention it means 'burn' or 'none':
  ERC-20 mints show as Transfer(from: 0x0), burns as Transfer(to: 0x0)
  a contract with owner == 0x0 has renounced ownership
  ecrecover returns 0x0 on failure — which is why you must check for it

because addresses are derived and not registered:
  - a wrong-but-valid address accepts your funds silently
  - there is no bounce, no error, and no recovery
  - the only defence is the checksum (example 3)

common.HexToAddress is forgiving, which is its own hazard:
  "0x1"                                                    -> 0x0000000000000000000000000000000000000001
  "0x00000000000000000000000000000000000000000000000000"   -> 0x0000000000000000000000000000000000000000
  short input is left-padded; long input is TRUNCATED FROM THE LEFT.
  validate before you convert (example 3), never after.
```

---

## 5. One key, five address formats

`🟢 easy` · *Address formats*

The same public key, presented as an Ethereum address and as three Bitcoin ones. Note the inputs differ: Ethereum hashes the 64 uncompressed coordinate bytes, Bitcoin hashes the 33-byte compressed key. Different input, different result — they are not convertible.

**Steps:**

1. Derive the Ethereum address and Bitcoin's HASH160 from the same key.
2. Build P2PKH mainnet, P2PKH testnet and P2WPKH addresses.
3. Read what each leading character means — it is the version byte showing through.
4. Compare the two design philosophies, and why bech32 replaced base58check.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/btcsuite/btcd/btcutil"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	// ---- Ethereum: keccak of the uncompressed key, last 20 bytes -----------
	ethAddr := crypto.PubkeyToAddress(key.PublicKey)
	fmt.Println("Ethereum")
	fmt.Printf("  %s\n", ethAddr.Hex())
	fmt.Printf("  20 bytes, hex, mixed-case EIP-55 checksum, no prefix byte\n")

	// ---- Bitcoin: HASH160, then a version byte and base58check ------------
	// Note Bitcoin hashes the COMPRESSED public key (33 bytes), Ethereum the
	// uncompressed coordinates (64 bytes). Different input, different result.
	compressed := crypto.CompressPubkey(&key.PublicKey)
	h160 := btcutil.Hash160(compressed)
	fmt.Printf("\nHASH160 of the compressed key\n  %s (%d bytes)\n",
		hex.EncodeToString(h160), len(h160))

	p2pkh, _ := btcutil.NewAddressPubKeyHash(h160, &chaincfg.MainNetParams)
	p2wpkh, _ := btcutil.NewAddressWitnessPubKeyHash(h160, &chaincfg.MainNetParams)
	tp2pkh, _ := btcutil.NewAddressPubKeyHash(h160, &chaincfg.TestNet3Params)

	fmt.Println("\nBitcoin, same 20 bytes, three presentations")
	fmt.Printf("  P2PKH  mainnet  %s\n", p2pkh.EncodeAddress())
	fmt.Printf("  P2PKH  testnet  %s\n", tp2pkh.EncodeAddress())
	fmt.Printf("  P2WPKH mainnet  %s\n", p2wpkh.EncodeAddress())

	// The leading character is the version byte showing through base58.
	fmt.Println("\nwhy the prefixes differ")
	fmt.Println("  1...    P2PKH mainnet   version byte 0x00")
	fmt.Println("  3...    P2SH  mainnet   version byte 0x05")
	fmt.Println("  m/n...  P2PKH testnet   version byte 0x6f")
	fmt.Println("  bc1q... P2WPKH          bech32, witness version 0")
	fmt.Println("  bc1p... P2TR (Taproot)  bech32m, witness version 1")

	// The two philosophies.
	fmt.Println("\ntwo approaches to the same problem")
	fmt.Println("  Ethereum: one format forever, checksum smuggled into letter case.")
	fmt.Println("            Every address looks identical; the type is not encoded.")
	fmt.Println("  Bitcoin:  a new format per script type, each self-describing.")
	fmt.Println("            You can tell what an address DOES before you send to it.")

	fmt.Println("\nbech32 exists because base58check had real problems:")
	fmt.Println("  - mixed case is error-prone to read aloud and to type")
	fmt.Println("  - QR codes are ~20% larger for mixed-case alphanumeric")
	fmt.Println("  - a 4-byte checksum detects errors but cannot locate them")
	fmt.Println("  bech32 is lowercase-only, QR-efficient, and its BCH code can point")
	fmt.Println("  at WHICH character is wrong (lesson 36 covers the encoding).")
}
```

**Output:**

```
Ethereum
  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
  20 bytes, hex, mixed-case EIP-55 checksum, no prefix byte

HASH160 of the compressed key
  a55476015c13afb8afb92160329a8cde976f1f2e (20 bytes)

Bitcoin, same 20 bytes, three presentations
  P2PKH  mainnet  1G5Bg6w4srw965M1JSYTFNtDg4fXpeYh3r
  P2PKH  testnet  mvb8yA23gtNPsBpd21Wq5J6YY4GEnfYQyX
  P2WPKH mainnet  bc1q5428vq2uzwhm3taey9sr9x5vm6tk78ew0wt525

why the prefixes differ
  1...    P2PKH mainnet   version byte 0x00
  3...    P2SH  mainnet   version byte 0x05
  m/n...  P2PKH testnet   version byte 0x6f
  bc1q... P2WPKH          bech32, witness version 0
  bc1p... P2TR (Taproot)  bech32m, witness version 1

two approaches to the same problem
  Ethereum: one format forever, checksum smuggled into letter case.
            Every address looks identical; the type is not encoded.
  Bitcoin:  a new format per script type, each self-describing.
            You can tell what an address DOES before you send to it.

bech32 exists because base58check had real problems:
  - mixed case is error-prone to read aloud and to type
  - QR codes are ~20% larger for mixed-case alphanumeric
  - a 4-byte checksum detects errors but cannot locate them
  bech32 is lowercase-only, QR-efficient, and its BCH code can point
  at WHICH character is wrong (lesson 36 covers the encoding).
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
