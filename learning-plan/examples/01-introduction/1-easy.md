# Step 01 — Introduction to Blockchain · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch          # first time only
# type the example into main.go, then:
go run .
```

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. The ledger is the log

`🟢 easy` · *The ledger model*

A ledger is an ordered list of events plus a rule for applying them. Balances are not stored anywhere — they are what you get by folding the list from the beginning. Everything in this course sits on top of that one idea.

**Steps:**

1. Define a `Transfer` and an `apply` function — together they are the state machine.
2. Fold a three-event log into a balance map, printing each step.
3. Note that nothing is stored but the log; the balances are computed.

```go
package main

import "fmt"

// A transfer is the only kind of event in this ledger.
type Transfer struct {
	From, To string
	Amount   int64
}

// apply mutates state by one event. This is the whole "state machine".
func apply(balances map[string]int64, t Transfer) {
	balances[t.From] -= t.Amount
	balances[t.To] += t.Amount
}

func main() {
	// The log IS the ledger. Ordered, append-only.
	log := []Transfer{
		{"mint", "alice", 100},
		{"alice", "bob", 30},
		{"bob", "carol", 10},
	}

	balances := map[string]int64{}
	for i, t := range log {
		apply(balances, t)
		fmt.Printf("applied #%d: %-5s -> %-5s %3d\n", i, t.From, t.To, t.Amount)
	}

	fmt.Println("\nfinal balances (derived by folding the log):")
	for _, who := range []string{"alice", "bob", "carol"} {
		fmt.Printf("  %-6s %4d\n", who, balances[who])
	}
}
```

**Output:**

```
applied #0: mint  -> alice 100
applied #1: alice -> bob    30
applied #2: bob   -> carol  10

final balances (derived by folding the log):
  alice    70
  bob      20
  carol    10
```

---

## 2. Balances are derived, not stored

`🟢 easy` · *The ledger model*

If the log is the source of truth, then any historical balance is just a shorter fold. This is why a chain can answer "what did alice own at block 12?" without a snapshot, and why an indexer (lesson 31) can rebuild its whole database from the chain.

**Steps:**

1. Write `stateAt(log, n)` that folds only the first `n` events.
2. Print the balance table at every height from 0 to 4.
3. Watch the state at height 4 differ from the state at height 2 — same log, different prefix.

```go
package main

import "fmt"

type Transfer struct {
	From, To string
	Amount   int64
}

// stateAt recomputes balances from scratch by folding the first n events.
// Nothing is stored but the log; every balance is derived.
func stateAt(log []Transfer, n int) map[string]int64 {
	balances := map[string]int64{}
	for _, t := range log[:n] {
		balances[t.From] -= t.Amount
		balances[t.To] += t.Amount
	}
	return balances
}

func main() {
	log := []Transfer{
		{"mint", "alice", 100},
		{"alice", "bob", 30},
		{"bob", "carol", 10},
		{"alice", "carol", 25},
	}

	fmt.Printf("%-8s %6s %6s %6s\n", "height", "alice", "bob", "carol")
	for n := 0; n <= len(log); n++ {
		b := stateAt(log, n)
		fmt.Printf("%-8d %6d %6d %6d\n", n, b["alice"], b["bob"], b["carol"])
	}

	fmt.Println("\nany historical balance is a replay away — no snapshot needed")
}
```

**Output:**

```
height    alice    bob  carol
0             0      0      0
1           100      0      0
2            70     30      0
3            70     20     10
4            45     20     35

any historical balance is a replay away — no snapshot needed
```

---

## 3. The double-spend

`🟢 easy` · *The core problem*

A digital coin is a number, and numbers copy for free. Here the exact same transfer is applied twice and the ledger cheerfully lets alice spend money she does not have. Nothing about storage or hashing prevents this; only a rule can.

**Steps:**

1. Give alice 100 and build a transfer spending all of it.
2. Apply that identical transfer twice through a ledger with no rules.
3. Read the final balance: alice is at -100, having spent 200.

```go
package main

import "fmt"

type Transfer struct {
	From, To string
	Amount   int64
}

func main() {
	// alice starts with 100 and spends all of it.
	spend := Transfer{"alice", "bob", 100}

	balances := map[string]int64{"alice": 100}

	// A naive ledger just applies whatever it is handed.
	naive := func(t Transfer) {
		balances[t.From] -= t.Amount
		balances[t.To] += t.Amount
	}

	naive(spend)
	fmt.Printf("after 1st apply: alice=%d bob=%d\n", balances["alice"], balances["bob"])

	// The exact same bytes, submitted again. Nothing here says "already spent".
	naive(spend)
	fmt.Printf("after 2nd apply: alice=%d bob=%d\n", balances["alice"], balances["bob"])

	fmt.Println()
	fmt.Println("alice spent 200 from a balance of 100 — a double-spend")
	fmt.Println("copying a digital coin is free; only a rule can stop it")
}
```

**Output:**

```
after 1st apply: alice=0 bob=100
after 2nd apply: alice=-100 bob=200

alice spent 200 from a balance of 100 — a double-spend
copying a digital coin is free; only a rule can stop it
```

---

## 4. Hash-linking records

`🟢 easy` · *Integrity*

Hash linking is the first of the three ingredients. Each record's hash covers both its own data and the previous record's hash, so the records form a chain that you cannot edit quietly. This is the *chain* half of the word.

**Steps:**

1. Define a `Record` holding data plus the previous record's hash.
2. Hash `prev ‖ data` so each hash commits to everything before it.
3. Build three records and print how each `prev` matches the one above it.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

type Record struct {
	Data string
	Prev [32]byte
}

// hash commits to both this record's data AND its predecessor's hash.
func (r Record) hash() [32]byte {
	h := sha256.New()
	h.Write(r.Prev[:])
	h.Write([]byte(r.Data))
	var out [32]byte
	copy(out[:], h.Sum(nil))
	return out
}

func short(b [32]byte) string { return hex.EncodeToString(b[:])[:12] }

func main() {
	var zero [32]byte

	chain := []Record{{Data: "alice -> bob 30", Prev: zero}}
	chain = append(chain, Record{Data: "bob -> carol 10", Prev: chain[0].hash()})
	chain = append(chain, Record{Data: "carol -> dave 5", Prev: chain[1].hash()})

	for i, r := range chain {
		fmt.Printf("record %d  prev=%s  hash=%s  %q\n", i, short(r.Prev), short(r.hash()), r.Data)
	}

	fmt.Println("\neach hash covers the one before it, so the records form a chain")
}
```

**Output:**

```
record 0  prev=000000000000  hash=d31f005940bc  "alice -> bob 30"
record 1  prev=d31f005940bc  hash=428e823f7923  "bob -> carol 10"
record 2  prev=428e823f7923  hash=ccac21118c61  "carol -> dave 5"

each hash covers the one before it, so the records form a chain
```

---

## 5. Tampering breaks every record after it

`🟢 easy` · *Integrity*

The point of hash linking is not that data cannot be changed — it is that changes cannot be hidden. Editing record 0 leaves record 1 pointing at a hash that no longer exists, and verification fails immediately.

**Steps:**

1. Build the same three-record chain and verify it walks cleanly.
2. Rewrite record 0's data from 30 to 300.
3. Re-verify: the break is reported at index 1, and the printed hashes show why.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

type Record struct {
	Data string
	Prev [32]byte
}

func (r Record) hash() [32]byte {
	h := sha256.New()
	h.Write(r.Prev[:])
	h.Write([]byte(r.Data))
	var out [32]byte
	copy(out[:], h.Sum(nil))
	return out
}

func short(b [32]byte) string { return hex.EncodeToString(b[:])[:12] }

// verify walks forward, checking every record points at its real predecessor.
func verify(chain []Record) (int, bool) {
	for i := 1; i < len(chain); i++ {
		if chain[i].Prev != chain[i-1].hash() {
			return i, false
		}
	}
	return -1, true
}

func main() {
	var zero [32]byte
	chain := []Record{{Data: "alice -> bob 30", Prev: zero}}
	chain = append(chain, Record{Data: "bob -> carol 10", Prev: chain[0].hash()})
	chain = append(chain, Record{Data: "carol -> dave 5", Prev: chain[1].hash()})

	i, ok := verify(chain)
	fmt.Printf("original chain valid: %v (broken at %d)\n", ok, i)

	// Rewrite history: make the first transfer bigger.
	fmt.Println("\ntampering with record 0: 30 -> 300")
	chain[0].Data = "alice -> bob 300"

	i, ok = verify(chain)
	fmt.Printf("tampered chain valid: %v (broken at %d)\n", ok, i)
	fmt.Printf("  record 1 stores prev=%s\n", short(chain[1].Prev))
	fmt.Printf("  record 0 now hashes to %s\n", short(chain[0].hash()))

	fmt.Println("\nediting one record invalidates every record after it")
}
```

**Output:**

```
original chain valid: true (broken at -1)

tampering with record 0: 30 -> 300
tampered chain valid: false (broken at 1)
  record 1 stores prev=d31f005940bc
  record 0 now hashes to f94797232b7a

editing one record invalidates every record after it
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
