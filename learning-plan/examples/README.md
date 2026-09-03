# Examples

One folder per lesson: `NN-title/`, matching the lesson file next door. Each folder is a small
library of **complete, runnable Go programs**, split into three files by difficulty.

```
NN-title/
├── README.md     index of every example + how to run them
├── 1-easy.md     🟢 one concept at a time
├── 2-medium.md   🟡 concepts combined, and the traps
├── 3-hard.md     🔴 real-shaped programs using the whole lesson
└── PROGRESS.md   a checkbox per example
```

## Running one

Every example is a full `package main` program. **Retype it** — that is where the learning happens —
then run it:

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch          # first time only
go get github.com/ethereum/go-ethereum@latest   # when the example needs it
# paste the example into main.go, then:
go run .
```

Examples that need a chain use an in-process `ethclient/simulated` backend or a local
`anvil` — never mainnet, never a real private key.

## Starting a new lesson's examples

Copy the template:

```bash
cp -r _template 09-proof-of-work
```

Then fill in the tier files, keeping the numbering continuous across all three (1 → N) and
adding each entry to that folder's `README.md` index and `PROGRESS.md`.

Every example must be `go build`-, `go vet`- and `gofmt`-clean, and **run before it is added** —
the **Output** block under each one is real stdout, not a guess.

---
*Each lesson's example seeds and tier targets are in its part spec under [../plan/](../plan/); the rules are in [../PLAN.md](../PLAN.md).*
