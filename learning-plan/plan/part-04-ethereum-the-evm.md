# Part 4 — Ethereum & the EVM

From your toy chain to the real one. Ethereum's architecture, RLP and the Patricia trie, the EVM as a stack machine you build yourself, the transaction types, and talking to a node from Go.

**Core spine.** Lessons 16–21 · 110 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 16 | [Ethereum Architecture](#16-ethereum-architecture) | 15 | 16 |
| 17 | [RLP & the Merkle Patricia Trie](#17-rlp-the-merkle-patricia-trie) | 05, 16 | 18 |
| 18 | [The EVM: a Stack Machine You Can Build](#18-the-evm-a-stack-machine-you-can-build) | 17 | 22 |
| 19 | [Ethereum Transactions Deep Dive](#19-ethereum-transactions-deep-dive) | 06, 17 | 18 |
| 20 | [JSON-RPC & go-ethereum's `ethclient`](#20-json-rpc-go-ethereums-ethclient) | 16 | 18 |
| 21 | [Sending Transactions from Go](#21-sending-transactions-from-go) | 19, 20 | 18 |

---

## 16 — Ethereum Architecture

**Lesson file:** [../16-ethereum-architecture.md](../16-ethereum-architecture.md) · **Examples folder:** `../examples/16-ethereum-architecture/`

| | |
|---|---|
| Prerequisites | [15](../15-account-model-state.md) |
| Unlocks | 17, 20, 28 |
| Examples | **16** — 🟢 5 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 10 |

*the whole machine: accounts, blocks, receipts, gas, the execution/consensus split, blobs and upgrades*

### Goals

- Draw Ethereum end to end, from a signed transaction to a finalized state change.
- Read a real block header field by field.
- Explain gas, the EIP-1559 fee market, and where the money goes.
- Describe the post-Merge client split and the Engine API.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The world computer framing**

   - One shared state machine, replicated by thousands of nodes, with metered execution.
   - Ethereum is slow and expensive *by construction* — you are paying every node to agree.
   - What that buys: credible neutrality, composability, no admin.
   - Where the framing misleads: it is not a computer you run programs on cheaply.

**2. Block anatomy post-Merge**

   - parentHash, stateRoot, transactionsRoot, receiptsRoot, logsBloom, number, gasLimit, gasUsed, timestamp.
   - baseFeePerGas (1559), withdrawalsRoot (Shanghai), blobGasUsed/excessBlobGas (Cancun).
   - Fields that became vestigial after the Merge: difficulty=0, mixHash=prevRandao, nonce=0.
   - Reading each one from a real block with `ethclient` and printing units.

**3. Three roots, three tries**

   - stateRoot commits to every account and every storage slot.
   - transactionsRoot commits to the ordered transaction list.
   - receiptsRoot commits to what each transaction *did* (status, gas, logs).
   - Each supports a Merkle proof — the basis of light clients and cross-chain messaging.

**4. Receipts**

   - status (1 success / 0 revert), cumulativeGasUsed, gasUsed, effectiveGasPrice, logs, logsBloom.
   - contractAddress on deployment; type; blobGasUsed/blobGasPrice for type 3.
   - **A receipt with status 0 still consumed gas and still exists** — success is never assumed.
   - This is the single most important object for a backend engineer.

**5. Gas**

   - Why metering exists: the halting problem, and DoS resistance.
   - gasLimit (per transaction and per block), gasUsed, out-of-gas semantics, the 21000 floor.
   - Intrinsic gas: 21000 + calldata cost (EIP-2028: 4 per zero byte, 16 per non-zero).
   - Gas refunds (SSTORE clearing) and the EIP-3529 cap.

**6. EIP-1559**

   - baseFeePerGas is burned; priorityFee (tip) goes to the proposer.
   - Base fee adjusts ±12.5% per block toward 50% block utilisation — implement the formula in Go.
   - maxFeePerGas vs maxPriorityFeePerGas; effectiveGasPrice = min(maxFee, baseFee + tip).
   - Why the refund of (maxFee − effective) makes overpaying maxFee safe.

**7. The execution/consensus split**

   - Execution layer (geth, erigon, nethermind, reth) — state, EVM, mempool, JSON-RPC.
   - Consensus layer (prysm, lighthouse, teku, nimbus) — slots, attestations, finality.
   - The Engine API over authenticated localhost with a JWT secret: `newPayload`, `forkchoiceUpdated`.
   - Running a node means running two processes; this surprises people (lesson 33).

**8. Slots, epochs and finality tags**

   - 12-second slots, 32-slot epochs; one proposer per slot, committees attesting.
   - The `latest` / `safe` / `finalized` block tags and what each guarantees.
   - A missed slot means an empty 12 seconds — your indexer must tolerate it.
   - Full treatment in lesson 28; here you only need enough to read an explorer.

**9. Blobs (EIP-4844)**

   - Blob-carrying transactions, KZG commitments, and a separate blob-gas market.
   - Blobs are pruned after ~18 days — they are for data *availability*, not storage.
   - Why this made L2s an order of magnitude cheaper (lesson 30).
   - Reading blobGasUsed and excessBlobGas from a header.

**10. Upgrades and EIPs**

   - How to read an EIP: motivation, specification, rationale, backwards compatibility.
   - Fork names and roughly what each changed: Homestead → Byzantium → Istanbul → London → Merge → Shanghai → Cancun → Prague.
   - Where to check what is live: the network upgrade page and client release notes.
   - Why your Go code must never assume a fixed set of transaction types.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Assuming a receipt means success — always check `receipt.Status`.
- Using `latest` for anything financial; it can be reorged out. Use `finalized` or a confirmation depth.
- Treating `gasPrice` as meaningful on a type-2 transaction — read `effectiveGasPrice` from the receipt.
- Assuming 12-second blocks; slots get missed and your `blockNumber + 1` polling stalls.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Fetch a block header and print every field with units.
- Fetch a receipt and print status, gas used and log count.
- Print the difference between `latest`, `safe` and `finalized` block numbers.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Compute effectiveGasPrice for a type-2 transaction and split burn from tip.
- Implement the base-fee update formula and verify it against 20 consecutive real blocks.
- Compute intrinsic gas for a transaction from its calldata and compare with the receipt.
- Detect a missed slot by comparing block timestamps.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Reconstruct total ETH burned in a block from its transactions and compare with an explorer.
- Decode blobGasUsed/excessBlobGas and compute the blob base fee.
- Walk 100 blocks and chart base fee against gas used to show the 50%-target feedback loop.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/core/types`, `math/big`, `context`

### Resources to cite

- Ethereum Yellow Paper: https://ethereum.github.io/yellowpaper/paper.pdf
- EIP-1559: https://eips.ethereum.org/EIPS/eip-1559
- EIP-4844 (blobs): https://eips.ethereum.org/EIPS/eip-4844
- Engine API spec: https://github.com/ethereum/execution-apis/tree/main/src/engine
- Ethereum network upgrades: https://ethereum.org/en/history/

---

## 17 — RLP & the Merkle Patricia Trie

**Lesson file:** [../17-rlp-merkle-patricia-trie.md](../17-rlp-merkle-patricia-trie.md) · **Examples folder:** `../examples/17-rlp-merkle-patricia-trie/`

| | |
|---|---|
| Prerequisites | [05](../05-merkle-trees.md), [16](../16-ethereum-architecture.md) |
| Unlocks | 18, 19, 30, 62, 64, 66 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 11 |

*Ethereum's serialization format and the trie that produces every root in the header*

### Goals

- Encode and decode RLP by hand and with go-ethereum's `rlp` package.
- Explain the MPT node types and hex-prefix encoding.
- Build a small trie and compute its root.
- Verify an `eth_getProof` result against a state root.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What RLP is and is not**

   - It encodes exactly two things: byte strings and lists of them. Nothing else.
   - No types, no field names, no schema — the decoder must already know the shape.
   - Why Ethereum chose it: minimal, deterministic, trivially implementable in any language.
   - Where it is used: transactions, headers, account bodies, trie nodes, devp2p messages.

**2. The RLP rules in full**

   - Single byte < 0x80 encodes as itself.
   - String 0–55 bytes: 0x80+len, then the bytes.
   - String >55 bytes: 0xb7+len(len), then len big-endian, then the bytes.
   - Lists mirror this with 0xc0 and 0xf7. Work each case through by hand.
   - Canonical form: exactly one valid encoding per value — non-canonical input must be rejected.

**3. Encoding integers**

   - Big-endian, minimal length, no leading zeros. Zero encodes as the *empty string* (0x80).
   - Consequence: you cannot distinguish 0 from "" from nil without schema knowledge.
   - This trips up every hand-written encoder; write the test.
   - Negative numbers are not representable — RLP has no signed form.

**4. go-ethereum's rlp package**

   - `rlp.Encode`/`rlp.EncodeToBytes`, `rlp.Decode`/`rlp.DecodeBytes`.
   - Struct encoding follows field order; unexported fields are skipped.
   - Tags: `rlp:"nil"`, `rlp:"optional"`, `rlp:"tail"` and when each is needed.
   - `rlp.RawValue` for pass-through, and streaming with `rlp.NewStream`.

**5. Why a plain Merkle tree is not enough**

   - State is a mutable key→value map with millions of entries, not a fixed ordered list.
   - You need: O(log n) updates, deterministic root regardless of insertion order, and key lookup.
   - A radix trie gives lookup; Merkle hashing gives the commitment. Combine them.
   - Determinism requirement: two nodes with the same data must produce the same root, always.

**6. MPT node types**

   - Leaf: [encodedPath, value]. Extension: [encodedPath, nextNodeRef]. Branch: 17 items (16 nibbles + value).
   - Keys are nibble paths (each byte → two 4-bit nibbles), so branching factor is 16.
   - Extension nodes exist purely to compress long shared paths.
   - Drawing the trie for a handful of keys is the fastest way to understand it.

**7. Hex-prefix (compact) encoding**

   - Nibble paths must be packed into bytes; odd lengths need a flag.
   - The prefix nibble encodes {leaf?, odd-length?} — four combinations, worth memorising.
   - Implement `hexToCompact`/`compactToHex` in Go and test both directions.
   - Getting this wrong produces a valid-looking trie with the wrong root.

**8. Node hashing and the inline rule**

   - A node's reference is keccak256(rlp(node)) — unless the RLP is under 32 bytes, then it is inlined.
   - The empty trie root is a fixed constant: keccak256(rlp("")).
   - Why the inline rule exists (space) and why it complicates proof verification.
   - Caching hashes and the dirty-node marking that geth's trie does.

**9. The four tries**

   - World state trie: keccak(address) → rlp(account). Note the *hashed* key (secure trie).
   - Storage trie per contract: keccak(slot) → rlp(value).
   - Transactions trie and receipts trie: rlp(index) → rlp(item), rebuilt per block.
   - Why the state trie hashes keys: to bound trie depth against adversarial address grinding.

**10. Proofs**

   - `eth_getProof` returns an account proof and per-slot storage proofs.
   - Verification: walk the node list, checking each hash and following the nibble path.
   - This is a *trustless read* — you do not have to trust the RPC provider (lesson 64).
   - Cross-chain messaging and rollup withdrawals are built on exactly this (lesson 67).

**11. What comes next**

   - Verkle tries: vector commitments, much smaller proofs, stateless clients (lesson 66).
   - Why the migration is hard and how long it has been coming.
   - One paragraph, so you recognise the terms.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Encoding integers with leading zeros — a non-canonical encoding that yields a different hash.
- Forgetting that the state trie keys are keccak(address), not address.
- Ignoring the <32-byte inline-node rule during proof verification.
- Assuming `rlp` handles maps or pointers-to-interfaces; it does not.
- Building a trie by insertion order and expecting a stable root without the canonical rules.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- RLP-encode `"dog"`, `[]`, `[[],[[]]]` and check against the spec's vectors.
- Encode integers 0, 15, 1024 and explain each byte.
- Round-trip a struct through `rlp.EncodeToBytes`/`DecodeBytes`.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Hand-write an RLP encoder for strings and lists and diff against `rlp.EncodeToBytes` on 20 inputs.
- Decode a real mainnet block header from raw RLP.
- Implement `hexToCompact`/`compactToHex` and test all four prefix cases.
- Insert 4 key/value pairs into a trie and print the root after each insert.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Build a transactions trie for a real block and confirm the root matches the header.
- Fetch `eth_getProof` for a storage slot and verify it against the block's stateRoot, by hand.
- Show that inserting the same pairs in a different order yields the same root.

### Packages & tools

`github.com/ethereum/go-ethereum/rlp`, `github.com/ethereum/go-ethereum/trie`, `github.com/ethereum/go-ethereum/common`, `github.com/ethereum/go-ethereum/crypto`

### Resources to cite

- RLP spec: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/
- Merkle Patricia Trie: https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/
- EIP-1186 (eth_getProof): https://eips.ethereum.org/EIPS/eip-1186
- geth trie package: https://pkg.go.dev/github.com/ethereum/go-ethereum/trie

---

## 18 — The EVM: a Stack Machine You Can Build

**Lesson file:** [../18-evm.md](../18-evm.md) · **Examples folder:** `../examples/18-evm/`

| | |
|---|---|
| Prerequisites | [17](../17-rlp-merkle-patricia-trie.md) |
| Unlocks | 22, 46, 62, 65 |
| Examples | **22** — 🟢 6 easy, 🟡 10 medium, 🔴 6 hard |
| Topics | 11 |

*opcodes, stack, memory, storage, calldata, gas accounting — and a mini-EVM written in Go*

### Goals

- Explain the EVM's execution model: 256-bit stack, volatile memory, persistent storage.
- Read raw bytecode and disassemble it.
- Implement an interpreter for a useful subset of opcodes in Go.
- Trace gas consumption and explain why storage is expensive.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The machine**

   - A stack of at most 1024 words, each 256 bits. No registers.
   - Byte-addressed, zero-initialised, volatile memory that grows on demand.
   - Persistent key→value storage per contract account, 256-bit keys and values.
   - A program counter over a `[]byte` of bytecode. That is the entire machine.

**2. Why 256 bits**

   - Hashes and addresses fit natively; no bignum library needed in consensus rules.
   - The cost: on a 64-bit CPU every operation is four words — and in Go you need `uint256`.
   - Wrapping arithmetic (mod 2²⁵⁶) — this is why Solidity <0.8 overflowed silently.
   - Signed operations use two's complement over the same 256 bits (SDIV, SMOD, SAR).

**3. Data locations and their costs**

   - Stack: 3 gas, effectively free. Memory: 3 gas/word plus **quadratic** expansion cost.
   - Storage: the expensive one. SSTORE 20000 (zero→nonzero), 2900 (update), refunds on clear.
   - EIP-2929 warm/cold: first access to a slot or address costs 2100/2600, later ones 100.
   - Calldata: read-only, cheap (CALLDATALOAD 3); code and returndata.

**4. Opcode families**

   - Arithmetic: ADD, MUL, SUB, DIV, MOD, ADDMOD, MULMOD, EXP, SIGNEXTEND.
   - Comparison/bitwise: LT, GT, SLT, SGT, EQ, ISZERO, AND, OR, XOR, NOT, BYTE, SHL, SHR, SAR.
   - Environment: ADDRESS, BALANCE, CALLER, CALLVALUE, CALLDATA*, GASPRICE, CHAINID, SELFBALANCE.
   - Block: BLOCKHASH, COINBASE, TIMESTAMP, NUMBER, PREVRANDAO, GASLIMIT, BASEFEE, BLOBBASEFEE.
   - Memory/storage: MLOAD, MSTORE, MSTORE8, MSIZE, SLOAD, SSTORE, TLOAD/TSTORE (EIP-1153).
   - Control: JUMP, JUMPI, JUMPDEST, PC, STOP, RETURN, REVERT, INVALID.
   - System: CREATE, CREATE2, CALL, CALLCODE, DELEGATECALL, STATICCALL, SELFDESTRUCT.
   - Logging: LOG0–LOG4 (lesson 25).

**5. JUMPDEST analysis**

   - Jumps may only land on a JUMPDEST byte that is not inside PUSH data.
   - You must pre-scan the code to build a valid-destination bitmap — do this once, cache it.
   - This is what prevents jump-oriented programming against contracts.
   - The subtlety: a 0x5B byte inside PUSH32 data is not a JUMPDEST.

**6. Contract creation**

   - A creation transaction runs *init code*, whose RETURN value becomes the runtime code.
   - CREATE address = keccak(rlp([sender, nonce]))[12:]; CREATE2 uses a salt and the init-code hash.
   - EIP-170 code size limit (24576 bytes) and EIP-3860 init-code limit.
   - Counterfactual deployment: you can know a CREATE2 address before deploying (lesson 47).

**7. The call family**

   - CALL: new context, new storage, value transfer allowed.
   - DELEGATECALL: **caller's** storage and address, callee's code — the entire proxy mechanism (lesson 46).
   - STATICCALL: forbids state changes; how `view` functions are enforced at the EVM level.
   - CALLCODE is deprecated. Value-bearing calls get a 2300-gas stipend.

**8. Gas mechanics**

   - The 63/64 rule (EIP-150): a call can forward at most 63/64 of remaining gas.
   - Out-of-gas consumes everything and reverts the frame — no refund of the work.
   - REVERT (returns data, refunds remaining gas) vs INVALID (consumes all) vs STOP.
   - Revert reasons: `Error(string)` ABI-encoded in the return data (lesson 23).

**9. Writing an interpreter in Go**

   - The loop: fetch opcode, look up handler in a jump table, charge gas, execute, advance PC.
   - Types: `Stack` with push/pop/peek and depth checks, `Memory` with expansion accounting.
   - An explicit `GasMeter` — never let gas accounting hide inside handlers.
   - Structured errors: `ErrStackUnderflow`, `ErrOutOfGas`, `ErrInvalidJump`.
   - Compare your implementation against geth's `core/vm` for the same bytecode.

**10. Reading real traces**

   - `debug_traceTransaction` structured logs: pc, op, gas, gasCost, stack, memory, storage.
   - callTracer and prestateTracer, and which providers expose them.
   - Finding where gas actually went — the practical skill this lesson buys you.
   - `cast run` as the fast local equivalent.

**11. The road ahead**

   - Transient storage (EIP-1153) and what it replaces.
   - EOF (EVM Object Format): code/data separation, static jumps, versioning.
   - Account abstraction changing what a 'sender' is (lesson 47).
   - Precompiles as the escape hatch for expensive cryptography (lesson 65).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using `big.Int` in the interpreter loop and allocating on every opcode — use `uint256`.
- Forgetting that arithmetic wraps mod 2²⁵⁶.
- Charging memory expansion linearly; it is quadratic and that is a real DoS boundary.
- Allowing a jump into PUSH data because you skipped JUMPDEST analysis.
- Modelling DELEGATECALL as a normal call — it is the single most consequential opcode for security.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 22).

**🟢 Easy — 6 examples** *(one concept in isolation)*

- Disassemble `0x6001600201` byte by byte.
- Implement PUSH/POP with a stack printed after each step.
- Add ADD, MUL and SUB with wrapping arithmetic.

**🟡 Medium — 10 examples** *(concepts combined, and the traps)*

- Implement DUP/SWAP and stack depth limits with proper errors.
- Add MSTORE/MLOAD with quadratic memory-expansion gas.
- Implement JUMPDEST analysis and reject a jump into PUSH data.
- Add SSTORE/SLOAD with EIP-2929 warm/cold accounting and print the cost difference.
- Implement REVERT and return an ABI-encoded `Error(string)` reason.

**🔴 Hard — 6 examples** *(real-shaped, multi-concept programs)*

- Run a real compiled contract's runtime bytecode through your interpreter and match geth's result.
- Implement CALL and DELEGATECALL with the 63/64 rule and nested gas accounting.
- Produce a structured trace in the same shape as `debug_traceTransaction` and diff it against the node's.

### Packages & tools

`github.com/holiman/uint256`, `github.com/ethereum/go-ethereum/core/vm`, `github.com/ethereum/go-ethereum/core/asm`, `github.com/ethereum/go-ethereum/crypto`

### Resources to cite

- evm.codes (opcode reference): https://www.evm.codes/
- Ethereum Yellow Paper (EVM): https://ethereum.github.io/yellowpaper/paper.pdf
- EIP-2929 (cold/warm access): https://eips.ethereum.org/EIPS/eip-2929
- EIP-150 (63/64 rule): https://eips.ethereum.org/EIPS/eip-150
- geth core/vm: https://github.com/ethereum/go-ethereum/tree/master/core/vm

---

## 19 — Ethereum Transactions Deep Dive

**Lesson file:** [../19-transaction-types.md](../19-transaction-types.md) · **Examples folder:** `../examples/19-transaction-types/`

| | |
|---|---|
| Prerequisites | [06](../06-keys-signatures.md), [17](../17-rlp-merkle-patricia-trie.md) |
| Unlocks | 21 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*legacy, 2930, 1559 and 4844 types — signing, RLP, chain id, hashing and sender recovery*

### Goals

- Construct and sign every live Ethereum transaction type in Go.
- Explain EIP-155 chain-id replay protection and the `v` encoding.
- Compute a transaction hash and recover the sender from raw bytes.
- Choose the right type and fee fields for a situation.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The typed-transaction envelope**

   - EIP-2718: a leading type byte (0x01, 0x02, 0x03), then type-specific RLP.
   - Legacy transactions have no type byte — they start with an RLP list prefix (≥0xc0).
   - That is how a decoder tells them apart; implement the discriminator in Go.
   - Why this was introduced: adding transaction formats without breaking every parser.

**2. Type 0 — legacy**

   - Fields: nonce, gasPrice, gasLimit, to, value, data, v, r, s.
   - `to == nil` means contract creation.
   - Still valid and still used; your code must handle it forever.
   - Signing payload: rlp([nonce, gasPrice, gasLimit, to, value, data]) pre-155.

**3. EIP-155 replay protection**

   - Sign over rlp([..., chainId, 0, 0]); encode v = 35 + 2·chainId + parity.
   - Motivation: after the DAO fork, ETH transactions replayed on ETC and drained accounts.
   - Extracting chainId back out of v, and detecting an unprotected (v ∈ {27,28}) transaction.
   - Why every signer in this course must be chain-id aware.

**4. Type 1 — EIP-2930**

   - Adds chainId as a first-class field and an accessList of (address, storageKeys).
   - The access list pre-warms slots, converting cold (2600/2100) costs into warm (100).
   - Rarely worth it by hand; matters for contracts that touch many known slots.
   - v is now a plain 0/1 parity bit, not the 155 encoding.

**5. Type 2 — EIP-1559**

   - Fields: chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList.
   - effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas).
   - You are refunded maxFee − effective, so a generous maxFee is safe and a low one is not.
   - The inclusion condition: maxFeePerGas ≥ baseFee. Below that you simply wait.

**6. Type 3 — EIP-4844**

   - Adds maxFeePerBlobGas and blobVersionedHashes; the blobs travel in a sidecar, not the payload.
   - KZG commitments and proofs; versioned hash = 0x01 ‖ sha256(commitment)[1:].
   - Blob gas is a separate market with its own exponential pricing.
   - Almost exclusively used by rollups (lesson 30); you mostly need to *decode* these.

**7. The signing hash vs the transaction hash**

   - Signing hash: keccak of (typeByte ‖ rlp(payload-without-signature)).
   - Transaction hash: keccak of the *full* signed encoding, including v/r/s.
   - These are different digests and confusing them is a top-5 beginner bug.
   - In Go: `types.Signer.Hash(tx)` vs `tx.Hash()`.

**8. The go-ethereum API**

   - `types.NewTx(&types.DynamicFeeTx{...})` and the sibling `LegacyTx`, `AccessListTx`, `BlobTx`.
   - `types.LatestSignerForChainID`, `types.SignTx`, `types.Sender`.
   - Signer types: `HomesteadSigner`, `EIP155Signer`, `LondonSigner`, `CancunSigner` — pick by fork.
   - `tx.MarshalBinary()` for the raw bytes you broadcast with `eth_sendRawTransaction`.

**9. Sender recovery**

   - There is no `from` field on the wire — it is recovered from (r, s, v) and the signing hash.
   - Implement it end to end and compare against `types.Sender`.
   - Consequence: a malformed signature yields a *valid but wrong* sender, not an error.
   - Why you validate the recovered sender against what you expected.

**10. Practical rules**

   - gasLimit vs `eth_estimateGas`: add headroom, because state moves between estimate and inclusion.
   - Nonce gaps stall everything behind them (lesson 21).
   - Contract creation: `to == nil`, and the address comes from sender+nonce (lesson 07).
   - Data-only transactions, self-sends for cancellation, and zero-value calls.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Signing with the wrong signer for the fork, producing an invalid `v`.
- Confusing the signing hash with the transaction hash.
- Setting maxFeePerGas below the current base fee and wondering why nothing happens.
- Assuming type 2 everywhere — you still must decode types 0, 1 and 3 from the chain.
- Reusing a nonce across two signed transactions and getting a silent replacement.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Decode a raw transaction and print its type and fields.
- Build and sign a legacy transaction; print its RLP and hash.
- Extract the chain id from a type-0 `v` value.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build and sign a type-2 transaction and decode the typed envelope byte by byte.
- Compute the signing hash by hand and compare with `types.Signer.Hash`.
- Recover the sender from a real mainnet raw transaction.
- Show that changing the chain id changes the signature and breaks replay.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Implement a decoder that handles all four types from raw bytes and round-trips each.
- Build a type-1 transaction with an access list and measure the gas saved on `anvil`.
- Decode a type-3 blob transaction from a real block and verify a versioned hash against its commitment.

### Packages & tools

`github.com/ethereum/go-ethereum/core/types`, `github.com/ethereum/go-ethereum/crypto`, `github.com/ethereum/go-ethereum/rlp`, `github.com/ethereum/go-ethereum/crypto/kzg4844`, `math/big`

### Resources to cite

- EIP-155 (replay protection): https://eips.ethereum.org/EIPS/eip-155
- EIP-2718 (typed transactions): https://eips.ethereum.org/EIPS/eip-2718
- EIP-2930 (access lists): https://eips.ethereum.org/EIPS/eip-2930
- EIP-1559: https://eips.ethereum.org/EIPS/eip-1559
- EIP-4844: https://eips.ethereum.org/EIPS/eip-4844

---

## 20 — JSON-RPC & go-ethereum's `ethclient`

**Lesson file:** [../20-json-rpc-ethclient.md](../20-json-rpc-ethclient.md) · **Examples folder:** `../examples/20-json-rpc-ethclient/`

| | |
|---|---|
| Prerequisites | [16](../16-ethereum-architecture.md) |
| Unlocks | 21, 31, 33, 54 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 10 |

*the node API, `ethclient` from Go, queries, filters, subscriptions, batching and provider realities*

### Goals

- Call the Ethereum JSON-RPC API directly and through `ethclient`.
- Read blocks, receipts, balances, storage and code from Go.
- Subscribe to new heads and logs over WebSocket, with reconnection.
- Handle provider limits, batching and errors realistically.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. JSON-RPC 2.0 basics**

   - The envelope: jsonrpc, method, params, id — and the error object shape.
   - Transports: HTTP (request/response), WebSocket (adds subscriptions), IPC (local, fastest).
   - Hex quantity encoding: `0x1a`, no leading zeros; data is `0x`-prefixed and byte-aligned.
   - Doing one call with `net/http` by hand so the abstraction never feels magic.

**2. The methods you will actually use**

   - `eth_blockNumber`, `eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_getTransactionByHash`.
   - `eth_getTransactionReceipt`, `eth_getBalance`, `eth_getCode`, `eth_getStorageAt`, `eth_getTransactionCount`.
   - `eth_call`, `eth_estimateGas`, `eth_getLogs`, `eth_sendRawTransaction`, `eth_getProof`.
   - `eth_feeHistory` for fee estimation; `eth_chainId`, `net_version`.

**3. Block tags**

   - `latest` (can be reorged), `safe`, `finalized`, `earliest`, `pending` (node-specific, unreliable).
   - A block number or hash for historical queries — hash-based queries are reorg-proof.
   - Which tag to use for what: display vs accounting vs settlement.
   - `pending` differs wildly across clients; do not build on it.

**4. ethclient in Go**

   - `ethclient.Dial` / `DialContext`; every method takes a `context.Context`.
   - `BlockByNumber`, `HeaderByNumber` (cheaper), `TransactionReceipt`, `BalanceAt`, `CodeAt`, `StorageAt`.
   - `FilterLogs` with `ethereum.FilterQuery`; `SubscribeNewHead`, `SubscribeFilterLogs`.
   - `ethereum.NotFound` — the sentinel error you must handle everywhere.

**5. Raw and batch calls**

   - `rpc.Client` for methods `ethclient` does not wrap (`debug_*`, `trace_*`, `txpool_*`).
   - `BatchCallContext` with `[]rpc.BatchElem` — one round trip for many calls.
   - Batching 100 receipt fetches vs 100 requests: measure the difference.
   - Per-element errors: a batch can partially fail; check every element.

**6. eth_call and state overrides**

   - Read contract state without a transaction, at any block, for free.
   - State overrides: pretend an account has a balance, code or storage — powerful for simulation.
   - Historical calls need an archive node; a full node will error on old blocks.
   - Simulating a transaction before sending it (lessons 21, 40).

**7. Subscriptions**

   - WebSocket only. `SubscribeNewHead` gives headers; `SubscribeFilterLogs` gives logs.
   - The `Err()` channel: subscriptions die, and a dead subscription is silent otherwise.
   - Mandatory reconnect loop with backoff, and re-fetching the gap on reconnect.
   - Why production systems subscribe *and* poll: subscriptions drop, polling is boring and reliable.

**8. Provider realities**

   - Rate limits and compute-unit accounting; `getLogs` block-range caps (often 2k–10k).
   - Archive vs full: which methods silently fail on old blocks.
   - Inconsistent error strings across providers — never match on message text alone.
   - Free tiers, and what breaks first when you outgrow them.

**9. Error handling and retries**

   - Classify: transient (429, 5xx, timeout) vs permanent (invalid params, revert).
   - Exponential backoff with jitter, bounded by the caller's `context` deadline.
   - Idempotency: reads are safe to retry; `eth_sendRawTransaction` needs care (lesson 21).
   - Never a bare retry loop without a deadline — link forward to lesson 35.

**10. Debug and trace namespaces**

   - `debug_traceTransaction` with callTracer/prestateTracer; `trace_block` (Erigon/OpenEthereum style).
   - Cost and availability — most free tiers do not expose these.
   - Decoding a call tree in Go to find internal transfers.
   - The alternative when you cannot trace: reconstruct from logs (lesson 25).

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Ignoring `ethereum.NotFound` and treating a missing receipt as an error.
- Requesting a 100k-block `getLogs` range and getting a truncated or failed response.
- Relying on a subscription without a reconnect path — you will silently stop receiving data.
- Using `pending` for balances or nonces in production.
- Calling without a `context` deadline and hanging a worker forever.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Dial a node and print chain id, latest block and gas price.
- Fetch a block and list its transactions with values in ether.
- Read an account's balance, nonce and code size.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Do one raw JSON-RPC call with `net/http` and compare with `ethclient`.
- Batch 50 `eth_getTransactionReceipt` calls and compare latency against sequential.
- `FilterLogs` over a 10k-block range, chunked to respect a 2k-block provider cap.
- Read a contract's storage slot with `StorageAt` and decode it.
- Handle `ethereum.NotFound` correctly for a nonexistent transaction.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Subscribe to new heads, kill the connection, and show the reconnect loop backfilling the gap.
- A retry wrapper that classifies transient vs permanent errors and honours a context deadline.
- Simulate a transaction with `eth_call` plus a state override that fakes a token balance.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/rpc`, `github.com/ethereum/go-ethereum`, `context`, `net/http`, `time`

### Resources to cite

- Ethereum JSON-RPC spec: https://ethereum.org/en/developers/docs/apis/json-rpc/
- execution-apis repository: https://github.com/ethereum/execution-apis
- ethclient package: https://pkg.go.dev/github.com/ethereum/go-ethereum/ethclient
- rpc package: https://pkg.go.dev/github.com/ethereum/go-ethereum/rpc

---

## 21 — Sending Transactions from Go

**Lesson file:** [../21-sending-transactions.md](../21-sending-transactions.md) · **Examples folder:** `../examples/21-sending-transactions/`

| | |
|---|---|
| Prerequisites | [19](../19-transaction-types.md), [20](../20-json-rpc-ethclient.md) |
| Unlocks | 32, 54, 59 |
| Examples | **18** — 🟢 5 easy, 🟡 8 medium, 🔴 5 hard |
| Topics | 11 |

*nonce management, gas estimation, 1559 fees, signing, broadcast, confirmation and stuck-tx recovery*

### Goals

- Send a signed transaction from Go, end to end.
- Manage nonces correctly under concurrency.
- Set 1559 fees that get included without overpaying.
- Wait for confirmations safely and recover a stuck transaction.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The pipeline**

   - build → estimate gas → price fees → sign → broadcast → wait → confirm → survive reorgs.
   - Each stage has its own failure mode; enumerate them before writing code.
   - Where to persist state so a crash between stages is recoverable.
   - The mental model: sending is a distributed transaction with an unreliable participant.

**2. Nonce sources**

   - `NonceAt(latest)` = confirmed only. `PendingNonceAt` = includes the node's mempool view.
   - Both lie under concurrency: two goroutines get the same pending nonce.
   - Providers disagree about `pending` — a load-balanced RPC makes this worse.
   - The only correct source is a counter you own.

**3. A nonce manager**

   - Per-address mutex plus an in-memory next-nonce, seeded from `PendingNonceAt` at startup.
   - Reserve → sign → send → commit; on send failure, release the nonce back.
   - Persisting it (lesson 32) so a restart does not replay or skip.
   - Gap recovery: detect a hole and fill it with a self-send.

**4. Gas estimation**

   - `EstimateGas` executes against current state; it fails for state-dependent reverts.
   - Add headroom (10–30%) because state moves between estimate and inclusion.
   - Estimating for a contract call that requires an approval you have not made yet — the classic false failure.
   - Hard caps: never send an unbounded gasLimit from user input.

**5. Fee strategy**

   - `SuggestGasTipCap` for the tip; base fee from the latest header.
   - A common rule: maxFee = 2·baseFee + tip, giving ~6 blocks of headroom.
   - `eth_feeHistory` percentiles for a smarter estimate.
   - You are refunded the difference, so err generous on maxFee and tight on tip.

**6. Signing options**

   - Raw `types.SignTx` with an in-memory key — fine for tests, never for production.
   - `bind.TransactOpts` with a `Signer` func — what abigen expects (lesson 24).
   - `accounts/keystore` and `bind.NewKeyedTransactorWithChainID`.
   - A remote signer behind an interface — the production answer (lesson 32).

**7. Broadcast and its error taxonomy**

   - `already known` — the same transaction; usually benign, treat as success.
   - `replacement transaction underpriced` — same nonce, insufficient fee bump (needs ≥10%).
   - `nonce too low` — already mined or your counter drifted.
   - `insufficient funds for gas * price + value` — check balance against maxFee·gasLimit + value.
   - `intrinsic gas too low`, `exceeds block gas limit`, and provider-specific wrappers.

**8. Waiting for a receipt**

   - `bind.WaitMined` or a poll loop with backoff and a context deadline.
   - **Check `receipt.Status`** — a mined transaction can have reverted and still cost you gas.
   - Extract the revert reason by replaying with `eth_call` at the mined block.
   - Timeouts: what to do when nothing lands (it is still pending, not gone).

**9. Confirmation depth and reorgs**

   - One confirmation is not final. Pick a depth per chain and per value.
   - Re-check the receipt's blockHash still belongs to the canonical chain.
   - A transaction can be mined, reorged out, and re-mined in a different block with a different index.
   - Persist by transaction hash, never by (block, index).

**10. Stuck transactions**

   - Cancel: same nonce, zero value, to self, ≥10% higher fees.
   - Speed up: same nonce, same payload, higher fees.
   - Why you cannot 'delete' a transaction — you can only replace it.
   - Monitoring pending age and auto-bumping (lesson 35).

**11. Idempotency and crash safety**

   - Persist the signed raw transaction *before* broadcasting; re-broadcast is safe and cheap.
   - A state machine: created → signed → broadcast → mined → confirmed, with the nonce recorded.
   - Never re-sign on retry — sign once, store, resend the same bytes.
   - This design is what lesson 59 scales up to batched withdrawals.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Fetching the nonce per request instead of owning a counter — duplicate nonces under load.
- Treating a receipt as success without checking `Status`.
- Re-signing on retry with a fresh nonce and accidentally sending twice.
- Fee bump below 10% — the node rejects the replacement.
- Keying your database on (blockNumber, txIndex), which a reorg invalidates.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 18).

**🟢 Easy — 5 examples** *(one concept in isolation)*

- Send 0.001 ETH on `anvil` and wait for the receipt.
- Print the receipt's status, gas used and effective gas price.
- Estimate gas for a simple transfer and compare with the actual 21000.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Compute maxFee and tip from the latest base fee and `SuggestGasTipCap`.
- Fire five concurrent sends from one key with a nonce manager and show all five land in order.
- Trigger `replacement transaction underpriced` and then succeed with a 12% bump.
- Send a transaction that reverts and extract the revert reason by replaying with `eth_call`.

**🔴 Hard — 5 examples** *(real-shaped, multi-concept programs)*

- Cancel a pending transaction with a same-nonce zero-value self-send.
- A persistent send pipeline: sign once, store raw bytes, crash, restart, re-broadcast, confirm exactly once.
- Wait for N confirmations and detect a reorg that moved the transaction to a different block.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/core/types`, `sync`, `context`, `time`

### Resources to cite

- bind package: https://pkg.go.dev/github.com/ethereum/go-ethereum/accounts/abi/bind
- EIP-1559 fee mechanics: https://eips.ethereum.org/EIPS/eip-1559
- Ethereum docs — transactions: https://ethereum.org/en/developers/docs/transactions/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
