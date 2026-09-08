# Step 10 — Transactions & the UTXO Model · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. What exactly gets signed

`🟡 medium` · *Signing*

'Sign the transaction' is not precise enough to implement, because the signature lives inside the transaction. What actually gets signed is a **trimmed copy**: signatures cleared, and the referenced output's locking hash placed in the pubkey slot of the one input being signed.

**Steps:**

1. Print one sighash per input and see that they differ — a signature cannot be moved between inputs.
2. Mutate five different things and confirm every one changes the sighash.
3. Sign for real, then compare the signed and trimmed serializations.
4. Confirm the sighashes are the same before and after signing — they have to be.
5. Read the sighash types (ALL / NONE / SINGLE / ANYONECANPAY) that lesson 36 covers properly.

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
// "Sign the transaction" is not precise enough to implement.
//
// The signature lives INSIDE the transaction, so it cannot commit to a
// transaction that already contains it. What actually gets signed is a
// TRIMMED COPY: signatures cleared, and — for the one input being signed —
// the referenced output's locking hash put in the pubkey slot, so the
// signature also commits to WHICH coin this input spends.
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

// TrimmedCopy returns a DEEP copy with every signature and pubkey cleared.
// Deep matters: a shallow copy shares the input slice, so writing into the
// copy corrupts the original (example 8).
func (t *Transaction) TrimmedCopy() *Transaction {
	ins := make([]TxInput, len(t.Inputs))
	for i, in := range t.Inputs {
		ins[i] = TxInput{Prev: in.Prev} // Signature and PubKey left nil
	}
	outs := make([]TxOutput, len(t.Outputs))
	for i, o := range t.Outputs {
		outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
	}
	return &Transaction{Inputs: ins, Outputs: outs}
}

// SigHash: what input i actually signs.
func (t *Transaction) SigHash(i int, prevPubKeyHash []byte) [32]byte {
	c := t.TrimmedCopy()
	c.Inputs[i].PubKey = prevPubKeyHash // only this one input is "filled in"
	return c.TxID()
}

// --------------------------------------------------------------- test cast

const (
	aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
	bobKey   = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d" // TEST ONLY
)

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func pkh(hexKey string) []byte {
	priv, _ := crypto.HexToECDSA(hexKey)
	return hash160(crypto.CompressPubkey(&priv.PublicKey))
}

func id(b byte) [32]byte { var x [32]byte; x[0] = b; return x }

func main() {
	alice, bob := pkh(aliceKey), pkh(bobKey)

	// Alice spends two of her own coins into one payment to bob, plus change.
	tx := &Transaction{
		Inputs: []TxInput{
			{Prev: Outpoint{id(0xaa), 0}},
			{Prev: Outpoint{id(0xbb), 1}},
		},
		Outputs: []TxOutput{
			{Value: 70, PubKeyHash: bob},
			{Value: 29, PubKeyHash: alice},
		},
	}
	// Both inputs spend outputs locked to alice.
	prevPKH := [][]byte{alice, alice}

	fmt.Println("=== one sighash per input ===")
	for i := range tx.Inputs {
		h := tx.SigHash(i, prevPKH[i])
		fmt.Printf("  input %d signs %x\n", i, h)
	}
	fmt.Println("  The two differ even though the outputs are identical, because")
	fmt.Println("  each fills in ITS OWN pubkey slot. So a signature made for")
	fmt.Println("  input 0 is not valid at input 1 — you cannot lift a signature")
	fmt.Println("  from one position to another.")

	fmt.Println("\n=== what the signature commits to ===")
	base := tx.SigHash(0, alice)
	type variant struct {
		what string
		mut  func(*Transaction)
	}
	for _, v := range []variant{
		{"output 0's value: 70 -> 71", func(t *Transaction) { t.Outputs[0].Value = 71 }},
		{"output 0's recipient: bob -> alice", func(t *Transaction) { t.Outputs[0].PubKeyHash = alice }},
		{"the two outputs swapped", func(t *Transaction) { t.Outputs[0], t.Outputs[1] = t.Outputs[1], t.Outputs[0] }},
		{"input 1's outpoint index: 1 -> 9", func(t *Transaction) { t.Inputs[1].Prev.Index = 9 }},
		{"a third output appended", func(t *Transaction) { t.Outputs = append(t.Outputs, TxOutput{1, bob}) }},
	} {
		m := tx.TrimmedCopy()
		v.mut(m)
		fmt.Printf("  %-36s sighash unchanged: %v\n", v.what, m.SigHash(0, alice) == base)
	}

	fmt.Println("\n  Everything that decides where the money goes is covered. That is")
	fmt.Println("  the point: once signed, the transaction is frozen. Which is also")
	fmt.Println("  the pitfall — sign only when every output is final, or you have")
	fmt.Println("  to start over (example 7).")

	// Now sign both inputs for real, so the trimming is visible on a
	// transaction that actually carries signatures.
	fmt.Println("\n=== the trimmed copy, on a signed transaction ===")
	for i := range tx.Inputs {
		priv, _ := crypto.HexToECDSA(aliceKey)
		h := tx.SigHash(i, prevPKH[i])
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			panic(err)
		}
		tx.Inputs[i].Signature = sig
		tx.Inputs[i].PubKey = crypto.CompressPubkey(&priv.PublicKey)
	}
	c := tx.TrimmedCopy()
	fmt.Printf("  signed, serialized : %d bytes\n", len(tx.Serialize()))
	fmt.Printf("  trimmed, serialized: %d bytes  (2 x 65-byte sig + 2 x 33-byte key gone)\n", len(c.Serialize()))
	fmt.Println("  what survives: every outpoint, every output value, every")
	fmt.Println("  output's locking hash, and the ORDER of all of them.")
	fmt.Printf("  and the sighashes are the same as before signing: %v\n",
		tx.SigHash(0, alice) == base)
	fmt.Println("  — they have to be, or verification could never recompute them.")

	fmt.Println("\n=== what is NOT covered, and why ===")
	fmt.Println("  The signatures themselves. A signature cannot commit to itself,")
	fmt.Println("  so anyone can re-encode one without breaking validity — and in")
	fmt.Println("  our model the txid covers signatures, so re-encoding changes the")
	fmt.Println("  id. That is malleability again (example 3).")

	fmt.Println("\n=== sighash types (previewed; lesson 36 does them properly) ===")
	fmt.Println("  Bitcoin lets each signature choose HOW MUCH of the transaction")
	fmt.Println("  it covers, with one byte appended to the signature:")
	fmt.Println("    ALL (0x01)           every input and every output   <- what we do")
	fmt.Println("    NONE (0x02)          the inputs; outputs are free to change")
	fmt.Println("    SINGLE (0x03)        the inputs and the output at the same index")
	fmt.Println("    | ANYONECANPAY (0x80) only THIS input; others may be added")
	fmt.Println()
	fmt.Println("  These are what make CoinJoin, crowdfunded transactions and")
	fmt.Println("  payment channels expressible. They are also sharp: SIGHASH_SINGLE")
	fmt.Println("  with no matching output hashes the value 1 in Bitcoin — a bug")
	fmt.Println("  preserved for compatibility, and a real way to lose coins.")
}
```

**Output:**

```
=== one sighash per input ===
  input 0 signs 65886f9963b8d9b66727d4b184bfc05caed29538e88b6cbb54ca28db5166f782
  input 1 signs 519c8357eef5211d6351860a615ccb4a53e267c2909afc066c5ac75a6e50f7d5
  The two differ even though the outputs are identical, because
  each fills in ITS OWN pubkey slot. So a signature made for
  input 0 is not valid at input 1 — you cannot lift a signature
  from one position to another.

=== what the signature commits to ===
  output 0's value: 70 -> 71           sighash unchanged: false
  output 0's recipient: bob -> alice   sighash unchanged: false
  the two outputs swapped              sighash unchanged: false
  input 1's outpoint index: 1 -> 9     sighash unchanged: false
  a third output appended              sighash unchanged: false

  Everything that decides where the money goes is covered. That is
  the point: once signed, the transaction is frozen. Which is also
  the pitfall — sign only when every output is final, or you have
  to start over (example 7).

=== the trimmed copy, on a signed transaction ===
  signed, serialized : 356 bytes
  trimmed, serialized: 160 bytes  (2 x 65-byte sig + 2 x 33-byte key gone)
  what survives: every outpoint, every output value, every
  output's locking hash, and the ORDER of all of them.
  and the sighashes are the same as before signing: true
  — they have to be, or verification could never recompute them.

=== what is NOT covered, and why ===
  The signatures themselves. A signature cannot commit to itself,
  so anyone can re-encode one without breaking validity — and in
  our model the txid covers signatures, so re-encoding changes the
  id. That is malleability again (example 3).

=== sighash types (previewed; lesson 36 does them properly) ===
  Bitcoin lets each signature choose HOW MUCH of the transaction
  it covers, with one byte appended to the signature:
    ALL (0x01)           every input and every output   <- what we do
    NONE (0x02)          the inputs; outputs are free to change
    SINGLE (0x03)        the inputs and the output at the same index
    | ANYONECANPAY (0x80) only THIS input; others may be added

  These are what make CoinJoin, crowdfunded transactions and
  payment channels expressible. They are also sharp: SIGHASH_SINGLE
  with no matching output hashes the value 1 in Bitcoin — a bug
  preserved for compatibility, and a real way to lose coins.
```

---

## 7. Sign it, then try to change it

`🟡 medium` · *Signing*

Sign a transaction, verify it, then try to change it. Six mutations, six rejections — including appending an output after the fact, which is why a wallet must sign LAST.

**Steps:**

1. Implement `Sign` and `Verify` over the trimmed copy.
2. Watch verification fail before signing, for the right reason.
3. Change an output's value by one unit and see the signature die.
4. Try five more mutations: recipient, order, outpoint, one flipped signature bit, an extra output.
5. Restore everything and confirm it verifies again.

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
// Signing and verifying, end to end — and then the demonstration that the
// signature is worth having: change ONE unit of one output's value and the
// whole transaction becomes invalid.
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

// ------------------------------------------------------------ sign / verify

// Sign fills in every input. prevOuts[i] is the output that input i spends —
// the caller has to look those up, because the transaction only names them.
func (t *Transaction) Sign(priv *ecdsa.PrivateKey, prevOuts []TxOutput) error {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			return fmt.Errorf("input %d: %w", i, err)
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return nil
}

// Verify re-derives each sighash and checks the signature over it. Note that
// it needs prevOuts too: a signature is only meaningful against the coin it
// claims to spend.
func (t *Transaction) Verify(prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: public key does not match the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if len(in.Signature) < 64 || !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: bad signature", i)
		}
	}
	return nil
}

const aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
const bobKey = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"   // TEST ONLY

func result(err error) string {
	if err == nil {
		return "ok"
	}
	return "REJECTED: " + err.Error()
}

func pkh(hexKey string) []byte {
	priv, _ := crypto.HexToECDSA(hexKey)
	return hash160(crypto.CompressPubkey(&priv.PublicKey))
}

func main() {
	alicePriv, _ := crypto.HexToECDSA(aliceKey)
	alice, bob := pkh(aliceKey), pkh(bobKey)

	// The coin alice is spending: 50 units, locked to her key hash.
	funding := TxOutput{Value: 50, PubKeyHash: alice}
	var prevID [32]byte
	prevID[0] = 0xa1

	tx := &Transaction{
		Inputs:  []TxInput{{Prev: Outpoint{prevID, 0}}},
		Outputs: []TxOutput{{30, bob}, {19, alice}},
	}
	prevOuts := []TxOutput{funding}

	fmt.Println("=== before signing ===")
	fmt.Printf("  verify: %s\n", result(tx.Verify(prevOuts)))

	if err := tx.Sign(alicePriv, prevOuts); err != nil {
		panic(err)
	}
	fmt.Println("\n=== signed ===")
	fmt.Printf("  sighash    %x\n", tx.SigHash(0, alice))
	fmt.Printf("  signature  %x… (%d bytes: r‖s‖v)\n", tx.Inputs[0].Signature[:16], len(tx.Inputs[0].Signature))
	fmt.Printf("  pubkey     %x (33 bytes, compressed)\n", tx.Inputs[0].PubKey)
	fmt.Printf("  hash160    %x  == output's lock: %v\n",
		hash160(tx.Inputs[0].PubKey), bytes.Equal(hash160(tx.Inputs[0].PubKey), funding.PubKeyHash))
	fmt.Printf("  verify:    %s\n", result(tx.Verify(prevOuts)))

	fmt.Println("\n=== now tamper with it ===")
	check := func(label string, mutate, restore func()) {
		mutate()
		err := tx.Verify(prevOuts)
		res := "REJECTED"
		if err == nil {
			res = "accepted"
		}
		fmt.Printf("  %-38s %-9s %v\n", label, res, err)
		restore()
	}

	v := tx.Outputs[0].Value
	check("output 0: 30 -> 31",
		func() { tx.Outputs[0].Value = 31 },
		func() { tx.Outputs[0].Value = v })

	check("output 0: pay alice instead of bob",
		func() { tx.Outputs[0].PubKeyHash = alice },
		func() { tx.Outputs[0].PubKeyHash = bob })

	check("outputs swapped",
		func() { tx.Outputs[0], tx.Outputs[1] = tx.Outputs[1], tx.Outputs[0] },
		func() { tx.Outputs[0], tx.Outputs[1] = tx.Outputs[1], tx.Outputs[0] })

	idx := tx.Inputs[0].Prev.Index
	check("spend a different outpoint index",
		func() { tx.Inputs[0].Prev.Index = 1 },
		func() { tx.Inputs[0].Prev.Index = idx })

	sig := tx.Inputs[0].Signature
	flipped := append([]byte(nil), sig...)
	flipped[10] ^= 0x01
	check("one bit flipped in the signature",
		func() { tx.Inputs[0].Signature = flipped },
		func() { tx.Inputs[0].Signature = sig })

	outs := tx.Outputs
	check("a third output appended after signing",
		func() { tx.Outputs = append(append([]TxOutput(nil), outs...), TxOutput{1, bob}) },
		func() { tx.Outputs = outs })

	fmt.Printf("\n  everything restored -> verify: %s\n", result(tx.Verify(prevOuts)))

	fmt.Println("\n=== the habit this forces ===")
	fmt.Println("  Sign LAST. Coin selection, the change output, the fee, the")
	fmt.Println("  ordering — all of it has to be final before the first signature,")
	fmt.Println("  because there is no such thing as editing a signed transaction.")
	fmt.Println("  A wallet that signs early and adjusts the fee afterwards produces")
	fmt.Println("  transactions that every node silently drops.")
}
```

**Output:**

```
=== before signing ===
  verify: REJECTED: input 0: public key does not match the output's lock

=== signed ===
  sighash    4253a87573a3eef361b8f9fda2e63f0386104c67ded92d2b84cd3377847e387b
  signature  a3e92131f7b580eb76b72aa88bd97775… (65 bytes: r‖s‖v)
  pubkey     038318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75 (33 bytes, compressed)
  hash160    a55476015c13afb8afb92160329a8cde976f1f2e  == output's lock: true
  verify:    ok

=== now tamper with it ===
  output 0: 30 -> 31                     REJECTED  input 0: bad signature
  output 0: pay alice instead of bob     REJECTED  input 0: bad signature
  outputs swapped                        REJECTED  input 0: bad signature
  spend a different outpoint index       REJECTED  input 0: bad signature
  one bit flipped in the signature       REJECTED  input 0: bad signature
  a third output appended after signing  REJECTED  input 0: bad signature

  everything restored -> verify: ok

=== the habit this forces ===
  Sign LAST. Coin selection, the change output, the fee, the
  ordering — all of it has to be final before the first signature,
  because there is no such thing as editing a signed transaction.
  A wallet that signs early and adjusts the fee afterwards produces
  transactions that every node silently drops.
```

---

## 8. The copy that was not a copy

`🟡 medium` · *The pitfall*

`c := *t` copies the struct. The struct holds slice *headers*, so the copy shares the elements — and `TrimmedCopy` then wipes the signature you made on the previous pass round the loop. Signing succeeds, verification fails, and only for multi-input transactions.

**Steps:**

1. Sign a one-input transaction with the buggy copy: it works, and every test passes.
2. Sign a two-input one and watch input 0's signature vanish.
3. Sign a three-input one and watch only the last survive.
4. Switch to the deep copy and scribble all over the result — the original is byte-identical.
5. Read why neither `go vet` nor `-race` will find this, and which test does.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"strings"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// The pitfall that eats an afternoon: TrimmedCopy that is not actually a copy.
//
// `c := *t` copies the struct. The struct contains SLICE HEADERS, and copying
// a slice header copies the pointer, not the elements. So `c.Inputs[i] = ...`
// writes straight into the original's inputs — and clears the signature you
// made on the previous pass round the loop.
//
// The symptom is horrible: signing succeeds, verification fails, and only for
// transactions with more than one input.
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

// ---------------------------------------------------- the broken copy

// TrimmedCopyBuggy looks right and compiles. `c := *t` is a struct copy, and
// the loop then writes through c.Inputs — which is the SAME backing array
// as t.Inputs.
func (t *Transaction) TrimmedCopyBuggy() *Transaction {
	c := *t
	for i := range c.Inputs {
		c.Inputs[i].Signature = nil
		c.Inputs[i].PubKey = nil
	}
	return &c
}

func (t *Transaction) SigHashBuggy(i int, prevPubKeyHash []byte) [32]byte {
	c := t.TrimmedCopyBuggy()
	c.Inputs[i].PubKey = prevPubKeyHash
	return c.TxID()
}

// ------------------------------------------------------------ sign / verify

func (t *Transaction) sign(priv *ecdsa.PrivateKey, prevOuts []TxOutput, buggy bool) {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		var h [32]byte
		if buggy {
			h = t.SigHashBuggy(i, prevOuts[i].PubKeyHash)
		} else {
			h = t.SigHash(i, prevOuts[i].PubKeyHash)
		}
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			panic(err)
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
}

func (t *Transaction) Verify(prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: public key does not match the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if len(in.Signature) < 64 || !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: bad signature", i)
		}
	}
	return nil
}

const aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
const bobKey = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"   // TEST ONLY

func pkh(hexKey string) []byte {
	priv, _ := crypto.HexToECDSA(hexKey)
	return hash160(crypto.CompressPubkey(&priv.PublicKey))
}

func build(alice, bob []byte, n int) (*Transaction, []TxOutput) {
	tx := &Transaction{Outputs: []TxOutput{{int64(30 * n), bob}, {int64(19 * n), alice}}}
	var prevOuts []TxOutput
	for i := 0; i < n; i++ {
		var id [32]byte
		id[0] = byte(0xa0 + i)
		tx.Inputs = append(tx.Inputs, TxInput{Prev: Outpoint{id, 0}})
		prevOuts = append(prevOuts, TxOutput{Value: 50, PubKeyHash: alice})
	}
	return tx, prevOuts
}

func state(t *Transaction) string {
	var b bytes.Buffer
	for i, in := range t.Inputs {
		fmt.Fprintf(&b, "in%d[sig=%2dB key=%2dB] ", i, len(in.Signature), len(in.PubKey))
	}
	return strings.TrimSpace(b.String())
}

func main() {
	priv, _ := crypto.HexToECDSA(aliceKey)
	alice, bob := pkh(aliceKey), pkh(bobKey)

	fmt.Println("=== one input: the bug is invisible ===")
	tx, prev := build(alice, bob, 1)
	tx.sign(priv, prev, true)
	fmt.Printf("  after signing: %s\n", state(tx))
	fmt.Printf("  verify: %s\n", result(tx.Verify(prev)))
	fmt.Println("  With a single input the loop runs once, so nothing gets")
	fmt.Println("  clobbered afterwards. Every test you wrote passes.")

	fmt.Println("\n=== two inputs: the signature vanishes ===")
	tx, prev = build(alice, bob, 2)
	tx.sign(priv, prev, true)
	fmt.Printf("  after signing: %s\n", state(tx))
	fmt.Printf("  verify: %s\n", result(tx.Verify(prev)))
	fmt.Println("  Signing input 1 called TrimmedCopyBuggy, which walked the")
	fmt.Println("  SHARED input slice and nil'd input 0's signature and key.")

	fmt.Println("\n=== three inputs: only the last one survives ===")
	tx, prev = build(alice, bob, 3)
	tx.sign(priv, prev, true)
	fmt.Printf("  after signing: %s\n", state(tx))

	fmt.Println("\n=== the deep copy ===")
	tx, prev = build(alice, bob, 3)
	tx.sign(priv, prev, false)
	fmt.Printf("  after signing: %s\n", state(tx))
	fmt.Printf("  verify: %s\n", result(tx.Verify(prev)))

	fmt.Println("\n=== proving the copy does not touch the original ===")
	before := tx.Serialize()
	c := tx.TrimmedCopy()
	c.Inputs[0].PubKey = []byte("scribble")
	c.Inputs[1].Signature = []byte("scribble")
	c.Outputs[0].Value = 999999
	c.Outputs[0].PubKeyHash[0] ^= 0xff
	after := tx.Serialize()
	fmt.Printf("  scribbled all over the copy — original bytes identical: %v\n",
		bytes.Equal(before, after))
	fmt.Printf("  and it still verifies: %s\n", result(tx.Verify(prev)))

	fmt.Println("\n=== the rule ===")
	fmt.Println("  A copy is only as deep as the fields you will write to.")
	fmt.Println("    c := *t                       copies the struct, SHARES the slices")
	fmt.Println("    c.Inputs = slices.Clone(...)  copies the elements, SHARES their []byte")
	fmt.Println("    append([]byte(nil), b...)     copies the bytes")
	fmt.Println()
	fmt.Println("  TrimmedCopy writes to Inputs[i].Signature/PubKey, so the inputs")
	fmt.Println("  must be element-copied. Nothing writes to an output's PubKeyHash")
	fmt.Println("  during signing — but copying it anyway costs 20 bytes and removes")
	fmt.Println("  a whole class of future bug.")
	fmt.Println()
	fmt.Println("  Go's own defence: `go vet` will not catch this, and neither will")
	fmt.Println("  `-race` — it is not a data race, it is one goroutine writing")
	fmt.Println("  where it did not mean to. The test that catches it is a")
	fmt.Println("  MULTI-INPUT sign-then-verify. Write that test first.")
}

func result(err error) string {
	if err == nil {
		return "ok"
	}
	return "REJECTED: " + err.Error()
}
```

**Output:**

```
=== one input: the bug is invisible ===
  after signing: in0[sig=65B key=33B]
  verify: ok
  With a single input the loop runs once, so nothing gets
  clobbered afterwards. Every test you wrote passes.

=== two inputs: the signature vanishes ===
  after signing: in0[sig= 0B key= 0B] in1[sig=65B key=33B]
  verify: REJECTED: input 0: public key does not match the output's lock
  Signing input 1 called TrimmedCopyBuggy, which walked the
  SHARED input slice and nil'd input 0's signature and key.

=== three inputs: only the last one survives ===
  after signing: in0[sig= 0B key= 0B] in1[sig= 0B key= 0B] in2[sig=65B key=33B]

=== the deep copy ===
  after signing: in0[sig=65B key=33B] in1[sig=65B key=33B] in2[sig=65B key=33B]
  verify: ok

=== proving the copy does not touch the original ===
  scribbled all over the copy — original bytes identical: true
  and it still verifies: ok

=== the rule ===
  A copy is only as deep as the fields you will write to.
    c := *t                       copies the struct, SHARES the slices
    c.Inputs = slices.Clone(...)  copies the elements, SHARES their []byte
    append([]byte(nil), b...)     copies the bytes

  TrimmedCopy writes to Inputs[i].Signature/PubKey, so the inputs
  must be element-copied. Nothing writes to an output's PubKeyHash
  during signing — but copying it anyway costs 20 bytes and removes
  a whole class of future bug.

  Go's own defence: `go vet` will not catch this, and neither will
  `-race` — it is not a data race, it is one goroutine writing
  where it did not mean to. The test that catches it is a
  MULTI-INPUT sign-then-verify. Write that test first.
```

---

## 9. Which input failed

`🟡 medium` · *Verification*

A transaction with inputs from three different people is normal — CoinJoin, exchange batching, consolidation. Each input is checked independently, and one failure kills all of it. 'invalid transaction' is a useless log line, so the error carries the index.

**Steps:**

1. Build and sign a three-party transaction.
2. Break each input in turn and confirm the reported index is right.
3. Substitute another person's key and get a different sentinel error.
4. Break two inputs and see only the first reported — a deliberate DoS defence.
5. Use `errors.As` and `errors.Is` on the result, the way a mempool would.
6. Read the check ordering: structural, then hash, then the expensive signature.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Verification, properly: per input, and reporting WHICH input failed and why.
//
// A transaction with three inputs from three different people is normal —
// CoinJoin, exchange batching, a wallet consolidating dust. Each input is
// checked independently, and a single failure invalidates all of it.
//
// "invalid transaction" is a useless log line when you are running a mempool.
// Return a typed error carrying the index and the reason.
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

// ---------------------------------------------------------------- errors

var (
	ErrKeyMismatch = errors.New("public key does not match the output's lock")
	ErrBadSig      = errors.New("signature does not verify")
	ErrNoSig       = errors.New("input is not signed")
	ErrArity       = errors.New("input count does not match the outputs supplied")
)

// InputError says which input failed, so a mempool can log it, a wallet can
// re-sign just that one, and a test can assert on the index.
type InputError struct {
	Index int
	Err   error
}

func (e *InputError) Error() string { return fmt.Sprintf("input %d: %v", e.Index, e.Err) }
func (e *InputError) Unwrap() error { return e.Err }

func (t *Transaction) Verify(prevOuts []TxOutput) error {
	if len(prevOuts) != len(t.Inputs) {
		return ErrArity
	}
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return &InputError{i, ErrNoSig}
		}
		// 1. does this key open that lock?
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return &InputError{i, ErrKeyMismatch}
		}
		// 2. did the holder of that key authorise THIS transaction?
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return &InputError{i, ErrBadSig}
		}
	}
	return nil
}

// SignInput signs exactly one input — which is what a multi-party transaction
// needs, since no single party holds all the keys.
func (t *Transaction) SignInput(i int, priv *ecdsa.PrivateKey, prevOut TxOutput) error {
	h := t.SigHash(i, prevOut.PubKeyHash)
	sig, err := crypto.Sign(h[:], priv)
	if err != nil {
		return err
	}
	t.Inputs[i].Signature = sig
	t.Inputs[i].PubKey = crypto.CompressPubkey(&priv.PublicKey)
	return nil
}

// TEST ONLY — published Hardhat/anvil keys.
var keys = []string{
	"ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80", // alice
	"59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d", // bob
	"5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a", // carol
	"7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6", // mallory
}

var names = []string{"alice", "bob", "carol", "mallory"}

func priv(i int) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(keys[i])
	if err != nil {
		panic(err)
	}
	return k
}

func pkh(i int) []byte { return hash160(crypto.CompressPubkey(&priv(i).PublicKey)) }

// build returns a fresh 3-in, 1-out transaction, fully signed by alice, bob
// and carol, plus the outputs it spends.
func build() (*Transaction, []TxOutput) {
	tx := &Transaction{Outputs: []TxOutput{{Value: 145, PubKeyHash: pkh(1)}}}
	var prevOuts []TxOutput
	for i := 0; i < 3; i++ {
		var id [32]byte
		id[0] = byte(0xa0 + i)
		tx.Inputs = append(tx.Inputs, TxInput{Prev: Outpoint{id, 0}})
		prevOuts = append(prevOuts, TxOutput{Value: 50, PubKeyHash: pkh(i)})
	}
	for i := 0; i < 3; i++ {
		if err := tx.SignInput(i, priv(i), prevOuts[i]); err != nil {
			panic(err)
		}
	}
	return tx, prevOuts
}

func main() {
	tx, prevOuts := build()

	fmt.Println("=== a three-party transaction ===")
	for i, in := range tx.Inputs {
		fmt.Printf("  input %d  %-8s spends %x…:%d for %d\n",
			i, names[i], in.Prev.TxID[:2], in.Prev.Index, prevOuts[i].Value)
	}
	fmt.Printf("  output 0 145 to bob, fee %d\n", 150-145)
	fmt.Printf("\n  verify: %s\n", show(tx.Verify(prevOuts)))

	fmt.Println("\n=== break one input at a time ===")
	cases := []struct {
		what string
		mut  func(*Transaction, []TxOutput)
	}{
		{"input 0: signature bit flipped", func(t *Transaction, _ []TxOutput) { t.Inputs[0].Signature[7] ^= 1 }},
		{"input 1: signature bit flipped", func(t *Transaction, _ []TxOutput) { t.Inputs[1].Signature[7] ^= 1 }},
		{"input 2: signature bit flipped", func(t *Transaction, _ []TxOutput) { t.Inputs[2].Signature[7] ^= 1 }},
		{"input 1: not signed at all", func(t *Transaction, _ []TxOutput) { t.Inputs[1] = TxInput{Prev: t.Inputs[1].Prev} }},
		{"input 2: mallory's key substituted", func(t *Transaction, p []TxOutput) {
			_ = t.SignInput(2, priv(3), p[2])
		}},
		{"input 0: outpoint repointed after signing", func(t *Transaction, _ []TxOutput) { t.Inputs[0].Prev.Index = 7 }},
		{"output value raised after signing", func(t *Transaction, _ []TxOutput) { t.Outputs[0].Value = 150 }},
	}
	for _, c := range cases {
		t2, p2 := build()
		c.mut(t2, p2)
		err := t2.Verify(p2)
		var ie *InputError
		idx := "-"
		if errors.As(err, &ie) {
			idx = fmt.Sprint(ie.Index)
		}
		fmt.Printf("  %-42s failing input: %-3s %s\n", c.what, idx, show(err))
	}

	fmt.Println("\n=== two failures: only the first is reported ===")
	t3, p3 := build()
	t3.Inputs[0].Signature[7] ^= 1
	t3.Inputs[2].Signature[7] ^= 1
	fmt.Printf("  inputs 0 and 2 both broken -> %s\n", show(t3.Verify(p3)))
	fmt.Println("  That is deliberate. Validation is a rejection decision, not a")
	fmt.Println("  report: the first failure settles it, and doing less work on")
	fmt.Println("  invalid data is a DoS defence, not an optimisation.")

	fmt.Println("\n=== the error type earns its keep ===")
	t4, p4 := build()
	t4.Inputs[1].Signature[7] ^= 1
	err := t4.Verify(p4)
	var ie *InputError
	fmt.Printf("  errors.As(&InputError):    %v, index %d\n", errors.As(err, &ie), ie.Index)
	fmt.Printf("  errors.Is(ErrBadSig):      %v\n", errors.Is(err, ErrBadSig))
	fmt.Printf("  errors.Is(ErrKeyMismatch): %v\n", errors.Is(err, ErrKeyMismatch))
	fmt.Println()
	fmt.Println("  A mempool uses the sentinel to decide policy — a bad signature")
	fmt.Println("  means ban the peer, a missing signature means the wallet is")
	fmt.Println("  still assembling — and the index to say which one.")

	fmt.Println("\n=== the ordering that matters ===")
	fmt.Println("  1. cheap structural checks (lengths, counts) — no crypto")
	fmt.Println("  2. hash the key and compare 20 bytes — one HASH160")
	fmt.Println("  3. verify the signature — the expensive one, tens of microseconds")
	fmt.Println()
	fmt.Println("  Attacker-supplied data reaches step 3 only after passing 1 and 2.")
	fmt.Println("  Doing it the other way round hands anyone a cheap way to burn")
	fmt.Println("  your CPU (lesson 13).")
}

func show(err error) string {
	if err == nil {
		return "ok"
	}
	return "REJECTED: " + err.Error()
}
```

**Output:**

```
=== a three-party transaction ===
  input 0  alice    spends a000…:0 for 50
  input 1  bob      spends a100…:0 for 50
  input 2  carol    spends a200…:0 for 50
  output 0 145 to bob, fee 5

  verify: ok

=== break one input at a time ===
  input 0: signature bit flipped             failing input: 0   REJECTED: input 0: signature does not verify
  input 1: signature bit flipped             failing input: 1   REJECTED: input 1: signature does not verify
  input 2: signature bit flipped             failing input: 2   REJECTED: input 2: signature does not verify
  input 1: not signed at all                 failing input: 1   REJECTED: input 1: input is not signed
  input 2: mallory's key substituted         failing input: 2   REJECTED: input 2: public key does not match the output's lock
  input 0: outpoint repointed after signing  failing input: 0   REJECTED: input 0: signature does not verify
  output value raised after signing          failing input: 0   REJECTED: input 0: signature does not verify

=== two failures: only the first is reported ===
  inputs 0 and 2 both broken -> REJECTED: input 0: signature does not verify
  That is deliberate. Validation is a rejection decision, not a
  report: the first failure settles it, and doing less work on
  invalid data is a DoS defence, not an optimisation.

=== the error type earns its keep ===
  errors.As(&InputError):    true, index 1
  errors.Is(ErrBadSig):      true
  errors.Is(ErrKeyMismatch): false

  A mempool uses the sentinel to decide policy — a bad signature
  means ban the peer, a missing signature means the wallet is
  still assembling — and the index to say which one.

=== the ordering that matters ===
  1. cheap structural checks (lengths, counts) — no crypto
  2. hash the key and compare 20 bytes — one HASH160
  3. verify the signature — the expensive one, tens of microseconds

  Attacker-supplied data reaches step 3 only after passing 1 and 2.
  Doing it the other way round hands anyone a cheap way to burn
  your CPU (lesson 13).
```

---

## 10. The check that makes ownership mean anything

`🟡 medium` · *Verification*

Verification has two halves, and only one is cryptography. Skip the second — that the public key hashes to the *output's* lock — and anyone can spend anyone's coins with a perfectly valid signature: their own.

**Steps:**

1. Write both verifiers: signature-only, and signature plus binding.
2. Have mallory sign a spend of alice's coin with mallory's key. The signature is genuine.
3. Watch the sig-only verifier hand over the coin, and the full one refuse.
4. Have mallory substitute alice's public key instead, and watch the other half catch it.
5. Try replaying alice's signature onto mallory's outputs.
6. Read why the key is hashed at all, and what that means for address reuse.

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
// Verification has TWO halves, and only one of them is cryptography.
//
//   1. does this signature verify against the public key in the input?
//   2. does that public key hash to the lock on the output being spent?
//
// Skip (2) and anyone can spend anyone's coins with a perfectly valid
// signature — their own. This example runs the theft both ways.
//
// In Bitcoin the two halves are the two halves of the P2PKH script:
//   OP_DUP OP_HASH160 <pkh> OP_EQUALVERIFY OP_CHECKSIG
//                           ^^^^^^^^^^^^^^ this is check (2)
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

// ------------------------------------------------------- two verifiers

// VerifySigOnly does the cryptography and forgets the binding. Every
// signature it accepts is genuine. It is still completely broken.
func VerifySigOnly(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if len(in.Signature) != 65 || !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

// VerifyFull adds the one comparison that makes ownership mean anything.
func VerifyFull(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: key %x… does not open lock %x…",
				i, hash160(in.PubKey)[:4], prevOuts[i].PubKeyHash[:4])
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if len(in.Signature) != 65 || !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

// TEST ONLY — published Hardhat/anvil keys.
const (
	aliceKey   = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
	bobKey     = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
	malloryKey = "7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6"
)

func key(h string) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(h)
	if err != nil {
		panic(err)
	}
	return k
}

func pub(k *ecdsa.PrivateKey) []byte  { return crypto.CompressPubkey(&k.PublicKey) }
func lock(k *ecdsa.PrivateKey) []byte { return hash160(pub(k)) }

func spend(prev Outpoint, value int64, to []byte) *Transaction {
	return &Transaction{
		Inputs:  []TxInput{{Prev: prev}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
}

func signWith(t *Transaction, i int, k *ecdsa.PrivateKey, prevOut TxOutput) {
	h := t.SigHash(i, prevOut.PubKeyHash)
	sig, err := crypto.Sign(h[:], k)
	if err != nil {
		panic(err)
	}
	t.Inputs[i].Signature = sig
	t.Inputs[i].PubKey = pub(k)
}

func report(label string, sigOnly, full error) {
	fmt.Printf("  %-46s sig-only: %-9s full: %s\n", label, verdict(sigOnly), verdict(full))
}

func verdict(err error) string {
	if err == nil {
		return "accepted"
	}
	return "REJECTED"
}

func main() {
	alice, bob, mallory := key(aliceKey), key(bobKey), key(malloryKey)

	// The coin: 50 units, locked to alice.
	var coin Outpoint
	coin.TxID[0] = 0xa1
	funded := []TxOutput{{Value: 50, PubKeyHash: lock(alice)}}

	fmt.Println("=== the coin ===")
	fmt.Printf("  outpoint %x…:%d  value %d\n", coin.TxID[:4], coin.Index, funded[0].Value)
	fmt.Printf("  locked to %x  (alice)\n", funded[0].PubKeyHash)
	fmt.Printf("  mallory's lock would be %x\n", lock(mallory))

	fmt.Println("\n=== 1. alice spends her own coin ===")
	honest := spend(coin, 49, lock(bob))
	signWith(honest, 0, alice, funded[0])
	report("alice's key, alice's coin", VerifySigOnly(honest, funded), VerifyFull(honest, funded))

	fmt.Println("\n=== 2. mallory spends alice's coin, signing with her own key ===")
	theft := spend(coin, 49, lock(mallory))
	signWith(theft, 0, mallory, funded[0])
	fmt.Printf("  mallory's signature is genuine and verifies: %v\n",
		crypto.VerifySignature(theft.Inputs[0].PubKey,
			sigHashOf(theft, 0, funded[0]), theft.Inputs[0].Signature[:64]))
	report("mallory's key, alice's coin", VerifySigOnly(theft, funded), VerifyFull(theft, funded))
	fmt.Println("  The sig-only verifier hands mallory 50 units. Nothing about the")
	fmt.Println("  signature is forged — it just belongs to the wrong person.")

	fmt.Println("\n=== 3. so mallory claims alice's public key instead ===")
	swap := spend(coin, 49, lock(mallory))
	signWith(swap, 0, mallory, funded[0])
	swap.Inputs[0].PubKey = pub(alice) // the lock now matches...
	report("alice's pubkey, mallory's signature", VerifySigOnly(swap, funded), VerifyFull(swap, funded))
	fmt.Println("  ...and now the signature does not. To pass BOTH checks she needs")
	fmt.Println("  a key that hashes to alice's lock AND signs — i.e. alice's key.")

	fmt.Println("\n=== 4. and she cannot replay alice's signature ===")
	replay := spend(coin, 49, lock(mallory)) // same coin, different destination
	replay.Inputs[0].Signature = honest.Inputs[0].Signature
	replay.Inputs[0].PubKey = honest.Inputs[0].PubKey
	report("alice's signature, mallory's outputs", VerifySigOnly(replay, funded), VerifyFull(replay, funded))
	fmt.Println("  The sighash covers the outputs (example 6), so alice's signature")
	fmt.Println("  is bound to alice's payment and nothing else.")

	fmt.Println("\n=== why hash the key at all ===")
	fmt.Printf("  pubkey    %x  (33 bytes)\n", pub(alice))
	fmt.Printf("  HASH160   %x  (20 bytes)\n", lock(alice))
	fmt.Println()
	fmt.Println("  - shorter: 20 bytes fits an address a human can copy (lesson 07)")
	fmt.Println("  - the public key stays SECRET until the coin is first spent, so")
	fmt.Println("    an unspent output is protected by a hash preimage as well as by")
	fmt.Println("    the discrete log. That is the standing argument against address")
	fmt.Println("    REUSE: spend once and the key is on-chain forever.")
	fmt.Println()
	fmt.Println("  Ethereum makes the same trade differently: it recovers the key")
	fmt.Println("  from the signature (lesson 06) and compares the derived ADDRESS.")
	fmt.Println("  Same two checks, folded into one operation.")
}

func sigHashOf(t *Transaction, i int, prevOut TxOutput) []byte {
	h := t.SigHash(i, prevOut.PubKeyHash)
	return h[:]
}
```

**Output:**

```
=== the coin ===
  outpoint a1000000…:0  value 50
  locked to a55476015c13afb8afb92160329a8cde976f1f2e  (alice)
  mallory's lock would be 245289ae1d4ab1b27e13e44e77a3ce0ebb2f445c

=== 1. alice spends her own coin ===
  alice's key, alice's coin                      sig-only: accepted  full: accepted

=== 2. mallory spends alice's coin, signing with her own key ===
  mallory's signature is genuine and verifies: true
  mallory's key, alice's coin                    sig-only: accepted  full: REJECTED
  The sig-only verifier hands mallory 50 units. Nothing about the
  signature is forged — it just belongs to the wrong person.

=== 3. so mallory claims alice's public key instead ===
  alice's pubkey, mallory's signature            sig-only: REJECTED  full: REJECTED
  ...and now the signature does not. To pass BOTH checks she needs
  a key that hashes to alice's lock AND signs — i.e. alice's key.

=== 4. and she cannot replay alice's signature ===
  alice's signature, mallory's outputs           sig-only: REJECTED  full: REJECTED
  The sighash covers the outputs (example 6), so alice's signature
  is bound to alice's payment and nothing else.

=== why hash the key at all ===
  pubkey    038318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75  (33 bytes)
  HASH160   a55476015c13afb8afb92160329a8cde976f1f2e  (20 bytes)

  - shorter: 20 bytes fits an address a human can copy (lesson 07)
  - the public key stays SECRET until the coin is first spent, so
    an unspent output is protected by a hash preimage as well as by
    the discrete log. That is the standing argument against address
    REUSE: spend once and the key is on-chain forever.

  Ethereum makes the same trade differently: it recovers the key
  from the signature (lesson 06) and compares the derived ADDRESS.
  Same two checks, folded into one operation.
```

---

## 11. Conservation of value

`🟡 medium` · *Value*

Three rules: inputs cover outputs, the difference is the fee, and the coinbase claims at most subsidy plus fees. The third is what pins the first two down — without it a miner could mint anything and the others would still hold.

**Steps:**

1. Run six transactions through the fee rule, including a negative output and an out-of-range one.
2. Notice again that forgetting the change output is a 20 BTC tip, not an error.
3. Check five coinbase claims against subsidy + fees, one satoshi apart.
4. Confirm that claiming LESS is legal — and destroys the difference.
5. Total a whole block's fees and check the coinbase against them.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"

	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Conservation of value. Three rules, and a chain is only sound if all three
// hold on every block:
//
//   1. an ordinary transaction: sum(inputs) >= sum(outputs)
//   2. the difference is the FEE, and the miner claims it
//   3. the coinbase claims at most subsidy(height) + sum(fees)
//
// Rule 3 is the one that ties the other two down. Without it a miner could
// mint whatever it liked and rules 1 and 2 would still hold.
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

// -------------------------------------------------------------- the rules

const (
	Coin          = int64(100_000_000)
	HalvingPeriod = int64(210_000)
	MaxMoney      = 21_000_000 * Coin // the sanity bound every value is checked against
)

var (
	ErrNegative   = errors.New("negative output value")
	ErrTooLarge   = errors.New("value out of range")
	ErrOverflow   = errors.New("value sum overflows")
	ErrNotCovered = errors.New("outputs exceed inputs")
	ErrOverClaim  = errors.New("coinbase claims more than subsidy plus fees")
)

func Subsidy(height int64) int64 {
	h := height / HalvingPeriod
	if h >= 64 {
		return 0
	}
	return 50 * Coin >> uint(h)
}

// sumOutputs is where the range check lives. Checking each value against
// [0, MaxMoney] BEFORE adding is what makes the sum safe — example 17 shows
// what happens to a chain that only checks the total.
func sumOutputs(t *Transaction) (int64, error) {
	var total int64
	for i, o := range t.Outputs {
		switch {
		case o.Value < 0:
			return 0, fmt.Errorf("output %d: %w (%d)", i, ErrNegative, o.Value)
		case o.Value > MaxMoney:
			return 0, fmt.Errorf("output %d: %w (%d)", i, ErrTooLarge, o.Value)
		}
		total += o.Value
		if total > MaxMoney {
			return 0, fmt.Errorf("output %d: %w", i, ErrOverflow)
		}
	}
	return total, nil
}

// Fee returns in - out for an ordinary transaction, and the error that makes
// the transaction invalid if it does not conserve value.
func Fee(t *Transaction, prevOuts []TxOutput) (int64, error) {
	var in int64
	for _, p := range prevOuts {
		in += p.Value
	}
	out, err := sumOutputs(t)
	if err != nil {
		return 0, err
	}
	if out > in {
		return 0, fmt.Errorf("%w: in %s, out %s", ErrNotCovered, btc(in), btc(out))
	}
	return in - out, nil
}

// CheckCoinbase enforces rule 3.
func CheckCoinbase(cb *Transaction, height, fees int64) error {
	claimed, err := sumOutputs(cb)
	if err != nil {
		return err
	}
	allowed := Subsidy(height) + fees
	if claimed > allowed {
		return fmt.Errorf("%w: claimed %s, allowed %s", ErrOverClaim, btc(claimed), btc(allowed))
	}
	return nil
}

func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func out(v int64) TxOutput { return TxOutput{Value: v, PubKeyHash: make([]byte, 20)} }
func tx(vals ...int64) *Transaction {
	t := &Transaction{Inputs: []TxInput{{Signature: make([]byte, 65), PubKey: make([]byte, 33)}}}
	for _, v := range vals {
		t.Outputs = append(t.Outputs, out(v))
	}
	return t
}

func main() {
	fmt.Println("=== rule 1 & 2: the fee is what is left over ===")
	prev := []TxOutput{out(50 * Coin)}
	for _, c := range []struct {
		what string
		t    *Transaction
	}{
		{"pay 30, change 19.9", tx(30*Coin, 1_990_000_000)},
		{"pay 30, change 20 (no fee)", tx(30*Coin, 20*Coin)},
		{"pay 30, no change output", tx(30 * Coin)},
		{"pay 30, change 25 (over-spend)", tx(30*Coin, 25*Coin)},
		{"pay -1 (negative output)", tx(-1 * Coin)},
		{"pay 22,000,000 (out of range)", tx(22_000_000 * Coin)},
	} {
		fee, err := Fee(c.t, prev)
		if err != nil {
			fmt.Printf("  %-32s REJECTED: %v\n", c.what, err)
			continue
		}
		fmt.Printf("  %-32s fee %s\n", c.what, btc(fee))
	}
	fmt.Println("\n  Note the third row. Forgetting the change output is not an")
	fmt.Println("  error — it is a 20 BTC tip (example 2).")

	fmt.Println("\n=== rule 3: what the coinbase may claim ===")
	height := int64(700_000)
	fees := int64(37_500_000) // 0.375 BTC of fees collected from this block
	fmt.Printf("  height %d -> subsidy %s\n", height, btc(Subsidy(height)))
	fmt.Printf("  fees collected        %s\n", btc(fees))
	fmt.Printf("  therefore allowed     %s\n", btc(Subsidy(height)+fees))
	fmt.Println()
	for _, claim := range []int64{
		Subsidy(height) + fees,     // exactly right
		Subsidy(height),            // legal: forgets the fees, burns them
		Subsidy(height) + fees + 1, // one satoshi too many
		Subsidy(height) * 2,        // greedy
		0,                          // legal, and block 501726 really did this
	} {
		err := CheckCoinbase(tx(claim), height, fees)
		status := "ok"
		if err != nil {
			status = "REJECTED: " + err.Error()
		}
		fmt.Printf("  claims %-14s %s\n", btc(claim), status)
	}
	fmt.Println("\n  Claiming LESS is always legal, and the difference is destroyed —")
	fmt.Println("  there is no account for it to go to. Miners have done this by")
	fmt.Println("  accident more than once (example 5).")

	fmt.Println("\n=== a whole block ===")
	type entry struct {
		name string
		t    *Transaction
		prev []TxOutput
	}
	body := []entry{
		{"tx A", tx(30*Coin, 19*Coin), []TxOutput{out(50 * Coin)}},
		{"tx B", tx(5*Coin, 4*Coin), []TxOutput{out(10 * Coin)}},
		{"tx C", tx(1 * Coin), []TxOutput{out(1*Coin + 25_000)}},
	}
	var totalFees int64
	for _, e := range body {
		f, err := Fee(e.t, e.prev)
		if err != nil {
			fmt.Printf("  %s REJECTED: %v\n", e.name, err)
			return
		}
		totalFees += f
		fmt.Printf("  %-5s in %-14s out %-14s fee %s\n", e.name,
			btc(e.prev[0].Value), btc(mustSum(e.t)), btc(f))
	}
	fmt.Printf("  %-5s %-17s %-18s fee %s  <- total\n", "", "", "", btc(totalFees))
	fmt.Printf("  %-5s subsidy %s + fees %s = coinbase may claim %s\n",
		"cb", btc(Subsidy(height)), btc(totalFees), btc(Subsidy(height)+totalFees))
	fmt.Printf("\n  block valid: %v\n", CheckCoinbase(tx(Subsidy(height)+totalFees), height, totalFees) == nil)

	fmt.Println("\n=== the three habits ===")
	fmt.Println("  1. Range-check every VALUE before summing, not just the total.")
	fmt.Println("     `0 <= v <= MaxMoney` on each output makes overflow of the sum")
	fmt.Println("     impossible by construction (example 17 shows the alternative).")
	fmt.Println("  2. Compute the fee as in - out. Never let a transaction state")
	fmt.Println("     its own fee: there is nothing to check it against.")
	fmt.Println("  3. Enforce the coinbase bound in the BLOCK checks, not the")
	fmt.Println("     transaction checks — it needs every other fee in the block.")
}

func mustSum(t *Transaction) int64 {
	s, err := sumOutputs(t)
	if err != nil {
		panic(err)
	}
	return s
}
```

**Output:**

```
=== rule 1 & 2: the fee is what is left over ===
  pay 30, change 19.9              fee 0.10000000
  pay 30, change 20 (no fee)       fee 0.00000000
  pay 30, no change output         fee 20.00000000
  pay 30, change 25 (over-spend)   REJECTED: outputs exceed inputs: in 50.00000000, out 55.00000000
  pay -1 (negative output)         REJECTED: output 0: negative output value (-100000000)
  pay 22,000,000 (out of range)    REJECTED: output 0: value out of range (2200000000000000)

  Note the third row. Forgetting the change output is not an
  error — it is a 20 BTC tip (example 2).

=== rule 3: what the coinbase may claim ===
  height 700000 -> subsidy 6.25000000
  fees collected        0.37500000
  therefore allowed     6.62500000

  claims 6.62500000     ok
  claims 6.25000000     ok
  claims 6.62500001     REJECTED: coinbase claims more than subsidy plus fees: claimed 6.62500001, allowed 6.62500000
  claims 12.50000000    REJECTED: coinbase claims more than subsidy plus fees: claimed 12.50000000, allowed 6.62500000
  claims 0.00000000     ok

  Claiming LESS is always legal, and the difference is destroyed —
  there is no account for it to go to. Miners have done this by
  accident more than once (example 5).

=== a whole block ===
  tx A  in 50.00000000    out 49.00000000    fee 1.00000000
  tx B  in 10.00000000    out 9.00000000     fee 1.00000000
  tx C  in 1.00025000     out 1.00000000     fee 0.00025000
                                             fee 2.00025000  <- total
  cb    subsidy 6.25000000 + fees 2.00025000 = coinbase may claim 8.25025000

  block valid: true

=== the three habits ===
  1. Range-check every VALUE before summing, not just the total.
     `0 <= v <= MaxMoney` on each output makes overflow of the sum
     impossible by construction (example 17 shows the alternative).
  2. Compute the fee as in - out. Never let a transaction state
     its own fee: there is nothing to check it against.
  3. Enforce the coinbase bound in the BLOCK checks, not the
     transaction checks — it needs every other fee in the block.
```

---

## 12. Building the UTXO set

`🟡 medium` · *The UTXO set*

The blocks are the history; the UTXO set is the state you actually need. Replay a three-block chain into a `map[Outpoint]TxOutput`, then maintain it as one delta per block: deletes, then inserts.

**Steps:**

1. Apply three blocks and print the set after each.
2. Compute everyone's balance, and see that it is a scan of the whole set.
3. Check conservation: 200 units of subsidy issued, 200 unspent, 2 of fees merely recycled.
4. Revert the last block and watch the set go backwards — what every reorg does.
5. Read why the spent outputs have to be stored separately for that to be possible.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"sort"

	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// The UTXO set is the real state of a UTXO chain.
//
// The blocks are the HISTORY; the set is what you actually need to validate
// the next transaction. Build it by replaying the chain once, then maintain
// it incrementally — one delta per block, deletes then inserts.
//
// (Coinbase maturity is deliberately not enforced here; example 16 adds it.)
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

// ------------------------------------------------------------- the UTXO set

// The whole thing. In lesson 12 this becomes a bucket in a disk KV store; the
// shape does not change.
type UTXOSet map[Outpoint]TxOutput

// Delta is what one block does to the set: outpoints to remove, outputs to
// add. Building it before touching the set is what makes application atomic —
// a block that turns out to be invalid halfway through must leave nothing
// behind (the validate-then-mutate rule from lesson 08).
type Delta struct {
	Spend  []Outpoint
	Create map[Outpoint]TxOutput
}

// Apply inserts FIRST, then deletes. An output that a block both creates and
// spends appears in Create and in Spend; deleting first would put it back and
// mint money from nothing.
func (s UTXOSet) Apply(d Delta) {
	for op, o := range d.Create {
		s[op] = o
	}
	for _, op := range d.Spend {
		delete(s, op)
	}
}

// Revert undoes a block — needed on every reorg (lesson 14). Note it needs
// the outputs that were spent, which the set no longer holds: that is why
// nodes keep an "undo file" per block.
func (s UTXOSet) Revert(d Delta, spent map[Outpoint]TxOutput) {
	for op := range d.Create {
		delete(s, op)
	}
	for op, o := range spent {
		s[op] = o
	}
}

func (s UTXOSet) Balance(pkh []byte) int64 {
	var n int64
	for _, o := range s {
		if bytes.Equal(o.PubKeyHash, pkh) {
			n += o.Value
		}
	}
	return n
}

func (s UTXOSet) Coins(pkh []byte) []Outpoint {
	var ops []Outpoint
	for op, o := range s {
		if bytes.Equal(o.PubKeyHash, pkh) {
			ops = append(ops, op)
		}
	}
	sortOutpoints(ops)
	return ops
}

func sortOutpoints(ops []Outpoint) {
	sort.Slice(ops, func(i, j int) bool {
		if c := bytes.Compare(ops[i].TxID[:], ops[j].TxID[:]); c != 0 {
			return c < 0
		}
		return ops[i].Index < ops[j].Index
	})
}

// ----------------------------------------------------------------- a chain

type Block struct {
	Height int64
	Txs    []*Transaction
}

// BlockDelta computes the delta without applying it.
func BlockDelta(b *Block) Delta {
	d := Delta{Create: map[Outpoint]TxOutput{}}
	for _, t := range b.Txs {
		if !t.IsCoinbase() {
			for _, in := range t.Inputs {
				d.Spend = append(d.Spend, in.Prev)
			}
		}
		id := t.TxID()
		for i, o := range t.Outputs {
			d.Create[Outpoint{id, uint32(i)}] = o
		}
	}
	return d
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

// ------------------------------------------------------------------ actors

var people = []string{"alice", "bob", "carol", "miner"}

func who(pkh []byte) string {
	for _, n := range people {
		if bytes.Equal(pkh, addr(n)) {
			return n
		}
	}
	return "?"
}

// addr is a stand-in for HASH160(pubkey) — the keys do not matter here, only
// that each person has a distinct 20-byte lock.
func addr(name string) []byte { return hash160([]byte(name)) }

func coinbase(height, value int64, to string) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height)) // BIP-34: unique per height
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: addr(to)}},
	}
}

func spend(ops []Outpoint, outs ...TxOutput) *Transaction {
	t := &Transaction{Outputs: outs}
	for _, op := range ops {
		// Signatures are checked in examples 7-10; here we only track value.
		t.Inputs = append(t.Inputs, TxInput{Prev: op, Signature: make([]byte, 65), PubKey: make([]byte, 33)})
	}
	return t
}

func pay(v int64, to string) TxOutput { return TxOutput{Value: v, PubKeyHash: addr(to)} }

func (s UTXOSet) dump(label string) {
	ops := make([]Outpoint, 0, len(s))
	for op := range s {
		ops = append(ops, op)
	}
	sortOutpoints(ops)
	fmt.Printf("  %s (%d entries)\n", label, len(s))
	for _, op := range ops {
		fmt.Printf("    %x…:%d  %-6s %4d\n", op.TxID[:6], op.Index, who(s[op].PubKeyHash), s[op].Value)
	}
}

func main() {
	set := UTXOSet{}

	// --- block 1: alice is funded ------------------------------------------
	b1 := &Block{Height: 1, Txs: []*Transaction{coinbase(1, 100, "alice")}}

	// --- block 2: alice pays bob and carol, keeps change --------------------
	aliceCoin := Outpoint{b1.Txs[0].TxID(), 0}
	txA := spend([]Outpoint{aliceCoin}, pay(30, "bob"), pay(25, "carol"), pay(44, "alice"))
	b2 := &Block{Height: 2, Txs: []*Transaction{coinbase(2, 50+1, "miner"), txA}}

	// --- block 3: bob pays carol; carol pays alice --------------------------
	txB := spend([]Outpoint{{txA.TxID(), 0}}, pay(12, "carol"), pay(17, "bob"))
	txC := spend([]Outpoint{{txA.TxID(), 1}}, pay(25, "alice"))
	b3 := &Block{Height: 3, Txs: []*Transaction{coinbase(3, 50+1, "miner"), txB, txC}}

	fmt.Println("=== replaying the chain ===")
	var spentHistory []map[Outpoint]TxOutput
	for _, b := range []*Block{b1, b2, b3} {
		d := BlockDelta(b)

		// Remember what we are about to delete, so the block can be reverted.
		// Only outputs that were already in the set: one created and spent
		// inside this same block is undone by dropping it, not by restoring it.
		spent := map[Outpoint]TxOutput{}
		for _, op := range d.Spend {
			if o, ok := set[op]; ok {
				spent[op] = o
			}
		}
		spentHistory = append(spentHistory, spent)

		before := len(set)
		set.Apply(d)
		fmt.Printf("\n  block %d: txs %d, -%d spent, +%d created  (set: %d -> %d)\n",
			b.Height, len(b.Txs), len(d.Spend), len(d.Create), before, len(set))
		set.dump(fmt.Sprintf("set after block %d", b.Height))
	}

	fmt.Println("\n=== balances ===")
	fmt.Printf("  %-8s %-8s %s\n", "who", "balance", "coins")
	var total int64
	for _, n := range people {
		bal := set.Balance(addr(n))
		total += bal
		fmt.Printf("  %-8s %-8d %d\n", n, bal, len(set.Coins(addr(n))))
	}
	fmt.Printf("  %-8s %-8d  <- every unit ever ISSUED, still unspent\n", "total", total)
	fmt.Println("  Subsidies were 100 + 50 + 50 = 200. The coinbases claimed 202,")
	fmt.Println("  because 2 of that was fees — value that already existed and was")
	fmt.Println("  recycled, not created. Conservation holds (example 11).")

	fmt.Println("\n  Note where a balance comes from: a SCAN. The protocol has no")
	fmt.Println("  balance field, so a node that indexes only by outpoint cannot")
	fmt.Println("  answer 'how much does alice have?' at all. Bitcoin Core cannot;")
	fmt.Println("  block explorers keep a separate address index to do it.")

	fmt.Println("\n=== reverting block 3 ===")
	d3 := BlockDelta(b3)
	set.Revert(d3, spentHistory[2])
	fmt.Printf("  set is back to %d entries; alice %d, bob %d, carol %d\n",
		len(set), set.Balance(addr("alice")), set.Balance(addr("bob")), set.Balance(addr("carol")))
	fmt.Println("  A reorg is exactly this, per block, in reverse order — which is")
	fmt.Println("  why the spent outputs have to be stored somewhere (lesson 14).")
	set.Apply(d3)

	fmt.Println("\n=== what this set costs in the real world ===")
	fmt.Println("  Bitcoin's UTXO set is well over a hundred million entries and")
	fmt.Println("  several GB. It grows whenever transactions create more outputs")
	fmt.Println("  than they consume, which is the DUST problem: a 1-satoshi output")
	fmt.Println("  costs the network permanent state forever, and costs its owner")
	fmt.Println("  more in fees to spend than it is worth (example 15).")
	fmt.Println()
	fmt.Println("  That asymmetry is why exchanges CONSOLIDATE — spending hundreds")
	fmt.Println("  of small outputs into one when fees are low, to shrink both the")
	fmt.Println("  global set and their own future fees.")
}
```

**Output:**

```
=== replaying the chain ===

  block 1: txs 1, -0 spent, +1 created  (set: 0 -> 1)
  set after block 1 (1 entries)
    66073a83d6ea…:0  alice   100

  block 2: txs 2, -1 spent, +4 created  (set: 1 -> 4)
  set after block 2 (4 entries)
    cc7ddf8c3cb2…:0  bob      30
    cc7ddf8c3cb2…:1  carol    25
    cc7ddf8c3cb2…:2  alice    44
    e331a460b7b1…:0  miner    51

  block 3: txs 3, -2 spent, +4 created  (set: 4 -> 6)
  set after block 3 (6 entries)
    23c0ede3e364…:0  alice    25
    a26ffb27f4d1…:0  carol    12
    a26ffb27f4d1…:1  bob      17
    cb2da4473e20…:0  miner    51
    cc7ddf8c3cb2…:2  alice    44
    e331a460b7b1…:0  miner    51

=== balances ===
  who      balance  coins
  alice    69       2
  bob      17       1
  carol    12       1
  miner    102      2
  total    200       <- every unit ever ISSUED, still unspent
  Subsidies were 100 + 50 + 50 = 200. The coinbases claimed 202,
  because 2 of that was fees — value that already existed and was
  recycled, not created. Conservation holds (example 11).

  Note where a balance comes from: a SCAN. The protocol has no
  balance field, so a node that indexes only by outpoint cannot
  answer 'how much does alice have?' at all. Bitcoin Core cannot;
  block explorers keep a separate address index to do it.

=== reverting block 3 ===
  set is back to 4 entries; alice 44, bob 30, carol 25
  A reorg is exactly this, per block, in reverse order — which is
  why the spent outputs have to be stored somewhere (lesson 14).

=== what this set costs in the real world ===
  Bitcoin's UTXO set is well over a hundred million entries and
  several GB. It grows whenever transactions create more outputs
  than they consume, which is the DUST problem: a 1-satoshi output
  costs the network permanent state forever, and costs its owner
  more in fees to spend than it is worth (example 15).

  That asymmetry is why exchanges CONSOLIDATE — spending hundreds
  of small outputs into one when fees are low, to shrink both the
  global set and their own future fees.
```

---

## 13. Double spends, and applying a block atomically

`🟡 medium` · *The UTXO set*

A signature proves you *may* spend a coin, not that you have not already. One rule at three scopes: once per transaction, once per block, and still in the set. The middle one is easy to forget, because each transaction is individually valid.

**Steps:**

1. Accept a block where one transaction spends another's output — a legal chain.
2. Reverse those two transactions and watch it fail: parents must come first.
3. Reject the same outpoint spent by two transactions, and by one transaction twice.
4. Apply a block and see the cross-block case need no extra machinery.
5. Run a half-invalid block through an apply-as-you-go validator and watch the set corrupt.
6. Run it through validate-then-mutate and watch the set survive untouched.

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
// A signature proves you MAY spend a coin. It does not prove you have not
// already spent it. Preventing that is the entire job of the chain, and it
// comes down to one rule applied at three scopes:
//
//   - within one transaction : an outpoint may appear once
//   - within one block       : an outpoint may be spent once
//   - across the chain       : an outpoint must still be in the UTXO set
//
// The middle one is easy to forget, because each transaction is individually
// valid. This example writes the block validator that catches it — and does
// it atomically, so a block rejected halfway leaves nothing behind.
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

type UTXOSet map[Outpoint]TxOutput

type Delta struct {
	Spend  []Outpoint
	Create map[Outpoint]TxOutput
}

// Apply inserts FIRST, then deletes. An output that a block both creates and
// spends appears in Create and in Spend; deleting first would put it back and
// mint money from nothing.
func (s UTXOSet) Apply(d Delta) {
	for op, o := range d.Create {
		s[op] = o
	}
	for _, op := range d.Spend {
		delete(s, op)
	}
}

type Block struct {
	Height int64
	Txs    []*Transaction
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

var (
	ErrNoCoinbase    = errors.New("block does not start with a coinbase")
	ErrExtraCoinbase = errors.New("more than one coinbase")
	ErrMissingInput  = errors.New("input spends an output that does not exist")
	ErrDoubleSpend   = errors.New("outpoint spent twice")
	ErrNotCovered    = errors.New("outputs exceed inputs")
)

// ValidateBlock returns the delta the block WOULD apply, or an error. It never
// touches the set — that is the whole point. The caller applies the delta only
// if it comes back clean.
func ValidateBlock(set UTXOSet, b *Block) (Delta, int64, error) {
	if len(b.Txs) == 0 || !b.Txs[0].IsCoinbase() {
		return Delta{}, 0, ErrNoCoinbase
	}

	// created holds outputs made EARLIER IN THIS BLOCK. A transaction may
	// spend one — that is how a chain of dependent transactions gets into a
	// single block — but only if its parent appears first.
	created := map[Outpoint]TxOutput{}
	spent := map[Outpoint]int{} // outpoint -> index of the tx that spent it
	d := Delta{Create: map[Outpoint]TxOutput{}}
	var fees int64

	for i, t := range b.Txs {
		if t.IsCoinbase() {
			if i != 0 {
				return Delta{}, 0, fmt.Errorf("tx %d: %w", i, ErrExtraCoinbase)
			}
		} else {
			var in int64
			for _, input := range t.Inputs {
				op := input.Prev

				// (a) has something in this block already spent it?
				if j, dup := spent[op]; dup {
					by := fmt.Sprintf("already spent by tx %d", j)
					if j == i {
						by = "named twice by this same transaction"
					}
					return Delta{}, 0, fmt.Errorf("tx %d: %w: %x…:%d, %s",
						i, ErrDoubleSpend, op.TxID[:6], op.Index, by)
				}
				// (b) does it exist — in the set, or created earlier here?
				prev, ok := set[op]
				if !ok {
					if prev, ok = created[op]; !ok {
						return Delta{}, 0, fmt.Errorf("tx %d: %w: %x…:%d",
							i, ErrMissingInput, op.TxID[:6], op.Index)
					}
				}
				spent[op] = i
				in += prev.Value
				d.Spend = append(d.Spend, op)
			}
			var out int64
			for _, o := range t.Outputs {
				out += o.Value
			}
			if out > in {
				return Delta{}, 0, fmt.Errorf("tx %d: %w: in %d, out %d", i, ErrNotCovered, in, out)
			}
			fees += in - out
		}

		id := t.TxID()
		for k, o := range t.Outputs {
			op := Outpoint{id, uint32(k)}
			created[op] = o
			d.Create[op] = o
		}
	}
	return d, fees, nil
}

// ApplyAsYouGo is the tempting version: validate and mutate in one pass. It is
// wrong, and the failure is silent — the set is left half-updated.
func ApplyAsYouGo(set UTXOSet, b *Block) error {
	for i, t := range b.Txs {
		if !t.IsCoinbase() {
			for _, input := range t.Inputs {
				if _, ok := set[input.Prev]; !ok {
					return fmt.Errorf("tx %d: %w", i, ErrMissingInput)
				}
				delete(set, input.Prev) // <- mutation before the block is known good
			}
		}
		id := t.TxID()
		for k, o := range t.Outputs {
			set[Outpoint{id, uint32(k)}] = o
		}
	}
	return nil
}

// -------------------------------------------------------------- scaffolding

func addr(name string) []byte { return hash160([]byte(name)) }

func coinbase(height, value int64, to string) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height))
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: addr(to)}},
	}
}

func spend(ops []Outpoint, outs ...TxOutput) *Transaction {
	t := &Transaction{Outputs: outs}
	for _, op := range ops {
		t.Inputs = append(t.Inputs, TxInput{Prev: op, Signature: make([]byte, 65), PubKey: make([]byte, 33)})
	}
	return t
}

func pay(v int64, to string) TxOutput { return TxOutput{Value: v, PubKeyHash: addr(to)} }

func fingerprint(s UTXOSet) string {
	ops := make([]Outpoint, 0, len(s))
	for op := range s {
		ops = append(ops, op)
	}
	sort.Slice(ops, func(i, j int) bool {
		if c := bytes.Compare(ops[i].TxID[:], ops[j].TxID[:]); c != 0 {
			return c < 0
		}
		return ops[i].Index < ops[j].Index
	})
	h := sha256.New()
	for _, op := range ops {
		h.Write(op.TxID[:])
		binary.Write(h, binary.BigEndian, op.Index)
		binary.Write(h, binary.BigEndian, s[op].Value)
	}
	return fmt.Sprintf("%d entries, %x", len(s), h.Sum(nil)[:6])
}

func try(set UTXOSet, label string, b *Block) {
	_, fees, err := ValidateBlock(set, b)
	if err != nil {
		fmt.Printf("  %-40s REJECTED: %v\n", label, err)
		return
	}
	fmt.Printf("  %-40s ok, fees %d\n", label, fees)
}

func main() {
	// Two coins, both alice's.
	var c1, c2 Outpoint
	c1.TxID[0], c2.TxID[0] = 0xc1, 0xc2
	set := UTXOSet{
		c1: pay(50, "alice"),
		c2: pay(20, "alice"),
	}
	fmt.Printf("=== starting set ===\n  %s\n", fingerprint(set))
	fmt.Println("  c100…:0 alice 50")
	fmt.Println("  c200…:0 alice 20")

	fmt.Println("\n=== blocks that are fine ===")
	// A simple payment.
	txA := spend([]Outpoint{c1}, pay(30, "bob"), pay(19, "alice"))
	try(set, "one payment", &Block{1, []*Transaction{coinbase(1, 51, "miner"), txA}})

	// Two payments from two different coins.
	txB := spend([]Outpoint{c2}, pay(19, "carol"))
	try(set, "two payments, different coins", &Block{1, []*Transaction{coinbase(1, 52, "miner"), txA, txB}})

	// A CHAIN: txC spends an output txA creates in the same block.
	txC := spend([]Outpoint{{txA.TxID(), 0}}, pay(29, "carol"))
	try(set, "chained: txC spends txA's output", &Block{1, []*Transaction{coinbase(1, 52, "miner"), txA, txC}})

	fmt.Println("\n=== ordering matters ===")
	try(set, "the same two, txC before txA", &Block{1, []*Transaction{coinbase(1, 52, "miner"), txC, txA}})
	fmt.Println("  Bitcoin requires a parent to appear before its child in the")
	fmt.Println("  block. That keeps validation a single forward pass — no")
	fmt.Println("  topological sort, no second pass, no cycles to worry about.")

	fmt.Println("\n=== the double spends ===")
	// Same outpoint, two transactions, one block.
	evil1 := spend([]Outpoint{c1}, pay(49, "bob"))
	evil2 := spend([]Outpoint{c1}, pay(49, "mallory"))
	try(set, "same outpoint in two txs", &Block{1, []*Transaction{coinbase(1, 52, "miner"), evil1, evil2}})

	// Same outpoint twice inside ONE transaction — the version that slips
	// past validators that only compare transactions with each other.
	evil3 := spend([]Outpoint{c1, c1}, pay(99, "mallory"))
	try(set, "same outpoint twice in one tx", &Block{1, []*Transaction{coinbase(1, 52, "miner"), evil3}})

	// Spending something that is not in the set at all.
	var ghost Outpoint
	ghost.TxID[0] = 0xff
	try(set, "spending an outpoint that never existed",
		&Block{1, []*Transaction{coinbase(1, 51, "miner"), spend([]Outpoint{ghost}, pay(1, "mallory"))}})

	fmt.Println("\n=== spending across blocks ===")
	d, _, _ := ValidateBlock(set, &Block{1, []*Transaction{coinbase(1, 51, "miner"), txA}})
	set.Apply(d)
	fmt.Printf("  block 1 applied      -> %s\n", fingerprint(set))
	try(set, "block 2 spends c1 again", &Block{2, []*Transaction{coinbase(2, 50, "miner"), evil2}})
	fmt.Println("  Once applied, the outpoint is simply gone from the set, so the")
	fmt.Println("  cross-block case needs no extra machinery at all.")

	fmt.Println("\n=== validate-then-mutate, and why ===")
	// A block whose first transaction is fine and whose second is not.
	good := spend([]Outpoint{{txA.TxID(), 0}}, pay(29, "carol"))
	bad := spend([]Outpoint{ghost}, pay(1, "mallory"))
	blk := &Block{2, []*Transaction{coinbase(2, 51, "miner"), good, bad}}

	before := fingerprint(set)
	naive := UTXOSet{}
	for k, v := range set {
		naive[k] = v
	}
	err := ApplyAsYouGo(naive, blk)
	fmt.Printf("  apply-as-you-go: %v\n", err)
	fmt.Printf("    set before : %s\n", before)
	fmt.Printf("    set after  : %s   <- corrupted\n", fingerprint(naive))

	if _, _, err := ValidateBlock(set, blk); err != nil {
		fmt.Printf("\n  validate-then-mutate: %v\n", err)
	}
	fmt.Printf("    set before : %s\n", before)
	fmt.Printf("    set after  : %s   <- untouched\n", fingerprint(set))
	fmt.Println()
	fmt.Println("  The corrupted node does not crash. It carries on with a state")
	fmt.Println("  nobody else has, accepts blocks nobody else accepts, and forks")
	fmt.Println("  itself off the network — the same failure mode as lesson 08's")
	fmt.Println("  partially-appended block, one level up.")
}
```

**Output:**

```
=== starting set ===
  2 entries, 43a721f2b9ab
  c100…:0 alice 50
  c200…:0 alice 20

=== blocks that are fine ===
  one payment                              ok, fees 1
  two payments, different coins            ok, fees 2
  chained: txC spends txA's output         ok, fees 2

=== ordering matters ===
  the same two, txC before txA             REJECTED: tx 1: input spends an output that does not exist: a30eed9f2a0e…:0
  Bitcoin requires a parent to appear before its child in the
  block. That keeps validation a single forward pass — no
  topological sort, no second pass, no cycles to worry about.

=== the double spends ===
  same outpoint in two txs                 REJECTED: tx 2: outpoint spent twice: c10000000000…:0, already spent by tx 1
  same outpoint twice in one tx            REJECTED: tx 1: outpoint spent twice: c10000000000…:0, named twice by this same transaction
  spending an outpoint that never existed  REJECTED: tx 1: input spends an output that does not exist: ff0000000000…:0

=== spending across blocks ===
  block 1 applied      -> 4 entries, 23009ec8fcbd
  block 2 spends c1 again                  REJECTED: tx 1: input spends an output that does not exist: c10000000000…:0
  Once applied, the outpoint is simply gone from the set, so the
  cross-block case needs no extra machinery at all.

=== validate-then-mutate, and why ===
  apply-as-you-go: tx 2: input spends an output that does not exist
    set before : 4 entries, 23009ec8fcbd
    set after  : 5 entries, 9fee41ab7107   <- corrupted

  validate-then-mutate: tx 2: input spends an output that does not exist: ff0000000000…:0
    set before : 4 entries, 23009ec8fcbd
    set after  : 4 entries, 23009ec8fcbd   <- untouched

  The corrupted node does not crash. It carries on with a state
  nobody else has, accepts blocks nobody else accepts, and forks
  itself off the network — the same failure mode as lesson 08's
  partially-appended block, one level up.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
