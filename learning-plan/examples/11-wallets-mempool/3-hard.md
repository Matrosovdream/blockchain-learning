# Step 11 — Wallets, Fees & the Mempool · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. Replace-by-fee

`🔴 hard` · *Fee bumping*

Replace-by-fee. The estimate was wrong and the transaction is stuck below the cut line, so the sender publishes a replacement spending the same inputs. BIP-125's five rules all exist to stop the same attack: making a node do work, or give up bandwidth, for free.

**Steps:**

1. Implement all five rules and trip each one in turn.
2. See that beating the fee by 1 satoshi is not enough — rule 4 makes you pay for the relay too.
3. Turn full-RBF on and watch rule 1 stop mattering, as it did in Bitcoin Core v28.
4. Attach 120 descendants and watch rule 5 refuse a 10,000,000-sat replacement.
5. Work out what a bump really costs, and why the cumulative column is the wrong one.

```go
package main

import (
	"errors"
	"fmt"
	"sort"
)

// ===========================================================================
// Replace-by-fee. A transaction is stuck: the estimate was wrong (example 11)
// and it is sitting below the cut line. RBF lets the sender publish a
// replacement spending the same inputs at a higher fee, and the pool swaps
// one for the other.
//
// BIP-125 imposes five rules. Every one of them exists to stop the same
// attack: making a node do work, or give up bandwidth, for free.
// ===========================================================================

const minRelayRate = 1 // sat/byte, the price of the bandwidth a replacement costs

type Input struct {
	Outpoint string
	Sequence uint32 // < 0xfffffffe signals "replaceable" (BIP-125 rule 1)
}

type Tx struct {
	Name   string
	Inputs []Input
	Size   int64
	Fee    int64
	Parent string // for descendants held in the pool
	Unconf bool   // does it spend an output that is still unconfirmed?
}

func (t *Tx) Rate() int64 { return t.Fee / t.Size }

const replaceableMax = 0xfffffffe

func (t *Tx) SignalsRBF() bool {
	for _, in := range t.Inputs {
		if in.Sequence < replaceableMax {
			return true
		}
	}
	return false
}

// ------------------------------------------------------------------- pool

type Pool struct {
	txs      map[string]*Tx
	spentBy  map[string]string // outpoint -> tx name
	children map[string][]string
}

func NewPool() *Pool {
	return &Pool{txs: map[string]*Tx{}, spentBy: map[string]string{}, children: map[string][]string{}}
}

func (p *Pool) Insert(t *Tx) {
	p.txs[t.Name] = t
	for _, in := range t.Inputs {
		p.spentBy[in.Outpoint] = t.Name
	}
	if t.Parent != "" {
		p.children[t.Parent] = append(p.children[t.Parent], t.Name)
	}
}

// family returns a transaction and everything descended from it — all of
// which dies with it.
func (p *Pool) family(name string) []*Tx {
	out := []*Tx{p.txs[name]}
	for _, c := range p.children[name] {
		out = append(out, p.family(c)...)
	}
	return out
}

func (p *Pool) drop(name string) {
	for _, t := range p.family(name) {
		for _, in := range t.Inputs {
			delete(p.spentBy, in.Outpoint)
		}
		delete(p.children, t.Name)
		delete(p.txs, t.Name)
	}
}

// -------------------------------------------------------- the BIP-125 rules

var (
	ErrNoConflict   = errors.New("nothing to replace")
	ErrNotSignalled = errors.New("rule 1: the original does not signal replaceability")
	ErrNewUnconf    = errors.New("rule 2: the replacement adds a new unconfirmed input")
	ErrFeeTooLow    = errors.New("rule 3: absolute fee is not higher than everything it evicts")
	ErrNoBandwidth  = errors.New("rule 4: does not pay for its own relay bandwidth")
	ErrTooBig       = errors.New("rule 5: would evict too many transactions")
)

const maxEvictions = 100

// Replace applies BIP-125 and, if it passes, swaps the family out for the
// replacement. It returns what was evicted.
func (p *Pool) Replace(r *Tx, fullRBF bool) ([]string, error) {
	// Which pool transactions does this conflict with?
	conflicts := map[string]bool{}
	for _, in := range r.Inputs {
		if n, ok := p.spentBy[in.Outpoint]; ok {
			conflicts[n] = true
		}
	}
	if len(conflicts) == 0 {
		return nil, ErrNoConflict
	}

	names := make([]string, 0, len(conflicts))
	for n := range conflicts {
		names = append(names, n)
	}
	sort.Strings(names)

	// --- rule 1: the original opted in --------------------------------------
	// Bitcoin Core has shipped full-RBF since v28, which drops this rule: a
	// signal that anyone can ignore was never a safety property, only a hint.
	if !fullRBF {
		for _, n := range names {
			if !p.txs[n].SignalsRBF() {
				return nil, fmt.Errorf("%w (%s)", ErrNotSignalled, n)
			}
		}
	}

	// --- rule 5: bound the work ---------------------------------------------
	var evicted []*Tx
	for _, n := range names {
		evicted = append(evicted, p.family(n)...)
	}
	if len(evicted) > maxEvictions {
		return nil, fmt.Errorf("%w: %d > %d", ErrTooBig, len(evicted), maxEvictions)
	}

	// --- rule 2: no NEW unconfirmed inputs -----------------------------------
	// Otherwise a replacement could depend on something the node has not
	// validated, forcing it to do unbounded extra work.
	if r.Unconf {
		return nil, ErrNewUnconf
	}

	// --- rules 3 and 4: the money -------------------------------------------
	var evictedFee, evictedSize int64
	for _, t := range evicted {
		evictedFee += t.Fee
		evictedSize += t.Size
	}
	if r.Fee <= evictedFee {
		return nil, fmt.Errorf("%w: pays %d, must beat %d", ErrFeeTooLow, r.Fee, evictedFee)
	}
	// The node is about to relay the replacement to every peer. Somebody has
	// to pay for those bytes, and it is not the network.
	need := evictedFee + minRelayRate*r.Size
	if r.Fee < need {
		return nil, fmt.Errorf("%w: pays %d, needs %d (%d evicted + %d for %d bytes)",
			ErrNoBandwidth, r.Fee, need, evictedFee, minRelayRate*r.Size, r.Size)
	}

	var gone []string
	for _, t := range evicted {
		gone = append(gone, t.Name)
	}
	sort.Strings(gone)
	for _, n := range names {
		p.drop(n)
	}
	p.Insert(r)
	return gone, nil
}

// ------------------------------------------------------------------- demo

func in(op string, seq uint32) Input { return Input{op, seq} }

const (
	replaceable = 0xfffffffd // BIP-125 opt-in
	final       = 0xffffffff // "do not replace me"
)

func show(p *Pool, label string) {
	fmt.Printf("  %s:\n", label)
	names := make([]string, 0, len(p.txs))
	for n := range p.txs {
		names = append(names, n)
	}
	sort.Strings(names)
	for _, n := range names {
		t := p.txs[n]
		fmt.Printf("    %-12s %-6d bytes  %-8d sat  %3d sat/byte\n", t.Name, t.Size, t.Fee, t.Rate())
	}
}

func attempt(p *Pool, label string, r *Tx, fullRBF bool) {
	gone, err := p.Replace(r, fullRBF)
	if err != nil {
		fmt.Printf("  %-30s REJECTED: %v\n", label, err)
		return
	}
	fmt.Printf("  %-30s accepted, evicted %v\n", label, gone)
}

func main() {
	// --------------------------------------------------------------- setup
	p := NewPool()
	original := &Tx{
		Name: "payment", Size: 214, Fee: 428, // 2 sat/byte — too cheap
		Inputs: []Input{in("coinA:0", replaceable)},
	}
	p.Insert(original)

	fmt.Println("=== a transaction that is stuck ===")
	show(p, "pool")
	fmt.Printf("  it signals replaceability: %v (sequence %#x)\n\n",
		original.SignalsRBF(), original.Inputs[0].Sequence)

	fmt.Println("=== the five rules, one at a time ===")

	// Rule 3: must beat the absolute fee, not just the rate.
	attempt(p, "same fee, smaller size", &Tx{
		Name: "bump-a", Size: 182, Fee: 428, Inputs: []Input{in("coinA:0", replaceable)}}, false)

	// Rule 4: must also pay for the bytes it is about to make peers carry.
	attempt(p, "beats the fee by 1 sat", &Tx{
		Name: "bump-b", Size: 214, Fee: 429, Inputs: []Input{in("coinA:0", replaceable)}}, false)

	// Rule 2: no new unconfirmed inputs.
	attempt(p, "adds an unconfirmed input", &Tx{
		Name: "bump-c", Size: 356, Fee: 5000, Unconf: true,
		Inputs: []Input{in("coinA:0", replaceable), in("coinB:0", replaceable)}}, false)

	// Nothing to replace.
	attempt(p, "spends a different coin", &Tx{
		Name: "unrelated", Size: 214, Fee: 5000, Inputs: []Input{in("coinZ:0", replaceable)}}, false)

	// The one that works.
	good := &Tx{Name: "bump", Size: 214, Fee: 4280, Inputs: []Input{in("coinA:0", replaceable)}}
	attempt(p, "10x the fee rate", good, false)
	fmt.Println()
	show(p, "pool now")
	fmt.Printf("\n  paid %d sat instead of %d — the full cost of the mistake, not\n", good.Fee, original.Fee)
	fmt.Println("  the difference. A replacement replaces; it does not top up.")

	// --------------------------------------------------------------- rule 1
	fmt.Println("\n=== rule 1: opting in, and full-RBF ===")
	q := NewPool()
	q.Insert(&Tx{Name: "final-tx", Size: 214, Fee: 428, Inputs: []Input{in("coinC:0", final)}})
	repl := func() *Tx {
		return &Tx{Name: "bump-final", Size: 214, Fee: 4280, Inputs: []Input{in("coinC:0", final)}}
	}
	attempt(q, "original did not signal", repl(), false)
	attempt(q, "the same, with full-RBF on", repl(), true)
	fmt.Println()
	fmt.Println("  The signal was never a guarantee. A miner was always free to")
	fmt.Println("  take the higher fee, and anyone could always run a patched node.")
	fmt.Println("  Bitcoin Core made full-RBF the default in v28, which settled a")
	fmt.Println("  long argument by admitting what was already true: an unconfirmed")
	fmt.Println("  transaction is a proposal, not a payment.")

	// --------------------------------------------------------------- rule 5
	fmt.Println("\n=== rule 5: bounding the damage ===")
	s := NewPool()
	s.Insert(&Tx{Name: "root", Size: 214, Fee: 2140, Inputs: []Input{in("coinD:0", replaceable)}})
	for i := 0; i < 120; i++ {
		parent := "root"
		if i > 0 {
			parent = fmt.Sprintf("desc%d", i-1)
		}
		s.Insert(&Tx{
			Name: fmt.Sprintf("desc%d", i), Size: 182, Fee: 1820, Parent: parent,
			Inputs: []Input{in(fmt.Sprintf("chain%d:0", i), replaceable)},
		})
	}
	fmt.Printf("  a chain of %d transactions hangs off 'root'\n", len(s.txs)-1)
	attempt(s, "replacing root", &Tx{
		Name: "big-bump", Size: 214, Fee: 10_000_000, Inputs: []Input{in("coinD:0", replaceable)}}, false)
	fmt.Println()
	fmt.Println("  The fee is enormous and it still fails. Rule 5 caps the work a")
	fmt.Println("  single message can make a node do: without it, one cheap")
	fmt.Println("  replacement could force the removal and re-indexing of thousands")
	fmt.Println("  of entries, repeatedly, for the price of one transaction.")

	// --------------------------------------------------------------- costs
	fmt.Println("\n=== what a bump actually costs ===")
	fmt.Printf("  %-8s %-10s %-12s %-12s %s\n",
		"attempt", "rate", "fee paid", "cumulative", "note")
	var cumulative int64
	for i, rate := range []int64{2, 5, 12, 30} {
		fee := 214 * rate
		cumulative += fee
		note := "replaced"
		if i == 3 {
			note = "confirmed"
		}
		fmt.Printf("  %-8d %-10d %-12d %-12d %s\n", i+1, rate, fee, cumulative, note)
	}
	fmt.Printf("\n  Only the last fee is ever PAID — the replaced transactions never\n")
	fmt.Println("  confirm, so their fees are never collected. The cumulative column")
	fmt.Println("  is what a naive accounting would show, and it is wrong.")

	fmt.Println("\n=== what this means for a wallet ===")
	fmt.Println("  - signal replaceability on everything, always; there is no cost")
	fmt.Println("  - keep the inputs identical when bumping, so rule 2 cannot bite")
	fmt.Println("  - take the extra fee out of the CHANGE output, not by adding an")
	fmt.Println("    input, and stop when the change hits dust")
	fmt.Println("  - re-sign from scratch: a replacement is a different transaction")
	fmt.Println("    with a different txid, not an edit (lesson 10, example 6)")
	fmt.Println("  - and tell the user the truth: the original may still confirm")
	fmt.Println("    right up until the replacement does. Example 15 is what happens")
	fmt.Println("    when someone else has an interest in which one wins.")
}
```

**Output:**

```
=== a transaction that is stuck ===
  pool:
    payment      214    bytes  428      sat    2 sat/byte
  it signals replaceability: true (sequence 0xfffffffd)

=== the five rules, one at a time ===
  same fee, smaller size         REJECTED: rule 3: absolute fee is not higher than everything it evicts: pays 428, must beat 428
  beats the fee by 1 sat         REJECTED: rule 4: does not pay for its own relay bandwidth: pays 429, needs 642 (428 evicted + 214 for 214 bytes)
  adds an unconfirmed input      REJECTED: rule 2: the replacement adds a new unconfirmed input
  spends a different coin        REJECTED: nothing to replace
  10x the fee rate               accepted, evicted [payment]

  pool now:
    bump         214    bytes  4280     sat   20 sat/byte

  paid 4280 sat instead of 428 — the full cost of the mistake, not
  the difference. A replacement replaces; it does not top up.

=== rule 1: opting in, and full-RBF ===
  original did not signal        REJECTED: rule 1: the original does not signal replaceability (final-tx)
  the same, with full-RBF on     accepted, evicted [final-tx]

  The signal was never a guarantee. A miner was always free to
  take the higher fee, and anyone could always run a patched node.
  Bitcoin Core made full-RBF the default in v28, which settled a
  long argument by admitting what was already true: an unconfirmed
  transaction is a proposal, not a payment.

=== rule 5: bounding the damage ===
  a chain of 120 transactions hangs off 'root'
  replacing root                 REJECTED: rule 5: would evict too many transactions: 121 > 100

  The fee is enormous and it still fails. Rule 5 caps the work a
  single message can make a node do: without it, one cheap
  replacement could force the removal and re-indexing of thousands
  of entries, repeatedly, for the price of one transaction.

=== what a bump actually costs ===
  attempt  rate       fee paid     cumulative   note
  1        2          428          428          replaced
  2        5          1070         1498         replaced
  3        12         2568         4066         replaced
  4        30         6420         10486        confirmed

  Only the last fee is ever PAID — the replaced transactions never
  confirm, so their fees are never collected. The cumulative column
  is what a naive accounting would show, and it is wrong.

=== what this means for a wallet ===
  - signal replaceability on everything, always; there is no cost
  - keep the inputs identical when bumping, so rule 2 cannot bite
  - take the extra fee out of the CHANGE output, not by adding an
    input, and stop when the change hits dust
  - re-sign from scratch: a replacement is a different transaction
    with a different txid, not an edit (lesson 10, example 6)
  - and tell the user the truth: the original may still confirm
    right up until the replacement does. Example 15 is what happens
    when someone else has an interest in which one wins.
```

---

## 15. Transaction pinning

`🔴 hard` · *Attacks*

Pinning: using the mempool's own rules to stop someone else bumping their fee. Attach a huge, cheap child to a counterparty's transaction and rule 3 makes replacing it arbitrarily expensive — while the package is too cheap for any miner to take. Unconfirmable and unreplaceable at once.

**Steps:**

1. Bump a transaction normally, then attach a 99 kB child and bump it again.
2. Watch the required fee go from 642 sat to 99,642 — 465 sat/byte for a 214-byte transaction.
3. Do it the other way, with 110 tiny descendants, and watch NO fee be large enough.
4. Read why this is theft rather than nuisance when the contract has a deadline.
5. Apply the three fixes: the CPFP carve-out, TRUC/v3 topology limits, and package relay.
6. Watch the same attack against a v3 transaction cost 1,642 sat instead of 99,642.

```go
package main

import (
	"errors"
	"fmt"
	"sort"
)

// ===========================================================================
// Transaction pinning: using the mempool's own rules to stop someone else
// from bumping their fee.
//
// BIP-125 rule 3 says a replacement must beat the ABSOLUTE fee of everything
// it evicts, including descendants. So if I can attach a huge, cheap child to
// your transaction, I can make replacing it arbitrarily expensive — while the
// package as a whole is so cheap that no miner will ever take it.
//
// Your transaction is now unconfirmable AND unreplaceable. If it is part of a
// contract with a deadline, that is money.
// ===========================================================================

const minRelayRate = 1 // sat/byte

type Tx struct {
	Name   string
	Spends []string
	Size   int64
	Fee    int64
	Parent string
	V3     bool // opted into the TRUC/v3 topology rules
}

func (t *Tx) Rate() int64 { return t.Fee / t.Size }

type Pool struct {
	txs      map[string]*Tx
	spentBy  map[string]string
	children map[string][]string
	// TRUC (BIP-431) limits, applied only to v3 transactions.
	truc bool
}

func NewPool(truc bool) *Pool {
	return &Pool{txs: map[string]*Tx{}, spentBy: map[string]string{},
		children: map[string][]string{}, truc: truc}
}

var (
	ErrTrucChildren = errors.New("TRUC: a v3 transaction may have at most one unconfirmed child")
	ErrTrucSize     = errors.New("TRUC: a child of a v3 transaction may not exceed 1000 bytes")
	ErrFeeTooLow    = errors.New("BIP-125 rule 3/4: fee does not beat what it would evict")
	ErrTooMany      = errors.New("BIP-125 rule 5: would evict more than 100 transactions")
)

const (
	trucChildSize = 1_000
	maxEvictions  = 100
)

func (p *Pool) Insert(t *Tx) error {
	if t.Parent != "" {
		parent, ok := p.txs[t.Parent]
		if !ok {
			return fmt.Errorf("parent %s not in the pool", t.Parent)
		}
		if p.truc && parent.V3 {
			if len(p.children[t.Parent]) >= 1 {
				return ErrTrucChildren
			}
			if t.Size > trucChildSize {
				return fmt.Errorf("%w (%d bytes)", ErrTrucSize, t.Size)
			}
		}
		p.children[t.Parent] = append(p.children[t.Parent], t.Name)
	}
	p.txs[t.Name] = t
	for _, op := range t.Spends {
		p.spentBy[op] = t.Name
	}
	return nil
}

func (p *Pool) family(name string) []*Tx {
	out := []*Tx{p.txs[name]}
	for _, c := range p.children[name] {
		out = append(out, p.family(c)...)
	}
	return out
}

// CostToReplace is the whole story of pinning in four lines.
func (p *Pool) CostToReplace(name string, replacementSize int64) (need int64, evicting int, err error) {
	fam := p.family(name)
	if len(fam) > maxEvictions {
		return 0, len(fam), ErrTooMany
	}
	var fees int64
	for _, t := range fam {
		fees += t.Fee
	}
	return fees + minRelayRate*replacementSize, len(fam), nil
}

// PackageRate is what a miner sees: the family's combined fee over its
// combined size. This is what decides whether it is ever mined at all.
func (p *Pool) PackageRate(name string) (int64, int64, int64) {
	var fee, size int64
	for _, t := range p.family(name) {
		fee += t.Fee
		size += t.Size
	}
	return fee, size, fee / size
}

func (p *Pool) dump(label string) {
	fmt.Printf("  %s:\n", label)
	names := make([]string, 0, len(p.txs))
	for n := range p.txs {
		names = append(names, n)
	}
	sort.Strings(names)
	for _, n := range names {
		t := p.txs[n]
		tag := ""
		if t.Parent != "" {
			tag = " (child of " + t.Parent + ")"
		}
		fmt.Printf("    %-16s %-8d bytes %-10d sat %4d sat/byte%s\n",
			t.Name, t.Size, t.Fee, t.Rate(), tag)
	}
}

func outcome(err error) string {
	if err == nil {
		return "accepted"
	}
	return "REJECTED: " + err.Error()
}

func main() {
	// ------------------------------------------------------------- the setup
	// Alice and Mallory are counterparties. Alice's transaction has an output
	// Mallory can spend — a refund branch, a shared channel output, an escrow
	// leg. This is normal; it is the whole point of a contract.
	fmt.Println("=== the honest situation ===")
	plain := NewPool(false)
	_ = plain.Insert(&Tx{Name: "alice-tx", Size: 214, Fee: 428, Spends: []string{"coin:0"}})
	plain.dump("pool")
	need, _, _ := plain.CostToReplace("alice-tx", 214)
	fmt.Printf("\n  alice wants 20 sat/byte, i.e. %d sat\n", 214*20)
	fmt.Printf("  BIP-125 requires at least %d sat  -> she pays %d\n", need, 214*20)
	fmt.Println("  Fine. A 10x bump costs 10x.")

	// ------------------------------------------------------------- the pin
	fmt.Println("\n=== mallory attaches a child ===")
	pinned := NewPool(false)
	_ = pinned.Insert(&Tx{Name: "alice-tx", Size: 214, Fee: 428, Spends: []string{"coin:0"}})
	_ = pinned.Insert(&Tx{
		Name: "mallory-child", Parent: "alice-tx", Size: 99_000, Fee: 99_000,
		Spends: []string{"alice-tx:1"},
	})
	pinned.dump("pool")

	fee, size, rate := pinned.PackageRate("alice-tx")
	fmt.Printf("\n  as a package: %d sat over %d bytes = %d sat/byte\n", fee, size, rate)
	fmt.Println("  No miner will ever touch that. The package is 99 kB of block")
	fmt.Println("  space at the bottom of the fee market.")

	need, evicting, _ := pinned.CostToReplace("alice-tx", 214)
	fmt.Printf("\n  and to replace alice-tx, evicting %d transactions:\n", evicting)
	fmt.Printf("    required fee    %d sat\n", need)
	fmt.Printf("    for a %d-byte transaction, that is %d sat/byte\n", int64(214), need/214)
	fmt.Printf("    against the %d sat/byte she actually wanted: %dx\n", int64(20), (need/214)/20)
	fmt.Println()
	fmt.Println("  Mallory spent 99,000 sat that she will never actually pay —")
	fmt.Println("  the child cannot confirm either — to cost alice 99,642 or the")
	fmt.Println("  contract. That asymmetry is the attack.")

	// --------------------------------------------------------- the other pin
	fmt.Println("\n=== the other way: rule 5 ===")
	counted := NewPool(false)
	_ = counted.Insert(&Tx{Name: "alice-tx", Size: 214, Fee: 428, Spends: []string{"coin:0"}})
	prev := "alice-tx"
	for i := 0; i < 110; i++ {
		name := fmt.Sprintf("junk%d", i)
		_ = counted.Insert(&Tx{Name: name, Parent: prev, Size: 150, Fee: 150,
			Spends: []string{fmt.Sprintf("%s:0", prev)}})
		prev = name
	}
	_, evicting, err := counted.CostToReplace("alice-tx", 214)
	fmt.Printf("  mallory attaches a chain of 110 tiny descendants\n")
	fmt.Printf("  replacing alice-tx would evict %d: %v\n", evicting, err)
	fmt.Println("  No fee is large enough. The transaction is simply frozen.")

	// ---------------------------------------------------------- why it hurts
	fmt.Println("\n=== why this is not just annoying ===")
	fmt.Println("  Most contracts with a counterparty have a DEADLINE. A Lightning")
	fmt.Println("  HTLC must be claimed on-chain before its timelock expires; an")
	fmt.Println("  atomic swap has a refund branch that opens at a fixed height.")
	fmt.Println("  If your transaction cannot confirm and cannot be bumped before")
	fmt.Println("  that height, your counterparty takes the money — legitimately,")
	fmt.Println("  by the contract's own rules.")
	fmt.Println()
	fmt.Println("  So pinning turns a mempool POLICY detail into a theft primitive.")
	fmt.Println("  Nothing here is consensus-invalid; every transaction is honest.")

	// ------------------------------------------------------------- the fixes
	fmt.Println("\n=== fix 1: the CPFP carve-out (Core 0.19) ===")
	fmt.Println("  A narrow exception: one extra descendant is allowed past the")
	fmt.Println("  usual limits if it is small and the parent has few children.")
	fmt.Println("  It let Lightning attach an anchor-output child even when the")
	fmt.Println("  counterparty had already attached one. It helped, and it did")
	fmt.Println("  not solve rule 3 — the absolute-fee pin above still works.")

	fmt.Println("\n=== fix 2: TRUC / v3 transactions (BIP-431) ===")
	fmt.Println("  Opt into a stricter topology and the pin becomes impossible:")
	fmt.Println("    - at most one unconfirmed parent and one unconfirmed child")
	fmt.Println("    - the child may not exceed 1000 bytes")
	fmt.Println("    - both must be v3, so nobody can attach a non-v3 monster")
	fmt.Println()
	truc := NewPool(true)
	_ = truc.Insert(&Tx{Name: "alice-tx", Size: 214, Fee: 428, Spends: []string{"coin:0"}, V3: true})
	fmt.Printf("  mallory tries the 99 kB child : %s\n",
		outcome(truc.Insert(&Tx{Name: "mallory-child", Parent: "alice-tx", Size: 99_000, Fee: 99_000,
			Spends: []string{"alice-tx:1"}})))
	fmt.Printf("  mallory tries a 1000-byte one : %s\n",
		outcome(truc.Insert(&Tx{Name: "mallory-small", Parent: "alice-tx", Size: 1_000, Fee: 1_000,
			Spends: []string{"alice-tx:1"}})))
	fmt.Printf("  and then a second child       : %s\n",
		outcome(truc.Insert(&Tx{Name: "mallory-again", Parent: "alice-tx", Size: 500, Fee: 500,
			Spends: []string{"alice-tx:2"}})))

	trucNeed, _, _ := truc.CostToReplace("alice-tx", 214)
	fmt.Printf("\n  worst case cost to replace alice-tx: %d sat (%d sat/byte)\n",
		trucNeed, trucNeed/214)
	fmt.Printf("  under the old rules it was:         %d sat (%d sat/byte)\n", need, need/214)
	fmt.Println()
	fmt.Println("  The pin is bounded by construction: one child, capped at 1000")
	fmt.Println("  bytes, so the worst an attacker can add to the replacement cost")
	fmt.Println("  is about 1000 sat rather than 99,000.")

	fmt.Println("\n=== fix 3: package relay ===")
	fmt.Println("  The deeper problem is that a node judges each transaction alone.")
	fmt.Println("  Package relay (Core 28) lets a parent and child be submitted and")
	fmt.Println("  evaluated TOGETHER, so a low-fee parent can ride in on its")
	fmt.Println("  child's fee — which is what CPFP was always supposed to do")
	fmt.Println("  (example 16) and could not, when the parent was below the pool's")
	fmt.Println("  minimum on its own.")

	fmt.Println("\n=== the general lesson ===")
	fmt.Println("  Every mempool rule is a resource limit, and every resource limit")
	fmt.Println("  is something an adversary can aim at someone else. Rule 3 exists")
	fmt.Println("  to stop free relay; it also hands a counterparty a lever. There")
	fmt.Println("  is no version of these rules with no lever — only versions where")
	fmt.Println("  the lever is short enough to price in.")
	fmt.Println()
	fmt.Println("  Which is why protocols with deadlines should assume the worst:")
	fmt.Println("  use TRUC, keep fee-bumping paths short, and never design a")
	fmt.Println("  timeout that assumes your transaction will confirm.")
}
```

**Output:**

```
=== the honest situation ===
  pool:
    alice-tx         214      bytes 428        sat    2 sat/byte

  alice wants 20 sat/byte, i.e. 4280 sat
  BIP-125 requires at least 642 sat  -> she pays 4280
  Fine. A 10x bump costs 10x.

=== mallory attaches a child ===
  pool:
    alice-tx         214      bytes 428        sat    2 sat/byte
    mallory-child    99000    bytes 99000      sat    1 sat/byte (child of alice-tx)

  as a package: 99428 sat over 99214 bytes = 1 sat/byte
  No miner will ever touch that. The package is 99 kB of block
  space at the bottom of the fee market.

  and to replace alice-tx, evicting 2 transactions:
    required fee    99642 sat
    for a 214-byte transaction, that is 465 sat/byte
    against the 20 sat/byte she actually wanted: 23x

  Mallory spent 99,000 sat that she will never actually pay —
  the child cannot confirm either — to cost alice 99,642 or the
  contract. That asymmetry is the attack.

=== the other way: rule 5 ===
  mallory attaches a chain of 110 tiny descendants
  replacing alice-tx would evict 111: BIP-125 rule 5: would evict more than 100 transactions
  No fee is large enough. The transaction is simply frozen.

=== why this is not just annoying ===
  Most contracts with a counterparty have a DEADLINE. A Lightning
  HTLC must be claimed on-chain before its timelock expires; an
  atomic swap has a refund branch that opens at a fixed height.
  If your transaction cannot confirm and cannot be bumped before
  that height, your counterparty takes the money — legitimately,
  by the contract's own rules.

  So pinning turns a mempool POLICY detail into a theft primitive.
  Nothing here is consensus-invalid; every transaction is honest.

=== fix 1: the CPFP carve-out (Core 0.19) ===
  A narrow exception: one extra descendant is allowed past the
  usual limits if it is small and the parent has few children.
  It let Lightning attach an anchor-output child even when the
  counterparty had already attached one. It helped, and it did
  not solve rule 3 — the absolute-fee pin above still works.

=== fix 2: TRUC / v3 transactions (BIP-431) ===
  Opt into a stricter topology and the pin becomes impossible:
    - at most one unconfirmed parent and one unconfirmed child
    - the child may not exceed 1000 bytes
    - both must be v3, so nobody can attach a non-v3 monster

  mallory tries the 99 kB child : REJECTED: TRUC: a child of a v3 transaction may not exceed 1000 bytes (99000 bytes)
  mallory tries a 1000-byte one : accepted
  and then a second child       : REJECTED: TRUC: a v3 transaction may have at most one unconfirmed child

  worst case cost to replace alice-tx: 1642 sat (7 sat/byte)
  under the old rules it was:         99642 sat (465 sat/byte)

  The pin is bounded by construction: one child, capped at 1000
  bytes, so the worst an attacker can add to the replacement cost
  is about 1000 sat rather than 99,000.

=== fix 3: package relay ===
  The deeper problem is that a node judges each transaction alone.
  Package relay (Core 28) lets a parent and child be submitted and
  evaluated TOGETHER, so a low-fee parent can ride in on its
  child's fee — which is what CPFP was always supposed to do
  (example 16) and could not, when the parent was below the pool's
  minimum on its own.

=== the general lesson ===
  Every mempool rule is a resource limit, and every resource limit
  is something an adversary can aim at someone else. Rule 3 exists
  to stop free relay; it also hands a counterparty a lever. There
  is no version of these rules with no lever — only versions where
  the lever is short enough to price in.

  Which is why protocols with deadlines should assume the worst:
  use TRUC, keep fee-bumping paths short, and never design a
  timeout that assumes your transaction will confirm.
```

---

## 16. Child-pays-for-parent

`🔴 hard` · *Fee bumping*

Child-pays-for-parent. You cannot replace a transaction you did not send — but if it pays you, you can spend its output at a high fee and dare the miner to take the child without the parent. The unit of selection stops being a transaction and becomes a PACKAGE.

**Steps:**

1. Compute each transaction's ancestor fee rate, which is what Core actually sorts by.
2. Run a per-transaction greedy miner and watch it skip the highest-rate transaction in the pool.
3. Run a package-aware one and collect 1.56x from the same mempool and the same block size.
4. Check the ORDER: parents before children, or the block is invalid.
5. Work out why CPFP costs more than RBF, and tabulate when each one is the only option.
6. Read the catch: CPFP rescues a parent that is cheap, not one that never got into the pool.

```go
package main

import (
	"fmt"
	"sort"
)

// ===========================================================================
// Child-pays-for-parent. A transaction is stuck below the cut line, and the
// person who wants it confirmed cannot replace it — because they did not
// send it. So they spend one of its outputs, at a high fee, and dare the
// miner to take the child without the parent.
//
// The miner cannot. So the unit of selection stops being a transaction and
// becomes a PACKAGE: a transaction plus its unconfirmed ancestors, ranked by
// their combined fee rate.
//
// A miner who does not implement this leaves money on the table. This example
// measures how much.
// ===========================================================================

const blockCap = 2_000 // bytes

type Tx struct {
	Name   string
	Parent string // "" if all its inputs are confirmed
	Size   int64
	Fee    int64
}

func (t *Tx) Rate() int64 { return t.Fee / t.Size }

type Pool struct {
	txs   map[string]*Tx
	order []string // insertion order, for deterministic iteration
}

func NewPool(txs ...*Tx) *Pool {
	p := &Pool{txs: map[string]*Tx{}}
	for _, t := range txs {
		p.txs[t.Name] = t
		p.order = append(p.order, t.Name)
	}
	return p
}

// ancestors returns t and every unconfirmed ancestor, parents first.
func (p *Pool) ancestors(name string, confirmed map[string]bool) []*Tx {
	t := p.txs[name]
	if t == nil || confirmed[name] {
		return nil
	}
	var out []*Tx
	if t.Parent != "" && !confirmed[t.Parent] {
		out = append(out, p.ancestors(t.Parent, confirmed)...)
	}
	return append(out, t)
}

// ancestorRate is the number Bitcoin Core sorts by: the fee rate of the whole
// package a miner would have to take in order to take THIS transaction.
func (p *Pool) ancestorRate(name string, confirmed map[string]bool) (fee, size int64) {
	for _, a := range p.ancestors(name, confirmed) {
		fee += a.Fee
		size += a.Size
	}
	return
}

// ---------------------------------------------------- 1. per-transaction

// GreedyPerTx sorts by each transaction's OWN fee rate and takes whatever
// fits, skipping anything whose parent is not already in the block. It is
// correct — it never produces an invalid block — and it is leaving money on
// the floor.
func GreedyPerTx(p *Pool) (fees, used int64, picked []string) {
	names := append([]string(nil), p.order...)
	sort.SliceStable(names, func(i, j int) bool {
		a, b := p.txs[names[i]], p.txs[names[j]]
		if l, r := a.Fee*b.Size, b.Fee*a.Size; l != r {
			return l > r
		}
		return a.Name < b.Name
	})
	in := map[string]bool{}
	for _, n := range names {
		t := p.txs[n]
		if t.Parent != "" && !in[t.Parent] {
			continue // parent is not in the block, so this cannot be either
		}
		if used+t.Size > blockCap {
			continue
		}
		used += t.Size
		fees += t.Fee
		in[n] = true
		picked = append(picked, n)
	}
	return
}

// ------------------------------------------------------ 2. package-aware

// GreedyPackage repeatedly takes the transaction with the best ANCESTOR fee
// rate and adds its whole ancestor set, parents first. After each round the
// included transactions count as confirmed, so the remaining packages get
// cheaper — a second child of the same parent no longer has to pay for it.
func GreedyPackage(p *Pool) (fees, used int64, picked []string) {
	done := map[string]bool{}
	for {
		bestName := ""
		var bestFee, bestSize int64
		for _, n := range p.order {
			if done[n] {
				continue
			}
			f, s := p.ancestorRate(n, done)
			if used+s > blockCap {
				continue
			}
			if bestName == "" || f*bestSize > bestFee*s {
				bestName, bestFee, bestSize = n, f, s
			}
		}
		if bestName == "" {
			return
		}
		for _, a := range p.ancestors(bestName, done) {
			done[a.Name] = true
			picked = append(picked, a.Name)
			used += a.Size
			fees += a.Fee
		}
	}
}

// --------------------------------------------------------------------- demo

func main() {
	// A stuck payment from alice to bob, and bob's child bumping it.
	stuck := &Tx{Name: "alice-payment", Size: 214, Fee: 428}                       // 2 sat/byte
	bump := &Tx{Name: "bob-cpfp", Parent: "alice-payment", Size: 300, Fee: 60_000} // 200 sat/byte

	pool := NewPool(
		stuck, bump,
		&Tx{Name: "indep-1", Size: 700, Fee: 35_000},
		&Tx{Name: "indep-2", Size: 600, Fee: 24_000},
		&Tx{Name: "indep-3", Size: 500, Fee: 17_500},
		&Tx{Name: "indep-4", Size: 400, Fee: 12_000},
		&Tx{Name: "indep-5", Size: 300, Fee: 7_500},
	)

	fmt.Printf("=== the mempool (block holds %d bytes) ===\n", blockCap)
	fmt.Printf("  %-16s %-8s %-9s %-11s %s\n", "transaction", "bytes", "fee", "own rate", "package rate")
	none := map[string]bool{}
	for _, n := range pool.order {
		t := pool.txs[n]
		f, s := pool.ancestorRate(n, none)
		tag := ""
		if t.Parent != "" {
			tag = "  (child of " + t.Parent + ")"
		}
		fmt.Printf("  %-16s %-8d %-9d %-11d %d%s\n", t.Name, t.Size, t.Fee, t.Rate(), f/s, tag)
	}
	fmt.Println("\n  alice-payment pays 2 sat/byte and is going nowhere. bob cannot")
	fmt.Println("  replace it — he does not have alice's keys, and RBF replaces a")
	fmt.Println("  transaction, it does not amend one. What he CAN do is spend its")
	fmt.Println("  output at 200 sat/byte, so that taking his money means taking")
	fmt.Println("  hers too.")

	fmt.Println("\n=== a miner who sorts by each transaction's own rate ===")
	f1, u1, p1 := GreedyPerTx(pool)
	fmt.Printf("  %d bytes used, %d sat collected\n", u1, f1)
	fmt.Printf("  %v\n", p1)
	fmt.Println("  bob-cpfp is the highest-rate transaction in the pool and it is")
	fmt.Println("  not in the block, because its parent is not. The miner skipped")
	fmt.Println("  60,000 sat to include 17,500.")

	fmt.Println("\n=== a miner who sorts by ancestor rate ===")
	f2, u2, p2 := GreedyPackage(pool)
	fmt.Printf("  %d bytes used, %d sat collected\n", u2, f2)
	fmt.Printf("  %v\n", p2)
	fmt.Printf("\n  %.2fx the revenue, from the same mempool and the same block size.\n",
		float64(f2)/float64(f1))
	fmt.Println("  Note the ORDER: alice-payment appears before bob-cpfp. A block")
	fmt.Println("  with them the other way round is invalid (lesson 10).")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== what the package rate actually says ===")
	fmt.Printf("  alice-payment alone : %d sat over %d bytes = %d sat/byte\n",
		stuck.Fee, stuck.Size, stuck.Rate())
	fmt.Printf("  bob-cpfp alone      : %d sat over %d bytes = %d sat/byte\n",
		bump.Fee, bump.Size, bump.Rate())
	pf, ps := pool.ancestorRate("bob-cpfp", none)
	fmt.Printf("  the two together    : %d sat over %d bytes = %d sat/byte\n", pf, ps, pf/ps)
	fmt.Println()
	fmt.Println("  The child has to pay for the parent's bytes as well as its own.")
	fmt.Println("  Which is why CPFP is expensive: bumping a 214-byte parent from")
	fmt.Println("  2 to 20 sat/byte via a 300-byte child costs")
	target := int64(20)
	need := target*(stuck.Size+300) - stuck.Fee
	fmt.Printf("      %d x %d - %d = %d sat\n", target, stuck.Size+300, stuck.Fee, need)
	fmt.Printf("  where replacing it outright would have cost %d.\n", target*stuck.Size)

	// ---------------------------------------------------------------------
	fmt.Println("\n=== RBF or CPFP? ===")
	fmt.Printf("  %-26s %-22s %s\n", "", "RBF", "CPFP")
	for _, r := range [][3]string{
		{"who can do it", "only the sender", "anyone with an output"},
		{"needs a new signature", "yes, on a new tx", "yes, on the child"},
		{"the txid", "changes", "unchanged"},
		{"cost", "pays for its own size", "pays for parent + child"},
		{"blocked by a pin", "yes (example 15)", "partly; needs package relay"},
		{"works for the receiver", "no", "yes — this is the point"},
	} {
		fmt.Printf("  %-26s %-22s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  RBF is cheaper and is what a sender should reach for first.")
	fmt.Println("  CPFP is the only option when the money is already coming TO you:")
	fmt.Println("  an exchange crediting a slow deposit, a Lightning node forcing a")
	fmt.Println("  channel close, anyone accepting a payment from a wallet with a")
	fmt.Println("  bad fee estimator.")

	// ---------------------------------------------------------------------
	fmt.Println("\n=== the catch ===")
	fmt.Println("  A parent below the pool's MINIMUM never gets into the mempool at")
	fmt.Println("  all, so there is no output for a child to spend and nothing for")
	fmt.Println("  ancestor-rate selection to find. CPFP only rescues a parent that")
	fmt.Println("  is cheap, not one that is invisible.")
	fmt.Println()
	fmt.Println("  That is the hole package relay (example 15) fills: submit parent")
	fmt.Println("  and child together, judge them together, admit them together.")
}
```

**Output:**

```
=== the mempool (block holds 2000 bytes) ===
  transaction      bytes    fee       own rate    package rate
  alice-payment    214      428       2           2
  bob-cpfp         300      60000     200         117  (child of alice-payment)
  indep-1          700      35000     50          50
  indep-2          600      24000     40          40
  indep-3          500      17500     35          35
  indep-4          400      12000     30          30
  indep-5          300      7500      25          25

  alice-payment pays 2 sat/byte and is going nowhere. bob cannot
  replace it — he does not have alice's keys, and RBF replaces a
  transaction, it does not amend one. What he CAN do is spend its
  output at 200 sat/byte, so that taking his money means taking
  hers too.

=== a miner who sorts by each transaction's own rate ===
  1800 bytes used, 76500 sat collected
  [indep-1 indep-2 indep-3]
  bob-cpfp is the highest-rate transaction in the pool and it is
  not in the block, because its parent is not. The miner skipped
  60,000 sat to include 17,500.

=== a miner who sorts by ancestor rate ===
  1814 bytes used, 119428 sat collected
  [alice-payment bob-cpfp indep-1 indep-2]

  1.56x the revenue, from the same mempool and the same block size.
  Note the ORDER: alice-payment appears before bob-cpfp. A block
  with them the other way round is invalid (lesson 10).

=== what the package rate actually says ===
  alice-payment alone : 428 sat over 214 bytes = 2 sat/byte
  bob-cpfp alone      : 60000 sat over 300 bytes = 200 sat/byte
  the two together    : 60428 sat over 514 bytes = 117 sat/byte

  The child has to pay for the parent's bytes as well as its own.
  Which is why CPFP is expensive: bumping a 214-byte parent from
  2 to 20 sat/byte via a 300-byte child costs
      20 x 514 - 428 = 9852 sat
  where replacing it outright would have cost 4280.

=== RBF or CPFP? ===
                             RBF                    CPFP
  who can do it              only the sender        anyone with an output
  needs a new signature      yes, on a new tx       yes, on the child
  the txid                   changes                unchanged
  cost                       pays for its own size  pays for parent + child
  blocked by a pin           yes (example 15)       partly; needs package relay
  works for the receiver     no                     yes — this is the point

  RBF is cheaper and is what a sender should reach for first.
  CPFP is the only option when the money is already coming TO you:
  an exchange crediting a slow deposit, a Lightning node forcing a
  channel close, anyone accepting a payment from a wallet with a
  bad fee estimator.

=== the catch ===
  A parent below the pool's MINIMUM never gets into the mempool at
  all, so there is no output for a child to spend and nothing for
  ancestor-rate selection to find. CPFP only rescues a parent that
  is cheap, not one that is invisible.

  That is the hole package relay (example 15) fills: submit parent
  and child together, judge them together, admit them together.
```

---

## 17. One pool, many goroutines

`🔴 hard` · *Concurrency*

The mempool is written by the network layer and read by the miner, continuously. Two standard answers — an `RWMutex`, or one goroutine that owns the state behind a command channel — built here behind one interface, and the measurement that actually decides throughput.

**Steps:**

1. Build both, keeping all synchronisation out of the data structure itself.
2. Hammer each with 8 writers and 4 readers and assert the final state is identical.
3. Confirm `Close()` leaves no goroutine behind, and that `go run -race` is clean.
4. Measure the real mistake: validating inside the lock serializes 100% of the work.
5. Read the comparison table, and why a mempool wants the mutex.
6. Read why this example prints no timings — and the exact commands to measure it yourself.

```go
package main

import (
	"fmt"
	"runtime"
	"sort"
	"sync"
	"sync/atomic"
)

// ===========================================================================
// The mempool is written by the network layer and read by the miner, both
// continuously. That makes it a real Go concurrency problem, and the two
// standard answers are:
//
//   1. a sync.RWMutex around shared state
//   2. one goroutine that OWNS the state, fed by a channel of commands
//
// This example builds both behind one interface, proves they produce
// identical state under identical concurrent load, and measures the thing
// that actually decides throughput: how much work happens while the lock is
// held.
//
// It deliberately prints no timings — see "why there are no numbers here".
// ===========================================================================

type Entry struct {
	Name string
	Size int64
	Fee  int64
}

func (e Entry) Rate() int64 { return e.Fee / e.Size }

type Pool interface {
	Add(Entry) bool
	Best(n int) []Entry
	Evict(k int) int
	Stats() (count int, bytes int64)
	Close()
}

// ------------------------------------------------------------- shared core

// state is the actual data structure. Neither implementation below has any
// synchronisation in here — that is the point: the locking strategy is a
// wrapper, and keeping it out of the data structure is what makes swapping
// strategies a 30-line change instead of a rewrite.
type state struct {
	m     map[string]Entry
	bytes int64
}

func newState() *state { return &state{m: map[string]Entry{}} }

func (s *state) add(e Entry) bool {
	if _, ok := s.m[e.Name]; ok {
		return false
	}
	s.m[e.Name] = e
	s.bytes += e.Size
	return true
}

func (s *state) best(n int) []Entry {
	out := make([]Entry, 0, len(s.m))
	for _, e := range s.m {
		out = append(out, e)
	}
	sort.Slice(out, func(i, j int) bool {
		if a, b := out[i].Fee*out[j].Size, out[j].Fee*out[i].Size; a != b {
			return a > b
		}
		return out[i].Name < out[j].Name
	})
	if n < len(out) {
		out = out[:n]
	}
	return out
}

func (s *state) evict(k int) int {
	all := s.best(len(s.m))
	n := 0
	for i := len(all) - 1; i >= 0 && n < k; i-- {
		delete(s.m, all[i].Name)
		s.bytes -= all[i].Size
		n++
	}
	return n
}

// ------------------------------------------------------ 1. the RWMutex pool

type MutexPool struct {
	mu sync.RWMutex
	st *state
}

func NewMutexPool() *MutexPool { return &MutexPool{st: newState()} }

func (p *MutexPool) Add(e Entry) bool {
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.st.add(e)
}

// Best takes the READ lock, so any number of miners can build templates at
// once. That asymmetry is the whole reason to reach for RWMutex over Mutex:
// this workload is read-heavy.
func (p *MutexPool) Best(n int) []Entry {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return p.st.best(n)
}

func (p *MutexPool) Evict(k int) int {
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.st.evict(k)
}

func (p *MutexPool) Stats() (int, int64) {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return len(p.st.m), p.st.bytes
}

func (p *MutexPool) Close() {}

// ------------------------------------------------- 2. the owning goroutine

// ActorPool never shares the state at all. One goroutine holds it; everyone
// else sends a closure and waits for the reply. "Do not communicate by
// sharing memory; share memory by communicating."
type ActorPool struct {
	cmds chan func(*state)
	done chan struct{}
	wg   sync.WaitGroup
}

func NewActorPool() *ActorPool {
	p := &ActorPool{cmds: make(chan func(*state)), done: make(chan struct{})}
	p.wg.Add(1)
	go func() {
		defer p.wg.Done()
		st := newState()
		for {
			select {
			case fn := <-p.cmds:
				fn(st)
			case <-p.done:
				return
			}
		}
	}()
	return p
}

// do sends a command and waits for it to run. The reply channel is buffered,
// so the owner never blocks on a caller that has gone away (lesson 09).
func (p *ActorPool) do(fn func(*state)) {
	reply := make(chan struct{}, 1)
	p.cmds <- func(st *state) {
		fn(st)
		reply <- struct{}{}
	}
	<-reply
}

func (p *ActorPool) Add(e Entry) bool {
	var ok bool
	p.do(func(st *state) { ok = st.add(e) })
	return ok
}

func (p *ActorPool) Best(n int) []Entry {
	var out []Entry
	p.do(func(st *state) { out = st.best(n) })
	return out
}

func (p *ActorPool) Evict(k int) int {
	var n int
	p.do(func(st *state) { n = st.evict(k) })
	return n
}

func (p *ActorPool) Stats() (int, int64) {
	var c int
	var b int64
	p.do(func(st *state) { c, b = len(st.m), st.bytes })
	return c, b
}

func (p *ActorPool) Close() {
	close(p.done)
	p.wg.Wait() // cancel, THEN wait — nothing outlives Close
}

// ------------------------------------------------------------- the workload

const (
	writers      = 8
	perWriter    = 500
	readers      = 4
	readsEach    = 200
	templateSize = 20
)

func entry(w, i int) Entry {
	size := int64(150 + (w*7+i*13)%900)
	rate := int64(1 + (w*11+i*17)%80)
	return Entry{Name: fmt.Sprintf("w%d-t%03d", w, i), Size: size, Fee: size * rate}
}

// hammer runs the same deterministic set of operations against a pool from
// many goroutines at once. The ORDER is nondeterministic; the final state is
// not, because every add is distinct and no reader mutates anything.
func hammer(p Pool) (added, reads int64) {
	var wg sync.WaitGroup
	for w := 0; w < writers; w++ {
		wg.Add(1)
		go func(w int) {
			defer wg.Done()
			for i := 0; i < perWriter; i++ {
				if p.Add(entry(w, i)) {
					atomic.AddInt64(&added, 1)
				}
			}
		}(w)
	}
	for r := 0; r < readers; r++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := 0; i < readsEach; i++ {
				_ = p.Best(templateSize)
				atomic.AddInt64(&reads, 1)
			}
		}()
	}
	wg.Wait()
	return
}

func fingerprint(p Pool) string {
	best := p.Best(1 << 30)
	var fee, size int64
	for _, e := range best {
		fee += e.Fee
		size += e.Size
	}
	top := "none"
	if len(best) > 0 {
		top = best[0].Name
	}
	return fmt.Sprintf("%d entries, %d bytes, %d sat, best=%s", len(best), size, fee, top)
}

// ------------------------------------------- the mistake worth measuring

// Validation is the expensive part of admission: a signature check is tens of
// microseconds (lesson 10). Where it happens relative to the lock decides
// whether the pool scales.
const validationCost = 50 // arbitrary work units per admission

type instrumented struct {
	mu        sync.Mutex
	st        *state
	underLock int64 // work units performed while the lock was held
	totalWork int64
}

func (p *instrumented) validate() {
	atomic.AddInt64(&p.totalWork, validationCost)
}

// AddSlow validates INSIDE the critical section. Every other goroutine waits
// through it, so the pool admits transactions strictly one at a time.
func (p *instrumented) AddSlow(e Entry) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.validate()
	atomic.AddInt64(&p.underLock, validationCost)
	p.st.add(e)
}

// AddFast validates first, on data nobody else can see, and takes the lock
// only for the insert. Validation of different transactions now overlaps.
func (p *instrumented) AddFast(e Entry) {
	p.validate() // no lock held
	p.mu.Lock()
	defer p.mu.Unlock()
	p.st.add(e)
}

func main() {
	fmt.Printf("=== the workload ===\n")
	fmt.Printf("  %d writers x %d adds, %d readers x %d template builds\n",
		writers, perWriter, readers, readsEach)
	fmt.Println("  (GOMAXPROCS is whatever your machine has; it changes the")
	fmt.Println("  interleaving and none of the printed results)")

	fmt.Println("\n=== both implementations, same load ===")
	before := runtime.NumGoroutine()

	m := NewMutexPool()
	ma, mr := hammer(m)
	mf := fingerprint(m)
	m.Close()

	a := NewActorPool()
	aa, ar := hammer(a)
	af := fingerprint(a)
	a.Close()

	fmt.Printf("  %-16s adds %d, reads %d\n", "RWMutex", ma, mr)
	fmt.Printf("  %-16s %s\n", "", mf)
	fmt.Printf("  %-16s adds %d, reads %d\n", "owning goroutine", aa, ar)
	fmt.Printf("  %-16s %s\n", "", af)
	fmt.Printf("\n  identical final state: %v\n", mf == af)
	fmt.Println("  The interleaving differed on every run; the result cannot,")
	fmt.Println("  because adds are distinct and reads do not mutate. That is the")
	fmt.Println("  property to design for and the one to assert in tests.")

	fmt.Printf("\n  goroutines before %d, after %d — Close() left nothing behind\n",
		before, runtime.NumGoroutine())

	fmt.Println("\n=== eviction, from the same shared state ===")
	m2 := NewMutexPool()
	hammer(m2)
	c0, b0 := m2.Stats()
	n := m2.Evict(1000)
	c1, b1 := m2.Stats()
	fmt.Printf("  before %d entries / %d bytes\n", c0, b0)
	fmt.Printf("  evicted %d (lowest fee rate first)\n", n)
	fmt.Printf("  after  %d entries / %d bytes\n", c1, b1)
	fmt.Printf("  accounting holds: %v\n", c0-c1 == n)
	m2.Close()

	fmt.Println("\n=== where the validation goes ===")
	for _, c := range []struct {
		name string
		fn   func(*instrumented, Entry)
	}{
		{"validate inside the lock", (*instrumented).AddSlow},
		{"validate before the lock", (*instrumented).AddFast},
	} {
		p := &instrumented{st: newState()}
		var wg sync.WaitGroup
		for w := 0; w < writers; w++ {
			wg.Add(1)
			go func(w int) {
				defer wg.Done()
				for i := 0; i < perWriter; i++ {
					c.fn(p, entry(w, i))
				}
			}(w)
		}
		wg.Wait()
		fmt.Printf("  %-26s total work %-8d serialized under the lock %-8d (%d%%)\n",
			c.name, p.totalWork, p.underLock, p.underLock*100/p.totalWork)
	}
	fmt.Println()
	fmt.Println("  Same total work, and in the first case ALL of it is serialized.")
	fmt.Println("  Eight cores admitting transactions at the speed of one. The fix")
	fmt.Println("  is not a faster lock — it is doing the expensive part outside it.")
	fmt.Println()
	fmt.Println("  Which is why admission is written in two phases (example 9):")
	fmt.Println("    read a consistent snapshot of what you need (RLock, release)")
	fmt.Println("    verify signatures with no lock held")
	fmt.Println("    take the write lock, RE-CHECK the conflicts, insert")
	fmt.Println("  The re-check matters: the world moved while you were verifying.")

	fmt.Println("\n=== choosing between the two ===")
	fmt.Printf("  %-24s %-28s %s\n", "", "RWMutex", "owning goroutine")
	for _, r := range [][3]string{
		{"read-heavy load", "scales: readers share RLock", "serialized through one chan"},
		{"complex invariants", "easy to get subtly wrong", "impossible to violate"},
		{"deadlock risk", "real, with several locks", "none — one owner"},
		{"backpressure", "none; callers just block", "natural: a bounded channel"},
		{"cost per op", "a few ns uncontended", "two channel ops, ~hundreds"},
		{"debugging", "-race, mutex profile", "read the owner's loop"},
	} {
		fmt.Printf("  %-24s %-28s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  For a mempool, RWMutex. The load is read-heavy, the operations")
	fmt.Println("  are short, and the actor's channel becomes the bottleneck the")
	fmt.Println("  lock was supposed to be. Reach for the owning goroutine when the")
	fmt.Println("  invariants are complicated enough that you do not trust yourself")
	fmt.Println("  to hold the right lock every time.")

	fmt.Println("\n=== why there are no numbers here ===")
	fmt.Println("  Any timing this program printed would be a number from one")
	fmt.Println("  machine on one run, and it would change with GOMAXPROCS, other")
	fmt.Println("  load, and the scheduler's mood. Published output that cannot be")
	fmt.Println("  reproduced is worse than no output.")
	fmt.Println()
	fmt.Println("  Measure it yourself, on your hardware, with your workload:")
	fmt.Println()
	fmt.Println("      go test -race ./...                 # correctness first")
	fmt.Println("      go test -bench=Pool -benchmem")
	fmt.Println("      go test -bench=Pool -cpu=1,2,4,8     # does it actually scale?")
	fmt.Println("      go test -bench=Pool -mutexprofile=m.out")
	fmt.Println("      go tool pprof -top m.out             # where the waiting is")
	fmt.Println()
	fmt.Println("  And write the -race test first. A mempool bug that only appears")
	fmt.Println("  under load is a bug you will meet in production, at 3am, on the")
	fmt.Println("  one node that had a fast peer.")
}
```

**Output:**

```
=== the workload ===
  8 writers x 500 adds, 4 readers x 200 template builds
  (GOMAXPROCS is whatever your machine has; it changes the
  interleaving and none of the printed results)

=== both implementations, same load ===
  RWMutex          adds 4000, reads 800
                   4000 entries, 2357200 bytes, 95576340 sat, best=w0-t047
  owning goroutine adds 4000, reads 800
                   4000 entries, 2357200 bytes, 95576340 sat, best=w0-t047

  identical final state: true
  The interleaving differed on every run; the result cannot,
  because adds are distinct and reads do not mutate. That is the
  property to design for and the one to assert in tests.

  goroutines before 1, after 1 — Close() left nothing behind

=== eviction, from the same shared state ===
  before 4000 entries / 2357200 bytes
  evicted 1000 (lowest fee rate first)
  after  3000 entries / 1768637 bytes
  accounting holds: true

=== where the validation goes ===
  validate inside the lock   total work 200000   serialized under the lock 200000   (100%)
  validate before the lock   total work 200000   serialized under the lock 0        (0%)

  Same total work, and in the first case ALL of it is serialized.
  Eight cores admitting transactions at the speed of one. The fix
  is not a faster lock — it is doing the expensive part outside it.

  Which is why admission is written in two phases (example 9):
    read a consistent snapshot of what you need (RLock, release)
    verify signatures with no lock held
    take the write lock, RE-CHECK the conflicts, insert
  The re-check matters: the world moved while you were verifying.

=== choosing between the two ===
                           RWMutex                      owning goroutine
  read-heavy load          scales: readers share RLock  serialized through one chan
  complex invariants       easy to get subtly wrong     impossible to violate
  deadlock risk            real, with several locks     none — one owner
  backpressure             none; callers just block     natural: a bounded channel
  cost per op              a few ns uncontended         two channel ops, ~hundreds
  debugging                -race, mutex profile         read the owner's loop

  For a mempool, RWMutex. The load is read-heavy, the operations
  are short, and the actor's channel becomes the bottleneck the
  lock was supposed to be. Reach for the owning goroutine when the
  invariants are complicated enough that you do not trust yourself
  to hold the right lock every time.

=== why there are no numbers here ===
  Any timing this program printed would be a number from one
  machine on one run, and it would change with GOMAXPROCS, other
  load, and the scheduler's mood. Published output that cannot be
  reproduced is worse than no output.

  Measure it yourself, on your hardware, with your workload:

      go test -race ./...                 # correctness first
      go test -bench=Pool -benchmem
      go test -bench=Pool -cpu=1,2,4,8     # does it actually scale?
      go test -bench=Pool -mutexprofile=m.out
      go tool pprof -top m.out             # where the waiting is

  And write the -race test first. A mempool bug that only appears
  under load is a bug you will meet in production, at 3am, on the
  one node that had a fast peer.
```

---

## 18. A wallet, a mempool and a miner

`🔴 hard` · *Assembly*

Lesson 10's chain with a wallet on one side and a mempool on the other. The wallet selects, sizes, signs and broadcasts; the pool admits, orders, replaces and evicts; the miner builds a template by ancestor fee rate. None of it is consensus, and the chain would validate fine without any of it.

**Steps:**

1. Mature a coinbase, fund three wallets, then send six payments into 700 bytes of block space.
2. Watch three confirm and three wait.
3. Bump one with RBF and watch the original evicted.
4. Bump another with CPFP and watch a 12 sat/byte transaction get mined ahead of a 20.
5. Check conservation: every unit unspent equals every unit the coinbases issued.
6. Read the diff from lesson 10, and what lesson 12 puts on disk.

```go
package main

import (
	"bytes"
	"crypto/ecdsa"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"math/big"
	"sort"

	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/ripemd160"
)

// ===========================================================================
// Lesson 10's chain, with a wallet and a mempool either side of it.
//
// NEW in this lesson:
//   + Wallet: coin selection on effective values, fee from a rate, change,
//     signing, and a preflight check before broadcast
//   + Mempool: admission, a fee-rate order, conflict tracking, RBF, eviction
//   + BlockTemplate: the miner picks by ANCESTOR fee rate, under a size cap
//   + the chain tells the pool what confirmed, so it can drop it
//
// UNCHANGED from lessons 08-10 and elided here so the new code is visible:
// difficulty retargeting and median-time-past. Lesson 10's example 18 has
// them; everything else below is the same code.
// ===========================================================================

const (
	HeaderVersion    = 1
	HeaderSize       = 92
	CoinbaseMaturity = 5 // Bitcoin: 100
	HalvingPeriod    = 20
	Coin             = int64(100_000_000)
	InitialReward    = 50 * Coin
	MaxMoney         = 21_000_000 * Coin

	MaxBlockSize = 700 // bytes of transactions, so the fee market bites
	MinRelayRate = 1   // sat/byte
	MaxPoolBytes = 4_000
)

// ---------------------------------------------------- header & PoW (08, 09)

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

func (h Header) Bytes() []byte {
	buf := bytes.NewBuffer(make([]byte, 0, HeaderSize))
	binary.Write(buf, binary.BigEndian, h.Version)
	buf.Write(h.PrevHash[:])
	buf.Write(h.MerkleRoot[:])
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Bits)
	binary.Write(buf, binary.BigEndian, h.Nonce)
	binary.Write(buf, binary.BigEndian, h.Height)
	return buf.Bytes()
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

func BitsToTarget(bits uint32) *big.Int {
	exp, m := bits>>24, bits&0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

var maxTarget = BitsToTarget(0x2000ffff)

func CheckPoW(h Header) bool {
	t := BitsToTarget(h.Bits)
	if t.Sign() <= 0 || t.Cmp(maxTarget) > 0 {
		return false
	}
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(t) < 0
}

func Mine(h Header) Header {
	target := BitsToTarget(h.Bits)
	buf := h.Bytes()
	h1, h2 := sha256.New(), sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var val big.Int
	for nonce := uint32(0); ; nonce++ {
		binary.BigEndian.PutUint32(buf[80:84], nonce)
		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])
		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])
		if val.SetBytes(d2).Cmp(target) < 0 {
			h.Nonce = nonce
			return h
		}
	}
}

// ------------------------------------------------------- transactions (10)

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type TxInput struct {
	Prev      Outpoint
	Signature []byte
	PubKey    []byte
	Sequence  uint32 // < 0xfffffffe signals RBF (example 14)
}

type TxOutput struct {
	Value      int64
	PubKeyHash []byte
}

type Transaction struct {
	Inputs  []TxInput
	Outputs []TxOutput
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

func (t *Transaction) Serialize() []byte {
	var b bytes.Buffer
	binary.Write(&b, binary.BigEndian, uint32(len(t.Inputs)))
	for _, in := range t.Inputs {
		b.Write(in.Prev.TxID[:])
		binary.Write(&b, binary.BigEndian, in.Prev.Index)
		binary.Write(&b, binary.BigEndian, in.Sequence)
		writeBytes(&b, in.Signature)
		writeBytes(&b, in.PubKey)
	}
	binary.Write(&b, binary.BigEndian, uint32(len(t.Outputs)))
	for _, o := range t.Outputs {
		binary.Write(&b, binary.BigEndian, o.Value)
		writeBytes(&b, o.PubKeyHash)
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

func (t *Transaction) Size() int64 { return int64(len(t.Serialize())) }

func (t *Transaction) TrimmedCopy() *Transaction {
	ins := make([]TxInput, len(t.Inputs))
	for i, in := range t.Inputs {
		ins[i] = TxInput{Prev: in.Prev, Sequence: in.Sequence}
	}
	outs := make([]TxOutput, len(t.Outputs))
	for i, o := range t.Outputs {
		outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
	}
	return &Transaction{Inputs: ins, Outputs: outs}
}

func (t *Transaction) SigHash(i int, prevPKH []byte) [32]byte {
	c := t.TrimmedCopy()
	c.Inputs[i].PubKey = prevPKH
	return c.TxID()
}

func hash160(b []byte) []byte {
	s := sha256.Sum256(b)
	r := ripemd160.New()
	r.Write(s[:])
	return r.Sum(nil)
}

func Subsidy(height int64) int64 {
	h := height / HalvingPeriod
	if h >= 64 {
		return 0
	}
	return InitialReward >> uint(h)
}

func NewCoinbase(height int64, to []byte, value int64) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height))
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data, Sequence: 0xffffffff}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
}

const tagLeaf, tagNode byte = 0x00, 0x01

func MerkleRoot(txs []*Transaction) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256([]byte{tagLeaf})
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		id := t.TxID()
		level[i] = sha256.Sum256(append([]byte{tagLeaf}, id[:]...))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i])
				continue
			}
			b := append([]byte{tagNode}, level[i][:]...)
			next = append(next, sha256.Sum256(append(b, level[i+1][:]...)))
		}
		level = next
	}
	return level[0]
}

// ------------------------------------------------------------ UTXO set (10)

type UTXOEntry struct {
	Out      TxOutput
	Height   int64
	Coinbase bool
}

type UTXOSet map[Outpoint]UTXOEntry

func (s UTXOSet) Balance(pkh []byte) int64 {
	var n int64
	for _, e := range s {
		if bytes.Equal(e.Out.PubKeyHash, pkh) {
			n += e.Out.Value
		}
	}
	return n
}

// ----------------------------------------------------------------- the chain

type Block struct {
	Header Header
	Txs    []*Transaction
}

var (
	ErrBadPoW       = errors.New("insufficient proof of work")
	ErrNoCoinbase   = errors.New("first transaction is not a coinbase")
	ErrBadMerkle    = errors.New("merkle root mismatch")
	ErrOversize     = errors.New("block is too large")
	ErrMissingInput = errors.New("input is not in the UTXO set")
	ErrDoubleSpend  = errors.New("outpoint spent twice in one block")
	ErrImmature     = errors.New("coinbase output is not yet spendable")
	ErrBadSig       = errors.New("signature does not verify")
	ErrKeyMismatch  = errors.New("public key does not match the output")
	ErrNotCovered   = errors.New("outputs exceed inputs")
	ErrOverClaim    = errors.New("coinbase claims more than subsidy plus fees")
)

type Chain struct {
	blocks []Block
	utxo   UTXOSet
}

func NewChain(minerPKH []byte) *Chain {
	cb := NewCoinbase(0, minerPKH, Subsidy(0))
	txs := []*Transaction{cb}
	h := Mine(Header{Version: HeaderVersion, MerkleRoot: MerkleRoot(txs),
		Timestamp: 1700000000, Bits: 0x2000ffff, Height: 0})
	c := &Chain{utxo: UTXOSet{}}
	c.blocks = append(c.blocks, Block{h, txs})
	id := cb.TxID()
	c.utxo[Outpoint{id, 0}] = UTXOEntry{cb.Outputs[0], 0, true}
	return c
}

func (c *Chain) Tip() Header   { return c.blocks[len(c.blocks)-1].Header }
func (c *Chain) Height() int64 { return int64(c.Tip().Height) }

// Connect validates a block against the UTXO set and returns the fees. It
// mutates nothing until it has finished (lesson 08's validate-then-mutate).
func (c *Chain) Connect(b Block) (int64, UTXOSet, []Outpoint, error) {
	if !CheckPoW(b.Header) {
		return 0, nil, nil, ErrBadPoW
	}
	if len(b.Txs) == 0 || !b.Txs[0].IsCoinbase() {
		return 0, nil, nil, ErrNoCoinbase
	}
	if MerkleRoot(b.Txs) != b.Header.MerkleRoot {
		return 0, nil, nil, ErrBadMerkle
	}
	var body int64
	for _, t := range b.Txs[1:] {
		body += t.Size()
	}
	if body > MaxBlockSize {
		return 0, nil, nil, fmt.Errorf("%w: %d > %d", ErrOversize, body, MaxBlockSize)
	}

	height := int64(b.Header.Height)
	create := UTXOSet{}
	var spend []Outpoint
	spentBy := map[Outpoint]int{}
	var fees int64

	for i, t := range b.Txs {
		var in int64
		if !t.IsCoinbase() {
			for k, input := range t.Inputs {
				op := input.Prev
				if j, dup := spentBy[op]; dup {
					return 0, nil, nil, fmt.Errorf("tx %d: %w (also tx %d)", i, ErrDoubleSpend, j)
				}
				e, ok := c.utxo[op]
				if !ok {
					if e2, ok2 := create[op]; ok2 {
						e, ok = e2, true
					}
				}
				if !ok {
					return 0, nil, nil, fmt.Errorf("tx %d: %w", i, ErrMissingInput)
				}
				if e.Coinbase && height-e.Height < CoinbaseMaturity {
					return 0, nil, nil, fmt.Errorf("tx %d: %w", i, ErrImmature)
				}
				if !bytes.Equal(hash160(input.PubKey), e.Out.PubKeyHash) {
					return 0, nil, nil, fmt.Errorf("tx %d: %w", i, ErrKeyMismatch)
				}
				sh := t.SigHash(k, e.Out.PubKeyHash)
				if len(input.Signature) != 65 || !crypto.VerifySignature(input.PubKey, sh[:], input.Signature[:64]) {
					return 0, nil, nil, fmt.Errorf("tx %d: %w", i, ErrBadSig)
				}
				spentBy[op] = i
				spend = append(spend, op)
				in += e.Out.Value
			}
		}
		var out int64
		for _, o := range t.Outputs {
			if o.Value < 0 || o.Value > MaxMoney {
				return 0, nil, nil, fmt.Errorf("tx %d: value out of range", i)
			}
			out += o.Value
		}
		if !t.IsCoinbase() {
			if out > in {
				return 0, nil, nil, fmt.Errorf("tx %d: %w", i, ErrNotCovered)
			}
			fees += in - out
		}
		id := t.TxID()
		for k, o := range t.Outputs {
			create[Outpoint{id, uint32(k)}] = UTXOEntry{o, height, t.IsCoinbase()}
		}
	}

	var claimed int64
	for _, o := range b.Txs[0].Outputs {
		claimed += o.Value
	}
	if allowed := Subsidy(height) + fees; claimed > allowed {
		return 0, nil, nil, fmt.Errorf("%w: %s > %s", ErrOverClaim, btc(claimed), btc(allowed))
	}
	return fees, create, spend, nil
}

func (c *Chain) Append(b Block) error {
	_, create, spend, err := c.Connect(b)
	if err != nil {
		return err
	}
	// Insert FIRST, then delete. An output created and spent inside the same
	// block appears in both sets; deleting first would re-create it and mint
	// money out of nothing.
	for op, e := range create {
		c.utxo[op] = e
	}
	for _, op := range spend {
		delete(c.utxo, op)
	}
	c.blocks = append(c.blocks, b)
	return nil
}

// ------------------------------------------------------------- mempool (NEW)

type PoolEntry struct {
	Tx        *Transaction
	Name      string
	Size      int64
	Fee       int64
	Parent    [32]byte // zero if all inputs are confirmed
	HasParent bool
}

func (e *PoolEntry) Rate() int64 { return e.Fee / e.Size }

type Mempool struct {
	byID     map[[32]byte]*PoolEntry
	claimed  map[Outpoint][32]byte
	outputs  map[Outpoint]TxOutput
	children map[[32]byte][][32]byte
	bytes    int64
	floor    int64
}

func NewMempool() *Mempool {
	return &Mempool{byID: map[[32]byte]*PoolEntry{}, claimed: map[Outpoint][32]byte{},
		outputs: map[Outpoint]TxOutput{}, children: map[[32]byte][][32]byte{}}
}

var (
	ErrPoolDup      = errors.New("already in the pool")
	ErrPoolOrphan   = errors.New("parent not known")
	ErrPoolConflict = errors.New("conflicts with a pool transaction")
	ErrPoolCheap    = errors.New("below the pool floor")
	ErrRBFNoSignal  = errors.New("BIP-125 rule 1: original does not signal replaceability")
	ErrRBFFee       = errors.New("BIP-125 rules 3/4: fee does not beat what it evicts")
)

func (m *Mempool) lookup(c *Chain, op Outpoint) (TxOutput, bool, bool) {
	if e, ok := c.utxo[op]; ok {
		return e.Out, false, true
	}
	o, ok := m.outputs[op]
	return o, true, ok
}

// Accept runs the admission checks of example 9, plus RBF from example 14.
func (m *Mempool) Accept(c *Chain, name string, t *Transaction) (evicted []string, err error) {
	id := t.TxID()
	if _, ok := m.byID[id]; ok {
		return nil, ErrPoolDup
	}
	size := t.Size()

	prevOuts := make([]TxOutput, 0, len(t.Inputs))
	var parent [32]byte
	hasParent := false
	for _, in := range t.Inputs {
		o, fromPool, ok := m.lookup(c, in.Prev)
		if !ok {
			return nil, fmt.Errorf("%w: %x…:%d", ErrPoolOrphan, in.Prev.TxID[:6], in.Prev.Index)
		}
		if fromPool {
			parent, hasParent = in.Prev.TxID, true
		}
		prevOuts = append(prevOuts, o)
	}

	var in, out int64
	for _, p := range prevOuts {
		in += p.Value
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if out > in {
		return nil, ErrNotCovered
	}
	fee := in - out
	if fee/size < m.floor {
		return nil, fmt.Errorf("%w: %d < %d sat/byte", ErrPoolCheap, fee/size, m.floor)
	}

	// Conflicts -> RBF, or rejection.
	conflicts := map[[32]byte]bool{}
	for _, i := range t.Inputs {
		if other, ok := m.claimed[i.Prev]; ok {
			conflicts[other] = true
		}
	}
	if len(conflicts) > 0 {
		var victims []*PoolEntry
		for cid := range conflicts {
			victims = append(victims, m.family(cid)...)
		}
		var vfee int64
		for _, v := range victims {
			vfee += v.Fee
			if !signalsRBF(v.Tx) {
				return nil, fmt.Errorf("%w (%s)", ErrRBFNoSignal, v.Name)
			}
		}
		if fee < vfee+MinRelayRate*size {
			return nil, fmt.Errorf("%w: %d < %d", ErrRBFFee, fee, vfee+MinRelayRate*size)
		}
		for _, v := range victims {
			evicted = append(evicted, v.Name)
		}
		sort.Strings(evicted)
		for cid := range conflicts {
			m.drop(cid)
		}
	}

	for _, i := range t.Inputs {
		if _, ok := m.claimed[i.Prev]; ok {
			return nil, ErrPoolConflict
		}
	}
	if err := verify(t, prevOuts); err != nil {
		return nil, err
	}

	e := &PoolEntry{Tx: t, Name: name, Size: size, Fee: fee, Parent: parent, HasParent: hasParent}
	m.byID[id] = e
	m.bytes += size
	for _, i := range t.Inputs {
		m.claimed[i.Prev] = id
	}
	for k, o := range t.Outputs {
		m.outputs[Outpoint{id, uint32(k)}] = o
	}
	if hasParent {
		m.children[parent] = append(m.children[parent], id)
	}
	m.trim(&evicted)
	return evicted, nil
}

const rbfSignal = 0xfffffffd

func signalsRBF(t *Transaction) bool {
	for _, in := range t.Inputs {
		if in.Sequence < 0xfffffffe {
			return true
		}
	}
	return false
}

func (m *Mempool) family(id [32]byte) []*PoolEntry {
	e, ok := m.byID[id]
	if !ok {
		return nil
	}
	out := []*PoolEntry{e}
	for _, c := range m.children[id] {
		out = append(out, m.family(c)...)
	}
	return out
}

func (m *Mempool) drop(id [32]byte) {
	for _, e := range m.family(id) {
		eid := e.Tx.TxID()
		for _, i := range e.Tx.Inputs {
			delete(m.claimed, i.Prev)
		}
		for k := range e.Tx.Outputs {
			delete(m.outputs, Outpoint{eid, uint32(k)})
		}
		delete(m.children, eid)
		delete(m.byID, eid)
		m.bytes -= e.Size
	}
}

// trim evicts the lowest fee rate until the pool is under budget, raising the
// floor as it goes (example 13).
func (m *Mempool) trim(evicted *[]string) {
	for m.bytes > MaxPoolBytes {
		all := m.entries()
		if len(all) == 0 {
			return
		}
		worst := all[len(all)-1]
		if r := worst.Rate() + 1; r > m.floor {
			m.floor = r
		}
		*evicted = append(*evicted, worst.Name+" (evicted)")
		m.drop(worst.Tx.TxID())
	}
}

// entries returns the pool sorted by ANCESTOR fee rate, best first — the
// order a miner walks (example 16).
func (m *Mempool) entries() []*PoolEntry {
	out := make([]*PoolEntry, 0, len(m.byID))
	for _, e := range m.byID {
		out = append(out, e)
	}
	rate := func(e *PoolEntry) (int64, int64) {
		fee, size := e.Fee, e.Size
		for p := e; p.HasParent; {
			q, ok := m.byID[p.Parent]
			if !ok {
				break
			}
			fee += q.Fee
			size += q.Size
			p = q
		}
		return fee, size
	}
	sort.Slice(out, func(i, j int) bool {
		fi, si := rate(out[i])
		fj, sj := rate(out[j])
		if a, b := fi*sj, fj*si; a != b {
			return a > b
		}
		return out[i].Name < out[j].Name
	})
	return out
}

// ancestorsOf returns e's unconfirmed ancestors, parents first.
func (m *Mempool) ancestorsOf(e *PoolEntry, done map[[32]byte]bool) []*PoolEntry {
	var chain []*PoolEntry
	for p := e; ; {
		chain = append([]*PoolEntry{p}, chain...)
		if !p.HasParent {
			break
		}
		q, ok := m.byID[p.Parent]
		if !ok || done[p.Parent] {
			break
		}
		p = q
	}
	return chain
}

// Template builds the body of the next block: packages by ancestor fee rate,
// parents before children, under the size cap.
func (m *Mempool) Template() ([]*Transaction, int64, int64) {
	done := map[[32]byte]bool{}
	var body []*Transaction
	var used, fees int64
	for _, e := range m.entries() {
		id := e.Tx.TxID()
		if done[id] {
			continue
		}
		pkg := m.ancestorsOf(e, done)
		var sz, fee int64
		for _, p := range pkg {
			if done[p.Tx.TxID()] {
				continue
			}
			sz += p.Size
			fee += p.Fee
		}
		if used+sz > MaxBlockSize {
			continue // SKIP, do not STOP (example 12)
		}
		for _, p := range pkg {
			pid := p.Tx.TxID()
			if done[pid] {
				continue
			}
			done[pid] = true
			body = append(body, p.Tx)
		}
		used += sz
		fees += fee
	}
	return body, used, fees
}

// Confirmed drops everything a new block contained, and anything that
// conflicts with it.
func (m *Mempool) Confirmed(b Block) {
	for _, t := range b.Txs {
		m.drop(t.TxID())
		for _, in := range t.Inputs {
			if id, ok := m.claimed[in.Prev]; ok {
				m.drop(id)
			}
		}
	}
}

func verify(t *Transaction, prevOuts []TxOutput) error {
	for i, in := range t.Inputs {
		if len(in.Signature) != 65 || len(in.PubKey) != 33 {
			return fmt.Errorf("input %d: not signed", i)
		}
		if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
			return fmt.Errorf("input %d: %w", i, ErrKeyMismatch)
		}
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
			return fmt.Errorf("input %d: %w", i, ErrBadSig)
		}
	}
	return nil
}

// -------------------------------------------------------------- wallet (NEW)

type Wallet struct {
	Name string
	priv *ecdsa.PrivateKey
}

func NewWallet(name, hexKey string) *Wallet {
	k, err := crypto.HexToECDSA(hexKey)
	if err != nil {
		panic(err)
	}
	return &Wallet{name, k}
}

func (w *Wallet) PKH() []byte { return hash160(crypto.CompressPubkey(&w.priv.PublicKey)) }

// Sizes of OUR serialization, so the estimate matches the serializer exactly
// (example 4). Bitcoin's segwit-discounted numbers are 11 / 68 / 31.
const (
	overheadSize = 8
	inputSize    = 146 // 32 txid + 4 index + 4 sequence + 4+65 sig + 4+33 pubkey
	outputSize   = 32
	dustLimit    = 294
)

func estimate(nIn, nOut int) int64 {
	return int64(overheadSize + nIn*inputSize + nOut*outputSize)
}

var ErrInsufficient = errors.New("insufficient funds at this fee rate")

// Pay selects on EFFECTIVE values, so the fee/size circularity never appears
// (example 7), then signs every input last (lesson 10, example 7).
//
// A wallet has to know what it has already spent: a coin claimed by something
// in the mempool is not available, or the second payment is an accidental
// double-spend. `reuse` is the deliberate exception, for a fee bump.
func (w *Wallet) Pay(c *Chain, m *Mempool, to []byte, amount, rate int64, reuse map[Outpoint]bool) (*Transaction, error) {
	type coin struct {
		op  Outpoint
		out TxOutput
	}
	available := func(op Outpoint) bool {
		if _, claimed := m.claimed[op]; claimed {
			return reuse[op]
		}
		return true
	}
	var coins []coin
	height := c.Height() + 1
	for op, e := range c.utxo {
		if !bytes.Equal(e.Out.PubKeyHash, w.PKH()) || !available(op) {
			continue
		}
		if e.Coinbase && height-e.Height < CoinbaseMaturity {
			continue
		}
		coins = append(coins, coin{op, e.Out})
	}
	// Outputs of transactions still in the pool are spendable too — that is
	// what makes a chain of unconfirmed transactions possible (example 9).
	for op, o := range m.outputs {
		if bytes.Equal(o.PubKeyHash, w.PKH()) && available(op) {
			coins = append(coins, coin{op, o})
		}
	}
	sort.Slice(coins, func(i, j int) bool {
		if coins[i].out.Value != coins[j].out.Value {
			return coins[i].out.Value > coins[j].out.Value
		}
		return bytes.Compare(coins[i].op.TxID[:], coins[j].op.TxID[:]) < 0
	})

	target := amount + rate*(overheadSize+outputSize)
	var chosen []coin
	var sumEff, total int64
	for _, x := range coins {
		ev := x.out.Value - rate*inputSize
		if ev <= 0 {
			continue
		}
		chosen = append(chosen, x)
		sumEff += ev
		total += x.out.Value
		if sumEff >= target {
			break
		}
	}
	if sumEff < target {
		return nil, ErrInsufficient
	}

	nOut := 2
	fee := estimate(len(chosen), nOut) * rate
	change := total - amount - fee
	outs := []TxOutput{{Value: amount, PubKeyHash: to}}
	if change >= dustLimit {
		outs = append(outs, TxOutput{Value: change, PubKeyHash: w.PKH()})
	} else {
		fee = total - amount
	}

	t := &Transaction{Outputs: outs}
	prevOuts := make([]TxOutput, 0, len(chosen))
	for _, x := range chosen {
		t.Inputs = append(t.Inputs, TxInput{Prev: x.op, Sequence: rbfSignal})
		prevOuts = append(prevOuts, x.out)
	}
	pub := crypto.CompressPubkey(&w.priv.PublicKey)
	for i := range t.Inputs {
		h := t.SigHash(i, prevOuts[i].PubKeyHash)
		sig, err := crypto.Sign(h[:], w.priv)
		if err != nil {
			return nil, err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = pub
	}
	return t, nil
}

// Bump spends ONE named output back to yourself at a high fee rate. That is
// all a child-pays-for-parent bump is: a small transaction whose only purpose
// is to raise the fee rate of the package its parent sits in (example 16).
func (w *Wallet) Bump(op Outpoint, out TxOutput, rate int64) (*Transaction, error) {
	size := estimate(1, 1)
	fee := size * rate
	if out.Value-fee < dustLimit {
		return nil, ErrInsufficient
	}
	t := &Transaction{
		Inputs:  []TxInput{{Prev: op, Sequence: rbfSignal}},
		Outputs: []TxOutput{{Value: out.Value - fee, PubKeyHash: w.PKH()}},
	}
	h := t.SigHash(0, out.PubKeyHash)
	sig, err := crypto.Sign(h[:], w.priv)
	if err != nil {
		return nil, err
	}
	t.Inputs[0].Signature = sig
	t.Inputs[0].PubKey = crypto.CompressPubkey(&w.priv.PublicKey)
	return t, nil
}

// ------------------------------------------------------------------- output

func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func short(h [32]byte) string { return hex.EncodeToString(h[:6]) }

func rateOf(t *Transaction, c *Chain, m *Mempool) int64 {
	e := m.byID[t.TxID()]
	if e == nil {
		return 0
	}
	return e.Rate()
}

func main() {
	miner := NewWallet("miner", "7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6")
	alice := NewWallet("alice", "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	bob := NewWallet("bob", "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d")
	carol := NewWallet("carol", "5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a")

	c := NewChain(miner.PKH())
	pool := NewMempool()

	mineOne := func(elapsed int64) Block {
		body, used, fees := pool.Template()
		height := c.Height() + 1
		cb := NewCoinbase(height, miner.PKH(), Subsidy(height)+fees)
		txs := append([]*Transaction{cb}, body...)
		p := c.Tip()
		h := Mine(Header{Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot(txs),
			Timestamp: p.Timestamp + elapsed, Bits: 0x2000ffff, Height: uint64(height)})
		b := Block{h, txs}
		if err := c.Append(b); err != nil {
			panic(err)
		}
		pool.Confirmed(b)
		fmt.Printf("  block %-3d %s  %d tx, %d/%d bytes, fees %s\n",
			height, short(h.Hash()), len(body), used, MaxBlockSize, btc(fees))
		return b
	}

	fmt.Println("=== warm up: six empty blocks so a coinbase matures ===")
	for i := 0; i < 6; i++ {
		mineOne(60)
	}
	fmt.Printf("  miner has %s in %d coins\n", btc(c.utxo.Balance(miner.PKH())), countCoins(c, miner.PKH()))

	fmt.Println("\n=== the miner funds three wallets, two coins each ===")
	for round := 0; round < 2; round++ {
		for _, to := range []*Wallet{alice, bob, carol} {
			t, err := miner.Pay(c, pool, to.PKH(), 20*Coin, 20, nil)
			if err != nil {
				fmt.Println("  ", err)
				return
			}
			if _, err := pool.Accept(c, fmt.Sprintf("fund-%s-%d", to.Name, round+1), t); err != nil {
				fmt.Println("  ", err)
				return
			}
		}
		mineOne(60)
	}
	for _, w := range []*Wallet{alice, bob, carol} {
		fmt.Printf("  %-6s %s in %d coins\n", w.Name, btc(c.utxo.Balance(w.PKH())), countCoins(c, w.PKH()))
	}

	fmt.Printf("\n=== six payments, %d bytes of room ===\n", MaxBlockSize)
	type send struct {
		from, to *Wallet
		amt      int64
		rate     int64
		name     string
	}
	for _, s := range []send{
		{alice, bob, 1 * Coin, 40, "alice-fast"},
		{bob, carol, 2 * Coin, 30, "bob-normal"},
		{carol, alice, 1 * Coin, 25, "carol-quick"},
		{carol, bob, 1 * Coin, 20, "carol-later"},
		{alice, bob, 3 * Coin, 12, "alice-slow"},
		{bob, alice, 1 * Coin, 2, "bob-cheap"},
	} {
		t, err := s.from.Pay(c, pool, s.to.PKH(), s.amt, s.rate, nil)
		if err != nil {
			fmt.Printf("  %-12s %v\n", s.name, err)
			continue
		}
		ev, err := pool.Accept(c, s.name, t)
		if err != nil {
			fmt.Printf("  %-12s REJECTED: %v\n", s.name, err)
			continue
		}
		fmt.Printf("  %-12s %d bytes at %2d sat/byte, fee %s%s\n",
			s.name, t.Size(), rateOf(t, c, pool), btc(pool.byID[t.TxID()].Fee), evictedNote(ev))
	}

	fmt.Println("\n  the pool, in the order the miner will walk it:")
	dumpPool(pool)

	fmt.Println("\n=== mine one block ===")
	mineOne(60)
	fmt.Println("  still waiting:")
	dumpPool(pool)

	fmt.Println("\n=== bob bumps his stuck transaction (RBF) ===")
	if stuck := findEntry(pool, "bob-cheap"); stuck != nil {
		reuse := map[Outpoint]bool{}
		for _, in := range stuck.Tx.Inputs {
			reuse[in.Prev] = true
		}
		t, err := bob.Pay(c, pool, alice.PKH(), 1*Coin, 60, reuse)
		if err != nil {
			fmt.Println("  ", err)
		} else if ev, err := pool.Accept(c, "bob-bumped", t); err != nil {
			fmt.Printf("  REJECTED: %v\n", err)
		} else {
			fmt.Printf("  bob-bumped   %d bytes at %d sat/byte, fee %s%s\n",
				t.Size(), rateOf(t, c, pool), btc(pool.byID[t.TxID()].Fee), evictedNote(ev))
		}
	}

	fmt.Println("\n=== alice bumps hers with a child instead (CPFP) ===")
	if slow := findEntry(pool, "alice-slow"); slow != nil {
		id := slow.Tx.TxID()
		var op Outpoint
		var out TxOutput
		for k, o := range slow.Tx.Outputs {
			if bytes.Equal(o.PubKeyHash, alice.PKH()) {
				op, out = Outpoint{id, uint32(k)}, o
			}
		}
		child, err := alice.Bump(op, out, 90)
		if err != nil {
			fmt.Println("  ", err)
		} else if _, err := pool.Accept(c, "alice-cpfp", child); err != nil {
			fmt.Printf("  REJECTED: %v\n", err)
		} else {
			e := pool.byID[child.TxID()]
			pkgFee, pkgSize := e.Fee+slow.Fee, e.Size+slow.Size
			fmt.Printf("  alice-cpfp   %d bytes at %d sat/byte, spending alice-slow's change\n",
				e.Size, e.Rate())
			fmt.Printf("  alice-slow's package is now %s over %d bytes = %d sat/byte, up from %d\n",
				btc(pkgFee), pkgSize, pkgFee/pkgSize, slow.Rate())
		}
	}
	fmt.Println("\n  the pool now:")
	dumpPool(pool)

	fmt.Println("\n=== drain it ===")
	for i := 0; i < 3; i++ {
		mineOne(60)
	}
	fmt.Printf("  pool holds %d transactions\n", len(pool.byID))
	fmt.Println()
	fmt.Println("  Read block 10 again: alice-slow paid 12 sat/byte and was mined")
	fmt.Println("  BEFORE carol-later, which paid 20. Its child paid for it. That")
	fmt.Println("  is the whole point of ranking by ancestor rate rather than by")
	fmt.Println("  each transaction's own — and a miner who sorts the other way")
	fmt.Println("  leaves the child's fee on the table (example 16).")

	fmt.Println("\n=== final state ===")
	var total int64
	for _, w := range []*Wallet{miner, alice, bob, carol} {
		b := c.utxo.Balance(w.PKH())
		total += b
		fmt.Printf("  %-8s %-16s %d coins\n", w.Name, btc(b), countCoins(c, w.PKH()))
	}
	var issued int64
	for h := int64(0); h <= c.Height(); h++ {
		issued += Subsidy(h)
	}
	fmt.Printf("  %-8s %-16s issued by %d coinbases: %s — conservation: %v\n",
		"total", btc(total), len(c.blocks), btc(issued), total == issued)

	fmt.Println("\n--- the diff from lesson 10 ---")
	fmt.Println("  + Wallet.Pay: effective-value selection, fee from a rate, change,")
	fmt.Println("    dust handling, and signing last")
	fmt.Println("  + Mempool: admission, conflict tracking, RBF, a byte budget and")
	fmt.Println("    a floor that rises when it evicts")
	fmt.Println("  + Template: packages by ancestor fee rate, parents first, skip")
	fmt.Println("    rather than stop, under a size cap")
	fmt.Println("  + the chain tells the pool what confirmed")
	fmt.Println()
	fmt.Println("  Note that NONE of it is consensus. Delete the whole mempool and")
	fmt.Println("  the chain still validates every block correctly — it just has")
	fmt.Println("  nothing to put in one.")

	fmt.Println("\n--- what lesson 12 changes ---")
	fmt.Println("  Everything here lives in a map and dies with the process. Lesson")
	fmt.Println("  12 puts the chain and the UTXO set on disk, in a store that")
	fmt.Println("  survives a crash mid-block — which turns validate-then-mutate")
	fmt.Println("  into a real write batch, and makes the undo data of lesson 10")
	fmt.Println("  something you can actually reload.")
}

func countCoins(c *Chain, pkh []byte) int {
	n := 0
	for _, e := range c.utxo {
		if bytes.Equal(e.Out.PubKeyHash, pkh) {
			n++
		}
	}
	return n
}

func findEntry(m *Mempool, name string) *PoolEntry {
	for _, e := range m.byID {
		if e.Name == name {
			return e
		}
	}
	return nil
}

func evictedNote(ev []string) string {
	if len(ev) == 0 {
		return ""
	}
	return fmt.Sprintf("  (replaced %v)", ev)
}

func dumpPool(m *Mempool) {
	es := m.entries()
	if len(es) == 0 {
		fmt.Println("    (empty)")
		return
	}
	fmt.Printf("    %-14s %-8s %-14s %-10s %-10s %s\n",
		"name", "bytes", "fee", "own rate", "pkg rate", "parent")
	for _, e := range es {
		pf, ps := e.Fee, e.Size
		parent := ""
		if e.HasParent {
			if q, ok := m.byID[e.Parent]; ok {
				pf, ps = pf+q.Fee, ps+q.Size
				parent = q.Name
			}
		}
		if parent == "" {
			fmt.Printf("    %-14s %-8d %-14s %-10d %d\n", e.Name, e.Size, btc(e.Fee), e.Rate(), pf/ps)
			continue
		}
		fmt.Printf("    %-14s %-8d %-14s %-10d %-10d %s\n",
			e.Name, e.Size, btc(e.Fee), e.Rate(), pf/ps, parent)
	}
}
```

**Output:**

```
=== warm up: six empty blocks so a coinbase matures ===
  block 1   004416ff3431  0 tx, 0/700 bytes, fees 0.00000000
  block 2   005f3963382b  0 tx, 0/700 bytes, fees 0.00000000
  block 3   0038d982bc41  0 tx, 0/700 bytes, fees 0.00000000
  block 4   009e3908300e  0 tx, 0/700 bytes, fees 0.00000000
  block 5   000c63fa1b5f  0 tx, 0/700 bytes, fees 0.00000000
  block 6   009f1e42650e  0 tx, 0/700 bytes, fees 0.00000000
  miner has 350.00000000 in 7 coins

=== the miner funds three wallets, two coins each ===
  block 7   003c03b4dbd4  3 tx, 654/700 bytes, fees 0.00013080
  block 8   003740b8030e  3 tx, 654/700 bytes, fees 0.00013080
  alice  40.00000000 in 2 coins
  bob    40.00000000 in 2 coins
  carol  40.00000000 in 2 coins

=== six payments, 700 bytes of room ===
  alice-fast   218 bytes at 40 sat/byte, fee 0.00008720
  bob-normal   218 bytes at 30 sat/byte, fee 0.00006540
  carol-quick  218 bytes at 25 sat/byte, fee 0.00005450
  carol-later  218 bytes at 20 sat/byte, fee 0.00004360
  alice-slow   218 bytes at 12 sat/byte, fee 0.00002616
  bob-cheap    218 bytes at  2 sat/byte, fee 0.00000436

  the pool, in the order the miner will walk it:
    name           bytes    fee            own rate   pkg rate   parent
    alice-fast     218      0.00008720     40         40
    bob-normal     218      0.00006540     30         30
    carol-quick    218      0.00005450     25         25
    carol-later    218      0.00004360     20         20
    alice-slow     218      0.00002616     12         12
    bob-cheap      218      0.00000436     2          2

=== mine one block ===
  block 9   005bd8edcb6d  3 tx, 654/700 bytes, fees 0.00020710
  still waiting:
    name           bytes    fee            own rate   pkg rate   parent
    carol-later    218      0.00004360     20         20
    alice-slow     218      0.00002616     12         12
    bob-cheap      218      0.00000436     2          2

=== bob bumps his stuck transaction (RBF) ===
  bob-bumped   218 bytes at 60 sat/byte, fee 0.00013080  (replaced [bob-cheap])

=== alice bumps hers with a child instead (CPFP) ===
  alice-cpfp   186 bytes at 90 sat/byte, spending alice-slow's change
  alice-slow's package is now 0.00019356 over 404 bytes = 47 sat/byte, up from 12

  the pool now:
    name           bytes    fee            own rate   pkg rate   parent
    bob-bumped     218      0.00013080     60         60
    alice-cpfp     186      0.00016740     90         47         alice-slow
    carol-later    218      0.00004360     20         20
    alice-slow     218      0.00002616     12         12

=== drain it ===
  block 10  004f061d8872  3 tx, 622/700 bytes, fees 0.00032436
  block 11  00903ae78d93  1 tx, 218/700 bytes, fees 0.00004360
  block 12  00539ecc5c80  0 tx, 0/700 bytes, fees 0.00000000
  pool holds 0 transactions

  Read block 10 again: alice-slow paid 12 sat/byte and was mined
  BEFORE carol-later, which paid 20. Its child paid for it. That
  is the whole point of ranking by ancestor rate rather than by
  each transaction's own — and a miner who sorts the other way
  leaves the child's fee on the table (example 16).

=== final state ===
  miner    530.00057506     13 coins
  alice    37.99971924      4 coins
  bob      41.99980380      5 coins
  carol    39.99990190      3 coins
  total    650.00000000     issued by 13 coinbases: 650.00000000 — conservation: true

--- the diff from lesson 10 ---
  + Wallet.Pay: effective-value selection, fee from a rate, change,
    dust handling, and signing last
  + Mempool: admission, conflict tracking, RBF, a byte budget and
    a floor that rises when it evicts
  + Template: packages by ancestor fee rate, parents first, skip
    rather than stop, under a size cap
  + the chain tells the pool what confirmed

  Note that NONE of it is consensus. Delete the whole mempool and
  the chain still validates every block correctly — it just has
  nothing to put in one.

--- what lesson 12 changes ---
  Everything here lives in a map and dies with the process. Lesson
  12 puts the chain and the UTXO set on disk, in a store that
  survives a crash mid-block — which turns validate-then-mutate
  into a real write batch, and makes the undo data of lesson 10
  something you can actually reload.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
