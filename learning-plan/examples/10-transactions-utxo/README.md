# Step 10 — Transactions & the UTXO Model · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                              # first time only
go get github.com/ethereum/go-ethereum@latest    # secp256k1 sign/verify
go get golang.org/x/crypto@latest                # RIPEMD-160, for HASH160
# paste the example into main.go, then:
go run .
```

No chain and no node — these programs build the transaction layer themselves. Every key is a
**published Hardhat/anvil test key**, every amount and timestamp is fixed, and nothing is timed,
so the output reproduces exactly. Examples 1, 3, 5, 14, 15 and 17 need no dependencies at all.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | accounts vs UTXO, the four types, txids, the coinbase, the subsidy |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | signing, verification, the copy bug, value, the UTXO set, double spends |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | coin selection, dust and privacy, maturity, the overflow, assembly |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. Two ledgers, one payment](1-easy.md#1-two-ledgers-one-payment)
- [2. Inputs, outputs and outpoints](1-easy.md#2-inputs-outputs-and-outpoints)
- [3. The txid is a hash of the bytes](1-easy.md#3-the-txid-is-a-hash-of-the-bytes)
- [4. The coinbase transaction](1-easy.md#4-the-coinbase-transaction)
- [5. The subsidy schedule](1-easy.md#5-the-subsidy-schedule)

### 🟡 [Medium](2-medium.md)

- [6. What exactly gets signed](2-medium.md#6-what-exactly-gets-signed)
- [7. Sign it, then try to change it](2-medium.md#7-sign-it-then-try-to-change-it)
- [8. The copy that was not a copy](2-medium.md#8-the-copy-that-was-not-a-copy)
- [9. Which input failed](2-medium.md#9-which-input-failed)
- [10. The check that makes ownership mean anything](2-medium.md#10-the-check-that-makes-ownership-mean-anything)
- [11. Conservation of value](2-medium.md#11-conservation-of-value)
- [12. Building the UTXO set](2-medium.md#12-building-the-utxo-set)
- [13. Double spends, and applying a block atomically](2-medium.md#13-double-spends-and-applying-a-block-atomically)

### 🔴 [Hard](3-hard.md)

- [14. Coin selection behind an interface](3-hard.md#14-coin-selection-behind-an-interface)
- [15. Change, dust, and what they leak](3-hard.md#15-change-dust-and-what-they-leak)
- [16. Coinbase maturity, and the cascade it prevents](3-hard.md#16-coinbase-maturity-and-the-cascade-it-prevents)
- [17. The value overflow incident](3-hard.md#17-the-value-overflow-incident)
- [18. Transactions in the chain](3-hard.md#18-transactions-in-the-chain)

## Three that reproduce real incidents

**[4. The coinbase transaction](1-easy.md#4-the-coinbase-transaction)** reproduces the duplicate-
coinbase bug: blocks 91722/91880 and 91812/91842 each contain byte-identical coinbases, and the
second overwrote the first. 100 BTC destroyed, and the reason BIP-34 puts the height in the coinbase.

**[16. Coinbase maturity](3-hard.md#16-coinbase-maturity-and-the-cascade-it-prevents)** turns the
maturity rule off and follows a coinbase through four spends to an exchange, then orphans the block
that made it. All four transactions die at once, and none of them can be re-mined.

**[17. The value overflow incident](3-hard.md#17-the-value-overflow-incident)** runs the actual
transaction from block 74638 — 0.5 BTC in, two outputs of 92,233,720,368.54277039 BTC out — through
four validators. The naive one accepts it, as Bitcoin did on 15 August 2010.

## The arc

1 — the two ownership models, and why UTXO validates in parallel.  
2–3 — the four types, and the serialization that makes a txid mean something.  
4–5 — where coins come from, and the schedule that stops them coming.  
6–8 — what a signature actually covers, and the deep-copy bug that eats an afternoon.  
9–10 — verification proper: which input failed, and the check that makes ownership real.  
11–13 — value, the fee, the set that holds the state, and the double spends it stops.  
14–15 — choosing coins, and what that choice tells a chain analyst about you.  
16–17 — the two rules that exist because of things that actually went wrong.  
18 — all of it folded into lesson 09's chain.

---

*Lesson: [../../10-transactions-utxo.md](../../10-transactions-utxo.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
