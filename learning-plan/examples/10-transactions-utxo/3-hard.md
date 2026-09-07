# Step 10 — Transactions & the UTXO Model · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. Coin selection behind an interface

`🔴 hard` · *Coin selection*

Choosing which coins to spend, behind one interface. Largest-first, smallest-first and Bitcoin Core's branch-and-bound, compared on the same wallet at four payment sizes — inputs, fee, change, and the effect on the UTXO count.

**Steps:**

1. Compute each coin's EFFECTIVE value: what it is worth after paying to spend it.
2. See two coins in the wallet come out negative — that is what dust means.
3. Implement two greedy selectors and Core's exact-match branch-and-bound.
4. Compare fee, change and UTXO-count delta across four payment sizes.
5. Read the trade: fee today against fee tomorrow, and both against privacy.
6. Read the honest caveat — optimal selection is NP-hard and 'optimal' is not even well defined.

```go
package main

import (
	"errors"
	"fmt"
	"sort"
)

// ===========================================================================
// Coin selection: choosing which of your unspent outputs to spend.
//
// It looks like a knapsack problem and it is, but the objective is not just
// "hit the amount". Every extra input costs ~68 vbytes of fee, every change
// output creates a new UTXO you will pay to spend later, and the SHAPE of
// what you pick tells a chain analyst who you are (example 15).
//
// Three strategies behind one interface, compared on the same wallet.
// ===========================================================================

type Outpoint struct {
	TxID  [32]byte
	Index uint32
}

type Coin struct {
	Op    Outpoint
	Value int64
}

// vsize model for a P2WPKH spend, in vbytes.
const (
	overheadVSize = 11
	inputVSize    = 68
	outputVSize   = 31
	dustThreshold = 294 // below this an output is not worth creating
)

// Selection is what a selector returns. Change == 0 means no change output.
type Selection struct {
	Coins  []Coin
	Change int64
}

type Selector interface {
	Name() string
	Select(coins []Coin, payment, feeRate int64) (Selection, error)
}

var ErrInsufficient = errors.New("insufficient funds")

// ------------------------------------------------------- the shared arithmetic

// effectiveValue is a coin's value minus what it costs to spend it. A 400-sat
// coin at 20 sat/vB has an effective value of 400 - 1360 = -960: including it
// makes you POORER. This one line is why dust is unspendable.
func effectiveValue(c Coin, feeRate int64) int64 {
	return c.Value - feeRate*inputVSize
}

// selectionTarget is what the effective values have to reach: the payment
// plus the parts of the fee that do not depend on the inputs.
func selectionTarget(payment, feeRate int64) int64 {
	return payment + feeRate*(overheadVSize+outputVSize)
}

// finish turns a chosen set into a Selection, deciding whether change is
// worth creating.
func finish(chosen []Coin, payment, feeRate int64) Selection {
	var sumEff int64
	for _, c := range chosen {
		sumEff += effectiveValue(c, feeRate)
	}
	change := sumEff - selectionTarget(payment, feeRate) - feeRate*outputVSize
	if change < dustThreshold {
		change = 0 // drop it: the excess becomes fee
	}
	return Selection{Coins: chosen, Change: change}
}

func (s Selection) VSize() int64 {
	outs := int64(1)
	if s.Change > 0 {
		outs = 2
	}
	return overheadVSize + int64(len(s.Coins))*inputVSize + outs*outputVSize
}

func (s Selection) Total() int64 {
	var n int64
	for _, c := range s.Coins {
		n += c.Value
	}
	return n
}

func (s Selection) Fee(payment int64) int64 { return s.Total() - payment - s.Change }

// --------------------------------------------------------------- strategies

// LargestFirst: fewest inputs, smallest fee today. Leaves the small coins
// behind, so the wallet slowly fills with dust it can never afford to spend.
type LargestFirst struct{}

func (LargestFirst) Name() string { return "largest-first" }

func (LargestFirst) Select(coins []Coin, payment, feeRate int64) (Selection, error) {
	c := append([]Coin(nil), coins...)
	sort.Slice(c, func(i, j int) bool { return c[i].Value > c[j].Value })
	return greedy(c, payment, feeRate)
}

// SmallestFirst: consolidates. More inputs and a bigger fee today, but the
// UTXO set shrinks and the next payment is cheaper.
type SmallestFirst struct{}

func (SmallestFirst) Name() string { return "smallest-first" }

func (SmallestFirst) Select(coins []Coin, payment, feeRate int64) (Selection, error) {
	c := append([]Coin(nil), coins...)
	sort.Slice(c, func(i, j int) bool { return c[i].Value < c[j].Value })
	return greedy(c, payment, feeRate)
}

func greedy(sorted []Coin, payment, feeRate int64) (Selection, error) {
	target := selectionTarget(payment, feeRate)
	var chosen []Coin
	var sumEff int64
	for _, c := range sorted {
		if effectiveValue(c, feeRate) <= 0 {
			continue // spending it costs more than it is worth
		}
		chosen = append(chosen, c)
		sumEff += effectiveValue(c, feeRate)
		if sumEff >= target {
			return finish(chosen, payment, feeRate), nil
		}
	}
	return Selection{}, ErrInsufficient
}

// BranchAndBound is Bitcoin Core's. It looks for a subset whose effective
// values land in [target, target+costOfChange] — an EXACT match, needing no
// change output at all. Depth-first over coins sorted descending, with the
// classic two-branch (include / exclude) recursion and a try budget.
type BranchAndBound struct{ Fallback Selector }

func (BranchAndBound) Name() string { return "branch-and-bound" }

func (b BranchAndBound) Select(coins []Coin, payment, feeRate int64) (Selection, error) {
	c := make([]Coin, 0, len(coins))
	for _, x := range coins {
		if effectiveValue(x, feeRate) > 0 {
			c = append(c, x)
		}
	}
	sort.Slice(c, func(i, j int) bool { return c[i].Value > c[j].Value })

	target := selectionTarget(payment, feeRate)
	// Creating a change output costs its own bytes now, plus an input's
	// worth of bytes when you eventually spend it. Overshooting by less
	// than that is cheaper than making change.
	const longTermFeeRate = 10
	costOfChange := feeRate*outputVSize + longTermFeeRate*inputVSize

	var remaining int64
	for _, x := range c {
		remaining += effectiveValue(x, feeRate)
	}

	var best []Coin
	tries := 0
	var search func(i int, sum int64, rem int64, cur []Coin)
	search = func(i int, sum, rem int64, cur []Coin) {
		if best != nil || tries > 100_000 {
			return
		}
		tries++
		switch {
		case sum > target+costOfChange: // overshot the window
			return
		case sum >= target: // landed in it
			best = append([]Coin(nil), cur...)
			return
		case i == len(c) || sum+rem < target: // cannot reach it from here
			return
		}
		ev := effectiveValue(c[i], feeRate)
		search(i+1, sum+ev, rem-ev, append(cur, c[i])) // include
		search(i+1, sum, rem-ev, cur)                  // exclude
	}
	search(0, 0, remaining, nil)

	if best != nil {
		return Selection{Coins: best, Change: 0}, nil // exact: no change output
	}
	return b.Fallback.Select(coins, payment, feeRate) // Core falls back too
}

// ----------------------------------------------------------------- the wallet

func wallet() []Coin {
	vals := []int64{
		1_000_000, 500_000, 250_000, 120_000, 90_000,
		60_000, 40_000, 25_000, 12_000, 8_000,
		5_000, 3_000, 1_500, 800, 400,
	}
	coins := make([]Coin, len(vals))
	for i, v := range vals {
		var op Outpoint
		op.TxID[0] = byte(i) // the outpoints are cosmetic here; only values matter
		coins[i] = Coin{Op: op, Value: v}
	}
	return coins
}

func main() {
	coins := wallet()
	const feeRate = 20 // sat/vB

	var total int64
	for _, c := range coins {
		total += c.Value
	}
	fmt.Printf("=== the wallet: %d coins, %d sat total ===\n", len(coins), total)
	fmt.Printf("  %-12s %-14s %s\n", "value", "effective", "note")
	for _, c := range coins {
		ev := effectiveValue(c, feeRate)
		if ev <= 0 {
			fmt.Printf("  %-12d %-14d %s\n", c.Value, ev, "unspendable at this fee rate")
			continue
		}
		fmt.Printf("  %-12d %d\n", c.Value, ev)
	}
	fmt.Printf("\n  fee rate %d sat/vB -> each input costs %d sat to spend,\n", feeRate, feeRate*inputVSize)
	fmt.Printf("  each output costs %d sat to create.\n", feeRate*outputVSize)

	sels := []Selector{LargestFirst{}, SmallestFirst{}, BranchAndBound{Fallback: LargestFirst{}}}

	for _, payment := range []int64{300_000, 75_000, 1_200_000, 1_000_000} {
		fmt.Printf("\n=== paying %d sat ===\n", payment)
		fmt.Printf("  %-17s %-7s %-7s %-8s %-9s %-9s %s\n",
			"strategy", "inputs", "vsize", "fee", "change", "utxos", "fee as % of payment")
		for _, s := range sels {
			sel, err := s.Select(coins, payment, feeRate)
			if err != nil {
				fmt.Printf("  %-17s %v\n", s.Name(), err)
				continue
			}
			// Net effect on the wallet's coin count: n spent, change back or not.
			delta := -len(sel.Coins)
			if sel.Change > 0 {
				delta++
			}
			fee := sel.Fee(payment)
			fmt.Printf("  %-17s %-7d %-7d %-8d %-9d %-+9d %.2f%%\n",
				s.Name(), len(sel.Coins), sel.VSize(), fee, sel.Change, delta,
				float64(fee)*100/float64(payment))
		}
	}

	fmt.Println("\n=== reading the table ===")
	fmt.Println("  largest-first   pays the smallest fee today and produces a big")
	fmt.Println("                  change output. It never touches the small coins,")
	fmt.Println("                  so they accumulate until they are unspendable.")
	fmt.Println("  smallest-first  costs several times as much in fees, but the")
	fmt.Println("                  utxos column is strongly negative: it is cleaning")
	fmt.Println("                  up. Run it when fees are low, not when they spike.")
	fmt.Println("  branch-and-bound looks for a combination that needs NO change")
	fmt.Println("                  output. When it finds one the transaction is 31")
	fmt.Println("                  vbytes smaller, no new UTXO is created, and there")
	fmt.Println("                  is no change output for an analyst to identify.")
	fmt.Println("                  When it does not, it falls back — as Core does.")

	fmt.Println("\n=== why an interface ===")
	fmt.Println("  The selector is the piece you will replace most often: fee")
	fmt.Println("  spikes, consolidation windows, privacy modes, regulatory rules")
	fmt.Println("  about which coins may be mixed. Keeping it behind")
	fmt.Println("      Select(coins, payment, feeRate) (Selection, error)")
	fmt.Println("  means the signing path never changes, and each strategy can be")
	fmt.Println("  tested against the same wallet fixture — which is exactly what")
	fmt.Println("  the table above is.")

	fmt.Println("\n=== the honest caveat ===")
	fmt.Println("  Optimal coin selection is NP-hard, and 'optimal' is not even")
	fmt.Println("  well defined: fee now trades against fee later, and both trade")
	fmt.Println("  against privacy. Every real wallet ships a heuristic. Core runs")
	fmt.Println("  BnB first and falls back to a randomised knapsack; the point is")
	fmt.Println("  not to find the best answer but to avoid the bad ones.")
}
```

**Output:**

```
=== the wallet: 15 coins, 2115700 sat total ===
  value        effective      note
  1000000      998640
  500000       498640
  250000       248640
  120000       118640
  90000        88640
  60000        58640
  40000        38640
  25000        23640
  12000        10640
  8000         6640
  5000         3640
  3000         1640
  1500         140
  800          -560           unspendable at this fee rate
  400          -960           unspendable at this fee rate

  fee rate 20 sat/vB -> each input costs 1360 sat to spend,
  each output costs 620 sat to create.

=== paying 300000 sat ===
  strategy          inputs  vsize   fee      change    utxos     fee as % of payment
  largest-first     1       141     2820     697180    +0        0.94%
  smallest-first    10      753     15060    49440     -9        5.02%
  branch-and-bound  4       314     7000     0         -4        2.33%

=== paying 75000 sat ===
  strategy          inputs  vsize   fee      change    utxos     fee as % of payment
  largest-first     1       141     2820     922180    +0        3.76%
  smallest-first    7       549     10980    8520      -6        14.64%
  branch-and-bound  3       246     5000     0         -3        6.67%

=== paying 1200000 sat ===
  strategy          inputs  vsize   fee      change    utxos     fee as % of payment
  largest-first     2       209     4180     295820    -1        0.35%
  smallest-first    13      957     19140    895360    -12       1.59%
  branch-and-bound  5       382     8000     0         -5        0.67%

=== paying 1000000 sat ===
  strategy          inputs  vsize   fee      change    utxos     fee as % of payment
  largest-first     2       209     4180     495820    -1        0.42%
  smallest-first    12      889     17780    96720     -11       1.78%
  branch-and-bound  7       518     11000    0         -7        1.10%

=== reading the table ===
  largest-first   pays the smallest fee today and produces a big
                  change output. It never touches the small coins,
                  so they accumulate until they are unspendable.
  smallest-first  costs several times as much in fees, but the
                  utxos column is strongly negative: it is cleaning
                  up. Run it when fees are low, not when they spike.
  branch-and-bound looks for a combination that needs NO change
                  output. When it finds one the transaction is 31
                  vbytes smaller, no new UTXO is created, and there
                  is no change output for an analyst to identify.
                  When it does not, it falls back — as Core does.

=== why an interface ===
  The selector is the piece you will replace most often: fee
  spikes, consolidation windows, privacy modes, regulatory rules
  about which coins may be mixed. Keeping it behind
      Select(coins, payment, feeRate) (Selection, error)
  means the signing path never changes, and each strategy can be
  tested against the same wallet fixture — which is exactly what
  the table above is.

=== the honest caveat ===
  Optimal coin selection is NP-hard, and 'optimal' is not even
  well defined: fee now trades against fee later, and both trade
  against privacy. Every real wallet ships a heuristic. Core runs
  BnB first and falls back to a randomised knapsack; the point is
  not to find the best answer but to avoid the bad ones.
```

---

## 15. Change, dust, and what they leak

`🔴 hard` · *Coin selection*

Change is the output you pay yourself, and it is the biggest privacy leak in a UTXO chain. Part one: when is change worth creating? Part two: how an analyst picks it out. Part three: the common-input-ownership heuristic, which is the one that actually collapses a wallet.

**Steps:**

1. Tabulate the dust threshold against the fee rate and see why dust is negative, not small.
2. Decide change-or-fee across seven input values and three outcomes.
3. Run the round-number heuristic over six transactions and score it.
4. Cluster addresses with union-find over a week of ordinary activity.
5. Watch ONE consolidation transaction merge every address alice ever used, retroactively.
6. Watch a CoinJoin make the heuristic confidently wrong — which is the point of one.

```go
package main

import (
	"fmt"
	"math"
	"sort"
	"strings"
)

// ===========================================================================
// Change is the output you pay yourself. It is also the single biggest
// privacy leak in a UTXO chain, and the source of dust.
//
// Part 1: when is change worth creating at all?
// Part 2: how an analyst picks the change output out of a transaction.
// Part 3: the common-input-ownership heuristic, which is the one that
//         actually collapses a wallet into a single identity.
// ===========================================================================

const (
	overheadVSize = 11
	inputVSize    = 68
	outputVSize   = 31
)

func main() {
	dust()
	changeOrFee()
	roundNumbers()
	clustering()
	defences()
}

// ------------------------------------------------------ 1. the dust threshold

func dust() {
	fmt.Println("=== when a coin costs more to spend than it holds ===")
	fmt.Printf("  %-14s %-16s %s\n", "fee rate", "cost to spend", "coins below this are dust")
	for _, rate := range []int64{1, 3, 5, 10, 20, 50, 100, 300} {
		cost := rate * inputVSize
		fmt.Printf("  %-14s %-16d %d sat\n", fmt.Sprintf("%d sat/vB", rate), cost, cost)
	}
	fmt.Println()
	fmt.Println("  Bitcoin Core's relay policy fixes the threshold at the 3 sat/vB")
	fmt.Println("  rate — 294 sat for a P2WPKH output, 546 for a legacy P2PKH — and")
	fmt.Println("  simply refuses to relay a transaction creating anything smaller.")
	fmt.Println()
	fmt.Println("  Note what that means. A dust output is not 'small'; it is")
	fmt.Println("  NEGATIVE. Spending a 400-sat coin at 20 sat/vB costs 1360 sat, so")
	fmt.Println("  including it in a transaction reduces what you can pay. It sits")
	fmt.Println("  in the global UTXO set forever, costing every node memory, and")
	fmt.Println("  nobody will ever pay to remove it.")
	fmt.Println()
	fmt.Println("  Which is why dust is also an ATTACK: send a thousand people 500")
	fmt.Println("  sat each, wait for a wallet to sweep it up with their real coins,")
	fmt.Println("  and you have linked those addresses together for free.")
}

// ------------------------------------------------------- 2. change, or fee?

func changeOrFee() {
	fmt.Println("\n=== change, or just let the miner have it ===")
	const (
		feeRate   = 20
		payment   = 100_000
		dustLimit = 294 // Core's P2WPKH dust threshold

		withVSize = int64(overheadVSize + inputVSize + 2*outputVSize) // 141
		noVSize   = int64(overheadVSize + inputVSize + outputVSize)   // 110
	)
	feeWith, feeNoMin := int64(feeRate*withVSize), int64(feeRate*noVSize)

	fmt.Printf("  paying %d sat at %d sat/vB, from a single input\n", payment, feeRate)
	fmt.Printf("  with change:    %d vB -> fee %d\n", withVSize, feeWith)
	fmt.Printf("  without change: %d vB -> fee %d minimum\n\n", noVSize, feeNoMin)
	fmt.Printf("  %-12s %-14s %-9s %-7s %s\n", "input", "change if made", "fee paid", "vsize", "decision")
	for _, in := range []int64{106_000, 103_500, 103_200, 103_000, 102_400, 102_100, 101_500} {
		change := in - payment - feeWith
		switch {
		case change >= dustLimit:
			fmt.Printf("  %-12d %-14d %-9d %-7d create change\n", in, change, feeWith, withVSize)
		case in-payment >= feeNoMin:
			fmt.Printf("  %-12d %-14d %-9d %-7d change below %d: drop it, %d extra to the miner\n",
				in, change, in-payment, noVSize, dustLimit, in-payment-feeNoMin)
		default:
			fmt.Printf("  %-12d %-14d %-9s %-7s cannot afford the fee\n", in, change, "-", "-")
		}
	}
	fmt.Println()
	fmt.Println("  Dropping the change output is not charity. It saves 31 vbytes")
	fmt.Println("  now, saves an input's worth of fee later, and creates no new")
	fmt.Println("  UTXO — usually worth more than the few hundred satoshi given up.")
	fmt.Println("  Never create change you would not pay to spend.")
}

func max64(a, b int64) int64 {
	if a > b {
		return a
	}
	return b
}

// ----------------------------------------------- 3. which output is change?

type tx struct {
	name     string
	outputs  []int64
	changeIx int // ground truth, for scoring the heuristic
}

// guessChange applies the round-number heuristic: a human paying a human
// picks a round amount; the change is whatever is left, and is not round.
func guessChange(outs []int64) int {
	roundness := func(v int64) int {
		n := 0
		for v%10 == 0 && v > 0 {
			v /= 10
			n++
		}
		return n
	}
	best, bestScore := 0, math.MinInt
	for i, v := range outs {
		if s := -roundness(v); s > bestScore {
			best, bestScore = i, s
		}
	}
	return best
}

func fmtVals(v []int64) string {
	parts := make([]string, len(v))
	for i, x := range v {
		parts[i] = fmt.Sprint(x)
	}
	return "[" + strings.Join(parts, " ") + "]"
}

func roundNumbers() {
	fmt.Println("\n=== heuristic 1: the round-number payment ===")
	txs := []tx{
		{"buy a coffee", []int64{500_000, 1_234_567}, 1},
		{"pay an invoice", []int64{2_500_000, 118_311}, 1},
		{"pay rent", []int64{150_000_000, 47_219_004}, 1},
		{"donate", []int64{100_000, 8_442_101}, 1},
		{"sweep to an exchange", []int64{3_301_997, 2_000_000}, 0},
		{"pay another wallet of yours", []int64{4_000_000, 2_000_000}, -1},
	}
	right, scored := 0, 0
	fmt.Printf("  %-28s %-22s %-8s %s\n", "transaction", "outputs", "guessed", "correct")
	for _, t := range txs {
		g := guessChange(t.outputs)
		mark := "n/a"
		if t.changeIx >= 0 {
			scored++
			if g == t.changeIx {
				right++
				mark = "yes"
			} else {
				mark = "NO"
			}
		}
		fmt.Printf("  %-28s %-22s out[%d]   %s\n", t.name, fmtVals(t.outputs), g, mark)
	}
	fmt.Printf("\n  correct on %d of %d scoreable transactions\n", right, scored)
	fmt.Println("  The last row is why this is a heuristic and not a rule: two")
	fmt.Println("  round outputs and it has nothing to work with.")
	fmt.Println()
	fmt.Println("  Analysts stack several of these:")
	fmt.Println("    - script type: change usually matches the INPUTS' type")
	fmt.Println("    - address reuse: an output to an address seen before is not change")
	fmt.Println("    - the 'unnecessary input' test: if dropping an input would still")
	fmt.Println("      cover one output, that output is probably the payment")
	fmt.Println("    - behaviour: the output spent next by the same cluster is change")
}

// -------------------------- 4. common input ownership, and what it collapses

type unionFind map[string]string

func (u unionFind) find(x string) string {
	if _, ok := u[x]; !ok {
		u[x] = x
	}
	for u[x] != x {
		u[x] = u[u[x]]
		x = u[x]
	}
	return x
}

func (u unionFind) union(a, b string) {
	ra, rb := u.find(a), u.find(b)
	if ra != rb {
		u[ra] = rb
	}
}

func (u unionFind) clusters() [][]string {
	byRoot := map[string][]string{}
	for k := range u {
		r := u.find(k)
		byRoot[r] = append(byRoot[r], k)
	}
	out := make([][]string, 0, len(byRoot))
	for _, v := range byRoot {
		sort.Strings(v)
		out = append(out, v)
	}
	sort.Slice(out, func(i, j int) bool {
		if len(out[i]) != len(out[j]) {
			return len(out[i]) > len(out[j])
		}
		return out[i][0] < out[j][0]
	})
	return out
}

func clustering() {
	fmt.Println("\n=== heuristic 2: common input ownership ===")
	fmt.Println("  If two addresses appear as inputs of the SAME transaction, one")
	fmt.Println("  wallet signed for both — because one wallet held both keys.")
	fmt.Println("  This is the heuristic that actually deanonymises chains.")

	// A week of ordinary activity. Each entry is one transaction's inputs.
	history := [][]string{
		{"alice-1", "alice-2"},
		{"alice-3"},
		{"alice-2", "alice-4"},
		{"bob-1", "bob-2"},
		{"bob-3", "bob-1"},
		{"carol-1"},
		{"carol-2", "carol-3"},
	}

	u := unionFind{}
	for _, inputs := range history {
		for _, a := range inputs {
			u.find(a) // register
		}
		for i := 1; i < len(inputs); i++ {
			u.union(inputs[0], inputs[i])
		}
	}
	fmt.Println("\n  after ordinary use:")
	for _, c := range u.clusters() {
		fmt.Printf("    %d %s: %v\n", len(c), plural(len(c), "address", "addresses"), c)
	}
	fmt.Println("  alice-3 is still separate: it has never shared a transaction.")

	// Now alice consolidates her dust — one transaction, every address.
	consolidation := []string{"alice-1", "alice-2", "alice-3", "alice-4"}
	for i := 1; i < len(consolidation); i++ {
		u.union(consolidation[0], consolidation[i])
	}
	fmt.Println("\n  after ONE consolidation transaction by alice:")
	for _, c := range u.clusters() {
		fmt.Printf("    %d %s: %v\n", len(c), plural(len(c), "address", "addresses"), c)
	}
	fmt.Println("  Every address alice has ever used is now provably one wallet,")
	fmt.Println("  retroactively, and there is no undo. Consolidating dust saves")
	fmt.Println("  fees and costs privacy — the trade example 14 does not show.")

	// CoinJoin: several wallets in one transaction, deliberately.
	fmt.Println("\n=== what breaks the heuristic: CoinJoin ===")
	v := unionFind{}
	for _, inputs := range history {
		for _, a := range inputs {
			v.find(a)
		}
		for i := 1; i < len(inputs); i++ {
			v.union(inputs[0], inputs[i])
		}
	}
	joint := []string{"alice-1", "bob-1", "carol-1"}
	for i := 1; i < len(joint); i++ {
		v.union(joint[0], joint[i])
	}
	fmt.Printf("  one CoinJoin of %v gives:\n", joint)
	for _, c := range v.clusters() {
		fmt.Printf("    %d %s: %v\n", len(c), plural(len(c), "address", "addresses"), c)
	}
	fmt.Println("  The heuristic now says all three people are one wallet, which is")
	fmt.Println("  false. That is the point: CoinJoin does not hide anything, it")
	fmt.Println("  makes the assumption WRONG, and a heuristic that is sometimes")
	fmt.Println("  wrong is much less useful than one that is always right.")
}

func plural(n int, one, many string) string {
	if n == 1 {
		return one
	}
	return many
}

func defences() {
	fmt.Println("\n=== what a wallet actually does about it ===")
	fmt.Println("  - a fresh address for every receive, including change; the HD")
	fmt.Println("    wallet of lesson 07 exists so this costs nothing")
	fmt.Println("  - change on the same script type as the inputs, so the type")
	fmt.Println("    heuristic gets nothing")
	fmt.Println("  - avoid change entirely when possible: branch-and-bound")
	fmt.Println("    (example 14) exists as much for this as for the fee")
	fmt.Println("  - coin control: never mix outputs from different sources in one")
	fmt.Println("    transaction unless you mean to link them")
	fmt.Println("  - treat unsolicited dust as radioactive; wallets flag and")
	fmt.Println("    quarantine it rather than sweep it")
	fmt.Println()
	fmt.Println("  None of this is cryptography. It is all coin selection — which")
	fmt.Println("  is why the selector is the most security-relevant part of a")
	fmt.Println("  wallet after key storage.")
}
```

**Output:**

```
=== when a coin costs more to spend than it holds ===
  fee rate       cost to spend    coins below this are dust
  1 sat/vB       68               68 sat
  3 sat/vB       204              204 sat
  5 sat/vB       340              340 sat
  10 sat/vB      680              680 sat
  20 sat/vB      1360             1360 sat
  50 sat/vB      3400             3400 sat
  100 sat/vB     6800             6800 sat
  300 sat/vB     20400            20400 sat

  Bitcoin Core's relay policy fixes the threshold at the 3 sat/vB
  rate — 294 sat for a P2WPKH output, 546 for a legacy P2PKH — and
  simply refuses to relay a transaction creating anything smaller.

  Note what that means. A dust output is not 'small'; it is
  NEGATIVE. Spending a 400-sat coin at 20 sat/vB costs 1360 sat, so
  including it in a transaction reduces what you can pay. It sits
  in the global UTXO set forever, costing every node memory, and
  nobody will ever pay to remove it.

  Which is why dust is also an ATTACK: send a thousand people 500
  sat each, wait for a wallet to sweep it up with their real coins,
  and you have linked those addresses together for free.

=== change, or just let the miner have it ===
  paying 100000 sat at 20 sat/vB, from a single input
  with change:    141 vB -> fee 2820
  without change: 110 vB -> fee 2200 minimum

  input        change if made fee paid  vsize   decision
  106000       3180           2820      141     create change
  103500       680            2820      141     create change
  103200       380            2820      141     create change
  103000       180            3000      110     change below 294: drop it, 800 extra to the miner
  102400       -420           2400      110     change below 294: drop it, 200 extra to the miner
  102100       -720           -         -       cannot afford the fee
  101500       -1320          -         -       cannot afford the fee

  Dropping the change output is not charity. It saves 31 vbytes
  now, saves an input's worth of fee later, and creates no new
  UTXO — usually worth more than the few hundred satoshi given up.
  Never create change you would not pay to spend.

=== heuristic 1: the round-number payment ===
  transaction                  outputs                guessed  correct
  buy a coffee                 [500000 1234567]       out[1]   yes
  pay an invoice               [2500000 118311]       out[1]   yes
  pay rent                     [150000000 47219004]   out[1]   yes
  donate                       [100000 8442101]       out[1]   yes
  sweep to an exchange         [3301997 2000000]      out[0]   yes
  pay another wallet of yours  [4000000 2000000]      out[0]   n/a

  correct on 5 of 5 scoreable transactions
  The last row is why this is a heuristic and not a rule: two
  round outputs and it has nothing to work with.

  Analysts stack several of these:
    - script type: change usually matches the INPUTS' type
    - address reuse: an output to an address seen before is not change
    - the 'unnecessary input' test: if dropping an input would still
      cover one output, that output is probably the payment
    - behaviour: the output spent next by the same cluster is change

=== heuristic 2: common input ownership ===
  If two addresses appear as inputs of the SAME transaction, one
  wallet signed for both — because one wallet held both keys.
  This is the heuristic that actually deanonymises chains.

  after ordinary use:
    3 addresses: [alice-1 alice-2 alice-4]
    3 addresses: [bob-1 bob-2 bob-3]
    2 addresses: [carol-2 carol-3]
    1 address: [alice-3]
    1 address: [carol-1]
  alice-3 is still separate: it has never shared a transaction.

  after ONE consolidation transaction by alice:
    4 addresses: [alice-1 alice-2 alice-3 alice-4]
    3 addresses: [bob-1 bob-2 bob-3]
    2 addresses: [carol-2 carol-3]
    1 address: [carol-1]
  Every address alice has ever used is now provably one wallet,
  retroactively, and there is no undo. Consolidating dust saves
  fees and costs privacy — the trade example 14 does not show.

=== what breaks the heuristic: CoinJoin ===
  one CoinJoin of [alice-1 bob-1 carol-1] gives:
    7 addresses: [alice-1 alice-2 alice-4 bob-1 bob-2 bob-3 carol-1]
    2 addresses: [carol-2 carol-3]
    1 address: [alice-3]
  The heuristic now says all three people are one wallet, which is
  false. That is the point: CoinJoin does not hide anything, it
  makes the assumption WRONG, and a heuristic that is sometimes
  wrong is much less useful than one that is always right.

=== what a wallet actually does about it ===
  - a fresh address for every receive, including change; the HD
    wallet of lesson 07 exists so this costs nothing
  - change on the same script type as the inputs, so the type
    heuristic gets nothing
  - avoid change entirely when possible: branch-and-bound
    (example 14) exists as much for this as for the fee
  - coin control: never mix outputs from different sources in one
    transaction unless you mean to link them
  - treat unsolicited dust as radioactive; wallets flag and
    quarantine it rather than sweep it

  None of this is cryptography. It is all coin selection — which
  is why the selector is the most security-relevant part of a
  wallet after key storage.
```

---

## 16. Coinbase maturity, and the cascade it prevents

`🔴 hard` · *Maturity*

A coinbase output cannot be spent for 100 blocks. It is the only rule in the system that exists purely because of reorgs: a coinbase has no inputs, so if its block is orphaned the coins never existed — and everything descended from them dies at once.

**Steps:**

1. Enforce the rule and try to spend at seven different depths.
2. Note the boundary: `< maturity`, not `<=`, and why one block out is a chain split.
3. Turn the rule off and build a four-deep chain of spends ending at an exchange.
4. Orphan the coinbase's block and watch all three descendants become unminable.
5. Turn the rule back on and watch the cascade fail to start.
6. Read what 100 blocks costs a miner, and where the same idea appears elsewhere.

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
// Coinbase maturity: a coinbase output cannot be spent for 100 blocks.
//
// It is the only rule in the system that exists purely because of REORGS.
// A coinbase has no inputs, so if the block that created it is orphaned the
// coins simply cease to have ever existed — and every transaction that spent
// them, and every transaction that spent THOSE outputs, becomes invalid at
// once. An ordinary transaction just gets re-mined; a coinbase spend cannot.
//
// This example enforces the rule, then turns it off to watch the cascade.
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

const CoinbaseMaturity = 100

// The UTXO set has to remember two extra things per entry: the height it was
// created at, and whether it came from a coinbase.
type Entry struct {
	Out      TxOutput
	Height   int64
	Coinbase bool
}

type UTXOSet map[Outpoint]Entry

var (
	ErrMissingInput = errors.New("input spends an output that does not exist")
	ErrImmature     = errors.New("coinbase output is not yet spendable")
)

// CheckSpend is the whole rule.
func (s UTXOSet) CheckSpend(op Outpoint, spendHeight int64) error {
	e, ok := s[op]
	if !ok {
		return fmt.Errorf("%w: %x…:%d", ErrMissingInput, op.TxID[:6], op.Index)
	}
	if e.Coinbase && spendHeight-e.Height < CoinbaseMaturity {
		return fmt.Errorf("%w: created at height %d, spendable from %d, tried at %d",
			ErrImmature, e.Height, e.Height+CoinbaseMaturity, spendHeight)
	}
	return nil
}

var nullOutpoint = Outpoint{Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
	return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}

func addr(name string) []byte { return hash160([]byte(name)) }

func coinbase(height int64, value int64, to string) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height))
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: addr(to)}},
	}
}

func spend(op Outpoint, v int64, to string) *Transaction {
	return &Transaction{
		Inputs:  []TxInput{{Prev: op, Signature: make([]byte, 65), PubKey: make([]byte, 33)}},
		Outputs: []TxOutput{{Value: v, PubKeyHash: addr(to)}},
	}
}

func (s UTXOSet) addTx(t *Transaction, height int64) {
	id := t.TxID()
	for i, o := range t.Outputs {
		s[Outpoint{id, uint32(i)}] = Entry{Out: o, Height: height, Coinbase: t.IsCoinbase()}
	}
}

func main() {
	// ---------------------------------------------------------- the rule
	set := UTXOSet{}
	cb := coinbase(10, 50, "miner")
	set.addTx(cb, 10)
	coin := Outpoint{cb.TxID(), 0}

	fmt.Printf("=== a coinbase created at height 10 ===\n")
	fmt.Printf("  outpoint %x…:0, value %d, spendable from height %d\n\n",
		coin.TxID[:6], set[coin].Out.Value, 10+CoinbaseMaturity)
	fmt.Printf("  %-16s %-8s %s\n", "spend at height", "depth", "result")
	for _, h := range []int64{11, 50, 100, 109, 110, 111, 500} {
		err := set.CheckSpend(coin, h)
		res := "accepted"
		if err != nil {
			res = "REJECTED: " + err.Error()
		}
		fmt.Printf("  %-16d %-8d %s\n", h, h-10, res)
	}
	fmt.Println("\n  Depth is spendHeight - createHeight, so the coin is spendable in")
	fmt.Println("  the block 100 AFTER the one that made it — the point at which it")
	fmt.Println("  has 100 confirmations. Note the comparison is `< maturity`, not")
	fmt.Println("  `<=`: getting that boundary wrong by one block is not a bug, it")
	fmt.Println("  is a chain split, because two nodes would disagree about whether")
	fmt.Println("  a block is valid.")

	// An ordinary output made in the same block has no such restriction.
	ord := spend(Outpoint{cb.TxID(), 0}, 49, "alice")
	set.addTx(ord, 110)
	fmt.Printf("\n  an ORDINARY output created at height 110, spent at 111: %v\n",
		set.CheckSpend(Outpoint{ord.TxID(), 0}, 111) == nil)

	// ------------------------------------------------------- the reorg
	fmt.Println("\n=== why the rule exists ===")
	fmt.Println("  Build a chain where the rule is NOT enforced:")

	loose := UTXOSet{}
	cb10 := coinbase(10, 50, "miner")
	loose.addTx(cb10, 10)
	c10 := Outpoint{cb10.TxID(), 0}

	// Height 11: the miner immediately pays alice.
	t1 := spend(c10, 50, "alice")
	loose.addTx(t1, 11)
	delete(loose, c10)

	// Height 12: alice pays bob.
	t2 := spend(Outpoint{t1.TxID(), 0}, 50, "bob")
	loose.addTx(t2, 12)
	delete(loose, Outpoint{t1.TxID(), 0})

	// Height 13: bob pays an exchange, who credits his account and lets him
	// withdraw dollars.
	t3 := spend(Outpoint{t2.TxID(), 0}, 50, "exchange")
	loose.addTx(t3, 13)
	delete(loose, Outpoint{t2.TxID(), 0})

	fmt.Println("    height 10  coinbase 50 -> miner")
	fmt.Println("    height 11  miner  -> alice")
	fmt.Println("    height 12  alice  -> bob")
	fmt.Println("    height 13  bob    -> exchange   (who pays out real money)")
	t3id := t3.TxID()
	fmt.Printf("    the exchange holds %x…:0, worth %d\n",
		t3id[:6], loose[Outpoint{t3id, 0}].Out.Value)

	fmt.Println("\n  Now a competing branch wins from height 10. Block 10 is orphaned.")

	// Rebuild the set from the winning branch: block 10 never happened, so
	// its coinbase never existed.
	fmt.Println("\n  what happens to each transaction, in order:")
	type step struct {
		height int64
		name   string
		in     Outpoint
		tx     *Transaction
	}
	steps := []step{
		{11, "miner -> alice", c10, t1},
		{12, "alice -> bob", Outpoint{t1.TxID(), 0}, t2},
		{13, "bob -> exchange", Outpoint{t2.TxID(), 0}, t3},
	}
	rebuilt := UTXOSet{} // block 10 is gone; nothing exists yet
	for _, st := range steps {
		err := rebuilt.CheckSpend(st.in, st.height)
		if err != nil {
			fmt.Printf("    height %d  %-18s INVALID: %v\n", st.height, st.name, ErrMissingInput)
			continue
		}
		rebuilt.addTx(st.tx, st.height)
	}
	fmt.Println("\n  Three transactions, none of which did anything wrong, all dead.")
	fmt.Println("  They cannot even be re-mined: their inputs do not exist on any")
	fmt.Println("  chain. The exchange has paid out against coins that no longer")
	fmt.Println("  exist, and there is nobody to claw them back from.")

	fmt.Println("\n=== the same chain, with maturity enforced ===")
	strict := UTXOSet{}
	strict.addTx(cb10, 10)
	err := strict.CheckSpend(c10, 11)
	fmt.Printf("  height 11  miner -> alice   %v\n", err)
	fmt.Println("  The cascade never starts. Nothing downstream of a coinbase can")
	fmt.Println("  exist until the coinbase is 100 blocks deep — and a reorg that")
	fmt.Println("  deep has never happened by accident on Bitcoin. The worst was")
	fmt.Println("  24 blocks, in the March 2013 v0.7/v0.8 fork.")

	fmt.Println("\n=== what this costs the miner ===")
	fmt.Println("  At 10-minute blocks, 100 blocks is about 16 hours and 40 minutes")
	fmt.Println("  of locked-up revenue. Pools that pay out immediately are")
	fmt.Println("  therefore fronting their own money, which is why pool payouts")
	fmt.Println("  come out of the pool's balance and not out of the block itself.")

	fmt.Println("\n=== the same idea elsewhere ===")
	fmt.Println("  Any output whose existence depends on a specific block being")
	fmt.Println("  final needs the same treatment. Ethereum has no coinbase")
	fmt.Println("  maturity rule but does have a two-epoch finality delay for the")
	fmt.Println("  same reason (lesson 16); exchanges impose their own confirmation")
	fmt.Println("  requirements on deposits, which is the same rule bought at")
	fmt.Println("  retail (lesson 09, example 12).")
}
```

**Output:**

```
=== a coinbase created at height 10 ===
  outpoint a8fb09bc7977…:0, value 50, spendable from height 110

  spend at height  depth    result
  11               1        REJECTED: coinbase output is not yet spendable: created at height 10, spendable from 110, tried at 11
  50               40       REJECTED: coinbase output is not yet spendable: created at height 10, spendable from 110, tried at 50
  100              90       REJECTED: coinbase output is not yet spendable: created at height 10, spendable from 110, tried at 100
  109              99       REJECTED: coinbase output is not yet spendable: created at height 10, spendable from 110, tried at 109
  110              100      accepted
  111              101      accepted
  500              490      accepted

  Depth is spendHeight - createHeight, so the coin is spendable in
  the block 100 AFTER the one that made it — the point at which it
  has 100 confirmations. Note the comparison is `< maturity`, not
  `<=`: getting that boundary wrong by one block is not a bug, it
  is a chain split, because two nodes would disagree about whether
  a block is valid.

  an ORDINARY output created at height 110, spent at 111: true

=== why the rule exists ===
  Build a chain where the rule is NOT enforced:
    height 10  coinbase 50 -> miner
    height 11  miner  -> alice
    height 12  alice  -> bob
    height 13  bob    -> exchange   (who pays out real money)
    the exchange holds 6c4f70519155…:0, worth 50

  Now a competing branch wins from height 10. Block 10 is orphaned.

  what happens to each transaction, in order:
    height 11  miner -> alice     INVALID: input spends an output that does not exist
    height 12  alice -> bob       INVALID: input spends an output that does not exist
    height 13  bob -> exchange    INVALID: input spends an output that does not exist

  Three transactions, none of which did anything wrong, all dead.
  They cannot even be re-mined: their inputs do not exist on any
  chain. The exchange has paid out against coins that no longer
  exist, and there is nobody to claw them back from.

=== the same chain, with maturity enforced ===
  height 11  miner -> alice   coinbase output is not yet spendable: created at height 10, spendable from 110, tried at 11
  The cascade never starts. Nothing downstream of a coinbase can
  exist until the coinbase is 100 blocks deep — and a reorg that
  deep has never happened by accident on Bitcoin. The worst was
  24 blocks, in the March 2013 v0.7/v0.8 fork.

=== what this costs the miner ===
  At 10-minute blocks, 100 blocks is about 16 hours and 40 minutes
  of locked-up revenue. Pools that pay out immediately are
  therefore fronting their own money, which is why pool payouts
  come out of the pool's balance and not out of the block itself.

=== the same idea elsewhere ===
  Any output whose existence depends on a specific block being
  final needs the same treatment. Ethereum has no coinbase
  maturity rule but does have a two-epoch finality delay for the
  same reason (lesson 16); exchanges impose their own confirmation
  requirements on deposits, which is the same rule bought at
  retail (lesson 09, example 12).
```

---

## 17. The value overflow incident

`🔴 hard` · *Value*

CVE-2010-5139, reproduced with the real numbers. One transaction in block 74638 spent 0.5 BTC and created two outputs of 92,233,720,368.54277039 BTC. Their sum overflows int64 and comes out **negative**, so `sum(outputs) > sum(inputs)` was false.

**Steps:**

1. Print the two output values and the true sum, then the int64 sum: -997538.
2. Note that this program could not be written with constants — Go rejects those at compile time.
3. Run four validators over it: naive, range-checked, checked-addition, and big.Int.
4. Confirm the three fixes agree with the naive one on honest transactions.
5. Read what happened next: a patch in five hours and the only deliberate rewrite of Bitcoin's history.
6. Meet the same bug in 2018 as batchOverflow, and the fix Solidity shipped in 2020.

```go
package main

import (
	"errors"
	"fmt"
	"math"
	"math/big"
)

// ===========================================================================
// CVE-2010-5139 — the value overflow incident, 15 August 2010.
//
// One transaction in block 74638 spent 0.5 BTC and created two outputs of
// 92,233,720,368.54277039 BTC each. Their sum overflows a signed 64-bit
// integer and comes out NEGATIVE, so the check
//
//     if sum(outputs) > sum(inputs) { reject }
//
// passed. 184.5 billion BTC existed for five hours.
//
// This example reproduces it with the real numbers, then fixes it three ways.
// ===========================================================================

const (
	Coin     = int64(100_000_000)
	MaxMoney = 21_000_000 * Coin
)

// The two output values from the actual transaction, in satoshis.
const attackOutput = int64(9_223_372_036_854_277_039)

type Output struct {
	Value int64
	Label string
}

var (
	ErrNotCovered = errors.New("outputs exceed inputs")
	ErrRange      = errors.New("value outside the legal range")
	ErrOverflow   = errors.New("value sum overflows")
)

// ------------------------------------------------- 1. the validator as written

// CheckNaive is what Bitcoin did before 0.3.10. Every line of it is
// reasonable-looking, and it is exploitable.
func CheckNaive(in int64, outs []Output) error {
	var sum int64
	for _, o := range outs {
		sum += o.Value // <- wraps silently, no panic, no vet warning
	}
	if sum > in {
		return fmt.Errorf("%w: in %d, out %d", ErrNotCovered, in, sum)
	}
	return nil
}

// ------------------------------------------------------------- 2. three fixes

// CheckRange is Bitcoin's actual fix: MoneyRange() on every value AND on the
// running total. Bounding each term below 21e14 makes an overflow of a sum of
// any plausible number of them impossible.
func CheckRange(in int64, outs []Output) error {
	if in < 0 || in > MaxMoney {
		return fmt.Errorf("input: %w", ErrRange)
	}
	var sum int64
	for i, o := range outs {
		if o.Value < 0 || o.Value > MaxMoney {
			return fmt.Errorf("output %d: %w (%d)", i, ErrRange, o.Value)
		}
		sum += o.Value
		if sum < 0 || sum > MaxMoney {
			return fmt.Errorf("output %d: %w", i, ErrRange)
		}
	}
	if sum > in {
		return fmt.Errorf("%w: in %d, out %d", ErrNotCovered, in, sum)
	}
	return nil
}

// addChecked is the general tool for when there is no domain bound to lean on.
func addChecked(a, b int64) (int64, bool) {
	s := a + b
	// Overflow happened iff the operands share a sign and the result differs.
	if (a > 0 && b > 0 && s < 0) || (a < 0 && b < 0 && s > 0) {
		return 0, false
	}
	return s, true
}

func CheckChecked(in int64, outs []Output) error {
	var sum int64
	for i, o := range outs {
		var ok bool
		if sum, ok = addChecked(sum, o.Value); !ok {
			return fmt.Errorf("output %d: %w", i, ErrOverflow)
		}
	}
	if sum > in {
		return fmt.Errorf("%w: in %d, out %d", ErrNotCovered, in, sum)
	}
	return nil
}

// CheckBig cannot overflow at all — at the cost of an allocation per add.
func CheckBig(in int64, outs []Output) error {
	sum := new(big.Int)
	for _, o := range outs {
		sum.Add(sum, big.NewInt(o.Value))
	}
	if sum.Cmp(big.NewInt(in)) > 0 {
		return fmt.Errorf("%w: in %d, out %s", ErrNotCovered, in, sum)
	}
	return nil
}

// --------------------------------------------------------------------------

func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func btcBig(sat *big.Int) string {
	q, r := new(big.Int).QuoRem(sat, big.NewInt(Coin), new(big.Int))
	return fmt.Sprintf("%s.%08s", q, r)
}

func main() {
	fmt.Println("=== the transaction from block 74638 ===")
	in := int64(50_000_000) // 0.5 BTC
	outs := []Output{
		{attackOutput, "to the attacker"},
		{attackOutput, "to the attacker again"},
	}
	fmt.Printf("  input           %s BTC\n", btc(in))
	for _, o := range outs {
		fmt.Printf("  output          %s BTC   (%s)\n", btc(o.Value), o.Label)
	}

	fmt.Println("\n=== the arithmetic ===")
	fmt.Printf("  int64 max          %d\n", int64(math.MaxInt64))
	fmt.Printf("  one output         %d\n", attackOutput)
	fmt.Printf("  true sum           %s\n", new(big.Int).Add(big.NewInt(attackOutput), big.NewInt(attackOutput)))
	// These have to be variables. Writing `attackOutput + attackOutput` with
	// attackOutput as a constant does not compile:
	//     constant 18446744073708554078 of type int64 overflows int64
	// which is Go being helpful exactly once, in the case that never occurs
	// in real code.
	a, b := attackOutput, attackOutput
	fmt.Printf("  int64 sum          %d      <- wrapped, and NEGATIVE\n", a+b)
	fmt.Printf("  is it <= input?    %v\n", a+b <= in)
	fmt.Println()
	fmt.Println("  Go will not help here. A CONSTANT that overflows is a compile")
	fmt.Println("  error — the two values above had to be put in variables for")
	fmt.Println("  this program to build — but two int64 VARIABLES wrap silently:")
	fmt.Println("  no panic, no vet diagnostic, no -race report. As in C++.")

	fmt.Println("\n=== four validators, one transaction ===")
	checks := []struct {
		name string
		fn   func(int64, []Output) error
	}{
		{"naive sum (Bitcoin < 0.3.10)", CheckNaive},
		{"range-check each value", CheckRange},
		{"checked addition", CheckChecked},
		{"big.Int", CheckBig},
	}
	for _, c := range checks {
		err := c.fn(in, outs)
		if err == nil {
			fmt.Printf("  %-30s ACCEPTED  <- 184 billion BTC created\n", c.name)
			continue
		}
		fmt.Printf("  %-30s rejected: %v\n", c.name, err)
	}

	fmt.Println("\n=== and on ordinary transactions they all agree ===")
	ordinary := []Output{{30 * Coin, "payment"}, {19 * Coin, "change"}}
	overspend := []Output{{30 * Coin, "payment"}, {25 * Coin, "change"}}
	for _, c := range checks {
		fmt.Printf("  %-30s valid: %-6v overspend: %v\n", c.name,
			c.fn(50*Coin, ordinary) == nil, c.fn(50*Coin, overspend) == nil)
	}
	fmt.Println("  The fixes are not stricter about honest transactions. They are")
	fmt.Println("  only stricter about ones that could not exist.")

	fmt.Println("\n=== what actually happened ===")
	total := new(big.Int).Add(big.NewInt(attackOutput), big.NewInt(attackOutput))
	twoTo64 := new(big.Int).Lsh(big.NewInt(1), 64)
	fmt.Printf("  coins created   %s satoshi\n", total)
	fmt.Printf("                  = %s BTC\n", btcBig(total))
	fmt.Printf("  2^64            %s — the sum is %s short of it,\n",
		twoTo64, new(big.Int).Sub(twoTo64, total))
	fmt.Println("                  which is exactly the negative number the")
	fmt.Println("                  validator saw: -997538")
	fmt.Println("  block 74638     15 Aug 2010, 15:08 UTC")
	fmt.Println("  patched         0.3.10, about five hours later")
	fmt.Println("  resolved        the good chain overtook the bad one and ~53")
	fmt.Println("                  blocks were orphaned — the only time Bitcoin's")
	fmt.Println("                  history has been deliberately rewritten")
	fmt.Println()
	fmt.Println("  Note what the fix required: a hard fork, agreed in hours, on a")
	fmt.Println("  chain small enough that everyone could be reached. The same bug")
	fmt.Println("  today would not be fixable that way.")

	fmt.Println("\n=== the same bug, later, elsewhere ===")
	fmt.Println("  April 2018, batchOverflow (CVE-2018-10299): an ERC-20 contract")
	fmt.Println("  computed `amount * receivers.length` in unchecked Solidity. Two")
	fmt.Println("  receivers and amount = 2^255 wrapped the product to zero, so the")
	fmt.Println("  balance check passed and the loop credited 2^255 tokens twice.")
	fmt.Println("  Exchanges suspended ERC-20 deposits. Solidity made arithmetic")
	fmt.Println("  checked by default in 0.8.0, December 2020 — ten years after")
	fmt.Println("  this Bitcoin bug.")

	fmt.Println("\n=== the rule to take away ===")
	fmt.Println("  Bound every value at the edge, before it reaches arithmetic.")
	fmt.Println("  `0 <= v <= MaxMoney` on each output is one comparison and it")
	fmt.Println("  makes the whole class of bug unreachable. Checked addition and")
	fmt.Println("  big.Int are the fallbacks for when there is no such bound —")
	fmt.Println("  and uint256 (lesson 03) is what you use when there is, but it")
	fmt.Println("  is 2^256 wide.")
}
```

**Output:**

```
=== the transaction from block 74638 ===
  input           0.50000000 BTC
  output          92233720368.54277039 BTC   (to the attacker)
  output          92233720368.54277039 BTC   (to the attacker again)

=== the arithmetic ===
  int64 max          9223372036854775807
  one output         9223372036854277039
  true sum           18446744073708554078
  int64 sum          -997538      <- wrapped, and NEGATIVE
  is it <= input?    true

  Go will not help here. A CONSTANT that overflows is a compile
  error — the two values above had to be put in variables for
  this program to build — but two int64 VARIABLES wrap silently:
  no panic, no vet diagnostic, no -race report. As in C++.

=== four validators, one transaction ===
  naive sum (Bitcoin < 0.3.10)   ACCEPTED  <- 184 billion BTC created
  range-check each value         rejected: output 0: value outside the legal range (9223372036854277039)
  checked addition               rejected: output 1: value sum overflows
  big.Int                        rejected: outputs exceed inputs: in 50000000, out 18446744073708554078

=== and on ordinary transactions they all agree ===
  naive sum (Bitcoin < 0.3.10)   valid: true   overspend: false
  range-check each value         valid: true   overspend: false
  checked addition               valid: true   overspend: false
  big.Int                        valid: true   overspend: false
  The fixes are not stricter about honest transactions. They are
  only stricter about ones that could not exist.

=== what actually happened ===
  coins created   18446744073708554078 satoshi
                  = 184467440737.08554078 BTC
  2^64            18446744073709551616 — the sum is 997538 short of it,
                  which is exactly the negative number the
                  validator saw: -997538
  block 74638     15 Aug 2010, 15:08 UTC
  patched         0.3.10, about five hours later
  resolved        the good chain overtook the bad one and ~53
                  blocks were orphaned — the only time Bitcoin's
                  history has been deliberately rewritten

  Note what the fix required: a hard fork, agreed in hours, on a
  chain small enough that everyone could be reached. The same bug
  today would not be fixable that way.

=== the same bug, later, elsewhere ===
  April 2018, batchOverflow (CVE-2018-10299): an ERC-20 contract
  computed `amount * receivers.length` in unchecked Solidity. Two
  receivers and amount = 2^255 wrapped the product to zero, so the
  balance check passed and the loop credited 2^255 tokens twice.
  Exchanges suspended ERC-20 deposits. Solidity made arithmetic
  checked by default in 0.8.0, December 2020 — ten years after
  this Bitcoin bug.

=== the rule to take away ===
  Bound every value at the edge, before it reaches arithmetic.
  `0 <= v <= MaxMoney` on each output is one comparison and it
  makes the whole class of bug unreachable. Checked addition and
  big.Int are the fallbacks for when there is no such bound —
  and uint256 (lesson 03) is what you use when there is, but it
  is 2^256 wide.
```

---

## 18. Transactions in the chain

`🔴 hard` · *Assembly*

Lesson 09's chain with `Body []string` replaced by real transactions. The header, proof of work, retargeting, median-time-past and validate-then-mutate are untouched; about 200 lines of new code buys the coinbase, signatures, the UTXO set, fees, conservation and maturity.

**Steps:**

1. Read the six additions marked NEW against lesson 09's chain.
2. Mine six blocks so a coinbase matures, then make three real payments.
3. Check the balances against what the coinbases issued.
4. Watch seven rejections, one for each rule the transaction model adds.
5. Confirm every rejection left the chain AND the UTXO set untouched.
6. Revert the tip and re-apply it, the way lesson 14's reorgs will.
7. Read what lesson 11 adds next: the mempool between the wallet and the miner.

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
// Lesson 09's chain, with the body replaced by real transactions.
//
// Marked NEW against lesson 09:
//   + Block.Txs is []*Transaction, and the Merkle root is over txids
//   + tx[0] must be the coinbase, and only tx[0]
//   + a UTXO set, maintained as one delta per block
//   + per-input signature verification
//   + conservation of value, the fee, and the coinbase bound
//   + coinbase maturity
//   + reverting a block, for the reorgs of lesson 14
//
// Unchanged from lesson 09: the header layout, proof of work, the retarget
// rule, median-time-past, and validate-then-mutate. Roughly 200 lines of new
// code buys the entire transaction model.
//
// Demo parameters are small so the output fits on a screen. Bitcoin's values
// are in the comments.
// ===========================================================================

const (
	HeaderVersion  = 1
	HeaderSize     = 92
	MTPWindow      = 11
	targetSpacing  = 60 // seconds
	retargetBlocks = 8  // Bitcoin: 2016
	targetTimespan = targetSpacing * retargetBlocks

	CoinbaseMaturity = 5  // NEW. Bitcoin: 100
	HalvingPeriod    = 20 // NEW. Bitcoin: 210_000

	Coin          = int64(100_000_000)
	InitialReward = 50 * Coin
	MaxMoney      = 21_000_000 * Coin
)

// ------------------------------------------------------- header (lesson 08/09)

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

// ------------------------------------------------------- proof of work (09)

func BitsToTarget(bits uint32) *big.Int {
	exp, m := bits>>24, bits&0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

func TargetToBits(t *big.Int) uint32 {
	if t.Sign() == 0 {
		return 0
	}
	b := t.Bytes()
	exp := len(b)
	var m uint32
	if exp <= 3 {
		for i := 0; i < len(b); i++ {
			m = m<<8 | uint32(b[i])
		}
		m <<= uint(8 * (3 - exp))
	} else {
		m = uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
	}
	if m&0x00800000 != 0 {
		m >>= 8
		exp++
	}
	return uint32(exp)<<24 | m
}

var maxTarget = BitsToTarget(0x2000ffff)

func CheckPoW(h Header) bool {
	target := BitsToTarget(h.Bits)
	if target.Sign() <= 0 || target.Cmp(maxTarget) > 0 {
		return false
	}
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(target) < 0
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

// ---------------------------------------------------- transactions (NEW)

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

func Subsidy(height int64) int64 {
	h := height / HalvingPeriod
	if h >= 64 {
		return 0
	}
	return InitialReward >> uint(h)
}

func NewCoinbase(height int64, to []byte, value int64) *Transaction {
	data := make([]byte, 8)
	binary.BigEndian.PutUint64(data, uint64(height)) // BIP-34
	return &Transaction{
		Inputs:  []TxInput{{Prev: nullOutpoint, Signature: data}},
		Outputs: []TxOutput{{Value: value, PubKeyHash: to}},
	}
}

// ------------------------------------------------------------ merkle (05/08)

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

// ------------------------------------------------------------- UTXO set (NEW)

type Entry struct {
	Out      TxOutput
	Height   int64
	Coinbase bool
}

type UTXOSet map[Outpoint]Entry

type Delta struct {
	Spend  []Outpoint
	Create map[Outpoint]Entry
}

func (s UTXOSet) Apply(d Delta) {
	for _, op := range d.Spend {
		delete(s, op)
	}
	for op, e := range d.Create {
		s[op] = e
	}
}

func (s UTXOSet) Revert(d Delta, spent map[Outpoint]Entry) {
	for op := range d.Create {
		delete(s, op)
	}
	for op, e := range spent {
		s[op] = e
	}
}

func (s UTXOSet) Balance(pkh []byte) int64 {
	var n int64
	for _, e := range s {
		if bytes.Equal(e.Out.PubKeyHash, pkh) {
			n += e.Out.Value
		}
	}
	return n
}

func (s UTXOSet) Spendable(pkh []byte, height int64) []Outpoint {
	var ops []Outpoint
	for op, e := range s {
		if !bytes.Equal(e.Out.PubKeyHash, pkh) {
			continue
		}
		if e.Coinbase && height-e.Height < CoinbaseMaturity {
			continue
		}
		ops = append(ops, op)
	}
	sort.Slice(ops, func(i, j int) bool {
		if c := bytes.Compare(ops[i].TxID[:], ops[j].TxID[:]); c != 0 {
			return c < 0
		}
		return ops[i].Index < ops[j].Index
	})
	return ops
}

// -------------------------------------------------------------------- block

type Block struct {
	Header Header
	Txs    []*Transaction
}

var (
	ErrBadVersion    = errors.New("unsupported version")
	ErrEmptyBody     = errors.New("empty body")
	ErrBadMerkle     = errors.New("merkle root mismatch")
	ErrBadPoW        = errors.New("insufficient proof of work")
	ErrOrphan        = errors.New("parent not found")
	ErrBadHeight     = errors.New("height is not parent+1")
	ErrBadBits       = errors.New("bits do not match the retarget rule")
	ErrTooOld        = errors.New("timestamp at or before median-time-past")
	ErrNoCoinbase    = errors.New("first transaction is not a coinbase") // NEW
	ErrExtraCoinbase = errors.New("more than one coinbase")              // NEW
	ErrMissingInput  = errors.New("input spends an output that is not in the UTXO set")
	ErrDoubleSpend   = errors.New("outpoint spent twice in one block")      // NEW
	ErrImmature      = errors.New("coinbase output is not yet spendable")   // NEW
	ErrBadSig        = errors.New("signature does not verify")              // NEW
	ErrKeyMismatch   = errors.New("public key does not match the output")   // NEW
	ErrRange         = errors.New("value outside the legal range")          // NEW
	ErrNotCovered    = errors.New("outputs exceed inputs")                  // NEW
	ErrOverClaim     = errors.New("coinbase claims more than subsidy+fees") // NEW
)

func CheckStateless(b Block) error {
	if b.Header.Version != HeaderVersion {
		return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version)
	}
	if len(b.Txs) == 0 {
		return ErrEmptyBody
	}
	if !b.Txs[0].IsCoinbase() { // NEW
		return ErrNoCoinbase
	}
	for i, t := range b.Txs[1:] { // NEW
		if t.IsCoinbase() {
			return fmt.Errorf("tx %d: %w", i+1, ErrExtraCoinbase)
		}
	}
	if MerkleRoot(b.Txs) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	if !CheckPoW(b.Header) {
		return ErrBadPoW
	}
	return nil
}

// --------------------------------------------------------------------- chain

type Chain struct {
	byHash map[[32]byte]Block
	order  [][32]byte
	utxo   UTXOSet
	undo   []map[Outpoint]Entry // spent entries per block, for Revert
	deltas []Delta
}

func Genesis(to []byte) Block {
	cb := NewCoinbase(0, to, Subsidy(0))
	txs := []*Transaction{cb}
	h := Header{Version: HeaderVersion, MerkleRoot: MerkleRoot(txs),
		Timestamp: 1700000000, Bits: 0x2000ffff, Height: 0}
	return Block{Header: Mine(h), Txs: txs}
}

func NewChain(to []byte) *Chain {
	g := Genesis(to)
	c := &Chain{byHash: map[[32]byte]Block{}, utxo: UTXOSet{}}
	hh := g.Header.Hash()
	c.byHash[hh] = g
	c.order = append(c.order, hh)
	d, spent, _, err := c.connect(g, 0)
	if err != nil {
		panic(err)
	}
	c.utxo.Apply(d)
	c.undo = append(c.undo, spent)
	c.deltas = append(c.deltas, d)
	return c
}

func (c *Chain) Len() int       { return len(c.order) }
func (c *Chain) Tip() Block     { return c.byHash[c.order[len(c.order)-1]] }
func (c *Chain) At(i int) Block { return c.byHash[c.order[i]] }

func (c *Chain) MedianTimePast() int64 {
	n := MTPWindow
	if len(c.order) < n {
		n = len(c.order)
	}
	ts := make([]int64, 0, n)
	for _, h := range c.order[len(c.order)-n:] {
		ts = append(ts, c.byHash[h].Header.Timestamp)
	}
	sort.Slice(ts, func(i, j int) bool { return ts[i] < ts[j] })
	return ts[len(ts)/2]
}

func (c *Chain) NextBits() uint32 {
	parent := c.Tip().Header
	if (parent.Height+1)%retargetBlocks != 0 {
		return parent.Bits
	}
	first := c.At(len(c.order) - retargetBlocks).Header
	actual := parent.Timestamp - first.Timestamp
	if actual < targetTimespan/4 {
		actual = targetTimespan / 4
	}
	if actual > targetTimespan*4 {
		actual = targetTimespan * 4
	}
	t := new(big.Int).Mul(BitsToTarget(parent.Bits), big.NewInt(actual))
	t.Div(t, big.NewInt(targetTimespan))
	if t.Cmp(maxTarget) > 0 {
		t.Set(maxTarget)
	}
	return TargetToBits(t)
}

// connect is the NEW half of the validator: everything that needs the UTXO
// set. It returns the delta the block would apply and never mutates anything.
func (c *Chain) connect(b Block, height int64) (Delta, map[Outpoint]Entry, int64, error) {
	d := Delta{Create: map[Outpoint]Entry{}}
	spentEntries := map[Outpoint]Entry{}
	spentBy := map[Outpoint]int{}
	created := map[Outpoint]Entry{}
	var fees int64

	for i, t := range b.Txs {
		var in int64
		if !t.IsCoinbase() {
			for k, input := range t.Inputs {
				op := input.Prev
				if j, dup := spentBy[op]; dup {
					return Delta{}, nil, 0, fmt.Errorf("tx %d: %w: also spent by tx %d", i, ErrDoubleSpend, j)
				}
				e, ok := c.utxo[op]
				if !ok {
					if e, ok = created[op]; !ok {
						return Delta{}, nil, 0, fmt.Errorf("tx %d: %w: %x…:%d", i, ErrMissingInput, op.TxID[:6], op.Index)
					}
				}
				if e.Coinbase && height-e.Height < CoinbaseMaturity {
					return Delta{}, nil, 0, fmt.Errorf("tx %d: %w: made at %d, spendable from %d",
						i, ErrImmature, e.Height, e.Height+CoinbaseMaturity)
				}
				if !bytes.Equal(hash160(input.PubKey), e.Out.PubKeyHash) {
					return Delta{}, nil, 0, fmt.Errorf("tx %d: %w", i, ErrKeyMismatch)
				}
				sh := t.SigHash(k, e.Out.PubKeyHash)
				if len(input.Signature) != 65 || !crypto.VerifySignature(input.PubKey, sh[:], input.Signature[:64]) {
					return Delta{}, nil, 0, fmt.Errorf("tx %d: %w", i, ErrBadSig)
				}
				spentBy[op] = i
				if _, fromSet := c.utxo[op]; fromSet {
					spentEntries[op] = e
				}
				in += e.Out.Value
				d.Spend = append(d.Spend, op)
			}
		}

		var out int64
		for k, o := range t.Outputs {
			if o.Value < 0 || o.Value > MaxMoney {
				return Delta{}, nil, 0, fmt.Errorf("tx %d output %d: %w (%d)", i, k, ErrRange, o.Value)
			}
			out += o.Value
			if out > MaxMoney {
				return Delta{}, nil, 0, fmt.Errorf("tx %d: %w", i, ErrRange)
			}
		}
		if !t.IsCoinbase() {
			if out > in {
				return Delta{}, nil, 0, fmt.Errorf("tx %d: %w: in %s, out %s", i, ErrNotCovered, btc(in), btc(out))
			}
			fees += in - out
		}

		id := t.TxID()
		for k, o := range t.Outputs {
			op := Outpoint{id, uint32(k)}
			e := Entry{Out: o, Height: height, Coinbase: t.IsCoinbase()}
			created[op] = e
			d.Create[op] = e
		}
	}

	// The coinbase bound needs every other fee in the block, so it is checked
	// last (example 11).
	var claimed int64
	for _, o := range b.Txs[0].Outputs {
		claimed += o.Value
	}
	if allowed := Subsidy(height) + fees; claimed > allowed {
		return Delta{}, nil, 0, fmt.Errorf("%w: claimed %s, allowed %s", ErrOverClaim, btc(claimed), btc(allowed))
	}
	return d, spentEntries, fees, nil
}

func (c *Chain) Append(b Block) error {
	if err := CheckStateless(b); err != nil {
		return err
	}
	parent, ok := c.byHash[b.Header.PrevHash]
	if !ok {
		return fmt.Errorf("%w: %x", ErrOrphan, b.Header.PrevHash[:4])
	}
	if b.Header.Height != parent.Header.Height+1 {
		return fmt.Errorf("%w: %d after %d", ErrBadHeight, b.Header.Height, parent.Header.Height)
	}
	if want := c.NextBits(); b.Header.Bits != want {
		return fmt.Errorf("%w: got %#x, want %#x", ErrBadBits, b.Header.Bits, want)
	}
	if mtp := c.MedianTimePast(); b.Header.Timestamp <= mtp {
		return fmt.Errorf("%w: %d <= %d", ErrTooOld, b.Header.Timestamp, mtp)
	}
	d, spent, _, err := c.connect(b, int64(b.Header.Height))
	if err != nil {
		return err // NOTHING has been mutated
	}
	h := b.Header.Hash()
	c.byHash[h] = b
	c.order = append(c.order, h)
	c.utxo.Apply(d)
	c.undo = append(c.undo, spent)
	c.deltas = append(c.deltas, d)
	return nil
}

// MineNext assembles and mines a block on top of the tip.
func (c *Chain) MineNext(txs []*Transaction, minerPKH []byte, elapsed int64, extraFee int64) Block {
	p := c.Tip().Header
	height := int64(p.Height) + 1

	var fees int64
	for _, t := range txs {
		fees += c.feeOf(t)
	}
	cb := NewCoinbase(height, minerPKH, Subsidy(height)+fees+extraFee)
	body := append([]*Transaction{cb}, txs...)

	h := Header{
		Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot(body),
		Timestamp: p.Timestamp + elapsed, Bits: c.NextBits(), Height: uint64(height),
	}
	return Block{Header: Mine(h), Txs: body}
}

func (c *Chain) feeOf(t *Transaction) int64 {
	var in, out int64
	for _, i := range t.Inputs {
		if e, ok := c.utxo[i.Prev]; ok {
			in += e.Out.Value
		}
	}
	for _, o := range t.Outputs {
		out += o.Value
	}
	if in < out {
		return 0
	}
	return in - out
}

// ------------------------------------------------------------------- wallet

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

// Pay selects coins largest-first, builds the transaction, and signs every
// input. Everything before the signing has to be final (example 7).
func (w *Wallet) Pay(c *Chain, to []byte, amount, fee int64) (*Transaction, error) {
	height := int64(c.Tip().Header.Height) + 1
	ops := c.utxo.Spendable(w.PKH(), height)
	sort.Slice(ops, func(i, j int) bool { return c.utxo[ops[i]].Out.Value > c.utxo[ops[j]].Out.Value })

	var chosen []Outpoint
	var total int64
	for _, op := range ops {
		chosen = append(chosen, op)
		total += c.utxo[op].Out.Value
		if total >= amount+fee {
			break
		}
	}
	if total < amount+fee {
		return nil, fmt.Errorf("%s cannot cover %s + %s fee (has %s spendable)",
			w.Name, btc(amount), btc(fee), btc(total))
	}

	t := &Transaction{Outputs: []TxOutput{{Value: amount, PubKeyHash: to}}}
	if change := total - amount - fee; change > 0 {
		t.Outputs = append(t.Outputs, TxOutput{Value: change, PubKeyHash: w.PKH()})
	}
	for _, op := range chosen {
		t.Inputs = append(t.Inputs, TxInput{Prev: op})
	}
	for i := range t.Inputs {
		e := c.utxo[t.Inputs[i].Prev]
		sh := t.SigHash(i, e.Out.PubKeyHash)
		sig, err := crypto.Sign(sh[:], w.priv)
		if err != nil {
			return nil, err
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = crypto.CompressPubkey(&w.priv.PublicKey)
	}
	return t, nil
}

// --------------------------------------------------------------------- demo

func btc(sat int64) string {
	neg := ""
	if sat < 0 {
		neg, sat = "-", -sat
	}
	return fmt.Sprintf("%s%d.%08d", neg, sat/Coin, sat%Coin)
}

func short(h [32]byte) string { return hex.EncodeToString(h[:6]) }

func main() {
	// TEST ONLY — published Hardhat/anvil keys.
	miner := NewWallet("miner", "7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6")
	alice := NewWallet("alice", "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
	bob := NewWallet("bob", "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d")
	all := []*Wallet{miner, alice, bob}

	c := NewChain(miner.PKH())
	fmt.Printf("=== genesis ===\n  height 0, coinbase %s to miner, hash %s\n",
		btc(Subsidy(0)), short(c.Tip().Header.Hash()))

	fmt.Println("\n=== mining empty blocks so the coinbases mature ===")
	fmt.Printf("  (maturity is %d blocks here; Bitcoin uses 100)\n\n", CoinbaseMaturity)
	fmt.Printf("  %-8s %-14s %-12s %s\n", "height", "hash", "txs", "utxos")
	for i := 1; i <= 6; i++ {
		b := c.MineNext(nil, miner.PKH(), 60, 0)
		if err := c.Append(b); err != nil {
			fmt.Println("  append failed:", err)
			return
		}
		fmt.Printf("  %-8d %-14s %-12d %d\n", b.Header.Height, short(b.Header.Hash()), len(b.Txs), len(c.utxo))
	}

	fmt.Println("\n=== real payments ===")
	// The miner pays alice out of a matured coinbase.
	t1, err := miner.Pay(c, alice.PKH(), 20*Coin, Coin/10)
	if err != nil {
		fmt.Println(err)
		return
	}
	b7 := c.MineNext([]*Transaction{t1}, miner.PKH(), 60, 0)
	fmt.Printf("  height 7  miner -> alice %s, fee %s : %s\n", btc(20*Coin), btc(Coin/10), status(c.Append(b7)))

	// Alice pays bob; her coin is ordinary, so no maturity wait.
	t2, err := alice.Pay(c, bob.PKH(), 12*Coin, Coin/20)
	if err != nil {
		fmt.Println(err)
		return
	}
	b8 := c.MineNext([]*Transaction{t2}, miner.PKH(), 60, 0)
	fmt.Printf("  height 8  alice -> bob   %s, fee %s : %s\n", btc(12*Coin), btc(Coin/20), status(c.Append(b8)))

	// Two payments in one block, one of them spending the other's output.
	t3, _ := bob.Pay(c, alice.PKH(), 5*Coin, Coin/50)
	t4, _ := alice.Pay(c, miner.PKH(), 3*Coin, Coin/50)
	b9 := c.MineNext([]*Transaction{t3, t4}, miner.PKH(), 60, 0)
	fmt.Printf("  height 9  bob -> alice and alice -> miner : %s\n", status(c.Append(b9)))

	fmt.Println("\n=== balances ===")
	fmt.Printf("  %-8s %-16s %s\n", "who", "balance", "coins")
	var total int64
	for _, w := range all {
		bal := c.utxo.Balance(w.PKH())
		total += bal
		n := 0
		for _, e := range c.utxo {
			if bytes.Equal(e.Out.PubKeyHash, w.PKH()) {
				n++
			}
		}
		fmt.Printf("  %-8s %-16s %d\n", w.Name, btc(bal), n)
	}
	fmt.Printf("  %-8s %-16s %d entries in the UTXO set\n", "total", btc(total), len(c.utxo))
	var issued int64
	for h := int64(0); h <= int64(c.Tip().Header.Height); h++ {
		issued += Subsidy(h)
	}
	fmt.Printf("  issued by %d coinbases: %s — conservation holds: %v\n",
		c.Len(), btc(issued), issued == total)

	fmt.Println("\n=== every rejection the transaction model adds ===")
	reject := func(label string, mutate func(*Block)) {
		b := c.MineNext(nil, miner.PKH(), 60, 0)
		mutate(&b)
		b.Header.MerkleRoot = MerkleRoot(b.Txs)
		b.Header = Mine(b.Header) // re-mine, so PoW is never the reason
		fmt.Printf("  %-40s %s\n", label, status(c.Append(b)))
	}

	before := len(c.utxo.Spendable(miner.PKH(), int64(c.Tip().Header.Height)+1))

	reject("tampered signature", func(b *Block) {
		t, _ := miner.Pay(c, bob.PKH(), Coin, 0)
		t.Inputs[0].Signature[9] ^= 1
		b.Txs = append(b.Txs, t)
	})
	reject("double spend in one block", func(b *Block) {
		x, _ := miner.Pay(c, bob.PKH(), Coin, 0)
		y, _ := miner.Pay(c, alice.PKH(), Coin, 0)
		b.Txs = append(b.Txs, x, y)
	})
	reject("value created from nothing", func(b *Block) {
		t, _ := miner.Pay(c, bob.PKH(), Coin, 0)
		t.Outputs[0].Value = 999 * Coin
		resign(t, miner, c)
		b.Txs = append(b.Txs, t)
	})
	reject("coinbase claims one satoshi too much", func(b *Block) {
		b.Txs[0].Outputs[0].Value++
	})
	reject("immature coinbase spent", func(b *Block) {
		young := c.utxo[newestCoinbase(c, miner.PKH())]
		t := &Transaction{
			Inputs:  []TxInput{{Prev: newestCoinbase(c, miner.PKH())}},
			Outputs: []TxOutput{{Value: young.Out.Value, PubKeyHash: bob.PKH()}},
		}
		resign(t, miner, c)
		b.Txs = append(b.Txs, t)
	})
	reject("two coinbases", func(b *Block) {
		b.Txs = append(b.Txs, NewCoinbase(int64(b.Header.Height), bob.PKH(), Coin))
	})
	reject("spending an outpoint that does not exist", func(b *Block) {
		var ghost Outpoint
		ghost.TxID[0] = 0xff
		t := &Transaction{
			Inputs:  []TxInput{{Prev: ghost, Signature: make([]byte, 65), PubKey: make([]byte, 33)}},
			Outputs: []TxOutput{{Value: Coin, PubKeyHash: bob.PKH()}},
		}
		b.Txs = append(b.Txs, t)
	})

	after := len(c.utxo.Spendable(miner.PKH(), int64(c.Tip().Header.Height)+1))
	fmt.Printf("\n  chain length %d, %d UTXOs, miner's spendable coins %d -> %d\n",
		c.Len(), len(c.utxo), before, after)
	fmt.Println("  Every one of those rejections left the chain and the UTXO set")
	fmt.Println("  exactly as they were: connect() returns an error before Append")
	fmt.Println("  touches anything.")

	fmt.Println("\n=== reverting the tip, the way a reorg does ===")
	balBefore := c.utxo.Balance(alice.PKH())
	last := len(c.deltas) - 1
	c.utxo.Revert(c.deltas[last], c.undo[last])
	fmt.Printf("  alice: %s -> %s, utxos %d\n", btc(balBefore), btc(c.utxo.Balance(alice.PKH())), len(c.utxo))
	c.utxo.Apply(c.deltas[last])
	fmt.Printf("  re-applied: %s, utxos %d\n", btc(c.utxo.Balance(alice.PKH())), len(c.utxo))
	fmt.Println("  Lesson 14 turns this into choosing between two branches.")

	fmt.Println("\n--- the diff from lesson 09 ---")
	fmt.Println("  + Block.Txs, and a Merkle root over txids")
	fmt.Println("  + a coinbase, first and only")
	fmt.Println("  + a UTXO set applied as one delta per block, and revertible")
	fmt.Println("  + per-input signature verification against the spent output")
	fmt.Println("  + conservation, the fee, and the coinbase bound")
	fmt.Println("  + coinbase maturity")
	fmt.Println()
	fmt.Println("  Unchanged: the header, proof of work, retargeting, MTP, and")
	fmt.Println("  validate-then-mutate. connect() computes a delta and returns an")
	fmt.Println("  error; Append applies it only on success. Same rule as lesson 08.")

	fmt.Println("\n--- what lesson 11 changes ---")
	fmt.Println("  Right now the miner is handed exactly the transactions to")
	fmt.Println("  include. Lesson 11 puts a MEMPOOL in between: transactions")
	fmt.Println("  arrive, wait, get ordered by fee rate, and get evicted — and")
	fmt.Println("  the wallet has to guess what fee will actually get it mined.")
}

func status(err error) string {
	if err == nil {
		return "accepted"
	}
	return "REJECTED: " + err.Error()
}

func resign(t *Transaction, w *Wallet, c *Chain) {
	for i := range t.Inputs {
		e := c.utxo[t.Inputs[i].Prev]
		sh := t.SigHash(i, e.Out.PubKeyHash)
		sig, err := crypto.Sign(sh[:], w.priv)
		if err != nil {
			panic(err)
		}
		t.Inputs[i].Signature = sig
		t.Inputs[i].PubKey = crypto.CompressPubkey(&w.priv.PublicKey)
	}
}

func newestCoinbase(c *Chain, pkh []byte) Outpoint {
	var best Outpoint
	bestH := int64(-1)
	for op, e := range c.utxo {
		if e.Coinbase && bytes.Equal(e.Out.PubKeyHash, pkh) && e.Height > bestH {
			best, bestH = op, e.Height
		}
	}
	return best
}
```

**Output:**

```
=== genesis ===
  height 0, coinbase 50.00000000 to miner, hash 007c16ca5c31

=== mining empty blocks so the coinbases mature ===
  (maturity is 5 blocks here; Bitcoin uses 100)

  height   hash           txs          utxos
  1        0014716b045a   1            2
  2        00e29ba87715   1            3
  3        0049f529f57c   1            4
  4        004c08860ca0   1            5
  5        0025c863281a   1            6
  6        0092dbc66213   1            7

=== real payments ===
  height 7  miner -> alice 20.00000000, fee 0.10000000 : accepted
  height 8  alice -> bob   12.00000000, fee 0.05000000 : accepted
  height 9  bob -> alice and alice -> miner : accepted

=== balances ===
  who      balance          coins
  miner    483.09000000     11
  alice    9.93000000       2
  bob      6.98000000       1
  total    500.00000000     14 entries in the UTXO set
  issued by 10 coinbases: 500.00000000 — conservation holds: true

=== every rejection the transaction model adds ===
  tampered signature                       REJECTED: tx 1: signature does not verify
  double spend in one block                REJECTED: tx 2: outpoint spent twice in one block: also spent by tx 1
  value created from nothing               REJECTED: tx 1: outputs exceed inputs: in 50.00000000, out 1048.00000000
  coinbase claims one satoshi too much     REJECTED: coinbase claims more than subsidy+fees: claimed 50.00000001, allowed 50.00000000
  immature coinbase spent                  REJECTED: tx 1: coinbase output is not yet spendable: made at 9, spendable from 14
  two coinbases                            REJECTED: tx 1: more than one coinbase
  spending an outpoint that does not exist REJECTED: tx 1: input spends an output that is not in the UTXO set: ff0000000000…:0

  chain length 10, 14 UTXOs, miner's spendable coins 7 -> 7
  Every one of those rejections left the chain and the UTXO set
  exactly as they were: connect() returns an error before Append
  touches anything.

=== reverting the tip, the way a reorg does ===
  alice: 9.93000000 -> 7.95000000, utxos 11
  re-applied: 9.93000000, utxos 14
  Lesson 14 turns this into choosing between two branches.

--- the diff from lesson 09 ---
  + Block.Txs, and a Merkle root over txids
  + a coinbase, first and only
  + a UTXO set applied as one delta per block, and revertible
  + per-input signature verification against the spent output
  + conservation, the fee, and the coinbase bound
  + coinbase maturity

  Unchanged: the header, proof of work, retargeting, MTP, and
  validate-then-mutate. connect() computes a delta and returns an
  error; Append applies it only on success. Same rule as lesson 08.

--- what lesson 11 changes ---
  Right now the miner is handed exactly the transactions to
  include. Lesson 11 puts a MEMPOOL in between: transactions
  arrive, wait, get ordered by fee rate, and get evicted — and
  the wallet has to guess what fee will actually get it mined.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
