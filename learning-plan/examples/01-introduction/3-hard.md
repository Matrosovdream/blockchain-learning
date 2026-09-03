# Step 01 — Introduction to Blockchain · 🔴 Hard

Examples **10–12**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch          # first time only
# type the example into main.go, then:
go run .
```

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 10. Trusted server vs replicas under a partition

`🔴 hard` · *Trust models*

Two designs for the same ledger, in one program. A trusted server solves double-spending trivially by deciding the order — and that is exactly its weakness. Replicas with no agreement rule stay available under a partition, and pay for it with two irreconcilable histories.

**Steps:**

1. Run both spends through a single-server ledger; the second is rejected.
2. Split five replicas 2|3 and apply a different spend on each side.
3. Print every node's state after the heal: locally consistent, globally contradictory.

```go
package main

import "fmt"

type Tx struct {
	ID, From, To string
	Amount       int64
}

// A ledger applies a transaction only if the sender can afford it.
type Ledger struct {
	Name     string
	balances map[string]int64
}

func newLedger(name string) *Ledger {
	return &Ledger{Name: name, balances: map[string]int64{"alice": 100}}
}

func (l *Ledger) apply(tx Tx) bool {
	if l.balances[tx.From] < tx.Amount {
		return false
	}
	l.balances[tx.From] -= tx.Amount
	l.balances[tx.To] += tx.Amount
	return true
}

func (l *Ledger) String() string {
	return fmt.Sprintf("alice=%d bob=%d carol=%d",
		l.balances["alice"], l.balances["bob"], l.balances["carol"])
}

func main() {
	toBob := Tx{"tx1", "alice", "bob", 100}
	toCarol := Tx{"tx2", "alice", "carol", 100}

	// ---------- Design 1: one trusted server ----------
	fmt.Println("DESIGN 1 — one trusted server")
	server := newLedger("server")
	for _, tx := range []Tx{toBob, toCarol} {
		fmt.Printf("  %s: %v\n", tx.ID, server.apply(tx))
	}
	fmt.Printf("  state: %s\n", server)
	fmt.Println("  the second spend is impossible: one machine decided the order")
	fmt.Println("  cost: that machine can censor, be seized, or simply go down")

	// ---------- Design 2: replicas, no agreement rule ----------
	fmt.Println("\nDESIGN 2 — five replicas, network splits 2 | 3")
	sideA := []*Ledger{newLedger("n1"), newLedger("n2")}
	sideB := []*Ledger{newLedger("n3"), newLedger("n4"), newLedger("n5")}

	// Alice sends the same coins to different people on each side of the split.
	for _, l := range sideA {
		l.apply(toBob)
	}
	for _, l := range sideB {
		l.apply(toCarol)
	}

	fmt.Println("  during the partition, every node is internally consistent:")
	for _, l := range append(append([]*Ledger{}, sideA...), sideB...) {
		fmt.Printf("    %s: %s\n", l.Name, l)
	}

	fmt.Println("\n  network heals. Both histories are valid. Which one is real?")
	fmt.Printf("    side A says bob has 100, side B says carol has 100\n")
	fmt.Println("    nothing in the data answers this — you need a rule")
	fmt.Println("\nthat rule is consensus, and it is the only part that was new in 2009")
}
```

**Output:**

```
DESIGN 1 — one trusted server
  tx1: true
  tx2: false
  state: alice=0 bob=100 carol=0
  the second spend is impossible: one machine decided the order
  cost: that machine can censor, be seized, or simply go down

DESIGN 2 — five replicas, network splits 2 | 3
  during the partition, every node is internally consistent:
    n1: alice=0 bob=100 carol=0
    n2: alice=0 bob=100 carol=0
    n3: alice=0 bob=0 carol=100
    n4: alice=0 bob=0 carol=100
    n5: alice=0 bob=0 carol=100

  network heals. Both histories are valid. Which one is real?
    side A says bob has 100, side B says carol has 100
    nothing in the data answers this — you need a rule

that rule is consensus, and it is the only part that was new in 2009
```

---

## 11. Five nodes, two spends, no agreement

`🔴 hard` · *Divergence*

Every node here receives *both* transactions — nothing is lost, nothing is censored. Only the arrival order differs, because propagation takes time. First-come-first-served produces two permanent camps, and no amount of extra gossip repairs it.

**Steps:**

1. Give five nodes the same two conflicting transfers in different arrival orders.
2. Apply first-come-first-served with the affordability rule.
3. Count the distinct final states: two. The network has forked.

```go
package main

import "fmt"

type Tx struct {
	ID, From, To string
	Amount       int64
}

type Node struct {
	Name     string
	balances map[string]int64
}

func newNode(name string) *Node {
	return &Node{Name: name, balances: map[string]int64{"alice": 100}}
}

// First come, first served: whatever reaches this node first, wins.
func (n *Node) apply(tx Tx) bool {
	if n.balances[tx.From] < tx.Amount {
		return false
	}
	n.balances[tx.From] -= tx.Amount
	n.balances[tx.To] += tx.Amount
	return true
}

func (n *Node) state() string {
	return fmt.Sprintf("alice=%3d bob=%3d carol=%3d",
		n.balances["alice"], n.balances["bob"], n.balances["carol"])
}

func main() {
	toBob := Tx{"tx1", "alice", "bob", 100}
	toCarol := Tx{"tx2", "alice", "carol", 100}

	// Propagation delay decides arrival order, and it differs per node.
	arrivals := map[string][]Tx{
		"n1": {toBob, toCarol},
		"n2": {toBob, toCarol},
		"n3": {toCarol, toBob},
		"n4": {toCarol, toBob},
		"n5": {toBob, toCarol},
	}

	names := []string{"n1", "n2", "n3", "n4", "n5"}
	states := map[string]string{}

	fmt.Println("every node sees BOTH transactions — only the order differs")
	fmt.Println()
	for _, name := range names {
		n := newNode(name)
		var seen []string
		for _, tx := range arrivals[name] {
			ok := n.apply(tx)
			mark := "x"
			if ok {
				mark = "+"
			}
			seen = append(seen, mark+tx.ID)
		}
		states[name] = n.state()
		fmt.Printf("  %s  saw %v  ->  %s\n", name, seen, states[name])
	}

	distinct := map[string]int{}
	for _, s := range states {
		distinct[s]++
	}
	fmt.Printf("\ndistinct final states across 5 nodes: %d\n", len(distinct))
	fmt.Println("the network has permanently disagreed about who owns the money")
	fmt.Println("no amount of extra gossip fixes this — the inputs were identical")
}
```

**Output:**

```
every node sees BOTH transactions — only the order differs

  n1  saw [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n2  saw [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n3  saw [+tx2 xtx1]  ->  alice=  0 bob=  0 carol=100
  n4  saw [+tx2 xtx1]  ->  alice=  0 bob=  0 carol=100
  n5  saw [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0

distinct final states across 5 nodes: 2
the network has permanently disagreed about who owns the money
no amount of extra gossip fixes this — the inputs were identical
```

---

## 12. One shared ordering rule, five identical states

`🔴 hard` · *Consensus*

The fix is not more messages — it is a shared, deterministic ordering rule. Here nodes collect first and sort by transaction id before applying, so all five compute the same order from the same set. A real chain uses a rule an attacker cannot cheaply grind; that is what proof of work and proof of stake actually buy.

**Steps:**

1. Give each transaction a `txid` — a hash of its contents, identical on every node.
2. Collect into a pending set instead of applying on arrival, then sort by txid and commit.
3. Re-run the exact arrival orders from example 11 — all five states now match.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"slices"
)

type Tx struct {
	ID, From, To string
	Amount       int64
}

// txid is a deterministic name for a transaction: same bytes, same id, everywhere.
func (t Tx) txid() string {
	sum := sha256.Sum256([]byte(fmt.Sprintf("%s|%s|%s|%d", t.ID, t.From, t.To, t.Amount)))
	return hex.EncodeToString(sum[:])[:12]
}

type Node struct {
	Name     string
	pending  []Tx
	balances map[string]int64
}

func newNode(name string) *Node {
	return &Node{Name: name, balances: map[string]int64{"alice": 100}}
}

func (n *Node) receive(tx Tx) { n.pending = append(n.pending, tx) }

// The rule: do not apply on arrival. Collect, then order by txid, then apply.
// Every honest node runs this same function over the same set.
func (n *Node) commit() []string {
	slices.SortFunc(n.pending, func(a, b Tx) int {
		return cmpString(a.txid(), b.txid())
	})
	var outcome []string
	for _, tx := range n.pending {
		if n.balances[tx.From] < tx.Amount {
			outcome = append(outcome, "x"+tx.ID)
			continue
		}
		n.balances[tx.From] -= tx.Amount
		n.balances[tx.To] += tx.Amount
		outcome = append(outcome, "+"+tx.ID)
	}
	return outcome
}

func cmpString(a, b string) int {
	switch {
	case a < b:
		return -1
	case a > b:
		return 1
	}
	return 0
}

func (n *Node) state() string {
	return fmt.Sprintf("alice=%3d bob=%3d carol=%3d",
		n.balances["alice"], n.balances["bob"], n.balances["carol"])
}

func main() {
	toBob := Tx{"tx1", "alice", "bob", 100}
	toCarol := Tx{"tx2", "alice", "carol", 100}

	fmt.Printf("txid(tx1) = %s\ntxid(tx2) = %s\n\n", toBob.txid(), toCarol.txid())

	arrivals := map[string][]Tx{
		"n1": {toBob, toCarol},
		"n2": {toBob, toCarol},
		"n3": {toCarol, toBob},
		"n4": {toCarol, toBob},
		"n5": {toBob, toCarol},
	}

	names := []string{"n1", "n2", "n3", "n4", "n5"}
	states := map[string]int{}

	fmt.Println("same arrival orders as before, but ordering is now a shared rule")
	fmt.Println()
	for _, name := range names {
		n := newNode(name)
		for _, tx := range arrivals[name] {
			n.receive(tx)
		}
		outcome := n.commit()
		states[n.state()]++
		fmt.Printf("  %s  committed %v  ->  %s\n", name, outcome, n.state())
	}

	fmt.Printf("\ndistinct final states across 5 nodes: %d\n", len(states))
	fmt.Println("everyone agrees, and nobody was in charge")
	fmt.Println()
	fmt.Println("a real chain replaces 'sort by txid' with a rule an attacker cannot")
	fmt.Println("cheaply game — that is what proof of work and proof of stake buy")
}
```

**Output:**

```
txid(tx1) = 493d524eb34f
txid(tx2) = eb801e4b126b

same arrival orders as before, but ordering is now a shared rule

  n1  committed [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n2  committed [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n3  committed [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n4  committed [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0
  n5  committed [+tx1 xtx2]  ->  alice=  0 bob=100 carol=  0

distinct final states across 5 nodes: 1
everyone agrees, and nobody was in charge

a real chain replaces 'sort by txid' with a rule an attacker cannot
cheaply game — that is what proof of work and proof of stake buy
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
