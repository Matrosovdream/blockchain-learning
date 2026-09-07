# Step 07 — Addresses, Encodings & HD Wallets · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. The xpub leak

`🔴 hard` · *BIP-32*

The reason hardened derivation exists. An xpub contains the parent public key **and** the chain code, so anyone holding it can recompute the HMAC for any child — and if they also have one non-hardened child *private* key, they simply subtract to get the parent.

**Steps:**

1. Publish an xpub and leak exactly one child private key, as a sweep script might.
2. Pull the chain code and public key straight out of the base58-decoded xpub.
3. Recompute `IL = HMAC-SHA512(chainCode, pubkey ‖ index)`.
4. Recover the parent with `parentPriv = (childPriv − IL) mod n`, and confirm it matches.
5. Derive every sibling key to see the blast radius — then read why BIP-44's hardening bounds it.

```go
package main

import (
	"crypto/hmac"
	"crypto/sha512"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/btcsuite/btcd/btcec/v2"
	"github.com/btcsuite/btcd/btcutil/base58"
	"github.com/btcsuite/btcd/btcutil/hdkeychain"
	"github.com/btcsuite/btcd/chaincfg"
	"github.com/ethereum/go-ethereum/crypto"
	bip39 "github.com/tyler-smith/go-bip39"
)

const hardened = hdkeychain.HardenedKeyStart

var n = btcec.S256().N

func main() {
	const mnemonic = "test test test test test test test test test test test junk"
	seed := bip39.NewSeed(mnemonic, "")
	master, _ := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)

	// The account node: m/44'/60'/0'/0. Its xpub is what a watch-only server
	// holds (example 10), and it is often published without much thought.
	parent := master
	for _, i := range []uint32{hardened + 44, hardened + 60, hardened + 0, 0} {
		parent, _ = parent.Derive(i)
	}
	parentPriv, _ := parent.ECPrivKey()
	xpub, _ := parent.Neuter()

	fmt.Println("what the attacker has")
	fmt.Printf("  1. the xpub  %s\n", xpub.String())

	// And ONE non-hardened child private key. This leaks in ordinary ways:
	// a sweep script, a test fixture, an exported key for one address.
	const childIndex = 3
	child, _ := parent.Derive(childIndex)
	childPriv, _ := child.ECPrivKey()
	fmt.Printf("  2. ONE child private key, index %d\n", childIndex)
	fmt.Printf("     %s\n", hex.EncodeToString(childPriv.Serialize()))
	fmt.Printf("     (address %s)\n",
		crypto.PubkeyToAddress(childPriv.ToECDSA().PublicKey).Hex())

	// ---- the attack --------------------------------------------------------
	// Non-hardened derivation is:
	//     I  = HMAC-SHA512(chainCode, compressedParentPubKey ‖ index)
	//     childPriv = (IL + parentPriv) mod n
	// The xpub contains BOTH the parent public key and the chain code, so the
	// attacker can recompute IL — and then simply subtract.
	//
	//     parentPriv = (childPriv - IL) mod n

	// Pull the chain code and public key straight out of the serialized xpub.
	decoded := base58.Decode(xpub.String())
	chainCode := decoded[13:45]
	pubKey := decoded[45:78]
	fmt.Printf("\nfrom the xpub bytes:\n  chain code %s\n  pubkey     %s\n",
		hex.EncodeToString(chainCode), hex.EncodeToString(pubKey))

	data := make([]byte, 0, 37)
	data = append(data, pubKey...)
	data = binary.BigEndian.AppendUint32(data, childIndex)

	mac := hmac.New(sha512.New, chainCode)
	mac.Write(data)
	I := mac.Sum(nil)
	IL := new(big.Int).SetBytes(I[:32])

	fmt.Printf("\nI  = HMAC-SHA512(chainCode, pubkey ‖ index)\n")
	fmt.Printf("IL = %s\n", hex.EncodeToString(I[:32]))

	recovered := new(big.Int).Sub(new(big.Int).SetBytes(childPriv.Serialize()), IL)
	recovered.Mod(recovered, n)

	fmt.Printf("\nparentPriv = (childPriv - IL) mod n\n           = %s\n",
		hex.EncodeToString(recovered.Bytes()))
	fmt.Printf("actual       %s\n", hex.EncodeToString(parentPriv.Serialize()))
	fmt.Printf("\nRECOVERED: %v\n",
		hex.EncodeToString(recovered.Bytes()) == hex.EncodeToString(parentPriv.Serialize()))

	// With the parent key, every sibling falls immediately.
	fmt.Println("\nand now every address under that node is compromised:")
	recoveredKey, _ := crypto.ToECDSA(recovered.Bytes())
	_ = recoveredKey
	for i := 0; i < 4; i++ {
		c, _ := parent.Derive(uint32(i))
		p, _ := c.ECPrivKey()
		fmt.Printf("  index %d  %s  key %s...\n", i,
			crypto.PubkeyToAddress(p.ToECDSA().PublicKey).Hex(),
			hex.EncodeToString(p.Serialize())[:16])
	}

	fmt.Println("\nthe defence is structural, not procedural:")
	fmt.Println("  BIP-44 hardens purpose', coin_type' and account', so this leak")
	fmt.Println("  stops at the account node. The attacker gets one account's")
	fmt.Println("  addresses — not the master key, and not your other accounts.")
	fmt.Println("\nthe rule: treat an xpub as SECRET, and never export a single child")
	fmt.Println("private key from a node whose xpub you have shared.")
}
```

**Output:**

```
what the attacker has
  1. the xpub  xpub6DyUKdwoLWmUJ4Tn9Bbsdtx7B5Ws18mEN19e5HT52ikE53FiUheSQXrZUNPovqfyKmw4579A1Mm3GXXKM39N64uooBfJ4tNAzFsEbodRTx4
  2. ONE child private key, index 3
     7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6
     (address 0x90F79bf6EB2c4f870365E785982E1f101E93b906)

from the xpub bytes:
  chain code 236a4a5d77e29cc09182a8a6c9d288cb04e638fcd0743bbc068c16665d1e4d2c
  pubkey     03b93015e4ada498248da1ed7041e749ccb07a4489e2e33b38405f5c93d44703e9

I  = HMAC-SHA512(chainCode, pubkey ‖ index)
IL = 9f6156c38eb686b348700f9629e38de4db11a96d7f5ec42fd7fed6b7783abd42

parentPriv = (childPriv - IL) mod n
           = dd23ca549a97cb330b011aebb674730df8b14acaee42d211ab45692699ab8ba5
actual       dd23ca549a97cb330b011aebb674730df8b14acaee42d211ab45692699ab8ba5

RECOVERED: true

and now every address under that node is compromised:
  index 0  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  key ac0974bec39a17e3...
  index 1  0x70997970C51812dc3A010C7d01b50e0d17dc79C8  key 59c6995e998f97a5...
  index 2  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC  key 5de4111afa1a4b94...
  index 3  0x90F79bf6EB2c4f870365E785982E1f101E93b906  key 7c852118294e51e6...

the defence is structural, not procedural:
  BIP-44 hardens purpose', coin_type' and account', so this leak
  stops at the account node. The attacker gets one account's
  addresses — not the master key, and not your other accounts.

the rule: treat an xpub as SECRET, and never export a single child
private key from a node whose xpub you have shared.
```

---

## 15. CREATE: the address of your next deployment

`🔴 hard` · *Contract addresses*

A contract's address is `keccak256(rlp([sender, nonce]))[12:]` — derived from who deployed it and their nonce, not chosen. That makes your next deployment address fully predictable, and makes it shift if any other transaction sneaks in first.

**Steps:**

1. Encode `[sender, nonce]` with RLP and hash it for eight different nonces.
2. Check every result against `crypto.CreateAddress`.
3. Watch the RLP length change at 128 and 256 (lesson 17).
4. Note nonce 0 gives the address anvil deploys to first.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/rlp"
)

func main() {
	// A contract's address is not chosen. It is derived from WHO deployed it
	// and their nonce at the time:
	//
	//     address = keccak256(rlp([sender, nonce]))[12:]
	deployer := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266")

	fmt.Printf("deployer %s\n\n", deployer.Hex())
	fmt.Printf("%-7s %-70s %s\n", "nonce", "rlp([sender, nonce])", "contract address")
	for _, nonce := range []uint64{0, 1, 2, 3, 127, 128, 255, 256} {
		encoded, err := rlp.EncodeToBytes([]any{deployer, nonce})
		if err != nil {
			fmt.Println("rlp:", err)
			return
		}
		addr := common.BytesToAddress(crypto.Keccak256(encoded)[12:])

		// go-ethereum has a helper; confirm we agree.
		if addr != crypto.CreateAddress(deployer, nonce) {
			fmt.Println("MISMATCH at nonce", nonce)
			return
		}
		fmt.Printf("%-7d %-70s %s\n", nonce, hex.EncodeToString(encoded), addr.Hex())
	}

	fmt.Println("\nnote how the RLP length changes at 128 and 256 — that is lesson 17.")

	// Consequences.
	fmt.Println("\nwhat this means in practice")
	fmt.Println("  the address of your NEXT deployment is fully predictable")
	fmt.Println("  deploy the same contract from a fresh account and you get the")
	fmt.Println("  same address on every chain — which is how multi-chain deployments")
	fmt.Println("  keep one address everywhere")

	// The trap.
	fmt.Println("\nthe trap: it depends on the nonce, so ANY other transaction from")
	fmt.Println("that account between now and deployment shifts the address.")
	fmt.Println("Deploy from a dedicated account, or use CREATE2 (example 16).")

	// Contract accounts increment their own nonce when they deploy.
	fmt.Println("\nalso note: a CONTRACT deploying a contract uses the same rule,")
	fmt.Println("with its own nonce — which starts at 1, not 0 (EIP-161).")
}
```

**Output:**

```
deployer 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

nonce   rlp([sender, nonce])                                                   contract address
0       d694f39fd6e51aad88f6f4ce6ab8827279cfffb9226680                         0x5FbDB2315678afecb367f032d93F642f64180aa3
1       d694f39fd6e51aad88f6f4ce6ab8827279cfffb9226601                         0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
2       d694f39fd6e51aad88f6f4ce6ab8827279cfffb9226602                         0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
3       d694f39fd6e51aad88f6f4ce6ab8827279cfffb9226603                         0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
127     d694f39fd6e51aad88f6f4ce6ab8827279cfffb922667f                         0x5fc748f1FEb28d7b76fa1c6B07D8ba2d5535177c
128     d794f39fd6e51aad88f6f4ce6ab8827279cfffb922668180                       0xB82008565FdC7e44609fA118A4a681E92581e680
255     d794f39fd6e51aad88f6f4ce6ab8827279cfffb9226681ff                       0x01E21d7B8c39dc4C764c19b308Bd8b14B1ba139E
256     d894f39fd6e51aad88f6f4ce6ab8827279cfffb92266820100                     0x3C1Cb427D20F15563aDa8C249E71db76d7183B6c

note how the RLP length changes at 128 and 256 — that is lesson 17.

what this means in practice
  the address of your NEXT deployment is fully predictable
  deploy the same contract from a fresh account and you get the
  same address on every chain — which is how multi-chain deployments
  keep one address everywhere

the trap: it depends on the nonce, so ANY other transaction from
that account between now and deployment shifts the address.
Deploy from a dedicated account, or use CREATE2 (example 16).

also note: a CONTRACT deploying a contract uses the same rule,
with its own nonce — which starts at 1, not 0 (EIP-161).
```

---

## 16. CREATE2: an address before the contract

`🔴 hard` · *Contract addresses*

CREATE2 replaces the nonce with a salt and a hash of the init code, so the address is computable before anything is deployed. That enables counterfactual accounts — and creates the trap that an address with no code today can have code tomorrow.

**Steps:**

1. Compute `keccak256(0xff ‖ sender ‖ salt ‖ keccak256(initCode))[12:]` by hand.
2. Check against `crypto.CreateAddress2` for four salts.
3. Read what counterfactual deployment enables, including smart accounts (lesson 47).
4. Change the deployer, then one byte of init code, and watch the address move.
5. Understand why `code.length == 0` does not mean 'this is an EOA'.

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// CREATE2 removes the nonce from the equation:
	//
	//     address = keccak256(0xff ‖ sender ‖ salt ‖ keccak256(initCode))[12:]
	//
	// Every input is chosen by the deployer, so the address is known BEFORE
	// the contract exists — "counterfactual" deployment.
	deployer := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266")
	initCode := common.FromHex("0x6080604052348015600f57600080fd5b50603f80601d6000396000f3fe6080604052600080fdfea2646970667358")
	initHash := crypto.Keccak256(initCode)

	fmt.Printf("deployer      %s\n", deployer.Hex())
	fmt.Printf("init code     %s... (%d bytes)\n", hex.EncodeToString(initCode)[:32], len(initCode))
	fmt.Printf("keccak(init)  %s\n", hex.EncodeToString(initHash))

	compute := func(salt [32]byte) common.Address {
		buf := make([]byte, 0, 85)
		buf = append(buf, 0xff)
		buf = append(buf, deployer.Bytes()...)
		buf = append(buf, salt[:]...)
		buf = append(buf, initHash...)
		return common.BytesToAddress(crypto.Keccak256(buf)[12:])
	}

	fmt.Printf("\n%-10s %s\n", "salt", "predicted address")
	for i := 0; i < 4; i++ {
		var salt [32]byte
		salt[31] = byte(i)
		addr := compute(salt)

		// go-ethereum's helper must agree.
		var h common.Hash
		copy(h[:], initHash)
		if addr != crypto.CreateAddress2(deployer, salt, initHash) {
			fmt.Println("MISMATCH at salt", i)
			return
		}
		fmt.Printf("%-10d %s\n", i, addr.Hex())
	}

	fmt.Println("\nnothing has been deployed. These addresses are computed from")
	fmt.Println("public inputs alone, by anyone, at any time.")

	// What counterfactual deployment enables.
	fmt.Println("\nwhat that enables")
	fmt.Println("  give a user a deposit address before their contract exists")
	fmt.Println("  deploy only when it is first needed, and let the user pay")
	fmt.Println("  smart-account wallets (lesson 47) — the address is known from")
	fmt.Println("  day one and the contract appears with the first transaction")
	fmt.Println("  the same address on every chain, independent of nonce history")

	// And the security consequence.
	fmt.Println("\nTHE TRAP: an address with no code today can have code tomorrow.")
	fmt.Println("  a check like `addr.code.length == 0` does NOT mean 'this is an EOA'")
	fmt.Println("  it means 'no code RIGHT NOW' — and CREATE2 lets someone deploy")
	fmt.Println("  there later, at an address you already approved or funded.")
	fmt.Println("  Worse, a self-destructed CREATE2 contract could historically be")
	fmt.Println("  redeployed with DIFFERENT code at the same address; EIP-6780")
	fmt.Println("  restricted SELFDESTRUCT, but the design lesson stands.")

	fmt.Println("\nchanging ANY input changes the address:")
	var salt [32]byte
	salt[31] = 1
	other := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")
	fmt.Printf("  different deployer  %s\n",
		crypto.CreateAddress2(other, salt, initHash).Hex())
	fmt.Printf("  one byte of initcode changed:\n")
	altered := append([]byte(nil), initCode...)
	altered[0] ^= 0x01
	fmt.Printf("  %s\n", crypto.CreateAddress2(deployer, salt, crypto.Keccak256(altered)).Hex())
	fmt.Println("\n  so the address commits to the exact bytecode — which is why you")
	fmt.Println("  can trust a counterfactual address to hold the code you expect.")
}
```

**Output:**

```
deployer      0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
init code     6080604052348015600f57600080fd5b... (46 bytes)
keccak(init)  a7ceaccafaa19454125dea2848f36826b93be38da28022caea98b000ca2f3193

salt       predicted address
0          0x8fA138a0580117C497D3dA2CC2B998f7aa849a09
1          0x1378e00a429e379A6a4987a619E18016922Bf105
2          0xB2569618893Ca2889702514DF13177493D6179D8
3          0xAF4921753180e22506f33a517f74d980D30e5416

nothing has been deployed. These addresses are computed from
public inputs alone, by anyone, at any time.

what that enables
  give a user a deposit address before their contract exists
  deploy only when it is first needed, and let the user pay
  smart-account wallets (lesson 47) — the address is known from
  day one and the contract appears with the first transaction
  the same address on every chain, independent of nonce history

THE TRAP: an address with no code today can have code tomorrow.
  a check like `addr.code.length == 0` does NOT mean 'this is an EOA'
  it means 'no code RIGHT NOW' — and CREATE2 lets someone deploy
  there later, at an address you already approved or funded.
  Worse, a self-destructed CREATE2 contract could historically be
  redeployed with DIFFERENT code at the same address; EIP-6780
  restricted SELFDESTRUCT, but the design lesson stands.

changing ANY input changes the address:
  different deployer  0x4e3D4E0C1F40eca067bcBB56bfE1A5d4BBBb21c7
  one byte of initcode changed:
  0xba038FC3D6B99A2A719A56C49F666da3fA20d5fD

  so the address commits to the exact bytecode — which is why you
  can trust a counterfactual address to hold the code you expect.
```

---

## 17. Vanity addresses, and how Profanity lost $160M

`🔴 hard` · *Entropy*

Grinding a vanity prefix is easy — and the 2022 Profanity disaster shows what happens when the grinder's *starting entropy* is only 32 bits. Wintermute lost around $160M from an address whose private key an attacker could search for in hours.

**Steps:**

1. Grind prefixes of 1 to 4 hex characters and compare tries against the expected 16ⁿ (tries are reported, not elapsed time — only the former is reproducible).
2. Read the exponential cost table out to 10 characters.
3. Understand what Profanity actually did wrong: a 32-bit seed, then a deterministic walk.
4. Note the safe alternative — grind a CREATE2 salt instead, where no private key is involved.

```go
package main

import (
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"strings"

	"github.com/ethereum/go-ethereum/crypto"
)

// Deterministic candidate keys, so this example reproduces exactly.
// A real grinder uses crypto/rand — and that difference is the whole point
// of the Profanity story at the bottom.
func candidate(seed []byte, i uint64) []byte {
	buf := make([]byte, len(seed)+8)
	copy(buf, seed)
	binary.BigEndian.PutUint64(buf[len(seed):], i)
	return crypto.Keccak256(buf)
}

func main() {
	seed := []byte("lesson-07-vanity-demo")

	for _, prefix := range []string{"a", "ab", "abc", "abcd"} {
		var found uint64
		var addr string
		for i := uint64(0); i < 20_000_000; i++ {
			key, err := crypto.ToECDSA(candidate(seed, i))
			if err != nil {
				continue // out of range for the curve (lesson 06, example 13)
			}
			a := strings.ToLower(crypto.PubkeyToAddress(key.PublicKey).Hex()[2:])
			if strings.HasPrefix(a, prefix) {
				found, addr = i, crypto.PubkeyToAddress(key.PublicKey).Hex()
				break
			}
		}
		expected := 1
		for range prefix {
			expected *= 16
		}
		// Report tries, not elapsed time: the try count is deterministic, the
		// wall clock is not.
		fmt.Printf("prefix 0x%-6s found after %8d tries (expected ~%8d)  %s\n",
			prefix, found+1, expected, addr)
	}

	// The cost is exponential in the prefix length.
	fmt.Println("\nexpected attempts by prefix length")
	e := 1
	for n := 1; n <= 10; n++ {
		e *= 16
		note := ""
		switch n {
		case 4:
			note = "  <- seconds"
		case 6:
			note = "  <- minutes"
		case 8:
			note = "  <- hours on a GPU"
		case 10:
			note = "  <- specialist hardware"
		}
		fmt.Printf("  %2d hex chars  %16d%s\n", n, e, note)
	}

	// The real key, and where it went wrong.
	key, _ := crypto.ToECDSA(candidate(seed, 0))
	fmt.Printf("\nan example candidate key: %s...\n", hex.EncodeToString(crypto.FromECDSA(key))[:24])

	fmt.Println("\nPROFANITY (2022) — the vanity generator that cost ~$160M")
	fmt.Println("  Profanity seeded its search with a 32-BIT random value, then")
	fmt.Println("  walked deterministically from there. So the whole keyspace it")
	fmt.Println("  could ever reach was 2^32 — about 4 billion keys.")
	fmt.Println("  Given a Profanity-generated ADDRESS, an attacker could search")
	fmt.Println("  that space in hours and recover the private key.")
	fmt.Println("  Wintermute lost ~$160M from a Profanity-generated vanity address.")
	fmt.Println("  Several other victims followed.")

	fmt.Println("\nthe lesson is lesson 06's, again: the curve was fine, the entropy")
	fmt.Println("was not. A vanity address is cosmetic; the key behind it must still")
	fmt.Println("come from 256 bits of real randomness.")
	fmt.Println("\nif you must have one, grind the SALT of a CREATE2 deployment")
	fmt.Println("(example 16) — no private key is involved at all.")
}
```

**Output:**

```
prefix 0xa      found after        6 tries (expected ~      16)  0xa0F9fDAd53d368bBcad1e3e373e2735B354439Fc
prefix 0xab     found after      126 tries (expected ~     256)  0xaB7d3e1760911fdc3b114b7C634784779Afe207C
prefix 0xabc    found after     6529 tries (expected ~    4096)  0xABc5b42a6ab617B1a7dfA916ce883D8d8c3Fb016
prefix 0xabcd   found after   144461 tries (expected ~   65536)  0xAbcDb70B6aaEDd4a57aEf1f416eE6B3646F10A72

expected attempts by prefix length
   1 hex chars                16
   2 hex chars               256
   3 hex chars              4096
   4 hex chars             65536  <- seconds
   5 hex chars           1048576
   6 hex chars          16777216  <- minutes
   7 hex chars         268435456
   8 hex chars        4294967296  <- hours on a GPU
   9 hex chars       68719476736
  10 hex chars     1099511627776  <- specialist hardware

an example candidate key: bc80c92700a28668a61536a3...

PROFANITY (2022) — the vanity generator that cost ~$160M
  Profanity seeded its search with a 32-BIT random value, then
  walked deterministically from there. So the whole keyspace it
  could ever reach was 2^32 — about 4 billion keys.
  Given a Profanity-generated ADDRESS, an attacker could search
  that space in hours and recover the private key.
  Wintermute lost ~$160M from a Profanity-generated vanity address.
  Several other victims followed.

the lesson is lesson 06's, again: the curve was fine, the entropy
was not. A vanity address is cosmetic; the key behind it must still
come from 256 bits of real randomness.

if you must have one, grind the SALT of a CREATE2 deployment
(example 16) — no private key is involved at all.
```

---

## 18. Storing a key: keystore v3

`🔴 hard` · *Storage*

Web3 keystore v3: scrypt over the passphrase, AES-128-CTR over the key, and a Keccak MAC that distinguishes a wrong passphrase from a corrupted file. The passphrase is the entire security, and scrypt's memory-hardness is what stands between a stolen file and the key.

**Steps:**

1. Import a test key and inspect the JSON that gets written.
2. Read the cipher, KDF and parameters out of the file.
3. Unlock with the right passphrase and with a wrong one.
4. Compare light and standard scrypt parameters, and note the memory cost.
5. Read the five storage tiers, from a process variable to an MPC service.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
	"sort"
	"time"

	"github.com/ethereum/go-ethereum/accounts/keystore"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	dir, err := os.MkdirTemp("", "keystore")
	if err != nil {
		fmt.Println("tempdir:", err)
		return
	}
	defer os.RemoveAll(dir)

	// LightScryptN/P for a fast example. PRODUCTION USES StandardScryptN/P,
	// which is ~256x more work and takes about a second — deliberately.
	ks := keystore.NewKeyStore(dir, keystore.LightScryptN, keystore.LightScryptP)

	// The published anvil test key. TEST KEY ONLY.
	key, _ := crypto.HexToECDSA("ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	const passphrase = "correct horse battery staple"

	start := time.Now()
	acct, err := ks.ImportECDSA(key, passphrase)
	if err != nil {
		fmt.Println("import:", err)
		return
	}
	fmt.Printf("imported %s\n", acct.Address.Hex())
	fmt.Printf("encrypted in %s (light params)\n", bucket(time.Since(start)))

	// Look at what was actually written.
	files, _ := filepath.Glob(filepath.Join(dir, "*"))
	raw, _ := os.ReadFile(files[0])
	var v3 map[string]any
	if err := json.Unmarshal(raw, &v3); err != nil {
		fmt.Println("parse:", err)
		return
	}

	fmt.Printf("\nthe file is JSON, version %v, %d bytes\n", v3["version"], len(raw))
	fmt.Println("top-level fields:", sortedKeys(v3))

	c := v3["crypto"].(map[string]any)
	fmt.Println("\ncrypto fields:", sortedKeys(c))
	fmt.Printf("  cipher     %v\n", c["cipher"])
	fmt.Printf("  kdf        %v\n", c["kdf"])

	kdfp := c["kdfparams"].(map[string]any)
	fmt.Printf("  kdfparams  n=%v r=%v p=%v dklen=%v\n",
		kdfp["n"], kdfp["r"], kdfp["p"], kdfp["dklen"])

	fmt.Println("\nhow it works")
	fmt.Println("  derivedKey = scrypt(passphrase, salt, n, r, p, 32)")
	fmt.Println("  ciphertext = AES-128-CTR(derivedKey[0:16], iv, privateKey)")
	fmt.Println("  mac        = keccak256(derivedKey[16:32] ‖ ciphertext)")
	fmt.Println("  the MAC is what tells a wrong passphrase from a corrupted file")

	// Unlock with the right passphrase, and with the wrong one.
	if err := ks.Unlock(acct, passphrase); err != nil {
		fmt.Println("\nunlock:", err)
		return
	}
	fmt.Printf("\nunlock with the right passphrase: ok\n")

	err = ks.Unlock(acct, "wrong passphrase")
	fmt.Printf("unlock with the wrong one:        %v\n", err)

	// The passphrase IS the security. scrypt's cost is what stands between a
	// stolen file and the key.
	fmt.Println("\nthe passphrase is the entire security of this file")
	fmt.Printf("  light    n=%d  — this example, fast, NOT for real keys\n", keystore.LightScryptN)
	// scrypt's memory cost is about 128 * N * r bytes; r is 8 here.
	fmt.Printf("  standard n=%d  — ~1s per attempt, ~%d MB of memory\n",
		keystore.StandardScryptN, 128*keystore.StandardScryptN*8>>20)
	fmt.Println("  scrypt is memory-hard on purpose: it makes GPU and ASIC")
	fmt.Println("  brute-forcing far less effective than it would be against PBKDF2")

	fmt.Println("\nwhere keys actually live, in increasing order of safety")
	fmt.Println("  1. a variable in your process        — lessons 06-07 only")
	fmt.Println("  2. a keystore file + passphrase      — this example")
	fmt.Println("  3. an OS keychain or secret manager")
	fmt.Println("  4. a hardware wallet                 — the key never leaves the device;")
	fmt.Println("     you send a hash and receive a signature")
	fmt.Println("  5. an HSM or MPC service             — lessons 32, 43")

	fmt.Println("\nand the seed phrase behind it all (examples 6-9) needs its own plan:")
	fmt.Println("  metal backup, geographic separation, a Shamir split for large sums,")
	fmt.Println("  and an inheritance procedure someone else can actually follow.")
}

func sortedKeys(m map[string]any) []string {
	out := make([]string, 0, len(m))
	for k := range m {
		out = append(out, k)
	}
	sort.Strings(out)
	return out
}

func bucket(d time.Duration) string {
	switch {
	case d < 100*time.Millisecond:
		return "< 100ms"
	case d < time.Second:
		return "< 1s"
	}
	return ">= 1s"
}
```

**Output:**

```
imported 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
encrypted in < 100ms (light params)

the file is JSON, version 3, 489 bytes
top-level fields: [address crypto id version]

crypto fields: [cipher cipherparams ciphertext kdf kdfparams mac]
  cipher     aes-128-ctr
  kdf        scrypt
  kdfparams  n=4096 r=8 p=6 dklen=32

how it works
  derivedKey = scrypt(passphrase, salt, n, r, p, 32)
  ciphertext = AES-128-CTR(derivedKey[0:16], iv, privateKey)
  mac        = keccak256(derivedKey[16:32] ‖ ciphertext)
  the MAC is what tells a wrong passphrase from a corrupted file

unlock with the right passphrase: ok
unlock with the wrong one:        could not decrypt key with given password

the passphrase is the entire security of this file
  light    n=4096  — this example, fast, NOT for real keys
  standard n=262144  — ~1s per attempt, ~256 MB of memory
  scrypt is memory-hard on purpose: it makes GPU and ASIC
  brute-forcing far less effective than it would be against PBKDF2

where keys actually live, in increasing order of safety
  1. a variable in your process        — lessons 06-07 only
  2. a keystore file + passphrase      — this example
  3. an OS keychain or secret manager
  4. a hardware wallet                 — the key never leaves the device;
     you send a hash and receive a signature
  5. an HSM or MPC service             — lessons 32, 43

and the seed phrase behind it all (examples 6-9) needs its own plan:
  metal backup, geographic separation, a Shamir split for large sums,
  and an inheritance procedure someone else can actually follow.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
