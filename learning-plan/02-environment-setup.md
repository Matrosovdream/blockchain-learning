# 02 — Environment Setup & Tooling

> **Status:** ✅ written. Examples: 12/12 built and run.
> **Spec:** [plan/part-01-foundations.md](plan/part-01-foundations.md#02-environment-setup-tooling)

| | |
|---|---|
| **Part** | Part 1 — Foundations |
| **Prerequisites** | [01](01-introduction.md) |
| **Unlocks** | 03 |
| **Examples** | [12](examples/02-environment-setup/) (🟢 5 · 🟡 4 · 🔴 3) |

*Go toolchain, the module layout for this repo, node clients, testnets, faucets, and the CLI toolbox.*

By the end of this lesson you can dial a chain from Go, read its state, and you have a preflight
script you will run before every remaining lesson. You also establish secret hygiene now, while you
have nothing to lose — because the habits you form here are the ones you will still have when you
do.

## Goals

- Have a Go module ready to run every example in this repo.
- Install and sanity-check a chain to talk to: `anvil`, local geth, or a hosted RPC.
- Get testnet funds and read your own transaction on an explorer.
- Establish secret hygiene before you ever hold a key that matters.

## Concepts

### 1. Go setup for this repo

Check what you have:

```bash
go version    # need 1.22 or newer
go env GOPATH GOMODCACHE
```

Go 1.22+ matters because of the loop-variable change and the `net/http` routing improvements this
course assumes. Nothing in this repo is pinned globally — no `GOFLAGS`, no vendored tree, no
toolchain directive forcing a specific patch release. Each module resolves its own dependencies, so
the repo does not fight whatever else you have installed.

There are two places you write code, and they have different rules.

**`practice/` — your exercise answers.** One module, one subfolder per lesson:

```bash
cd practice
go mod init github.com/Matrosovdream/blockchain-learning/practice
mkdir 02-environment-setup
# write 02-environment-setup/main.go, then:
go run ./02-environment-setup
```

Each subfolder is its own `package main`, so `go run ./NN-name` works from the module root and the
folders never conflict. This directory is gitignored on purpose — it is your scratch work, not
course material.

**`/tmp/bc-ex` — the scratch loop for examples.** The examples are meant to be *retyped*, not
copy-pasted, and then thrown away:

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch                             # once
go get github.com/ethereum/go-ethereum@latest   # once
# paste an example into main.go
go run .
```

Retyping is not a ritual. It forces you to read every line, and you will catch things your eyes skip
when scrolling. Example [1](examples/02-environment-setup/1-easy.md#1-what-am-i-actually-running)
prints your toolchain and the versions the linker actually used — worth running now so you know what
you are on.

### 2. The Go blockchain toolbox

Four dependencies carry most of this course.

**`github.com/ethereum/go-ethereum`** — geth, used as a *library*. This is the single most important
import in the repo, and you will pull different pieces of it in different lessons:

| Package | What you use it for | First appears |
|---|---|---|
| `crypto` | Keccak-256, secp256k1 keys, signing, recovery | [04](04-hash-functions.md), [06](06-keys-signatures.md) |
| `common` | `Address`, `Hash`, hex helpers | [03](03-bytes-encoding.md) |
| `rlp` | Ethereum's serialization format | [17](17-rlp-merkle-patricia-trie.md) |
| `core/types` | `Block`, `Transaction`, `Receipt`, `Log` | [16](16-ethereum-architecture.md) |
| `core/vm` | the EVM interpreter | [18](18-evm.md) |
| `trie` | Merkle Patricia trie and proof verification | [17](17-rlp-merkle-patricia-trie.md) |
| `accounts/abi` | ABI encode/decode | [23](23-abi-encoding.md) |
| `accounts/abi/bind` | generated contract bindings | [24](24-abigen-bindings.md) |
| `ethclient` | the JSON-RPC client | this lesson |
| `ethclient/simulated` | an in-process chain for tests | this lesson, [34](34-testing-blockchain-go.md) |

**`github.com/holiman/uint256`** — a fixed-size 256-bit integer that does not allocate. The EVM's
word size is 256 bits, and `math/big` allocates on nearly every operation, which matters in an
interpreter loop ([18](18-evm.md)) or a hot decode path ([57](57-high-throughput-ingestion.md)).
Use `math/big` for APIs and readability, `uint256` where the profiler tells you to.

**`github.com/btcsuite/btcd`** — Bitcoin: `txscript`, `btcutil`, `chaincfg`, and `btcec/v2` for
secp256k1 and Schnorr. Arrives properly in [36](36-bitcoin-deep-dive.md).

**`golang.org/x/crypto`** — `sha3` (for Keccak), `scrypt`, `ripemd160`, `argon2`, `hkdf`. The
standard library does not ship Keccak-256, and the distinction between it and NIST SHA3-256 will
bite you in [04](04-hash-functions.md).

> **go-ethereum is a big dependency.** A first build pulls a lot and takes a while. It is fine —
> the module cache is shared, so it happens once. Keep one scratch module rather than creating a new
> one per example, and do not add `go get` to any hot loop of your workflow.

### 3. Choosing a chain to talk to

You need something to dial. Three options, and you will end up using all three.

| | `anvil` | `geth --dev` | Hosted RPC |
|---|---|---|---|
| Startup | instant | seconds | none |
| Funded accounts | 10, published keys | 1 | none — you fund it |
| Mining | instant or interval | instant or interval | real block times |
| Real mainnet state | yes, via `--fork-url` | no | yes, it *is* mainnet |
| Historical/archive | within the fork | no | depends on plan |
| Rate limits | none | none | yes, and they bite |
| Realism | good | high | total |

**`anvil`** (part of Foundry) is the default for this course. It starts in milliseconds, funds ten
accounts, mines on demand, and can fork mainnet at a pinned block. Nearly every example from
[21](21-sending-transactions.md) onward assumes it.

**`geth --dev`** is a real client in single-node dev mode. Useful when you want to see actual client
behaviour and logs, and required in [33](33-node-operations.md) and [63](63-private-networks.md).

**A hosted RPC provider** (Alchemy, Infura, QuickNode, Ankr) is how you read real chains. Free tiers
are generous enough for this course, but they have rate limits, `eth_getLogs` block-range caps, and
they often withhold the `debug_*` namespace and historical state. Example
[11](examples/02-environment-setup/3-hard.md#11-probe-what-the-endpoint-can-actually-do) probes
exactly this, and you should run it against any provider before designing around a method.

**There is also a fourth option, and this lesson uses it: no installs at all.** go-ethereum's
`ethclient/simulated` builds a real node stack in your process, and you can ask it to serve HTTP
JSON-RPC:

```go
port := freePort()
sim := simulated.NewBackend(types.GenesisAlloc{}, func(nc *node.Config, ec *ethconfig.Config) {
    nc.HTTPHost, nc.HTTPPort = "127.0.0.1", port
    nc.HTTPModules = []string{"eth", "net", "web3"}
})
defer sim.Close()
url := fmt.Sprintf("http://127.0.0.1:%d", port)
```

That is a genuine JSON-RPC endpoint over genuine HTTP — not a mock. Its chain id is **1337**, the
same id anvil uses, and example [7](examples/02-environment-setup/2-medium.md#7-the-ten-funded-dev-accounts)
funds the same ten accounts anvil funds. So every example in this lesson runs today with nothing
installed, and runs unchanged against real anvil tomorrow. That pattern —
`RPC_URL` set means your node, unset means an in-process one — is worth keeping in your own tools.

### 4. The Solidity toolchain (yes, you need it)

You are not going to become a Solidity developer, but you cannot be a competent Go engineer in this
space without being able to compile a contract and poke at one from the command line. Install
[Foundry](https://book.getfoundry.sh/):

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge --version && cast --version && anvil --version
```

Four tools:

- **`forge`** — build and test contracts. `forge build`, `forge test`, `forge create`,
  `forge inspect <C> storageLayout`.
- **`cast`** — the Swiss army knife. `cast call`, `cast send`, `cast block`, `cast storage`,
  `cast 4byte-decode`, `cast wallet sign`.
- **`anvil`** — the local chain.
- **`chisel`** — a Solidity REPL. Rarely needed.

**`cast` is your ground truth.** When your Go code produces a selector, an ABI encoding or a
signature and you are not sure it is right, `cast` gives you the reference answer in one command.
Several later lessons explicitly say "verify against `cast`" — that is why.

You can also use `solc` directly. Pin the version: **the same source compiled by different compiler
versions produces different bytecode**, which matters when verifying a deployed contract
([22](22-solidity-basics.md)) and when a chain has not enabled an opcode a newer compiler emits
([65](65-precompiles-forks.md)).

### 5. Testnets, faucets and explorers

Testnets are real networks with worthless money. Use them for anything involving a signature.

The names change. **Sepolia** has been the general-purpose application testnet for some time, and
**Hoodi** is the current validator/staking testnet. Goerli and Holesky were both retired. Rather
than trust a date-stamped list in a lesson, check
[ethereum.org/en/developers/docs/networks/](https://ethereum.org/en/developers/docs/networks/)
and your client's release notes — testnet deprecation is announced there first, and a deprecated
testnet degrades quietly before it dies.

**Faucets** hand out test ETH, usually rate-limited per address per day and often requiring a
sign-in to prove you are not a script. Get some now; waiting until you need it mid-lesson is
annoying. If a faucet asks for mainnet funds or a mainnet signature to prove liveness, understand
what you are signing before you sign it.

**Never learn on mainnet.** Not "mostly avoid" — never. A misconfigured loop that sends 200
transactions costs nothing on Sepolia and real money on mainnet, and the mistake that hurts is
always the one you did not anticipate. Reading mainnet is fine and you will do it often; writing to
it is not part of this course.

**Block explorers** (Etherscan and its per-chain siblings) are a debugging tool, not a UI. For any
transaction you send, learn to read:

- the **status** — success or reverted, and note a reverted transaction still cost gas;
- the **receipt** — gas used, effective gas price, the actual fee paid;
- the **logs** — the events emitted, decoded if the contract is verified ([25](25-events-logs.md));
- the **internal transactions** — value moved by contract calls, which produce no top-level
  transaction of their own. This one catches people out constantly
  ([58](58-deposit-detection.md) is largely about it);
- the **input data** — the calldata, decoded if the ABI is known ([23](23-abi-encoding.md)).

**Verified source** means someone uploaded source that compiles to the deployed bytecode, and the
explorer checked it. Unverified bytecode is not automatically malicious — but you cannot read what it
does, and for anything holding value that is a reason to walk away.

### 6. Secret hygiene from day one

You currently have nothing worth stealing. That is exactly why this is the right moment to build the
habits, because you will not retrofit them later.

**Configuration comes from the environment, and missing means fatal.** Never a default private key,
never a default that points at mainnet. Example
[2](examples/02-environment-setup/1-easy.md#2-config-from-the-environment-fail-fast) reports *every*
missing variable at once rather than the first — one restart instead of five:

```go
func Load() (Config, error) {
    var missing []string
    c := Config{RPCURL: os.Getenv("RPC_URL"), ChainID: os.Getenv("CHAIN_ID")}
    if c.RPCURL == "" {
        missing = append(missing, "RPC_URL")
    }
    if c.ChainID == "" {
        missing = append(missing, "CHAIN_ID")
    }
    if len(missing) > 0 {
        return c, fmt.Errorf("missing required environment variables: %s", strings.Join(missing, ", "))
    }
    return c, nil
}
```

**This repo's `.gitignore` already covers the dangerous paths** — `.env`, `.env.*`, `*.local`,
`*.key`, `keystore/`, `wallet.dat`, `mnemonic.txt`. Read it once so you know what is protected and,
more importantly, what is not.

**Make secrets unprintable.** A key that cannot be printed cannot be leaked by a hasty debug line.
Example [3](examples/02-environment-setup/1-easy.md#3-a-secret-that-cannot-be-printed) closes the two
paths a value normally escapes through:

```go
type Secret string

func (s Secret) String() string        { return "[REDACTED]" }
func (s Secret) LogValue() slog.Value  { return slog.StringValue("[REDACTED]") }
func (s Secret) Reveal() string        { return string(s) } // deliberate, and greppable
```

`fmt` goes through `String`, `slog` goes through `LogValue`, and the only way to the real value is a
method you can grep for in review. Lesson [32](32-key-management-signing.md) builds this out into a
proper signing boundary.

**Label test keys as test keys, every single time.** The examples in this course use the published
anvil key `0xac0974be...ff80`, always with a comment saying so. The habit matters: an unlabelled key
in a repo is indistinguishable from a real one, and reviewers cannot tell.

**Secret scanning belongs in CI before you have anything to lose.** `gitleaks` or `trufflehog` on
every push. And understand the recovery procedure: if a key ever touches a commit, **rotate it** —
rewriting history does not help, because the object was already pushed, mirrored, and possibly
indexed within seconds.

### 7. The CLI toolbox

You will live in these:

- **`cast`** — your reference implementation for anything Go produces.
- **`curl` + `jq`** — raw JSON-RPC. Worth doing by hand once so `ethclient` never feels like magic:
  ```bash
  curl -s -X POST -H 'Content-Type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' \
    http://127.0.0.1:8545 | jq -r .result
  ```
- **`websocat`** — the equivalent for WebSocket subscriptions ([20](20-json-rpc-ethclient.md)).
- **`xxd`** / **`hexdump -C`** — look at raw bytes. You will do this constantly from
  [03](03-bytes-encoding.md) onward, and reading a hex dump fluently is a real skill here.
- **A `Makefile`** — you will retype the same four commands hundreds of times. Write them down once.
- **Editor** — `gopls` for Go, plus a Solidity extension so contracts are readable rather than beige.

### 8. Your first verification script

Everything above comes together in a preflight check: dial an endpoint, confirm it is the chain you
expect, and confirm it can answer the questions you are about to ask. Run it before every lesson.

The three failure modes each need a different response, and example
[9](examples/02-environment-setup/2-medium.md#9-the-three-ways-connecting-fails) separates them:

| Failure | How you detect it | What to do |
|---|---|---|
| Nothing listening | `errors.As(err, &net.Error)` | fix the URL or start the node — not retryable on its own |
| Wrong chain | your own `ErrWrongChain` | **stop.** Catch it at startup, never at signing time |
| Deadline exceeded | `errors.Is(err, context.DeadlineExceeded)` | maybe retry with backoff, inside a longer budget |

Detect them by **type**, never by matching the error string — the text contains the URL and varies
between providers and transports.

Two rules that hold for the rest of the course. First, **every RPC call gets a `context` deadline**;
a hung provider must never become a hung worker. Second, **verify the chain id before you do anything
else**, and close the client if it is wrong — a half-valid client is worse than none:

```go
func Connect(ctx context.Context, cfg Config) (*ethclient.Client, error) {
    c, err := ethclient.DialContext(ctx, cfg.RPCURL)
    if err != nil {
        return nil, fmt.Errorf("dial %s: %w", cfg.RPCURL, err)
    }
    got, err := c.ChainID(ctx)
    if err != nil {
        c.Close()
        return nil, fmt.Errorf("chain id: %w", err)
    }
    if got.Cmp(cfg.ChainID) != 0 {
        c.Close()
        return nil, fmt.Errorf("%w: endpoint is %s, config expects %s", ErrWrongChain, got, cfg.ChainID)
    }
    return c, nil
}
```

Example [10](examples/02-environment-setup/3-hard.md#10-chainenv-load-dial-verify-hand-back-a-client)
is the full version, and example
[12](examples/02-environment-setup/3-hard.md#12-preflight-the-check-you-run-before-every-lesson)
turns it into a report with fatal and advisory checks that exits non-zero when the environment is
not usable — which is also the shape of a Kubernetes readiness probe ([68](68-deploying-operating.md)).

Write it into `practice/02-environment-setup/main.go`. You will run it more than any other program
in this course.

## Exercises

Write these in `practice/02-environment-setup/`.

1. **Scaffold the module.** Create `practice/` as a module, add an `02-environment-setup` subfolder,
   and get `go run ./02-environment-setup` working. Print your Go version, platform, and the
   go-ethereum version the linker resolved.
2. **Install and verify the toolchain.** Install Foundry. Confirm `forge`, `cast` and `anvil` all
   report versions. Start `anvil` in one terminal and, from another, get the latest block number
   with `cast block-number` *and* with raw `curl | jq`. Confirm they agree.
3. **Build your preflight.** Write the script from topic 8: load `RPC_URL` and `CHAIN_ID` from the
   environment, dial, verify the chain id, print head / base fee / gas tip, and exit non-zero if
   anything fatal fails. Run it against `anvil`.
4. **Break it on purpose, three ways.** Point it at a dead port. Point it at `anvil` while
   `CHAIN_ID=1`. Give it a 1-millisecond deadline. Confirm each produces a *different, specific*
   message, and that you detect all three with `errors.Is`/`errors.As` rather than string matching.
5. **Probe a real provider.** Sign up for a free tier somewhere, point the probe from example 11 at
   it, and write down which of archive state, `eth_getLogs`, `debug_*` and `txpool_*` you actually
   get. Then find the documented `eth_getLogs` block-range cap. You will need this number in
   [31](31-blockchain-indexer.md).
6. **Do a full testnet round trip.** Get Sepolia ETH from a faucet. Send a small amount to your own
   second address with `cast send`. Find the transaction on the explorer and write down its status,
   gas used, effective gas price, and total fee paid. Then read those same values from Go.
7. **Make a secret unprintable.** Implement the `Secret` type. Write a test asserting that
   `fmt.Sprintf("%v")`, `fmt.Sprintf("%s")` and a `slog` line all produce `[REDACTED]`. Then try to
   leak it anyway — `%#v`, JSON marshalling, embedding it in a struct — and fix whatever escapes.
8. **Write the Makefile.** Targets for: start anvil, run the preflight, run a lesson's practice
   folder, and `gofmt`+`vet` everything. You will use it every day.

## Best Practices & Pitfalls

- **Never commit a `.env`, a keystore or a key — and if you do, rotate, do not rewrite.**
  *Why:* this is the single most common way people lose funds, testnet and mainnet alike. Bots scan
  public commits within seconds; by the time you force-push, the key is already harvested. Git
  history rewriting does not un-leak a secret.
- **Do not learn on mainnet, and do not "just read" it without a budget.**
  *Why:* the accidental 200-transaction loop is free on Sepolia. And read-only work against a free
  RPC tier will hit a rate limit mid-lesson, which looks exactly like a bug in your code
  ([33](33-node-operations.md)).
- **Verify the chain id at startup and refuse to continue on mismatch.**
  *Why:* a transaction signed for one chain is invalid on another — that is deliberate replay
  protection ([19](19-transaction-types.md)) — but the failure surfaces late and confusingly. Catch
  it when you connect, not when you sign.
- **Every RPC call carries a `context` deadline.**
  *Why:* without one, a slow provider turns into an indefinitely blocked goroutine, and a worker pool
  that never returns. Set it at the call site, not globally.
- **Classify errors by type, never by message text.**
  *Why:* error strings contain URLs and differ between providers, transports and versions. Wrap with
  `%w`, define your own sentinels, and branch with `errors.Is`/`errors.As`.
- **Probe a provider's capabilities before you design around a method.**
  *Why:* `debug_*`, `txpool_*` and historical state are commonly missing on free tiers, and you will
  discover it after building on them. Example 11 takes ten seconds.
- **Pin the Solidity compiler version.**
  *Why:* identical source under different `solc` versions produces different bytecode, which breaks
  source verification and can emit opcodes a target chain has not enabled.
- **Keep one scratch module, not one per example.**
  *Why:* go-ethereum is a large dependency graph. Resolving it once is fine; resolving it forty times
  is an afternoon.

## Checklist

- [ ] I have Go 1.22+ and a `practice/` module where `go run ./NN-name` works.
- [ ] I can explain what each of the four core dependencies is for.
- [ ] I have `forge`, `cast` and `anvil` installed and version-checked.
- [ ] I can start a local chain and read its block number from both `cast` and Go.
- [ ] I know how to reach a chain with no installs at all, using the in-process backend.
- [ ] I have testnet ETH and have read one of my own transactions on an explorer.
- [ ] I can name the five things worth reading on an explorer's transaction page.
- [ ] My secrets come from the environment, missing config is fatal, and my keys cannot be printed.
- [ ] I have a preflight script that verifies the chain id and exits non-zero when unusable.
- [ ] I can distinguish a dead endpoint, a wrong chain and a timeout using `errors.Is`/`errors.As`.

## Resources

**Setup**

- Go — Getting started: https://go.dev/doc/tutorial/getting-started
- Foundry Book: https://book.getfoundry.sh/
- Foundry — Anvil: https://book.getfoundry.sh/anvil/
- go-ethereum docs: https://geth.ethereum.org/docs

**Reference**

- Ethereum networks & testnets: https://ethereum.org/en/developers/docs/networks/
- JSON-RPC API: https://ethereum.org/en/developers/docs/apis/json-rpc/
- `ethclient` package: https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient
- `ethclient/simulated`: https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient/simulated
- Go — `context` package: https://pkg.go.dev/context
- Go — `log/slog` package: https://pkg.go.dev/log/slog

**Hygiene**

- gitleaks: https://github.com/gitleaks/gitleaks
- Go — working with errors: https://go.dev/blog/go1.13-errors

---

**Examples:** [`examples/02-environment-setup/`](examples/02-environment-setup/) — **12 runnable Go
programs** (🟢 5 easy · 🟡 4 medium · 🔴 3 hard). Examples 1–3 need only the standard library;
4–12 start a real HTTP JSON-RPC endpoint in-process, so they run with nothing installed and work
unchanged against anvil.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
