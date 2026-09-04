# Step 05 — Merkle Trees & Proofs · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get github.com/ethereum/go-ethereum@latest   # examples 13, 14
# paste the example into main.go, then:
go run .
```

No chain, no node, no keys. Everything except examples 13 and 14 needs only the standard library.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | construction, the root as a commitment, the cost model |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | proofs, odd leaves, CVE-2012-2459, second preimages, sorted pairs |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | airdrops, multiproofs, sparse trees, SPV, MMRs |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. Building a tree, level by level](1-easy.md#1-building-a-tree-level-by-level)
- [2. The root commits to content and order](1-easy.md#2-the-root-commits-to-content-and-order)
- [3. A two-leaf proof, folded by hand](1-easy.md#3-a-two-leaf-proof-folded-by-hand)
- [4. Why not just hash everything together?](1-easy.md#4-why-not-just-hash-everything-together)
- [5. What a proof costs](1-easy.md#5-what-a-proof-costs)

### 🟡 [Medium](2-medium.md)

- [6. A proof for one leaf of eight](2-medium.md#6-a-proof-for-one-leaf-of-eight)
- [7. Six ways to break a proof](2-medium.md#7-six-ways-to-break-a-proof)
- [8. What a proof does not prove](2-medium.md#8-what-a-proof-does-not-prove)
- [9. Odd leaves, three conventions](2-medium.md#9-odd-leaves-three-conventions)
- [10. CVE-2012-2459: two blocks, one root](2-medium.md#10-cve-2012-2459-two-blocks-one-root)
- [11. The second-preimage attack](2-medium.md#11-the-second-preimage-attack)
- [12. One byte of domain separation](2-medium.md#12-one-byte-of-domain-separation)
- [13. Sorted pairs, no direction bits](2-medium.md#13-sorted-pairs-no-direction-bits)

### 🔴 [Hard](3-hard.md)

- [14. An OpenZeppelin-compatible airdrop](3-hard.md#14-an-openzeppelin-compatible-airdrop)
- [15. A multiproof for three leaves](3-hard.md#15-a-multiproof-for-three-leaves)
- [16. A sparse tree that proves absence](3-hard.md#16-a-sparse-tree-that-proves-absence)
- [17. SPV: inclusion from an 80-byte header](3-hard.md#17-spv-inclusion-from-an-80-byte-header)
- [18. A Merkle Mountain Range](3-hard.md#18-a-merkle-mountain-range)

## Three that are more than demonstrations

**[8. What a proof does not prove](2-medium.md#8-what-a-proof-does-not-prove)** — the one to read
twice. A valid proof against an attacker-chosen root verifies perfectly. Where the root comes from
is the entire security of the scheme, and it is the part most integrations get wrong.

**[10. CVE-2012-2459](2-medium.md#10-cve-2012-2459-two-blocks-one-root)** — a real Bitcoin
vulnerability reproduced in thirty lines: two different transaction lists, one identical root.

**[11](2-medium.md#11-the-second-preimage-attack)** and
**[12](2-medium.md#12-one-byte-of-domain-separation)** — a working second-preimage forgery, then
the single byte that defeats it.

## The arc

1–2 — build a tree; see that the root commits to content and order.  
3–5 — what a proof is, why concatenation is not enough, and what it all costs.  
6–8 — generate and verify proofs, break them six ways, then learn their limits.  
9–10 — the odd-leaf question, and the CVE that came from answering it badly.  
11–13 — the second-preimage attack, the one-byte fix, and the sorted-pair variant.  
14–18 — the real uses: airdrops, multiproofs, absence proofs, SPV, and append-only logs.

---

*Lesson: [../../05-merkle-trees.md](../../05-merkle-trees.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
