# Step 02 — Environment Setup & Tooling · 🔴 Hard

Examples **10–12**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 10. chainenv: load, dial, verify, hand back a client

`🔴 hard` · *Reusable setup*

This is the chunk you lift into `practice/` and reuse for the rest of the course: load config, dial, verify the chain, hand back a client that is safe to use. Note that a client which reached the wrong network is closed and never returned — a half-valid client is worse than none.

**Steps:**

1. Write `LoadConfig` returning every missing variable at once, with typed sentinel errors.
2. Write `Connect` that dials, checks the chain id, and closes on mismatch.
3. Exercise both failure paths, then the happy path.
4. In a real project this lives in `internal/chainenv`; here it is inline because every example is one file.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"math/big"
	"net"
	"os"
	"strconv"
	"time"

	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/eth/ethconfig"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/ethclient/simulated"
	"github.com/ethereum/go-ethereum/node"
)

// ===========================================================================
// This is the chunk you will lift into practice/ and reuse for the rest of the
// course. In a real project it lives in its own package (internal/chainenv);
// here it is inline because every example must be one runnable file.
// ===========================================================================

var (
	ErrMissingConfig = errors.New("missing configuration")
	ErrWrongChain    = errors.New("wrong chain")
)

// Config is what the process needs from its environment.
type Config struct {
	RPCURL  string
	ChainID *big.Int
	Timeout time.Duration
}

// LoadConfig reads the environment and reports every problem at once.
func LoadConfig() (Config, error) {
	var cfg Config
	var missing []string

	cfg.RPCURL = os.Getenv("RPC_URL")
	if cfg.RPCURL == "" {
		missing = append(missing, "RPC_URL")
	}
	raw := os.Getenv("CHAIN_ID")
	if raw == "" {
		missing = append(missing, "CHAIN_ID")
	} else {
		n, ok := new(big.Int).SetString(raw, 10)
		if !ok {
			return cfg, fmt.Errorf("CHAIN_ID %q is not a number", raw)
		}
		cfg.ChainID = n
	}
	cfg.Timeout = 10 * time.Second
	if raw := os.Getenv("RPC_TIMEOUT_SECONDS"); raw != "" {
		n, err := strconv.Atoi(raw)
		if err != nil {
			return cfg, fmt.Errorf("RPC_TIMEOUT_SECONDS %q is not a number", raw)
		}
		cfg.Timeout = time.Duration(n) * time.Second
	}
	if len(missing) > 0 {
		return cfg, fmt.Errorf("%w: %v", ErrMissingConfig, missing)
	}
	return cfg, nil
}

// Connect dials, verifies the chain, and returns a client that is safe to use.
// A client that reached the wrong network is never returned — it is closed.
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

// ===========================================================================

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
	// Start a chain and point the environment at it, so this example is
	// self-contained. In practice these come from your shell or .env.
	url, stop := endpoint()
	defer stop()
	os.Setenv("RPC_URL", url)
	os.Setenv("CHAIN_ID", "1337")

	// --- the failure path, first -------------------------------------------
	os.Unsetenv("CHAIN_ID")
	if _, err := LoadConfig(); err != nil {
		fmt.Printf("config: %v  (ErrMissingConfig=%v)\n", err, errors.Is(err, ErrMissingConfig))
	}
	os.Setenv("CHAIN_ID", "1")
	cfg, _ := LoadConfig()
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if _, err := Connect(ctx, cfg); err != nil {
		fmt.Printf("connect: %v  (ErrWrongChain=%v)\n", err, errors.Is(err, ErrWrongChain))
	}

	// --- the happy path ----------------------------------------------------
	os.Setenv("CHAIN_ID", "1337")
	cfg, err := LoadConfig()
	if err != nil {
		fmt.Println("config:", err)
		os.Exit(1)
	}
	client, err := Connect(ctx, cfg)
	if err != nil {
		fmt.Println("connect:", err)
		os.Exit(1)
	}
	defer client.Close()

	head, err := client.BlockNumber(ctx)
	if err != nil {
		fmt.Println("head:", err)
		os.Exit(1)
	}
	fmt.Printf("\nready: chain %s, head %d, timeout %s\n", cfg.ChainID, head, cfg.Timeout)
	fmt.Println("every later lesson starts from exactly this point")
}
```

**Output:**

```
config: missing configuration: [CHAIN_ID]  (ErrMissingConfig=true)
connect: wrong chain: endpoint is 1337, config expects 1  (ErrWrongChain=true)

ready: chain 1337, head 0, timeout 10s
every later lesson starts from exactly this point
```

---

## 11. Probe what the endpoint can actually do

`🔴 hard` · *Provider capabilities*

Providers differ in what they will actually answer, and the differences are not documented consistently. Historical state, tracing and the mempool view are the three that most often turn out to be missing *after* you designed around them. Probe first.

**Steps:**

1. Check historical state with `BalanceAt` at block 0 — archive nodes answer, pruned ones error.
2. Check `eth_feeHistory` and `eth_getLogs`, which lessons 21, 25 and 31 depend on.
3. Drop to the raw `*rpc.Client` for `debug_*` and `txpool_*`, which `ethclient` does not wrap.
4. Read the JSON-RPC error code: −32601 is "method not found".

```go
package main

import (
	"context"
	"fmt"
	"math/big"
	"net"
	"os"
	"time"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/eth/ethconfig"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/ethclient/simulated"
	"github.com/ethereum/go-ethereum/node"
	"github.com/ethereum/go-ethereum/rpc"
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

// probe runs one capability check and reports yes/no plus a short reason.
func probe(name string, fn func() error) {
	err := fn()
	if err == nil {
		fmt.Printf("  %-24s yes\n", name)
		return
	}
	reason := "unsupported"
	var rpcErr rpc.Error
	if ok := asRPCError(err, &rpcErr); ok {
		reason = fmt.Sprintf("rpc error %d", rpcErr.ErrorCode())
	}
	fmt.Printf("  %-24s no   (%s)\n", name, reason)
}

func asRPCError(err error, target *rpc.Error) bool {
	if e, ok := err.(rpc.Error); ok {
		*target = e
		return true
	}
	return false
}

func main() {
	url, stop := endpoint()
	defer stop()

	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Println("dial:", err)
		os.Exit(1)
	}
	defer client.Close()
	raw := client.Client() // the underlying *rpc.Client, for unwrapped methods

	id, _ := client.ChainID(ctx)
	head, _ := client.BlockNumber(ctx)
	fmt.Printf("endpoint: chain %s, head %d\n\n", id, head)
	fmt.Println("capabilities")

	// Historical state: an archive node answers, a pruned full node errors.
	probe("archive state", func() error {
		_, err := client.BalanceAt(ctx, common.Address{}, big.NewInt(0))
		return err
	})

	// Fee history — needed for the gas estimation in lesson 21.
	probe("eth_feeHistory", func() error {
		var out any
		return raw.CallContext(ctx, &out, "eth_feeHistory", "0x1", "latest", nil)
	})

	// Log filtering — the whole of lesson 25 and the indexer in lesson 31.
	probe("eth_getLogs", func() error {
		_, err := client.FilterLogs(ctx, ethereum.FilterQuery{
			FromBlock: big.NewInt(0), ToBlock: big.NewInt(int64(head)),
		})
		return err
	})

	// The debug namespace — tracing. Most public providers do not expose it.
	probe("debug_traceBlock", func() error {
		var out any
		return raw.CallContext(ctx, &out, "debug_traceBlockByNumber", "latest", nil)
	})

	// The txpool namespace — the mempool view used in lesson 54.
	probe("txpool_status", func() error {
		var out any
		return raw.CallContext(ctx, &out, "txpool_status")
	})

	fmt.Println("\nrun this against every provider before you design around a method")
	fmt.Println("free tiers routinely drop debug_*, txpool_* and historical state")
}
```

**Output:**

```
endpoint: chain 1337, head 0

capabilities
  archive state            yes
  eth_feeHistory           yes
  eth_getLogs              yes
  debug_traceBlock         no   (rpc error -32601)
  txpool_status            no   (rpc error -32601)

run this against every provider before you design around a method
free tiers routinely drop debug_*, txpool_* and historical state
```

---

## 12. Preflight: the check you run before every lesson

`🔴 hard` · *Preflight*

The script to run before starting any lesson, and the shape of a readiness probe in production (lesson 68). Fatal checks stop the run; advisory ones warn. Latency is bucketed rather than printed exactly, so the report is reproducible.

**Steps:**

1. Define checks as data — name, fatal or not, and a function.
2. Run each with its own per-call deadline inside one overall budget.
3. Warn on a stale head, fail on the wrong chain id.
4. Exit non-zero when a fatal check fails, so it composes into a `make` target or CI.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"math/big"
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

// check is one line of the report. Fatal checks abort the run.
type check struct {
	name  string
	fatal bool
	run   func(context.Context, *ethclient.Client) (string, error)
}

// bucket keeps the report reproducible: exact millisecond counts would differ
// on every run and on every machine.
func bucket(d time.Duration) string {
	switch {
	case d < 50*time.Millisecond:
		return "< 50ms"
	case d < 250*time.Millisecond:
		return "< 250ms"
	case d < time.Second:
		return "< 1s"
	}
	return ">= 1s"
}

func gwei(wei *big.Int) string {
	if wei == nil {
		return "n/a"
	}
	q, r := new(big.Int).QuoRem(wei, big.NewInt(1e9), new(big.Int))
	return fmt.Sprintf("%s.%09d", q, r)
}

func main() {
	url, stop := endpoint()
	defer stop()

	want := big.NewInt(1337)
	if raw := os.Getenv("CHAIN_ID"); raw != "" {
		if n, ok := new(big.Int).SetString(raw, 10); ok {
			want = n
		}
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	source := "in-process dev chain"
	if os.Getenv("RPC_URL") != "" {
		source = "RPC_URL"
	}
	fmt.Printf("preflight  (endpoint: %s, expecting chain %s)\n\n", source, want)

	start := time.Now()
	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Printf("  %-18s FAIL  %v\n", "dial", err)
		fmt.Println("\nnot usable — stop here and fix the endpoint")
		os.Exit(1)
	}
	defer client.Close()
	fmt.Printf("  %-18s ok    %s\n", "dial", bucket(time.Since(start)))

	checks := []check{
		{"chain id", true, func(ctx context.Context, c *ethclient.Client) (string, error) {
			got, err := c.ChainID(ctx)
			if err != nil {
				return "", err
			}
			if got.Cmp(want) != 0 {
				return "", fmt.Errorf("endpoint is %s, expected %s", got, want)
			}
			return got.String(), nil
		}},
		{"head", true, func(ctx context.Context, c *ethclient.Client) (string, error) {
			n, err := c.BlockNumber(ctx)
			if err != nil {
				return "", err
			}
			return fmt.Sprintf("block %d", n), nil
		}},
		{"head freshness", false, func(ctx context.Context, c *ethclient.Client) (string, error) {
			h, err := c.HeaderByNumber(ctx, nil)
			if err != nil {
				return "", err
			}
			if h.Time == 0 {
				return "genesis (dev chain)", nil
			}
			age := time.Since(time.Unix(int64(h.Time), 0))
			if age > 5*time.Minute {
				return "", fmt.Errorf("head is %s old — node is behind", age.Round(time.Second))
			}
			return "fresh", nil
		}},
		{"base fee", false, func(ctx context.Context, c *ethclient.Client) (string, error) {
			h, err := c.HeaderByNumber(ctx, nil)
			if err != nil {
				return "", err
			}
			if h.BaseFee == nil {
				return "", errors.New("pre-London chain, no base fee")
			}
			return gwei(h.BaseFee) + " gwei", nil
		}},
		{"gas tip", false, func(ctx context.Context, c *ethclient.Client) (string, error) {
			t, err := c.SuggestGasTipCap(ctx)
			if err != nil {
				return "", err
			}
			return gwei(t) + " gwei", nil
		}},
	}

	failed := false
	for _, ck := range checks {
		callCtx, callCancel := context.WithTimeout(ctx, 5*time.Second)
		start := time.Now()
		detail, err := ck.run(callCtx, client)
		took := bucket(time.Since(start))
		callCancel()

		switch {
		case err == nil:
			fmt.Printf("  %-18s ok    %-9s %s\n", ck.name, took, detail)
		case ck.fatal:
			fmt.Printf("  %-18s FAIL  %-9s %v\n", ck.name, took, err)
			failed = true
		default:
			fmt.Printf("  %-18s warn  %-9s %v\n", ck.name, took, err)
		}
	}

	if failed {
		fmt.Println("\nnot usable — fix the failures above before starting the lesson")
		os.Exit(1)
	}
	fmt.Println("\nready. Run this before every lesson; it costs a second and saves an hour.")
}
```

**Output:**

```
preflight  (endpoint: in-process dev chain, expecting chain 1337)

  dial               ok    < 50ms
  chain id           ok    < 50ms    1337
  head               ok    < 50ms    block 0
  head freshness     ok    < 50ms    genesis (dev chain)
  base fee           ok    < 50ms    1.000000000 gwei
  gas tip            ok    < 50ms    0.001000000 gwei

ready. Run this before every lesson; it costs a second and saves an hour.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
