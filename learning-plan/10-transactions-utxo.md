# 10 — Transactions & the UTXO Model

> **Status:** ✅ written. Examples: 18/18 built and run.
> **Spec:** [plan/part-03-build-a-blockchain-from-scratch-go.md](plan/part-03-build-a-blockchain-from-scratch-go.md#10-transactions-the-utxo-model)

| | |
|---|---|
| **Part** | Part 3 — Build a Blockchain from Scratch (Go) |
| **Prerequisites** | [06](06-keys-signatures.md), [08](08-blocks-and-chain.md) |
| **Examples** | [18](examples/10-transactions-utxo/) (🟢 5 · 🟡 8 · 🔴 5) |

*inputs, outputs, the coinbase, signing over a trimmed copy, and maintaining the UTXO set*

Lesson 08 built blocks whose bodies were `[]string`. Lesson 09 made those blocks expensive to
produce. Neither lesson has yet said what a block is actually *for*. This one replaces the strings
with transactions, and in doing so introduces the only piece of state a UTXO chain really has: the
set of unspent outputs.

The shape of that state is the whole lesson. A signature says *you may spend this coin*; it says
nothing about whether you already have. Preventing that is not cryptography — it is bookkeeping,
and getting the bookkeeping wrong is how chains have actually failed.

[Example 18](examples/10-transactions-utxo/3-hard.md#18-transactions-in-the-chain) folds all of it
into lesson 09's chain. The header, the proof of work, the retarget rule and median-time-past are
untouched.

## Goals

- Model UTXO transactions in Go: inputs referencing prior outputs, outputs locking value.
- Sign and verify a transaction the way Bitcoin does.
- Build the coinbase transaction and enforce conservation of value.
- Maintain and query a UTXO set.

## Concepts

### 1. Accounts vs UTXO

There are two ways to answer "who owns what", and they lead to different systems all the way down.

An **account model** stores a balance per account. A payment subtracts from one number and adds to
another. Ethereum works this way, and so does every bank you have ever used.

A **UTXO model** stores a set of unspent coins, each locked to someone. A payment *destroys* the
coins it spends and *creates* new ones. Bitcoin works this way. There is no balance anywhere in the
protocol — a wallet computes one by scanning for coins it can spend.

[Example 1](examples/10-transactions-utxo/1-easy.md#1-two-ledgers-one-payment) runs the same
payment through both. The account version edits two integers; the UTXO version turns one 50-unit
coin into a 30-unit payment and a 20-unit change coin.

The interesting difference is what a validator has to look at.

```go
// account model: needs the CURRENT global balance of the sender
if ledger[from] < amount { return ErrInsufficient }

// UTXO model: needs only the outputs this transaction NAMES
for _, in := range tx.Inputs {
    if _, ok := utxo[in.Prev]; !ok { return ErrMissingInput }
}
```

In the account model, two transactions from the same sender are not independent: the first changes
the answer for the second. So execution has to be ordered, and every transaction needs a nonce to
stop it being replayed. In the UTXO model, a transaction's validity depends only on the outpoints it
names. Two transactions touching different outpoints never interact.

That single property buys three things:

- **Parallel validation.** Signature checks across a block can be spread over cores, because no
  transaction's result depends on another's — subject to the ordering rule in
  [§7](#7-the-utxo-set).
- **Simple SPV proofs.** A light client can be shown one coin's provenance with a Merkle branch
  ([05](05-merkle-trees.md)) and nothing else.
- **No global state to synchronise.** Validating a transaction never requires knowing anything
  about accounts it does not mention.

The cost is real, and it is paid by wallets rather than by nodes. There is no `balanceOf`. Bitcoin
Core cannot tell you an address's balance at all — it indexes by outpoint, and block explorers keep
a separate address index to answer that question. Every payment needs coin selection
([§8](#8-coin-selection)), an explicit change output, and a fee that grows with the number of inputs.

### 2. Transaction structure

Four types, and they do not change for the rest of Part 3:

```go
// Outpoint addresses one output: which transaction, and which output in it.
type Outpoint struct {
    TxID  [32]byte
    Index uint32
}

type TxInput struct {
    Prev      Outpoint // the output being spent
    Signature []byte   // proof it may be spent
    PubKey    []byte   // 33-byte compressed key; must hash to Prev's PubKeyHash
}

type TxOutput struct {
    Value      int64  // integer units. NEVER float64 (lesson 03).
    PubKeyHash []byte // HASH160(pubkey): who may spend this next
}

type Transaction struct {
    Inputs  []TxInput
    Outputs []TxOutput
}
```

An output **locks** value to a hash of a public key. An input **unlocks** one earlier output by
naming it and proving ownership. That is the entire model;
[example 2](examples/10-transactions-utxo/1-easy.md#2-inputs-outputs-and-outpoints) builds one by
hand.

Two details are worth dwelling on.

**Outputs have no id.** They are addressed from outside, as `(txid, index)`. Which means the txid
has to be a deterministic function of the transaction's bytes — the same rules as lesson 08's
header, one level up: fixed field order, fixed endianness, and a length prefix on every
variable-length field.

```go
func (t *Transaction) TxID() [32]byte {
    f := sha256.Sum256(t.Serialize())
    return sha256.Sum256(f[:])
}
```

[Example 3](examples/10-transactions-utxo/1-easy.md#3-the-txid-is-a-hash-of-the-bytes) shows the
consequences: swap two outputs and the id changes (it must — "output 0" now means something else);
change one unit of value and about half of the 256 bits flip. It also hashes a three-entry map a
thousand times and gets a thousand answers, which is the Go-specific way to lose a week. Anything
that goes into a hash comes out of a slice, in a documented order, never out of a map and never out
of `fmt` or JSON.

Note what *is* covered by our txid: the signatures. Bitcoin's legacy txid covers them too, so a
third party can re-encode a signature and change the transaction's id without changing what it does.
That is **transaction malleability** — Mt. Gox's public explanation for its 2014 losses, and the
reason segwit moved signatures out of the txid ([36](36-bitcoin-deep-dive.md)).

**Outputs are consumed whole.** There is no way to spend 30 of a 50-unit output. Naming it in an
input spends all of it, and anything you want back must be an explicit change output. Forget it, or
compute it wrong, and the value does not stay yours — it silently becomes fee. In September 2023
Paxos paid roughly 19.8 BTC (about $500,000) to move 0.07 BTC after a fee-calculation bug of exactly
this family. Nothing was invalid; the miner simply kept the rest.

### 3. The coinbase transaction

Every unit of currency that exists was created by a coinbase transaction. It is the one transaction
allowed to have no real inputs, so it is the only place new supply can appear — and the only place
where [§6](#6-conservation-of-value)'s rule does not apply.

```go
// The null outpoint: 32 zero bytes and index 0xffffffff. Nothing can ever
// hash to all zeros, so this cannot collide with a real output.
var nullOutpoint = Outpoint{TxID: [32]byte{}, Index: 0xffffffff}

func (t *Transaction) IsCoinbase() bool {
    return len(t.Inputs) == 1 && t.Inputs[0].Prev == nullOutpoint
}
```

The input's signature slot carries **arbitrary data** instead, and because it is not validated it has
been used for everything: Satoshi's `The Times 03/Jan/2009 Chancellor on brink of second bailout for
banks`; the **extra nonce** that gives a miner a fresh 2³² header nonces when the first batch runs
out ([09](09-proof-of-work.md)); pool names and soft-fork signalling; and, since BIP-34, the block
height.

That last one is not decoration.
[Example 4](examples/10-transactions-utxo/1-easy.md#4-the-coinbase-transaction) reproduces the bug it
fixes: two blocks that pay the same miner the same reward with the same data contain *byte-identical*
coinbase transactions, and therefore the same txid. It happened twice on mainnet — blocks 91722/91880
and 91812/91842 — and each time the second coinbase overwrote the first in the UTXO set, destroying
50 BTC. BIP-30 banned duplicate txids; BIP-34 made them impossible by requiring the height first.

**The subsidy schedule** is one line:

```go
func Subsidy(height int64) int64 {
    halvings := height / 210_000
    if halvings >= 64 {
        return 0
    }
    return 50 * Coin >> uint(halvings)
}
```

[Example 5](examples/10-transactions-utxo/1-easy.md#5-the-subsidy-schedule) prints all 33 eras. The
total comes to 20,999,999.9769 BTC — the famous "21 million" is integer truncation in the last few
eras, nothing more — and the subsidy reaches zero at height 6,930,000, around 2140. Actual supply
will be lower: a coinbase may claim *less* than it is owed, and several have. Block 501726 claimed
nothing at all.

That schedule is also the long-run security problem. Miners buy hashrate with what they are paid, and
hashrate is what a 51% attack has to out-spend ([09](09-proof-of-work.md)). The subsidy halves
toward zero, so fees have to carry the entire security budget eventually. Whether a fee market alone
can pay for enough hashrate is genuinely unsettled, and it is why fee-market design
([11](11-wallets-mempool.md)) is not a side topic.

Finally, two rules that are easy to forget and are enforced at the *block* level, not the
transaction level:

- **Value.** The coinbase may claim at most `Subsidy(height) + sum(fees of the other transactions)`.
- **Maturity.** Coinbase outputs are unspendable for 100 blocks — [§6](#6-conservation-of-value)
  explains why, and [example 16](examples/10-transactions-utxo/3-hard.md#16-coinbase-maturity-and-the-cascade-it-prevents)
  shows what happens without it.

### 4. What exactly gets signed

"Sign the transaction" is not precise enough to implement. The signature lives *inside* the
transaction, so it cannot commit to a transaction that already contains it. And each input signs
independently, because in general each input belongs to a different person.

What actually gets signed is a **trimmed copy**: every signature and public key cleared, and — for
the one input being signed — the referenced output's locking hash substituted into the pubkey slot.

```go
func (t *Transaction) TrimmedCopy() *Transaction {
    ins := make([]TxInput, len(t.Inputs))
    for i, in := range t.Inputs {
        ins[i] = TxInput{Prev: in.Prev} // Signature and PubKey left nil
    }
    outs := make([]TxOutput, len(t.Outputs))
    for i, o := range t.Outputs {
        outs[i] = TxOutput{Value: o.Value, PubKeyHash: append([]byte(nil), o.PubKeyHash...)}
    }
    return &Transaction{Inputs: ins, Outputs: outs}
}

func (t *Transaction) SigHash(i int, prevPubKeyHash []byte) [32]byte {
    c := t.TrimmedCopy()
    c.Inputs[i].PubKey = prevPubKeyHash // only this one input is "filled in"
    return c.TxID()
}
```

Substituting the locking hash is what binds the signature to *which coin* this input spends. Because
each input fills in its own slot, the sighashes differ even when the outputs are identical — so a
signature made for input 0 cannot be lifted to input 1.
[Example 6](examples/10-transactions-utxo/2-medium.md#6-what-exactly-gets-signed) mutates five
different things and confirms every one changes the sighash: an output's value, its recipient, the
output order, an input's outpoint, and appending an output.

Which is exactly the habit this forces: **sign last**. Coin selection, the change output, the fee and
the ordering all have to be final before the first signature, because there is no such thing as
editing a signed transaction. A wallet that signs early and adjusts the fee afterwards produces
transactions that every node silently drops.

**The pitfall.** `c := *t` copies the struct — and the struct holds slice *headers*. Copying a slice
header copies the pointer, not the elements, so a loop that writes through `c.Inputs[i]` writes
straight into the original.

```go
// WRONG. Compiles, looks right, and destroys the signature you just made.
func (t *Transaction) TrimmedCopyBuggy() *Transaction {
    c := *t
    for i := range c.Inputs {
        c.Inputs[i].Signature = nil // writes into t.Inputs too
        c.Inputs[i].PubKey = nil
    }
    return &c
}
```

The symptom is horrible: signing succeeds, verification fails, and *only for transactions with more
than one input*. [Example 8](examples/10-transactions-utxo/2-medium.md#8-the-copy-that-was-not-a-copy)
shows one input working perfectly, two inputs losing input 0's signature, and three inputs leaving
only the last. Neither `go vet` nor `-race` will find it — it is not a data race, it is one goroutine
writing where it did not mean to. The test that catches it is a multi-input sign-then-verify. Write
that test first.

Bitcoin generalises all of this with **sighash types**, one byte appended to each signature that
chooses how much of the transaction it covers: `ALL`, `NONE`, `SINGLE`, and the `ANYONECANPAY`
modifier. They are what make CoinJoin, crowdfunded transactions and payment channels expressible, and
they are sharp — `SIGHASH_SINGLE` with no matching output hashes the value 1 in Bitcoin, a bug
preserved for compatibility and a real way to lose coins. Lesson [36](36-bitcoin-deep-dive.md) does them
properly.

### 5. Verification

Verification has two halves, and only one of them is cryptography:

```go
for i, in := range t.Inputs {
    // 1. does this key open that lock?
    if !bytes.Equal(hash160(in.PubKey), prevOuts[i].PubKeyHash) {
        return &InputError{i, ErrKeyMismatch}
    }
    // 2. did the holder of that key authorise THIS transaction?
    h := t.SigHash(i, prevOuts[i].PubKeyHash)
    if !crypto.VerifySignature(in.PubKey, h[:], in.Signature[:64]) {
        return &InputError{i, ErrBadSig}
    }
}
```

Skip the first check and the system is completely broken — and it will not look broken, because every
signature it accepts is genuine.
[Example 10](examples/10-transactions-utxo/2-medium.md#10-the-check-that-makes-ownership-mean-anything)
has mallory spend alice's coin by signing with her *own* key. The signature verifies. A validator
that only checks the cryptography hands over the coin. In Bitcoin these are the two halves of the
P2PKH script — `OP_DUP OP_HASH160 <pkh> OP_EQUALVERIFY OP_CHECKSIG` — and `OP_EQUALVERIFY` is the one
people forget when hand-rolling. Ethereum makes the same trade differently: it recovers the key from
the signature ([06](06-keys-signatures.md)) and compares the derived address, folding both checks
into one operation.

Note also what verification needs that the transaction does not contain: `prevOuts`. A transaction
names its inputs but does not carry their values or their locks, so the validator must look each one
up. That lookup is the UTXO set ([§7](#7-the-utxo-set)).

**Any single failing input invalidates the whole transaction** — there is no partial acceptance. But
"invalid transaction" is a useless log line when you are running a mempool, so return which input
failed and why:

```go
type InputError struct {
    Index int
    Err   error
}

func (e *InputError) Error() string { return fmt.Sprintf("input %d: %v", e.Index, e.Err) }
func (e *InputError) Unwrap() error { return e.Err }
```

[Example 9](examples/10-transactions-utxo/2-medium.md#9-which-input-failed) breaks each input of a
three-party transaction in turn and uses `errors.As` and `errors.Is` on the result the way a mempool
would: a bad signature is a reason to ban a peer, a missing signature means the wallet is still
assembling. It also shows that when two inputs are broken only the first is reported — deliberately.
Validation is a rejection decision, not a report, and doing less work on invalid data is a DoS
defence rather than an optimisation.

That is also why the checks are ordered cheapest-first: structural checks (lengths, counts) with no
crypto, then one HASH160 and a 20-byte comparison, and only then the signature, which costs tens of
microseconds. Attacker-supplied data reaches the expensive step only after passing the cheap ones.

### 6. Conservation of value

Three rules, and a chain is sound only if all three hold on every block:

1. an ordinary transaction: `sum(inputs) >= sum(outputs)`
2. the difference is the **fee**, and the miner claims it
3. the coinbase claims at most `Subsidy(height) + sum(fees)`

Rule 3 is what pins the other two down — without it a miner could mint whatever it liked and rules 1
and 2 would still hold. It is also why the coinbase bound belongs in the *block* checks: it needs
every other fee in the block. [Example 11](examples/10-transactions-utxo/2-medium.md#11-conservation-of-value)
runs all three. Note that claiming *less* is always legal, and the difference is destroyed — there is
no account for it to go to.

Never let a transaction state its own fee. There is nothing to check it against; the fee is `in -
out` and that is all it can be.

**Overflow is the sharp edge here**, and it is not hypothetical. On 15 August 2010, one transaction
in block 74638 spent 0.5 BTC and created two outputs of 92,233,720,368.54277039 BTC each. Their sum
overflows a signed 64-bit integer and comes out *negative*, so this check passed:

```go
var sum int64
for _, o := range outs {
    sum += o.Value // wraps silently: no panic, no vet warning
}
if sum > in { return ErrNotCovered }  // -997538 > 50000000 is false
```

184.5 billion BTC existed for five hours. CVE-2010-5139 was patched in 0.3.10 the same day, and the
bad chain was abandoned after about 53 blocks — the only time Bitcoin's transaction history has been
deliberately rewritten. [Example 17](examples/10-transactions-utxo/3-hard.md#17-the-value-overflow-incident)
reproduces it with the real numbers and fixes it three ways.

The fix Bitcoin actually shipped is the one to copy: **range-check every value at the edge**, before
it reaches any arithmetic.

```go
if o.Value < 0 || o.Value > MaxMoney {
    return ErrRange
}
sum += o.Value
if sum > MaxMoney {
    return ErrRange
}
```

Bounding each term below 21×10¹⁴ makes overflow of a sum of any plausible number of them impossible
by construction. Checked addition and `big.Int` are the fallbacks for when there is no such domain
bound. Go will not help you here: a *constant* that overflows is a compile error, but two `int64`
*variables* wrap silently, exactly as in C++.

The same class of bug reappeared in April 2018 as `batchOverflow` (CVE-2018-10299), where an ERC-20
contract computed `amount * receivers.length` in unchecked Solidity and minted 2²⁵⁵ tokens twice.
Exchanges suspended ERC-20 deposits. Solidity made arithmetic checked by default in 0.8.0 in December
2020 — ten years after the Bitcoin bug.

**Within-block double-spend detection** is the third piece. Each transaction in a block is
individually valid, so nothing catches two of them spending the same outpoint unless you track spent
outpoints as you validate:

```go
spentBy := map[Outpoint]int{}
for i, t := range b.Txs {
    for _, in := range t.Inputs {
        if j, dup := spentBy[in.Prev]; dup {
            return fmt.Errorf("tx %d: %w: also spent by tx %d", i, ErrDoubleSpend, j)
        }
        spentBy[in.Prev] = i
    }
}
```

The same map catches an outpoint named twice inside a *single* transaction — the version that slips
past validators that only compare transactions with each other.
[Example 13](examples/10-transactions-utxo/2-medium.md#13-double-spends-and-applying-a-block-atomically)
covers all three scopes.

**Coinbase maturity** exists for a reason that only shows up under reorg. A coinbase has no inputs,
so if the block that created it is orphaned, the coins never existed — and every transaction that
spent them, and every transaction that spent *those* outputs, becomes invalid at once. An ordinary
transaction just gets re-mined into the new chain; a coinbase spend cannot be, because its input does
not exist on any chain. [Example 16](examples/10-transactions-utxo/3-hard.md#16-coinbase-maturity-and-the-cascade-it-prevents)
turns the rule off, builds a four-deep chain of spends ending at an exchange that pays out real
money, then orphans the coinbase's block and watches all of it die.

```go
if e.Coinbase && spendHeight-e.Height < CoinbaseMaturity {
    return ErrImmature
}
```

Note the comparison is `<`, not `<=`. Getting that boundary wrong by one block is not a bug, it is a
chain split. Bitcoin's 100 blocks is about 16 hours and 40 minutes of locked-up revenue, which is why
mining pools pay out of their own balance rather than out of the block — and it is comfortably deeper
than the worst accidental reorg in Bitcoin's history, the 24 blocks of the March 2013 v0.7/v0.8 fork.

### 7. The UTXO set

The blocks are the *history*. The UTXO set is the *state* — the only thing you actually need to
validate the next transaction. Build it by replaying the chain once, then maintain it incrementally.

```go
type UTXOSet map[Outpoint]Entry

type Entry struct {
    Out      TxOutput
    Height   int64 // needed for the maturity rule
    Coinbase bool
}
```

Applying a block is **one delta**: insert the new outputs, then delete the spent outpoints. Build the
delta first and apply it only if the whole block validates — the same validate-then-mutate rule as
lesson 08's `Append`, one level up.

```go
type Delta struct {
    Spend  []Outpoint
    Create map[Outpoint]Entry
}

// Insert first, then delete: an output a block both creates and spends is in
// Create AND in Spend, and deleting first would put it back.
func (s UTXOSet) Apply(d Delta) {
    for op, e := range d.Create { s[op] = e }
    for _, op := range d.Spend { delete(s, op) }
}
```

The tempting alternative — validate and mutate in one pass — fails silently.
[Example 13](examples/10-transactions-utxo/2-medium.md#13-double-spends-and-applying-a-block-atomically)
runs a half-invalid block through both and fingerprints the set: the one-pass version leaves it
corrupted, and the node does not crash. It carries on with a state nobody else has, accepts blocks
nobody else accepts, and forks itself off the network.

**Reverting** a block needs something the set no longer holds: the outputs that were deleted. That is
why real nodes write an *undo file* per block. Lesson [14](14-consensus-forks.md) turns this
into choosing between two branches; [example 12](examples/10-transactions-utxo/2-medium.md#12-building-the-utxo-set)
does one revert and one re-apply.

Two ordering rules matter when validating a block:

- A transaction **may** spend an output created earlier in the same block. That is how a chain of
  dependent transactions gets mined together.
- The parent **must appear first**. Bitcoin requires it, which keeps validation a single forward
  pass: no topological sort, no second pass, and no cycles to reason about.

**Size and growth.** Bitcoin's UTXO set is well over a hundred million entries and several
gigabytes. It grows whenever transactions create more outputs than they consume — which is the
**dust problem**. A 1-satoshi output costs every node permanent state forever, and costs its owner
more in fees to spend than it is worth. Nobody will ever pay to remove it. That asymmetry is why
exchanges consolidate: spending hundreds of small outputs into one when fees are low shrinks both the
global set and their own future costs. It is also why unsolicited dust is an *attack* — see
[§8](#8-coin-selection).

In memory a `map[Outpoint]Entry` is fine. Lesson [12](12-persistence-chainstate.md) puts it on disk,
and the shape does not change.

### 8. Coin selection

Choosing which of your coins to spend looks like a knapsack problem, and it is — but the objective is
not simply "reach the amount". Every extra input costs about 68 vbytes of fee, every change output
creates a UTXO you will pay to spend later, and the *shape* of what you pick tells a chain analyst
who you are.

The one line that organises all of it is a coin's **effective value**:

```go
// what the coin is worth after paying to spend it
func effectiveValue(c Coin, feeRate int64) int64 {
    return c.Value - feeRate*inputVSize
}
```

A 400-satoshi coin at 20 sat/vB has an effective value of −960: including it makes you *poorer*. That
is what "dust" means. It is not a small coin, it is a negative one.

[Example 14](examples/10-transactions-utxo/3-hard.md#14-coin-selection-behind-an-interface) puts three
strategies behind one interface and compares them on the same wallet:

| | inputs | fee | change | UTXOs |
|---|---|---|---|---|
| **largest-first** | fewest | lowest today | large | unchanged or grows |
| **smallest-first** | many | several times higher | small | shrinks sharply |
| **branch-and-bound** | exact match | middling | **none** | shrinks |

Largest-first minimises today's fee and never touches the small coins, so the wallet slowly fills
with dust it can never afford to spend. Smallest-first consolidates — run it when fees are low, not
when they spike. Branch-and-bound is Bitcoin Core's: it searches for a subset whose effective values
land in `[target, target+costOfChange]`, needing *no change output at all*. When it finds one the
transaction is 31 vbytes smaller, no new UTXO is created, and there is nothing for an analyst to
identify. When it does not, Core falls back to a randomised knapsack, and so should you.

Optimal selection is NP-hard, and "optimal" is not even well defined: fee now trades against fee
later, and both trade against privacy. Every real wallet ships a heuristic. The point is not to find
the best answer but to avoid the bad ones — which is why the selector belongs behind an interface:

```go
type Selector interface {
    Name() string
    Select(coins []Coin, payment, feeRate int64) (Selection, error)
}
```

It is the piece you will replace most often — fee spikes, consolidation windows, privacy modes — and
keeping it behind one method means the signing path never changes and every strategy can be tested
against the same wallet fixture.

**The change output is the leak.**
[Example 15](examples/10-transactions-utxo/3-hard.md#15-change-dust-and-what-they-leak) runs the
heuristics an analyst actually uses. The round-number test — a human paying a human picks a round
amount, so the *other* output is change — gets five of five on ordinary payments and nothing at all
when both outputs are round. Analysts stack several: change usually matches the inputs' script type;
an output to a previously seen address is not change; if dropping an input would still cover one
output, that output is probably the payment.

The one that actually deanonymises chains is **common input ownership**: if two addresses appear as
inputs of the same transaction, one wallet held both keys. Example 15 clusters a week of ordinary
activity with union-find, then has alice consolidate her dust in a single transaction — and every
address she has ever used collapses into one identity, retroactively, with no undo. That is the trade
example 14's table does not show. It is also why unsolicited dust is an attack: send a thousand
people 500 satoshi and wait for a wallet to sweep it up alongside their real coins.

CoinJoin is the counter-move, and it is worth being precise about what it does. It does not hide
anything. It puts several people's inputs in one transaction so the assumption behind the heuristic
becomes *wrong* — and a heuristic that is sometimes wrong is far less useful than one that is always
right.

What a wallet does about all this is not cryptography. It is coin selection: a fresh address for
every receive including change (which is what the HD wallet of [07](07-addresses-wallets-hd.md) is
for), change on the same script type as the inputs, avoiding change entirely where possible, coin
control so unrelated outputs are never mixed, and quarantining dust rather than sweeping it. Which
makes the selector the most security-relevant part of a wallet after key storage.

## Exercises

Continue the program from [09](09-proof-of-work.md) in `practice/`.

1. **The four types.** Define `Outpoint`, `TxInput`, `TxOutput` and `Transaction`, write
   `Serialize` and `TxID`, and table-test that reordering outputs, changing a value, and changing a
   signature each produce a different id — and that rebuilding the same transaction produces the
   same one.
2. **The coinbase.** Write `NewCoinbase(height, to, value)` with the null outpoint and a BIP-34
   height prefix, and `Subsidy(height)`. Assert that two coinbases at different heights paying the
   same miner the same amount have different txids, and that two at the *same* height do not.
3. **Sign and verify.** Implement `TrimmedCopy`, `SigHash`, `Sign` and `Verify`. Then write the test
   that matters: a **two-input** transaction, signed and verified, plus an assertion that the
   original is byte-identical before and after `TrimmedCopy`.
4. **Break it six ways.** Table-test verification against: an output value changed by one, a changed
   recipient, swapped outputs, a repointed outpoint, one flipped signature bit, and an output
   appended after signing. All six must fail, and the error must name input 0.
5. **The theft that verifies.** Write the signature-only verifier as well as the full one. Have a
   second key sign a spend of the first key's coin, and assert the sig-only verifier accepts while
   the full one rejects with a key-mismatch error.
6. **Conservation.** Implement `Fee(tx, prevOuts)` with per-value range checks and
   `CheckCoinbase(cb, height, fees)`. Feed it the block-74638 numbers and assert it rejects them,
   then assert it still accepts an ordinary payment.
7. **The UTXO set.** Build `UTXOSet`, `Delta`, `Apply` and `Revert`. Replay a three-block chain,
   assert the balances, revert the last block, assert the balances again, re-apply, and assert the
   set fingerprint matches the original.
8. **Atomic blocks.** Write `ValidateBlock` returning `(Delta, fees, error)` without mutating
   anything, and cover: a double spend across two transactions, an outpoint named twice in one
   transaction, a child before its parent, and a half-valid block. Fingerprint the set before and
   after each rejection and assert it is unchanged.
9. **Maturity.** Add `Height` and `Coinbase` to your UTXO entries and enforce the rule. Test the
   boundary at exactly `maturity-1` and `maturity`, and write the reorg case: build a spend chain
   from a coinbase, revert its block, and assert every descendant is now unspendable.
10. **Two selectors.** Implement `Selector` with largest-first and smallest-first, using effective
    values. Report inputs, fee, change and the UTXO-count delta for four payment sizes, and assert
    that no selection ever includes a coin with a negative effective value.

## Best Practices & Pitfalls

- **Deep-copy the inputs in `TrimmedCopy`, and prove the original is untouched.**
  *Why:* `c := *t` shares the slice backing array, so trimming for input 1 erases the signature you
  made for input 0. It works perfectly with one input, which is why it reaches production
  (example 8).
- **Sign only when every output is final.**
  *Why:* the sighash covers every value, recipient and ordering. Adjusting a fee after signing
  produces a transaction that every node drops, with no error anywhere in your code.
- **Always check that `HASH160(pubkey)` equals the spent output's `PubKeyHash`.**
  *Why:* without it, a valid signature over the sighash is enough to spend anyone's coins — the
  signature is genuine, it just belongs to the wrong person (example 10).
- **Range-check every value before summing, not just the total.**
  *Why:* `0 <= v <= MaxMoney` on each output makes overflow of the sum unreachable. Checking only the
  total is CVE-2010-5139.
- **Compute the fee as `in - out`; never let a transaction declare it.**
  *Why:* a declared fee is unverifiable — there is nothing to check it against.
- **Enforce the coinbase bound in the block checks, not the transaction checks.**
  *Why:* it needs every other fee in the block, which a single transaction cannot see.
- **Track spent outpoints while validating a block.**
  *Why:* every transaction in a double-spending pair is individually valid. Nothing else catches it,
  including the case of one outpoint named twice inside one transaction.
- **Build the delta, then apply it — never mutate the UTXO set while validating.**
  *Why:* a block rejected halfway leaves a state nobody else has, and the node forks itself off the
  network without crashing (example 13).
- **Store the spent entries alongside each block.**
  *Why:* reverting needs the outputs the set no longer holds. Without an undo record a reorg is
  unimplementable.
- **Enforce coinbase maturity, and get the boundary right.**
  *Why:* a reorg erases a coinbase and everything descended from it, and none of it can be re-mined.
  `<` versus `<=` is a one-block disagreement, which is a chain split.
- **Require the parent transaction to appear before its child in a block.**
  *Why:* it keeps validation a single forward pass, with no topological sort and no cycles.
- **Never create change below the dust threshold.**
  *Why:* it costs more to spend than it holds, so it becomes permanent global state. Give it to the
  miner instead.
- **Treat coin selection as security-relevant.**
  *Why:* consolidating dust merges every address you have used into one cluster, permanently. That
  is a bigger privacy decision than anything in the signing code.

## Checklist

- [ ] I can explain what UTXO buys over accounts, and what it costs.
- [ ] I can define the four types and serialize a transaction deterministically.
- [ ] I can explain why an output is addressed as `(txid, index)` and why order is part of identity.
- [ ] I can build a coinbase, explain the null outpoint, and say what the data field is for.
- [ ] I can compute the subsidy at any height and explain the security-budget problem.
- [ ] I can implement `TrimmedCopy` and `SigHash` and say exactly what a signature commits to.
- [ ] I know why a shallow copy breaks multi-input signing, and which test catches it.
- [ ] I can name both halves of verification and explain what breaks if the binding check is missing.
- [ ] I can enforce conservation of value without an overflow, and explain CVE-2010-5139.
- [ ] I can maintain a UTXO set as one atomic delta per block, and revert one.
- [ ] I can detect a double spend at all three scopes: transaction, block, and chain.
- [ ] I can explain coinbase maturity in terms of reorgs, not in terms of policy.
- [ ] I can implement two coin-selection strategies and compare their fee and privacy consequences.

## Resources

**Specifications**

- Bitcoin transactions reference: https://developer.bitcoin.org/reference/transactions.html
- Bitcoin devguide — transactions: https://developer.bitcoin.org/devguide/transactions.html
- BIP-30 (duplicate txids): https://github.com/bitcoin/bips/blob/master/bip-0030.mediawiki
- BIP-34 (height in coinbase): https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki
- BIP-125 (replace-by-fee), for lesson 11: https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki

**Incidents**

- CVE-2010-5139, the value overflow incident: https://en.bitcoin.it/wiki/Value_overflow_incident
- CVE-2012-2459 (duplicate txids in a Merkle tree), the sequel: https://en.bitcoin.it/wiki/CVEs
- batchOverflow, CVE-2018-10299: https://nvd.nist.gov/vuln/detail/CVE-2018-10299

**Coin selection & privacy**

- Bitcoin Core's branch-and-bound (Erhardt, 2016): https://murch.one/erhardt2016coinselection.pdf
- Common-input-ownership and change heuristics: https://en.bitcoin.it/wiki/Privacy

**Go**

- `crypto/ecdsa`: https://pkg.go.dev/crypto/ecdsa
- `encoding/binary`: https://pkg.go.dev/encoding/binary
- `errors` (`As`, `Is`, `Unwrap`): https://pkg.go.dev/errors

---

**Examples:** [`examples/10-transactions-utxo/`](examples/10-transactions-utxo/) — **18 runnable Go
programs** (🟢 5 easy · 🟡 8 medium · 🔴 5 hard). Example 4 reproduces the duplicate-coinbase bug that
destroyed 100 BTC; example 16 turns coinbase maturity off and follows the cascade to an exchange;
example 17 runs the actual block-74638 transaction through four validators; example 18 folds the
whole transaction model into lesson 09's chain.

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
