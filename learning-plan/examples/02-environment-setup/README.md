# Step 02 — Environment Setup & Tooling · Examples

A library of **12 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # first time only
go get github.com/ethereum/go-ethereum@latest   # needed from example 4 onward
# paste the example into main.go, then:
go run .
```

Examples 4–12 need a chain. They start one **in-process** when `RPC_URL` is unset, so they run with
no installs. Point `RPC_URL` at `anvil`, `geth --dev` or a testnet and the same code talks to that
instead.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and every one was run twice to confirm it is reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | toolchain, config, secret hygiene, connecting, deadlines |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–9 | chain state, dev accounts, the raw RPC layer, failure modes |
| 🔴 Hard | [3-hard.md](3-hard.md) | 10–12 | the reusable setup helper, capability probing, preflight |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. What am I actually running?](1-easy.md#1-what-am-i-actually-running)
- [2. Config from the environment, fail fast](1-easy.md#2-config-from-the-environment-fail-fast)
- [3. A secret that cannot be printed](1-easy.md#3-a-secret-that-cannot-be-printed)
- [4. A chain to talk to, with no installs](1-easy.md#4-a-chain-to-talk-to-with-no-installs)
- [5. Every call gets a deadline](1-easy.md#5-every-call-gets-a-deadline)

### 🟡 [Medium](2-medium.md)

- [6. Head, base fee and what gas costs](2-medium.md#6-head-base-fee-and-what-gas-costs)
- [7. The ten funded dev accounts](2-medium.md#7-the-ten-funded-dev-accounts)
- [8. Raw JSON-RPC vs ethclient](2-medium.md#8-raw-json-rpc-vs-ethclient)
- [9. The three ways connecting fails](2-medium.md#9-the-three-ways-connecting-fails)

### 🔴 [Hard](3-hard.md)

- [10. chainenv: load, dial, verify, hand back a client](3-hard.md#10-chainenv-load-dial-verify-hand-back-a-client)
- [11. Probe what the endpoint can actually do](3-hard.md#11-probe-what-the-endpoint-can-actually-do)
- [12. Preflight: the check you run before every lesson](3-hard.md#12-preflight-the-check-you-run-before-every-lesson)

## No installs required

Lesson 01's examples needed nothing but the standard library. These need a chain — so examples
4–12 share one helper that starts a **real HTTP JSON-RPC endpoint in-process** when `RPC_URL` is
unset:

```go
func endpoint() (string, func()) {
	if u := os.Getenv("RPC_URL"); u != "" {
		return u, func() {} // your anvil / geth --dev / testnet
	}
	port := freePort()
	sim := simulated.NewBackend(types.GenesisAlloc{}, func(nc *node.Config, ec *ethconfig.Config) {
		nc.HTTPHost, nc.HTTPPort = "127.0.0.1", port
		nc.HTTPModules = []string{"eth", "net", "web3"}
	})
	return fmt.Sprintf("http://127.0.0.1:%d", port), func() { sim.Close() }
}
```

It is a genuine geth node stack serving genuine JSON-RPC over HTTP — not a mock. Chain id is
**1337**, the same id anvil uses, and example 7 funds the same ten accounts anvil funds. So you can
work the whole lesson today and switch to real anvil later without changing a line.

## The arc

1–3 — know your toolchain, load config safely, and make secrets unprintable.  
4–5 — get a chain to talk to, and put a deadline on every call.  
6–8 — read the state you will read constantly, and see the raw RPC layer under `ethclient`.  
9–10 — classify the ways connecting fails, then package the whole thing into one reusable helper.  
11–12 — find out what your provider can actually do, and build the preflight check you run daily.

---

*Lesson: [../../02-environment-setup.md](../../02-environment-setup.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
