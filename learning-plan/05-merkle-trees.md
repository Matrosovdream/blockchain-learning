# 05 — Merkle Trees & Proofs

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-02-cryptography-foundations.md](plan/part-02-cryptography-foundations.md#05-merkle-trees-proofs)

| | |
|---|---|
| **Part** | Part 2 — Cryptography Foundations |
| **Prerequisites** | [04](04-hash-functions.md) |
| **Unlocks** | 08, 17, 39 |
| **Examples** | [18](examples/05-merkle-trees/) (🟢 5 · 🟡 8 · 🔴 5) |

*Building a tree, the root as a commitment, inclusion proofs, and the traps (odd leaves, second preimage).*

A Merkle tree is one idea — hash things in pairs until one hash is left — with a surprising number of
ways to get it wrong. The construction takes ten minutes to learn. The traps in this lesson are what
separate a working implementation from one that quietly accepts forged proofs.

## Goals

- Build a Merkle tree over a list of items in Go.
- Generate and verify an inclusion proof without holding the whole list.
- Explain why a block header only needs one 32-byte root.
- Know the odd-leaf duplication bug and the second-preimage defence.

## Concepts

### 1. The problem

You have a million transactions. You want to convince someone that one specific transaction is among
them, and you want the proof to be small.

Hashing everything together is a perfectly good *commitment* — change any item and the digest moves.
But it proves nothing about a single element, because the only way to check it is to recompute it,
which means receiving all million items. Example
[4](examples/05-merkle-trees/1-easy.md#4-why-not-just-hash-everything-together) demonstrates the
commitment working and then the proof failing to exist.

What you actually want is three things at once:

| | Cost |
|---|---|
| **Commitment** | O(1) — one 32-byte root, small enough for a block header |
| **Proof** | O(log n) — 20 hashes for a million leaves |
| **Construction** | O(n) — build it once per block |

That is a Merkle tree, and it is why the design shows up everywhere: Bitcoin and Ethereum block
headers, light clients ([64](64-light-clients-spv.md)), airdrop allowlists, rollup withdrawal proofs
([67](67-bridges-cross-chain.md)), and Certificate Transparency logs outside this field entirely.

### 2. Construction

Hash every leaf. Pair adjacent hashes and hash each pair. Repeat until one node remains:

```go
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
root := level[0]
```

The `leafHash`/`nodeHash` pair is exactly the one from lesson 04's example 16 — `0x00` for leaves and
`0x01` for internal nodes. Topic 5 explains why that byte is load-bearing. Example
[1](examples/05-merkle-trees/1-easy.md#1-building-a-tree-level-by-level) prints every level, and the
root it produces is the same one lesson 04 printed.

**Representation.** Two options. Keep a `[][]byte` per level — simple, easy to generate proofs from,
and what every example here does. Or use a flat array with index arithmetic, which is more compact
and what performance-sensitive implementations use. Build bottom-up rather than recursing top-down:
the levels are exactly what a proof needs.

**The root commits to content *and* order.** Example
[2](examples/05-merkle-trees/1-easy.md#2-the-root-commits-to-content-and-order) edits one leaf and
swaps two, and both move the root. That is why a block header pins down its entire body with 32
bytes.

It is equally important to know what the root does **not** commit to: how many leaves there are
(unless you put the count in a leaf), whether they are unique, or anything at all about items that
are absent. Topic 3 and example 8 come back to this.

### 3. Inclusion proofs

A proof is one sibling hash per level, plus which side each sibling is on:

```go
type Step struct {
    Hash    [32]byte
    IsRight bool
}
```

Generating it is index arithmetic. `i ^ 1` flips the last bit to give you the sibling; `i /= 2` walks
up a level:

```go
func (t *Tree) Proof(index int) []Step {
    var proof []Step
    for lvl := 0; lvl < len(t.levels)-1; lvl++ {
        sibling := index ^ 1
        proof = append(proof, Step{
            Hash:    t.levels[lvl][sibling],
            IsRight: sibling > index,
        })
        index /= 2
    }
    return proof
}
```

Verification is a fold from the leaf up, and it needs no tree at all — twelve lines and four
variables:

```go
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
```

Example [6](examples/05-merkle-trees/2-medium.md#6-a-proof-for-one-leaf-of-eight) proves leaf 2 of 8
with three hashes; the verifier never sees the other seven transactions. Example
[7](examples/05-merkle-trees/2-medium.md#7-six-ways-to-break-a-proof) tries six tamperings — a
flipped bit, a wrong direction, a substituted leaf, a truncated proof, reordered steps, and a
structurally valid proof from a different tree — and all six fail.

**The numbers.** Depth is ⌈log₂ n⌉, so a proof is that many hashes:

| Leaves | Depth | Proof |
|---|---|---|
| 1,024 | 10 | 320 bytes |
| 1,048,576 | 20 | 640 bytes |
| 16,777,216 | 24 | 768 bytes |

Doubling the set adds exactly one hash. Example
[5](examples/05-merkle-trees/1-easy.md#5-what-a-proof-costs) works it out: for a million 250-byte
transactions, a proof is 640 bytes instead of 250 MiB.

**Now the part that matters most.** A valid proof proves exactly one thing:

> This leaf is somewhere in the tree with this root.

It does **not** prove the leaf is unique — example
[8](examples/05-merkle-trees/2-medium.md#8-what-a-proof-does-not-prove) builds a tree containing the
same leaf twice and produces two valid proofs for it. It does not prove *where* the leaf is, unless
the index is inside the leaf. It says nothing about absence (topic 7). And critically, **it does not
prove the root is the right root** — the example verifies a proof against an attacker-chosen root and
it passes perfectly.

So the root must come from somewhere you trust: a block header, a signed message, a contract's
storage. Never from the prover. For an airdrop this has a direct consequence — one valid proof is
replayable forever unless you record claims on-chain.

### 4. Odd numbers of leaves

A level with an odd number of nodes cannot be paired evenly. There are three common answers, and
example [9](examples/05-merkle-trees/2-medium.md#9-odd-leaves-three-conventions) implements all three:

- **Duplicate** — hash the last node with itself. Bitcoin does this.
- **Promote** — carry the odd node up a level unchanged. Most libraries do this.
- **Pad** — extend the level to a power of two with a fixed sentinel. Needed when depth must be
  constant, as in ZK circuits ([39](39-zero-knowledge-proofs.md)) and sparse trees.

All three are internally consistent, and all three produce **different roots**. A proof generated
under one will never verify under another. Worse, they *agree* when the leaf count is a power of two
— so a test with four leaves passes under every convention and tells you nothing.

Bitcoin's choice turned out to be exploitable. Because an odd node is hashed with itself, a block
with transactions `[a b c]` and one with `[a b c c]` produce the **identical** Merkle root, and
therefore the identical block hash. Example
[10](examples/05-merkle-trees/2-medium.md#10-cve-2012-2459-two-blocks-one-root) reproduces it:

```
block with [a b c]     root 8b1d539cf3a142f7...
block with [a b c c]   root 8b1d539cf3a142f7...
IDENTICAL ROOTS: true
```

That is **CVE-2012-2459**. An attacker takes a valid block, duplicates its last transaction, and
broadcasts it. The mutated block has the same hash, so nodes believe they have already seen it — but
it is invalid, because a duplicated transaction is a double-spend. Nodes that cached the failure
would then reject the honest block too. A network-splitting denial of service, from a padding rule.

Bitcoin Core fixed it by detecting the duplication explicitly rather than changing the tree, because
the rule is consensus-critical and permanent. If you are designing something new: **promote or pad,
and document which**.

### 5. Second-preimage resistance

Here is the attack that makes domain separation non-negotiable.

In a naive tree, `leafHash(x) = H(x)` and `nodeHash(l, r) = H(l‖r)`. An internal node's preimage is
64 bytes — two concatenated child hashes. But a *leaf* can also be 64 bytes. So construct a leaf
whose contents are exactly `hA‖hB`:

```go
forgedLeaf := append(hA[:], hB[:]...)   // 64 bytes
leafHash(forgedLeaf) == nodeHash(hA, hB)  // true — they are the same preimage
```

That forged leaf now verifies against the honest root with a one-step proof. Example
[11](examples/05-merkle-trees/2-medium.md#11-the-second-preimage-attack) does it. You have proven
membership of data that was never in the tree — a **second preimage** for the root, since
`{hA‖hB, hC‖hD}` and `{tx-a, tx-b, tx-c, tx-d}` produce the same commitment.

The defence is one byte per hash, and it comes from RFC 6962 (Certificate Transparency):

```go
func leafHash(d []byte) [32]byte { return H(append([]byte{0x00}, d...)) }
func nodeHash(l, r [32]byte) [32]byte { return H(append(append([]byte{0x01}, l[:]...), r[:]...)) }
```

Now the two preimages start with different bytes and can never coincide. Example
[12](examples/05-merkle-trees/2-medium.md#12-one-byte-of-domain-separation) shows the same forgery
attempt failing.

Plenty of libraries omit the tags and are, in practice, fine — because their leaves happen to always
be 32 bytes, so a 64-byte leaf is unrepresentable. **That is a property of the caller, not of the
tree.** It is exactly the kind of assumption that holds until someone adds a feature. Tag your
hashes.

### 6. Sorted-pair trees

OpenZeppelin's `MerkleProof.sol` uses a different convention: sort each pair before hashing.

```go
func hashPair(a, b [32]byte) [32]byte {
    if bytes.Compare(a[:], b[:]) > 0 {
        a, b = b, a
    }
    return keccak256(append(a[:], b[:]...))
}
```

Because the order is derived from the values, **a proof needs no direction bits** — it is a bare list
of hashes. That makes the on-chain verifier simpler and slightly cheaper, since it never branches.

The trade is that the tree no longer commits to which side a leaf is on, so it can prove membership
but not position. For an allowlist that is exactly what you want; for anything index-dependent it is
not. Example [13](examples/05-merkle-trees/2-medium.md#13-sorted-pairs-no-direction-bits) shows both
sides of the trade.

Sorted pairs also reopen the topic-5 hole: a proof element that happens to equal a real internal node
can let a 64-byte leaf impersonate one. OpenZeppelin's answer is to **hash leaves twice**, so a leaf
digest is a hash *of* a hash and can never be read as two concatenated children:

```solidity
bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
```

Example [14](examples/05-merkle-trees/3-hard.md#14-an-openzeppelin-compatible-airdrop) builds a
complete allowlist to those rules and verifies it with a faithful Go port of OpenZeppelin's `verify`
loop. Before shipping one, also check your root against the real contract with `forge`/`cast`
([02](02-environment-setup.md)) — a Go reimplementation agreeing with itself is not the same as
agreeing with Solidity.

> **Never mix conventions.** A sorted-pair proof will not verify against a directional verifier, and
> vice versa. They are different trees with different roots. Pick one per system and write it down.

### 7. Variants you will meet

**Merkle Patricia Trie** — key/value rather than a list, so you can look up by key and prove what a
key maps to. This is what Ethereum actually uses for state, and [17](17-rlp-merkle-patricia-trie.md)
covers it in full.

**Sparse Merkle trees** — a tree over the entire 2²⁵⁶ key space, almost all of it empty. You never
build it: you precompute the hash of an empty subtree at each depth and prune anything untouched.
The payoff is **proofs of absence**, which an ordinary Merkle tree cannot provide at all. Example
[16](examples/05-merkle-trees/3-hard.md#16-a-sparse-tree-that-proves-absence) implements one over
full 256-bit keys; a proof is 256 hashes of which 253 are default constants, which is why they
compress so well. Used for rollup state and nullifier sets ([30](30-layer2-scaling.md)).

**Merkle Mountain Ranges** — an append-only structure: a list of perfect binary trees of decreasing
height. Appending merges only the newest peaks and never rewrites existing nodes, so old subtree
hashes stay valid forever. Example
[18](examples/05-merkle-trees/3-hard.md#18-a-merkle-mountain-range) shows the peak heights tracking
the binary representation of the leaf count, and measures the merge cost at amortised O(1).

**Verkle trees** — vector commitments instead of hashes, giving roughly constant-size proofs instead
of O(log n). This is what would make stateless Ethereum practical; [66](66-state-growth-verkle.md)
covers where that stands.

### 8. Merkle proofs in production

**Airdrops and allowlists** are the most common real use, and the pattern is worth memorising:
compute the root off-chain, deploy it as a constant, and have claimants submit `(amount, proof)`.
The contract recomputes the leaf from `msg.sender` — never from user input — verifies, and records
the claim. Example 14 builds the off-chain half; example 8 explains why the claim record is not
optional.

**SPV** is where the design started. A light client stores only 80-byte headers, reads the Merkle
root out of bytes 36–68, and verifies transaction inclusion from a handful of sibling hashes.
Example [17](examples/05-merkle-trees/3-hard.md#17-spv-inclusion-from-an-80-byte-header) does it with
Bitcoin's real rules. Be precise about what it establishes: this transaction is in a block with this
header. It does not check the transaction's validity, does not prove the header is on the strongest
chain, and cannot rule out a conflicting spend elsewhere. [64](64-light-clients-spv.md) builds the
rest.

**Rollup withdrawals and L1↔L2 messaging** are Merkle proofs against a state root posted to L1
([67](67-bridges-cross-chain.md)). Same mechanism, much higher stakes.

**Multiproofs** prove several leaves at once, sharing the parts of their paths that overlap. Example
[15](examples/05-merkle-trees/3-hard.md#15-a-multiproof-for-three-leaves) proves three leaves of
eight with four hashes instead of nine, and the saving grows with the batch — which is why rollups
batching withdrawals use them.

## Exercises

Write these in `practice/05-merkle-trees/`.

1. **Build it yourself.** Write `Tree` with `Build`, `Root`, `Proof` and `Verify`, using domain tags.
   Table-test it for every leaf index at sizes 1, 2, 3, 4, 7 and 8.
2. **Break your own verifier.** Write tests for all six tamperings in example 7. Then check: does
   your verifier reject a proof that is *longer* than the tree's depth? Most first attempts do not.
3. **Prove the conventions differ.** Implement all three odd-leaf rules behind one interface. Assert
   they agree on 4 leaves and disagree on 5. Then generate a proof under `promote` and confirm it
   fails against a `duplicate` verifier.
4. **Reproduce the CVE.** Build Bitcoin's exact rules and find your own pair of transaction lists
   with an identical root. Then write the check Bitcoin Core added, and confirm it catches yours.
5. **Mount and fix the second-preimage attack.** Implement an untagged tree, forge a 64-byte leaf,
   and verify it against the honest root. Add tags and confirm the same forgery fails. Keep both as
   a regression test.
6. **OpenZeppelin compatibility.** Build a sorted-pair allowlist with double-hashed leaves. Write a
   tiny Solidity contract using `MerkleProof.verify`, deploy it to `anvil`, and confirm your Go
   proofs are accepted on-chain. This is the exercise that will catch a real encoding mistake.
7. **Multiproofs.** Extend your tree with `MultiProof`/`VerifyMulti`. Measure the saving for 2, 4, 8
   and 16 leaves out of 1024, and plot where it stops mattering.
8. **Absence.** Implement a sparse Merkle tree over 256-bit keys. Prove a key present, prove another
   absent, then delete a key and confirm its absence proof now verifies.

## Best Practices & Pitfalls

- **Domain-separate leaves and internal nodes.**
  *Why:* without tags, a 64-byte leaf hashes identically to an internal node, and anyone can prove
  membership of data that was never in the tree (example 11). One byte per hash.
- **Never mix sorted-pair and directional proofs.**
  *Why:* they build different trees with different roots. A proof from one silently fails against the
  other's verifier, and the failure looks like corrupt data rather than a convention mismatch.
- **Pick an odd-leaf rule, document it, and test an odd count.**
  *Why:* duplicate, promote and pad all agree on powers of two, so a 4-leaf test passes under every
  convention. Production then breaks on 5.
- **Do not copy Bitcoin's duplicate-last-hash rule into a new system.**
  *Why:* it makes `[a b c]` and `[a b c c]` share a root (CVE-2012-2459). Bitcoin keeps it because
  changing it would fork the chain; you have no such constraint.
- **The root must come from a source you trust — never from the prover.**
  *Why:* a proof verifies against *whatever root you give it*. Supply an attacker's root and the
  attacker's leaf verifies perfectly (example 8). Take the root from a block header, a contract, or a
  signed message.
- **A proof of inclusion is not a proof of uniqueness, position or absence.**
  *Why:* the same leaf can appear twice and both copies verify. If position matters, put the index in
  the leaf. If absence matters, you need a sparse tree.
- **Record claims when a proof authorises something.**
  *Why:* a valid proof stays valid forever. Without an on-chain `claimed` mapping, one airdrop proof
  can be submitted repeatedly.
- **Recompute the leaf from trusted inputs, not from what the user sent.**
  *Why:* a contract should build the leaf from `msg.sender` and the amount it verifies, not accept a
  caller-supplied leaf. Otherwise the proof authorises whatever the caller claims.
- **Bound the proof length against the tree's depth.**
  *Why:* an unbounded fold lets an attacker keep hashing until something matches, and it is a cheap
  denial-of-service vector on-chain.
- **Verify your Go implementation against the actual on-chain verifier.**
  *Why:* an encoding difference — padding, hash function, leaf format — produces a root that is
  self-consistent in Go and rejected by Solidity. Check with `cast` before you deploy.

## Checklist

- [ ] I can build a Merkle tree and compute its root in Go.
- [ ] I can generate an inclusion proof and verify it without the tree.
- [ ] I can state the proof size for a million leaves and explain why it is logarithmic.
- [ ] I can explain precisely what a valid proof does and does not prove.
- [ ] I know the three odd-leaf conventions and why they agree on powers of two.
- [ ] I can explain CVE-2012-2459 and reproduce it.
- [ ] I can mount the second-preimage attack and fix it with domain separation.
- [ ] I can build a sorted-pair tree whose proofs OpenZeppelin's verifier accepts.
- [ ] I know why OpenZeppelin hashes leaves twice.
- [ ] I can explain what a sparse Merkle tree adds, and what an MMR is for.
- [ ] I can describe what SPV proves and what it does not.

## Resources

**Specifications**

- RFC 6962 (Certificate Transparency) — the canonical domain-separated tree: https://datatracker.ietf.org/doc/html/rfc6962
- Bitcoin developer reference — block chain and Merkle trees: https://developer.bitcoin.org/reference/block_chain.html
- Bitcoin whitepaper §7–8 (reclaiming disk space, SPV): https://bitcoin.org/bitcoin.pdf

**Implementations**

- OpenZeppelin `MerkleProof`: https://docs.openzeppelin.com/contracts/5.x/api/utils#MerkleProof
- OpenZeppelin `merkle-tree` (the JS builder that pairs with it): https://github.com/OpenZeppelin/merkle-tree
- Certificate Transparency in Go (`trillian`): https://github.com/google/trillian

**Background**

- CVE-2012-2459 discussion: https://bitcointalk.org/?topic=102395
- Vitalik on sparse Merkle trees: https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/
- Merkle Mountain Ranges (Grin): https://docs.grin.mw/wiki/chain-state/merkle-mountain-range/

---

**Examples:** [`examples/05-merkle-trees/`](examples/05-merkle-trees/) — **18 runnable Go programs**
(🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Three earn special attention: example 8 verifies a proof
against an attacker-chosen root, example 10 reproduces CVE-2012-2459, and examples 11–12 mount a
second-preimage forgery and then defeat it with a single byte.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
