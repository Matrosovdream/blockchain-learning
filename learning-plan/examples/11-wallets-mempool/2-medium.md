# Step 11 — Wallets, Fees & the Mempool · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Building a spend, end to end

`🟡 medium` · *Building a spend*

The whole builder, end to end, printing its working at every stage: select, estimate, fee, change, sign, check. The order is not negotiable — signing last is what makes everything before it safe to change.

**Steps:**

1. Walk one payment through all six stages and read the numbers.
2. Run the same payment at five fee rates and watch it fail at the top of the range.
3. Ask for an amount that leaves change below the dust threshold and watch the output disappear into the fee.
4. Confirm the size estimate matches the serializer exactly.
5. Try to overspend, and note that it fails BEFORE any signature exists.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"sort"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Building a spend, end to end. Six stages, and the order is not negotiable:
//
//   1. select coins          -- which UTXOs to spend
//   2. estimate the size     -- from the SHAPE, before signing (example 4)
//   3. compute the fee       -- rate x size, never a round number
//   4. create the change     -- or fold it into the fee if it is dust
//   5. sign every input      -- last, because signing freezes everything
//   6. check it before sending
//
// Stages 1-3 are circular: the fee needs the size, the size needs the inputs,
// the inputs need the fee. Example 7 is about that; here the loop just runs.
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

func Sign(t *Transaction, priv *ecdsa.PrivateKey, prevOuts []TxOutput) error {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			return err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return nil
}

func Verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: key does not open the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

// --------------------------------------------------------------- the wallet

// Our serialization has no witness discount, so the numbers are bigger than
// Bitcoin's (11 / 68 / 31 vbytes, example 3). They are exact for OUR format,
// which is what matters: an estimate that does not match the serializer is
// just a different bug.
const (
	overheadSize = 8   // 4 input count + 4 output count
	inputSize    = 142 // 32 txid + 4 index + 4+65 signature + 4+33 pubkey
	outputSize   = 32  // 8 value + 4+20 pubkeyhash

	dustThreshold = 294
)

type Coin struct {
	Op  Outpoint
	Out TxOutput
}

type Wallet struct {
	priv  *ecdsa.PrivateKey
	coins []Coin
}

func (w *Wallet) PKH() []byte { return hash160(crypto.CompressPubkey(&w.priv.PublicKey)) }

func estimateSize(nIn, nOut int) int64 {
	return int64(overheadSize + nIn*inputSize + nOut*outputSize)
}

var ErrInsufficient = errors.New("insufficient funds")

// Spend is the whole builder. It reports what it did at every stage, because
// a wallet that cannot explain its fee is a wallet nobody trusts.
func (w *Wallet) Spend(to []byte, amount, feeRate int64, log bool) (*Transaction, []TxOutput, error) {
	// --- 1. select ---------------------------------------------------------
	coins := append([]Coin(nil), w.coins...)
	sort.Slice(coins, func(i, j int) bool { return coins[i].Out.Value > coins[j].Out.Value })

	var chosen []Coin
	var total int64
	for i := 0; ; i++ {
		if i >= len(coins) {
			return nil, nil, fmt.Errorf("%w: have %d", ErrInsufficient, total)
		}
		chosen = append(chosen, coins[i])
		total += coins[i].Out.Value

		// --- 2 & 3. size, then fee -----------------------------------------
		// Assume a change output while deciding; drop it in stage 4 if it
		// turns out to be dust.
		size := estimateSize(len(chosen), 2)
		fee := size * feeRate
		if total >= amount+fee {
			break
		}
	}

	size := estimateSize(len(chosen), 2)
	fee := size * feeRate
	change := total - amount - fee

	// --- 4. change, or not -------------------------------------------------
	outs := []TxOutput{{Value: amount, PubKeyHash: to}}
	if change >= dustThreshold {
		outs = append(outs, TxOutput{Value: change, PubKeyHash: w.PKH()})
	} else {
		// No change output: the transaction is smaller, and everything that
		// would have been change goes to the miner instead.
		size = estimateSize(len(chosen), 1)
		fee = total - amount
		change = 0
	}

	t := &Transaction{Outputs: outs}
	prevOuts := make([]TxOutput, 0, len(chosen))
	for _, c := range chosen {
		t.Inputs = append(t.Inputs, TxInput{Prev: c.Op})
		prevOuts = append(prevOuts, c.Out)
	}

	if log {
		fmt.Printf("  1. selected      %d coin(s), %d sat total\n", len(chosen), total)
		for _, c := range chosen {
			fmt.Printf("       %x…:%d  %d\n", c.Op.TxID[:4], c.Op.Index, c.Out.Value)
		}
		fmt.Printf("  2. estimated     %d bytes (%d in, %d out)\n", size, len(t.Inputs), len(outs))
		fmt.Printf("  3. fee           %d sat, target %d sat/byte, actual %d\n", fee, feeRate, fee/size)
		if change > 0 {
			fmt.Printf("  4. change        %d sat back to %x\n", change, w.PKH()[:6])
		} else {
			fmt.Printf("  4. change        none — below the %d sat dust threshold,\n", dustThreshold)
			fmt.Printf("                   so %d sat goes to the miner instead\n", fee)
		}
	}

	// --- 5. sign -----------------------------------------------------------
	if err := Sign(t, w.priv, prevOuts); err != nil {
		return nil, nil, err
	}
	if log {
		actual := int64(len(t.Serialize()))
		fmt.Printf("  5. signed        %d input(s); real size %d bytes, estimate was %d (error %+d)\n",
			len(t.Inputs), actual, size, size-actual)
	}
	return t, prevOuts, nil
}

// --- 6. the pre-broadcast checks (example 8 goes into these properly) -------

func Check(t *Transaction, prevOuts []TxOutput, ownPKH, to []byte, amount, feeRate int64) error {
	if err := Verify(t, prevOuts); err != nil {
		return err
	}
	var in, out int64
	for _, p := range prevOuts {
		in += p.Value
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if out > in {
		return errors.New("outputs exceed inputs")
	}
	if !bytes.Equal(t.Outputs[0].PubKeyHash, to) || t.Outputs[0].Value != amount {
		return errors.New("output 0 is not the payment we asked for")
	}
	for _, o := range t.Outputs[1:] {
		if !bytes.Equal(o.PubKeyHash, ownPKH) {
			return errors.New("a non-payment output does not come back to us")
		}
	}
	if rate := (in - out) / int64(len(t.Serialize())); rate < feeRate-1 {
		return fmt.Errorf("fee rate %d is below the target %d", rate, feeRate)
	}
	return nil
}

// -------------------------------------------------------------------- setup

const (
	aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
	bobKey   = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d" // TEST ONLY
)

func key(h string) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(h)
	if err != nil {
		panic(err)
	}
	return k
}

func wallet(k *ecdsa.PrivateKey, values ...int64) *Wallet {
	w := &Wallet{priv: k}
	pkh := hash160(crypto.CompressPubkey(&k.PublicKey))
	for i, v := range values {
		var id [32]byte
		id[0] = byte(0xc0 + i)
		w.coins = append(w.coins, Coin{Outpoint{id, 0}, TxOutput{Value: v, PubKeyHash: pkh}})
	}
	return w
}

func main() {
	alice := wallet(key(aliceKey), 500_000, 120_000, 40_000, 8_000, 900)
	bob := hash160(crypto.CompressPubkey(&key(bobKey).PublicKey))

	var have int64
	for _, c := range alice.coins {
		have += c.Out.Value
	}
	fmt.Printf("=== alice's wallet: %d coins, %d sat ===\n", len(alice.coins), have)

	fmt.Println("\n=== paying bob 100,000 sat at 20 sat/vB ===")
	t, prev, err := alice.Spend(bob, 100_000, 20, true)
	if err != nil {
		fmt.Println("  ", err)
		return
	}
	fmt.Printf("  6. checks        %s\n", res(Check(t, prev, alice.PKH(), bob, 100_000, 20)))
	fmt.Printf("     txid          %x\n", t.TxID())

	fmt.Println("\n=== the same payment at five fee rates ===")
	fmt.Printf("  %-10s %-8s %-8s %-9s %-10s %s\n",
		"rate", "inputs", "bytes", "fee", "change", "total cost to alice")
	for _, rate := range []int64{1, 20, 200, 1000, 3000} {
		t, prev, err := alice.Spend(bob, 100_000, rate, false)
		if err != nil {
			fmt.Printf("  %-10d %v\n", rate, err)
			continue
		}
		var in, out int64
		for _, p := range prev {
			in += p.Value
		}
		var change int64
		for _, o := range t.Outputs[1:] {
			out += o.Value
			change += o.Value
		}
		fee := in - 100_000 - change
		fmt.Printf("  %-10d %-8d %-8d %-9d %-10d %d\n",
			rate, len(t.Inputs), len(t.Serialize()), fee, change, 100_000+fee)
	}
	fmt.Println("\n  The fee rises linearly with the rate until the wallet runs out")
	fmt.Println("  of coins that are worth spending. Adding an input to cover a")
	fmt.Println("  bigger fee makes the transaction bigger, which makes the fee")
	fmt.Println("  bigger again — the circularity of example 7, which at high enough")
	fmt.Println("  rates simply never closes.")

	fmt.Println("\n=== when the change would be dust ===")
	// Ask for an amount that leaves almost exactly the fee behind.
	small := wallet(key(aliceKey), 104_480)
	t2, prev2, err := small.Spend(bob, 100_000, 20, true)
	if err != nil {
		fmt.Println("  ", err)
		return
	}
	fmt.Printf("  6. checks        %s\n", res(Check(t2, prev2, small.PKH(), bob, 100_000, 20)))
	fmt.Printf("     outputs       %d — payment only\n", len(t2.Outputs))

	fmt.Println("\n=== what the wallet cannot do ===")
	_, _, err = alice.Spend(bob, 1_000_000, 20, false)
	fmt.Printf("  paying more than it holds: %v\n", err)
	fmt.Println()
	fmt.Println("  Note it fails BEFORE signing. A wallet that discovers it cannot")
	fmt.Println("  pay only after producing signatures has leaked its public keys")
	fmt.Println("  for nothing, and has a signed object it must be careful to")
	fmt.Println("  destroy rather than accidentally broadcast.")
}

func res(err error) string {
	if err == nil {
		return "ok"
	}
	return "FAILED: " + err.Error()
}
```

**Output:**

```
=== alice's wallet: 5 coins, 668900 sat ===

=== paying bob 100,000 sat at 20 sat/vB ===
  1. selected      1 coin(s), 500000 sat total
       c0000000…:0  500000
  2. estimated     214 bytes (1 in, 2 out)
  3. fee           4280 sat, target 20 sat/byte, actual 20
  4. change        395720 sat back to a55476015c13
  5. signed        1 input(s); real size 214 bytes, estimate was 214 (error +0)
  6. checks        ok
     txid          ff231b02de0a64d4953313d584fc611911881888a966aec3c8c8deec2f218f45

=== the same payment at five fee rates ===
  rate       inputs   bytes    fee       change     total cost to alice
  1          1        214      214       399786     100214
  20         1        214      4280      395720     104280
  200        1        214      42800     357200     142800
  1000       1        214      214000    186000     314000
  3000       insufficient funds: have 668900

  The fee rises linearly with the rate until the wallet runs out
  of coins that are worth spending. Adding an input to cover a
  bigger fee makes the transaction bigger, which makes the fee
  bigger again — the circularity of example 7, which at high enough
  rates simply never closes.

=== when the change would be dust ===
  1. selected      1 coin(s), 104480 sat total
       c0000000…:0  104480
  2. estimated     182 bytes (1 in, 1 out)
  3. fee           4480 sat, target 20 sat/byte, actual 24
  4. change        none — below the 294 sat dust threshold,
                   so 4480 sat goes to the miner instead
  5. signed        1 input(s); real size 182 bytes, estimate was 182 (error +0)
  6. checks        ok
     outputs       1 — payment only

=== what the wallet cannot do ===
  paying more than it holds: insufficient funds: have 668900

  Note it fails BEFORE signing. A wallet that discovers it cannot
  pay only after producing signatures has leaked its public keys
  for nothing, and has a signed object it must be careful to
  destroy rather than accidentally broadcast.
```

---

## 7. The fee/size circularity

`🟡 medium` · *Building a spend*

Three ways to break the fee/size circularity, in order of how well they work: ignore it and under-pay; iterate to a fixpoint; or price each input's own fee into the coin and make the loop unnecessary.

**Steps:**

1. Select for the amount, add the fee afterwards, and come up short.
2. Iterate instead, and watch it converge in two rounds on a healthy wallet.
3. Run the same loop on 200 dust coins and watch it grind through 47 rounds to 134 inputs.
4. Switch to effective values and get a better answer in one pass — 130 inputs and 12,000 sat less.
5. Tabulate 'spendable' against the fee rate and see most of a wallet stop being money at 200 sat/byte.

```go
package main

import (
	"errors"
	"fmt"
	"sort"
)

// ===========================================================================
// The circularity at the heart of every wallet:
//
//     the FEE depends on the SIZE
//     the SIZE depends on the INPUTS
//     the INPUTS depend on the FEE
//
// Three ways to break it, in order of how well they work.
// ===========================================================================

const (
	overheadSize = 8
	inputSize    = 142
	outputSize   = 32

	dustThreshold = 294
)

type Coin struct {
	Name  string
	Value int64
}

func size(nIn, nOut int) int64 {
	return int64(overheadSize + nIn*inputSize + nOut*outputSize)
}

var (
	ErrInsufficient = errors.New("insufficient funds")
	ErrNoConverge   = errors.New("selection did not converge")
)

// ------------------------------------------- 1. ignore it, and under-pay

// SelectNaive picks coins to cover the AMOUNT, then works out the fee
// afterwards. Every wallet writes this first.
func SelectNaive(coins []Coin, amount, rate int64) ([]Coin, int64, int64) {
	var chosen []Coin
	var total int64
	for _, c := range coins {
		if total >= amount {
			break
		}
		chosen = append(chosen, c)
		total += c.Value
	}
	fee := size(len(chosen), 2) * rate
	return chosen, total, fee
}

// ------------------------------------------------- 2. iterate to a fixpoint

// SelectIterative re-selects with the fee implied by the current selection,
// and repeats until the selection stops changing. It converges when coins are
// large relative to the per-input fee, and does not when they are not.
func SelectIterative(coins []Coin, amount, rate int64, trace bool) ([]Coin, int64, error) {
	need := amount // first guess: no fee at all
	if trace {
		fmt.Printf("      %-7s %-8s %-8s %-11s %-11s %s\n",
			"round", "inputs", "bytes", "fee", "needed", "selected")
	}
	var prev int
	const maxRounds = 60
	for round := 1; round <= maxRounds; round++ {
		var chosen []Coin
		var total int64
		for _, c := range coins {
			if total >= need {
				break
			}
			chosen = append(chosen, c)
			total += c.Value
		}
		fee := size(len(chosen), 2) * rate
		if trace && (round <= 3 || round%5 == 0 || total >= amount+fee) {
			fmt.Printf("      %-7d %-8d %-8d %-11d %-11d %d\n",
				round, len(chosen), size(len(chosen), 2), fee, need, total)
		}
		if total < amount+fee {
			if len(chosen) == len(coins) {
				return nil, 0, ErrInsufficient
			}
			if len(chosen) == prev {
				return nil, 0, ErrNoConverge // adding inputs stopped helping
			}
			prev = len(chosen)
			need = amount + fee
			continue
		}
		return chosen, total, nil
	}
	return nil, 0, ErrNoConverge
}

// ------------------------------------- 3. price the input into the coin

// effectiveValue is what a coin is really worth: its value minus the fee for
// the input that spends it. Once values are quoted this way the fee no longer
// depends on how many coins you pick, and the circularity is gone.
func effectiveValue(c Coin, rate int64) int64 { return c.Value - rate*inputSize }

// SelectEffective needs no loop and no guess. The target is the payment plus
// the parts of the fee that do not scale with the inputs.
func SelectEffective(coins []Coin, amount, rate int64) ([]Coin, int64, error) {
	target := amount + rate*(overheadSize+outputSize)
	var chosen []Coin
	var sumEff, total int64
	for _, c := range coins {
		if ev := effectiveValue(c, rate); ev > 0 {
			chosen = append(chosen, c)
			sumEff += ev
			total += c.Value
			if sumEff >= target {
				return chosen, total, nil
			}
		}
	}
	return nil, 0, ErrInsufficient
}

// --------------------------------------------------------------------------

func report(chosen []Coin, total, amount, rate int64) {
	nOut := 2
	fee := size(len(chosen), nOut) * rate
	change := total - amount - fee
	if change < dustThreshold {
		nOut = 1
		fee = total - amount
		change = 0
	}
	sz := size(len(chosen), nOut)
	fmt.Printf("      %d inputs, %d bytes, fee %d, change %d, actual rate %d\n",
		len(chosen), sz, fee, change, fee/sz)
}

// names is for the demo output only; long selections are summarised.
func names(cs []Coin) string {
	if len(cs) > 6 {
		return fmt.Sprintf("[%s %s ... %s] (%d coins)",
			cs[0].Name, cs[1].Name, cs[len(cs)-1].Name, len(cs))
	}
	out := make([]string, len(cs))
	for i, c := range cs {
		out[i] = c.Name
	}
	return fmt.Sprint(out)
}

func main() {
	// A wallet of a few decent coins and a lot of small ones — the shape any
	// wallet ends up with after a while (lesson 10, example 14).
	var wallet []Coin
	for i, v := range []int64{60_000, 30_000, 12_000, 9_000, 7_000, 6_000, 5_500, 5_000, 4_800, 4_500} {
		wallet = append(wallet, Coin{fmt.Sprintf("c%d", i), v})
	}
	sort.SliceStable(wallet, func(i, j int) bool { return wallet[i].Value > wallet[j].Value })

	var have int64
	for _, c := range wallet {
		have += c.Value
	}
	fmt.Printf("=== wallet: %d coins, %d sat ===\n  ", len(wallet), have)
	for i, c := range wallet {
		if i > 0 {
			fmt.Print(" ")
		}
		fmt.Printf("%d", c.Value)
	}
	fmt.Println()

	// ---------------------------------------------------------------------
	fmt.Println("\n=== 1. select for the amount, then add the fee ===")
	const amount, rate = 58_000, 20
	fmt.Printf("  paying %d at %d sat/byte\n\n", amount, rate)
	chosen, total, fee := SelectNaive(wallet, amount, rate)
	fmt.Printf("      picked %s = %d sat\n", names(chosen), total)
	fmt.Printf("      fee for %d inputs: %d\n", len(chosen), fee)
	fmt.Printf("      amount + fee = %d, but we only selected %d\n", amount+fee, total)
	fmt.Printf("      short by %d sat\n", amount+fee-total)
	fmt.Println()
	fmt.Println("  The transaction cannot be built. If the wallet ships it anyway")
	fmt.Println("  by shaving the change, it under-pays and sits in the mempool.")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== 2. iterate until it stops changing ===")
	chosen, total, err := SelectIterative(wallet, amount, rate, true)
	if err != nil {
		fmt.Println("     ", err)
	} else {
		fmt.Printf("\n      converged: %s = %d sat\n", names(chosen), total)
		report(chosen, total, amount, rate)
	}
	fmt.Println()
	fmt.Println("  Two rounds. Each round adds inputs, which raises the fee, which")
	fmt.Println("  may need more inputs. It terminates quickly here because the")
	fmt.Println("  coins are worth far more than an input costs. That is not")
	fmt.Println("  always true.")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== when iterating grinds ===")
	var dusty []Coin
	for i := 0; i < 200; i++ {
		dusty = append(dusty, Coin{fmt.Sprintf("d%d", i), 3_000})
	}
	fmt.Printf("  wallet: 200 coins of 3000 sat = %d sat\n", 200*3000)
	fmt.Printf("  paying %d at %d sat/byte -> each input costs %d sat in fee,\n",
		20_000, rate, rate*inputSize)
	fmt.Printf("  so a 3000-sat coin contributes %d sat towards the payment\n\n",
		3_000-rate*inputSize)
	dChosen, dTotal, err := SelectIterative(dusty, 20_000, rate, true)
	if err != nil {
		fmt.Printf("\n      result: %v\n", err)
	} else {
		fmt.Printf("\n      converged on %d inputs, %d sat\n", len(dChosen), dTotal)
		report(dChosen, dTotal, 20_000, rate)
	}
	fmt.Println()
	fmt.Println("  Every round adds inputs, every input raises the fee, and the")
	fmt.Println("  requirement chases the selection up the wallet. It gets there,")
	fmt.Println("  but only just, and the round cap is load-bearing: with slightly")
	fmt.Println("  smaller coins the fixpoint does not exist at all and the loop")
	fmt.Println("  runs until it hits the cap or the wallet runs out.")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== 3. price the input into the coin, and the loop disappears ===")
	fmt.Printf("  %-8s %-10s %-14s %s\n", "coin", "value", "effective", "note")
	for _, c := range dusty[:2] {
		fmt.Printf("  %-8s %-10d %-14d costs %d to spend\n",
			c.Name, c.Value, effectiveValue(c, rate), rate*inputSize)
	}
	for _, c := range wallet[:2] {
		fmt.Printf("  %-8s %-10d %-14d costs %d to spend\n",
			c.Name, c.Value, effectiveValue(c, rate), rate*inputSize)
	}

	fmt.Println("\n  the same two problems, single pass, no iteration:")
	for _, c := range []struct {
		label  string
		coins  []Coin
		amount int64
	}{
		{"the good wallet", wallet, amount},
		{"the dusty wallet", dusty, 20_000},
	} {
		chosen, total, err := SelectEffective(c.coins, c.amount, rate)
		if err != nil {
			fmt.Printf("    %-18s %v\n", c.label, err)
			continue
		}
		fmt.Printf("    %-18s %s = %d sat\n", c.label, names(chosen), total)
		report(chosen, total, c.amount, rate)
	}

	fmt.Println()
	fmt.Println("  Same answer for the good wallet. For the dusty one it finds a")
	fmt.Printf("  BETTER one in a single pass: 130 inputs against the loop's %d, and\n", len(dChosen))
	fmt.Println("  12,000 sat less in fees, because it never overshoots chasing a")
	fmt.Println("  moving target. No loop, no round cap, no oscillation.")
	fmt.Println()
	fmt.Println("  Effective values also make the comparison between coins honest:")
	fmt.Println("  a 3000-sat coin at 20 sat/byte is worth 160, and the wallet can")
	fmt.Println("  say so in the UI instead of surprising the user at send time.")

	fmt.Println("\n=== and it degrades gracefully with the fee rate ===")
	fmt.Printf("  %-10s %-12s %-14s %s\n", "rate", "spendable", "usable coins", "can pay 58,000?")
	for _, r := range []int64{1, 5, 20, 50, 100, 200} {
		var spendable int64
		usable := 0
		for _, c := range wallet {
			if ev := effectiveValue(c, r); ev > 0 {
				spendable += ev
				usable++
			}
		}
		_, _, err := SelectEffective(wallet, amount, r)
		fmt.Printf("  %-10d %-12d %-14d %v\n", r, spendable, usable, err == nil)
	}
	fmt.Println("\n  'Spendable' is not 'balance'. At 200 sat/byte more than half")
	fmt.Println("  this wallet has negative effective value and simply is not money")
	fmt.Println("  any more — until fees come down.")
}
```

**Output:**

```
=== wallet: 10 coins, 143800 sat ===
  60000 30000 12000 9000 7000 6000 5500 5000 4800 4500

=== 1. select for the amount, then add the fee ===
  paying 58000 at 20 sat/byte

      picked [c0] = 60000 sat
      fee for 1 inputs: 4280
      amount + fee = 62280, but we only selected 60000
      short by 2280 sat

  The transaction cannot be built. If the wallet ships it anyway
  by shaving the change, it under-pays and sits in the mempool.

=== 2. iterate until it stops changing ===
      round   inputs   bytes    fee         needed      selected
      1       1        214      4280        58000       60000
      2       2        356      7120        62280       90000

      converged: [c0 c1] = 90000 sat
      2 inputs, 356 bytes, fee 7120, change 24880, actual rate 20

  Two rounds. Each round adds inputs, which raises the fee, which
  may need more inputs. It terminates quickly here because the
  coins are worth far more than an input costs. That is not
  always true.

=== when iterating grinds ===
  wallet: 200 coins of 3000 sat = 600000 sat
  paying 20000 at 20 sat/byte -> each input costs 2840 sat in fee,
  so a 3000-sat coin contributes 160 sat towards the payment

      round   inputs   bytes    fee         needed      selected
      1       7        1066     21320       20000       21000
      2       14       2060     41200       41320       42000
      3       21       3054     61080       61200       63000
      5       34       4900     98000       100960      102000
      10      61       8734     174680      180480      183000
      15      81       11574    231480      240120      243000
      20      96       13704    274080      285560      288000
      25      107      15266    305320      319640      321000
      30      117      16686    333720      348040      351000
      35      122      17396    347920      365080      366000
      40      127      18106    362120      379280      381000
      45      132      18816    376320      393480      396000
      47      134      19100    382000      399160      402000

      converged on 134 inputs, 402000 sat
      134 inputs, 19068 bytes, fee 382000, change 0, actual rate 20

  Every round adds inputs, every input raises the fee, and the
  requirement chases the selection up the wallet. It gets there,
  but only just, and the round cap is load-bearing: with slightly
  smaller coins the fixpoint does not exist at all and the loop
  runs until it hits the cap or the wallet runs out.

=== 3. price the input into the coin, and the loop disappears ===
  coin     value      effective      note
  d0       3000       160            costs 2840 to spend
  d1       3000       160            costs 2840 to spend
  c0       60000      57160          costs 2840 to spend
  c1       30000      27160          costs 2840 to spend

  the same two problems, single pass, no iteration:
    the good wallet    [c0 c1] = 90000 sat
      2 inputs, 356 bytes, fee 7120, change 24880, actual rate 20
    the dusty wallet   [d0 d1 ... d129] (130 coins) = 390000 sat
      130 inputs, 18500 bytes, fee 370000, change 0, actual rate 20

  Same answer for the good wallet. For the dusty one it finds a
  BETTER one in a single pass: 130 inputs against the loop's 134, and
  12,000 sat less in fees, because it never overshoots chasing a
  moving target. No loop, no round cap, no oscillation.

  Effective values also make the comparison between coins honest:
  a 3000-sat coin at 20 sat/byte is worth 160, and the wallet can
  say so in the UI instead of surprising the user at send time.

=== and it degrades gracefully with the fee rate ===
  rate       spendable    usable coins   can pay 58,000?
  1          142380       10             true
  5          136700       10             true
  20         115400       10             true
  50         82600        4              true
  100        61600        2              false
  200        33200        2              false

  'Spendable' is not 'balance'. At 200 sat/byte more than half
  this wallet has negative effective value and simply is not money
  any more — until fees come down.
```

---

## 8. Checking your own work before broadcast

`🟡 medium` · *Building a spend*

The last thing a wallet does before broadcasting is check its own work — because half of these failures are things the network will happily accept. A change output to the wrong address is a perfectly valid transaction that destroys your money.

**Steps:**

1. Write ten preflight checks over a `Draft` that carries the user's INTENT, not just the transaction.
2. Break the draft ten ways and watch each check fire, cheapest first.
3. Tabulate which of the ten a node would have caught.
4. Count them: five are invisible to the network, and those five are the ones that lose money.
5. Read how to run these for real — in the code path, failing closed, with the intent stored beside the transaction.

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
// The last thing a wallet does before broadcasting is check its own work.
//
// This is not paranoia. Half of these failures are things the NETWORK will
// happily accept: a change output to the wrong address is a perfectly valid
// transaction that destroys your money, and there is no undo. The node has
// no idea which output was meant to be change.
//
// So the wallet checks the things only the wallet knows.
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

func Sign(t *Transaction, priv *ecdsa.PrivateKey, prevOuts []TxOutput) error {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			return err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return nil
}

func Verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: key does not open the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

// ------------------------------------------------------------------ preflight

// Draft is everything the checks need: the transaction, the outputs it
// spends, and the INTENT — what the user actually asked for.
type Draft struct {
	Tx       *Transaction
	PrevOuts []TxOutput
	OwnPKH   []byte // where change is allowed to go
	Payee    []byte // who the user said to pay
	Amount   int64  // how much the user said to pay
	FeeRate  int64  // what rate the user agreed to
}

func (d *Draft) In() (n int64) {
	for _, p := range d.PrevOuts {
		n += p.Value
	}
	return
}

func (d *Draft) Out() (n int64) {
	for _, o := range d.Tx.Outputs {
		n += o.Value
	}
	return
}

func (d *Draft) Fee() int64  { return d.In() - d.Out() }
func (d *Draft) Size() int64 { return int64(len(d.Tx.Serialize())) }

type Check struct {
	Name string
	Fn   func(*Draft) error
}

// Policy limits. These are the wallet's, not the network's — pick them to
// suit the money you are moving, and make them configurable.
const (
	maxFeeRate     = 500 // sat/byte; above this, something has gone wrong
	maxFeeFraction = 10  // percent of the amount
	dustThreshold  = 294
)

var Preflight = []Check{
	{"every input is signed", func(d *Draft) error {
		for i, in := range d.Tx.Inputs {
			if len(in.Signature) != 65 || len(in.PubKey) != 33 {
				return fmt.Errorf("input %d has no signature", i)
			}
		}
		return nil
	}},

	{"no input appears twice", func(d *Draft) error {
		seen := map[Outpoint]int{}
		for i, in := range d.Tx.Inputs {
			if j, dup := seen[in.Prev]; dup {
				return fmt.Errorf("inputs %d and %d spend the same outpoint", j, i)
			}
			seen[in.Prev] = i
		}
		return nil
	}},

	{"we own every input", func(d *Draft) error {
		for i, p := range d.PrevOuts {
			if !bytes.Equal(p.PubKeyHash, d.OwnPKH) {
				return fmt.Errorf("input %d spends an output we do not control", i)
			}
		}
		return nil
	}},

	{"every signature verifies", func(d *Draft) error {
		return Verify(d.Tx, d.PrevOuts)
	}},

	{"value is conserved", func(d *Draft) error {
		if d.Out() > d.In() {
			return fmt.Errorf("outputs %d exceed inputs %d", d.Out(), d.In())
		}
		return nil
	}},

	{"the payment is what was asked for", func(d *Draft) error {
		for _, o := range d.Tx.Outputs {
			if bytes.Equal(o.PubKeyHash, d.Payee) {
				if o.Value != d.Amount {
					return fmt.Errorf("paying %d, user asked for %d", o.Value, d.Amount)
				}
				return nil
			}
		}
		return errors.New("no output pays the intended recipient")
	}},

	// The one that saves careers. Every output that is not the payment is,
	// by definition, change — and change that does not come back to us is
	// money we have given away, irreversibly, with a valid transaction.
	{"all change comes back to us", func(d *Draft) error {
		for i, o := range d.Tx.Outputs {
			if bytes.Equal(o.PubKeyHash, d.Payee) {
				continue
			}
			if !bytes.Equal(o.PubKeyHash, d.OwnPKH) {
				return fmt.Errorf("output %d (%d sat) goes to %x…, which is neither the payee nor us",
					i, o.Value, o.PubKeyHash[:6])
			}
		}
		return nil
	}},

	{"no dust outputs", func(d *Draft) error {
		for i, o := range d.Tx.Outputs {
			if o.Value < dustThreshold {
				return fmt.Errorf("output %d is %d sat, below the %d dust threshold",
					i, o.Value, dustThreshold)
			}
		}
		return nil
	}},

	{"the fee is not absurd", func(d *Draft) error {
		if rate := d.Fee() / d.Size(); rate > maxFeeRate {
			return fmt.Errorf("fee rate %d sat/byte exceeds the %d limit", rate, maxFeeRate)
		}
		if d.Fee()*100 > d.Amount*maxFeeFraction {
			return fmt.Errorf("fee %d is more than %d%% of the %d being sent",
				d.Fee(), maxFeeFraction, d.Amount)
		}
		return nil
	}},

	{"the fee is not too low", func(d *Draft) error {
		if rate := d.Fee() / d.Size(); rate < d.FeeRate {
			return fmt.Errorf("fee rate %d sat/byte is below the agreed %d", rate, d.FeeRate)
		}
		return nil
	}},
}

func run(label string, d *Draft) {
	fmt.Printf("  %-34s ", label)
	for _, c := range Preflight {
		if err := c.Fn(d); err != nil {
			fmt.Printf("BLOCKED [%s]\n      %v\n", c.Name, err)
			return
		}
	}
	fmt.Printf("ok — fee %d sat over %d bytes (%d sat/byte)\n", d.Fee(), d.Size(), d.Fee()/d.Size())
}

// -------------------------------------------------------------------- setup

const (
	aliceKey   = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
	bobKey     = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d" // TEST ONLY
	strangeKey = "5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a" // TEST ONLY
)

func key(h string) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(h)
	if err != nil {
		panic(err)
	}
	return k
}

func pkhOf(h string) []byte {
	return hash160(crypto.CompressPubkey(&key(h).PublicKey))
}

// draft builds a good 1-in, 2-out payment: 100,000 to bob, change to alice.
func draft() *Draft {
	alice, bob := pkhOf(aliceKey), pkhOf(bobKey)
	var id [32]byte
	id[0] = 0xc0
	prev := TxOutput{Value: 200_000, PubKeyHash: alice}

	t := &Transaction{
		Inputs: []TxInput{{Prev: Outpoint{id, 0}}},
		Outputs: []TxOutput{
			{Value: 100_000, PubKeyHash: bob},
			{Value: 95_720, PubKeyHash: alice}, // 200000 - 100000 - 4280 fee
		},
	}
	prevOuts := []TxOutput{prev}
	if err := Sign(t, key(aliceKey), prevOuts); err != nil {
		panic(err)
	}
	return &Draft{Tx: t, PrevOuts: prevOuts, OwnPKH: alice, Payee: bob, Amount: 100_000, FeeRate: 20}
}

// resign re-signs after a mutation, so the check being demonstrated is the
// one that fires — not just "the signature broke".
func resign(d *Draft) *Draft {
	if err := Sign(d.Tx, key(aliceKey), d.PrevOuts); err != nil {
		panic(err)
	}
	return d
}

func main() {
	fmt.Println("=== the transaction the wallet meant to build ===")
	run("as built", draft())

	fmt.Println("\n=== ten ways it can be wrong ===")

	d := draft()
	d.Tx.Inputs[0].Signature = nil
	run("an input was never signed", d)

	d = draft()
	d.Tx.Inputs[0].Signature[10] ^= 1
	run("a signature is corrupt", d)

	d = draft()
	d.Tx.Inputs = append(d.Tx.Inputs, d.Tx.Inputs[0])
	d.PrevOuts = append(d.PrevOuts, d.PrevOuts[0])
	run("the same coin added twice", resign(d))

	d = draft()
	d.PrevOuts[0].PubKeyHash = pkhOf(strangeKey)
	run("spending a coin that is not ours", d)

	d = draft()
	d.Tx.Outputs[1].Value = 195_720
	run("outputs exceed inputs", resign(d))

	d = draft()
	d.Tx.Outputs[0].Value = 10_000
	run("the payment amount is wrong", resign(d))

	// The expensive one: a bug in change-address derivation, a stale address
	// book, an off-by-one in the HD index. All produce this.
	d = draft()
	d.Tx.Outputs[1].PubKeyHash = pkhOf(strangeKey)
	run("change goes to a stranger", resign(d))

	d = draft()
	d.Tx.Outputs[1].Value = 100
	run("the change is dust", resign(d))

	// The Paxos failure mode (lesson 10): change computed wrong, so almost
	// everything becomes fee. Perfectly valid, and gone.
	d = draft()
	d.Tx.Outputs = d.Tx.Outputs[:1]
	run("change output dropped entirely", resign(d))

	d = draft()
	d.Tx.Outputs[1].Value = 99_000
	run("fee is below the agreed rate", resign(d))

	fmt.Println("\n=== which of these would the network have caught? ===")
	fmt.Printf("  %-34s %s\n", "failure", "rejected by a node?")
	for _, r := range []struct {
		what string
		net  string
	}{
		{"an input was never signed", "yes"},
		{"a signature is corrupt", "yes"},
		{"the same coin added twice", "yes"},
		{"spending a coin that is not ours", "yes"},
		{"outputs exceed inputs", "yes"},
		{"the payment amount is wrong", "NO — perfectly valid"},
		{"change goes to a stranger", "NO — perfectly valid"},
		{"the change is dust", "policy only, not consensus"},
		{"change output dropped entirely", "NO — it is just a big fee"},
		{"fee is below the agreed rate", "NO — it just waits, forever"},
	} {
		fmt.Printf("  %-34s %s\n", r.what, r.net)
	}
	fmt.Println()
	fmt.Println("  Five of ten are invisible to the network, and those five are the")
	fmt.Println("  ones that lose money. A node validates that a transaction is")
	fmt.Println("  LEGAL. Only the wallet knows what it was supposed to DO.")

	fmt.Println("\n=== how to run these for real ===")
	fmt.Println("  - as a function, on the draft, before the signature leaves the")
	fmt.Println("    process — not as a test, because a test does not run in prod")
	fmt.Println("  - fail closed: an unknown output type is a failure, not a pass")
	fmt.Println("  - show the user the fee in the same units they think in, and")
	fmt.Println("    make an unusual one require a second confirmation")
	fmt.Println("  - log the DRAFT (outpoints, amounts, addresses), never the keys")
	fmt.Println("    (example 2), so a post-mortem is possible at all")
	fmt.Println("  - keep the intent — payee, amount, rate — as data next to the")
	fmt.Println("    transaction. Checks that cannot see the intent can only ever")
	fmt.Println("    re-check what a node would.")
}
```

**Output:**

```
=== the transaction the wallet meant to build ===
  as built                           ok — fee 4280 sat over 214 bytes (20 sat/byte)

=== ten ways it can be wrong ===
  an input was never signed          BLOCKED [every input is signed]
      input 0 has no signature
  a signature is corrupt             BLOCKED [every signature verifies]
      input 0: signature does not verify
  the same coin added twice          BLOCKED [no input appears twice]
      inputs 0 and 1 spend the same outpoint
  spending a coin that is not ours   BLOCKED [we own every input]
      input 0 spends an output we do not control
  outputs exceed inputs              BLOCKED [value is conserved]
      outputs 295720 exceed inputs 200000
  the payment amount is wrong        BLOCKED [the payment is what was asked for]
      paying 10000, user asked for 100000
  change goes to a stranger          BLOCKED [all change comes back to us]
      output 1 (95720 sat) goes to 84a3b88b8d05…, which is neither the payee nor us
  the change is dust                 BLOCKED [no dust outputs]
      output 1 is 100 sat, below the 294 dust threshold
  change output dropped entirely     BLOCKED [the fee is not absurd]
      fee rate 549 sat/byte exceeds the 500 limit
  fee is below the agreed rate       BLOCKED [the fee is not too low]
      fee rate 4 sat/byte is below the agreed 20

=== which of these would the network have caught? ===
  failure                            rejected by a node?
  an input was never signed          yes
  a signature is corrupt             yes
  the same coin added twice          yes
  spending a coin that is not ours   yes
  outputs exceed inputs              yes
  the payment amount is wrong        NO — perfectly valid
  change goes to a stranger          NO — perfectly valid
  the change is dust                 policy only, not consensus
  change output dropped entirely     NO — it is just a big fee
  fee is below the agreed rate       NO — it just waits, forever

  Five of ten are invisible to the network, and those five are the
  ones that lose money. A node validates that a transaction is
  LEGAL. Only the wallet knows what it was supposed to DO.

=== how to run these for real ===
  - as a function, on the draft, before the signature leaves the
    process — not as a test, because a test does not run in prod
  - fail closed: an unknown output type is a failure, not a pass
  - show the user the fee in the same units they think in, and
    make an unusual one require a second confirmation
  - log the DRAFT (outpoints, amounts, addresses), never the keys
    (example 2), so a post-mortem is possible at all
  - keep the intent — payee, amount, rate — as data next to the
    transaction. Checks that cannot see the intent can only ever
    re-check what a node would.
```

---

## 9. Mempool admission

`🟡 medium` · *The mempool*

Admission: a transaction arrives from a peer and the node decides alone whether to keep and relay it. The checks run cheapest-first, because everything here is attacker-supplied and rejecting must cost less than sending.

**Steps:**

1. Order the checks: structure, inputs exist, conflicts, signatures, fee.
2. Accept a chain of unconfirmed transactions — the child's input is not in the UTXO set at all.
3. Reject a double-spend, a duplicate, a forged signature, a cheap fee and an overspend.
4. Hold a transaction whose parent has not arrived in a BOUNDED orphan pool.
5. Deliver the parent and watch the orphan get promoted automatically.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"errors"
	"fmt"
	"sort"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Mempool admission. A transaction arrives from a peer; the node has to
// decide, on its own, whether to keep and relay it.
//
// The checks run cheapest-first, because everything here is attacker-supplied
// (lesson 10, example 9):
//
//   1. structure and size        -- no crypto
//   2. do its inputs exist?      -- map lookups; if not, it may be an ORPHAN
//   3. does it conflict?         -- map lookups against what is already here
//   4. do the signatures verify? -- the expensive one, last
//   5. does it pay enough?       -- policy, not consensus (example 10)
//
// Then, and only then, insert — and see whether any orphan waiting for THIS
// transaction can now be admitted.
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

func Sign(t *Transaction, priv *ecdsa.PrivateKey, prevOuts []TxOutput) error {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			return err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return nil
}

func Verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: key does not open the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

// -------------------------------------------------------------- node state

type UTXOSet map[Outpoint]TxOutput

type Entry struct {
	Tx    *Transaction
	Name  string
	Size  int64
	Fee   int64
	Spent []Outpoint // what this entry claims, so conflicts are one lookup
}

type Mempool struct {
	byID      map[[32]byte]*Entry
	claimedBy map[Outpoint][32]byte // outpoint -> the pool tx that spends it
	outputs   map[Outpoint]TxOutput // outputs created by pool transactions
	orphans   map[[32]byte]*Entry
	waiting   map[Outpoint][][32]byte // missing outpoint -> orphans waiting on it
}

func NewMempool() *Mempool {
	return &Mempool{
		byID: map[[32]byte]*Entry{}, claimedBy: map[Outpoint][32]byte{},
		outputs: map[Outpoint]TxOutput{}, orphans: map[[32]byte]*Entry{},
		waiting: map[Outpoint][][32]byte{},
	}
}

type Node struct {
	utxo UTXOSet
	pool *Mempool
	// policy
	minRate int64
	maxSize int64
}

var (
	ErrEmpty      = errors.New("no inputs or no outputs")
	ErrOversize   = errors.New("larger than the policy limit")
	ErrOrphan     = errors.New("parent not known — held in the orphan pool")
	ErrConflict   = errors.New("conflicts with a transaction already in the pool")
	ErrDuplicate  = errors.New("already in the pool")
	ErrBadSig     = errors.New("signature does not verify")
	ErrNotCovered = errors.New("outputs exceed inputs")
	ErrTooCheap   = errors.New("fee rate below the pool minimum")
)

// lookup finds the output an input spends: first in the confirmed UTXO set,
// then among outputs created by transactions already in the pool. The second
// case is what lets a chain of unconfirmed transactions exist at all.
func (n *Node) lookup(op Outpoint) (TxOutput, bool) {
	if o, ok := n.utxo[op]; ok {
		return o, true
	}
	o, ok := n.pool.outputs[op]
	return o, ok
}

func (n *Node) Accept(name string, t *Transaction) error {
	id := t.TxID()

	// --- 1. structure ------------------------------------------------------
	if len(t.Inputs) == 0 || len(t.Outputs) == 0 {
		return ErrEmpty
	}
	size := int64(len(t.Serialize()))
	if size > n.maxSize {
		return fmt.Errorf("%w: %d > %d bytes", ErrOversize, size, n.maxSize)
	}
	if _, ok := n.pool.byID[id]; ok {
		return ErrDuplicate
	}

	// --- 2. do the inputs exist? -------------------------------------------
	prevOuts := make([]TxOutput, 0, len(t.Inputs))
	var missing []Outpoint
	for _, in := range t.Inputs {
		o, ok := n.lookup(in.Prev)
		if !ok {
			missing = append(missing, in.Prev)
			continue
		}
		prevOuts = append(prevOuts, o)
	}
	if len(missing) > 0 {
		n.pool.holdOrphan(name, t, size, missing)
		return fmt.Errorf("%w (%d missing)", ErrOrphan, len(missing))
	}

	// --- 3. conflicts ------------------------------------------------------
	for _, in := range t.Inputs {
		if other, ok := n.pool.claimedBy[in.Prev]; ok {
			return fmt.Errorf("%w: %x…:%d is already claimed by %s",
				ErrConflict, in.Prev.TxID[:6], in.Prev.Index, n.pool.byID[other].Name)
		}
	}

	// --- 4. signatures -----------------------------------------------------
	if err := Verify(t, prevOuts); err != nil {
		return fmt.Errorf("%w: %v", ErrBadSig, err)
	}

	// --- 5. value and fee --------------------------------------------------
	var in, out int64
	for _, p := range prevOuts {
		in += p.Value
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if out > in {
		return fmt.Errorf("%w: in %d, out %d", ErrNotCovered, in, out)
	}
	fee := in - out
	if fee/size < n.minRate {
		return fmt.Errorf("%w: %d < %d sat/byte", ErrTooCheap, fee/size, n.minRate)
	}

	// --- insert ------------------------------------------------------------
	e := &Entry{Tx: t, Name: name, Size: size, Fee: fee}
	for _, i := range t.Inputs {
		e.Spent = append(e.Spent, i.Prev)
		n.pool.claimedBy[i.Prev] = id
	}
	for i, o := range t.Outputs {
		n.pool.outputs[Outpoint{id, uint32(i)}] = o
	}
	n.pool.byID[id] = e
	return nil
}

// holdOrphan keeps a transaction whose parents have not arrived, indexed by
// what it is waiting for. The pool must be BOUNDED — an unbounded orphan pool
// is a free memory-exhaustion attack (lesson 13).
const maxOrphans = 100

func (m *Mempool) holdOrphan(name string, t *Transaction, size int64, missing []Outpoint) {
	if len(m.orphans) >= maxOrphans {
		return // silently dropped; a real node evicts at random
	}
	id := t.TxID()
	m.orphans[id] = &Entry{Tx: t, Name: name, Size: size}
	for _, op := range missing {
		m.waiting[op] = append(m.waiting[op], id)
	}
}

// promote re-tries every orphan waiting on an output that now exists.
func (n *Node) promote() []string {
	var admitted []string
	changed := true
	for changed {
		changed = false
		var ready [][32]byte
		for op, ids := range n.pool.waiting {
			if _, ok := n.lookup(op); ok {
				ready = append(ready, ids...)
				delete(n.pool.waiting, op)
			}
		}
		sort.Slice(ready, func(i, j int) bool { return bytes.Compare(ready[i][:], ready[j][:]) < 0 })
		for _, id := range ready {
			e, ok := n.pool.orphans[id]
			if !ok {
				continue
			}
			delete(n.pool.orphans, id)
			if err := n.Accept(e.Name, e.Tx); err == nil {
				admitted = append(admitted, e.Name)
				changed = true
			}
		}
	}
	return admitted
}

// -------------------------------------------------------------- scaffolding

const (
	aliceKey   = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
	bobKey     = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d" // TEST ONLY
	malloryKey = "5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a" // TEST ONLY
)

func key(h string) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(h)
	if err != nil {
		panic(err)
	}
	return k
}
func pkhOf(h string) []byte { return hash160(crypto.CompressPubkey(&key(h).PublicKey)) }

// pay builds and signs a spend of `ops` (all owned by `from`).
func pay(from string, ops []Outpoint, prevOuts []TxOutput, outs ...TxOutput) *Transaction {
	t := &Transaction{Outputs: outs}
	for _, op := range ops {
		t.Inputs = append(t.Inputs, TxInput{Prev: op})
	}
	if err := Sign(t, key(from), prevOuts); err != nil {
		panic(err)
	}
	return t
}

func out(v int64, who string) TxOutput { return TxOutput{Value: v, PubKeyHash: pkhOf(who)} }

func try(n *Node, name string, t *Transaction) {
	err := n.Accept(name, t)
	if err != nil {
		fmt.Printf("  %-26s REJECTED: %v\n", name, err)
	} else {
		fmt.Printf("  %-26s accepted\n", name)
	}
	if got := n.promote(); len(got) > 0 {
		fmt.Printf("  %-26s -> orphans promoted: %v\n", "", got)
	}
}

func main() {
	alice := pkhOf(aliceKey)

	// Two confirmed coins, both alice's.
	var c1, c2 Outpoint
	c1.TxID[0], c2.TxID[0] = 0xc1, 0xc2
	n := &Node{
		utxo:    UTXOSet{c1: {Value: 200_000, PubKeyHash: alice}, c2: {Value: 50_000, PubKeyHash: alice}},
		pool:    NewMempool(),
		minRate: 5,
		maxSize: 100_000,
	}
	fmt.Printf("=== node state ===\n  utxo set: 2 coins (200,000 and 50,000), both alice's\n")
	fmt.Printf("  policy:   min %d sat/byte, max %d bytes per transaction\n", n.minRate, n.maxSize)

	fmt.Println("\n=== ordinary arrivals ===")
	good := pay(aliceKey, []Outpoint{c1}, []TxOutput{n.utxo[c1]}, out(100_000, bobKey), out(95_720, aliceKey))
	try(n, "alice -> bob", good)

	// A child spending the change of a transaction still in the pool.
	changeOp := Outpoint{good.TxID(), 1}
	child := pay(aliceKey, []Outpoint{changeOp}, []TxOutput{good.Outputs[1]}, out(90_000, bobKey))
	try(n, "child of the above", child)
	fmt.Println("  A chain of unconfirmed transactions is normal. The second one's")
	fmt.Println("  input is not in the UTXO set at all — it is an output of the")
	fmt.Println("  first, which is still only in the pool.")

	fmt.Println("\n=== the rejections ===")

	// Same coin, different destination: a double-spend attempt.
	conflict := pay(aliceKey, []Outpoint{c1}, []TxOutput{n.utxo[c1]}, out(190_000, malloryKey))
	try(n, "double spend of c1", conflict)
	fmt.Println("  Not invalid — just second. Whichever a node hears first is the")
	fmt.Println("  one it keeps, which is exactly why zero-confirmation payments")
	fmt.Println("  are not safe (lesson 09, example 12).")

	try(n, "the same transaction twice", good)

	forged := pay(malloryKey, []Outpoint{c2}, []TxOutput{n.utxo[c2]}, out(45_000, malloryKey))
	try(n, "mallory signs alice's coin", forged)

	cheap := pay(aliceKey, []Outpoint{c2}, []TxOutput{n.utxo[c2]}, out(49_900, bobKey))
	try(n, "fee below the floor", cheap)

	rich := pay(aliceKey, []Outpoint{c2}, []TxOutput{n.utxo[c2]}, out(60_000, bobKey))
	try(n, "spending more than it has", rich)

	empty := &Transaction{Inputs: []TxInput{{Prev: c2}}}
	try(n, "no outputs", empty)

	fmt.Println("\n=== the orphan pool ===")
	// A grandchild arrives BEFORE its parent. This happens constantly: peers
	// relay in whatever order the network delivers.
	var future Outpoint
	future.TxID[0] = 0xf0
	futureOut := TxOutput{Value: 80_000, PubKeyHash: alice}
	unseen := pay(aliceKey, []Outpoint{future}, []TxOutput{futureOut}, out(75_000, bobKey))
	try(n, "child of an unseen tx", unseen)
	fmt.Printf("  orphan pool: %d held, waiting on %d outpoint(s)\n",
		len(n.pool.orphans), len(n.pool.waiting))

	fmt.Println("\n  ...and now the parent arrives:")
	n.utxo[future] = futureOut // as if a block or a peer delivered it
	fmt.Printf("  %-26s the missing output now exists\n", "parent seen")
	if got := n.promote(); len(got) > 0 {
		fmt.Printf("  %-26s -> orphans promoted: %v\n", "", got)
	}
	fmt.Printf("  orphan pool: %d held; mempool: %d transactions\n",
		len(n.pool.orphans), len(n.pool.byID))

	fmt.Println("\n=== the pool now ===")
	var entries []*Entry
	for _, e := range n.pool.byID {
		entries = append(entries, e)
	}
	sort.Slice(entries, func(i, j int) bool {
		a, b := entries[i], entries[j]
		if l, r := a.Fee*b.Size, b.Fee*a.Size; l != r {
			return l > r
		}
		return a.Name < b.Name
	})
	fmt.Printf("  %-24s %-8s %-10s %s\n", "transaction", "bytes", "fee", "sat/byte")
	for _, e := range entries {
		fmt.Printf("  %-24s %-8d %-10d %d\n", e.Name, e.Size, e.Fee, e.Fee/e.Size)
	}

	fmt.Println("\n=== the ordering is the security property ===")
	fmt.Println("  Every check before the signature check is a map lookup. That is")
	fmt.Println("  deliberate: anyone can send a node a megabyte of garbage, and it")
	fmt.Println("  must cost the node less to reject than it cost the attacker to")
	fmt.Println("  send. Verifying first would invert that.")
	fmt.Println()
	fmt.Println("  And note what is NOT here: none of this is consensus. A block")
	fmt.Println("  containing the double-spend, or the cheap transaction, would be")
	fmt.Println("  perfectly valid. Example 10 is about that gap.")
}
```

**Output:**

```
=== node state ===
  utxo set: 2 coins (200,000 and 50,000), both alice's
  policy:   min 5 sat/byte, max 100000 bytes per transaction

=== ordinary arrivals ===
  alice -> bob               accepted
  child of the above         accepted
  A chain of unconfirmed transactions is normal. The second one's
  input is not in the UTXO set at all — it is an output of the
  first, which is still only in the pool.

=== the rejections ===
  double spend of c1         REJECTED: conflicts with a transaction already in the pool: c10000000000…:0 is already claimed by alice -> bob
  Not invalid — just second. Whichever a node hears first is the
  one it keeps, which is exactly why zero-confirmation payments
  are not safe (lesson 09, example 12).
  the same transaction twice REJECTED: already in the pool
  mallory signs alice's coin REJECTED: signature does not verify: input 0: key does not open the output's lock
  fee below the floor        REJECTED: fee rate below the pool minimum: 0 < 5 sat/byte
  spending more than it has  REJECTED: outputs exceed inputs: in 50000, out 60000
  no outputs                 REJECTED: no inputs or no outputs

=== the orphan pool ===
  child of an unseen tx      REJECTED: parent not known — held in the orphan pool (1 missing)
  orphan pool: 1 held, waiting on 1 outpoint(s)

  ...and now the parent arrives:
  parent seen                the missing output now exists
                             -> orphans promoted: [child of an unseen tx]
  orphan pool: 0 held; mempool: 3 transactions

=== the pool now ===
  transaction              bytes    fee        sat/byte
  child of the above       182      5720       31
  child of an unseen tx    182      5000       27
  alice -> bob             214      4280       20

=== the ordering is the security property ===
  Every check before the signature check is a map lookup. That is
  deliberate: anyone can send a node a megabyte of garbage, and it
  must cost the node less to reject than it cost the attacker to
  send. Verifying first would invert that.

  And note what is NOT here: none of this is consensus. A block
  containing the double-spend, or the cheap transaction, would be
  perfectly valid. Example 10 is about that gap.
```

---

## 10. Policy is not consensus

`🟡 medium` · *Policy*

Two different words that both get called 'valid'. **Consensus** is global and binding; breaking it makes any block containing you invalid. **Policy** is local and advisory. The gap between them is where 'my transaction disappeared' lives.

**Steps:**

1. Run nine transactions through both validators and compare the verdicts.
2. Find the six that are consensus-valid and still will not be relayed.
3. Re-run them against a relaxed config and watch five of the six become fine.
4. Line up Bitcoin Core's knobs against geth's txpool flags.
5. Read the operational consequences — including why you can never rely on policy for safety.

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
// Two different words that both get called "valid".
//
//   CONSENSUS is global and binding. If a transaction breaks a consensus rule
//   then any block containing it is invalid, and every node on the network
//   agrees. Changing these rules is a fork.
//
//   POLICY is local and advisory. It decides what THIS node is willing to
//   keep in its mempool and pass on to its peers. Every node may set it
//   differently. Changing it is a config flag.
//
// The gap between them is where "my transaction disappeared" lives.
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

func Sign(t *Transaction, priv *ecdsa.PrivateKey, prevOuts []TxOutput) error {
	pub := crypto.CompressPubkey(&priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], priv)
		if err != nil {
			return err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return nil
}

func Verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: key does not open the output's lock", i)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: signature does not verify", i)
		}
	}
	return nil
}

type UTXOSet map[Outpoint]TxOutput

// -------------------------------------------------------- consensus rules

var (
	ErrNoInputs    = errors.New("no inputs")
	ErrNoOutputs   = errors.New("no outputs")
	ErrMissing     = errors.New("input not in the UTXO set")
	ErrDoubleSpend = errors.New("input named twice")
	ErrBadSig      = errors.New("signature does not verify")
	ErrNotCovered  = errors.New("outputs exceed inputs")
	ErrNegative    = errors.New("negative output value")
)

// Consensus is lesson 10's validator. Break any of this and no block can
// carry the transaction.
func Consensus(u UTXOSet, t *Transaction) error {
	if len(t.Inputs) == 0 {
		return ErrNoInputs
	}
	if len(t.Outputs) == 0 {
		return ErrNoOutputs
	}
	seen := map[Outpoint]bool{}
	prevOuts := make([]TxOutput, 0, len(t.Inputs))
	var in int64
	for _, i := range t.Inputs {
		if seen[i.Prev] {
			return ErrDoubleSpend
		}
		seen[i.Prev] = true
		o, ok := u[i.Prev]
		if !ok {
			return ErrMissing
		}
		prevOuts = append(prevOuts, o)
		in += o.Value
	}
	var out int64
	for _, o := range t.Outputs {
		if o.Value < 0 {
			return ErrNegative
		}
		out += o.Value
	}
	if out > in {
		return ErrNotCovered
	}
	if err := Verify(t, prevOuts); err != nil {
		return fmt.Errorf("%w: %v", ErrBadSig, err)
	}
	return nil
}

// ------------------------------------------------------------ policy rules

// Policy is this node's config. Every value here is a knob, and a different
// node may have it set differently — or not have the rule at all.
type Policy struct {
	MinFeeRate       int64 // sat/byte
	MaxSize          int64 // bytes
	DustLimit        int64
	MaxOutputs       int
	MaxAncestors     int
	AllowNonStandard bool
}

var (
	ErrTooCheap    = errors.New("below the minimum relay fee")
	ErrTooBig      = errors.New("larger than the policy limit")
	ErrDust        = errors.New("creates an output below the dust limit")
	ErrNonStandard = errors.New("non-standard: output lock is not a 20-byte key hash")
	ErrTooManyOuts = errors.New("more outputs than policy allows")
	ErrLongChain   = errors.New("too many unconfirmed ancestors")
)

func (p Policy) Check(u UTXOSet, t *Transaction, ancestors int) error {
	size := int64(len(t.Serialize()))
	if size > p.MaxSize {
		return fmt.Errorf("%w: %d > %d", ErrTooBig, size, p.MaxSize)
	}
	if len(t.Outputs) > p.MaxOutputs {
		return fmt.Errorf("%w: %d > %d", ErrTooManyOuts, len(t.Outputs), p.MaxOutputs)
	}
	for i, o := range t.Outputs {
		if len(o.PubKeyHash) != 20 && !p.AllowNonStandard {
			return fmt.Errorf("%w (output %d, %d bytes)", ErrNonStandard, i, len(o.PubKeyHash))
		}
		if o.Value < p.DustLimit {
			return fmt.Errorf("%w: output %d is %d < %d", ErrDust, i, o.Value, p.DustLimit)
		}
	}
	if ancestors > p.MaxAncestors {
		return fmt.Errorf("%w: %d > %d", ErrLongChain, ancestors, p.MaxAncestors)
	}
	var in, out int64
	for _, i := range t.Inputs {
		in += u[i.Prev].Value
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if rate := (in - out) / size; rate < p.MinFeeRate {
		return fmt.Errorf("%w: %d < %d sat/byte", ErrTooCheap, rate, p.MinFeeRate)
	}
	return nil
}

// -------------------------------------------------------------- scaffolding

const (
	aliceKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" // TEST ONLY
	bobKey   = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d" // TEST ONLY
)

func key(h string) *ecdsa.PrivateKey {
	k, err := crypto.HexToECDSA(h)
	if err != nil {
		panic(err)
	}
	return k
}
func pkhOf(h string) []byte { return hash160(crypto.CompressPubkey(&key(h).PublicKey)) }

func main() {
	alice, bob := pkhOf(aliceKey), pkhOf(bobKey)
	var coin Outpoint
	coin.TxID[0] = 0xc1
	u := UTXOSet{coin: {Value: 200_000, PubKeyHash: alice}}

	policy := Policy{MinFeeRate: 5, MaxSize: 4_000, DustLimit: 294, MaxOutputs: 40, MaxAncestors: 25}

	build := func(outs []TxOutput) *Transaction {
		t := &Transaction{Inputs: []TxInput{{Prev: coin}}, Outputs: outs}
		if err := Sign(t, key(aliceKey), []TxOutput{u[coin]}); err != nil {
			panic(err)
		}
		return t
	}

	// A bare 32-byte lock: no key hash, so no standard script template fits.
	weird := make([]byte, 32)
	for i := range weird {
		weird[i] = byte(i)
	}

	manyOuts := make([]TxOutput, 0, 60)
	for i := 0; i < 60; i++ {
		manyOuts = append(manyOuts, TxOutput{Value: 3_000, PubKeyHash: bob})
	}

	type row struct {
		name      string
		tx        *Transaction
		ancestors int
	}
	rows := []row{
		{"ordinary payment", build([]TxOutput{{100_000, bob}, {95_720, alice}}), 0},
		{"pays 1 sat/byte", build([]TxOutput{{100_000, bob}, {99_786, alice}}), 0},
		{"pays nothing at all", build([]TxOutput{{100_000, bob}, {100_000, alice}}), 0},
		{"200-sat change output", build([]TxOutput{{195_520, bob}, {200, alice}}), 0},
		{"non-standard 32-byte lock", build([]TxOutput{{195_720, bob}, {0, weird}}), 0},
		{"60 outputs", build(manyOuts), 0},
		{"26 unconfirmed ancestors", build([]TxOutput{{100_000, bob}, {95_720, alice}}), 26},
		{"spends more than it has", build([]TxOutput{{300_000, bob}}), 0},
		{"signature broken", brokenSig(u, coin, bob), 0},
	}

	fmt.Println("=== the same transactions, two verdicts ===")
	fmt.Printf("  %-28s %-12s %s\n", "transaction", "consensus", "policy (this node)")
	for _, r := range rows {
		c := Consensus(u, r.tx)
		p := policy.Check(u, r.tx, r.ancestors)
		cs, ps := "valid", "relay"
		if c != nil {
			cs = "INVALID"
		}
		if p != nil {
			ps = "drop: " + p.Error()
		}
		if c != nil {
			ps = "n/a"
		}
		fmt.Printf("  %-28s %-12s %s\n", r.name, cs, ps)
	}

	fmt.Println("\n=== read the middle rows again ===")
	fmt.Println("  Six of these are CONSENSUS-VALID and this node will not relay")
	fmt.Println("  them. A block containing any of them is perfectly good, and every")
	fmt.Println("  node on the network — including this one — would accept it.")
	fmt.Println()
	fmt.Println("  So the transaction is not rejected. It is IGNORED. It does not")
	fmt.Println("  propagate, it does not appear in any explorer, and it produces no")
	fmt.Println("  error anywhere the sender can see. From the wallet's point of")
	fmt.Println("  view it simply evaporated.")

	fmt.Println("\n=== the same node, a different config ===")
	relaxed := Policy{MinFeeRate: 0, MaxSize: 100_000, DustLimit: 0,
		MaxOutputs: 1000, MaxAncestors: 100, AllowNonStandard: true}
	fmt.Printf("  %-28s %-16s %s\n", "transaction", "strict policy", "relaxed policy")
	for _, r := range rows {
		if Consensus(u, r.tx) != nil {
			continue
		}
		a, b := policy.Check(u, r.tx, r.ancestors), relaxed.Check(u, r.tx, r.ancestors)
		fmt.Printf("  %-28s %-16s %s\n", r.name, verdict(a), verdict(b))
	}
	fmt.Println("\n  Nothing about the transactions changed. Policy is not a property")
	fmt.Println("  of a transaction; it is a property of whoever you asked.")

	fmt.Println("\n=== what this looks like in the two big implementations ===")
	fmt.Printf("  %-24s %-32s %s\n", "policy", "Bitcoin Core", "Ethereum (geth txpool)")
	for _, x := range [][3]string{
		{"minimum fee to relay", "minRelayTxFee, 1 sat/vB", "--txpool.pricelimit, 1 wei"},
		{"replacement price bump", "BIP-125 rules (example 14)", "--txpool.pricebump, 10%"},
		{"per-sender queue limit", "n/a", "--txpool.accountslots, 16"},
		{"total pool size", "-maxmempool, 300 MB", "--txpool.globalslots, 4096"},
		{"chain of unconfirmed", "25 ancestors, 25 descendants", "nonce gap -> 'queued'"},
		{"dust", "546 / 294 sat by script type", "n/a - no UTXOs"},
		{"exotic scripts", "IsStandard(): fixed templates", "n/a"},
		{"expiry", "-mempoolexpiry, 336 hours", "--txpool.lifetime, 3 hours"},
	} {
		fmt.Printf("  %-24s %-32s %s\n", x[0], x[1], x[2])
	}

	fmt.Println("\n=== why the distinction exists ===")
	fmt.Println("  Consensus rules are expensive to change and must be identical")
	fmt.Println("  everywhere, so they are kept minimal. Everything that is merely")
	fmt.Println("  a good idea — do not waste bandwidth on 1-satoshi outputs, do not")
	fmt.Println("  hold a 400-deep chain of unconfirmed transactions in RAM — goes")
	fmt.Println("  in policy, where it can be tuned per node and changed in a point")
	fmt.Println("  release without splitting the network.")
	fmt.Println()
	fmt.Println("  Policy is also where new features are TESTED. A rule is usually")
	fmt.Println("  policy for years before anyone proposes making it consensus.")

	fmt.Println("\n=== the operational consequences ===")
	fmt.Println("  - Never report 'invalid' to a user when you mean 'not relayed'.")
	fmt.Println("    They are different problems with different fixes.")
	fmt.Println("  - A transaction that no node will relay can still be mined, by")
	fmt.Println("    handing it to a miner directly. That is what Bitcoin")
	fmt.Println("    'accelerator' services and Ethereum's private order flow are:")
	fmt.Println("    a way round the relay layer, not round consensus.")
	fmt.Println("  - Which means you cannot rely on policy for safety. If something")
	fmt.Println("    must never happen, it has to be a consensus rule.")
	fmt.Println("  - And test against policy, not just consensus. A transaction your")
	fmt.Println("    unit tests accept may be one no peer will carry.")
}

func verdict(err error) string {
	if err == nil {
		return "relay"
	}
	return "drop"
}

func brokenSig(u UTXOSet, coin Outpoint, bob []byte) *Transaction {
	t := &Transaction{
		Inputs:  []TxInput{{Prev: coin}},
		Outputs: []TxOutput{{Value: 195_720, PubKeyHash: bob}},
	}
	if err := Sign(t, key(aliceKey), []TxOutput{u[coin]}); err != nil {
		panic(err)
	}
	t.Inputs[0].Signature[5] ^= 1
	return t
}
```

**Output:**

```
=== the same transactions, two verdicts ===
  transaction                  consensus    policy (this node)
  ordinary payment             valid        relay
  pays 1 sat/byte              valid        drop: below the minimum relay fee: 1 < 5 sat/byte
  pays nothing at all          valid        drop: below the minimum relay fee: 0 < 5 sat/byte
  200-sat change output        valid        drop: creates an output below the dust limit: output 1 is 200 < 294
  non-standard 32-byte lock    valid        drop: non-standard: output lock is not a 20-byte key hash (output 1, 32 bytes)
  60 outputs                   valid        drop: more outputs than policy allows: 60 > 40
  26 unconfirmed ancestors     valid        drop: too many unconfirmed ancestors: 26 > 25
  spends more than it has      INVALID      n/a
  signature broken             INVALID      n/a

=== read the middle rows again ===
  Six of these are CONSENSUS-VALID and this node will not relay
  them. A block containing any of them is perfectly good, and every
  node on the network — including this one — would accept it.

  So the transaction is not rejected. It is IGNORED. It does not
  propagate, it does not appear in any explorer, and it produces no
  error anywhere the sender can see. From the wallet's point of
  view it simply evaporated.

=== the same node, a different config ===
  transaction                  strict policy    relaxed policy
  ordinary payment             relay            relay
  pays 1 sat/byte              drop             relay
  pays nothing at all          drop             relay
  200-sat change output        drop             relay
  non-standard 32-byte lock    drop             relay
  60 outputs                   drop             relay
  26 unconfirmed ancestors     drop             relay

  Nothing about the transactions changed. Policy is not a property
  of a transaction; it is a property of whoever you asked.

=== what this looks like in the two big implementations ===
  policy                   Bitcoin Core                     Ethereum (geth txpool)
  minimum fee to relay     minRelayTxFee, 1 sat/vB          --txpool.pricelimit, 1 wei
  replacement price bump   BIP-125 rules (example 14)       --txpool.pricebump, 10%
  per-sender queue limit   n/a                              --txpool.accountslots, 16
  total pool size          -maxmempool, 300 MB              --txpool.globalslots, 4096
  chain of unconfirmed     25 ancestors, 25 descendants     nonce gap -> 'queued'
  dust                     546 / 294 sat by script type     n/a - no UTXOs
  exotic scripts           IsStandard(): fixed templates    n/a
  expiry                   -mempoolexpiry, 336 hours        --txpool.lifetime, 3 hours

=== why the distinction exists ===
  Consensus rules are expensive to change and must be identical
  everywhere, so they are kept minimal. Everything that is merely
  a good idea — do not waste bandwidth on 1-satoshi outputs, do not
  hold a 400-deep chain of unconfirmed transactions in RAM — goes
  in policy, where it can be tuned per node and changed in a point
  release without splitting the network.

  Policy is also where new features are TESTED. A rule is usually
  policy for years before anyone proposes making it consensus.

=== the operational consequences ===
  - Never report 'invalid' to a user when you mean 'not relayed'.
    They are different problems with different fixes.
  - A transaction that no node will relay can still be mined, by
    handing it to a miner directly. That is what Bitcoin
    'accelerator' services and Ethereum's private order flow are:
    a way round the relay layer, not round consensus.
  - Which means you cannot rely on policy for safety. If something
    must never happen, it has to be a consensus rule.
  - And test against policy, not just consensus. A transaction your
    unit tests accept may be one no peer will carry.
```

---

## 11. Estimating a fee from history

`🟡 medium` · *Fees*

There is no oracle. A node predicts what a transaction must pay by bucketing recent confirmations by fee rate and recording how long each bucket waited. Then a spike arrives and the whole thing is revealed for what it is: a prediction about other people.

**Steps:**

1. Simulate 600 blocks of bursty demand against a fixed capacity.
2. Bucket the confirmations and read the table: 8-12 sat/byte clears in one block, 3-4 needs twenty-five.
3. Turn that into estimates for six confirmation targets.
4. Triple the demand and watch three transactions paying the old estimates never confirm at all.
5. Watch the estimator relearn — 8 sat/byte becomes 21 — and read what a wallet should do instead of trusting it.

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
)

// ===========================================================================
// Fee estimation. There is no oracle: a node predicts what a transaction must
// pay by watching what recent transactions actually paid and how long they
// waited.
//
// The method here is Bitcoin Core's, simplified: bucket confirmed
// transactions by fee rate, and for each bucket record how many confirmed
// within N blocks. To estimate for a target of N, walk the buckets upwards
// and return the first one that hits the success threshold.
//
// Everything is seeded, so the numbers below reproduce exactly.
// ===========================================================================

const (
	blockCapacity = 4_000 // bytes per block
	txSize        = 200   // every transaction the same size, to keep it readable
	successRate   = 85    // percent; Core uses 85% for "economical", 95% for "conservative"
	horizon       = 30    // blocks tracked
)

type Tx struct {
	Rate    int64 // sat/byte
	Arrived int   // block height it appeared at
	Probe   int   // 0 for ordinary traffic; >0 tags a transaction we follow
}

// --------------------------------------------------------------- the buckets

// Fee-rate buckets. Core uses a 1.05 geometric series with ~200 buckets; this
// is the same idea with 12, so the table fits on a screen.
var bucketFloor = []int64{1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233}

func bucketOf(rate int64) int {
	i := sort.Search(len(bucketFloor), func(i int) bool { return bucketFloor[i] > rate }) - 1
	if i < 0 {
		return 0
	}
	return i
}

// Tracker is the estimator's memory: per bucket, how many transactions
// confirmed within each number of blocks.
type Tracker struct {
	total  []int
	within [][]int // within[bucket][blocks] = count confirmed in <= blocks
}

func NewTracker() *Tracker {
	t := &Tracker{total: make([]int, len(bucketFloor)), within: make([][]int, len(bucketFloor))}
	for i := range t.within {
		t.within[i] = make([]int, horizon+1)
	}
	return t
}

func (t *Tracker) Record(rate int64, waited int) {
	b := bucketOf(rate)
	t.total[b]++
	if waited > horizon {
		return
	}
	for n := waited; n <= horizon; n++ {
		t.within[b][n]++
	}
}

// Estimate returns the lowest fee rate whose bucket confirmed at least
// `successRate` percent of its transactions within `target` blocks.
func (t *Tracker) Estimate(target int) (int64, bool) {
	if target > horizon {
		target = horizon
	}
	for b := range bucketFloor {
		if t.total[b] < 20 { // not enough data to say anything
			continue
		}
		if t.within[b][target]*100 >= t.total[b]*successRate {
			return bucketFloor[b], true
		}
	}
	return 0, false
}

// ------------------------------------------------------------ the simulation

type sim struct {
	r      *rand.Rand
	pool   []Tx
	height int
	track  *Tracker
	// stats
	confirmed int
}

// arrive adds n transactions with fee rates drawn from a long-tailed
// distribution — most people accept the default, a few pay a lot.
func (s *sim) arrive(n int) {
	for i := 0; i < n; i++ {
		var rate int64
		switch u := s.r.Float64(); {
		case u < 0.40:
			rate = 1 + int64(s.r.Intn(5)) // 1-5: the defaults and the patient
		case u < 0.75:
			rate = 6 + int64(s.r.Intn(10)) // 6-15: ordinary payments
		case u < 0.93:
			rate = 16 + int64(s.r.Intn(25)) // 16-40: in a hurry
		default:
			rate = 41 + int64(s.r.Intn(160)) // whales and mistakes
		}
		s.pool = append(s.pool, Tx{Rate: rate, Arrived: s.height})
	}
}

// demand is what makes this a market: most blocks are under-subscribed, and
// every so often a burst arrives and the backlog takes several blocks to
// clear. A constant arrival rate would produce a single cut-off fee rate and
// nothing to estimate.
func (s *sim) demand() int {
	n := 8 + s.r.Intn(16) // 8-23, comfortably under capacity most of the time
	if s.r.Float64() < 0.15 {
		n += 40 // a burst
	}
	return n
}

// mine fills one block greedily by fee rate (example 3) and records how long
// each included transaction waited.
func (s *sim) mine() {
	sort.SliceStable(s.pool, func(i, j int) bool {
		if s.pool[i].Rate != s.pool[j].Rate {
			return s.pool[i].Rate > s.pool[j].Rate
		}
		return s.pool[i].Arrived < s.pool[j].Arrived
	})
	room := blockCapacity
	cut := 0
	for cut < len(s.pool) && room >= txSize {
		s.track.Record(s.pool[cut].Rate, s.height-s.pool[cut].Arrived)
		s.confirmed++
		room -= txSize
		cut++
	}
	s.pool = s.pool[cut:]
	s.height++
}

func rateOrNone(rate int64, ok bool) string {
	if !ok {
		return "no bucket qualifies"
	}
	return fmt.Sprintf("%d sat/byte", rate)
}

func main() {
	s := &sim{r: rand.New(rand.NewSource(7)), track: NewTracker()}

	// Steady state: 22 arrivals per block against room for 20. Slightly more
	// demand than supply, which is what a real fee market looks like.
	arrived := 0
	for i := 0; i < 600; i++ {
		n := s.demand()
		arrived += n
		s.arrive(n)
		s.mine()
	}

	fmt.Printf("=== 600 blocks of ordinary demand ===\n")
	fmt.Printf("  block capacity   %d transactions\n", blockCapacity/txSize)
	fmt.Printf("  arrivals         %d in total, %.1f per block on average\n",
		arrived, float64(arrived)/600)
	fmt.Printf("  confirmed        %d\n", s.confirmed)
	fmt.Printf("  still waiting    %d\n", len(s.pool))

	fmt.Println("\n=== what the tracker learned ===")
	fmt.Printf("  %-14s %-9s %-10s %-10s %-10s %s\n",
		"bucket", "seen", "<=1 blk", "<=3", "<=6", "<=25")
	for b, floor := range bucketFloor {
		if s.track.total[b] == 0 {
			continue
		}
		pct := func(n int) string {
			return fmt.Sprintf("%d%%", n*100/s.track.total[b])
		}
		hi := "+"
		if b+1 < len(bucketFloor) {
			hi = fmt.Sprintf("-%d", bucketFloor[b+1]-1)
		}
		fmt.Printf("  %-14s %-9d %-10s %-10s %-10s %s\n",
			fmt.Sprintf("%d%s sat/B", floor, hi), s.track.total[b],
			pct(s.track.within[b][1]), pct(s.track.within[b][3]),
			pct(s.track.within[b][6]), pct(s.track.within[b][25]))
	}

	fmt.Println("\n=== the estimates that come out ===")
	fmt.Printf("  %-18s %s\n", "target", fmt.Sprintf("lowest rate that clears %d%% of the time", successRate))
	for _, target := range []int{1, 2, 3, 6, 12, 25} {
		rate, ok := s.track.Estimate(target)
		if !ok {
			fmt.Printf("  %-18s no bucket qualifies\n", fmt.Sprintf("%d block(s)", target))
			continue
		}
		fmt.Printf("  %-18s %d sat/byte\n", fmt.Sprintf("%d block(s)", target), rate)
	}
	fmt.Println("\n  Read the columns above and the estimates fall out of them: the")
	fmt.Println("  8-12 bucket clears 90% of the time in one block, the 5-7 bucket")
	fmt.Println("  needs three, and the 3-4 bucket needs twenty-five. Patience is")
	fmt.Println("  worth about 3x here. In a real fee spike it is worth 50x, which")
	fmt.Println("  is why the next section matters.")

	// ---------------------------------------------------------------- spike
	fmt.Println("\n=== then demand triples ===")
	slowRate, _ := s.track.Estimate(25)
	sixBlock, _ := s.track.Estimate(6)
	oneBlock, _ := s.track.Estimate(1)

	spike := &sim{r: rand.New(rand.NewSource(11)), track: NewTracker(), height: s.height}
	spike.pool = append([]Tx(nil), s.pool...)

	// Send one transaction at each of the old estimates, right as the spike
	// starts, and watch how long they actually take.
	type probe struct {
		name   string
		rate   int64
		sent   int
		waited int
		done   bool
	}
	probes := []*probe{
		{name: "paid the 25-block estimate", rate: slowRate},
		{name: "paid the 6-block estimate", rate: sixBlock},
		{name: "paid the 1-block estimate", rate: oneBlock},
		{name: "paid 3x the 1-block estimate", rate: oneBlock * 3},
	}
	for i, p := range probes {
		p.sent = spike.height
		spike.pool = append(spike.pool, Tx{Rate: p.rate, Arrived: spike.height, Probe: i + 1})
	}

	const spikeBlocks = 60
	for i := 0; i < spikeBlocks; i++ {
		spike.arrive(3 * spike.demand())
		sort.SliceStable(spike.pool, func(a, b int) bool {
			if spike.pool[a].Rate != spike.pool[b].Rate {
				return spike.pool[a].Rate > spike.pool[b].Rate
			}
			return spike.pool[a].Arrived < spike.pool[b].Arrived
		})
		room := blockCapacity / txSize
		if room > len(spike.pool) {
			room = len(spike.pool)
		}
		for _, t := range spike.pool[:room] {
			spike.track.Record(t.Rate, spike.height-t.Arrived)
			if t.Probe > 0 {
				p := probes[t.Probe-1]
				p.done, p.waited = true, spike.height-p.sent
			}
		}
		spike.pool = spike.pool[room:]
		spike.height++
	}

	fmt.Printf("  arrivals triple; capacity is still %d per block\n\n", blockCapacity/txSize)
	fmt.Printf("  %-32s %-10s %s\n", "transaction", "rate", "blocks waited")
	for _, p := range probes {
		if !p.done {
			fmt.Printf("  %-32s %-10d still unconfirmed after %d\n", p.name, p.rate, spikeBlocks)
			continue
		}
		fmt.Printf("  %-32s %-10d %d\n", p.name, p.rate, p.waited)
	}

	after6, ok6 := spike.track.Estimate(6)
	after1, ok1 := spike.track.Estimate(1)
	fmt.Println("\n  what the estimator says once it has SEEN the spike:")
	fmt.Printf("    1 block : %s (was %d)\n", rateOrNone(after1, ok1), oneBlock)
	fmt.Printf("    6 blocks: %s (was %d)\n", rateOrNone(after6, ok6), sixBlock)

	fmt.Println("\n  The estimate was not wrong when it was made. It described the")
	fmt.Println("  last 600 blocks accurately, and then the world changed. A fee")
	fmt.Println("  estimate is a PREDICTION ABOUT OTHER PEOPLE, and there is no")
	fmt.Println("  version of it that is a promise.")

	fmt.Println("\n=== so what does a wallet actually do ===")
	fmt.Println("  - estimate for a target, and SHOW the target, not a fee: 'about")
	fmt.Println("    30 minutes' is honest, '4200 sat' pretends to certainty")
	fmt.Println("  - always signal replaceability, so the estimate can be revised")
	fmt.Println("    later (example 14)")
	fmt.Println("  - let the receiver fix it too, with child-pays-for-parent")
	fmt.Println("    (example 16)")
	fmt.Println("  - never round a fee UP to look tidy; round the RATE and let the")
	fmt.Println("    size decide (example 3)")
	fmt.Println("  - and remember the pool is not global: your node's estimate comes")
	fmt.Println("    from what YOUR node saw (example 10)")
}
```

**Output:**

```
=== 600 blocks of ordinary demand ===
  block capacity   20 transactions
  arrivals         12761 in total, 21.3 per block on average
  confirmed        11997
  still waiting    764

=== what the tracker learned ===
  bucket         seen      <=1 blk    <=3        <=6        <=25
  1-1 sat/B      465       1%         1%         1%         1%
  2-2 sat/B      840       14%        19%        29%        46%
  3-4 sat/B      2031      35%        49%        61%        95%
  5-7 sat/B      1902      66%        85%        95%        100%
  8-12 sat/B     2193      90%        99%        100%       100%
  13-20 sat/B    1826      99%        100%       100%       100%
  21-33 sat/B    1159      100%       100%       100%       100%
  34-54 sat/B    735       100%       100%       100%       100%
  55-88 sat/B    203       100%       100%       100%       100%
  89-143 sat/B   319       100%       100%       100%       100%
  144-232 sat/B  324       100%       100%       100%       100%

=== the estimates that come out ===
  target             lowest rate that clears 85% of the time
  1 block(s)         8 sat/byte
  2 block(s)         8 sat/byte
  3 block(s)         5 sat/byte
  6 block(s)         5 sat/byte
  12 block(s)        5 sat/byte
  25 block(s)        3 sat/byte

  Read the columns above and the estimates fall out of them: the
  8-12 bucket clears 90% of the time in one block, the 5-7 bucket
  needs three, and the 3-4 bucket needs twenty-five. Patience is
  worth about 3x here. In a real fee spike it is worth 50x, which
  is why the next section matters.

=== then demand triples ===
  arrivals triple; capacity is still 20 per block

  transaction                      rate       blocks waited
  paid the 25-block estimate       3          still unconfirmed after 60
  paid the 6-block estimate        5          still unconfirmed after 60
  paid the 1-block estimate        8          still unconfirmed after 60
  paid 3x the 1-block estimate     24         0

  what the estimator says once it has SEEN the spike:
    1 block : 21 sat/byte (was 8)
    6 blocks: 21 sat/byte (was 5)

  The estimate was not wrong when it was made. It described the
  last 600 blocks accurately, and then the world changed. A fee
  estimate is a PREDICTION ABOUT OTHER PEOPLE, and there is no
  version of it that is a promise.

=== so what does a wallet actually do ===
  - estimate for a target, and SHOW the target, not a fee: 'about
    30 minutes' is honest, '4200 sat' pretends to certainty
  - always signal replaceability, so the estimate can be revised
    later (example 14)
  - let the receiver fix it too, with child-pays-for-parent
    (example 16)
  - never round a fee UP to look tidy; round the RATE and let the
    size decide (example 3)
  - and remember the pool is not global: your node's estimate comes
    from what YOUR node saw (example 10)
```

---

## 12. Filling a block

`🟡 medium` · *Block assembly*

Choosing a block's contents is the knapsack problem, which is NP-hard — and everyone ships greedy anyway. This measures exactly how much that costs: brute-force the true optimum over 65,536 subsets and compare.

**Steps:**

1. Run greedy-by-fee-rate and brute force on the same mempool.
2. Repeat over 500 random pools and get the distribution: exact half the time, 98% on average.
3. See what greedy has to get right — and how much SKIP-do-not-STOP is worth.
4. Read the other three requirements: ancestors, the coinbase reserve, and deterministic tie-breaking.
5. Scale the numbers up to a real 4,000,000-weight block and a 50,000-transaction pool.

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
)

// ===========================================================================
// Block assembly: pick the subset of the mempool that maximises fees, subject
// to a size cap. That is the knapsack problem, which is NP-hard — and yet
// every miner solves it in milliseconds, because greedy-by-fee-rate is close
// enough that nobody has ever bothered with anything better.
//
// This example measures exactly how close.
// ===========================================================================

const (
	blockCap     = 4_000 // bytes available to transactions
	coinbaseSize = 200   // reserved for the coinbase (lesson 10)
	usable       = blockCap - coinbaseSize
)

type Tx struct {
	Name string
	Size int64
	Fee  int64
}

func (t Tx) Rate() float64 { return float64(t.Fee) / float64(t.Size) }

// ------------------------------------------------------------------ greedy

// Greedy sorts by fee rate and takes anything that still fits. Note the
// "still fits" rather than "stop": skipping over one big transaction to take
// three small ones is most of what makes this work as well as it does.
func Greedy(pool []Tx, capacity int64) (fees, used int64, picked []Tx) {
	sorted := append([]Tx(nil), pool...)
	sort.SliceStable(sorted, func(i, j int) bool {
		if a, b := sorted[i].Fee*sorted[j].Size, sorted[j].Fee*sorted[i].Size; a != b {
			return a > b
		}
		return sorted[i].Size < sorted[j].Size
	})
	for _, t := range sorted {
		if used+t.Size > capacity {
			continue
		}
		used += t.Size
		fees += t.Fee
		picked = append(picked, t)
	}
	return
}

// ------------------------------------------------------------- brute force

// Optimal enumerates all 2^n subsets. Fine for n <= 20 and a demo; hopeless
// for the 50,000-transaction mempool a real miner has.
func Optimal(pool []Tx, capacity int64) (fees, used int64, picked []Tx) {
	n := len(pool)
	best := -1
	var bestFee, bestUsed int64
	for mask := 0; mask < 1<<n; mask++ {
		var f, u int64
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				u += pool[i].Size
				if u > capacity {
					break
				}
				f += pool[i].Fee
			}
		}
		if u <= capacity && f > bestFee {
			best, bestFee, bestUsed = mask, f, u
		}
	}
	if best < 0 {
		return 0, 0, nil
	}
	for i := 0; i < n; i++ {
		if best&(1<<i) != 0 {
			picked = append(picked, pool[i])
		}
	}
	return bestFee, bestUsed, picked
}

// --------------------------------------------------------------- generation

func randomPool(r *rand.Rand, n int) []Tx {
	pool := make([]Tx, n)
	for i := range pool {
		size := int64(150 + r.Intn(1200))
		rate := int64(1 + r.Intn(60))
		pool[i] = Tx{fmt.Sprintf("tx%02d", i), size, size * rate}
	}
	return pool
}

func names(txs []Tx) []string {
	out := make([]string, len(txs))
	for i, t := range txs {
		out[i] = t.Name
	}
	sort.Strings(out)
	return out
}

func main() {
	r := rand.New(rand.NewSource(3))
	pool := randomPool(r, 16)
	sort.SliceStable(pool, func(i, j int) bool { return pool[i].Rate() > pool[j].Rate() })

	fmt.Printf("=== the mempool: %d transactions ===\n", len(pool))
	fmt.Printf("  %-8s %-8s %-10s %s\n", "tx", "bytes", "fee", "sat/byte")
	var totalSize, totalFee int64
	for _, t := range pool {
		totalSize += t.Size
		totalFee += t.Fee
		fmt.Printf("  %-8s %-8d %-10d %.0f\n", t.Name, t.Size, t.Fee, t.Rate())
	}
	fmt.Printf("\n  total %d bytes, %d sat — but the block holds %d bytes\n",
		totalSize, totalFee, usable)
	fmt.Printf("  (%d cap, %d reserved for the coinbase)\n", blockCap, coinbaseSize)

	fmt.Println("\n=== greedy by fee rate ===")
	gf, gu, gp := Greedy(pool, usable)
	fmt.Printf("  %d transactions, %d bytes used, %d sat\n", len(gp), gu, gf)
	fmt.Printf("  %v\n", names(gp))

	fmt.Println("\n=== the true optimum, by brute force over 65,536 subsets ===")
	of, ou, op := Optimal(pool, usable)
	fmt.Printf("  %d transactions, %d bytes used, %d sat\n", len(op), ou, of)
	fmt.Printf("  %v\n", names(op))
	fmt.Printf("\n  greedy captured %.2f%% of the optimum, leaving %d sat on the table\n",
		float64(gf)*100/float64(of), of-gf)

	// ---------------------------------------------------------------------
	fmt.Println("\n=== over 500 random mempools ===")
	r2 := rand.New(rand.NewSource(99))
	var worst = 100.0
	var sum float64
	exact := 0
	const trials = 500
	for i := 0; i < trials; i++ {
		p := randomPool(r2, 14)
		g, _, _ := Greedy(p, usable)
		o, _, _ := Optimal(p, usable)
		pct := float64(g) * 100 / float64(o)
		sum += pct
		if pct < worst {
			worst = pct
		}
		if g == o {
			exact++
		}
	}
	fmt.Printf("  greedy hit the exact optimum in %d of %d cases (%.0f%%)\n", exact, trials, float64(exact)*100/trials)
	fmt.Printf("  average capture      %.3f%% of optimal\n", sum/trials)
	fmt.Printf("  worst case seen      %.2f%%\n", worst)
	fmt.Println()
	fmt.Println("  That is the whole answer to 'why does everyone ship greedy'.")
	fmt.Println("  The loss is a fraction of a percent, the algorithm is one sort,")
	fmt.Println("  and the exact version is exponential in a set of 50,000.")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== what greedy actually has to get right ===")
	fmt.Println("  1. SKIP, do not STOP. When the best remaining transaction does")
	fmt.Println("     not fit, keep walking — smaller ones behind it still do.")
	fmt.Println("     Stopping instead of skipping is the single most common bug")
	fmt.Println("     here, and it is worth a lot:")
	stopFees, stopUsed := stopAtFirst(pool, usable)
	fmt.Printf("       stop at the first that does not fit: %d sat, %d bytes\n", stopFees, stopUsed)
	fmt.Printf("       skip and carry on:                   %d sat, %d bytes\n", gf, gu)

	fmt.Println()
	fmt.Println("  2. ANCESTORS. A transaction whose parent is still unconfirmed")
	fmt.Println("     cannot go in without its parent, and must go in AFTER it.")
	fmt.Println("     So the unit of selection is not a transaction but a PACKAGE,")
	fmt.Println("     ranked by the package's combined fee rate — which is how a")
	fmt.Println("     rich child drags a poor parent into a block (example 16).")
	fmt.Println()
	fmt.Println("  3. THE RESERVE. The coinbase has to fit, and so does whatever")
	fmt.Println("     the block header and the transaction count cost. Assembling")
	fmt.Println("     to the full cap and then adding the coinbase produces an")
	fmt.Println("     oversize block that every node rejects — after you mined it.")
	fmt.Println()
	fmt.Println("  4. DETERMINISM. Two transactions with the same fee rate must be")
	fmt.Println("     ordered by something stable, or the same mempool produces")
	fmt.Println("     different blocks on different runs and nothing is testable.")

	fmt.Println("\n=== the real numbers ===")
	fmt.Println("  Bitcoin: 4,000,000 weight units, a mempool of 20,000-100,000")
	fmt.Println("  transactions, and a new template built every time the pool")
	fmt.Println("  changes. Core keeps the pool pre-sorted by ancestor fee rate so")
	fmt.Println("  assembly is close to a linear scan.")
	fmt.Println()
	fmt.Println("  Ethereum: 30,000,000 gas, and the ordering problem is worse —")
	fmt.Println("  transactions from one sender must go in nonce order, and the")
	fmt.Println("  VALUE of an ordering depends on what the transactions DO to each")
	fmt.Println("  other. That is where MEV comes from (lesson 40), and why block")
	fmt.Println("  building there is an auction rather than a sort.")
}

// stopAtFirst is the buggy version: it stops at the first transaction that
// does not fit instead of skipping it.
func stopAtFirst(pool []Tx, capacity int64) (fees, used int64) {
	sorted := append([]Tx(nil), pool...)
	sort.SliceStable(sorted, func(i, j int) bool {
		if a, b := sorted[i].Fee*sorted[j].Size, sorted[j].Fee*sorted[i].Size; a != b {
			return a > b
		}
		return sorted[i].Size < sorted[j].Size
	})
	for _, t := range sorted {
		if used+t.Size > capacity {
			break
		}
		used += t.Size
		fees += t.Fee
	}
	return
}
```

**Output:**

```
=== the mempool: 16 transactions ===
  tx       bytes    fee        sat/byte
  tx03     1266     73428      58
  tx07     756      40824      54
  tx10     783      34452      44
  tx14     1062     43542      41
  tx02     1127     45080      40
  tx08     624      24336      39
  tx01     246      7626       31
  tx05     1016     31496      31
  tx12     391      11730      30
  tx04     894      25032      28
  tx09     824      18952      23
  tx11     959      22057      23
  tx00     358      6444       18
  tx13     597      8955       15
  tx06     731      5848       8
  tx15     584      4672       8

  total 12218 bytes, 404474 sat — but the block holds 3800 bytes
  (4000 cap, 200 reserved for the coinbase)

=== greedy by fee rate ===
  5 transactions, 3675 bytes used, 180666 sat
  [tx01 tx03 tx07 tx08 tx10]

=== the true optimum, by brute force over 65,536 subsets ===
  4 transactions, 3773 bytes used, 183668 sat
  [tx02 tx03 tx07 tx08]

  greedy captured 98.37% of the optimum, leaving 3002 sat on the table

=== over 500 random mempools ===
  greedy hit the exact optimum in 256 of 500 cases (51%)
  average capture      98.160% of optimal
  worst case seen      84.73%

  That is the whole answer to 'why does everyone ship greedy'.
  The loss is a fraction of a percent, the algorithm is one sort,
  and the exact version is exponential in a set of 50,000.

=== what greedy actually has to get right ===
  1. SKIP, do not STOP. When the best remaining transaction does
     not fit, keep walking — smaller ones behind it still do.
     Stopping instead of skipping is the single most common bug
     here, and it is worth a lot:
       stop at the first that does not fit: 148704 sat, 2805 bytes
       skip and carry on:                   180666 sat, 3675 bytes

  2. ANCESTORS. A transaction whose parent is still unconfirmed
     cannot go in without its parent, and must go in AFTER it.
     So the unit of selection is not a transaction but a PACKAGE,
     ranked by the package's combined fee rate — which is how a
     rich child drags a poor parent into a block (example 16).

  3. THE RESERVE. The coinbase has to fit, and so does whatever
     the block header and the transaction count cost. Assembling
     to the full cap and then adding the coinbase produces an
     oversize block that every node rejects — after you mined it.

  4. DETERMINISM. Two transactions with the same fee rate must be
     ordered by something stable, or the same mempool produces
     different blocks on different runs and nothing is testable.

=== the real numbers ===
  Bitcoin: 4,000,000 weight units, a mempool of 20,000-100,000
  transactions, and a new template built every time the pool
  changes. Core keeps the pool pre-sorted by ancestor fee rate so
  assembly is close to a linear scan.

  Ethereum: 30,000,000 gas, and the ordering problem is worse —
  transactions from one sender must go in nonce order, and the
  VALUE of an ordering depends on what the transactions DO to each
  other. That is where MEV comes from (lesson 40), and why block
  building there is an auction rather than a sort.
```

---

## 13. Eviction, the floor, and expiry

`🟡 medium` · *The mempool*

A mempool that never forgets is a memory-exhaustion attack with extra steps. Three defences: evict the lowest fee rate, raise a dynamic floor so the same traffic does not come straight back, and expire what is never going to be mined.

**Steps:**

1. Build the min-heap on fee rate with `container/heap` — Pop gives you exactly what to throw away.
2. Overflow the budget and watch the two cheapest go, and the floor rise behind them.
3. Send a transaction that would have been fine ten minutes ago and watch the floor reject it.
4. Decay the floor on a timer, so one busy afternoon does not lock the node out permanently.
5. Evict a cheap parent and watch its 30-sat/byte children go with it — and read why they had to.
6. Expire a two-week-old transaction and read why that is about honesty, not memory.

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

// ===========================================================================
// A mempool that never forgets is a memory-exhaustion attack with extra
// steps: anyone can send an endless stream of cheap, valid transactions.
//
// So the pool has a byte budget, and three ways to stay under it:
//
//   EVICTION  drop the lowest fee rate first, when full
//   THE FLOOR raise the minimum fee to what was just evicted, so the same
//             traffic does not come straight back in
//   EXPIRY    drop anything older than N blocks, however good its fee
//
// The floor is the interesting one: it makes the pool self-regulating, and
// it is why "my fee was fine an hour ago" is a real support ticket.
// ===========================================================================

const (
	maxBytes     = 3_000 // deliberately tiny; Bitcoin Core defaults to 300 MB
	expiryBlocks = 20    // Core: 336 hours
	halfLife     = 10    // blocks; the floor halves this often
)

type Entry struct {
	Name   string
	Parent string // "" if it spends only confirmed outputs
	Size   int64
	Fee    int64
	Added  int64 // block height
	index  int   // maintained by container/heap
}

// Rate comparison in integers, so there is no rounding to disagree about.
func lower(a, b *Entry) bool {
	l, r := a.Fee*b.Size, b.Fee*a.Size
	if l != r {
		return l < r
	}
	return a.Added < b.Added // among equals, the older one goes first
}

// ------------------------------------------------------- the min-heap

// feeHeap is a min-heap on fee rate: Pop gives the transaction a miner would
// take LAST, which is exactly the one to throw away first.
type feeHeap []*Entry

func (h feeHeap) Len() int           { return len(h) }
func (h feeHeap) Less(i, j int) bool { return lower(h[i], h[j]) }
func (h feeHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i]; h[i].index = i; h[j].index = j }
func (h *feeHeap) Push(x any)        { e := x.(*Entry); e.index = len(*h); *h = append(*h, e) }
func (h *feeHeap) Pop() any {
	old := *h
	n := len(old)
	e := old[n-1]
	old[n-1] = nil
	e.index = -1 // so a later remove() knows it is no longer in the heap
	*h = old[:n-1]
	return e
}

// --------------------------------------------------------------- the pool

type Mempool struct {
	entries  map[string]*Entry
	children map[string][]string
	h        feeHeap
	bytes    int64
	floor    int64 // sat/byte; raised by eviction, decayed by time
	floorSet int64 // block height at which the floor was last raised
	height   int64
	log      []string
}

func NewMempool() *Mempool {
	return &Mempool{entries: map[string]*Entry{}, children: map[string][]string{}}
}

// Floor returns the current minimum, decayed since it was last raised.
// Core halves it every twelve hours; the same idea, faster.
func (m *Mempool) Floor() int64 {
	f := m.floor
	for h := m.floorSet; h+halfLife <= m.height && f > 0; h += halfLife {
		f /= 2
	}
	return f
}

func (m *Mempool) Add(e *Entry) error {
	rate := e.Fee / e.Size
	if f := m.Floor(); rate < f {
		return fmt.Errorf("rate %d is below the pool floor of %d", rate, f)
	}
	if e.Parent != "" {
		if _, ok := m.entries[e.Parent]; !ok {
			return fmt.Errorf("parent %s is not in the pool", e.Parent)
		}
		m.children[e.Parent] = append(m.children[e.Parent], e.Name)
	}
	e.Added = m.height
	m.entries[e.Name] = e
	heap.Push(&m.h, e)
	m.bytes += e.Size
	m.trim()
	return nil
}

// trim evicts until the pool is under budget, raising the floor as it goes.
func (m *Mempool) trim() {
	for m.bytes > maxBytes && m.h.Len() > 0 {
		worst := heap.Pop(&m.h).(*Entry)
		if _, live := m.entries[worst.Name]; !live {
			continue // already removed as somebody's descendant
		}
		rate := worst.Fee / worst.Size
		n := m.remove(worst.Name)

		// The new floor is what we just had to throw away, plus one. Anything
		// cheaper would only be evicted again on arrival.
		if rate+1 > m.floor {
			m.floor, m.floorSet = rate+1, m.height
		}
		extra := ""
		if n > 1 {
			extra = fmt.Sprintf(" (+%d descendant(s))", n-1)
		}
		m.log = append(m.log, fmt.Sprintf("evicted %s at %d sat/byte%s; floor now %d",
			worst.Name, rate, extra, m.floor))
	}
}

// remove drops an entry and everything descended from it. Keeping a child
// whose parent has gone would leave an unspendable orphan in the pool.
func (m *Mempool) remove(name string) int {
	e, ok := m.entries[name]
	if !ok {
		return 0
	}
	n := 1
	for _, c := range m.children[name] {
		n += m.remove(c)
	}
	delete(m.children, name)
	delete(m.entries, name)
	m.bytes -= e.Size
	if e.index >= 0 && e.index < m.h.Len() && m.h[e.index] == e {
		heap.Remove(&m.h, e.index)
	}
	return n
}

// Expire drops everything older than expiryBlocks, whatever it paid.
func (m *Mempool) Expire() []string {
	var gone []string
	for _, e := range m.sorted() {
		if m.height-e.Added >= expiryBlocks {
			m.remove(e.Name)
			gone = append(gone, e.Name)
		}
	}
	return gone
}

func (m *Mempool) sorted() []*Entry {
	out := make([]*Entry, 0, len(m.entries))
	for _, e := range m.entries {
		out = append(out, e)
	}
	sort.Slice(out, func(i, j int) bool {
		if a, b := out[i].Fee*out[j].Size, out[j].Fee*out[i].Size; a != b {
			return a > b // highest fee rate first
		}
		return out[i].Name < out[j].Name
	})
	return out
}

func (m *Mempool) dump(label string) {
	fmt.Printf("  %s: %d transactions, %d/%d bytes, floor %d sat/byte\n",
		label, len(m.entries), m.bytes, maxBytes, m.Floor())
	for _, e := range m.sorted() {
		fmt.Printf("    %-8s %-6d bytes  %-8d sat  %3d sat/byte  age %d\n",
			e.Name, e.Size, e.Fee, e.Fee/e.Size, m.height-e.Added)
	}
}

func (m *Mempool) flush() {
	for _, l := range m.log {
		fmt.Printf("    %s\n", l)
	}
	m.log = nil
}

func tx(name string, size, rate int64) *Entry {
	return &Entry{Name: name, Size: size, Fee: size * rate}
}

func main() {
	m := NewMempool()

	fmt.Printf("=== a pool with a %d-byte budget ===\n", maxBytes)
	for _, t := range []*Entry{
		tx("high", 500, 60), tx("good", 600, 25), tx("okay", 700, 12),
		tx("meh", 600, 6), tx("cheap", 500, 2),
	} {
		if err := m.Add(t); err != nil {
			fmt.Printf("  %-8s REJECTED: %v\n", t.Name, err)
		}
	}
	m.dump("after 5 arrivals")

	fmt.Println("\n=== one more arrival pushes it over ===")
	if err := m.Add(tx("urgent", 800, 90)); err != nil {
		fmt.Println("  ", err)
	}
	m.flush()
	m.dump("now")
	fmt.Println("\n  The pool did not drop the newest transaction, or the oldest.")
	fmt.Println("  It dropped the one a miner would have taken last — which is the")
	fmt.Println("  only ranking that matches what the pool is FOR.")

	fmt.Println("\n=== the floor bites ===")
	for _, t := range []*Entry{tx("bargain", 300, 2), tx("fair", 300, 20)} {
		if err := m.Add(t); err != nil {
			fmt.Printf("  %-8s REJECTED: %v\n", t.Name, err)
			continue
		}
		fmt.Printf("  %-8s accepted\n", t.Name)
	}
	m.flush()
	fmt.Println()
	fmt.Println("  'bargain' would have been accepted ten minutes ago. Nothing")
	fmt.Println("  about it changed; the pool did. This is the mechanism behind")
	fmt.Println("  every 'my transaction was fine yesterday' report — and note")
	fmt.Println("  that it is POLICY, so a differently-configured node might")
	fmt.Println("  still take it (example 10).")

	fmt.Println("\n=== the floor decays ===")
	fmt.Printf("  %-10s %s\n", "height", "floor")
	for _, h := range []int64{0, 5, 10, 20, 30, 40, 50} {
		m.height = h
		fmt.Printf("  %-10d %d sat/byte\n", h, m.Floor())
	}
	m.height = 0
	fmt.Println()
	fmt.Println("  Halving on a timer, so a floor raised during a spike does not")
	fmt.Println("  outlive the spike. Without decay, one busy afternoon would lock")
	fmt.Println("  a node out of relaying cheap transactions until it restarted.")

	fmt.Println("\n=== evicting a parent takes its children ===")
	m2 := NewMempool()
	for _, t := range []*Entry{
		tx("rich", 900, 50),
		tx("poor", 700, 3),
	} {
		if err := m2.Add(t); err != nil {
			fmt.Println("  ", err)
		}
	}
	kid := tx("kid", 400, 30)
	kid.Parent = "poor"
	grandkid := tx("grandkid", 300, 30)
	grandkid.Parent = "kid"
	for _, t := range []*Entry{kid, grandkid} {
		if err := m2.Add(t); err != nil {
			fmt.Println("  ", err)
		}
	}
	m2.dump("before")
	fmt.Println("\n  now something better arrives:")
	if err := m2.Add(tx("better", 1200, 40)); err != nil {
		fmt.Println("  ", err)
	}
	m2.flush()
	m2.dump("after")
	fmt.Println("\n  'kid' and 'grandkid' paid 30 sat/byte and were still dropped.")
	fmt.Println("  They had to be: their inputs no longer exist anywhere the pool")
	fmt.Println("  can see. A pool that keeps them accumulates unspendable entries")
	fmt.Println("  it will never be able to mine or relay.")
	fmt.Println()
	fmt.Println("  Which also means eviction should really consider the DESCENDANT")
	fmt.Println("  fee rate, not the entry's own — dropping a cheap parent throws")
	fmt.Println("  away its expensive children. That is the same package view that")
	fmt.Println("  makes child-pays-for-parent work (example 16).")

	fmt.Println("\n=== expiry ===")
	m3 := NewMempool()
	for i, t := range []*Entry{tx("day1", 400, 30), tx("day2", 400, 30), tx("day3", 400, 30)} {
		m3.height = int64(i * 10)
		if err := m3.Add(t); err != nil {
			fmt.Println("  ", err)
		}
	}
	m3.height = 25
	fmt.Printf("  at height %d, expiry is %d blocks\n", m3.height, expiryBlocks)
	for _, e := range m3.sorted() {
		fmt.Printf("    %-8s added at %-4d age %d\n", e.Name, e.Added, m3.height-e.Added)
	}
	fmt.Printf("  expired: %v\n", m3.Expire())
	fmt.Println()
	fmt.Println("  Expiry is not about memory — the fee-rate eviction handles that.")
	fmt.Println("  It is about the pool telling the truth. A transaction nobody has")
	fmt.Println("  mined in two weeks is not going to be mined, and holding it makes")
	fmt.Println("  the wallet think it is still pending when it should re-send.")

	fmt.Println("\n=== the three policies, and what each protects ===")
	fmt.Printf("  %-22s %-26s %s\n", "policy", "protects against", "cost when wrong")
	for _, r := range [][3]string{
		{"byte budget", "memory exhaustion", "good transactions dropped"},
		{"dynamic floor", "refill after eviction", "cheap transactions locked out"},
		{"expiry", "stale pool state", "long-horizon transactions vanish"},
	} {
		fmt.Printf("  %-22s %-26s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  All three are local. None of them makes a transaction invalid,")
	fmt.Println("  and a miner who receives it directly can still mine it.")
}
```

**Output:**

```
=== a pool with a 3000-byte budget ===
  after 5 arrivals: 5 transactions, 2900/3000 bytes, floor 0 sat/byte
    high     500    bytes  30000    sat   60 sat/byte  age 0
    good     600    bytes  15000    sat   25 sat/byte  age 0
    okay     700    bytes  8400     sat   12 sat/byte  age 0
    meh      600    bytes  3600     sat    6 sat/byte  age 0
    cheap    500    bytes  1000     sat    2 sat/byte  age 0

=== one more arrival pushes it over ===
    evicted cheap at 2 sat/byte; floor now 3
    evicted meh at 6 sat/byte; floor now 7
  now: 4 transactions, 2600/3000 bytes, floor 7 sat/byte
    urgent   800    bytes  72000    sat   90 sat/byte  age 0
    high     500    bytes  30000    sat   60 sat/byte  age 0
    good     600    bytes  15000    sat   25 sat/byte  age 0
    okay     700    bytes  8400     sat   12 sat/byte  age 0

  The pool did not drop the newest transaction, or the oldest.
  It dropped the one a miner would have taken last — which is the
  only ranking that matches what the pool is FOR.

=== the floor bites ===
  bargain  REJECTED: rate 2 is below the pool floor of 7
  fair     accepted

  'bargain' would have been accepted ten minutes ago. Nothing
  about it changed; the pool did. This is the mechanism behind
  every 'my transaction was fine yesterday' report — and note
  that it is POLICY, so a differently-configured node might
  still take it (example 10).

=== the floor decays ===
  height     floor
  0          7 sat/byte
  5          7 sat/byte
  10         3 sat/byte
  20         1 sat/byte
  30         0 sat/byte
  40         0 sat/byte
  50         0 sat/byte

  Halving on a timer, so a floor raised during a spike does not
  outlive the spike. Without decay, one busy afternoon would lock
  a node out of relaying cheap transactions until it restarted.

=== evicting a parent takes its children ===
  before: 4 transactions, 2300/3000 bytes, floor 0 sat/byte
    rich     900    bytes  45000    sat   50 sat/byte  age 0
    grandkid 300    bytes  9000     sat   30 sat/byte  age 0
    kid      400    bytes  12000    sat   30 sat/byte  age 0
    poor     700    bytes  2100     sat    3 sat/byte  age 0

  now something better arrives:
    evicted poor at 3 sat/byte (+2 descendant(s)); floor now 4
  after: 2 transactions, 2100/3000 bytes, floor 4 sat/byte
    rich     900    bytes  45000    sat   50 sat/byte  age 0
    better   1200   bytes  48000    sat   40 sat/byte  age 0

  'kid' and 'grandkid' paid 30 sat/byte and were still dropped.
  They had to be: their inputs no longer exist anywhere the pool
  can see. A pool that keeps them accumulates unspendable entries
  it will never be able to mine or relay.

  Which also means eviction should really consider the DESCENDANT
  fee rate, not the entry's own — dropping a cheap parent throws
  away its expensive children. That is the same package view that
  makes child-pays-for-parent work (example 16).

=== expiry ===
  at height 25, expiry is 20 blocks
    day1     added at 0    age 25
    day2     added at 10   age 15
    day3     added at 20   age 5
  expired: [day1]

  Expiry is not about memory — the fee-rate eviction handles that.
  It is about the pool telling the truth. A transaction nobody has
  mined in two weeks is not going to be mined, and holding it makes
  the wallet think it is still pending when it should re-send.

=== the three policies, and what each protects ===
  policy                 protects against           cost when wrong
  byte budget            memory exhaustion          good transactions dropped
  dynamic floor          refill after eviction      cheap transactions locked out
  expiry                 stale pool state           long-horizon transactions vanish

  All three are local. None of them makes a transaction invalid,
  and a miner who receives it directly can still mine it.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
