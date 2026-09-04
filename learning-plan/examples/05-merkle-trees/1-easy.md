# Step 05 — Merkle Trees & Proofs · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get github.com/ethereum/go-ethereum@latest   # examples 13, 14
# paste the example into main.go, then:
go run .
```

No chain, no node, no keys. Everything except examples 13 and 14 needs only the standard library.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. Building a tree, level by level

`🟢 easy` · *Construction*

Hash every leaf, pair them, hash each pair, repeat until one node is left. That is the whole algorithm. The leaf and node hashers are exactly the ones from lesson 04's example 16 — and the root below is the same one that example printed.

**Steps:**

1. Hash the four items into level 0.
2. Pair adjacent hashes and hash each pair to get level 1.
3. Repeat until a single root remains, printing every level.
4. Note four leaves collapse into one 32-byte commitment two levels up.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// Domain tags, exactly as in lesson 04's example 16. A leaf can never be
// mistaken for an internal node, and vice versa (topic 5).
const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func leafHash(data []byte) [32]byte {
	return sha256.Sum256(append([]byte{tagLeaf}, data...))
}

func nodeHash(l, r [32]byte) [32]byte {
	buf := make([]byte, 0, 65)
	buf = append(buf, tagNode)
	buf = append(buf, l[:]...)
	buf = append(buf, r[:]...)
	return sha256.Sum256(buf)
}

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	items := [][]byte{[]byte("tx-a"), []byte("tx-b"), []byte("tx-c"), []byte("tx-d")}

	// Level 0: hash every leaf.
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = leafHash(it)
	}
	fmt.Printf("level 0 — %d leaves\n", len(level))
	for i, h := range level {
		fmt.Printf("  [%d] %s  %q\n", i, short(h), items[i])
	}

	// Then pair and hash upward until one node is left.
	depth := 1
	for len(level) > 1 {
		next := make([][32]byte, 0, (len(level)+1)/2)
		for i := 0; i < len(level); i += 2 {
			next = append(next, nodeHash(level[i], level[i+1]))
		}
		level = next
		fmt.Printf("level %d — %d node(s)\n", depth, len(level))
		for i, h := range level {
			fmt.Printf("  [%d] %s\n", i, short(h))
		}
		depth++
	}

	fmt.Printf("\nroot %s\n", hex.EncodeToString(level[0][:]))
	fmt.Printf("\n4 leaves collapsed into one 32-byte commitment, %d levels deep\n", depth-1)
}
```

**Output:**

```
level 0 — 4 leaves
  [0] 24536162901e5df6  "tx-a"
  [1] efc4942ae88b4394  "tx-b"
  [2] a60238363fd02697  "tx-c"
  [3] 7473871f4cd69271  "tx-d"
level 1 — 2 node(s)
  [0] f847efba820f4560
  [1] c72cc6b52d9fc77b
level 2 — 1 node(s)
  [0] 6e5818a6c1d68c5a

root 6e5818a6c1d68c5aa635dcb16b6aa1166c374ed812682c807d3cf0f7c4b854df

4 leaves collapsed into one 32-byte commitment, 2 levels deep
```

---

## 2. The root commits to content and order

`🟢 easy` · *Construction*

The root is a commitment to every leaf *and* to their order. Change one byte anywhere and it moves; swap two leaves and it moves. That is what lets a block header pin down a whole block body in 32 bytes.

**Steps:**

1. Compute a root over four balances.
2. Edit one leaf and compare.
3. Swap two leaves — same content, different order — and compare again.
4. Note what the root does *not* commit to: leaf count, uniqueness, or absence.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func leafHash(d []byte) [32]byte { return sha256.Sum256(append([]byte{tagLeaf}, d...)) }

func nodeHash(l, r [32]byte) [32]byte {
	buf := append([]byte{tagNode}, l[:]...)
	return sha256.Sum256(append(buf, r[:]...))
}

// Root over a power-of-two number of leaves. Odd counts are topic 4.
func Root(items [][]byte) [32]byte {
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = leafHash(it)
	}
	for len(level) > 1 {
		next := make([][32]byte, 0, len(level)/2)
		for i := 0; i < len(level); i += 2 {
			next = append(next, nodeHash(level[i], level[i+1]))
		}
		level = next
	}
	return level[0]
}

func main() {
	base := [][]byte{[]byte("alice:100"), []byte("bob:50"), []byte("carol:25"), []byte("dave:10")}
	root := Root(base)
	fmt.Printf("root                %s\n", hex.EncodeToString(root[:]))

	// One character changed in one leaf.
	edited := [][]byte{[]byte("alice:100"), []byte("bob:500"), []byte("carol:25"), []byte("dave:10")}
	editedRoot := Root(edited)
	fmt.Printf("one leaf edited     %s\n", hex.EncodeToString(editedRoot[:]))

	// The same leaves in a different order.
	swapped := [][]byte{[]byte("bob:50"), []byte("alice:100"), []byte("carol:25"), []byte("dave:10")}
	swappedRoot := Root(swapped)
	fmt.Printf("two leaves swapped  %s\n", hex.EncodeToString(swappedRoot[:]))

	fmt.Printf("\nedit changes the root:  %v\n", root != editedRoot)
	fmt.Printf("swap changes the root:  %v\n", root != swappedRoot)

	fmt.Println("\nthe root commits to the CONTENT and the ORDER of every leaf.")
	fmt.Println("that is why a block header needs only 32 bytes to pin down its whole body.")

	// It does not, however, commit to anything you did not put in a leaf.
	fmt.Println("\nwhat it does NOT commit to: how many leaves there are (unless you")
	fmt.Println("include the count), whether they are unique, or anything about absence.")
}
```

**Output:**

```
root                5a28a9389f1ec49170f4fc1e767960f6aa77936c156ae319199691c12166cb8d
one leaf edited     0f5c17b9e558d024dd68ea0d98f0ab261f9724090448b01b642ff5b4191aa161
two leaves swapped  5ce934e0b1486a7fd36d3537a0878d3ac93d4dddf5544cd0f53a0bcdec2545b0

edit changes the root:  true
swap changes the root:  true

the root commits to the CONTENT and the ORDER of every leaf.
that is why a block header needs only 32 bytes to pin down its whole body.

what it does NOT commit to: how many leaves there are (unless you
include the count), whether they are unique, or anything about absence.
```

---

## 3. A two-leaf proof, folded by hand

`🟢 easy` · *Proofs*

The smallest interesting tree. To verify that `tx-a` is in it you need the leaf, the sibling's *hash* (not its contents), which side the sibling is on, and the root. Fold once and compare.

**Steps:**

1. Hash two leaves and combine them into a root.
2. Verify `tx-a` using only the sibling hash and the root.
3. Fold in the wrong order and watch it fail — direction is part of the proof.
4. Try a leaf that is not in the tree.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func leafHash(d []byte) [32]byte { return sha256.Sum256(append([]byte{tagLeaf}, d...)) }

func nodeHash(l, r [32]byte) [32]byte {
	buf := append([]byte{tagNode}, l[:]...)
	return sha256.Sum256(append(buf, r[:]...))
}

func main() {
	// The smallest interesting tree: two leaves, one root.
	//
	//        root
	//        /  \
	//     hA      hB
	left := []byte("tx-a")
	right := []byte("tx-b")

	hA := leafHash(left)
	hB := leafHash(right)
	root := nodeHash(hA, hB)

	fmt.Printf("leaf A  %s\n", hex.EncodeToString(hA[:]))
	fmt.Printf("leaf B  %s\n", hex.EncodeToString(hB[:]))
	fmt.Printf("root    %s\n", hex.EncodeToString(root[:]))

	// A verifier who wants to check that "tx-a" is in the tree needs:
	//   - the leaf itself
	//   - hB (the sibling)
	//   - the knowledge that hB sits on the RIGHT
	//   - the root
	// It does not need "tx-b" — only its hash.
	fmt.Println("\nverifying \"tx-a\" with only the sibling HASH and the root:")

	computed := leafHash(left)
	fmt.Printf("  start with leaf(\"tx-a\")   %s\n", hex.EncodeToString(computed[:8]))
	computed = nodeHash(computed, hB) // sibling on the right
	fmt.Printf("  fold with sibling (right) %s\n", hex.EncodeToString(computed[:8]))
	fmt.Printf("  equals the root?          %v\n", computed == root)

	// Fold in the wrong order and it fails. Direction is part of the proof.
	wrong := nodeHash(hB, leafHash(left))
	fmt.Printf("\n  folded in the wrong order: %v\n", wrong == root)

	// And a leaf that is not in the tree cannot be made to fit.
	notThere := nodeHash(leafHash([]byte("tx-z")), hB)
	fmt.Printf("  a leaf that is not in the tree: %v\n", notThere == root)
}
```

**Output:**

```
leaf A  24536162901e5df6265aae57bd3f6225c9ee58170c7cb4d81787f18121e4a0fe
leaf B  efc4942ae88b4394c1bad4283bb197dd8e503663da80c5c61bd91115643b1fb7
root    f847efba820f456048ec00c49c6bb214b15d78520d25c660c20d53ae9c3f7192

verifying "tx-a" with only the sibling HASH and the root:
  start with leaf("tx-a")   24536162901e5df6
  fold with sibling (right) f847efba820f4560
  equals the root?          true

  folded in the wrong order: false
  a leaf that is not in the tree: false
```

---

## 4. Why not just hash everything together?

`🟢 easy` · *The problem*

Hashing the concatenation of every item is also a commitment, so why build a tree? Because to convince someone one item is in the set, they have to recompute the hash — which means you must send them everything.

**Steps:**

1. Hash four items together and confirm editing one changes the result.
2. Work out what proving membership would cost: the entire set.
3. Note concatenation without length prefixes is ambiguous too (lesson 04).
4. State the three properties we actually want: O(1) commitment, O(log n) proof, O(n) build.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	items := [][]byte{[]byte("tx-a"), []byte("tx-b"), []byte("tx-c"), []byte("tx-d")}

	// The obvious idea: hash everything together.
	h := sha256.New()
	for _, it := range items {
		h.Write(it)
	}
	flat := h.Sum(nil)
	fmt.Printf("hash of the concatenation: %s\n", hex.EncodeToString(flat))

	// It IS a commitment — change anything and it changes.
	h.Reset()
	for _, it := range [][]byte{[]byte("tx-a"), []byte("tx-B"), []byte("tx-c"), []byte("tx-d")} {
		h.Write(it)
	}
	fmt.Printf("after editing one item:    %s\n", hex.EncodeToString(h.Sum(nil)))

	fmt.Println("\nso what is wrong with it?")
	fmt.Println()
	fmt.Println("  to convince someone that \"tx-c\" is in the set, you must send them")
	fmt.Println("  EVERY item so they can recompute the hash. For a block with 3,000")
	fmt.Println("  transactions that is megabytes, to prove one of them.")

	// Concatenation without length prefixes is also ambiguous (lesson 04).
	h.Reset()
	h.Write([]byte("tx-ab"))
	h.Write([]byte("c"))
	amb1 := h.Sum(nil)
	h.Reset()
	h.Write([]byte("tx-a"))
	h.Write([]byte("bc"))
	amb2 := h.Sum(nil)
	fmt.Printf("\n  and [\"tx-ab\",\"c\"] hashes the same as [\"tx-a\",\"bc\"]: %v\n",
		hex.EncodeToString(amb1) == hex.EncodeToString(amb2))

	fmt.Println("\nwhat we actually want:")
	fmt.Println("  O(1)      commitment    — one 32-byte root in the header")
	fmt.Println("  O(log n)  proof         — 20 hashes for a million items")
	fmt.Println("  O(n)      construction  — build it once per block")
	fmt.Println("\nthat is a Merkle tree.")
}
```

**Output:**

```
hash of the concatenation: 66c6ccfbadf6be72833c47b118b84184384c50e6182a7520733d7cbbee89432e
after editing one item:    e8ba232528cc1ca548af0f800ae6fece97246216405a5c4bb38d7b620c7ac3fe

so what is wrong with it?

  to convince someone that "tx-c" is in the set, you must send them
  EVERY item so they can recompute the hash. For a block with 3,000
  transactions that is megabytes, to prove one of them.

  and ["tx-ab","c"] hashes the same as ["tx-a","bc"]: true

what we actually want:
  O(1)      commitment    — one 32-byte root in the header
  O(log n)  proof         — 20 hashes for a million items
  O(n)      construction  — build it once per block

that is a Merkle tree.
```

---

## 5. What a proof costs

`🟢 easy` · *Proofs*

A proof is one sibling hash per level, and depth is ⌈log₂ n⌉. That single fact is the entire cost model — and it is why light clients are possible at all.

**Steps:**

1. Tabulate depth and proof size against leaf count, up to 2²⁴.
2. Compare a 640-byte proof against sending a million transactions.
3. Note that doubling the set adds exactly one hash to every proof.

```go
package main

import "fmt"

func main() {
	// A proof is one sibling hash per level, and a tree over n leaves is
	// ceil(log2(n)) levels deep. That is the entire cost model.
	fmt.Printf("%-14s %-8s %-12s %s\n", "leaves", "depth", "proof hashes", "proof bytes")
	for _, n := range []int{2, 4, 8, 1024, 1 << 20, 1 << 24} {
		depth := 0
		for (1 << depth) < n {
			depth++
		}
		fmt.Printf("%-14d %-8d %-12d %d\n", n, depth, depth, depth*32)
	}

	// Put that next to the alternative.
	const leaves = 1 << 20
	const avgItem = 250 // bytes per transaction, roughly
	fmt.Printf("\nfor %d leaves of ~%d bytes each:\n", leaves, avgItem)
	fmt.Printf("  send the whole set  %d bytes  (%d MiB)\n",
		leaves*avgItem, leaves*avgItem>>20)
	fmt.Printf("  send a Merkle proof %d bytes  (plus the leaf)\n", 20*32)
	fmt.Printf("  ratio               %dx smaller\n", leaves*avgItem/(20*32))

	// Doubling the set costs exactly one more hash.
	fmt.Println("\ndoubling the number of leaves adds ONE hash to every proof.")
	fmt.Println("that is what makes light clients possible (lesson 64).")
}
```

**Output:**

```
leaves         depth    proof hashes proof bytes
2              1        1            32
4              2        2            64
8              3        3            96
1024           10       10           320
1048576        20       20           640
16777216       24       24           768

for 1048576 leaves of ~250 bytes each:
  send the whole set  262144000 bytes  (250 MiB)
  send a Merkle proof 640 bytes  (plus the leaf)
  ratio               409600x smaller

doubling the number of leaves adds ONE hash to every proof.
that is what makes light clients possible (lesson 64).
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
