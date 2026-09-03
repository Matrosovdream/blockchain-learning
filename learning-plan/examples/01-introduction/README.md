# Step 01 — Introduction to Blockchain · Examples

A library of **12 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch          # first time only
# type the example into main.go, then:
go run .
```

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout. These twelve need nothing but the standard library: no node, no keys,
no network.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | the ledger as a log, the double-spend, hash linking |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–9 | ordering, apply-time validity, determinism, coordination cost |
| 🔴 Hard | [3-hard.md](3-hard.md) | 10–12 | trust models, divergence, and the ordering rule that fixes it |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. The ledger is the log](1-easy.md#1-the-ledger-is-the-log)
- [2. Balances are derived, not stored](1-easy.md#2-balances-are-derived-not-stored)
- [3. The double-spend](1-easy.md#3-the-double-spend)
- [4. Hash-linking records](1-easy.md#4-hash-linking-records)
- [5. Tampering breaks every record after it](1-easy.md#5-tampering-breaks-every-record-after-it)

### 🟡 [Medium](2-medium.md)

- [6. Order decides the outcome](2-medium.md#6-order-decides-the-outcome)
- [7. Submit-time checks are advice, apply-time checks are rules](2-medium.md#7-submit-time-checks-are-advice-apply-time-checks-are-rules)
- [8. Determinism: map order breaks agreement](2-medium.md#8-determinism-map-order-breaks-agreement)
- [9. The cost of agreement](2-medium.md#9-the-cost-of-agreement)

### 🔴 [Hard](3-hard.md)

- [10. Trusted server vs replicas under a partition](3-hard.md#10-trusted-server-vs-replicas-under-a-partition)
- [11. Five nodes, two spends, no agreement](3-hard.md#11-five-nodes-two-spends-no-agreement)
- [12. One shared ordering rule, five identical states](3-hard.md#12-one-shared-ordering-rule-five-identical-states)

## The arc

The twelve build one argument, in order:

1–2 — a ledger is an ordered log; balances are derived from it.  
3 — but a naive log lets the same coin be spent twice.  
4–5 — hashing chains the records so tampering is *detectable*…  
6–7 — …yet integrity says nothing about which of two conflicting spends is valid; order does.  
8–9 — agreement needs determinism, and agreement costs O(n²) messages.  
10–11 — so replicas without a rule diverge permanently under a partition.  
12 — a shared ordering rule makes five independent nodes agree with nobody in charge.

That last step is consensus, and it is the only genuinely new part of the 2009 design.

---

*Lesson: [../../01-introduction.md](../../01-introduction.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
