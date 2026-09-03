# Step 01 — Introduction to Blockchain · 🟡 Medium

Examples **6–9**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch          # first time only
# type the example into main.go, then:
go run .
```

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Order decides the outcome

`🟡 medium` · *Ordering*

Alice has 100 and two transfers of 60 are in flight. Both are individually valid; they cannot both apply. Which one wins depends entirely on the order, and the order is not in the data. This is the problem consensus exists to solve.

**Steps:**

1. Add one rule to `apply`: reject a transfer the sender cannot afford.
2. Run the same two operations in both possible orders.
3. Compare the final states — bob ends with 60 in one and 0 in the other.

```go
package main

import "fmt"

type Op struct {
	From, To string
	Amount   int64
}

// apply enforces one rule: you cannot send what you do not have.
func apply(state map[string]int64, op Op) bool {
	if state[op.From] < op.Amount {
		return false
	}
	state[op.From] -= op.Amount
	state[op.To] += op.Amount
	return true
}

func run(name string, ops []Op) map[string]int64 {
	state := map[string]int64{"alice": 100}
	fmt.Printf("%s:\n", name)
	for _, op := range ops {
		ok := apply(state, op)
		status := "REJECTED"
		if ok {
			status = "accepted"
		}
		fmt.Printf("  %-5s -> %-5s %3d  %s\n", op.From, op.To, op.Amount, status)
	}
	fmt.Printf("  final: alice=%d bob=%d carol=%d\n\n",
		state["alice"], state["bob"], state["carol"])
	return state
}

func main() {
	toBob := Op{"alice", "bob", 60}
	toCarol := Op{"alice", "carol", 60}

	a := run("order A: bob first", []Op{toBob, toCarol})
	b := run("order B: carol first", []Op{toCarol, toBob})

	fmt.Printf("same two operations, same rules, different outcome: bob %d vs %d\n",
		a["bob"], b["bob"])
	fmt.Println("agreeing on ORDER is the hard problem, not storing the data")
}
```

**Output:**

```
order A: bob first:
  alice -> bob    60  accepted
  alice -> carol  60  REJECTED
  final: alice=40 bob=60 carol=0

order B: carol first:
  alice -> carol  60  accepted
  alice -> bob    60  REJECTED
  final: alice=40 bob=0 carol=60

same two operations, same rules, different outcome: bob 60 vs 0
agreeing on ORDER is the hard problem, not storing the data
```

---

## 7. Submit-time checks are advice, apply-time checks are rules

`🟡 medium` · *Validity*

It is tempting to validate a transaction when it arrives. But the state it was checked against is already stale by the time it applies. Real systems check at submission as a *courtesy* (that is mempool policy, lesson 11) and enforce at apply time.

**Steps:**

1. Admit both transfers by checking each against the same 100-balance snapshot.
2. Apply the admitted set and watch alice go to -20.
3. Re-run with the check moved inside the apply loop; the second transfer is now rejected.

```go
package main

import "fmt"

type Op struct {
	ID     string
	From   string
	Amount int64
}

func main() {
	// alice has 100. Two operations spending 60 each arrive at the same moment.
	ops := []Op{{"tx1", "alice", 60}, {"tx2", "alice", 60}}

	// --- Submit-time checking: validate each op against the balance as it is NOW.
	fmt.Println("checked at SUBMIT time (against a snapshot):")
	snapshot := map[string]int64{"alice": 100}
	var accepted []Op
	for _, op := range ops {
		if snapshot[op.From] >= op.Amount {
			fmt.Printf("  %s ok (snapshot says alice=%d)\n", op.ID, snapshot[op.From])
			accepted = append(accepted, op)
		}
	}
	state := map[string]int64{"alice": 100}
	for _, op := range accepted {
		state[op.From] -= op.Amount
	}
	fmt.Printf("  both admitted, then applied: alice=%d  <- overdrawn\n\n", state["alice"])

	// --- Apply-time checking: validate against the state the op will actually mutate.
	fmt.Println("checked at APPLY time (against live state):")
	state = map[string]int64{"alice": 100}
	for _, op := range ops {
		if state[op.From] < op.Amount {
			fmt.Printf("  %s REJECTED (alice=%d, needs %d)\n", op.ID, state[op.From], op.Amount)
			continue
		}
		state[op.From] -= op.Amount
		fmt.Printf("  %s applied (alice=%d)\n", op.ID, state[op.From])
	}
	fmt.Printf("  final: alice=%d\n\n", state["alice"])

	fmt.Println("a snapshot check is advice; only the check at apply time is a rule")
}
```

**Output:**

```
checked at SUBMIT time (against a snapshot):
  tx1 ok (snapshot says alice=100)
  tx2 ok (snapshot says alice=100)
  both admitted, then applied: alice=-20  <- overdrawn

checked at APPLY time (against live state):
  tx1 applied (alice=40)
  tx2 REJECTED (alice=40, needs 60)
  final: alice=40

a snapshot check is advice; only the check at apply time is a rule
```

---

## 8. Determinism: map order breaks agreement

`🟡 medium` · *Determinism*

Nodes must reach byte-identical state, so every input to a state transition must be deterministic. Go map iteration is deliberately randomised, which makes it a silent consensus bug. The same applies to `time.Now`, `math/rand`, and floating point.

**Steps:**

1. Fold eight balances into a digest by walking the map directly, five times.
2. Print whether all five digests matched — they will not.
3. Sort the keys first and repeat; now the digest is stable across runs and machines.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"slices"
)

// digest folds a set of balances into one value — a toy "state root".
// How you walk the accounts decides what you get.
func digest(order []string, balances map[string]int64) string {
	h := sha256.New()
	for _, k := range order {
		fmt.Fprintf(h, "%s=%d;", k, balances[k])
	}
	return hex.EncodeToString(h.Sum(nil))[:16]
}

func main() {
	balances := map[string]int64{
		"alice": 10, "bob": 20, "carol": 30, "dave": 40,
		"erin": 50, "frank": 60, "grace": 70, "heidi": 80,
	}

	// Walking a Go map yields a different order almost every time.
	seen := map[string]bool{}
	for i := 0; i < 5; i++ {
		var order []string
		for k := range balances {
			order = append(order, k)
		}
		seen[digest(order, balances)] = true
	}
	fmt.Printf("map iteration, 5 runs — all digests identical? %v\n", len(seen) == 1)

	// Sorting first removes the only source of nondeterminism here.
	sortedSeen := map[string]bool{}
	var stable string
	for i := 0; i < 5; i++ {
		var order []string
		for k := range balances {
			order = append(order, k)
		}
		slices.Sort(order)
		stable = digest(order, balances)
		sortedSeen[stable] = true
	}
	fmt.Printf("sorted order,  5 runs — all digests identical? %v\n", len(sortedSeen) == 1)
	fmt.Printf("\nstable digest: %s\n", stable)

	fmt.Println("\nnodes must agree byte-for-byte, so map order, time.Now and")
	fmt.Println("floating point are all banned from consensus code")
}
```

**Output:**

```
map iteration, 5 runs — all digests identical? false
sorted order,  5 runs — all digests identical? true

stable digest: f007d7fb59d6be43

nodes must agree byte-for-byte, so map order, time.Now and
floating point are all banned from consensus code
```

---

## 9. The cost of agreement

`🟡 medium` · *The cost of agreement*

Agreement is not free. If every node must hear from every other node, message count grows with the square of the network. This single table explains why classic BFT (lesson 29) caps out in the hundreds and why Bitcoin replaced voting with a lottery.

**Steps:**

1. Compute n·(n−1) messages for a single broadcast round.
2. Double it for a two-round prepare/commit vote, as in PBFT.
3. Read the table: 10,000 nodes costs ~200 million messages per decision.

```go
package main

import "fmt"

func main() {
	fmt.Println("every node must hear every proposal, and hear what everyone else")
	fmt.Println("thinks about it — so naive agreement costs O(n^2) messages")
	fmt.Println()
	fmt.Printf("%8s %14s %16s\n", "nodes", "broadcast", "two-round vote")
	fmt.Printf("%8s %14s %16s\n", "-----", "---------", "--------------")

	for _, n := range []int{4, 10, 21, 100, 1000, 10000} {
		broadcast := n * (n - 1)  // everyone tells everyone once
		twoRound := 2 * broadcast // prepare + commit, as in classic BFT
		fmt.Printf("%8d %14d %16d\n", n, broadcast, twoRound)
	}

	fmt.Println()
	fmt.Println("this is why classic BFT stops scaling around a few hundred nodes,")
	fmt.Println("and why Bitcoin picked a lottery instead of a vote")
}
```

**Output:**

```
every node must hear every proposal, and hear what everyone else
thinks about it — so naive agreement costs O(n^2) messages

   nodes      broadcast   two-round vote
   -----      ---------   --------------
       4             12               24
      10             90              180
      21            420              840
     100           9900            19800
    1000         999000          1998000
   10000       99990000        199980000

this is why classic BFT stops scaling around a few hundred nodes,
and why Bitcoin picked a lottery instead of a vote
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
