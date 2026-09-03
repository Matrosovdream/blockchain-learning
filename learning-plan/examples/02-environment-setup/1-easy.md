# Step 02 — Environment Setup & Tooling · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🟡 medium](2-medium.md)

---

## 1. What am I actually running?

`🟢 easy` · *Toolchain*

Before anything else, know exactly what you are running. `runtime/debug.ReadBuildInfo` reports the versions the linker actually used — not what `go.mod` asked for. When something works on one machine and not another, this is the first thing to compare.

**Steps:**

1. Print the Go version and platform from `runtime`.
2. Print go-ethereum's own version constants, and prove `uint256` is linked by computing 2²⁵⁶−1.
3. Walk `info.Deps` for the dependencies that matter in this course.
4. Your version numbers will differ from the output below — that is the point of printing them.

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"

	"github.com/ethereum/go-ethereum/version"
	"github.com/holiman/uint256"
)

func main() {
	fmt.Println("toolchain")
	fmt.Printf("  go        %s\n", runtime.Version())
	fmt.Printf("  platform  %s/%s\n", runtime.GOOS, runtime.GOARCH)

	// go-ethereum reports its own version. This is the library you will import in
	// almost every lesson from 17 onward, so pin it and know which one you are on.
	fmt.Printf("  geth      v%d.%d.%d-%s\n", version.Major, version.Minor, version.Patch, version.Meta)

	// Prove uint256 is linked and working: 2^256 - 1, the largest EVM word.
	max := new(uint256.Int).Not(uint256.NewInt(0))
	fmt.Printf("  uint256   max = %s\n", max.Dec())

	info, ok := debug.ReadBuildInfo()
	if !ok {
		fmt.Println("\nno build info")
		return
	}
	fmt.Printf("\nmodule      %s\n", info.Main.Path)

	// What the linker ACTUALLY used — not what go.mod asked for. When a bug report
	// says "works on my machine", this is the first thing to compare.
	fmt.Println("\nlinked dependencies")
	want := map[string]bool{
		"github.com/ethereum/go-ethereum": true,
		"github.com/holiman/uint256":      true,
		"golang.org/x/crypto":             true,
	}
	for _, dep := range info.Deps {
		if want[dep.Path] {
			fmt.Printf("  %-35s %s\n", dep.Path, dep.Version)
		}
	}
}
```

**Output:**

```
toolchain
  go        go1.26.3
  platform  darwin/arm64
  geth      v1.17.5-stable
  uint256   max = 115792089237316195423570985008687907853269984665640564039457584007913129639935

module      scratch

linked dependencies
  github.com/ethereum/go-ethereum     v1.17.5
  github.com/holiman/uint256          v1.3.2
```

---

## 2. Config from the environment, fail fast

`🟢 easy` · *Configuration*

Configuration comes from the environment, and a missing value is a startup error, never a zero value you discover three hours later. Note that `Load` reports *every* missing variable at once: one restart instead of five.

**Steps:**

1. Define a `Config` with required and optional fields — and no defaults for anything secret.
2. Collect all missing variables and return them in one error.
3. Run once with an empty environment to see the failure, then again with it populated.

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// Config is everything this program needs from the outside world.
// No defaults for anything secret, and no defaults that point at mainnet.
type Config struct {
	RPCURL   string // required
	ChainID  string // required
	Explorer string // optional
}

// Load reads the environment and reports EVERY problem at once, not just the first.
// Failing fast with a complete list is the difference between one restart and five.
func Load() (Config, error) {
	c := Config{
		RPCURL:   os.Getenv("RPC_URL"),
		ChainID:  os.Getenv("CHAIN_ID"),
		Explorer: os.Getenv("EXPLORER_URL"), // optional: blank is fine
	}
	var missing []string
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

func main() {
	// Simulate an empty environment so the failure path is the one you see.
	os.Unsetenv("RPC_URL")
	os.Unsetenv("CHAIN_ID")

	if _, err := Load(); err != nil {
		fmt.Println("startup failed:", err)
		fmt.Println("  -> fix your .env and restart; do not carry on with zero values")
	}

	// Now with the environment populated.
	os.Setenv("RPC_URL", "http://127.0.0.1:8545")
	os.Setenv("CHAIN_ID", "31337")

	cfg, err := Load()
	if err != nil {
		fmt.Println("still failing:", err)
		return
	}
	fmt.Printf("\nloaded: rpc=%s chain=%s explorer=%q\n", cfg.RPCURL, cfg.ChainID, cfg.Explorer)
	fmt.Println("optional values may be empty; required ones may never be")
}
```

**Output:**

```
startup failed: missing required environment variables: RPC_URL, CHAIN_ID
  -> fix your .env and restart; do not carry on with zero values

loaded: rpc=http://127.0.0.1:8545 chain=31337 explorer=""
optional values may be empty; required ones may never be
```

---

## 3. A secret that cannot be printed

`🟢 easy` · *Secret hygiene*

A private key that cannot be printed cannot be leaked by a hasty debug line. A `Secret` type with `String()` and `LogValue()` closes the two paths a value normally escapes through — `fmt` and `slog` — and leaves one deliberate, greppable way to get it out.

**Steps:**

1. Wrap a key in a `Secret` type implementing `String() string` and `LogValue() slog.Value`.
2. Print it through `Println`, `%v`, `%s` and a `slog` handler — all four redact.
3. Use the explicit `Reveal()` escape hatch, which is easy to find in code review.
4. The key here is the public anvil test key. Never a real one, not even in an example.

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
)

// Secret wraps a value that must never reach a log, an error string or a crash dump.
// The two methods below are the entire defence: fmt goes through String, slog goes
// through LogValue, and there is no path that prints the real thing by accident.
type Secret string

func (s Secret) String() string { return "[REDACTED]" }

func (s Secret) LogValue() slog.Value { return slog.StringValue("[REDACTED]") }

// Reveal is the deliberate, greppable way to get the value out.
// If you ever wonder "where could this leak?", grep for this method.
func (s Secret) Reveal() string { return string(s) }

func main() {
	// A well-known Hardhat/anvil TEST key. Never a real one, not even in an example.
	key := Secret("0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")

	fmt.Println("fmt.Println :", key)
	fmt.Printf("fmt %%v      : %v\n", key)
	fmt.Printf("fmt %%s      : %s\n", key)

	log := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
		ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
			if a.Key == "time" {
				return slog.Attr{} // drop timestamps so this output is reproducible
			}
			return a
		},
	}))
	log.Info("signing", "key", key, "addr", "0xf39F...2266")

	// The escape hatch, used on purpose and easy to find in review.
	fmt.Printf("\nReveal()    : %s...%s (only where it is genuinely needed)\n",
		key.Reveal()[:6], key.Reveal()[len(key.Reveal())-4:])

	fmt.Println("\na key that cannot be printed cannot be leaked by a hasty debug line")
}
```

**Output:**

```
fmt.Println : [REDACTED]
fmt %v      : [REDACTED]
fmt %s      : [REDACTED]
level=INFO msg=signing key=[REDACTED] addr=0xf39F...2266

Reveal()    : 0xac09...ff80 (only where it is genuinely needed)

a key that cannot be printed cannot be leaked by a hasty debug line
```

---

## 4. A chain to talk to, with no installs

`🟢 easy` · *Connecting*

Every chain example from here on opens with the `endpoint` helper below. With `RPC_URL` set it uses your node — anvil, `geth --dev`, a testnet. With it unset it starts a real HTTP JSON-RPC endpoint **in-process**, so the example runs with no installs at all. Chain id is 1337, the same id anvil uses, so switching between them changes nothing else.

**Steps:**

1. Read `RPC_URL`; fall back to an in-process dev chain on a free port.
2. Dial with `ethclient.DialContext` and a deadline.
3. Ask the two questions every session starts with: who are you (chain id), where are you (head).

```go
package main

import (
	"context"
	"fmt"
	"net"
	"os"
	"time"

	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/eth/ethconfig"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/ethclient/simulated"
	"github.com/ethereum/go-ethereum/node"
)

// ---------------------------------------------------------------------------
// endpoint returns an HTTP JSON-RPC URL to talk to.
//
//	RPC_URL set   -> your own node: anvil, geth --dev, or a testnet.
//	RPC_URL unset -> an in-process dev chain, started here, no installs needed.
//
// Every chain example in this lesson opens with this helper. Chain id is 1337,
// the same id anvil uses, so switching between the two changes nothing else.
// ---------------------------------------------------------------------------
func endpoint() (string, func()) {
	if u := os.Getenv("RPC_URL"); u != "" {
		return u, func() {}
	}
	port := freePort()
	sim := simulated.NewBackend(types.GenesisAlloc{}, func(nc *node.Config, ec *ethconfig.Config) {
		nc.HTTPHost, nc.HTTPPort = "127.0.0.1", port
		nc.HTTPModules = []string{"eth", "net", "web3"}
	})
	return fmt.Sprintf("http://127.0.0.1:%d", port), func() { sim.Close() }
}

func freePort() int {
	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}
	defer l.Close()
	return l.Addr().(*net.TCPAddr).Port
}

func main() {
	url, stop := endpoint()
	defer stop()

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Dial is where a bad URL, a dead node or a firewall shows up.
	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Println("dial failed:", err)
		os.Exit(1)
	}
	defer client.Close()

	chainID, err := client.ChainID(ctx)
	if err != nil {
		fmt.Println("chain id failed:", err)
		os.Exit(1)
	}
	head, err := client.BlockNumber(ctx)
	if err != nil {
		fmt.Println("block number failed:", err)
		os.Exit(1)
	}

	source := "in-process dev chain"
	if os.Getenv("RPC_URL") != "" {
		source = "RPC_URL"
	}
	fmt.Printf("endpoint   %s\n", source)
	fmt.Printf("chain id   %s\n", chainID)
	fmt.Printf("head       %d\n", head)
	fmt.Println("\nthat is the whole handshake: dial, ask who you are, ask where you are")
}
```

**Output:**

```
endpoint   in-process dev chain
chain id   1337
head       0

that is the whole handshake: dial, ask who you are, ask where you are
```

---

## 5. Every call gets a deadline

`🟢 easy` · *Deadlines*

Every RPC call in this course gets a `context` deadline, because a hung provider must never become a hung worker. A timeout and a cancellation are different signals and deserve different responses — and you tell them apart with `errors.Is`, never by matching the message.

**Steps:**

1. Make a normal call with a 5-second deadline.
2. Make one with a deadline that has already expired, and one with a cancelled context.
3. Match with `errors.Is` against `context.DeadlineExceeded` and `context.Canceled`.
4. The raw error text contains the URL and varies by transport — never match on it.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net"
	"os"
	"time"

	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/eth/ethconfig"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/ethclient/simulated"
	"github.com/ethereum/go-ethereum/node"
)

func endpoint() (string, func()) {
	if u := os.Getenv("RPC_URL"); u != "" {
		return u, func() {}
	}
	port := freePort()
	sim := simulated.NewBackend(types.GenesisAlloc{}, func(nc *node.Config, ec *ethconfig.Config) {
		nc.HTTPHost, nc.HTTPPort = "127.0.0.1", port
		nc.HTTPModules = []string{"eth", "net", "web3"}
	})
	return fmt.Sprintf("http://127.0.0.1:%d", port), func() { sim.Close() }
}

func freePort() int {
	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}
	defer l.Close()
	return l.Addr().(*net.TCPAddr).Port
}

func main() {
	url, stop := endpoint()
	defer stop()

	client, err := ethclient.Dial(url)
	if err != nil {
		fmt.Println("dial failed:", err)
		os.Exit(1)
	}
	defer client.Close()

	// A sane deadline. Every RPC call in this course gets one, because a hung
	// provider must never become a hung worker.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	id, err := client.ChainID(ctx)
	fmt.Printf("with 5s deadline : chain id %v, err %v\n", id, err)

	// Now an impossible deadline: already expired before the call starts.
	tight, cancelTight := context.WithTimeout(context.Background(), time.Nanosecond)
	defer cancelTight()
	time.Sleep(time.Millisecond) // make sure it really has expired

	// Do not match on the error STRING — it contains the URL and varies by
	// transport. Match on the sentinel with errors.Is.
	_, err = client.ChainID(tight)
	report("with 1ns deadline", err)

	// A cancelled context is a different signal from a timed-out one: it means
	// somebody upstream gave up, usually because the user disconnected.
	cancelled, cancelNow := context.WithCancel(context.Background())
	cancelNow()
	_, err = client.ChainID(cancelled)
	report("cancelled ctx    ", err)

	fmt.Println("\ndistinguish the two: a timeout may be worth retrying, a cancel never is")
}

// report prints what the error IS, not what it says.
func report(label string, err error) {
	fmt.Printf("%s: failed=%v  DeadlineExceeded=%v  Canceled=%v\n",
		label, err != nil,
		errors.Is(err, context.DeadlineExceeded),
		errors.Is(err, context.Canceled))
}
```

**Output:**

```
with 5s deadline : chain id 1337, err <nil>
with 1ns deadline: failed=true  DeadlineExceeded=true  Canceled=false
cancelled ctx    : failed=true  DeadlineExceeded=false  Canceled=true

distinguish the two: a timeout may be worth retrying, a cancel never is
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
