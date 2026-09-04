# Step 05 — Merkle Trees & Proofs · 🟡 Medium

Examples **6–13**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. A proof for one leaf of eight

`🟡 medium` · *Proofs*

A reusable tree with `Proof` and `Verify`. The index trick is `i ^ 1` — flipping the last bit gives you the sibling, and `i /= 2` walks up a level. Verification is a twelve-line fold that needs no tree at all.

**Steps:**

1. Build over eight leaves and generate a proof for leaf 2.
2. Print each step with the side its sibling sits on.
3. Verify by folding from the leaf up.
4. Confirm all eight leaves verify against the same root.

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

// Step is one hop up the tree: a sibling hash and which side it sits on.
type Step struct {
	Hash    [32]byte
	IsRight bool // true when the sibling is the RIGHT child
}

type Tree struct{ levels [][][32]byte }

// Build assumes a power-of-two leaf count; odd counts are example 9.
func Build(items [][]byte) *Tree {
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = leafHash(it)
	}
	t := &Tree{levels: [][][32]byte{level}}
	for len(level) > 1 {
		next := make([][32]byte, 0, len(level)/2)
		for i := 0; i < len(level); i += 2 {
			next = append(next, nodeHash(level[i], level[i+1]))
		}
		level = next
		t.levels = append(t.levels, level)
	}
	return t
}

func (t *Tree) Root() [32]byte { return t.levels[len(t.levels)-1][0] }

// Proof collects one sibling per level on the path from leaf to root.
func (t *Tree) Proof(index int) []Step {
	var proof []Step
	for lvl := 0; lvl < len(t.levels)-1; lvl++ {
		sibling := index ^ 1 // flips the last bit: 0<->1, 2<->3, ...
		proof = append(proof, Step{
			Hash:    t.levels[lvl][sibling],
			IsRight: sibling > index,
		})
		index /= 2
	}
	return proof
}

// Verify folds from the leaf up. Twelve lines, and no tree required.
func Verify(leaf []byte, proof []Step, root [32]byte) bool {
	h := leafHash(leaf)
	for _, s := range proof {
		if s.IsRight {
			h = nodeHash(h, s.Hash)
		} else {
			h = nodeHash(s.Hash, h)
		}
	}
	return h == root
}

func main() {
	items := [][]byte{
		[]byte("tx-0"), []byte("tx-1"), []byte("tx-2"), []byte("tx-3"),
		[]byte("tx-4"), []byte("tx-5"), []byte("tx-6"), []byte("tx-7"),
	}
	t := Build(items)
	root := t.Root()
	fmt.Printf("8 leaves, root %s\n", hex.EncodeToString(root[:]))

	const idx = 2
	proof := t.Proof(idx)
	fmt.Printf("\nproof for leaf %d (%q): %d hashes, %d bytes\n",
		idx, items[idx], len(proof), len(proof)*32)
	for i, s := range proof {
		side := "left "
		if s.IsRight {
			side = "right"
		}
		fmt.Printf("  level %d  sibling on the %s  %s\n", i, side, hex.EncodeToString(s.Hash[:8]))
	}

	fmt.Printf("\nverifies: %v\n", Verify(items[idx], proof, root))

	// The verifier never saw the other seven transactions — only three hashes.
	fmt.Println("\nthe verifier holds: the leaf, 3 sibling hashes, and the root.")
	fmt.Println("it has never seen tx-0, tx-1, or tx-3 through tx-7.")

	// Every leaf verifies against the same root.
	all := true
	for i := range items {
		if !Verify(items[i], t.Proof(i), root) {
			all = false
		}
	}
	fmt.Printf("\nall 8 leaves verify against the same root: %v\n", all)
}
```

**Output:**

```
8 leaves, root c8496fa6f7d1f1d06e240c56ab38a54ca4be0746d319d8019a5da05d325e13b0

proof for leaf 2 ("tx-2"): 3 hashes, 96 bytes
  level 0  sibling on the right  b7e599ee270fe7b9
  level 1  sibling on the left   10555b9ebbe71511
  level 2  sibling on the right  fba486ab8b8333b4

verifies: true

the verifier holds: the leaf, 3 sibling hashes, and the root.
it has never seen tx-0, tx-1, or tx-3 through tx-7.

all 8 leaves verify against the same root: true
```

---

## 7. Six ways to break a proof

`🟡 medium` · *Proofs*

Six tampering attempts, all rejected. Worth reading as a checklist of what a verifier is actually enforcing — every hash, every direction, the length of the proof, and the root it is checked against.

**Steps:**

1. Flip one bit of a sibling hash.
2. Keep every hash correct but claim the wrong direction.
3. Substitute a different leaf, truncate the proof, reorder the steps.
4. Try a structurally valid proof from a different tree.

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

type Step struct {
	Hash    [32]byte
	IsRight bool
}

type Tree struct{ levels [][][32]byte }

func Build(items [][]byte) *Tree {
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = leafHash(it)
	}
	t := &Tree{levels: [][][32]byte{level}}
	for len(level) > 1 {
		next := make([][32]byte, 0, len(level)/2)
		for i := 0; i < len(level); i += 2 {
			next = append(next, nodeHash(level[i], level[i+1]))
		}
		level = next
		t.levels = append(t.levels, level)
	}
	return t
}

func (t *Tree) Root() [32]byte { return t.levels[len(t.levels)-1][0] }

func (t *Tree) Proof(i int) []Step {
	var p []Step
	for lvl := 0; lvl < len(t.levels)-1; lvl++ {
		sib := i ^ 1
		p = append(p, Step{Hash: t.levels[lvl][sib], IsRight: sib > i})
		i /= 2
	}
	return p
}

func Verify(leaf []byte, proof []Step, root [32]byte) bool {
	h := leafHash(leaf)
	for _, s := range proof {
		if s.IsRight {
			h = nodeHash(h, s.Hash)
		} else {
			h = nodeHash(s.Hash, h)
		}
	}
	return h == root
}

func clone(p []Step) []Step { return append([]Step(nil), p...) }

func main() {
	items := [][]byte{
		[]byte("tx-0"), []byte("tx-1"), []byte("tx-2"), []byte("tx-3"),
		[]byte("tx-4"), []byte("tx-5"), []byte("tx-6"), []byte("tx-7"),
	}
	t := Build(items)
	root := t.Root()
	good := t.Proof(2)

	fmt.Printf("honest proof for tx-2            %v\n", Verify(items[2], good, root))

	// 1. Corrupt one bit of one sibling hash.
	p1 := clone(good)
	p1[1].Hash[0] ^= 0x01
	fmt.Printf("one bit flipped in a sibling     %v\n", Verify(items[2], p1, root))

	// 2. Right sibling claimed as left. The hashes are all correct.
	p2 := clone(good)
	p2[0].IsRight = !p2[0].IsRight
	fmt.Printf("correct hashes, wrong direction  %v\n", Verify(items[2], p2, root))

	// 3. A different leaf with a valid proof for tx-2.
	fmt.Printf("tx-9 using tx-2's proof          %v\n", Verify([]byte("tx-9"), good, root))

	// 4. A proof that is too short — a truncation attack on a naive verifier.
	fmt.Printf("proof truncated to 2 steps       %v\n", Verify(items[2], good[:2], root))

	// 5. Steps reordered. Order is structural, not cosmetic.
	p5 := clone(good)
	p5[0], p5[2] = p5[2], p5[0]
	fmt.Printf("proof steps reordered            %v\n", Verify(items[2], p5, root))

	// 6. A proof from a DIFFERENT tree with the same shape.
	other := Build([][]byte{
		[]byte("zz-0"), []byte("zz-1"), []byte("zz-2"), []byte("zz-3"),
		[]byte("zz-4"), []byte("zz-5"), []byte("zz-6"), []byte("zz-7"),
	})
	fmt.Printf("proof from another tree          %v\n", Verify(items[2], other.Proof(2), root))

	fmt.Printf("\nroot %s\n", hex.EncodeToString(root[:12]))
	fmt.Println("\nevery tampering path fails, because the fold cannot reach the root")
	fmt.Println("unless every hash AND every direction is exactly right.")
}
```

**Output:**

```
honest proof for tx-2            true
one bit flipped in a sibling     false
correct hashes, wrong direction  false
tx-9 using tx-2's proof          false
proof truncated to 2 steps       false
proof steps reordered            false
proof from another tree          false

root c8496fa6f7d1f1d06e240c56

every tampering path fails, because the fold cannot reach the root
unless every hash AND every direction is exactly right.
```

---

## 8. What a proof does not prove

`🟡 medium` · *Proofs*

The most important example in the lesson. A valid proof proves exactly one thing: this leaf is somewhere in the tree with this root. It says nothing about uniqueness, position, absence — or whether the root is the *right* root.

**Steps:**

1. Build a tree containing the same leaf twice and produce two valid proofs for it.
2. Verify a proof against an attacker-chosen root and watch it pass.
3. Understand the consequence: the root must come from somewhere you trust, never from the prover.
4. For an airdrop this means tracking claims on-chain — one proof is replayable forever.

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

type Step struct {
	Hash    [32]byte
	IsRight bool
}

func Verify(leaf []byte, proof []Step, root [32]byte) bool {
	h := leafHash(leaf)
	for _, s := range proof {
		if s.IsRight {
			h = nodeHash(h, s.Hash)
		} else {
			h = nodeHash(s.Hash, h)
		}
	}
	return h == root
}

func main() {
	// A tree containing the SAME leaf twice.
	a := leafHash([]byte("alice:100"))
	b := leafHash([]byte("alice:100")) // duplicate, deliberately
	c := leafHash([]byte("bob:50"))
	d := leafHash([]byte("carol:25"))

	n0 := nodeHash(a, b)
	n1 := nodeHash(c, d)
	root := nodeHash(n0, n1)
	fmt.Printf("root %s\n", hex.EncodeToString(root[:]))

	// Both copies produce valid proofs.
	p0 := []Step{{Hash: b, IsRight: true}, {Hash: n1, IsRight: true}}
	p1 := []Step{{Hash: a, IsRight: false}, {Hash: n1, IsRight: true}}
	fmt.Printf("\n\"alice:100\" at index 0 verifies: %v\n", Verify([]byte("alice:100"), p0, root))
	fmt.Printf("\"alice:100\" at index 1 verifies: %v\n", Verify([]byte("alice:100"), p1, root))

	fmt.Println("\nA VALID PROOF PROVES ONE THING:")
	fmt.Println("  this leaf is somewhere in the tree with this root.")
	fmt.Println()
	fmt.Println("it does NOT prove:")
	fmt.Println("  - that the leaf appears only once   (shown above: it appears twice)")
	fmt.Println("  - that the leaf is at a given index (unless the index is in the leaf)")
	fmt.Println("  - anything about leaves NOT in the tree (see sparse trees, example 16)")
	fmt.Println("  - that the root is the RIGHT root    (that is the caller's problem)")

	// The last one is the most dangerous in practice.
	fake := nodeHash(leafHash([]byte("mallory:1000000")), leafHash([]byte("padding")))
	fp := []Step{{Hash: leafHash([]byte("padding")), IsRight: true}}
	fmt.Printf("\na proof against an ATTACKER-CHOSEN root verifies too: %v\n",
		Verify([]byte("mallory:1000000"), fp, fake))
	fmt.Println("  so the root must come from somewhere you trust — a block header,")
	fmt.Println("  a signed message, or a contract's storage. Never from the prover.")

	// Practical consequence for airdrops: put the index or a nullifier in the
	// leaf, and track claims, or the same proof can be replayed.
	fmt.Println("\nfor an airdrop this means: mark each leaf as claimed on-chain,")
	fmt.Println("or one valid proof can be submitted over and over (example 14).")
}
```

**Output:**

```
root 08aa2c7c72255430b1a848e646db6ec301ad199c2fb1245912169aa4048cf919

"alice:100" at index 0 verifies: true
"alice:100" at index 1 verifies: true

A VALID PROOF PROVES ONE THING:
  this leaf is somewhere in the tree with this root.

it does NOT prove:
  - that the leaf appears only once   (shown above: it appears twice)
  - that the leaf is at a given index (unless the index is in the leaf)
  - anything about leaves NOT in the tree (see sparse trees, example 16)
  - that the root is the RIGHT root    (that is the caller's problem)

a proof against an ATTACKER-CHOSEN root verifies too: true
  so the root must come from somewhere you trust — a block header,
  a signed message, or a contract's storage. Never from the prover.

for an airdrop this means: mark each leaf as claimed on-chain,
or one valid proof can be submitted over and over (example 14).
```

---

## 9. Odd leaves, three conventions

`🟡 medium` · *Odd leaves*

Levels with an odd node count have to be handled somehow, and there are three common answers. All three are self-consistent, all three give different roots, and — crucially — they agree when the leaf count is a power of two, which is how the mismatch hides in your tests.

**Steps:**

1. Compute the root over five leaves with duplicate, promote and pad.
2. Confirm all three differ.
3. Confirm all three agree on four leaves, so a 4-leaf test proves nothing.
4. Pick one, document it, and test an odd count.

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

// Three ways to fold a level with an odd number of nodes.
type Rule int

const (
	Duplicate Rule = iota // Bitcoin: hash the last node with itself
	Promote               // carry the odd node up unchanged
	Pad                   // pad the level to a power of two with a sentinel
)

var sentinel = leafHash([]byte("\x00SENTINEL"))

func root(items [][]byte, rule Rule) [32]byte {
	level := make([][32]byte, len(items))
	for i, it := range items {
		level[i] = leafHash(it)
	}
	if rule == Pad {
		for len(level)&(len(level)-1) != 0 { // until a power of two
			level = append(level, sentinel)
		}
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				switch rule {
				case Duplicate:
					next = append(next, nodeHash(level[i], level[i])) // itself
				case Promote:
					next = append(next, level[i]) // carried up
				}
				continue
			}
			next = append(next, nodeHash(level[i], level[i+1]))
		}
		level = next
	}
	return level[0]
}

func main() {
	five := [][]byte{[]byte("a"), []byte("b"), []byte("c"), []byte("d"), []byte("e")}

	names := map[Rule]string{Duplicate: "duplicate", Promote: "promote", Pad: "pad"}
	fmt.Println("five leaves, three conventions:")
	for _, r := range []Rule{Duplicate, Promote, Pad} {
		h := root(five, r)
		fmt.Printf("  %-10s %s\n", names[r], hex.EncodeToString(h[:]))
	}

	fmt.Println("\nall three are internally consistent. All three are DIFFERENT.")
	fmt.Println("a proof generated under one will never verify under another.")

	// With a power-of-two count they agree, which is how the mismatch hides.
	four := [][]byte{[]byte("a"), []byte("b"), []byte("c"), []byte("d")}
	rd, rp, rq := root(four, Duplicate), root(four, Promote), root(four, Pad)
	fmt.Printf("\nfour leaves — all three rules agree: %v\n", rd == rp && rp == rq)
	fmt.Println("so your tests pass on 4 leaves and production breaks on 5.")

	fmt.Println("\npick one, write it down, and test an odd count.")
	fmt.Println("  duplicate — Bitcoin. Has a known flaw (example 10).")
	fmt.Println("  promote   — most libraries. Simple and safe.")
	fmt.Println("  pad       — fixed depth, which some circuits and SMTs need.")
}
```

**Output:**

```
five leaves, three conventions:
  duplicate  605c72ca9351dd39f38678f4c1326df06d8fb1a58272792acaf70e8c191fb823
  promote    fe14a5426fbd70c0fa73f52342afed0da0bd23c4838662ccf6b88a3070ead97b
  pad        3ae870dc37df77d2b5318276b05dab541d06542bc2f20d587d8be3d3ebfabfe9

all three are internally consistent. All three are DIFFERENT.
a proof generated under one will never verify under another.

four leaves — all three rules agree: true
so your tests pass on 4 leaves and production breaks on 5.

pick one, write it down, and test an odd count.
  duplicate — Bitcoin. Has a known flaw (example 10).
  promote   — most libraries. Simple and safe.
  pad       — fixed depth, which some circuits and SMTs need.
```

---

## 10. CVE-2012-2459: two blocks, one root

`🟡 medium` · *Odd leaves*

Bitcoin duplicates the odd node, and that turned out to be exploitable. A block with transactions `[a b c]` and one with `[a b c c]` produce the **identical** Merkle root, so the mutated block has the same block hash — while being invalid.

**Steps:**

1. Implement Bitcoin's exact rules: double SHA-256, no tags, duplicate the odd node.
2. Compute the root for `[a b c]` and for `[a b c c]`.
3. Confirm they are identical.
4. Read why that was a network-splitting denial of service, and how Core fixed it.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// Bitcoin's actual rules: no domain tags, double SHA-256, and an odd node is
// hashed with ITSELF.
func hash256(b []byte) [32]byte {
	first := sha256.Sum256(b)
	return sha256.Sum256(first[:])
}

func txid(data []byte) [32]byte { return hash256(data) }

func merkleRoot(txids [][32]byte) [32]byte {
	level := append([][32]byte(nil), txids...)
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			r := i + 1
			if r == len(level) {
				r = i // duplicate the last one
			}
			buf := append(append([]byte{}, level[i][:]...), level[r][:]...)
			next = append(next, hash256(buf))
		}
		level = next
	}
	return level[0]
}

func main() {
	a, b, c := txid([]byte("tx-a")), txid([]byte("tx-b")), txid([]byte("tx-c"))

	// A block with three transactions.
	three := [][32]byte{a, b, c}
	// And a block with four, where the last is a copy of the third.
	four := [][32]byte{a, b, c, c}

	r3 := merkleRoot(three)
	r4 := merkleRoot(four)

	fmt.Printf("block with [a b c]     root %s\n", hex.EncodeToString(r3[:]))
	fmt.Printf("block with [a b c c]   root %s\n", hex.EncodeToString(r4[:]))
	fmt.Printf("\nIDENTICAL ROOTS: %v\n", r3 == r4)

	fmt.Println("\nCVE-2012-2459, in one line: because an odd node is hashed with")
	fmt.Println("itself, appending a copy of the last transaction does not change")
	fmt.Println("the Merkle root — and therefore does not change the block hash.")

	fmt.Println("\nwhy that mattered:")
	fmt.Println("  an attacker takes a valid block and duplicates its last transaction.")
	fmt.Println("  the mutated block has the SAME hash, so nodes think they have seen it,")
	fmt.Println("  but it is INVALID (a duplicate input is a double-spend).")
	fmt.Println("  nodes that cached the failure would then reject the honest block too.")
	fmt.Println("  result: a network-splitting denial of service.")

	fmt.Println("\nthe fix in Bitcoin Core was to detect the duplication explicitly,")
	fmt.Println("not to change the tree. The rule is consensus-critical and permanent.")

	// The same trick works at any level with an odd count.
	five := [][32]byte{a, b, c, a, b}
	six := [][32]byte{a, b, c, a, b, b}
	fmt.Printf("\nsame trick with 5 vs 6 leaves: %v\n", merkleRoot(five) == merkleRoot(six))

	fmt.Println("\nif you are designing a new tree: promote or pad, and domain-separate.")
}
```

**Output:**

```
block with [a b c]     root 8b1d539cf3a142f7dfc56a1049835fee91c29d2afb21e2a3b9777286048fd8fa
block with [a b c c]   root 8b1d539cf3a142f7dfc56a1049835fee91c29d2afb21e2a3b9777286048fd8fa

IDENTICAL ROOTS: true

CVE-2012-2459, in one line: because an odd node is hashed with
itself, appending a copy of the last transaction does not change
the Merkle root — and therefore does not change the block hash.

why that mattered:
  an attacker takes a valid block and duplicates its last transaction.
  the mutated block has the SAME hash, so nodes think they have seen it,
  but it is INVALID (a duplicate input is a double-spend).
  nodes that cached the failure would then reject the honest block too.
  result: a network-splitting denial of service.

the fix in Bitcoin Core was to detect the duplication explicitly,
not to change the tree. The rule is consensus-critical and permanent.

same trick with 5 vs 6 leaves: true

if you are designing a new tree: promote or pad, and domain-separate.
```

---

## 11. The second-preimage attack

`🟡 medium` · *Second preimage*

The attack on an untagged tree. Without domain separation, `leafHash(x) = H(x)` and `nodeHash(l,r) = H(l‖r)` — so a 64-byte "leaf" containing `hA‖hB` hashes to exactly the internal node above them. You can prove membership of data that was never in the tree.

**Steps:**

1. Build a naive four-leaf tree with no tags.
2. Construct a 64-byte forged leaf equal to `hA‖hB`.
3. Confirm its leaf hash equals the internal node.
4. Verify it against the honest root with a one-step proof.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// A tree with NO domain separation — the naive implementation almost everyone
// writes first, and the one many libraries still ship.
func leafHash(d []byte) [32]byte { return sha256.Sum256(d) }

func nodeHash(l, r [32]byte) [32]byte {
	return sha256.Sum256(append(append([]byte{}, l[:]...), r[:]...))
}

type Step struct {
	Hash    [32]byte
	IsRight bool
}

func Verify(leaf []byte, proof []Step, root [32]byte) bool {
	h := leafHash(leaf)
	for _, s := range proof {
		if s.IsRight {
			h = nodeHash(h, s.Hash)
		} else {
			h = nodeHash(s.Hash, h)
		}
	}
	return h == root
}

func main() {
	// An honest 4-leaf tree.
	items := [][]byte{[]byte("tx-a"), []byte("tx-b"), []byte("tx-c"), []byte("tx-d")}
	hA, hB := leafHash(items[0]), leafHash(items[1])
	hC, hD := leafHash(items[2]), leafHash(items[3])
	n0 := nodeHash(hA, hB)
	n1 := nodeHash(hC, hD)
	root := nodeHash(n0, n1)

	fmt.Printf("honest tree, 4 leaves, root %s\n", hex.EncodeToString(root[:]))
	honest := []Step{{Hash: hB, IsRight: true}, {Hash: n1, IsRight: true}}
	fmt.Printf("\"tx-a\" verifies: %v\n", Verify(items[0], honest, root))

	// THE ATTACK.
	// Without tags, leafHash(x) = H(x) and nodeHash(l,r) = H(l||r).
	// So a 64-byte "leaf" whose contents are exactly hA||hB hashes to n0.
	forgedLeaf := append(append([]byte{}, hA[:]...), hB[:]...)
	fmt.Printf("\nforged 'transaction' = hA || hB (%d bytes)\n", len(forgedLeaf))
	fmt.Printf("  leafHash(forged) == n0 ? %v\n", leafHash(forgedLeaf) == n0)

	// So it verifies against the SAME root, with a one-step proof.
	forgedProof := []Step{{Hash: n1, IsRight: true}}
	fmt.Printf("\nforged leaf verifies against the honest root: %v\n",
		Verify(forgedLeaf, forgedProof, root))

	fmt.Println("\nthat is a SECOND PREIMAGE: a different set of leaves —")
	fmt.Println("  {hA||hB, hC||hD}  instead of  {tx-a, tx-b, tx-c, tx-d}")
	fmt.Println("producing the identical root. Anyone can 'prove' membership of")
	fmt.Println("data that was never in the tree.")

	fmt.Println("\nwhen this bites: any system where a leaf can be 64 bytes and the")
	fmt.Println("verifier does not pin the tree's depth or the leaf's format.")
	fmt.Println("example 12 fixes it with one byte per hash.")
}
```

**Output:**

```
honest tree, 4 leaves, root 3ae3fd9168a3f2df9cf178008994dceef7ce6930b0f819d42a6451dee131f36f
"tx-a" verifies: true

forged 'transaction' = hA || hB (64 bytes)
  leafHash(forged) == n0 ? true

forged leaf verifies against the honest root: true

that is a SECOND PREIMAGE: a different set of leaves —
  {hA||hB, hC||hD}  instead of  {tx-a, tx-b, tx-c, tx-d}
producing the identical root. Anyone can 'prove' membership of
data that was never in the tree.

when this bites: any system where a leaf can be 64 bytes and the
verifier does not pin the tree's depth or the leaf's format.
example 12 fixes it with one byte per hash.
```

---

## 12. One byte of domain separation

`🟡 medium` · *Second preimage*

The defence, from RFC 6962: prefix leaves with `0x00` and internal nodes with `0x01`. The two preimages can now never be the same bytes, so the example 11 attack simply cannot be constructed. Cost: one byte per hash.

**Steps:**

1. Rebuild the same tree with domain tags.
2. Confirm honest proofs still verify.
3. Attempt the forgery and watch both the hash equality and the proof fail.
4. Compare the two preimages side by side — the tag byte is the entire defence.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// The defence, from RFC 6962 (Certificate Transparency): distinct prefixes for
// leaves and internal nodes. One byte, and the attack in example 11 dies.
const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func leafHash(d []byte) [32]byte { return sha256.Sum256(append([]byte{tagLeaf}, d...)) }

func nodeHash(l, r [32]byte) [32]byte {
	buf := append([]byte{tagNode}, l[:]...)
	return sha256.Sum256(append(buf, r[:]...))
}

type Step struct {
	Hash    [32]byte
	IsRight bool
}

func Verify(leaf []byte, proof []Step, root [32]byte) bool {
	h := leafHash(leaf)
	for _, s := range proof {
		if s.IsRight {
			h = nodeHash(h, s.Hash)
		} else {
			h = nodeHash(s.Hash, h)
		}
	}
	return h == root
}

func main() {
	items := [][]byte{[]byte("tx-a"), []byte("tx-b"), []byte("tx-c"), []byte("tx-d")}
	hA, hB := leafHash(items[0]), leafHash(items[1])
	hC, hD := leafHash(items[2]), leafHash(items[3])
	n0, n1 := nodeHash(hA, hB), nodeHash(hC, hD)
	root := nodeHash(n0, n1)

	fmt.Printf("tagged tree, root %s\n", hex.EncodeToString(root[:]))
	honest := []Step{{Hash: hB, IsRight: true}, {Hash: n1, IsRight: true}}
	fmt.Printf("honest proof still verifies: %v\n", Verify(items[0], honest, root))

	// The same attack, attempted.
	forged := append(append([]byte{}, hA[:]...), hB[:]...)
	fmt.Printf("\nleafHash(hA||hB) == n0 ? %v\n", leafHash(forged) == n0)
	fmt.Printf("  because leaf hashing prepends 0x00 and node hashing prepends 0x01,\n")
	fmt.Printf("  so the two preimages can never be the same bytes.\n")

	fmt.Printf("\nforged leaf verifies: %v\n",
		Verify(forged, []Step{{Hash: n1, IsRight: true}}, root))

	// Show the preimages side by side.
	leafPre := append([]byte{tagLeaf}, forged...)
	nodePre := append(append([]byte{tagNode}, hA[:]...), hB[:]...)
	fmt.Printf("\nleaf preimage starts %s... (%d bytes)\n",
		hex.EncodeToString(leafPre[:5]), len(leafPre))
	fmt.Printf("node preimage starts %s... (%d bytes)\n",
		hex.EncodeToString(nodePre[:5]), len(nodePre))
	fmt.Println("                     ^^ the tag byte is the whole defence")

	fmt.Println("\nRFC 6962 does exactly this, and it is why Certificate Transparency")
	fmt.Println("logs are safe against the attack. Cost: one byte per hash.")
	fmt.Println("\nsome libraries omit it and are 'fine' because their leaves are always")
	fmt.Println("32 bytes. That is a property of the caller, not the tree. Do not rely on it.")
}
```

**Output:**

```
tagged tree, root 6e5818a6c1d68c5aa635dcb16b6aa1166c374ed812682c807d3cf0f7c4b854df
honest proof still verifies: true

leafHash(hA||hB) == n0 ? false
  because leaf hashing prepends 0x00 and node hashing prepends 0x01,
  so the two preimages can never be the same bytes.

forged leaf verifies: false

leaf preimage starts 0024536162... (65 bytes)
node preimage starts 0124536162... (65 bytes)
                     ^^ the tag byte is the whole defence

RFC 6962 does exactly this, and it is why Certificate Transparency
logs are safe against the attack. Cost: one byte per hash.

some libraries omit it and are 'fine' because their leaves are always
32 bytes. That is a property of the caller, not the tree. Do not rely on it.
```

---

## 13. Sorted pairs, no direction bits

`🟡 medium` · *Sorted pairs*

OpenZeppelin's convention sorts each pair before hashing, so the order is derived from the values and a proof is just a list of hashes with no direction bits. Simpler on-chain, and slightly cheaper — but it can no longer prove position, only membership.

**Steps:**

1. Implement `hashPair` with a lexicographic sort and keccak256.
2. Verify a proof with no direction information at all.
3. Understand the trade-off in both directions.
4. Note the caveat that example 14 fixes: sorted pairs plus single-hashed leaves reopens example 11.

```go
package main

import (
	"bytes"
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/crypto"
)

// The convention OpenZeppelin's MerkleProof.sol uses: sort each pair before
// hashing. Because the order is derived from the values, a proof does not need
// to say which side each sibling is on — it is just a list of hashes.
func hashPair(a, b [32]byte) [32]byte {
	if bytes.Compare(a[:], b[:]) > 0 {
		a, b = b, a
	}
	return crypto.Keccak256Hash(append(a[:], b[:]...))
}

func leaf(d []byte) [32]byte { return crypto.Keccak256Hash(d) }

// Verify: fold the proof into the leaf. No direction bits anywhere.
func Verify(l [32]byte, proof [][32]byte, root [32]byte) bool {
	h := l
	for _, p := range proof {
		h = hashPair(h, p)
	}
	return h == root
}

func main() {
	items := [][]byte{[]byte("a"), []byte("b"), []byte("c"), []byte("d")}
	leaves := make([][32]byte, len(items))
	for i, it := range items {
		leaves[i] = leaf(it)
	}

	n0 := hashPair(leaves[0], leaves[1])
	n1 := hashPair(leaves[2], leaves[3])
	root := hashPair(n0, n1)

	fmt.Println("sorted-pair tree (keccak256)")
	for i, h := range leaves {
		fmt.Printf("  leaf %d %q %s\n", i, items[i], hex.EncodeToString(h[:8]))
	}
	fmt.Printf("  root     %s\n", hex.EncodeToString(root[:]))

	// A proof is just hashes — the verifier sorts at each step.
	proof := [][32]byte{leaves[1], n1}
	fmt.Printf("\nproof for \"a\": %d hashes, no direction bits\n", len(proof))
	fmt.Printf("verifies: %v\n", Verify(leaves[0], proof, root))

	// Which side leaf 0 was on is irrelevant — sorting decides.
	fmt.Printf("\nleaf 0 < leaf 1 lexicographically: %v\n",
		bytes.Compare(leaves[0][:], leaves[1][:]) < 0)
	fmt.Println("whichever it is, hashPair puts them in the same order every time.")

	// The trade-off. An unsorted tree distinguishes position; a sorted one does not.
	fmt.Println("\nwhat you gain: simpler proofs, and ~20 gas less per step on-chain")
	fmt.Println("               because the contract does not branch on a bit.")
	fmt.Println("what you lose: the tree no longer commits to WHICH SIDE a leaf is on,")
	fmt.Println("               so it cannot prove position — only membership.")

	// The caveat: with sorted pairs, a proof element that happens to equal a
	// real internal node can let a 64-byte leaf impersonate one. OpenZeppelin's
	// answer is to hash leaves TWICE so no leaf hash can equal a node hash.
	fmt.Println("\ncaveat: sorted pairs plus single-hashed leaves reopens the")
	fmt.Println("example 11 problem. OpenZeppelin's fix is to hash leaves twice —")
	fmt.Println("that is what example 14 implements.")

	// And never mix the two conventions.
	fmt.Println("\nan unsorted proof will not verify against a sorted verifier.")
	fmt.Println("they are different trees with different roots. Pick one per system.")
}
```

**Output:**

```
sorted-pair tree (keccak256)
  leaf 0 "a" 3ac225168df54212
  leaf 1 "b" b5553de315e0edf5
  leaf 2 "c" 0b42b6393c1f5306
  leaf 3 "d" f1918e8562236eb1
  root     68203f90e9d07dc5859259d7536e87a6ba9d345f2552b5b9de2999ddce9ce1bf

proof for "a": 2 hashes, no direction bits
verifies: true

leaf 0 < leaf 1 lexicographically: true
whichever it is, hashPair puts them in the same order every time.

what you gain: simpler proofs, and ~20 gas less per step on-chain
               because the contract does not branch on a bit.
what you lose: the tree no longer commits to WHICH SIDE a leaf is on,
               so it cannot prove position — only membership.

caveat: sorted pairs plus single-hashed leaves reopens the
example 11 problem. OpenZeppelin's fix is to hash leaves twice —
that is what example 14 implements.

an unsorted proof will not verify against a sorted verifier.
they are different trees with different roots. Pick one per system.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
