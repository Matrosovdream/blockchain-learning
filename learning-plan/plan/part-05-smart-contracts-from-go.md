# Part 5 — Smart Contracts from Go

Enough Solidity to read a contract, then everything on the Go side: ABI encoding, abigen bindings, events and logs, the ERC standards, and the bug classes that drain contracts.

**Core spine.** Lessons 22–27 · 109 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 22 | [Solidity Basics for Go Developers](#22-solidity-basics-for-go-developers) | 18 | 16 |
| 23 | [The Contract ABI: Encoding & Decoding](#23-the-contract-abi-encoding-decoding) | 22 | 20 |
| 24 | [Type-Safe Contracts with `abigen`](#24-type-safe-contracts-with-abigen) | 23 | 16 |
| 25 | [Events, Logs & Indexing Them](#25-events-logs-indexing-them) | 23 | 19 |
| 26 | [The ERC Standards from Go](#26-the-erc-standards-from-go) | 24, 25 | 19 |
| 27 | [Smart Contract Security](#27-smart-contract-security) | 26 | 19 |

---

## 22 — Solidity Basics for Go Developers

**Lesson file:** [../22-solidity-basics.md](../22-solidity-basics.md) · **Examples folder:** `../examples/22-solidity-basics/`

| | |
|---|---|
| Prerequisites | [18](../18-evm.md) |
| Unlocks | 23, 45, 48 |
| Examples | **16** — 🟢 5 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 11 |

*enough Solidity to read, compile and reason about a contract — mapped onto EVM mechanics you know*

### Goals

- Read a Solidity contract and predict its storage layout and gas behaviour.
- Write, compile and deploy a small contract with `solc`/Foundry.
- Map every Solidity concept onto the EVM mechanics from lesson 18.
- Recognise the constructs behind every ABI you will decode.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Contract anatomy**

   - SPDX licence, pragma, imports, contract, state variables, constructor, functions, modifiers, events, errors.
   - `contract` vs `interface` vs `library` vs `abstract contract`.
   - The mental shift from Go: no goroutines, no pointers, no floating point, and gas on every line.
   - Reading order for an unfamiliar contract: state → constructor → external functions → modifiers.

**2. The type system**

   - `uint256` is the default and the cheapest; smaller types cost more unless packed.
   - `address` vs `address payable`; `bytes32` (value, cheap) vs `bytes`/`string` (dynamic, expensive).
   - Fixed vs dynamic arrays, structs, enums, and `mapping` (no length, no iteration, no deletion of all).
   - No floats. Fixed-point is done by convention with 18 decimals.

**3. Data location**

   - `storage` (persistent, expensive), `memory` (per-call, cheap), `calldata` (read-only, cheapest).
   - The aliasing surprise: a `storage` reference mutates state; a `memory` copy does not.
   - Why function parameters on external functions should be `calldata`.
   - This maps directly onto lesson 18's SSTORE/MSTORE/CALLDATALOAD costs.

**4. Storage layout**

   - Slot 0, 1, 2… in declaration order; multiple small values pack into one 32-byte slot.
   - `mapping(k => v)` at slot p: the value lives at `keccak256(h(k) . p)`.
   - Dynamic array at slot p: length at p, elements from `keccak256(p)`.
   - Nested mappings compose the hash. **You can read all of this with `eth_getStorageAt` from Go** — do it.

**5. Visibility and mutability**

   - `public` (generates a getter), `external`, `internal`, `private` — and that `private` is not secret.
   - `view` (STATICCALL-able), `pure`, `payable`, and what each forbids at the EVM level.
   - Auto-generated getters and how they appear in the ABI.
   - Function overloading and how it affects selectors (lesson 23).

**6. Events**

   - `event Transfer(address indexed from, address indexed to, uint256 value)`.
   - `indexed` puts a parameter in a topic (searchable); non-indexed goes in `data`.
   - Max 3 indexed parameters (topic0 is the signature hash).
   - Contracts cannot read events — they exist purely for off-chain consumers like you (lesson 25).

**7. Errors and reverts**

   - `require(cond, "msg")`, `revert CustomError(args)`, `assert` (Panic), and their gas differences.
   - Custom errors are 4 bytes + args instead of a string — much cheaper.
   - How a revert reason reaches Go: ABI-encoded return data (lesson 23).
   - `try/catch` for external calls and what it can and cannot catch.

**8. Inheritance and libraries**

   - Linearisation (C3), `super`, `virtual`/`override`, constructor chaining.
   - Libraries: `internal` (inlined) vs `external` (DELEGATECALL, needs linking).
   - `using X for Y` — where `SafeERC20.safeTransfer` comes from.
   - Enough to navigate OpenZeppelin without getting lost.

**9. Compilation output**

   - ABI JSON, creation bytecode, deployed bytecode, metadata hash appended to the code.
   - Different compiler versions produce different bytecode for identical source — pin the version.
   - Optimizer runs and what they trade (deploy size vs runtime cost).
   - Source verification on explorers and why it matters to you as an integrator.

**10. The Go developer's mapping table**

   - Solidity type → ABI type → Go type: `uint256`→`*big.Int`, `address`→`common.Address`, `bytes32`→`[32]byte`.
   - `string`→`string`, `bytes`→`[]byte`, `T[]`→`[]T`, struct→generated Go struct.
   - Where abigen's choices surprise you (lesson 24).
   - Keep this table; you will use it constantly.

**11. Foundry workflow**

   - `forge init`, `forge build`, `forge test`, `forge create`, `forge inspect <C> storageLayout`.
   - `cast call`, `cast send`, `cast storage`, `cast 4byte-decode` — your ground truth when Go disagrees.
   - `anvil` as the local chain for everything in Part 5.
   - Full test coverage in lesson 48.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming `private` means confidential — all storage is public, and you will read it from Go in this lesson.
- Using a compiler version different from the one used to deploy and getting different bytecode.
- Forgetting that mappings cannot be iterated, then designing an indexer that needs iteration.
- Treating `assert` and `require` as interchangeable — different gas, different Panic codes.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Compile a `Counter` contract with `solc` and print the ABI from Go.
- Deploy it to `anvil` with `forge create` and read `number()` with `cast`.
- List the contract's functions and their mutability from the ABI JSON.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Read a public variable's storage slot with `eth_getStorageAt` and decode it in Go.
- Compute a mapping slot with `keccak256(key . slot)` and read a balance with no ABI call.
- Show slot packing: two `uint128`s in one slot, read both from a single `getStorageAt`.
- Compare `forge inspect storageLayout` against the slots you computed by hand.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Read a dynamic array's length and all elements straight from storage in Go.
- Decode a nested mapping (`mapping(address => mapping(address => uint256))`) allowance from storage.
- Trigger a custom error and decode its selector and arguments in Go.

### Packages & tools

`os/exec`, `encoding/json`, `github.com/ethereum/go-ethereum/accounts/abi`, `github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/crypto`

### Resources to cite

- Solidity docs: https://docs.soliditylang.org/
- Solidity storage layout: https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html
- Foundry Book: https://book.getfoundry.sh/
- OpenZeppelin Contracts: https://docs.openzeppelin.com/contracts/

---

## 23 — The Contract ABI: Encoding & Decoding

**Lesson file:** [../23-abi-encoding.md](../23-abi-encoding.md) · **Examples folder:** `../examples/23-abi-encoding/`

| | |
|---|---|
| Prerequisites | [22](../22-solidity-basics.md) |
| Unlocks | 24, 25, 47, 50 |
| Examples | **20** — 🟢 5 easy, 🟡 9 medium, 🔴 6 hard |
| Topics | 10 |

*function selectors, head/tail layout, dynamic types, revert data and EIP-712 — by hand and with `abi`*

### Goals

- Compute a function selector and encode arguments by hand.
- Explain the head/tail layout for dynamic types.
- Encode and decode with go-ethereum's `abi` package.
- Decode an unknown calldata blob and a revert reason.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The ABI is a calling convention**

   - calldata = 4-byte selector ‖ encoded arguments. That is the whole protocol.
   - The EVM has no notion of functions — the dispatcher is compiled Solidity comparing the selector.
   - This is why you can call any contract if you know its ABI, verified or not.
   - And why an unverified contract is still callable if you can guess the selectors.

**2. Function selectors**

   - `keccak256("transfer(address,uint256)")[:4]` — canonical signature, no spaces, no parameter names.
   - Canonical type names: `uint` → `uint256`, `address payable` → `address`, tuples as `(a,b)`.
   - 4 bytes = 4 billion values; collisions exist and have been weaponised (proxy selector clashes, lesson 46).
   - Reverse lookup: the 4byte directory and `cast 4byte`.

**3. Static encoding**

   - Everything is padded to 32 bytes. Numbers and addresses are left-padded; `bytesN` right-padded.
   - `bool` is a 32-byte 0 or 1. Negative `int` uses two's complement.
   - Fixed arrays of static types are inlined, in order, with no length prefix.
   - Structs of only static types are inlined too.

**4. Dynamic encoding — head and tail**

   - The head holds a 32-byte *offset* to the tail; the tail holds length then data.
   - Offsets are relative to the start of the enclosing tuple's encoding, not the calldata.
   - Work `(uint256, string, uint256[])` through byte by byte, annotating every word.
   - This is the part everyone gets wrong; do it manually before using the library.

**5. Nested dynamic types**

   - `string[]`, `bytes[]`, `(uint256, string)[]` — offsets inside offsets.
   - A tuple containing any dynamic member is itself dynamic.
   - The recursion rule, stated once and applied to three worked examples.
   - Malicious offsets: a decoder must bounds-check, and `abi.Unpack` does.

**6. Tuples and Go structs**

   - Solidity structs map to Go structs; field names come from the ABI's component names.
   - `abi:"fieldName"` tags and the unnamed-component problem.
   - `UnpackIntoInterface` for a typed result; `Unpack` for `[]interface{}`.
   - Anonymous structs generated by abigen (lesson 24).

**7. Return data and revert data**

   - Return data is encoded exactly like arguments, with no selector.
   - An `eth_call` to a non-contract address returns empty data and *succeeds* — a classic silent bug.
   - `Error(string)` = selector `0x08c379a0` + ABI-encoded string.
   - `Panic(uint256)` = `0x4e487b71` + a code (0x01 assert, 0x11 overflow, 0x12 div-by-zero, 0x32 OOB).
   - Custom errors: `keccak("MyError(uint256)")[:4]` + args — decode with the ABI's Errors map.

**8. The Go API**

   - `abi.JSON(strings.NewReader(abiJSON))` → `abi.ABI` with Methods, Events, Errors.
   - `Pack(name, args...)`, `Unpack(name, data)`, `UnpackIntoInterface(&out, name, data)`.
   - `abi.Arguments` and `abi.NewType` for building an encoder with no ABI file.
   - Type conversion rules Go-side: always `*big.Int` for `uint256`, never `int64`.

**9. encodePacked and its hazard**

   - Packed encoding drops padding and length prefixes — used for hashing, never for calls.
   - The collision: `("a","bc")` and `("ab","c")` pack identically.
   - Real exploits have come from signing packed data with adjacent dynamic types.
   - Rule: never `encodePacked` two dynamic types next to each other.

**10. EIP-712 typed structured data**

   - The problem: signing an opaque hash is unreadable and replayable.
   - domainSeparator = hashStruct(EIP712Domain{name, version, chainId, verifyingContract}).
   - digest = keccak256(0x19 0x01 ‖ domainSeparator ‖ hashStruct(message)).
   - Implementing it in Go with `signer/core/apitypes`, and verifying against `cast wallet sign-typed-data`.
   - Full application treatment in lesson 50.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reading offsets as absolute calldata positions instead of relative to the enclosing tuple.
- Treating an empty `eth_call` result as a zero value instead of 'address has no code'.
- Using `int64`/`uint64` for `uint256` and truncating silently.
- `encodePacked` with adjacent dynamic types in anything that gets signed.
- Assuming a 4-byte selector uniquely identifies a function.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 20).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Compute the `transfer(address,uint256)` selector and compare with `0xa9059cbb`.
- Encode a single `uint256` and print the 32 bytes.
- Decode a 32-byte return value into a `*big.Int`.

**🟡 Medium — 9 examples** *(concepts combined, and the traps)*

- Encode `(address, uint256)` by hand into 68 bytes and diff against `abi.Pack`.
- Encode `(uint256, string, uint256[])` and annotate every 32-byte word.
- Decode a real mainnet calldata blob given only the ABI JSON.
- Decode an `Error(string)` revert reason and a `Panic(uint256)` code.
- Build an encoder with `abi.NewType` without any ABI file.

**🔴 Hard — 6 examples** *(real-shaped, multi-concept programs)*

- Decode a nested `(uint256,(address,bytes)[])` argument and print the tree.
- Demonstrate the `encodePacked` collision and show the safe alternative.
- Sign and verify an EIP-712 typed message in Go and match `cast wallet sign-typed-data`.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi`, `github.com/ethereum/go-ethereum/crypto`, `github.com/ethereum/go-ethereum/signer/core/apitypes`, `math/big`, `strings`

### Resources to cite

- Solidity ABI specification: https://docs.soliditylang.org/en/latest/abi-spec.html
- EIP-712 typed data: https://eips.ethereum.org/EIPS/eip-712
- abi package: https://pkg.go.dev/github.com/ethereum/go-ethereum/accounts/abi
- Solidity panic codes: https://docs.soliditylang.org/en/latest/control-structures.html#panic-via-assert-and-error-via-require

---

## 24 — Type-Safe Contracts with `abigen`

**Lesson file:** [../24-abigen-bindings.md](../24-abigen-bindings.md) · **Examples folder:** `../examples/24-abigen-bindings/`

| | |
|---|---|
| Prerequisites | [23](../23-abi-encoding.md) |
| Unlocks | 26, 34 |
| Examples | **16** — 🟢 5 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 9 |

*generating Go bindings, deploying, calling, transacting, and the simulated backend*

### Goals

- Generate Go bindings for a contract with `abigen`.
- Deploy and interact with a contract entirely from Go.
- Use `CallOpts`/`TransactOpts` correctly, including historical reads.
- Test contract interaction against a simulated backend with no node running.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What abigen produces**

   - A `Deploy<C>` function, a `<C>` type with Caller/Transactor/Filterer embedded, and typed event structs.
   - Read methods become `func (c *C) Name(opts *bind.CallOpts, args...) (T, error)`.
   - Write methods become `func (c *C) Name(opts *bind.TransactOpts, args...) (*types.Transaction, error)`.
   - Events become `FilterName`, `WatchName` and `ParseName`.

**2. The toolchain**

   - `solc --abi --bin` → `abigen --abi X.abi --bin X.bin --pkg contracts --type C --out c.go`.
   - Or `forge build` then abigen against the artifact JSON.
   - Wiring it with `//go:generate` so `go generate ./...` regenerates everything.
   - Pinning the abigen version — the generated API has changed across go-ethereum releases.

**3. CallOpts**

   - `Pending` reads the node's pending state (avoid in production).
   - `BlockNumber` for historical reads — requires an archive node for old blocks.
   - `Context` for deadlines; `From` to simulate a caller for access-controlled views.
   - A nil `CallOpts` is legal and means 'latest, no caller' — usually not what you want.

**4. TransactOpts**

   - `Signer` func is mandatory; `bind.NewKeyedTransactorWithChainID` is the test-key path.
   - `Nonce`, `Value`, `GasLimit`, `GasFeeCap`, `GasTipCap` — leave nil to auto-fill, set them to control.
   - `NoSend: true` builds and signs without broadcasting — useful for simulation and for a signing service.
   - Reusing one `TransactOpts` across concurrent calls is a race and a nonce bug; clone per call.

**5. Deploying from Go**

   - `DeployC(auth, backend, ctorArgs...)` returns address, transaction and a bound instance.
   - The address is available immediately but the code is not — wait for the receipt.
   - `bind.WaitDeployed` and checking `receipt.Status`.
   - Constructor arguments are ABI-encoded and appended to the creation bytecode.

**6. Events through bindings**

   - `FilterTransfer(opts, from, to)` returns an iterator; remember to `Close()` it.
   - `WatchTransfer` needs a WebSocket backend and returns an `event.Subscription`.
   - The iterator's `Next()`/`Error()` pattern, and the leak if you abandon it.
   - When to drop to raw `FilterLogs` instead (lesson 25).

**7. The simulated backend**

   - `ethclient/simulated.NewBackend(alloc, opts...)` — an in-process chain, no node, no network.
   - `Commit()` mines a block; nothing is mined until you say so, which makes tests deterministic.
   - Funding accounts via `types.GenesisAlloc`; `AdjustTime` for time-dependent logic.
   - The API moved from `accounts/abi/bind/backends` — check your go-ethereum version.
   - Its limits: no real gas market, no reorgs, no other clients' quirks.

**8. When bindings get in the way**

   - Proxies: the ABI you have is the implementation's, the address is the proxy's — bind to the proxy anyway.
   - Multicall aggregation: you need raw `abi.Pack`, not a binding per call (lesson 26).
   - Contracts that return non-standard data (USDT's `transfer` with no return value) break generated decoders.
   - The escape hatch: `bind.NewBoundContract` with a hand-built ABI.

**9. Keeping generated code sane**

   - Put bindings in their own package/folder and exclude them from review diffs.
   - Regenerate reproducibly: pinned solc, pinned abigen, checked-in ABI files.
   - Do not hand-edit generated files — wrap them instead.
   - A thin domain wrapper so your business code never imports the generated package directly.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Sharing one `TransactOpts` across goroutines — nonce races and corrupted values.
- Forgetting `receipt.Status` after `WaitMined` and assuming a deployment succeeded.
- Binding generated code directly into domain logic, making it impossible to fake in tests.
- Abandoning a filter iterator without `Close()`.
- Assuming the simulated backend behaves like mainnet for gas or ordering.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Generate bindings for `Counter` and call `Number()` against `anvil`.
- Deploy `Counter` from Go and print the deployed address.
- Set up `//go:generate` and regenerate the bindings.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Deploy to a simulated backend, call `Increment()`, `Commit()`, and read the new value.
- Read an ERC-20 `balanceOf` at a historical block with `CallOpts.BlockNumber`.
- Use `NoSend: true` to build a signed transaction without broadcasting, then send it separately.
- Filter `Transfer` events with an iterator and close it properly.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Bind to a proxy address using the implementation ABI and call through it.
- Handle a token whose `transfer` returns nothing, without the generated decoder panicking.
- Write a `TokenReader` interface over the binding and provide a fake implementation for tests.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/ethclient/simulated`, `github.com/ethereum/go-ethereum/core/types`, `testing`

### Resources to cite

- go-ethereum — Go contract bindings: https://geth.ethereum.org/docs/developers/dapp-developer/native-bindings
- bind package: https://pkg.go.dev/github.com/ethereum/go-ethereum/accounts/abi/bind
- simulated backend: https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient/simulated

---

## 25 — Events, Logs & Indexing Them

**Lesson file:** [../25-events-logs.md](../25-events-logs.md) · **Examples folder:** `../examples/25-events-logs/`

| | |
|---|---|
| Prerequisites | [23](../23-abi-encoding.md) |
| Unlocks | 26, 31 |
| Examples | **19** — 🟢 5 easy, 🟡 9 medium, 🔴 5 hard |
| Topics | 9 |

*topics, the bloom filter, `eth_getLogs` at scale, decoding, subscriptions and reorg-safe consumption*

### Goals

- Decode an event log into typed Go values, including indexed parameters.
- Query logs efficiently with topic filters and chunked block ranges.
- Explain the logs bloom filter and what it is actually good for.
- Choose between polling and subscribing, and survive reorgs either way.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What a log is**

   - A contract-emitted record: cheap to write (375 gas + 375/topic + 8/byte), unreadable by contracts.
   - Logs exist for off-chain consumers. Your indexer is the intended audience.
   - They are committed to via receiptsRoot, so they are provable (lesson 17).
   - They are *not* state — you cannot query 'current value', only 'what happened'.

**2. Log structure**

   - address, topics [0..4]common.Hash, data []byte, plus blockNumber, blockHash, txHash, txIndex, logIndex, removed.
   - topic0 = keccak of the canonical event signature (unless the event is `anonymous`).
   - topics 1..3 = indexed parameters, each padded to 32 bytes.
   - data = ABI-encoded non-indexed parameters, exactly as in lesson 23.

**3. The indexed-dynamic-type trap**

   - An indexed `string` or `bytes` stores **keccak(value)**, not the value.
   - So you can filter by it but never recover it — a permanent data loss if the emitter did not also log it plainly.
   - Demonstrate it: emit an indexed string and try to read it back.
   - Design consequence: as an integrator, you sometimes cannot get what you need from logs alone.

**4. The bloom filter**

   - A 2048-bit filter per block over the log addresses and topics, in the header.
   - Probabilistic: 'definitely not here' or 'maybe here'. False positives by design.
   - Its real use is letting a node skip blocks fast — it is a prefilter, not an index.
   - Implement the bloom membership check in Go and measure the false-positive rate.

**5. eth_getLogs in practice**

   - `ethereum.FilterQuery{FromBlock, ToBlock, Addresses, Topics}`.
   - Topics is `[][]common.Hash`: position AND across slots, OR within a slot.
   - Provider block-range caps (2k–10k) and result-count caps; chunking is mandatory.
   - Adaptive chunking: halve the range on failure, grow it back on success (lesson 31).

**6. Decoding in Go**

   - `abi.ABI.Events[name].ID` for topic0; `UnpackIntoInterface` for the data section.
   - Indexed parameters must be decoded from topics manually — `abi.ParseTopics` helps.
   - Building a `map[common.Hash]abi.Event` dispatcher for multi-event contracts.
   - Unknown topic0: log it and move on, never crash the indexer.

**7. Polling vs subscribing**

   - `SubscribeFilterLogs`: low latency, but WebSocket connections die silently.
   - Polling `FilterLogs` on an interval: higher latency, trivially restartable, no gaps.
   - Production answer: poll as the source of truth, subscribe for latency, reconcile.
   - Never let a subscription be your only path to data.

**8. Reorgs and logs**

   - A subscription delivers `removed: true` logs when a block is reorged out.
   - `getLogs` simply stops returning them — you have to notice the block hash changed.
   - Rule: every effect from a log must be reversible, keyed by block.
   - Full treatment in lesson 31.

**9. Ordering and idempotency**

   - The only stable sort key is (blockNumber, logIndex). `txIndex` alone is not enough.
   - A natural unique key: (blockHash, logIndex) — survives reorgs by simply not matching.
   - Deduplication on replay; upserts rather than inserts.
   - Why you should store the raw log alongside the decoded row.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Expecting to read an indexed `string` parameter's value — you only get its hash.
- Requesting an unbounded block range and getting a silent truncation.
- Relying on a WebSocket subscription with no polling fallback.
- Using the bloom filter as an index and trusting a positive result without fetching.
- Keying rows on `txHash` alone when one transaction emits many logs.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 19).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Compute an event's topic0 and match it against a real `Transfer` log.
- Fetch logs for one contract over 100 blocks and print the count.
- Print a log's topics and data as hex.

**🟡 Medium — 9 examples** *(concepts combined, and the traps)*

- Decode a `Transfer` log into typed `from`, `to`, `value`.
- Filter by topic0 AND (topic1 IN [a, b]) and explain the semantics.
- Chunk a 50k-block query into 2k-block requests with progress output.
- Emit and read an indexed `string` and show the value is unrecoverable.
- Build a topic0 → decoder dispatcher for a contract with 5 event types.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Implement the bloom membership check and measure false positives over 1000 real blocks.
- Consume logs via subscription and polling simultaneously and reconcile the two streams.
- Handle a `removed: true` log by reversing its effect on local state.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi`, `github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/core/types`, `github.com/ethereum/go-ethereum`

### Resources to cite

- Ethereum docs — events and logs: https://ethereum.org/en/developers/docs/smart-contracts/anatomy/#events-and-logs
- JSON-RPC eth_getLogs: https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_getlogs
- Yellow Paper (bloom filter definition): https://ethereum.github.io/yellowpaper/paper.pdf

---

## 26 — The ERC Standards from Go

**Lesson file:** [../26-erc-standards.md](../26-erc-standards.md) · **Examples folder:** `../examples/26-erc-standards/`

| | |
|---|---|
| Prerequisites | [24](../24-abigen-bindings.md), [25](../25-events-logs.md) |
| Unlocks | 27, 40, 49, 52, 55 |
| Examples | **19** — 🟢 5 easy, 🟡 9 medium, 🔴 5 hard |
| Topics | 9 |

*ERC-20, ERC-721, ERC-1155, ERC-165 and Multicall — including every non-standard token that breaks your code*

### Goals

- Interact with ERC-20, ERC-721 and ERC-1155 tokens from Go.
- Handle decimals, non-standard tokens and missing return values.
- Detect interface support with ERC-165.
- Batch reads to cut RPC calls by orders of magnitude.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why standards exist**

   - A shared ABI is what makes wallets, DEXs, explorers and indexers possible at all.
   - The ERC process, and the difference between a standard and a widely-copied implementation.
   - OpenZeppelin as the de facto reference implementation.
   - Standards are advisory: nothing forces a contract to behave.

**2. ERC-20**

   - `totalSupply`, `balanceOf`, `transfer`, `transferFrom`, `approve`, `allowance` + `Transfer`/`Approval` events.
   - Optional metadata: `name`, `symbol`, `decimals` — optional, and some tokens omit them.
   - The allowance race: `approve` from non-zero to non-zero, and the increase/decrease convention.
   - Infinite approval as a UX/risk trade-off, and why integrators should cap.

**3. The non-standard token minefield**

   - USDT and others return **nothing** from `transfer` — a generated binding expecting `bool` fails.
   - Fee-on-transfer: you receive less than you sent. Always measure balance deltas.
   - Rebasing tokens: balances change with no `Transfer` event — indexers silently drift.
   - Blocklists, pausability, and upgradeable tokens whose behaviour changes under you.
   - Non-18 decimals: USDC/USDT are 6, WBTC is 8. Hardcoding 18 is a real production incident.

**4. Decimals are display-only**

   - All arithmetic happens in the smallest unit with `*big.Int`. Decimals affect *rendering* only.
   - Formatting with `big.Rat` or integer string surgery; parsing user input with a decimal cap.
   - A `TokenAmount{Raw *big.Int; Decimals uint8}` type so units cannot be mixed.
   - Never store a formatted string as the source of truth.

**5. ERC-721**

   - `ownerOf`, `balanceOf`, `safeTransferFrom`, `approve`, `setApprovalForAll`, `tokenURI`.
   - The receiver hook (`onERC721Received`) and why `safeTransferFrom` can revert.
   - Enumerable and Metadata extensions — and that most collections skip Enumerable.
   - Reconstructing ownership from `Transfer` events, including mints (from=0) and burns (to=0).

**6. ERC-1155**

   - Multi-token: one contract, many ids, fungible and non-fungible together.
   - `balanceOf(account, id)`, `balanceOfBatch`, `safeBatchTransferFrom`.
   - Different event shape: `TransferSingle` and `TransferBatch` — your indexer needs both.
   - The `{id}` URI substitution rule.

**7. ERC-165 interface detection**

   - `supportsInterface(bytes4)` where the id is the XOR of the interface's selectors.
   - Computing an interface id in Go and checking a contract.
   - Its limits: contracts can lie, and many older contracts do not implement it at all.
   - Fallback heuristics: try a call and see whether it reverts.

**8. Batching reads**

   - Multicall3 (`0xcA11bde05977b3631167028862bE2a173976CA11`, same address on most chains) — aggregate N calls into one `eth_call`.
   - Building the aggregate calldata in Go with `abi.Pack` and decoding the array of results.
   - `tryAggregate` so one failing call does not fail the batch.
   - The alternative: `rpc.BatchCallContext` (lesson 20) — compare round trips and gas-free-ness.

**9. Token accounting for an indexer**

   - Balances from `Transfer` events: mints, burns, and the zero address.
   - Why the derived balance can drift (rebasing, direct storage writes by the contract, fee-on-transfer).
   - Periodic reconciliation against `balanceOf` (lesson 35).
   - Storing both the event-derived and the on-chain balance, and alerting on divergence.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Hardcoding 18 decimals — USDC is 6 and this has cost real money.
- Assuming `transfer` returns a bool; USDT does not, and your binding will error.
- Trusting the amount you sent equals the amount received (fee-on-transfer).
- Deriving balances purely from events for a rebasing token.
- Calling `balanceOf` in a loop for 500 addresses instead of batching.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 19).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Read `name`, `symbol`, `decimals` and `totalSupply` of a mainnet ERC-20.
- Read one address's `balanceOf` and format it with the right decimals.
- Compute an ERC-165 interface id in Go.

**🟡 Medium — 9 examples** *(concepts combined, and the traps)*

- Format a raw USDC balance (6 decimals) into a human string with `big.Rat`.
- Handle a token whose `transfer` returns no value without panicking.
- Detect fee-on-transfer by simulating a transfer with `eth_call` and comparing balance deltas.
- Reconstruct an ERC-721 collection's current owners from `Transfer` logs.
- Decode `TransferSingle` and `TransferBatch` from an ERC-1155 contract.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Batch 100 `balanceOf` reads into one Multicall3 `eth_call` and compare latency with 100 calls.
- Use `tryAggregate` so a reverting call in the batch does not lose the others.
- Detect token type (20/721/1155) for an unknown address using ERC-165 plus call heuristics.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/accounts/abi`, `math/big`, `github.com/ethereum/go-ethereum/ethclient`

### Resources to cite

- ERC-20: https://eips.ethereum.org/EIPS/eip-20
- ERC-721: https://eips.ethereum.org/EIPS/eip-721
- ERC-1155: https://eips.ethereum.org/EIPS/eip-1155
- ERC-165: https://eips.ethereum.org/EIPS/eip-165
- Multicall3: https://github.com/mds1/multicall

---

## 27 — Smart Contract Security

**Lesson file:** [../27-contract-security.md](../27-contract-security.md) · **Examples folder:** `../examples/27-contract-security/`

| | |
|---|---|
| Prerequisites | [26](../26-erc-standards.md) |
| Unlocks | 40, 45, 46 |
| Examples | **19** — 🟢 5 easy, 🟡 9 medium, 🔴 5 hard |
| Topics | 11 |

*the bug classes that drain contracts, the real incidents, and how a Go integrator defends itself*

### Goals

- Recognise the major on-chain vulnerability classes and their real-world incidents.
- Apply checks-effects-interactions and reentrancy guards.
- Audit an integration for the risks *your* Go service creates.
- Use Slither, fuzzing and invariant tests at a working level.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Reentrancy**

   - The DAO (2016, ~3.6M ETH) — the incident that caused the ETH/ETC split.
   - Mechanism: an external call hands control to the callee before state is updated.
   - Checks–Effects–Interactions, and `ReentrancyGuard` as the belt to CEI's braces.
   - Cross-function and **read-only reentrancy** (a view function returning stale state mid-call) — the modern variant.
   - Build it, exploit it, fix it, re-run the exploit — all on a simulated backend.

**2. Access control**

   - Missing `onlyOwner`; public functions that should be internal.
   - Uninitialized proxies — Parity multisig (2017), 513k ETH frozen.
   - `tx.origin` vs `msg.sender` and the phishing attack that distinguishes them.
   - Role-based access (OpenZeppelin `AccessControl`) and the admin-key centralisation question.

**3. Arithmetic**

   - Pre-0.8: silent overflow (the 2018 `batchOverflow` token bugs).
   - 0.8+: checked by default, so the risk moved into `unchecked` blocks.
   - Precision loss: divide last, multiply first; rounding direction always favours the protocol.
   - Casting down (`uint256` → `uint128`) silently truncating.

**4. External calls**

   - Unchecked low-level `call` return values — the call fails, your code continues.
   - `transfer`/`send` and the 2300-gas stipend that breaks with contract recipients.
   - Push vs pull payments; a single reverting recipient DoSing a distribution loop.
   - Gas griefing and return-bomb attacks (huge returndata to exhaust gas).

**5. Oracle manipulation**

   - Spot price from an AMM reserve is manipulable within one transaction.
   - Flash loans make manipulation capital-free — bZx, Harvest, Cream and many others.
   - TWAPs, Chainlink feeds, and their own failure modes (stale rounds, depeg, L2 sequencer downtime).
   - Full treatment in lesson 53.

**6. Front-running and MEV as a security property**

   - The mempool is public; your transaction is a public intent.
   - Sandwich attacks on swaps; back-running liquidations; NFT mint sniping.
   - Defences: slippage bounds, deadlines, commit–reveal, private mempools (lesson 40).
   - As an integrator: never send a swap without `amountOutMin` computed off-chain.

**7. Signature bugs**

   - Replay across chains (missing chainId), across contracts (missing verifyingContract), or in time (missing nonce/deadline).
   - `ecrecover` returns address(0) on malformed input — comparing against an uninitialized variable passes.
   - Malleability (lesson 06) and why EIP-2 low-s matters here.
   - EIP-1271 contract signatures and the extra validation they require (lesson 50).

**8. Proxies and upgrades**

   - Storage collisions between proxy and implementation; unstructured storage slots (EIP-1967).
   - Function selector clashes between proxy and implementation.
   - Initializer discipline: `initializer` modifier, disabling initializers in the implementation.
   - Full treatment in lesson 46.

**9. Randomness**

   - `block.timestamp`, `blockhash`, `prevrandao` are all miner/proposer-influenceable.
   - The NFT mint that was gamed by reverting on an unwanted outcome.
   - VRFs and commit–reveal as the real answers (lesson 53).
   - As an integrator: assume any on-chain randomness you did not verify is adversarial.

**10. Integrator-side defence in Go**

   - Simulate with `eth_call` before every state-changing send; abort if the result moved.
   - Verify receipts *and* the emitted events, never assume a call did what it says.
   - Validate token behaviour on first sight (decimals, return values, transfer deltas).
   - Cap approvals; never grant infinite allowance from a hot wallet you control.
   - Treat every external address as hostile: bound gas, bound value, allowlist destinations.
   - Deal with upgradeable contracts: pin implementation addresses and alert on change.

**11. Tooling**

   - Slither for static analysis; what its common detectors actually catch.
   - Foundry fuzzing and invariant tests (lesson 48).
   - Echidna, Mythril, and the limits of automated tools.
   - What an audit buys and what it does not; bug bounties; the security checklist for this repo.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming a successful transaction means the intended effect happened — verify the events.
- Granting unlimited approvals from a service-controlled hot wallet.
- Integrating an upgradeable token and never noticing the implementation changed.
- Trusting `eth_call` results from unverified contracts without simulating the whole flow.
- Building slippage protection on-chain values you read one block earlier.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 19).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Deploy a vulnerable vault on a simulated backend and read its balance.
- Write the attacker contract's `receive()` that re-enters.
- Show an unchecked `call` returning false while execution continues.

**🟡 Medium — 9 examples** *(concepts combined, and the traps)*

- Drain the vault with the reentrant attacker, then fix it with checks-effects-interactions and re-run.
- Demonstrate a `tx.origin` phishing attack and the `msg.sender` fix.
- Replay an EIP-712 signature on a second chain id to show missing domain separation.
- Show precision loss from dividing before multiplying, with real numbers.
- From Go: detect a fee-on-transfer token by simulating and comparing balance deltas.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Manipulate a spot-price oracle inside one transaction and drain a lending pool on a simulated chain.
- Build a Go pre-flight checker: simulate, verify expected events, and refuse to send on mismatch.
- Monitor an EIP-1967 implementation slot and alert when a proxy's implementation changes.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient/simulated`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `testing`, `os/exec`

### Resources to cite

- Smart Contract Weakness Classification: https://swcregistry.io/
- OpenZeppelin security docs: https://docs.openzeppelin.com/contracts/5.x/security-considerations
- Ethereum smart contract security: https://ethereum.org/en/developers/docs/smart-contracts/security/
- Slither: https://github.com/crytic/slither
- rekt.news incident archive: https://rekt.news/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
