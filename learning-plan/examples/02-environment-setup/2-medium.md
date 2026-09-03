# Step 02 — Environment Setup & Tooling · 🟡 Medium

Examples **6–9**. Each is a complete `package main` program: read the concept and steps,
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

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [🔴 hard](3-hard.md)

---

## 6. Head, base fee and what gas costs

`🟡 medium` · *Chain state*

The three numbers you will look up constantly: where the chain is, what the base fee is, and what a unit of gas currently costs. `HeaderByNumber(nil)` means "latest" and is cheaper than `BlockByNumber` because it does not transfer the transaction list.

**Steps:**

1. Fetch the latest header and print height, gas limit, gas used and base fee.
2. Call `SuggestGasPrice` (legacy) and `SuggestGasTipCap` (EIP-1559) — two different questions.
3. Format wei as gwei with integer arithmetic only; there is no `float64` in a money path.
4. Add base fee and tip to get what one unit of gas costs right now (lesson 19).

```go
package main

import (
	"context"
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

func alloc() types.GenesisAlloc { return types.GenesisAlloc{} }

func endpoint() (string, func()) {
	if u := os.Getenv("RPC_URL"); u != "" {
		return u, func() {}
	}
	port := freePort()
	sim := simulated.NewBackend(alloc(), func(nc *node.Config, ec *ethconfig.Config) {
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

// gwei formats wei as gwei using integer arithmetic only.
// There is no float64 anywhere in this course's money paths — see lesson 03.
func gwei(wei *big.Int) string {
	if wei == nil {
		return "n/a"
	}
	q, r := new(big.Int).QuoRem(wei, big.NewInt(1e9), new(big.Int))
	return fmt.Sprintf("%s.%09d gwei", q, r)
}

func main() {
	url, stop := endpoint()
	defer stop()

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Println("dial:", err)
		os.Exit(1)
	}
	defer client.Close()

	// HeaderByNumber(nil) means "latest". Prefer it over BlockByNumber when you
	// only need header fields: it does not transfer the transaction list.
	head, err := client.HeaderByNumber(ctx, nil)
	if err != nil {
		fmt.Println("header:", err)
		os.Exit(1)
	}

	// Two different questions:
	//   SuggestGasPrice  - legacy, "what would a type-0 transaction pay?"
	//   SuggestGasTipCap - EIP-1559, "what tip gets me included?"  (lesson 21)
	gasPrice, errP := client.SuggestGasPrice(ctx)
	tip, errT := client.SuggestGasTipCap(ctx)

	fmt.Printf("height        %d\n", head.Number)
	fmt.Printf("gas limit     %d\n", head.GasLimit)
	fmt.Printf("gas used      %d\n", head.GasUsed)
	fmt.Printf("base fee      %s\n", gwei(head.BaseFee))
	if errP == nil {
		fmt.Printf("gas price     %s\n", gwei(gasPrice))
	}
	if errT == nil {
		fmt.Printf("priority tip  %s\n", gwei(tip))
	}

	// A type-2 transaction pays baseFee + tip, capped by maxFeePerGas (lesson 19).
	if head.BaseFee != nil && errT == nil {
		total := new(big.Int).Add(head.BaseFee, tip)
		fmt.Printf("\nbase + tip    %s   <- what one unit of gas costs right now\n", gwei(total))
	}
}
```

**Output:**

```
height        0
gas limit     60000000
gas used      0
base fee      1.000000000 gwei
gas price     1.001000000 gwei
priority tip  0.001000000 gwei

base + tip    1.001000000 gwei   <- what one unit of gas costs right now
```

---

## 7. The ten funded dev accounts

`🟡 medium` · *Dev accounts*

anvil funds ten accounts from a published mnemonic. The dev chain here funds the same ten with the same 10,000 ETH, so code you write against one works against the other. Every address below was derived from `m/44'/60'/0'/0/i` and checked, not copied from memory.

**Steps:**

1. Fund the ten canonical dev addresses in the genesis allocation.
2. Read each balance and nonce with `BalanceAt` and `NonceAt` at `nil` (latest).
3. Format wei as ether with integer division and a trimmed fraction — no floats.
4. Notice the mixed-case addresses: that casing is an EIP-55 checksum (lesson 07).

```go
package main

import (
	"context"
	"fmt"
	"math/big"
	"net"
	"os"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/eth/ethconfig"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/ethclient/simulated"
	"github.com/ethereum/go-ethereum/node"
)

// The ten accounts anvil funds by default, derived from the public
// "test test test ... junk" mnemonic. PUBLIC TEST ACCOUNTS — the private keys
// are on the internet. Never send anything real to these.
var devAccounts = []string{
	"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
	"0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
	"0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
	"0x90F79bf6EB2c4f870365E785982E1f101E93b906",
	"0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
	"0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc",
	"0x976EA74026E726554dB657fA54763abd0C3a0aa9",
	"0x14dC79964da2C08b23698B3D3cc7Ca32193d9955",
	"0x23618e81E3f5cdF7f54C3d65f7FBc0aBf5B21E8f",
	"0xa0Ee7A142d267C1f36714E4a8F75612F20a79720",
}

// alloc funds them with 10000 ETH each, exactly as anvil does.
func alloc() types.GenesisAlloc {
	tenk := new(big.Int).Mul(big.NewInt(10000), big.NewInt(1e18))
	g := types.GenesisAlloc{}
	for _, a := range devAccounts {
		g[common.HexToAddress(a)] = types.Account{Balance: new(big.Int).Set(tenk)}
	}
	return g
}

func endpoint() (string, func()) {
	if u := os.Getenv("RPC_URL"); u != "" {
		return u, func() {}
	}
	port := freePort()
	sim := simulated.NewBackend(alloc(), func(nc *node.Config, ec *ethconfig.Config) {
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

// ether formats wei with 18 decimals, trimmed. Integer math only.
func ether(wei *big.Int) string {
	q, r := new(big.Int).QuoRem(wei, big.NewInt(1e18), new(big.Int))
	frac := fmt.Sprintf("%018s", r.String())
	for len(frac) > 1 && frac[len(frac)-1] == '0' {
		frac = frac[:len(frac)-1]
	}
	return q.String() + "." + frac
}

func main() {
	url, stop := endpoint()
	defer stop()

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Println("dial:", err)
		os.Exit(1)
	}
	defer client.Close()

	fmt.Printf("%-4s %-44s %20s %s\n", "#", "address", "balance (ETH)", "nonce")
	total := new(big.Int)
	for i, a := range devAccounts {
		addr := common.HexToAddress(a)
		bal, err := client.BalanceAt(ctx, addr, nil) // nil == latest
		if err != nil {
			fmt.Printf("%-4d %-44s  error: %v\n", i, a, err)
			continue
		}
		nonce, err := client.NonceAt(ctx, addr, nil)
		if err != nil {
			fmt.Printf("%-4d %-44s  error: %v\n", i, a, err)
			continue
		}
		fmt.Printf("%-4d %-44s %20s %5d\n", i, addr.Hex(), ether(bal), nonce)
		total.Add(total, bal)
	}
	fmt.Printf("\ntotal funded: %s ETH across %d accounts\n", ether(total), len(devAccounts))
	fmt.Println("addresses print in EIP-55 mixed case — that casing is a checksum (lesson 07)")
}
```

**Output:**

```
#    address                                             balance (ETH) nonce
0    0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266                10000.0     0
1    0x70997970C51812dc3A010C7d01b50e0d17dc79C8                10000.0     0
2    0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC                10000.0     0
3    0x90F79bf6EB2c4f870365E785982E1f101E93b906                10000.0     0
4    0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65                10000.0     0
5    0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc                10000.0     0
6    0x976EA74026E726554dB657fA54763abd0C3a0aa9                10000.0     0
7    0x14dC79964da2C08b23698B3D3cc7Ca32193d9955                10000.0     0
8    0x23618e81E3f5cdF7f54C3d65f7FBc0aBf5B21E8f                10000.0     0
9    0xa0Ee7A142d267C1f36714E4a8F75612F20a79720                10000.0     0

total funded: 100000.0 ETH across 10 accounts
addresses print in EIP-55 mixed case — that casing is a checksum (lesson 07)
```

---

## 8. Raw JSON-RPC vs ethclient

`🟡 medium` · *The RPC layer*

`ethclient` is typed decoding over plain HTTP POSTs of JSON. Doing one call by hand demystifies the whole stack and shows you the hex quantity encoding you will meet everywhere. You need this when you reach a method `ethclient` does not wrap — `debug_*`, `txpool_*`, bundler RPC.

**Steps:**

1. Build the JSON-RPC 2.0 envelope and POST it with `net/http`.
2. Decode the hex quantity yourself with `big.Int.SetString(s, 16)`.
3. Make the same two calls through `ethclient` and compare.
4. Note the trap: a JSON-RPC error arrives with HTTP 200, so checking the status code is not enough.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"math/big"
	"net"
	"net/http"
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

// The JSON-RPC 2.0 envelope. This is all ethclient is doing underneath.
type rpcRequest struct {
	JSONRPC string `json:"jsonrpc"`
	ID      int    `json:"id"`
	Method  string `json:"method"`
	Params  []any  `json:"params"`
}

type rpcResponse struct {
	JSONRPC string          `json:"jsonrpc"`
	ID      int             `json:"id"`
	Result  json.RawMessage `json:"result"`
	Error   *struct {
		Code    int    `json:"code"`
		Message string `json:"message"`
	} `json:"error"`
}

func rawCall(ctx context.Context, url, method string, params ...any) (json.RawMessage, error) {
	if params == nil {
		params = []any{}
	}
	body, err := json.Marshal(rpcRequest{JSONRPC: "2.0", ID: 1, Method: method, Params: params})
	if err != nil {
		return nil, err
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(body))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	raw, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}
	var out rpcResponse
	if err := json.Unmarshal(raw, &out); err != nil {
		return nil, err
	}
	// A JSON-RPC error arrives with HTTP 200. Checking the status code is not enough.
	if out.Error != nil {
		return nil, fmt.Errorf("rpc error %d: %s", out.Error.Code, out.Error.Message)
	}
	return out.Result, nil
}

func main() {
	url, stop := endpoint()
	defer stop()

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// ---- layer 1: raw HTTP ------------------------------------------------
	chainIDHex, err := rawCall(ctx, url, "eth_chainId")
	if err != nil {
		fmt.Println("raw call:", err)
		os.Exit(1)
	}
	headHex, err := rawCall(ctx, url, "eth_blockNumber")
	if err != nil {
		fmt.Println("raw call:", err)
		os.Exit(1)
	}
	fmt.Println("raw JSON-RPC (net/http)")
	fmt.Printf("  eth_chainId     -> %s\n", chainIDHex)
	fmt.Printf("  eth_blockNumber -> %s\n", headHex)
	fmt.Println("  note: quantities are 0x-prefixed hex with no leading zeros")

	// Decode the hex quantity yourself.
	var s string
	if err := json.Unmarshal(chainIDHex, &s); err != nil {
		fmt.Println("decode:", err)
		os.Exit(1)
	}
	n, ok := new(big.Int).SetString(s[2:], 16)
	if !ok {
		fmt.Println("not hex:", s)
		os.Exit(1)
	}
	fmt.Printf("  decoded chain id = %s\n", n)

	// ---- layer 2: ethclient -----------------------------------------------
	client, err := ethclient.DialContext(ctx, url)
	if err != nil {
		fmt.Println("dial:", err)
		os.Exit(1)
	}
	defer client.Close()

	id, err := client.ChainID(ctx)
	if err != nil {
		fmt.Println("chain id:", err)
		os.Exit(1)
	}
	head, err := client.BlockNumber(ctx)
	if err != nil {
		fmt.Println("head:", err)
		os.Exit(1)
	}
	fmt.Println("\nethclient")
	fmt.Printf("  ChainID()     -> %s\n", id)
	fmt.Printf("  BlockNumber() -> %d\n", head)

	fmt.Println("\nsame endpoint, same answers — ethclient is typed decoding over the above")
	fmt.Println("drop to raw calls for methods it does not wrap (debug_*, txpool_*)")
}
```

**Output:**

```
raw JSON-RPC (net/http)
  eth_chainId     -> "0x539"
  eth_blockNumber -> "0x0"
  note: quantities are 0x-prefixed hex with no leading zeros
  decoded chain id = 1337

ethclient
  ChainID()     -> 1337
  BlockNumber() -> 0

same endpoint, same answers — ethclient is typed decoding over the above
drop to raw calls for methods it does not wrap (debug_*, txpool_*)
```

---

## 9. The three ways connecting fails

`🟡 medium` · *Failure modes*

Connecting fails in three distinct ways, and they call for three different responses. Wrap them in your own sentinel errors so callers can branch on the *kind* of failure instead of matching strings that change between providers.

**Steps:**

1. Dial a port nothing is listening on and detect it with `errors.As(&net.Error)`.
2. Connect successfully but to the wrong chain, and reject it with a custom `ErrWrongChain`.
3. Blow a deadline and detect it with `errors.Is(context.DeadlineExceeded)`.
4. Catch the wrong-chain case at startup — never at signing time (lesson 19).

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

// ErrWrongChain is your own sentinel, so callers can react to it specifically
// instead of matching on a message.
var ErrWrongChain = errors.New("wrong chain")

// connect dials and refuses to hand back a client on the wrong network.
func connect(ctx context.Context, url string, want *big.Int) (*ethclient.Client, error) {
	c, err := ethclient.DialContext(ctx, url)
	if err != nil {
		return nil, fmt.Errorf("dial: %w", err)
	}
	got, err := c.ChainID(ctx)
	if err != nil {
		c.Close()
		return nil, fmt.Errorf("chain id: %w", err)
	}
	if got.Cmp(want) != 0 {
		c.Close()
		return nil, fmt.Errorf("%w: endpoint is %s, expected %s", ErrWrongChain, got, want)
	}
	return c, nil
}

func main() {
	url, stop := endpoint()
	defer stop()

	// --- failure 1: nothing is listening -----------------------------------
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	dead := fmt.Sprintf("http://127.0.0.1:%d", freePort()) // a port we just released
	_, err := connect(ctx, dead, big.NewInt(1337))
	var netErr net.Error
	fmt.Println("1. nothing listening")
	fmt.Printf("   failed=%v  net.Error=%v  wrongChain=%v\n",
		err != nil, errors.As(err, &netErr), errors.Is(err, ErrWrongChain))
	fmt.Println("   -> your URL, your port, or the node is down. Not retryable by itself.")

	// --- failure 2: right endpoint, wrong network --------------------------
	_, err = connect(ctx, url, big.NewInt(1)) // expect mainnet, get the dev chain
	fmt.Println("\n2. connected, but the wrong chain")
	fmt.Printf("   failed=%v  wrongChain=%v\n", err != nil, errors.Is(err, ErrWrongChain))
	fmt.Printf("   %v\n", err)
	fmt.Println("   -> catch this at startup, never at signing time (lesson 19)")

	// --- failure 3: the deadline ran out -----------------------------------
	tight, cancelTight := context.WithTimeout(context.Background(), time.Nanosecond)
	defer cancelTight()
	time.Sleep(time.Millisecond)

	_, err = connect(tight, url, big.NewInt(1337))
	fmt.Println("\n3. deadline exceeded")
	fmt.Printf("   failed=%v  DeadlineExceeded=%v\n",
		err != nil, errors.Is(err, context.DeadlineExceeded))
	fmt.Println("   -> maybe retryable with backoff, inside a longer budget (lesson 35)")

	// --- and the happy path ------------------------------------------------
	client, err := connect(ctx, url, big.NewInt(1337))
	if err != nil {
		fmt.Println("\nunexpected:", err)
		os.Exit(1)
	}
	defer client.Close()
	head, _ := client.BlockNumber(ctx)
	fmt.Printf("\n4. connected and verified: chain 1337, head %d\n", head)
	fmt.Println("\nthree failure modes, three different responses — classify before you retry")
}
```

**Output:**

```
1. nothing listening
   failed=true  net.Error=true  wrongChain=false
   -> your URL, your port, or the node is down. Not retryable by itself.

2. connected, but the wrong chain
   failed=true  wrongChain=true
   wrong chain: endpoint is 1337, expected 1
   -> catch this at startup, never at signing time (lesson 19)

3. deadline exceeded
   failed=true  DeadlineExceeded=true
   -> maybe retryable with backoff, inside a longer budget (lesson 35)

4. connected and verified: chain 1337, head 0

three failure modes, three different responses — classify before you retry
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
