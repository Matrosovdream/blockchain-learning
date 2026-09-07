# Step 09 — Proof of Work & Mining · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch       # first time only
# paste the example into main.go, then:
go run .
```

**Standard library only.** Simulations use a seeded RNG and mining reports *hash counts* rather than
elapsed time, so every output reproduces exactly.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | the puzzle, verification, targets, difficulty |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | the mining loop, retargeting, timewarp, statistics, economics |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | parallel mining, leaks, 51%, selfish mining, assembly |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. The puzzle](1-easy.md#1-the-puzzle)
- [2. Verification is one hash](1-easy.md#2-verification-is-one-hash)
- [3. Compact bits to a 256-bit target](1-easy.md#3-compact-bits-to-a-256-bit-target)
- [4. Each bit doubles the work](1-easy.md#4-each-bit-doubles-the-work)
- [5. Leading zeros is a picture, not the rule](1-easy.md#5-leading-zeros-is-a-picture-not-the-rule)

### 🟡 [Medium](2-medium.md)

- [6. Target back to bits, canonically](2-medium.md#6-target-back-to-bits-canonically)
- [7. The allocation-free mining loop](2-medium.md#7-the-allocation-free-mining-loop)
- [8. When 4 billion nonces are not enough](2-medium.md#8-when-4-billion-nonces-are-not-enough)
- [9. Retargeting, integer-only](2-medium.md#9-retargeting-integer-only)
- [10. The timewarp attack](2-medium.md#10-the-timewarp-attack)
- [11. Block times are exponential](2-medium.md#11-block-times-are-exponential)
- [12. Confirmations, not minutes](2-medium.md#12-confirmations-not-minutes)
- [13. Hashrate, difficulty and the security budget](2-medium.md#13-hashrate-difficulty-and-the-security-budget)

### 🔴 [Hard](3-hard.md)

- [14. Parallel mining](3-hard.md#14-parallel-mining)
- [15. The goroutine leak](3-hard.md#15-the-goroutine-leak)
- [16. Simulating a 51% double-spend](3-hard.md#16-simulating-a-51-double-spend)
- [17. Selfish mining](3-hard.md#17-selfish-mining)
- [18. Proof of work in the chain](3-hard.md#18-proof-of-work-in-the-chain)

## Two that reproduce published results

**[12. Confirmations, not minutes](2-medium.md#12-confirmations-not-minutes)** implements the
Bitcoin whitepaper's section 11 calculation. The q=10%, z=6 cell comes out at 2.43e-04 —
the figure Satoshi published.

**[17. Selfish mining](3-hard.md#17-selfish-mining)** implements Eyal & Sirer's SM1 state
machine and reproduces their thresholds: profitable above **1/3** hashrate when the attacker
loses every tie, above **1/4** when they win half, and at almost any share when they win them all.

## The arc

1–2 — the puzzle, and the asymmetry that makes it verifiable by everyone.  
3–6 — targets, the compact encoding, and why counting zeros is not the rule.  
7–8 — making the loop fast, and what happens when 2³² nonces run out.  
9–10 — keeping block times on target, and the attack on the timestamps that drive it.  
11–13 — the statistics nobody expects, and what security actually costs.  
14–15 — mining across cores, and the two concurrency bugs that come with it.  
16–17 — the two attacks that define the safety thresholds.  
18 — all of it folded into lesson 08's chain, in about forty lines of diff.

---

*Lesson: [../../09-proof-of-work.md](../../09-proof-of-work.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
