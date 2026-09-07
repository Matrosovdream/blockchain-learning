# Step 09 — Proof of Work & Mining · 🔴 Hard

Examples **14–18**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch       # first time only
# paste the example into main.go, then:
go run .
```

**Standard library only.** Simulations use a seeded RNG and mining reports *hash counts* rather than
elapsed time, so every output reproduces exactly.

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md) · Next: [the index](README.md)

---

## 14. Parallel mining

`🔴 hard` · *Concurrency*

Shard the nonce space across `NumCPU` workers with a strided split, and cancel the losers through a `context` when one wins. Three details matter: a buffered result channel, `cancel()` before `wg.Wait()`, and a cancellation check that is periodic rather than per-hash.

**Steps:**

1. Give worker w the nonces w, w+N, w+2N — striding keeps everyone busy.
2. Buffer the result channel for every worker, so a late finisher cannot block.
3. Cancel, then wait, so nothing outlives the function.
4. Note the specific nonce is not printed: which worker wins depends on the scheduler, and *many* nonces satisfy any target.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/big"
	"runtime"
	"sync"
)

const headerSize = 92
const noncePos = 80

func headerBytes(ts int64, height uint64) []byte {
	b := make([]byte, headerSize)
	binary.BigEndian.PutUint32(b[0:4], 1)
	binary.BigEndian.PutUint64(b[68:76], uint64(ts))
	binary.BigEndian.PutUint64(b[84:92], height)
	return b
}

func hashOf(b []byte) [32]byte {
	f := sha256.Sum256(b)
	return sha256.Sum256(f[:])
}

type result struct {
	nonce  uint32
	worker int
}

// MineParallel shards the nonce space across workers. The first to find a
// solution cancels the rest through the context.
func MineParallel(ts int64, height uint64, target *big.Int, workers int) (result, bool) {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// BUFFERED, with room for every worker. If this were unbuffered, a worker
	// that finds a solution after the winner has been read would block forever
	// on the send — and leak (example 15).
	results := make(chan result, workers)

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func(w int) {
			defer wg.Done()

			buf := headerBytes(ts, height)
			h1, h2 := sha256.New(), sha256.New()
			d1 := make([]byte, 0, sha256.Size)
			d2 := make([]byte, 0, sha256.Size)
			var val big.Int

			// Disjoint ranges: worker w takes nonces w, w+workers, w+2*workers...
			// Striding beats splitting into blocks, because it keeps every
			// worker busy even if a solution sits early in the space.
			for nonce := uint32(w); ; nonce += uint32(workers) {
				// Check for cancellation periodically, not every iteration —
				// a select on every hash would dominate the loop.
				if nonce%4096 == uint32(w) {
					select {
					case <-ctx.Done():
						return
					default:
					}
				}
				binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], nonce)
				h1.Reset()
				h1.Write(buf)
				d1 = h1.Sum(d1[:0])
				h2.Reset()
				h2.Write(d1)
				d2 = h2.Sum(d2[:0])

				if val.SetBytes(d2).Cmp(target) < 0 {
					results <- result{nonce: nonce, worker: w}
					return
				}
			}
		}(w)
	}

	first, ok := <-results
	cancel()  // tell the losers to stop
	wg.Wait() // and wait for them, so nothing outlives this function
	return first, ok
}

func main() {
	target := new(big.Int).Lsh(big.NewInt(1), 256-22)
	const ts, height = 1700000000, 1

	workers := runtime.NumCPU()
	fmt.Printf("mining with %d workers (runtime.NumCPU)\n", workers)
	fmt.Println("each takes a strided slice of the nonce space: w, w+N, w+2N, ...")

	res, ok := MineParallel(ts, height, target, workers)
	if !ok {
		fmt.Println("no solution")
		return
	}

	// Verify independently.
	buf := headerBytes(ts, height)
	binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], res.nonce)
	sum := hashOf(buf)
	valid := new(big.Int).SetBytes(sum[:]).Cmp(target) < 0

	// NOTE: which worker wins, and therefore which nonce, depends on OS
	// scheduling — so this prints whether it worked, not which one it was.
	fmt.Printf("\nfound a solution: %v\n", ok)
	fmt.Printf("verifies:         %v\n", valid)
	fmt.Printf("nonce lies in the winning worker's stride: %v\n",
		res.nonce%uint32(workers) == uint32(res.worker))

	fmt.Println("\nwhy the specific nonce is not printed here")
	fmt.Println("  it differs between runs, because which worker gets there first")
	fmt.Println("  depends on the scheduler. That is fine: MANY nonces satisfy any")
	fmt.Println("  target, and a block needs one of them, not a particular one.")

	// The goroutine accounting.
	fmt.Printf("\ngoroutines still running: %d (main + runtime only)\n", runtime.NumGoroutine())
	fmt.Println("  cancel() then wg.Wait() guarantees every worker has returned")
	fmt.Println("  before MineParallel does. Without the Wait, the function could")
	fmt.Println("  return while workers still burn CPU on a block already solved.")

	fmt.Println("\nthree things this loop gets right")
	fmt.Println("  1. buffered result channel, sized for every worker")
	fmt.Println("  2. cancel() before wg.Wait(), so the losers are told to stop")
	fmt.Println("  3. the ctx check is periodic, not per-hash — a select on every")
	fmt.Println("     iteration would cost more than the hashing")

	fmt.Println("\nand the speedup is sub-linear: memory bandwidth, shared caches")
	fmt.Println("and thermal limits mean 8 cores do not give 8x. Measure, do not")
	fmt.Println("assume (lesson 57).")
}
```

**Output:**

```
mining with 10 workers (runtime.NumCPU)
each takes a strided slice of the nonce space: w, w+N, w+2N, ...

found a solution: true
verifies:         true
nonce lies in the winning worker's stride: true

why the specific nonce is not printed here
  it differs between runs, because which worker gets there first
  depends on the scheduler. That is fine: MANY nonces satisfy any
  target, and a block needs one of them, not a particular one.

goroutines still running: 1 (main + runtime only)
  cancel() then wg.Wait() guarantees every worker has returned
  before MineParallel does. Without the Wait, the function could
  return while workers still burn CPU on a block already solved.

three things this loop gets right
  1. buffered result channel, sized for every worker
  2. cancel() before wg.Wait(), so the losers are told to stop
  3. the ctx check is periodic, not per-hash — a select on every
     iteration would cost more than the hashing

and the speedup is sub-linear: memory bandwidth, shared caches
and thermal limits mean 8 cores do not give 8x. Measure, do not
assume (lesson 57).
```

---

## 15. The goroutine leak

`🔴 hard` · *Concurrency*

The same miner with two ordinary mistakes: an unbuffered result channel, and returning without waiting. A second worker that finds a solution blocks forever on the send. Twenty calls leak steadily — and each leaked goroutine is still hashing.

**Steps:**

1. Run the leaky version 20 times and check `runtime.NumGoroutine`.
2. Run the safe version 20 times and confirm no growth.
3. Read the two fixes: size the channel for every sender, and `cancel()` then `wg.Wait()`.
4. Note the leak *count* is scheduler-dependent, so only the fact of it is asserted.
5. Learn the tools: `NumGoroutine`, `-race`, `goleak`, and pprof's goroutine profile.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/big"
	"runtime"
	"sync"
	"time"
)

const headerSize = 92
const noncePos = 80

func headerBytes(ts int64) []byte {
	b := make([]byte, headerSize)
	binary.BigEndian.PutUint32(b[0:4], 1)
	binary.BigEndian.PutUint64(b[68:76], uint64(ts))
	return b
}

func mineRange(ctx context.Context, ts int64, start uint32, stride int,
	target *big.Int, out chan<- uint32) {
	buf := headerBytes(ts)
	h1, h2 := sha256.New(), sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var val big.Int

	for nonce := start; ; nonce += uint32(stride) {
		if nonce%2048 == start%2048 {
			select {
			case <-ctx.Done():
				return
			default:
			}
		}
		binary.BigEndian.PutUint32(buf[noncePos:noncePos+4], nonce)
		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])
		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])
		if val.SetBytes(d2).Cmp(target) < 0 {
			out <- nonce // BLOCKS if the channel is unbuffered and nobody reads
			return
		}
	}
}

// leaky uses an UNBUFFERED channel and never drains it. The winner is read;
// any other worker that also finds a solution blocks forever on the send.
func leaky(ts int64, target *big.Int, workers int) uint32 {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	out := make(chan uint32) // <- the bug: capacity 0
	for w := 0; w < workers; w++ {
		go mineRange(ctx, ts, uint32(w), workers, target, out)
	}
	return <-out // read ONE, then return. The rest are on their own.
}

// safe buffers the channel for every worker and waits for them to finish.
func safe(ts int64, target *big.Int, workers int) uint32 {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	out := make(chan uint32, workers) // room for everyone
	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func(w int) {
			defer wg.Done()
			mineRange(ctx, ts, uint32(w), workers, target, out)
		}(w)
	}
	first := <-out
	cancel()
	wg.Wait() // nothing outlives this function
	return first
}

func main() {
	// An easy target, so several workers find solutions at nearly the same time
	// and the race is actually exercised.
	target := new(big.Int).Lsh(big.NewInt(1), 256-10)
	const workers = 8

	base := runtime.NumGoroutine()
	// The exact leak COUNT depends on how many workers happen to finish before
	// cancellation, which is scheduler-dependent. The FACT of the leak is not.
	fmt.Printf("goroutines at start: %d\n", base)

	// --- the leaky version --------------------------------------------------
	for i := 0; i < 20; i++ {
		leaky(int64(1700000000+i), target, workers)
	}
	time.Sleep(50 * time.Millisecond) // let anything that CAN exit, exit
	afterLeaky := runtime.NumGoroutine()
	fmt.Printf("\nafter 20 leaky() calls with %d workers each\n", workers)
	fmt.Printf("  goroutines leaked: %v\n", afterLeaky > base)
	fmt.Println("  these are blocked forever on `out <- nonce`, holding their")
	fmt.Println("  buffers and hashers. They will never be collected.")

	// --- the safe version ---------------------------------------------------
	for i := 0; i < 20; i++ {
		safe(int64(1700000000+i), target, workers)
	}
	time.Sleep(50 * time.Millisecond)
	afterSafe := runtime.NumGoroutine()
	fmt.Printf("\nafter 20 safe() calls\n")
	fmt.Printf("  goroutines leaked: %v\n", afterSafe > afterLeaky)
	fmt.Println("  every worker returned before safe() did.")

	fmt.Println("\nthe two mistakes, and their fixes")
	fmt.Println("  1. unbuffered result channel")
	fmt.Println("     a second finisher blocks on send forever.")
	fmt.Println("     fix: make(chan T, workers) — room for every possible sender")
	fmt.Println("  2. returning without waiting")
	fmt.Println("     cancel() only ASKS; the workers may still be mid-loop.")
	fmt.Println("     fix: cancel() then wg.Wait() before returning")

	fmt.Println("\nwhy a miner makes this so visible")
	fmt.Println("  a new block arrives every few seconds during sync, so this")
	fmt.Println("  function runs constantly. A leak of 7 goroutines per call is")
	fmt.Println("  thousands within a minute — each still hashing, so the machine")
	fmt.Println("  slows down as well as growing.")

	fmt.Println("\nhow to catch it")
	fmt.Println("  - runtime.NumGoroutine() before and after, as above")
	fmt.Println("  - go test -race for the concurrent access")
	fmt.Println("  - go.uber.org/goleak in tests, which fails on any straggler")
	fmt.Println("  - pprof's goroutine profile, which shows exactly where they block")
}
```

**Output:**

```
goroutines at start: 1

after 20 leaky() calls with 8 workers each
  goroutines leaked: true
  these are blocked forever on `out <- nonce`, holding their
  buffers and hashers. They will never be collected.

after 20 safe() calls
  goroutines leaked: false
  every worker returned before safe() did.

the two mistakes, and their fixes
  1. unbuffered result channel
     a second finisher blocks on send forever.
     fix: make(chan T, workers) — room for every possible sender
  2. returning without waiting
     cancel() only ASKS; the workers may still be mid-loop.
     fix: cancel() then wg.Wait() before returning

why a miner makes this so visible
  a new block arrives every few seconds during sync, so this
  function runs constantly. A leak of 7 goroutines per call is
  thousands within a minute — each still hashing, so the machine
  slows down as well as growing.

how to catch it
  - runtime.NumGoroutine() before and after, as above
  - go test -race for the concurrent access
  - go.uber.org/goleak in tests, which fails on any straggler
  - pprof's goroutine profile, which shows exactly where they block
```

---

## 16. Simulating a 51% double-spend

`🔴 hard` · *Attacks*

A double-spend simulated directly: the attacker waits for z confirmations, then mines a private branch that omits the payment. Below 50% more confirmations kill it quickly; at 51% they only buy time.

**Steps:**

1. Simulate 20,000 attempts per cell with a seeded RNG.
2. Compare the results against the theory from example 12.
3. Read the real cases: Bitcoin Gold 2018, Ethereum Classic 2019 and 2020.
4. Understand why those chains and not Bitcoin — rented hashrate.
5. Learn precisely what a 51% attacker can and cannot do.

```go
package main

import (
	"fmt"
	"math/rand"
)

// A 51% double-spend, simulated. The attacker sends a payment, waits for it to
// reach z confirmations, and mines a competing branch in private that omits it.
// If the private branch ever overtakes the public one, the payment is undone.

type outcome struct {
	succeeded   bool
	blocksMined int
}

// attempt runs one double-spend attempt with hashrate share q, needing to
// overtake a z-block lead. It gives up after maxBlocks of public progress.
func attempt(rng *rand.Rand, q float64, z, maxBlocks int) outcome {
	attacker, honest := 0, z // the attacker starts z blocks behind
	mined := 0

	for honest-attacker < maxBlocks {
		mined++
		if rng.Float64() < q {
			attacker++
		} else {
			honest++
		}
		if attacker >= honest {
			return outcome{true, mined}
		}
		if mined > 200000 {
			break // give up
		}
	}
	return outcome{false, mined}
}

func main() {
	const trials = 20000
	rng := rand.New(rand.NewSource(7)) // seeded: reproducible

	fmt.Printf("%d simulated double-spend attempts per cell\n", trials)
	fmt.Println("(attacker abandons if it falls 100 blocks behind)")
	fmt.Println()

	qs := []float64{0.10, 0.25, 0.35, 0.45, 0.51}
	fmt.Printf("%-16s", "confirmations")
	for _, q := range qs {
		fmt.Printf("%10s", fmt.Sprintf("q=%.0f%%", q*100))
	}
	fmt.Println()

	for _, z := range []int{1, 2, 3, 6, 12} {
		fmt.Printf("%-16d", z)
		for _, q := range qs {
			wins := 0
			for t := 0; t < trials; t++ {
				if attempt(rng, q, z, 100).succeeded {
					wins++
				}
			}
			fmt.Printf("%10s", fmt.Sprintf("%.2f%%", float64(wins)*100/trials))
		}
		fmt.Println()
	}

	fmt.Println("\nthe shape of it")
	fmt.Println("  below 50%, more confirmations rapidly kill the attack")
	fmt.Println("  at 51%, confirmations only buy TIME — the attacker wins")
	fmt.Println("  eventually, because their branch grows faster on average")
	fmt.Println("  (this simulation cuts them off at 100 blocks behind, which is")
	fmt.Println("  why the 51% column is not 100%)")

	// The real cases.
	fmt.Println("\nthis has happened")
	fmt.Println("  Bitcoin Gold, May 2018    ~$18M double-spent; attacked again in 2020")
	fmt.Println("  Ethereum Classic, Jan 2019  ~$1.1M; then TWICE more in Aug 2020,")
	fmt.Println("                              one reorg over 7,000 blocks deep")
	fmt.Println("  Verge, Bitcoin SV, and others have all seen deep reorgs")

	fmt.Println("\nwhy those chains and not Bitcoin")
	fmt.Println("  they shared a hash algorithm with a much larger chain, so")
	fmt.Println("  attack hashrate could be RENTED rather than built. Renting an")
	fmt.Println("  hour of hashrate exceeding a small chain's entire network cost")
	fmt.Println("  a few thousand dollars (example 13).")

	fmt.Println("\nwhat a 51% attacker can and cannot do")
	fmt.Println("  CAN     reorder or exclude transactions; double-spend their own")
	fmt.Println("          coins; censor; mine every block")
	fmt.Println("  CANNOT  spend coins they have no key for (lesson 06)")
	fmt.Println("  CANNOT  create coins out of thin air or change the subsidy")
	fmt.Println("  CANNOT  make an invalid block valid — every node still checks")
	fmt.Println("          every rule (lesson 08)")
	fmt.Println("\n  proof of work secures the ORDER of history, and nothing else.")

	fmt.Println("\nthe defences that actually work")
	fmt.Println("  - deeper confirmations, scaled to the value and the chain")
	fmt.Println("  - watching for deep reorgs and halting deposits (lesson 31, 35)")
	fmt.Println("  - not being a small chain sharing an algorithm with a large one")
}
```

**Output:**

```
20000 simulated double-spend attempts per cell
(attacker abandons if it falls 100 blocks behind)

confirmations        q=10%     q=25%     q=35%     q=45%     q=51%
1                   11.15%    32.90%    53.16%    81.64%    99.90%
2                    1.18%    11.06%    28.93%    67.19%    99.84%
3                    0.10%     3.83%    15.52%    54.67%    99.69%
6                    0.01%     0.09%     2.42%    30.36%    99.46%
12                   0.00%     0.00%     0.08%     9.08%    98.89%

the shape of it
  below 50%, more confirmations rapidly kill the attack
  at 51%, confirmations only buy TIME — the attacker wins
  eventually, because their branch grows faster on average
  (this simulation cuts them off at 100 blocks behind, which is
  why the 51% column is not 100%)

this has happened
  Bitcoin Gold, May 2018    ~$18M double-spent; attacked again in 2020
  Ethereum Classic, Jan 2019  ~$1.1M; then TWICE more in Aug 2020,
                              one reorg over 7,000 blocks deep
  Verge, Bitcoin SV, and others have all seen deep reorgs

why those chains and not Bitcoin
  they shared a hash algorithm with a much larger chain, so
  attack hashrate could be RENTED rather than built. Renting an
  hour of hashrate exceeding a small chain's entire network cost
  a few thousand dollars (example 13).

what a 51% attacker can and cannot do
  CAN     reorder or exclude transactions; double-spend their own
          coins; censor; mine every block
  CANNOT  spend coins they have no key for (lesson 06)
  CANNOT  create coins out of thin air or change the subsidy
  CANNOT  make an invalid block valid — every node still checks
          every rule (lesson 08)

  proof of work secures the ORDER of history, and nothing else.

the defences that actually work
  - deeper confirmations, scaled to the value and the chain
  - watching for deep reorgs and halting deposits (lesson 31, 35)
  - not being a small chain sharing an algorithm with a large one
```

---

## 17. Selfish mining

`🔴 hard` · *Attacks*

Eyal & Sirer's result: a miner who withholds blocks can earn **more** than their fair share. This implements the SM1 state machine faithfully, and reproduces the published thresholds — ⅓ when the attacker loses every tie, ¼ when they win half, and profitable at any share when they win them all.

**Steps:**

1. Model the private lead, plus the 0' tie state where two tips compete.
2. Run 500,000 events at seven hashrate shares and three gamma values.
3. Check the thresholds against the paper: 1/3, 1/4, and ~0.
4. Understand why this lowered the practical safety threshold from 50% to nearer 25–33%.
5. Read what makes it hard in practice, and why fast propagation is the mitigation.

```go
package main

import (
	"fmt"
	"math/rand"
)

// Selfish mining (Eyal & Sirer, 2013). A miner who finds a block does not
// publish it. They mine privately on their own lead, and release blocks only
// to invalidate the honest network's work — wasting it. Below 50% hashrate,
// this can earn MORE than their fair share.

// simulate implements the SM1 state machine from Eyal & Sirer exactly.
//
// State is the selfish miner's private lead, plus a special "0'" state: a
// public tie, where the selfish miner has published a competing block and the
// honest network is split between the two tips.
//
// alpha = the selfish miner's hashrate share.
// gamma = the fraction of honest miners mining on the SELFISH tip during a tie.
func simulate(rng *rand.Rand, alpha, gamma float64, rounds int) (selfish, honest int) {
	lead := 0
	tie := false // true means we are in state 0'

	for i := 0; i < rounds; i++ {
		selfishFound := rng.Float64() < alpha

		if tie {
			// Two competing tips of equal height. Whoever extends one wins it.
			switch {
			case selfishFound:
				// Selfish extends its own tip: its block plus this one win.
				selfish += 2
			case rng.Float64() < gamma:
				// An honest miner on the SELFISH tip extends it: the selfish
				// block stands, and the honest miner gets the new one.
				selfish++
				honest++
			default:
				// An honest miner on the honest tip extends it: honest wins both.
				honest += 2
			}
			lead, tie = 0, false
			continue
		}

		if selfishFound {
			lead++
			continue
		}

		// An honest block was found.
		switch lead {
		case 0:
			// Nothing withheld: the honest block simply stands.
			honest++
		case 1:
			// Selfish publishes its single block to compete -> a tie.
			tie = true
		case 2:
			// Publishing both now orphans the honest block outright.
			selfish += 2
			lead = 0
		default:
			// A long lead: publish one to stay ahead, orphaning the honest block.
			selfish++
			lead--
		}
	}
	return selfish, honest
}

func main() {
	const rounds = 500000
	rng := rand.New(rand.NewSource(11)) // seeded: reproducible

	fmt.Printf("selfish mining over %d block-discovery events\n", rounds)
	fmt.Println("gamma = fraction of honest miners who build on the selfish block in a tie")
	fmt.Println()

	for _, gamma := range []float64{0.0, 0.5, 1.0} {
		fmt.Printf("gamma = %.1f\n", gamma)
		fmt.Printf("  %-10s %-14s %-14s %s\n", "hashrate", "fair share", "actual share", "profitable?")
		for _, alpha := range []float64{0.10, 0.20, 0.25, 0.30, 0.33, 0.40, 0.45} {
			s, h := simulate(rng, alpha, gamma, rounds)
			share := float64(s) / float64(s+h)
			mark := "no"
			if share > alpha+0.002 {
				mark = "YES"
			}
			fmt.Printf("  %-10s %-14s %-14s %s\n",
				fmt.Sprintf("%.0f%%", alpha*100),
				fmt.Sprintf("%.1f%%", alpha*100),
				fmt.Sprintf("%.1f%%", share*100), mark)
		}
		fmt.Println()
	}

	fmt.Println("reading it")
	fmt.Println("  gamma=0    the selfish miner always loses ties. Profitable only")
	fmt.Println("             above 1/3 of the hashrate — the published threshold.")
	fmt.Println("  gamma=0.5  half the network sees the selfish block first —")
	fmt.Println("             the threshold drops to around 25%.")
	fmt.Println("  gamma=1    the selfish miner wins every tie through network")
	fmt.Println("             position alone, and profits from almost any share.")

	fmt.Println("\nwhy this result mattered")
	fmt.Println("  the assumption before 2013 was that honest mining is optimal for")
	fmt.Println("  any miner below 50%. Eyal and Sirer showed that is false: a")
	fmt.Println("  well-connected miner with a THIRD of the hashrate can do better")
	fmt.Println("  by withholding. That lowers the practical safety threshold from")
	fmt.Println("  50% to somewhere nearer 25-33%.")

	fmt.Println("\nwhat makes it hard in practice")
	fmt.Println("  - it is publicly detectable: an unusual orphan rate points at it")
	fmt.Println("  - it damages confidence in the coin the attacker is paid in")
	fmt.Println("  - real block propagation is fast, pushing gamma low")
	fmt.Println("  - there is no clear evidence of it being run on Bitcoin at scale")

	fmt.Println("\nand the mitigation")
	fmt.Println("  faster, more uniform propagation lowers gamma — which is part of")
	fmt.Println("  why compact block relay and dedicated relay networks exist")
	fmt.Println("  (lesson 13). Tie-breaking rules that pick randomly rather than")
	fmt.Println("  first-seen also help.")
}
```

**Output:**

```
selfish mining over 500000 block-discovery events
gamma = fraction of honest miners who build on the selfish block in a tie

gamma = 0.0
  hashrate   fair share     actual share   profitable?
  10%        10.0%          3.6%           no
  20%        20.0%          13.0%          no
  25%        25.0%          19.5%          no
  30%        30.0%          27.2%          no
  33%        33.0%          32.7%          no
  40%        40.0%          48.4%          YES
  45%        45.0%          65.4%          YES

gamma = 0.5
  hashrate   fair share     actual share   profitable?
  10%        10.0%          7.2%           no
  20%        20.0%          18.3%          no
  25%        25.0%          25.1%          no
  30%        30.0%          32.8%          YES
  33%        33.0%          37.8%          YES
  40%        40.0%          52.5%          YES
  45%        45.0%          67.7%          YES

gamma = 1.0
  hashrate   fair share     actual share   profitable?
  10%        10.0%          10.9%          YES
  20%        20.0%          23.5%          YES
  25%        25.0%          30.4%          YES
  30%        30.0%          38.2%          YES
  33%        33.0%          43.1%          YES
  40%        40.0%          57.0%          YES
  45%        45.0%          71.4%          YES

reading it
  gamma=0    the selfish miner always loses ties. Profitable only
             above 1/3 of the hashrate — the published threshold.
  gamma=0.5  half the network sees the selfish block first —
             the threshold drops to around 25%.
  gamma=1    the selfish miner wins every tie through network
             position alone, and profits from almost any share.

why this result mattered
  the assumption before 2013 was that honest mining is optimal for
  any miner below 50%. Eyal and Sirer showed that is false: a
  well-connected miner with a THIRD of the hashrate can do better
  by withholding. That lowers the practical safety threshold from
  50% to somewhere nearer 25-33%.

what makes it hard in practice
  - it is publicly detectable: an unusual orphan rate points at it
  - it damages confidence in the coin the attacker is paid in
  - real block propagation is fast, pushing gamma low
  - there is no clear evidence of it being run on Bitcoin at scale

and the mitigation
  faster, more uniform propagation lowers gamma — which is part of
  why compact block relay and dedicated relay networks exist
  (lesson 13). Tie-breaking rules that pick randomly rather than
  first-seen also help.
```

---

## 18. Proof of work in the chain

`🔴 hard` · *Assembly*

Lesson 08's chain with proof of work added, and the diff is deliberately small: a `CheckPoW` line, a `Mine` loop, a `NextBits` retarget rule, and two new rejection paths. Everything else is unchanged — which is the payoff for having built the skeleton first.

**Steps:**

1. Read the four additions marked NEW against lesson 08's code.
2. Mine 16 blocks, fast then slow, and watch two retargets fire.
3. Reject a block with a tampered nonce (`ErrBadPoW`).
4. Reject a block with *genuine* proof of work at the wrong difficulty (`ErrBadBits`).
5. Read what lesson 10 changes next: `Body` becomes real transactions.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"errors"
	"fmt"
	"math/big"
	"sort"
)

// ===========================================================================
// Lesson 08's chain, with proof of work added. The diff is small and precise:
//
//   + Bits is now a real target, checked in CheckStateless
//   + Mine() grinds Nonce until the header hash is below that target
//   + the retarget rule decides each block's Bits from its parent
//
// Everything else — the header layout, Merkle root, genesis, the two
// validation passes, validate-then-mutate — is unchanged from lesson 08.
// ===========================================================================

const (
	HeaderVersion  = 1
	HeaderSize     = 92
	MTPWindow      = 11
	targetSpacing  = 60 // seconds; short so the demo retargets quickly
	retargetBlocks = 8  // Bitcoin uses 2016
	targetTimespan = targetSpacing * retargetBlocks
)

type Header struct {
	Version    uint32
	PrevHash   [32]byte
	MerkleRoot [32]byte
	Timestamp  int64
	Bits       uint32
	Nonce      uint32
	Height     uint64
}

func (h Header) Bytes() []byte {
	buf := bytes.NewBuffer(make([]byte, 0, HeaderSize))
	binary.Write(buf, binary.BigEndian, h.Version)
	buf.Write(h.PrevHash[:])
	buf.Write(h.MerkleRoot[:])
	binary.Write(buf, binary.BigEndian, h.Timestamp)
	binary.Write(buf, binary.BigEndian, h.Bits)
	binary.Write(buf, binary.BigEndian, h.Nonce)
	binary.Write(buf, binary.BigEndian, h.Height)
	return buf.Bytes()
}

func (h Header) Hash() [32]byte {
	f := sha256.Sum256(h.Bytes())
	return sha256.Sum256(f[:])
}

// ------------------------------------------------------------ proof of work

func BitsToTarget(bits uint32) *big.Int {
	exp := bits >> 24
	m := bits & 0x007fffff
	if exp <= 3 {
		return new(big.Int).Rsh(big.NewInt(int64(m)), uint(8*(3-exp)))
	}
	return new(big.Int).Lsh(big.NewInt(int64(m)), uint(8*(exp-3)))
}

func TargetToBits(t *big.Int) uint32 {
	if t.Sign() == 0 {
		return 0
	}
	b := t.Bytes()
	exp := len(b)
	var m uint32
	if exp <= 3 {
		for i := 0; i < len(b); i++ {
			m = m<<8 | uint32(b[i])
		}
		m <<= uint(8 * (3 - exp))
	} else {
		m = uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
	}
	if m&0x00800000 != 0 {
		m >>= 8
		exp++
	}
	return uint32(exp)<<24 | m
}

// A deliberately easy maximum for the demo, so it mines in seconds.
var maxTarget = BitsToTarget(0x2000ffff)

// CheckPoW: one hash, one comparison (example 2).
func CheckPoW(h Header) bool {
	target := BitsToTarget(h.Bits)
	if target.Sign() <= 0 || target.Cmp(maxTarget) > 0 {
		return false // below the minimum difficulty the chain allows
	}
	sum := h.Hash()
	return new(big.Int).SetBytes(sum[:]).Cmp(target) < 0
}

// Mine grinds the nonce. Allocation-free inner loop (example 7).
func Mine(h Header) (Header, uint64) {
	target := BitsToTarget(h.Bits)
	buf := h.Bytes()
	h1, h2 := sha256.New(), sha256.New()
	d1 := make([]byte, 0, sha256.Size)
	d2 := make([]byte, 0, sha256.Size)
	var val big.Int

	for nonce := uint32(0); ; nonce++ {
		binary.BigEndian.PutUint32(buf[80:84], nonce)
		h1.Reset()
		h1.Write(buf)
		d1 = h1.Sum(d1[:0])
		h2.Reset()
		h2.Write(d1)
		d2 = h2.Sum(d2[:0])
		if val.SetBytes(d2).Cmp(target) < 0 {
			h.Nonce = nonce
			return h, uint64(nonce) + 1
		}
	}
}

// ------------------------------------------------------------------- chain

const tagLeaf, tagNode byte = 0x00, 0x01

func MerkleRoot(txs []string) [32]byte {
	if len(txs) == 0 {
		return sha256.Sum256([]byte{tagLeaf})
	}
	level := make([][32]byte, len(txs))
	for i, t := range txs {
		level[i] = sha256.Sum256(append([]byte{tagLeaf}, t...))
	}
	for len(level) > 1 {
		var next [][32]byte
		for i := 0; i < len(level); i += 2 {
			if i+1 == len(level) {
				next = append(next, level[i])
				continue
			}
			b := append([]byte{tagNode}, level[i][:]...)
			next = append(next, sha256.Sum256(append(b, level[i+1][:]...)))
		}
		level = next
	}
	return level[0]
}

type Block struct {
	Header Header
	Body   []string
}

var (
	ErrBadVersion = errors.New("unsupported version")
	ErrEmptyBody  = errors.New("empty body")
	ErrBadMerkle  = errors.New("merkle root mismatch")
	ErrBadPoW     = errors.New("insufficient proof of work") // NEW in lesson 09
	ErrOrphan     = errors.New("parent not found")
	ErrBadHeight  = errors.New("height is not parent+1")
	ErrBadBits    = errors.New("bits do not match the retarget rule") // NEW
	ErrTooOld     = errors.New("timestamp at or before median-time-past")
)

func CheckStateless(b Block) error {
	if b.Header.Version != HeaderVersion {
		return fmt.Errorf("%w: %d", ErrBadVersion, b.Header.Version)
	}
	if len(b.Body) == 0 {
		return ErrEmptyBody
	}
	if MerkleRoot(b.Body) != b.Header.MerkleRoot {
		return ErrBadMerkle
	}
	if !CheckPoW(b.Header) { // <- the whole of lesson 09, in one line
		return ErrBadPoW
	}
	return nil
}

type Chain struct {
	byHash map[[32]byte]Block
	order  [][32]byte
}

func Genesis() Block {
	var zero [32]byte
	body := []string{"genesis"}
	h := Header{Version: HeaderVersion, PrevHash: zero, MerkleRoot: MerkleRoot(body),
		Timestamp: 1700000000, Bits: 0x2000ffff, Height: 0}
	mined, _ := Mine(h)
	return Block{Header: mined, Body: body}
}

func NewChain() *Chain {
	g := Genesis()
	c := &Chain{byHash: map[[32]byte]Block{}}
	hh := g.Header.Hash()
	c.byHash[hh] = g
	c.order = append(c.order, hh)
	return c
}

func (c *Chain) Len() int       { return len(c.order) }
func (c *Chain) Tip() Block     { return c.byHash[c.order[len(c.order)-1]] }
func (c *Chain) At(i int) Block { return c.byHash[c.order[i]] }

func (c *Chain) MedianTimePast() int64 {
	n := MTPWindow
	if len(c.order) < n {
		n = len(c.order)
	}
	ts := make([]int64, 0, n)
	for _, h := range c.order[len(c.order)-n:] {
		ts = append(ts, c.byHash[h].Header.Timestamp)
	}
	sort.Slice(ts, func(i, j int) bool { return ts[i] < ts[j] })
	return ts[len(ts)/2]
}

// NextBits applies the retarget rule (example 9). Every node computes this
// independently, so a block whose Bits disagree is invalid.
func (c *Chain) NextBits() uint32 {
	parent := c.Tip().Header
	next := parent.Height + 1
	if next%retargetBlocks != 0 {
		return parent.Bits // unchanged within a period
	}
	first := c.At(len(c.order) - retargetBlocks).Header
	actual := parent.Timestamp - first.Timestamp

	clamped := actual
	if clamped < targetTimespan/4 {
		clamped = targetTimespan / 4
	}
	if clamped > targetTimespan*4 {
		clamped = targetTimespan * 4
	}
	t := new(big.Int).Mul(BitsToTarget(parent.Bits), big.NewInt(clamped))
	t.Div(t, big.NewInt(targetTimespan))
	if t.Cmp(maxTarget) > 0 {
		t.Set(maxTarget)
	}
	return TargetToBits(t)
}

func (c *Chain) Append(b Block) error {
	if err := CheckStateless(b); err != nil {
		return err
	}
	parent, ok := c.byHash[b.Header.PrevHash]
	if !ok {
		return fmt.Errorf("%w: %x", ErrOrphan, b.Header.PrevHash[:4])
	}
	if b.Header.Height != parent.Header.Height+1 {
		return fmt.Errorf("%w: %d after %d", ErrBadHeight, b.Header.Height, parent.Header.Height)
	}
	if want := c.NextBits(); b.Header.Bits != want {
		return fmt.Errorf("%w: got %#x, want %#x", ErrBadBits, b.Header.Bits, want)
	}
	if mtp := c.MedianTimePast(); b.Header.Timestamp <= mtp {
		return fmt.Errorf("%w: %d <= %d", ErrTooOld, b.Header.Timestamp, mtp)
	}
	h := b.Header.Hash()
	c.byHash[h] = b
	c.order = append(c.order, h)
	return nil
}

// MineNext builds and mines a valid successor to the tip.
func (c *Chain) MineNext(txs []string, elapsed int64) (Block, uint64) {
	p := c.Tip().Header
	h := Header{
		Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot(txs),
		Timestamp: p.Timestamp + elapsed, Bits: c.NextBits(), Height: p.Height + 1,
	}
	mined, hashes := Mine(h)
	return Block{Header: mined, Body: txs}, hashes
}

// -------------------------------------------------------------------- demo

func short(h [32]byte) string { return hex.EncodeToString(h[:8]) }

func main() {
	c := NewChain()
	fmt.Printf("genesis  height 0  bits %#x  hash %s\n\n", c.Tip().Header.Bits, short(c.Tip().Header.Hash()))

	// Mine 16 blocks. Blocks 0-7 arrive fast (30s apart), so the first retarget
	// makes things harder; blocks 8-15 arrive slowly, and it eases off again.
	fmt.Printf("%-8s %-10s %-12s %-18s %s\n", "height", "spacing", "bits", "hash", "hashes to mine")
	for i := 1; i <= 16; i++ {
		spacing := int64(30) // twice as fast as the 60s target
		if i > 8 {
			spacing = 120 // then twice as slow
		}
		b, hashes := c.MineNext([]string{fmt.Sprintf("tx-%d", i)}, spacing)
		if err := c.Append(b); err != nil {
			fmt.Println("append failed:", err)
			return
		}
		marker := ""
		if b.Header.Height%retargetBlocks == 0 {
			marker = "  <- retarget"
		}
		fmt.Printf("%-8d %-10s %#-12x %-18s %d%s\n",
			b.Header.Height, fmt.Sprintf("%ds", spacing), b.Header.Bits,
			short(b.Header.Hash()), hashes, marker)
	}

	fmt.Println("\nblocks 1-8 came at 30s against a 60s target, so the retarget at")
	fmt.Println("height 8 made the puzzle harder. Blocks 9-16 came at 120s, so the")
	fmt.Println("retarget at height 16 eased it again.")

	// Every rejection path that lesson 09 adds.
	fmt.Println("\nrejections that are new in this lesson:")

	b, _ := c.MineNext([]string{"x"}, 60)
	b.Header.Nonce++ // invalidates the proof of work
	fmt.Printf("  tampered nonce       : %v\n", c.Append(b))

	// A block mined honestly, but at a difficulty the retarget rule does not
	// call for. The PoW is genuine — it is the BITS that are wrong, so this
	// exercises ErrBadBits rather than ErrBadPoW.
	p := c.Tip().Header
	wrong := Header{
		Version: HeaderVersion, PrevHash: p.Hash(), MerkleRoot: MerkleRoot([]string{"y"}),
		Timestamp: p.Timestamp + 60, Bits: 0x2000ff00, Height: p.Height + 1,
	}
	mined, _ := Mine(wrong)
	b2 := Block{Header: mined, Body: []string{"y"}}
	fmt.Printf("  valid PoW, wrong bits: %v\n", c.Append(b2))
	fmt.Printf("    (its own PoW is genuine: %v)\n", CheckPoW(b2.Header))

	fmt.Printf("\nchain length %d — both rejections left it untouched.\n", c.Len())

	fmt.Println("\n--- the diff from lesson 08 ---")
	fmt.Println("  + CheckPoW: one hash, one big.Int comparison")
	fmt.Println("  + Mine: the allocation-free grinding loop")
	fmt.Println("  + NextBits: the retarget rule, integer-only")
	fmt.Println("  + two new rejection paths: ErrBadPoW and ErrBadBits")
	fmt.Println("\n  everything else is lesson 08 unchanged. That is the point of")
	fmt.Println("  having built the skeleton first.")

	fmt.Println("\n--- what lesson 10 changes ---")
	fmt.Println("  Body stops being []string and becomes []*Transaction, with")
	fmt.Println("  inputs, outputs and signatures — and the first transaction in")
	fmt.Println("  every block becomes the coinbase that pays the miner.")
}
```

**Output:**

```
genesis  height 0  bits 0x2000ffff  hash 00f67ca09a49ea1e

height   spacing    bits         hash               hashes to mine
1        30s        0x2000ffff   007dca3726326061   79
2        30s        0x2000ffff   007db4abfb87e377   235
3        30s        0x2000ffff   00f7b8703dd4999f   37
4        30s        0x2000ffff   0045d1a4f130022d   163
5        30s        0x2000ffff   00dbcd1386843b45   273
6        30s        0x2000ffff   0050571a4fd7aa1d   38
7        30s        0x2000ffff   008145669d9e26be   525
8        30s        0x1f6fff90   003a9c79f8901632   247  <- retarget
9        120s       0x1f6fff90   00480ad275a35399   286
10       120s       0x1f6fff90   005bb0dafe31f730   203
11       120s       0x1f6fff90   005d566ff07cc9c8   368
12       120s       0x1f6fff90   005ab4f9735f512a   553
13       120s       0x1f6fff90   003e491ceafc3ea4   58
14       120s       0x1f6fff90   002af0a3e0c6701d   683
15       120s       0x1f6fff90   00542d939d21188e   1300
16       120s       0x2000c3ff   00aa8119f23faef7   3  <- retarget

blocks 1-8 came at 30s against a 60s target, so the retarget at
height 8 made the puzzle harder. Blocks 9-16 came at 120s, so the
retarget at height 16 eased it again.

rejections that are new in this lesson:
  tampered nonce       : insufficient proof of work
  valid PoW, wrong bits: bits do not match the retarget rule: got 0x2000ff00, want 0x2000c3ff
    (its own PoW is genuine: true)

chain length 17 — both rejections left it untouched.

--- the diff from lesson 08 ---
  + CheckPoW: one hash, one big.Int comparison
  + Mine: the allocation-free grinding loop
  + NextBits: the retarget rule, integer-only
  + two new rejection paths: ErrBadPoW and ErrBadBits

  everything else is lesson 08 unchanged. That is the point of
  having built the skeleton first.

--- what lesson 10 changes ---
  Body stops being []string and becomes []*Transaction, with
  inputs, outputs and signatures — and the first transaction in
  every block becomes the coinbase that pays the miner.
```

---

> ← Back to the [index](README.md) · Progress tracker: [PROGRESS.md](PROGRESS.md)
