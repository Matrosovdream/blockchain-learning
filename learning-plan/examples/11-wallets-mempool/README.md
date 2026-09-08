# Step 11 — Wallets, Fees & the Mempool · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                                # first time only
go get github.com/ethereum/go-ethereum@latest      # secp256k1 sign/verify
go get golang.org/x/crypto@latest                  # scrypt, RIPEMD-160
go get github.com/btcsuite/btcd@v0.24.2            # pin: v0.26+ split into /v2 modules
go get github.com/btcsuite/btcd/btcutil@latest     # BIP-32, example 1 only
go get github.com/tyler-smith/go-bip39@latest      # example 1 only
# paste the example into main.go, then:
go run .
```

No chain and no node. Every key comes from the **published Hardhat/anvil test mnemonic**, every
simulation is seeded, and nothing is timed — so all output reproduces exactly. Examples 3 and 11–17
need no dependencies at all.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | what a wallet is, the keystore, fee rate, size estimation, the pool |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | building a spend, the circularity, preflight, admission, policy, estimation, assembly, eviction |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | RBF, pinning, CPFP, concurrency, assembly |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. What a wallet actually holds](1-easy.md#1-what-a-wallet-actually-holds)
- [2. A key on disk](1-easy.md#2-a-key-on-disk)
- [3. Fee rate, not fee](1-easy.md#3-fee-rate-not-fee)
- [4. Estimating a size that does not exist yet](1-easy.md#4-estimating-a-size-that-does-not-exist-yet)
- [5. A mempool, ordered by what a miner wants](1-easy.md#5-a-mempool-ordered-by-what-a-miner-wants)

### 🟡 [Medium](2-medium.md)

- [6. Building a spend, end to end](2-medium.md#6-building-a-spend-end-to-end)
- [7. The fee/size circularity](2-medium.md#7-the-feesize-circularity)
- [8. Checking your own work before broadcast](2-medium.md#8-checking-your-own-work-before-broadcast)
- [9. Mempool admission](2-medium.md#9-mempool-admission)
- [10. Policy is not consensus](2-medium.md#10-policy-is-not-consensus)
- [11. Estimating a fee from history](2-medium.md#11-estimating-a-fee-from-history)
- [12. Filling a block](2-medium.md#12-filling-a-block)
- [13. Eviction, the floor, and expiry](2-medium.md#13-eviction-the-floor-and-expiry)

### 🔴 [Hard](3-hard.md)

- [14. Replace-by-fee](3-hard.md#14-replace-by-fee)
- [15. Transaction pinning](3-hard.md#15-transaction-pinning)
- [16. Child-pays-for-parent](3-hard.md#16-child-pays-for-parent)
- [17. One pool, many goroutines](3-hard.md#17-one-pool-many-goroutines)
- [18. A wallet, a mempool and a miner](3-hard.md#18-a-wallet-a-mempool-and-a-miner)

## Three that change how you read a fee

**[7. The fee/size circularity](2-medium.md#7-the-feesize-circularity)** runs the naive loop on a
wallet of dust: 47 rounds, 134 inputs, 382,000 sat in fees to move 20,000. Effective values find a
better answer in a single pass, and the loop disappears entirely.

**[11. Estimating a fee from history](2-medium.md#11-estimating-a-fee-from-history)** builds Bitcoin
Core's bucketed estimator, gets sensible numbers out of 600 blocks, then triples demand and watches
every one of them fail. An estimate is a prediction about other people.

**[15. Transaction pinning](3-hard.md#15-transaction-pinning)** turns BIP-125 rule 3 into a theft
primitive: a 99 kB child nobody will ever mine raises the cost of replacing its parent from 642 sat
to 99,642. Then TRUC brings it back down to 1,642.

## The arc

1–2 — what a wallet is, and how to put a key somewhere it survives.  
3–4 — the only number that matters, and the size you have to guess before it exists.  
5 — the waiting room, and the two indexes it needs.  
6–8 — building a spend, breaking the circularity, and checking your own work.  
9–10 — what a node will accept, and the gap between 'invalid' and 'ignored'.  
11–13 — predicting a fee, filling a block, and keeping the pool from eating the machine.  
14–16 — fixing a fee after the fact, and what a counterparty can do about that.  
17 — the concurrency problem the whole thing actually is.  
18 — a wallet, a pool and a miner around lesson 10's chain.

---

*Lesson: [../../11-wallets-mempool.md](../../11-wallets-mempool.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
