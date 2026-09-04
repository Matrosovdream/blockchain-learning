# Step 05 — Merkle Trees & Proofs · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. An OpenZeppelin-compatible airdrop

`🔴 hard` · *In production*

A complete airdrop allowlist whose proofs OpenZeppelin's `MerkleProof.verify` accepts. Three rules make it compatible: leaves hashed **twice**, pairs sorted, proofs bare. The double leaf hash is what closes the second-preimage hole in a sorted tree.

**Steps:**

1. Encode each claim as Solidity's `abi.encode(address,uint256)` would (lesson 03).
2. Hash leaves twice: `keccak256(keccak256(encoded))`.
3. Build with sorted pairs and generate per-claimant proofs.
4. Verify with a faithful Go port of OpenZeppelin's `verify` loop.
5. Confirm an outsider cannot get a proof, and an on-list address cannot inflate their amount.

```go
package main

import (
	"bytes"
	"encoding/hex"
	"fmt"
	"math/big"
	"sort"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// An airdrop allowlist whose proofs OpenZeppelin's MerkleProof.verify accepts.
//
// The three rules that make it compatible:
//   1. leaf  = keccak256(keccak256(abi.encode(account, amount)))   <- hashed TWICE
//   2. node  = keccak256(sorted(left, right))
//   3. proof = a bare list of hashes, no direction bits
//
// The double leaf hash is what stops a 64-byte leaf impersonating an internal
// node (examples 11 and 13): a leaf digest is a hash OF a hash, so it can never
// be read as two concatenated child hashes.

type Claim struct {
	Account common.Address
	Amount  *big.Int
}

// abiEncode produces the same 64 bytes as Solidity's abi.encode(address,uint256):
// each value left-padded into a 32-byte word (lesson 03).
func abiEncode(c Claim) []byte {
	out := make([]byte, 0, 64)
	out = append(out, common.LeftPadBytes(c.Account.Bytes(), 32)...)
	return append(out, common.LeftPadBytes(c.Amount.Bytes(), 32)...)
}

func leafHash(c Claim) [32]byte {
	inner := crypto.Keccak256(abiEncode(c))
	return crypto.Keccak256Hash(inner)
}

func hashPair(a, b [32]byte) [32]byte {
	if bytes.Compare(a[:], b[:]) > 0 {
		a, b = b, a
	}
	return crypto.Keccak256Hash(append(a[:], b[:]...))
}

type Tree struct{ levels [][][32]byte }

func Build(claims []Claim) *Tree {
	leaves := make([][32]byte, len(claims))
	for i, c := range claims {
		leaves[i] = leafHash(c)
	}
	// Sorting leaves is conventional for allowlists: it makes the tree
	// independent of input order, so the same set always gives the same root.
	sort.Slice(leaves, func(i, j int) bool {
		return bytes.Compare(leaves[i][:], leaves[j][:]) < 0
	})

	t := &Tree{levels: [][][32]byte{leaves}}
	level := leaves
	for len(level) > 1 {
		next := make([][32]byte, 0, (len(level)+1)/2)
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i]) // promote the odd node
				continue
			}
			next = append(next, hashPair(level[i], level[i+1]))
		}
		level = next
		t.levels = append(t.levels, level)
	}
	return t
}

func (t *Tree) Root() [32]byte { return t.levels[len(t.levels)-1][0] }

func (t *Tree) Proof(target [32]byte) ([][32]byte, bool) {
	idx := -1
	for i, h := range t.levels[0] {
		if h == target {
			idx = i
			break
		}
	}
	if idx < 0 {
		return nil, false
	}
	var proof [][32]byte
	for lvl := 0; lvl < len(t.levels)-1; lvl++ {
		sib := idx ^ 1
		if sib < len(t.levels[lvl]) { // an odd promoted node has no sibling
			proof = append(proof, t.levels[lvl][sib])
		}
		idx /= 2
	}
	return proof, true
}

// VerifySolidity is a faithful Go port of OpenZeppelin's MerkleProof.verify:
//
//	for (uint256 i = 0; i < proof.length; i++) {
//	    computedHash = _hashPair(computedHash, proof[i]);
//	}
//	return computedHash == root;
func VerifySolidity(leaf [32]byte, proof [][32]byte, root [32]byte) bool {
	h := leaf
	for _, p := range proof {
		h = hashPair(h, p)
	}
	return h == root
}

func eth(n int64) *big.Int {
	return new(big.Int).Mul(big.NewInt(n), new(big.Int).Exp(big.NewInt(10), big.NewInt(18), nil))
}

func main() {
	// Public anvil test accounts (lesson 02) — no real funds anywhere.
	claims := []Claim{
		{common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"), eth(100)},
		{common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8"), eth(250)},
		{common.HexToAddress("0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"), eth(75)},
		{common.HexToAddress("0x90F79bf6EB2c4f870365E785982E1f101E93b906"), eth(500)},
		{common.HexToAddress("0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65"), eth(10)},
	}

	t := Build(claims)
	root := t.Root()
	fmt.Printf("allowlist of %d claims\n", len(claims))
	fmt.Printf("merkle root  %s\n", hex.EncodeToString(root[:]))
	fmt.Println("  ^ this is the value you deploy in the contract's constructor")

	// Each claimant gets their leaf inputs and a proof.
	fmt.Println("\nper-claimant proofs (what you publish in a JSON file):")
	for _, c := range claims[:3] {
		lh := leafHash(c)
		proof, ok := t.Proof(lh)
		if !ok {
			fmt.Println("  no proof for", c.Account)
			continue
		}
		// new(big.Int).Div, not c.Amount.Div — the latter would overwrite the
		// claim we are about to hash again (lesson 03, example 11).
		whole := new(big.Int).Div(c.Amount, eth(1))
		fmt.Printf("  %s  %s ETH  proof=%d hashes  verifies=%v\n",
			c.Account.Hex()[:12]+"...", whole,
			len(proof), VerifySolidity(lh, proof, root))
	}

	// Someone not on the list cannot construct a proof.
	outsider := Claim{common.HexToAddress("0xa0Ee7A142d267C1f36714E4a8F75612F20a79720"), eth(100)}
	_, ok := t.Proof(leafHash(outsider))
	fmt.Printf("\naddress not on the list: proof found = %v\n", ok)

	// And an on-list address cannot inflate their amount: a different amount is
	// a different leaf, which is not in the tree.
	inflated := Claim{claims[0].Account, eth(999999)}
	_, ok = t.Proof(leafHash(inflated))
	fmt.Printf("on-list address, inflated amount: proof found = %v\n", ok)

	fmt.Println("\nthe Solidity side:")
	fmt.Println("  bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(msg.sender, amount))));")
	fmt.Println("  require(MerkleProof.verify(proof, root, leaf), \"bad proof\");")
	fmt.Println("  require(!claimed[msg.sender], \"already claimed\");   <- example 8's lesson")
}
```

**Output:**

```
allowlist of 5 claims
merkle root  6692544bafb1ffeb6e11c48433324410de65a140679e51fa909ad94ff8af6c4d
  ^ this is the value you deploy in the contract's constructor

per-claimant proofs (what you publish in a JSON file):
  0xf39Fd6e51a...  100 ETH  proof=3 hashes  verifies=true
  0x70997970C5...  250 ETH  proof=1 hashes  verifies=true
  0x3C44CdDdB6...  75 ETH  proof=3 hashes  verifies=true

address not on the list: proof found = false
on-list address, inflated amount: proof found = false

the Solidity side:
  bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(msg.sender, amount))));
  require(MerkleProof.verify(proof, root, leaf), "bad proof");
  require(!claimed[msg.sender], "already claimed");   <- example 8's lesson
```

---

## 15. A multiproof for three leaves

`🔴 hard` · *In production*

Proving several leaves at once costs far less than proving each separately, because their paths overlap near the root. Here three leaves of eight need four hashes instead of nine — and the saving grows with the batch.

**Steps:**

1. Generate three independent proofs and count the hashes.
2. Generate one multiproof that omits anything the verifier can derive itself.
3. Verify it, then break it two ways.
4. This is what rollups batching withdrawals rely on (lesson 67).

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"sort"
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

// SingleProof is the ordinary one-leaf proof, for comparison.
func (t *Tree) SingleProof(i int) [][32]byte {
	var p [][32]byte
	for lvl := 0; lvl < len(t.levels)-1; lvl++ {
		p = append(p, t.levels[lvl][i^1])
		i /= 2
	}
	return p
}

// MultiProof emits only the hashes the verifier CANNOT derive itself.
// Walking level by level: if both children of a pair are already known, the
// parent is computed and no proof hash is needed. Only half-known pairs
// contribute a sibling.
func (t *Tree) MultiProof(indices []int) [][32]byte {
	known := map[int]bool{}
	for _, i := range indices {
		known[i] = true
	}
	var proof [][32]byte
	for lvl := 0; lvl < len(t.levels)-1; lvl++ {
		var idx []int
		for i := range known {
			idx = append(idx, i)
		}
		sort.Ints(idx)

		next := map[int]bool{}
		for _, i := range idx {
			if i%2 == 1 && known[i-1] {
				continue // already handled as the left of this pair
			}
			sib := i ^ 1
			if !known[sib] {
				proof = append(proof, t.levels[lvl][sib])
			}
			next[i/2] = true
		}
		known = next
	}
	return proof
}

// VerifyMulti consumes the proof in the same order MultiProof produced it.
func VerifyMulti(indices []int, leaves [][]byte, proof [][32]byte, depth int, root [32]byte) bool {
	known := map[int][32]byte{}
	for k, i := range indices {
		known[i] = leafHash(leaves[k])
	}
	pos := 0
	for lvl := 0; lvl < depth; lvl++ {
		var idx []int
		for i := range known {
			idx = append(idx, i)
		}
		sort.Ints(idx)

		next := map[int][32]byte{}
		for _, i := range idx {
			if i%2 == 1 {
				if _, ok := known[i-1]; ok {
					continue
				}
			}
			sib := i ^ 1
			var sh [32]byte
			if h, ok := known[sib]; ok {
				sh = h
			} else {
				if pos >= len(proof) {
					return false
				}
				sh = proof[pos]
				pos++
			}
			if i%2 == 0 {
				next[i/2] = nodeHash(known[i], sh)
			} else {
				next[i/2] = nodeHash(sh, known[i])
			}
		}
		known = next
	}
	if pos != len(proof) || len(known) != 1 {
		return false
	}
	return known[0] == root
}

func main() {
	items := [][]byte{
		[]byte("tx-0"), []byte("tx-1"), []byte("tx-2"), []byte("tx-3"),
		[]byte("tx-4"), []byte("tx-5"), []byte("tx-6"), []byte("tx-7"),
	}
	t := Build(items)
	root := t.Root()
	depth := len(t.levels) - 1

	want := []int{1, 4, 6}
	fmt.Printf("proving leaves %v of %d\n\n", want, len(items))

	// Three independent proofs.
	total := 0
	for _, i := range want {
		n := len(t.SingleProof(i))
		total += n
		fmt.Printf("  single proof for leaf %d: %d hashes\n", i, n)
	}
	fmt.Printf("  three separate proofs   : %d hashes (%d bytes)\n", total, total*32)

	// One multiproof.
	mp := t.MultiProof(want)
	fmt.Printf("\n  multiproof              : %d hashes (%d bytes)\n", len(mp), len(mp)*32)
	fmt.Printf("  saved                   : %d hashes\n", total-len(mp))

	leaves := [][]byte{items[1], items[4], items[6]}
	fmt.Printf("\nmultiproof verifies: %v\n", VerifyMulti(want, leaves, mp, depth, root))

	// Tamper with it and it fails.
	bad := append([][32]byte(nil), mp...)
	bad[0][0] ^= 0x01
	fmt.Printf("with one bit flipped: %v\n", VerifyMulti(want, leaves, bad, depth, root))

	// Wrong leaf content fails too.
	wrong := [][]byte{items[1], []byte("tx-X"), items[6]}
	fmt.Printf("with a wrong leaf   : %v\n", VerifyMulti(want, wrong, mp, depth, root))

	fmt.Printf("\nroot %s\n", hex.EncodeToString(root[:12]))
	fmt.Println("\nthe more leaves you prove together, the more their paths overlap.")
	fmt.Println("rollups batching withdrawals rely on exactly this (lesson 67).")
}
```

**Output:**

```
proving leaves [1 4 6] of 8

  single proof for leaf 1: 3 hashes
  single proof for leaf 4: 3 hashes
  single proof for leaf 6: 3 hashes
  three separate proofs   : 9 hashes (288 bytes)

  multiproof              : 4 hashes (128 bytes)
  saved                   : 5 hashes

multiproof verifies: true
with one bit flipped: false
with a wrong leaf   : false

root c8496fa6f7d1f1d06e240c56

the more leaves you prove together, the more their paths overlap.
rollups batching withdrawals rely on exactly this (lesson 67).
```

---

## 16. A sparse tree that proves absence

`🔴 hard` · *Variants*

A sparse Merkle tree over the full 2²⁵⁶ key space. You never build it — you precompute the hash of an empty subtree at each depth and prune everything untouched. The payoff is the thing an ordinary tree cannot do: **prove absence**.

**Steps:**

1. Precompute the default hash for an empty subtree at all 256 depths.
2. Compute the root by recursing over the key bits, short-circuiting empty subtrees.
3. Prove a key is present, then prove a different key is *absent*.
4. Count how many of the 256 proof hashes are defaults — 253 of them, which is why these compress.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"sort"
)

// A sparse Merkle tree over the full 256-bit key space. It has 2^256 leaves,
// almost all of them empty — so you never build it. Instead you precompute the
// hash of an empty subtree at every depth and prune anything untouched.
//
// The payoff: it can prove ABSENCE. An ordinary Merkle tree cannot say anything
// about a leaf that is not in it.

const depth = 256

var defaults [depth + 1][32]byte

func init() {
	defaults[0] = sha256.Sum256(nil) // the empty leaf
	for i := 1; i <= depth; i++ {
		defaults[i] = sha256.Sum256(append(append([]byte{}, defaults[i-1][:]...), defaults[i-1][:]...))
	}
}

func leafHash(v []byte) [32]byte {
	if len(v) == 0 {
		return defaults[0]
	}
	return sha256.Sum256(v)
}

func nodeHash(l, r [32]byte) [32]byte {
	return sha256.Sum256(append(append([]byte{}, l[:]...), r[:]...))
}

// bit reports bit i of the key, counting from the most significant.
func bit(key [32]byte, i int) int { return int(key[i/8]>>(7-uint(i%8))) & 1 }

type SMT struct{ leaves map[[32]byte][]byte }

func NewSMT() *SMT { return &SMT{leaves: map[[32]byte][]byte{}} }

func (s *SMT) Set(key [32]byte, value []byte) { s.leaves[key] = value }

func (s *SMT) sortedKeys() [][32]byte {
	keys := make([][32]byte, 0, len(s.leaves))
	for k := range s.leaves {
		keys = append(keys, k)
	}
	sort.Slice(keys, func(i, j int) bool {
		for b := 0; b < 32; b++ {
			if keys[i][b] != keys[j][b] {
				return keys[i][b] < keys[j][b]
			}
		}
		return false
	})
	return keys
}

// compute returns the hash of the subtree at the given depth covering `keys`,
// which all share the first `d` bits. Empty subtrees short-circuit to a
// precomputed constant, which is what makes 2^256 leaves tractable.
func (s *SMT) compute(keys [][32]byte, d int) [32]byte {
	if len(keys) == 0 {
		return defaults[depth-d]
	}
	if d == depth {
		return leafHash(s.leaves[keys[0]])
	}
	split := sort.Search(len(keys), func(i int) bool { return bit(keys[i], d) == 1 })
	return nodeHash(s.compute(keys[:split], d+1), s.compute(keys[split:], d+1))
}

func (s *SMT) Root() [32]byte { return s.compute(s.sortedKeys(), 0) }

// Proof returns one sibling per level — 256 of them, but most are the default
// constants, so they compress extremely well in practice.
func (s *SMT) Proof(key [32]byte) [][32]byte {
	keys := s.sortedKeys()
	proof := make([][32]byte, depth)
	for d := 0; d < depth; d++ {
		split := sort.Search(len(keys), func(i int) bool { return bit(keys[i], d) == 1 })
		left, right := keys[:split], keys[split:]
		if bit(key, d) == 0 {
			proof[d] = s.compute(right, d+1)
			keys = left
		} else {
			proof[d] = s.compute(left, d+1)
			keys = right
		}
	}
	return proof
}

// Verify checks a key/value pair — and an EMPTY value proves absence.
func Verify(key [32]byte, value []byte, proof [][32]byte, root [32]byte) bool {
	h := leafHash(value)
	for d := depth - 1; d >= 0; d-- {
		if bit(key, d) == 0 {
			h = nodeHash(h, proof[d])
		} else {
			h = nodeHash(proof[d], h)
		}
	}
	return h == root
}

func key(s string) [32]byte { return sha256.Sum256([]byte(s)) }

func main() {
	s := NewSMT()
	s.Set(key("alice"), []byte("100"))
	s.Set(key("bob"), []byte("50"))
	s.Set(key("carol"), []byte("25"))

	root := s.Root()
	fmt.Printf("3 entries in a 2^256-leaf tree\n")
	fmt.Printf("root %s\n", hex.EncodeToString(root[:]))

	// Proof of PRESENCE.
	p := s.Proof(key("alice"))
	fmt.Printf("\nalice = 100 : %v\n", Verify(key("alice"), []byte("100"), p, root))
	fmt.Printf("alice = 999 : %v   <- wrong value\n", Verify(key("alice"), []byte("999"), p, root))

	// Proof of ABSENCE — the thing an ordinary Merkle tree cannot do.
	pd := s.Proof(key("dave"))
	fmt.Printf("\ndave is ABSENT: %v\n", Verify(key("dave"), nil, pd, root))
	fmt.Printf("  proving dave has a value: %v\n", Verify(key("dave"), []byte("1"), pd, root))

	// The proof is 256 hashes, but nearly all are default constants.
	defaultCount := 0
	for d, h := range pd {
		if h == defaults[depth-1-d] {
			defaultCount++
		}
	}
	fmt.Printf("\nproof length %d hashes (%d bytes uncompressed)\n", len(pd), len(pd)*32)
	fmt.Printf("  of which default/empty: %d\n", defaultCount)
	fmt.Printf("  genuinely distinct    : %d\n", len(pd)-defaultCount)
	fmt.Println("  real implementations send a bitmap plus the non-default hashes")

	// Updating one key changes the root, and nothing else needs rebuilding.
	s.Set(key("bob"), []byte("75"))
	fmt.Printf("\nafter bob 50 -> 75, root changes: %v\n", s.Root() != root)

	fmt.Println("\nwhere this is used: rollup state, account/nullifier sets, and")
	fmt.Println("anywhere you must prove something is NOT there (lessons 17, 30).")
}
```

**Output:**

```
3 entries in a 2^256-leaf tree
root a40762bd1bfba9ad8af7241e3ec670bba1a7f3f4991a531a07287f0ebb68bc8e

alice = 100 : true
alice = 999 : false   <- wrong value

dave is ABSENT: true
  proving dave has a value: false

proof length 256 hashes (8192 bytes uncompressed)
  of which default/empty: 253
  genuinely distinct    : 3
  real implementations send a bitmap plus the non-default hashes

after bob 50 -> 75, root changes: true

where this is used: rollup state, account/nullifier sets, and
anywhere you must prove something is NOT there (lessons 17, 30).
```

---

## 17. SPV: inclusion from an 80-byte header

`🔴 hard` · *In production*

Simplified Payment Verification, with Bitcoin's real rules. A light client holds only the 80-byte header, reads the Merkle root straight out of bytes 36–68, and verifies a transaction is in the block from three sibling hashes.

**Steps:**

1. Build an 8-transaction block using double SHA-256 and the duplication rule.
2. Assemble the 80-byte header and hash it.
3. Extract the Merkle root from the header and verify an inclusion proof against it.
4. Read carefully what SPV does *not* prove — validity, chain weight, or absence of a conflict.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"fmt"
)

// Bitcoin's real rules: double SHA-256, no domain tags, odd nodes duplicated,
// and hashes displayed reversed (lesson 03, example 10).
func hash256(b []byte) [32]byte {
	first := sha256.Sum256(b)
	return sha256.Sum256(first[:])
}

func reverse(b [32]byte) string {
	out := make([]byte, 32)
	for i := range b {
		out[i] = b[31-i]
	}
	return hex.EncodeToString(out)
}

func merkleRoot(ids [][32]byte) [32]byte {
	level := append([][32]byte(nil), ids...)
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			r := i + 1
			if r == len(level) {
				r = i
			}
			next = append(next, hash256(append(append([]byte{}, level[i][:]...), level[r][:]...)))
		}
		level = next
	}
	return level[0]
}

// merkleProof returns the sibling path for leaf `idx`, following the same
// duplication rule the root computation used.
func merkleProof(ids [][32]byte, idx int) [][32]byte {
	var proof [][32]byte
	level := append([][32]byte(nil), ids...)
	for len(level) > 1 {
		sib := idx ^ 1
		if sib == len(level) {
			sib = idx // the duplicated node is its own sibling
		}
		proof = append(proof, level[sib])

		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			r := i + 1
			if r == len(level) {
				r = i
			}
			next = append(next, hash256(append(append([]byte{}, level[i][:]...), level[r][:]...)))
		}
		level = next
		idx /= 2
	}
	return proof
}

func verifyProof(txid [32]byte, idx int, proof [][32]byte, root [32]byte) bool {
	h := txid
	for _, sib := range proof {
		if idx%2 == 0 {
			h = hash256(append(append([]byte{}, h[:]...), sib[:]...))
		} else {
			h = hash256(append(append([]byte{}, sib[:]...), h[:]...))
		}
		idx /= 2
	}
	return h == root
}

// buildHeader assembles the 80 bytes a light client actually downloads.
func buildHeader(prev, merkle [32]byte, time, bits, nonce uint32) []byte {
	h := make([]byte, 0, 80)
	h = binary.LittleEndian.AppendUint32(h, 1)
	h = append(h, prev[:]...)
	h = append(h, merkle[:]...)
	h = binary.LittleEndian.AppendUint32(h, time)
	h = binary.LittleEndian.AppendUint32(h, bits)
	return binary.LittleEndian.AppendUint32(h, nonce)
}

func main() {
	// A block with eight transactions. Only their ids matter here.
	var txids [][32]byte
	for i := 0; i < 8; i++ {
		txids = append(txids, hash256([]byte(fmt.Sprintf("transaction-%d", i))))
	}

	root := merkleRoot(txids)
	var prev [32]byte
	header := buildHeader(prev, root, 1231006505, 0x1d00ffff, 2083236893)
	blockHash := hash256(header)

	fmt.Printf("block with %d transactions\n", len(txids))
	fmt.Printf("  header      %d bytes\n", len(header))
	fmt.Printf("  merkle root %s\n", reverse(root))
	fmt.Printf("  block hash  %s\n", reverse(blockHash))

	// --- what a full node has vs what a light client has --------------------
	fmt.Println("\na full node stores every transaction. A light client (SPV) stores")
	fmt.Println("only the 80-byte header — and can still verify inclusion.")

	const idx = 5
	proof := merkleProof(txids, idx)

	fmt.Printf("\nSPV client wants to confirm transaction %d\n", idx)
	fmt.Printf("  it holds  : the 80-byte header (%d bytes)\n", len(header))
	fmt.Printf("  it is sent: the txid + %d sibling hashes (%d bytes)\n",
		len(proof), 32+len(proof)*32)
	fmt.Printf("  it does NOT receive the other %d transactions\n", len(txids)-1)

	// Extract the merkle root straight out of the header it already trusts.
	var fromHeader [32]byte
	copy(fromHeader[:], header[36:68])
	fmt.Printf("\n  merkle root read from header bytes 36..68: %v\n", fromHeader == root)
	fmt.Printf("  proof verifies against it              : %v\n",
		verifyProof(txids[idx], idx, proof, fromHeader))

	// A transaction that is not in the block cannot be made to verify.
	fake := hash256([]byte("transaction-999"))
	fmt.Printf("  a transaction not in the block         : %v\n",
		verifyProof(fake, idx, proof, fromHeader))

	fmt.Println("\nwhat SPV proves: this transaction is in a block with this header.")
	fmt.Println("what it does NOT prove:")
	fmt.Println("  - that the transaction is valid (no signature or script was checked)")
	fmt.Println("  - that this header is on the strongest chain (that needs the header chain)")
	fmt.Println("  - that no conflicting spend exists elsewhere")
	fmt.Println("\nlesson 64 builds the full light client on top of this.")
}
```

**Output:**

```
block with 8 transactions
  header      80 bytes
  merkle root 8c57dc69f85406600e2a873f7b26b298595f0faa9c58bf826c9a3bd0e5625077
  block hash  6cec2b01701c07fd473ac8288f02bc602a0f829efdfff0d44473b870c9bc208d

a full node stores every transaction. A light client (SPV) stores
only the 80-byte header — and can still verify inclusion.

SPV client wants to confirm transaction 5
  it holds  : the 80-byte header (80 bytes)
  it is sent: the txid + 3 sibling hashes (128 bytes)
  it does NOT receive the other 7 transactions

  merkle root read from header bytes 36..68: true
  proof verifies against it              : true
  a transaction not in the block         : false

what SPV proves: this transaction is in a block with this header.
what it does NOT prove:
  - that the transaction is valid (no signature or script was checked)
  - that this header is on the strongest chain (that needs the header chain)
  - that no conflicting spend exists elsewhere

lesson 64 builds the full light client on top of this.
```

---

## 18. A Merkle Mountain Range

`🔴 hard` · *Variants*

An append-only structure: a list of perfect binary trees of decreasing height. Appending merges only the newest peaks and never rewrites existing nodes, so old subtree hashes stay valid forever — exactly what a log that only grows needs.

**Steps:**

1. Append eight entries and watch the peak heights track the binary representation of the count.
2. Confirm the number of peaks always equals the popcount of the leaf count.
3. Measure the merge cost over 1024 appends — amortised O(1).
4. Understand why a plain Merkle tree cannot do this.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"math/bits"
)

// A Merkle Mountain Range is an append-only structure: a list of perfect binary
// trees ("peaks") of strictly decreasing height. Appending never rewrites the
// existing nodes — it only merges the newest ones, which is what makes it right
// for logs that grow forever.

const (
	tagLeaf byte = 0x00
	tagNode byte = 0x01
)

func leafHash(d []byte) [32]byte { return sha256.Sum256(append([]byte{tagLeaf}, d...)) }

func nodeHash(l, r [32]byte) [32]byte {
	buf := append([]byte{tagNode}, l[:]...)
	return sha256.Sum256(append(buf, r[:]...))
}

type peak struct {
	hash   [32]byte
	height int // 0 = a single leaf, 1 = 2 leaves, 2 = 4 leaves, ...
}

type MMR struct {
	peaks  []peak
	leaves int
	merges int // total merge operations performed, for the cost analysis
}

// Append adds one leaf and merges equal-height peaks. Amortised O(1) merges,
// worst case O(log n) — and it never touches anything else.
func (m *MMR) Append(data []byte) {
	p := peak{hash: leafHash(data), height: 0}
	for len(m.peaks) > 0 && m.peaks[len(m.peaks)-1].height == p.height {
		last := m.peaks[len(m.peaks)-1]
		m.peaks = m.peaks[:len(m.peaks)-1]
		p = peak{hash: nodeHash(last.hash, p.hash), height: p.height + 1}
		m.merges++
	}
	m.peaks = append(m.peaks, p)
	m.leaves++
}

// Root "bags" the peaks: fold them right to left into one commitment.
func (m *MMR) Root() [32]byte {
	if len(m.peaks) == 0 {
		return sha256.Sum256(nil)
	}
	h := m.peaks[len(m.peaks)-1].hash
	for i := len(m.peaks) - 2; i >= 0; i-- {
		h = nodeHash(m.peaks[i].hash, h)
	}
	return h
}

func (m *MMR) heights() string {
	out := ""
	for i, p := range m.peaks {
		if i > 0 {
			out += ","
		}
		out += fmt.Sprint(p.height)
	}
	return "[" + out + "]"
}

func main() {
	var m MMR
	fmt.Printf("%-8s %-6s %-14s %-18s %s\n", "leaves", "binary", "peak heights", "root", "peaks")
	for i := 1; i <= 8; i++ {
		m.Append([]byte(fmt.Sprintf("entry-%d", i)))
		r := m.Root()
		fmt.Printf("%-8d %-6b %-14s %-18s %d\n",
			m.leaves, m.leaves, m.heights(), hex.EncodeToString(r[:8]), len(m.peaks))
	}

	// The peak structure IS the binary representation of the leaf count.
	fmt.Printf("\nthe number of peaks equals the popcount of the leaf count:\n")
	for _, n := range []int{7, 8, 100, 1000} {
		var t MMR
		for i := 0; i < n; i++ {
			t.Append([]byte{byte(i)})
		}
		fmt.Printf("  %-5d leaves -> %d peaks (popcount %d) %v\n",
			n, len(t.peaks), bits.OnesCount(uint(n)), len(t.peaks) == bits.OnesCount(uint(n)))
	}

	// Cost of appending: total merges over n appends is at most n.
	var big MMR
	for i := 0; i < 1024; i++ {
		big.Append([]byte{byte(i)})
	}
	fmt.Printf("\n1024 appends performed %d merges (%.2f per append, amortised O(1))\n",
		big.merges, float64(big.merges)/1024)

	fmt.Println("\nwhy this matters:")
	fmt.Println("  a plain Merkle tree must be rebuilt when the leaf count changes shape.")
	fmt.Println("  an MMR only ever ADDS nodes, so old subtree hashes stay valid forever —")
	fmt.Println("  which is what you want for a log that never shrinks.")
	fmt.Println("\nused by: Grin/Beam, Bitcoin's Utreexo, and several rollup history commitments.")
}
```

**Output:**

```
leaves   binary peak heights   root               peaks
1        1      [0]            e868811a482c27d5   1
2        10     [1]            f0ba1dc15b0cb02f   1
3        11     [1,0]          5ab6417960994ac0   2
4        100    [2]            c13c4e5bb8cd6caa   1
5        101    [2,0]          4b0fc532e27b9a25   2
6        110    [2,1]          a690b3eddb9d7f6a   2
7        111    [2,1,0]        184490b19b9bf474   3
8        1000   [3]            9528ab773622638d   1

the number of peaks equals the popcount of the leaf count:
  7     leaves -> 3 peaks (popcount 3) true
  8     leaves -> 1 peaks (popcount 1) true
  100   leaves -> 3 peaks (popcount 3) true
  1000  leaves -> 6 peaks (popcount 6) true

1024 appends performed 1023 merges (1.00 per append, amortised O(1))

why this matters:
  a plain Merkle tree must be rebuilt when the leaf count changes shape.
  an MMR only ever ADDS nodes, so old subtree hashes stay valid forever —
  which is what you want for a log that never shrinks.

used by: Grin/Beam, Bitcoin's Utreexo, and several rollup history commitments.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
