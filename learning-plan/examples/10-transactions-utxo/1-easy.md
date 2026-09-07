# Step 10 — Transactions & the UTXO Model · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                              # first time only
go get github.com/ethereum/go-ethereum@latest    # secp256k1 sign/verify
go get golang.org/x/crypto@latest                # RIPEMD-160, for HASH160
# paste the example into main.go, then:
go run .
```

No chain and no node — these programs build the transaction layer themselves. Every key is a
**published Hardhat/anvil test key**, every amount and timestamp is fixed, and nothing is timed,
so the output reproduces exactly. Examples 1, 3, 5, 14, 15 and 17 need no dependencies at all.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. Two ledgers, one payment

`🟢 easy` · *Accounts vs UTXO*

Two ways to answer 'who owns what'. An account ledger stores a balance and a payment edits two numbers. A UTXO set stores unspent coins, and a payment destroys one and creates two. The same payment, both ways, side by side.

**Steps:**

1. Run the payment through an `AccountLedger` and watch two numbers change.
2. Run it through a `UTXOSet` and watch one coin become two.
3. Compute balances in both — and notice the UTXO one is a SCAN, not a lookup.
4. Read why UTXO validity depends only on the inputs a transaction names, and what that buys: parallel validation and SPV proofs.
5. Read the cost: no balance anywhere, coin selection, and a change output every time.

```go
package main

import (
	"fmt"
	"sort"
)

// ===========================================================================
// Two ways to answer "who owns what".
//
//   accounts : state is a balance per account. A payment edits two numbers.
//   UTXO     : state is a set of unspent coins. A payment destroys coins and
//              creates new ones. No balance is stored anywhere.
//
// Ethereum uses the first, Bitcoin the second. The difference is not
// cosmetic: it decides what a validator must look at, and therefore what
// can be checked in parallel.
// ===========================================================================

// ------------------------------------------------------------- the accounts

type AccountLedger map[string]int64

func (l AccountLedger) Pay(from, to string, amt int64) error {
	// To decide this is valid you need the CURRENT global balance of `from`.
	// That is why account chains also need a nonce: without one, the same
	// signed payment could be replayed.
	if l[from] < amt {
		return fmt.Errorf("insufficient: %s has %d, needs %d", from, l[from], amt)
	}
	l[from] -= amt
	l[to] += amt
	return nil
}

// ----------------------------------------------------------------- the UTXO

type Outpoint struct {
	TxID  string // a real chain uses [32]byte; a label reads better here
	Index int
}

type Coin struct {
	Owner string
	Value int64
}

type UTXOSet map[Outpoint]Coin

// Pay spends whole coins and returns the new ones. Outputs are consumed
// entirely — anything left over must come back as an explicit change coin.
func (s UTXOSet) Pay(txid, from, to string, amt int64) error {
	var spend []Outpoint
	var total int64
	for _, op := range s.sorted() { // sorted: selection must be deterministic
		if s[op].Owner != from {
			continue
		}
		spend = append(spend, op)
		total += s[op].Value
		if total >= amt {
			break
		}
	}
	if total < amt {
		return fmt.Errorf("insufficient: %s has %d, needs %d", from, total, amt)
	}
	for _, op := range spend {
		delete(s, op) // the inputs cease to exist
	}
	s[Outpoint{txid, 0}] = Coin{to, amt}
	if change := total - amt; change > 0 {
		s[Outpoint{txid, 1}] = Coin{from, change}
	}
	return nil
}

func (s UTXOSet) sorted() []Outpoint {
	ops := make([]Outpoint, 0, len(s))
	for op := range s {
		ops = append(ops, op)
	}
	sort.Slice(ops, func(i, j int) bool {
		if ops[i].TxID != ops[j].TxID {
			return ops[i].TxID < ops[j].TxID
		}
		return ops[i].Index < ops[j].Index
	})
	return ops
}

// Balance is DERIVED. The protocol never stores it; a wallet computes it.
func (s UTXOSet) Balance(who string) int64 {
	var n int64
	for _, c := range s {
		if c.Owner == who {
			n += c.Value
		}
	}
	return n
}

func (s UTXOSet) Dump() {
	for _, op := range s.sorted() {
		c := s[op]
		fmt.Printf("    %s:%d  %-6s %5d\n", op.TxID, op.Index, c.Owner, c.Value)
	}
}

func main() {
	fmt.Println("=== the account model ===")
	acc := AccountLedger{"alice": 50, "bob": 0, "carol": 0}
	fmt.Printf("  before : alice=%d bob=%d carol=%d\n", acc["alice"], acc["bob"], acc["carol"])
	if err := acc.Pay("alice", "bob", 30); err != nil {
		fmt.Println("  ", err)
	}
	fmt.Printf("  after  : alice=%d bob=%d carol=%d\n", acc["alice"], acc["bob"], acc["carol"])
	fmt.Println("  state changed: 2 numbers edited. The balance IS the state.")

	fmt.Println("\n=== the UTXO model ===")
	set := UTXOSet{{"genesis", 0}: {"alice", 50}}
	fmt.Println("  before :")
	set.Dump()
	if err := set.Pay("tx1", "alice", "bob", 30); err != nil {
		fmt.Println("  ", err)
	}
	fmt.Println("  after  :")
	set.Dump()
	fmt.Println("  state changed: 1 coin destroyed, 2 created. No balance was stored.")

	fmt.Println("\n=== balances are derived, not stored ===")
	for _, who := range []string{"alice", "bob", "carol"} {
		fmt.Printf("  %-6s accounts=%-4d utxo=%d (summed over %d coin(s))\n",
			who, acc[who], set.Balance(who), countCoins(set, who))
	}

	fmt.Println("\n=== the property that matters ===")
	fmt.Println("  accounts : validity depends on the CURRENT balance of the sender,")
	fmt.Println("             so two transactions from the same account cannot be")
	fmt.Println("             checked independently — the first changes the answer")
	fmt.Println("             for the second. Hence nonces, and ordered execution.")
	fmt.Println()
	fmt.Println("  UTXO     : validity depends only on the outputs the transaction")
	fmt.Println("             NAMES. Two transactions touching different outpoints")
	fmt.Println("             never interact, so they can be verified in parallel,")
	fmt.Println("             and a light client can be shown one coin's history")
	fmt.Println("             without the rest of the chain (SPV, lesson 05).")

	fmt.Println("\n=== the cost ===")
	fmt.Println("  There is no 'balance' in the protocol. A wallet that loses track")
	fmt.Println("  of its coins cannot ask the chain 'how much do I have?' — it must")
	fmt.Println("  rescan. And every payment needs coin selection (example 14),")
	fmt.Println("  a change output, and a fee that grows with the number of inputs.")
}

func countCoins(s UTXOSet, who string) int {
	n := 0
	for _, c := range s {
		if c.Owner == who {
			n++
		}
	}
	return n
}
```

**Output:**

```
=== the account model ===
  before : alice=50 bob=0 carol=0
  after  : alice=20 bob=30 carol=0
  state changed: 2 numbers edited. The balance IS the state.

=== the UTXO model ===
  before :
    genesis:0  alice     50
  after  :
    tx1:0  bob       30
    tx1:1  alice     20
  state changed: 1 coin destroyed, 2 created. No balance was stored.

=== balances are derived, not stored ===
  alice  accounts=20   utxo=20 (summed over 1 coin(s))
  bob    accounts=30   utxo=30 (summed over 1 coin(s))
  carol  accounts=0    utxo=0 (summed over 0 coin(s))

=== the property that matters ===
  accounts : validity depends on the CURRENT balance of the sender,
             so two transactions from the same account cannot be
             checked independently — the first changes the answer
             for the second. Hence nonces, and ordered execution.

  UTXO     : validity depends only on the outputs the transaction
             NAMES. Two transactions touching different outpoints
             never interact, so they can be verified in parallel,
             and a light client can be shown one coin's history
             without the rest of the chain (SPV, lesson 05).

=== the cost ===
  There is no 'balance' in the protocol. A wallet that loses track
  of its coins cannot ask the chain 'how much do I have?' — it must
  rescan. And every payment needs coin selection (example 14),
  a change output, and a fee that grows with the number of inputs.
```

---

## 2. Inputs, outputs and outpoints

`🟢 easy` · *Transaction structure*

The four types the rest of Part 3 is built on: `Outpoint`, `TxInput`, `TxOutput`, `Transaction`. An output locks value to a hash of a public key; an input unlocks one earlier output by naming it. That is the whole model.

**Steps:**

1. Build a payment with one input and two outputs — the payment and the change.
2. See that an output is addressed from OUTSIDE, as (txid, index): outputs carry no id.
3. Confirm the fee is what is left over, not a field.
4. Delete the change output and watch 20 units silently become fee — the Paxos failure mode.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// The four types the rest of Part 3 is built on.
//
// An output locks value to a hash of a public key. An input unlocks one
// earlier output by naming it and proving ownership. That is the whole model.
// ===========================================================================

// Outpoint addresses one output: which transaction, and which output in it.
type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint // the output being spent
	Signature []byte   // proof it may be spent (example 7)
	PubKey    []byte   // 33-byte compressed key; must hash to Prev's PubKeyHash
}

type TxOutput struct {
	Value      int64  // integer units. NEVER float64 (lesson 03).
	PubKeyHash []byte // HASH160(pubkey): who may spend this next
}

type Transaction struct {
	Inputs  []TxInput
	Outputs []TxOutput
}

// --------------------------------------------------------------- test cast

// Published Hardhat/anvil test keys. TEST ONLY — never use on a real chain.
const (
	aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
	bobKey   = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
)

func pubKeyHash(hexKey string) []byte {
	priv, err := crypto.HexToECDSA(hexKey)
	if err != nil {
		panic(err)
	}
	return hash160(crypto.CompressPubkey(&priv.PublicKey))
}

// HASH160 = RIPEMD160(SHA256(x)) — Bitcoin's "shorten a public key" function.
func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func main() {
	alice, bob := pubKeyHash(aliceKey), pubKeyHash(bobKey)

	// Pretend alice already owns output 0 of some earlier transaction,
	// worth 50 units. (Where it came from is example 4: a coinbase.)
	var prevID [32]byte
	copy(prevID[:], mustHex("a1b2c3d4"+"00000000000000000000000000000000000000000000000000000000"))
	funding := TxOutput{Value: 50, PubKeyHash: alice}

	fmt.Println("=== the output alice is about to spend ===")
	fmt.Printf("  outpoint   %s:%d\n", hex.EncodeToString(prevID[:4])+"…", 0)
	fmt.Printf("  value      %d\n", funding.Value)
	fmt.Printf("  locked to  %x  (alice)\n", funding.PubKeyHash)

	// She wants to pay bob 30 and keep a fee of 1 back for the miner.
	tx := Transaction{
		Inputs: []TxInput{
			{Prev: Outpoint{TxID: prevID, Index: 0}}, // Signature/PubKey filled in later
		},
		Outputs: []TxOutput{
			{Value: 30, PubKeyHash: bob},   // the payment
			{Value: 19, PubKeyHash: alice}, // the CHANGE, back to herself
		},
	}

	fmt.Println("\n=== the transaction ===")
	fmt.Println("  inputs:")
	for i, in := range tx.Inputs {
		fmt.Printf("    [%d] spends %s:%d   sig=%v pubkey=%v (not signed yet)\n",
			i, hex.EncodeToString(in.Prev.TxID[:4])+"…", in.Prev.Index,
			in.Signature != nil, in.PubKey != nil)
	}
	fmt.Println("  outputs:")
	names := map[string]string{fmt.Sprintf("%x", alice): "alice", fmt.Sprintf("%x", bob): "bob"}
	for i, out := range tx.Outputs {
		fmt.Printf("    [%d] %3d to %x  (%s)\n", i, out.Value, out.PubKeyHash, names[fmt.Sprintf("%x", out.PubKeyHash)])
	}

	var outSum int64
	for _, o := range tx.Outputs {
		outSum += o.Value
	}
	fmt.Printf("\n  in  %d\n  out %d\n  fee %d  <- the difference, claimed by the miner (example 11)\n",
		funding.Value, outSum, funding.Value-outSum)

	fmt.Println("\n=== outputs are consumed WHOLE ===")
	fmt.Println("  There is no way to spend 30 of a 50-unit output. The input names")
	fmt.Println("  the output; naming it spends all of it. Anything you want back")
	fmt.Println("  must be an explicit change output — output [1] above.")
	fmt.Println()
	fmt.Println("  Forget it, or compute it wrong, and the value does not stay yours:")
	fmt.Println("  it silently becomes fee. In September 2023 Paxos paid ~19.8 BTC")
	fmt.Println("  (~$500,000) to move 0.07 BTC after a fee-calculation bug of exactly")
	fmt.Println("  this family. Nothing was invalid — the miner simply kept the rest.")

	// The same transaction with the change output omitted.
	oops := Transaction{Inputs: tx.Inputs, Outputs: tx.Outputs[:1]}
	var oopsSum int64
	for _, o := range oops.Outputs {
		oopsSum += o.Value
	}
	fmt.Printf("\n  without the change output: in %d, out %d, fee %d\n",
		funding.Value, oopsSum, funding.Value-oopsSum)

	fmt.Println("\n=== what an output is addressed BY ===")
	fmt.Println("  Not by an id it carries — outputs have no id. They are addressed")
	fmt.Println("  from outside, as (txid, index). Which means the txid must be a")
	fmt.Println("  deterministic function of the transaction's bytes: example 3.")
}

func mustHex(s string) []byte {
	b, err := hex.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return b
}
```

**Output:**

```
=== the output alice is about to spend ===
  outpoint   a1b2c3d4…:0
  value      50
  locked to  a55476015c13afb8afb92160329a8cde976f1f2e  (alice)

=== the transaction ===
  inputs:
    [0] spends a1b2c3d4…:0   sig=false pubkey=false (not signed yet)
  outputs:
    [0]  30 to c4841b8f5e6f10a38dc5e672e183ffd9be0cd12f  (bob)
    [1]  19 to a55476015c13afb8afb92160329a8cde976f1f2e  (alice)

  in  50
  out 49
  fee 1  <- the difference, claimed by the miner (example 11)

=== outputs are consumed WHOLE ===
  There is no way to spend 30 of a 50-unit output. The input names
  the output; naming it spends all of it. Anything you want back
  must be an explicit change output — output [1] above.

  Forget it, or compute it wrong, and the value does not stay yours:
  it silently becomes fee. In September 2023 Paxos paid ~19.8 BTC
  (~$500,000) to move 0.07 BTC after a fee-calculation bug of exactly
  this family. Nothing was invalid — the miner simply kept the rest.

  without the change output: in 50, out 30, fee 20

=== what an output is addressed BY ===
  Not by an id it carries — outputs have no id. They are addressed
  from outside, as (txid, index). Which means the txid must be a
  deterministic function of the transaction's bytes: example 3.
```

---

## 3. The txid is a hash of the bytes

`🟢 easy` · *Serialization*

The txid is the double-SHA256 of the transaction's bytes, so those bytes have to be a *function* of the transaction. Length-prefix everything, fix the field order, and never iterate a map on the way into a hash.

**Steps:**

1. Serialize a transaction and read the byte layout field by field.
2. Rebuild the same transaction from scratch and confirm the id matches.
3. Change one unit of value and count the bits that differ (about half of 256).
4. Swap the two outputs and see the id change — order is part of the identity.
5. Change the signature bytes and meet transaction malleability.
6. Hash a map 1000 times, get 1000 answers, then sort and get one.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"sort"
)

// ===========================================================================
// The txid is a hash of the transaction's bytes. So the bytes have to be a
// FUNCTION of the transaction — one transaction, one byte string, always.
//
// Get that wrong and outputs cannot be addressed: two nodes would compute
// two different ids for the same transaction and each would think the other's
// inputs point at nothing.
// ===========================================================================

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

// Serialize writes a canonical encoding: fixed field order, fixed endianness,
// explicit lengths. Same rules as lesson 08's header, one level up.
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

// Length-prefix every variable-length field. Without the prefix, "ab"+"c" and
// "a"+"bc" serialize identically — two transactions, one txid.
func writeBytes(b *bytes.Buffer, p []byte) {
	binary.Write(b, binary.BigEndian, uint32(len(p)))
	b.Write(p)
}

func (t *Transaction) TxID() [32]byte {
	f := sha256.Sum256(t.Serialize())
	return sha256.Sum256(f[:])
}

// --------------------------------------------------- the non-canonical way

// A map has no order. Ranging over it and hashing what comes out gives a
// different answer nearly every time — in the SAME process, on the SAME data.
func mapDigest(outs map[string]int64) [32]byte {
	var b bytes.Buffer
	for k, v := range outs {
		b.WriteString(k)
		binary.Write(&b, binary.BigEndian, v)
	}
	return sha256.Sum256(b.Bytes())
}

func main() {
	build := func() Transaction {
		var prev [32]byte
		prev[0], prev[1] = 0xa1, 0xb2
		return Transaction{
			Inputs:  []TxInput{{Prev: Outpoint{prev, 0}, Signature: []byte("sig"), PubKey: []byte("pk")}},
			Outputs: []TxOutput{{30, []byte("bob-hash-20-bytes-xx")}, {19, []byte("alice-hash-20-byte-x")}},
		}
	}

	tx := build()
	raw := tx.Serialize()
	fmt.Printf("=== the bytes (%d of them) ===\n%x\n", len(raw), raw)
	fmt.Println("\n  layout:")
	fmt.Println("    u32  input count")
	fmt.Println("    [32] prev txid   u32 prev index")
	fmt.Println("    u32  len(sig)    sig bytes")
	fmt.Println("    u32  len(pubkey) pubkey bytes")
	fmt.Println("    u32  output count")
	fmt.Println("    i64  value       u32 len(hash)  hash bytes")

	fmt.Printf("\n=== txid ===\n  %x\n", tx.TxID())

	// 1. Same transaction, built again from scratch: same id.
	again := build()
	fmt.Printf("\n  rebuilt from scratch  -> same id: %v\n", again.TxID() == tx.TxID())

	// 2. One unit of value different: a completely different id (lesson 04).
	cheaper := build()
	cheaper.Outputs[0].Value = 29
	fmt.Printf("  one unit less on out[0] -> %x\n", cheaper.TxID())
	fmt.Printf("  bits differing from the original: %d of 256\n", hamming(tx.TxID(), cheaper.TxID()))

	// 3. Order is part of the identity: outputs are addressed BY INDEX.
	swapped := build()
	swapped.Outputs[0], swapped.Outputs[1] = swapped.Outputs[1], swapped.Outputs[0]
	fmt.Printf("\n  outputs swapped       -> same id: %v\n", swapped.TxID() == tx.TxID())
	fmt.Println("    (it must differ: 'output 0' means something different now)")

	// 4. The signature is inside the id. That is malleability.
	malleated := build()
	malleated.Inputs[0].Signature = []byte("SIG")
	fmt.Printf("\n  signature bytes changed -> same id: %v\n", malleated.TxID() == tx.TxID())
	fmt.Println("    Bitcoin's legacy txid covers the signatures too, so a third")
	fmt.Println("    party who re-encodes a signature changes the txid without")
	fmt.Println("    changing what the transaction does. That is TRANSACTION")
	fmt.Println("    MALLEABILITY — Mt. Gox's excuse in 2014, and the reason")
	fmt.Println("    segwit moved signatures out of the txid (lesson 36).")

	// 5. The trap: serializing from a map.
	fmt.Println("\n=== why not just range over a map ===")
	m := map[string]int64{"bob": 30, "alice": 19, "carol": 1}
	first := mapDigest(m)
	stable := true
	for i := 0; i < 1000; i++ {
		if mapDigest(m) != first {
			stable = false
			break
		}
	}
	fmt.Printf("  1000 digests of the SAME map, same process, all equal: %v\n", stable)
	fmt.Println("  Go randomises map iteration deliberately, so code that depends")
	fmt.Println("  on the order fails in testing rather than in production.")

	// The fix: sort into a canonical order first.
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	canon := func() [32]byte {
		var b bytes.Buffer
		for _, k := range keys {
			b.WriteString(k)
			binary.Write(&b, binary.BigEndian, m[k])
		}
		return sha256.Sum256(b.Bytes())
	}
	c := canon()
	ok := true
	for i := 0; i < 1000; i++ {
		if canon() != c {
			ok = false
			break
		}
	}
	fmt.Printf("  sorted first, 1000 digests all equal: %v\n", ok)

	fmt.Println("\n  Rule: anything that goes into a hash comes out of a slice, in a")
	fmt.Println("  documented order — never out of a map, and never out of fmt or JSON.")
}

func hamming(a, b [32]byte) int {
	n := 0
	for i := range a {
		x := a[i] ^ b[i]
		for ; x != 0; x &= x - 1 {
			n++
		}
	}
	return n
}
```

**Output:**

```
=== the bytes (121 of them) ===
00000001a1b2000000000000000000000000000000000000000000000000000000000000000000000000000373696700000002706b00000002000000000000001e00000014626f622d686173682d32302d62797465732d7878000000000000001300000014616c6963652d686173682d32302d627974652d78

  layout:
    u32  input count
    [32] prev txid   u32 prev index
    u32  len(sig)    sig bytes
    u32  len(pubkey) pubkey bytes
    u32  output count
    i64  value       u32 len(hash)  hash bytes

=== txid ===
  87ca16502100f72a4ac98b147bcc895f7dbafd418f8a5ae3d784785b79180345

  rebuilt from scratch  -> same id: true
  one unit less on out[0] -> 089d33326330b3761e24dd9ae5c3c6ed800b05ebd9ef77639d83b6471da43439
  bits differing from the original: 126 of 256

  outputs swapped       -> same id: false
    (it must differ: 'output 0' means something different now)

  signature bytes changed -> same id: false
    Bitcoin's legacy txid covers the signatures too, so a third
    party who re-encodes a signature changes the txid without
    changing what the transaction does. That is TRANSACTION
    MALLEABILITY — Mt. Gox's excuse in 2014, and the reason
    segwit moved signatures out of the txid (lesson 36).

=== why not just range over a map ===
  1000 digests of the SAME map, same process, all equal: false
  Go randomises map iteration deliberately, so code that depends
  on the order fails in testing rather than in production.
  sorted first, 1000 digests all equal: true

  Rule: anything that goes into a hash comes out of a slice, in a
  documented order — never out of a map, and never out of fmt or JSON.
```

---

## 4. The coinbase transaction

`🟢 easy` · *The coinbase*

The one transaction with no inputs. It creates the block subsidy plus the fees, and its input slot carries arbitrary data — the extra nonce, signalling, the block height, and in block 0, a newspaper headline.

**Steps:**

1. Build a coinbase carrying Satoshi's headline and print its txid.
2. See the null outpoint: 32 zero bytes and index 0xffffffff.
3. Vary the extra nonce and watch the txid — and therefore the Merkle root — change.
4. Reproduce the duplicate-coinbase bug that destroyed 100 BTC on mainnet.
5. Prefix the block height (BIP-34) and watch duplicates become impossible.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Every unit of currency that exists was created by a coinbase transaction.
// It is the one transaction allowed to have no inputs — so it is the one
// place where the conservation rule (example 11) does not apply, and the
// only place new supply can appear.
// ===========================================================================

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint
	Signature []byte // on a coinbase: arbitrary data, not a signature
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

// The null outpoint: 32 zero bytes and index 0xffffffff. Nothing can ever
// hash to all zeros, so this cannot collide with a real output.
var nullOutpoint = Outpoint{TxID: [32]byte{}, Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

// NewCoinbase pays `value` to `to`, carrying `data` in the input's slot where
// a signature would otherwise go.
func NewCoinbase(to []byte, value int64, data []byte) *Transaction {
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
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

const minerKey = "7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6" // TEST ONLY

func pubKeyHash(hexKey string) []byte {
	priv, _ := crypto.HexToECDSA(hexKey)
	s := sha256.Sum256(crypto.CompressPubkey(&priv.PublicKey))
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func main() {
	miner := pubKeyHash(minerKey)

	// Block 0's coinbase, with the message Satoshi put in it.
	headline := []byte("The Times 03/Jan/2009 Chancellor on brink of second bailout for banks")
	cb := NewCoinbase(miner, 50, headline)

	fmt.Println("=== a coinbase transaction ===")
	fmt.Printf("  is coinbase   %v\n", cb.IsCoinbase())
	fmt.Printf("  input[0].prev %x:%d   <- the null outpoint\n",
		cb.Inputs[0].Prev.TxID[:8], cb.Inputs[0].Prev.Index)
	fmt.Printf("  input[0].data %q\n", cb.Inputs[0].Signature)
	fmt.Printf("  output[0]     %d to %x\n", cb.Outputs[0].Value, cb.Outputs[0].PubKeyHash)
	fmt.Printf("  txid          %x\n", cb.TxID())

	fmt.Println("\n=== what the data field is for ===")
	fmt.Println("  It is not validated, so it has been used for everything:")
	fmt.Println("    - the headline above: a timestamp nobody could have forged")
	fmt.Println("    - the EXTRA NONCE, when 2^32 header nonces run out (lesson 09)")
	fmt.Println("    - signalling: \"/P2SH/\", \"/NYA/\", pool names, BIP-9 flags")
	fmt.Println("    - the block height, mandatory since BIP-34 — see below")

	fmt.Println("\n=== extra nonce: more search space, for free ===")
	for _, extra := range []uint64{0, 1, 2} {
		data := append([]byte("miner-x/"), make([]byte, 8)...)
		binary.BigEndian.PutUint64(data[8:], extra)
		id := NewCoinbase(miner, 50, data).TxID()
		fmt.Printf("  extra nonce %d -> coinbase txid %x…\n", extra, id[:12])
	}
	fmt.Println("  A different coinbase txid means a different Merkle root, which")
	fmt.Println("  means a different header, which means a fresh 2^32 nonces to try.")

	fmt.Println("\n=== the duplicate-coinbase bug ===")
	// Same miner, same reward, same data -> byte-identical transaction.
	a := NewCoinbase(miner, 50, []byte("/pool/"))
	b := NewCoinbase(miner, 50, []byte("/pool/"))
	fmt.Printf("  two blocks, identical coinbase data -> same txid: %v\n", a.TxID() == b.TxID())
	fmt.Println("  That happened on mainnet. Blocks 91722/91880 and 91812/91842 each")
	fmt.Println("  contain byte-identical coinbases, so the second overwrote the")
	fmt.Println("  first in the UTXO set and destroyed 50 BTC — twice.")
	fmt.Println("  BIP-30 banned duplicate txids; BIP-34 then made them impossible")
	fmt.Println("  by requiring the block HEIGHT as the first item of coinbase data.")

	// BIP-34 style: height first. Now the two are distinct by construction.
	withHeight := func(h uint64, tag string) *Transaction {
		d := make([]byte, 8, 8+len(tag))
		binary.BigEndian.PutUint64(d, h)
		return NewCoinbase(miner, 50, append(d, tag...))
	}
	c1, c2 := withHeight(91722, "/pool/"), withHeight(91880, "/pool/")
	fmt.Printf("\n  with the height prefixed -> same txid: %v\n", c1.TxID() == c2.TxID())

	fmt.Println("\n=== two rules that are easy to forget ===")
	fmt.Println("  1. VALUE: coinbase output <= subsidy + the fees of this block's")
	fmt.Println("     other transactions. Claiming more makes the block invalid")
	fmt.Println("     (example 11). Claiming LESS is legal — and burns the rest.")
	fmt.Println("  2. MATURITY: coinbase outputs are unspendable for 100 blocks,")
	fmt.Println("     because a reorg can orphan the block that created them and")
	fmt.Println("     every spend descended from it (example 16).")
}
```

**Output:**

```
=== a coinbase transaction ===
  is coinbase   true
  input[0].prev 0000000000000000:4294967295   <- the null outpoint
  input[0].data "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"
  output[0]     50 to 245289ae1d4ab1b27e13e44e77a3ce0ebb2f445c
  txid          17b7a55f4fc518930604e1125d22aed733383313a52c0b3f5aada3f46196a2b8

=== what the data field is for ===
  It is not validated, so it has been used for everything:
    - the headline above: a timestamp nobody could have forged
    - the EXTRA NONCE, when 2^32 header nonces run out (lesson 09)
    - signalling: "/P2SH/", "/NYA/", pool names, BIP-9 flags
    - the block height, mandatory since BIP-34 — see below

=== extra nonce: more search space, for free ===
  extra nonce 0 -> coinbase txid ed2794b54a60e13a335953cd…
  extra nonce 1 -> coinbase txid d5b6510d8630166c92f59c75…
  extra nonce 2 -> coinbase txid 1577639140e89f404fcc383e…
  A different coinbase txid means a different Merkle root, which
  means a different header, which means a fresh 2^32 nonces to try.

=== the duplicate-coinbase bug ===
  two blocks, identical coinbase data -> same txid: true
  That happened on mainnet. Blocks 91722/91880 and 91812/91842 each
  contain byte-identical coinbases, so the second overwrote the
  first in the UTXO set and destroyed 50 BTC — twice.
  BIP-30 banned duplicate txids; BIP-34 then made them impossible
  by requiring the block HEIGHT as the first item of coinbase data.

  with the height prefixed -> same txid: false

=== two rules that are easy to forget ===
  1. VALUE: coinbase output <= subsidy + the fees of this block's
     other transactions. Claiming more makes the block invalid
     (example 11). Claiming LESS is legal — and burns the rest.
  2. MATURITY: coinbase outputs are unspendable for 100 blocks,
     because a reorg can orphan the block that created them and
     every spend descended from it (example 16).
```

---

## 5. The subsidy schedule

`🟢 easy` · *Monetary policy*

`subsidy = 50 BTC >> (height / 210000)`. One line of integer arithmetic decides the entire monetary policy — and, because miners are paid out of it, the entire long-run security budget.

**Steps:**

1. Print all 33 halving eras and the running supply.
2. Watch the total land on 20,999,999.9769 BTC, and see that the shortfall is just truncation.
3. Note that actual supply is lower: coinbases may claim less, and several have.
4. Line up the five halvings that have happened against their dates.
5. Read the security-budget problem: the subsidy goes to zero, so fees have to carry it.

```go
package main

import "fmt"

// ===========================================================================
// The subsidy schedule: how much a coinbase may claim, and for how long.
//
//   subsidy(height) = 50 BTC >> (height / 210000)
//
// One line of integer arithmetic decides the entire monetary policy — and,
// because miners are paid out of it, the entire long-run security budget.
// ===========================================================================

const (
	Coin          = int64(100_000_000) // satoshis per BTC
	HalvingPeriod = int64(210_000)     // blocks
	InitialReward = 50 * Coin
)

// Subsidy is exactly Bitcoin's. The `>= 64` guard matters: shifting an int64
// by 64 or more is undefined in C and would be a consensus split; in Go it is
// defined (result 0), but writing the guard says what you mean.
func Subsidy(height int64) int64 {
	halvings := height / HalvingPeriod
	if halvings >= 64 {
		return 0
	}
	return InitialReward >> uint(halvings)
}

// btc formats satoshis without ever touching a float (lesson 03).
func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func main() {
	fmt.Println("=== the halving schedule ===")
	fmt.Printf("%-5s %-9s %-16s %-20s %s\n", "era", "height", "subsidy (BTC)", "created in era", "cumulative")

	var total int64
	era := int64(0)
	for ; ; era++ {
		s := Subsidy(era * HalvingPeriod)
		if s == 0 {
			break
		}
		created := s * HalvingPeriod
		total += created
		if era < 10 || era > 30 {
			fmt.Printf("%-5d %-9d %-16s %-20s %s\n",
				era, era*HalvingPeriod, btc(s), btc(created), btc(total))
		} else if era == 10 {
			fmt.Println("  ...")
		}
	}

	fmt.Printf("\n  subsidy reaches 0 at era %d, height %d\n", era, era*HalvingPeriod)
	fmt.Printf("  theoretical maximum supply: %s BTC\n", btc(total))
	fmt.Println("  (the famous \"21 million\" is 20,999,999.9769 — the shortfall is")
	fmt.Println("   integer truncation in the last few eras, nothing more)")

	fmt.Println("\n  Actual supply will be LOWER. Coinbases may claim less than the")
	fmt.Println("  subsidy, and several have: block 501726 claimed no reward at all,")
	fmt.Println("  destroying 12.5 BTC, and the duplicate coinbases of example 4")
	fmt.Println("  cost another 100. Nothing in the protocol replaces burned coins.")

	fmt.Println("\n=== the halvings that have happened ===")
	dates := []struct {
		height int64
		date   string
	}{
		{0, "2009-01-03"}, {210_000, "2012-11-28"}, {420_000, "2016-07-09"},
		{630_000, "2020-05-11"}, {840_000, "2024-04-19"},
	}
	for _, d := range dates {
		fmt.Printf("  height %-8d %s   subsidy becomes %s BTC\n", d.height, d.date, btc(Subsidy(d.height)))
	}
	fmt.Println("  Roughly four years apart, because 210,000 blocks at 10 minutes")
	fmt.Println("  is 1458 days — but it is BLOCKS, not time, that trigger it.")

	fmt.Println("\n=== the security budget problem ===")
	fmt.Println("  Miners are paid subsidy + fees, and buy hashrate with it. Hashrate")
	fmt.Println("  is what a 51% attack has to out-spend (lesson 09). So:")
	fmt.Println()
	fmt.Printf("    era 0  : %s BTC per block from subsidy\n", btc(Subsidy(0)))
	fmt.Printf("    era 4  : %s BTC per block from subsidy\n", btc(Subsidy(4*HalvingPeriod)))
	fmt.Printf("    era 10 : %s BTC per block from subsidy\n", btc(Subsidy(10*HalvingPeriod)))
	fmt.Printf("    era 33 : %s BTC per block from subsidy\n", btc(Subsidy(33*HalvingPeriod)))
	fmt.Println()
	fmt.Println("  The subsidy halves toward zero, so in the long run FEES have to")
	fmt.Println("  carry the whole security budget. Whether a fee market alone can")
	fmt.Println("  pay for enough hashrate is genuinely unsettled — it is the main")
	fmt.Println("  open question about proof of work's end state, and it is why")
	fmt.Println("  fee-market design (lesson 11) is not a side topic.")

	fmt.Println("\n=== where this lands in the code ===")
	fmt.Println("  A block's coinbase may claim at most:")
	fmt.Println("      Subsidy(height) + sum(fees of the other transactions)")
	fmt.Println("  Example 11 enforces it; example 18 wires it into the chain.")
}
```

**Output:**

```
=== the halving schedule ===
era   height    subsidy (BTC)    created in era       cumulative
0     0         50.00000000      10500000.00000000    10500000.00000000
1     210000    25.00000000      5250000.00000000     15750000.00000000
2     420000    12.50000000      2625000.00000000     18375000.00000000
3     630000    6.25000000       1312500.00000000     19687500.00000000
4     840000    3.12500000       656250.00000000      20343750.00000000
5     1050000   1.56250000       328125.00000000      20671875.00000000
6     1260000   0.78125000       164062.50000000      20835937.50000000
7     1470000   0.39062500       82031.25000000       20917968.75000000
8     1680000   0.19531250       41015.62500000       20958984.37500000
9     1890000   0.09765625       20507.81250000       20979492.18750000
  ...
31    6510000   0.00000002       0.00420000           20999999.97480000
32    6720000   0.00000001       0.00210000           20999999.97690000

  subsidy reaches 0 at era 33, height 6930000
  theoretical maximum supply: 20999999.97690000 BTC
  (the famous "21 million" is 20,999,999.9769 — the shortfall is
   integer truncation in the last few eras, nothing more)

  Actual supply will be LOWER. Coinbases may claim less than the
  subsidy, and several have: block 501726 claimed no reward at all,
  destroying 12.5 BTC, and the duplicate coinbases of example 4
  cost another 100. Nothing in the protocol replaces burned coins.

=== the halvings that have happened ===
  height 0        2009-01-03   subsidy becomes 50.00000000 BTC
  height 210000   2012-11-28   subsidy becomes 25.00000000 BTC
  height 420000   2016-07-09   subsidy becomes 12.50000000 BTC
  height 630000   2020-05-11   subsidy becomes 6.25000000 BTC
  height 840000   2024-04-19   subsidy becomes 3.12500000 BTC
  Roughly four years apart, because 210,000 blocks at 10 minutes
  is 1458 days — but it is BLOCKS, not time, that trigger it.

=== the security budget problem ===
  Miners are paid subsidy + fees, and buy hashrate with it. Hashrate
  is what a 51% attack has to out-spend (lesson 09). So:

    era 0  : 50.00000000 BTC per block from subsidy
    era 4  : 3.12500000 BTC per block from subsidy
    era 10 : 0.04882812 BTC per block from subsidy
    era 33 : 0.00000000 BTC per block from subsidy

  The subsidy halves toward zero, so in the long run FEES have to
  carry the whole security budget. Whether a fee market alone can
  pay for enough hashrate is genuinely unsettled — it is the main
  open question about proof of work's end state, and it is why
  fee-market design (lesson 11) is not a side topic.

=== where this lands in the code ===
  A block's coinbase may claim at most:
      Subsidy(height) + sum(fees of the other transactions)
  Example 11 enforces it; example 18 wires it into the chain.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
