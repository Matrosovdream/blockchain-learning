# Part 11 — Smart Contracts, Deeper

Everything past 'can call a contract': writing efficient Solidity, upgrading it safely, account abstraction, a real test suite, and the token standards beyond ERC-20/721. Take after Part 5.

**Extension.** Beyond the core 01–41 spine. Lessons 45–49 · 77 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 45 | [Solidity in Depth & Gas Optimization](#45-solidity-in-depth-gas-optimization) | 22, 27 | 15 |
| 46 | [Upgradeable Contracts & Proxy Patterns](#46-upgradeable-contracts-proxy-patterns) | 18, 27 | 16 |
| 47 | [Account Abstraction: ERC-4337 & EIP-7702](#47-account-abstraction-erc-4337-eip-7702) | 23, 46 | 16 |
| 48 | [Foundry: Tests, Fuzzing & Invariants](#48-foundry-tests-fuzzing-invariants) | 22, 34 | 15 |
| 49 | [Vaults, Permits & the Wider Token Standards](#49-vaults-permits-the-wider-token-standards) | 26 | 15 |

---

## 45 — Solidity in Depth & Gas Optimization

**Lesson file:** [../45-solidity-depth-gas.md](../45-solidity-depth-gas.md) · **Examples folder:** `../examples/45-solidity-depth-gas/`

| | |
|---|---|
| Prerequisites | [22](../22-solidity-basics.md), [27](../27-contract-security.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 7 medium, 🔴 4 hard |
| Topics | 10 |

*the language past the basics, storage layout as a cost model, assembly, and measured optimization*

### Goals

- Write non-trivial Solidity and predict its gas cost before running it.
- Optimize storage layout, calldata and loops with measured results.
- Read and write minimal Yul/inline assembly where it is justified.
- Know when optimization is a false economy.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Measure first**

   - `forge test --gas-report`, `forge snapshot`, and `cast estimate` — never guess.
   - The cost model: storage ≫ calldata ≫ memory ≫ stack (lesson 18's numbers, applied).
   - Where gas actually goes in a typical contract — usually 2–3 SSTOREs.
   - Deployment cost vs runtime cost, and the optimizer-runs knob that trades them.

**2. Storage layout as a cost model**

   - Slot packing: 4×uint64 in one slot is one SSTORE instead of four.
   - Order fields by size to enable packing; the compiler will not reorder for you.
   - Reading a packed slot is one SLOAD then masking — cheap. Writing part of it is a read-modify-write.
   - `forge inspect <C> storageLayout` and verifying with `cast storage`.

**3. Storage patterns**

   - Zero → non-zero costs 20000; non-zero → non-zero costs 2900; clearing refunds (capped by EIP-3529).
   - Caching a storage variable in memory inside a loop — the single highest-yield optimization.
   - `unchecked` blocks for loop counters that provably cannot overflow.
   - Transient storage (EIP-1153, `tstore`/`tload`) for within-transaction state like reentrancy guards.

**4. Calldata and function design**

   - `calldata` over `memory` for external function array/string parameters.
   - Calldata cost: 4 gas per zero byte, 16 per non-zero — which is why addresses are expensive and why zero-padding is cheap.
   - Fewer, larger arguments beat many small ones; packing arguments into a single `bytes32`.
   - Custom errors instead of `require` strings: 4 bytes vs 32+ bytes of revert data.

**5. Loops and arrays**

   - Cache `array.length` outside the loop; use `unchecked { ++i; }`.
   - Unbounded loops are a DoS vector, not just a gas cost.
   - Mapping vs array: mapping for lookup, array only when you must enumerate.
   - The enumerable-set pattern (mapping + array + index) and its removal trick.

**6. Yul and inline assembly**

   - When it is justified: bit manipulation, memory tricks, reading proxy storage slots, low-level calls.
   - When it is not: anything the optimizer already does, or anything you cannot test thoroughly.
   - The memory model: free memory pointer at 0x40, scratch space at 0x00–0x3f, and the rules you must not break.
   - Reading a specific storage slot with `sload` — you already do the Go side of this in lesson 22.

**7. Immutable, constant and clones**

   - `constant` and `immutable` live in bytecode, not storage — no SLOAD at all.
   - Minimal proxies (EIP-1167 clones) for cheap repeated deployment.
   - CREATE2 + clones as a factory pattern.
   - Deployment size limits (EIP-170) and splitting into libraries.

**8. Modern language features**

   - Custom errors, `try/catch`, user-defined value types, `using ... for` global.
   - `abi.decode` with structs; function types; `receive`/`fallback` semantics.
   - Solidity version differences that matter: 0.8 checked arithmetic, 0.8.20+ PUSH0 (breaks some L2s!).
   - Pinning the pragma and the compiler in CI.

**9. When optimization is wrong**

   - Readability and auditability are security properties; clever code has more bugs.
   - Optimize only what a gas report says is hot, and only on user-facing hot paths.
   - The real wins are usually architectural (fewer storage writes, batching), not micro.
   - A cautionary example: an assembly 'optimization' that introduced a real vulnerability.

**10. From Go**

   - Estimate gas for each variant with `eth_estimateGas` and produce your own comparison table.
   - Automate a regression check: fail CI if a function's gas exceeds a checked-in budget.
   - Read the deployed storage layout from Go and assert it matches expectations after an upgrade (lesson 46).
   - This is how you keep optimization honest.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Optimizing before measuring — most 'optimizations' change nothing.
- Packing fields that are always written together with fields that are not, causing extra read-modify-writes.
- Using PUSH0-emitting compiler versions on chains that do not support it.
- Assembly that bypasses the free memory pointer and corrupts later allocations.
- Unbounded loops over user-controlled arrays.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Run `forge test --gas-report` on a contract and read the output.
- Estimate gas from Go for two variants of the same function.
- Read a packed storage slot with `cast storage` and decode both values.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Reorder struct fields to pack them and measure the SSTORE saving.
- Cache a storage variable in a loop and measure the difference.
- Replace `require` strings with custom errors and measure both deploy and revert cost.
- Use `unchecked` on a loop counter and quantify the saving over 100 iterations.
- Deploy an EIP-1167 minimal proxy and compare its deployment gas with a full deployment.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Write a Yul snippet reading an arbitrary storage slot, and verify against `eth_getStorageAt` from Go.
- Implement an enumerable set with O(1) removal and gas-test insert/remove/iterate.
- A CI gas-budget check in Go that fails when a function exceeds its recorded budget.

### Packages & tools

`os/exec`, `github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `encoding/json`

### Resources to cite

- Solidity docs — layout in storage: https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html
- Yul: https://docs.soliditylang.org/en/latest/yul.html
- Foundry gas reports: https://book.getfoundry.sh/forge/gas-reports
- EIP-1167 minimal proxy: https://eips.ethereum.org/EIPS/eip-1167
- EIP-1153 transient storage: https://eips.ethereum.org/EIPS/eip-1153

---

## 46 — Upgradeable Contracts & Proxy Patterns

**Lesson file:** [../46-upgradeable-contracts.md](../46-upgradeable-contracts.md) · **Examples folder:** `../examples/46-upgradeable-contracts/`

| | |
|---|---|
| Prerequisites | [18](../18-evm.md), [27](../27-contract-security.md) |
| Unlocks | 47 |
| Examples | **16** — 🟢 4 easy, 🟡 8 medium, 🔴 4 hard |
| Topics | 9 |

*DELEGATECALL proxies, storage collisions, UUPS vs Transparent vs Beacon, and monitoring upgrades from Go*

### Goals

- Explain how a proxy separates code from storage via DELEGATECALL.
- Compare Transparent, UUPS, Beacon and Diamond patterns.
- Identify storage collisions and selector clashes.
- Detect and monitor implementation changes from a Go service.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why upgrades exist**

   - Contracts are immutable; bugs are not optional. Upgradeability is the pragmatic compromise.
   - The cost: an upgrade key is an admin key, and an admin key is centralisation and a target.
   - Alternatives: migration to a new contract, modular design, or genuine immutability.
   - The question to ask every protocol you integrate: who can change this code, and how fast?

**2. The DELEGATECALL mechanism**

   - A proxy holds the storage and the balance; the implementation holds the code.
   - DELEGATECALL runs the implementation's code in the proxy's context: `msg.sender`, `address(this)` and storage are the proxy's.
   - The fallback function forwards all calldata and returns the result verbatim — usually in assembly.
   - Write the minimal proxy fallback in Yul and understand every line.

**3. Storage collisions**

   - Proxy and implementation share one storage layout. A field at slot 0 in both collides.
   - The fix: EIP-1967 unstructured storage — implementation at `keccak256("eip1967.proxy.implementation") - 1`.
   - Between implementation versions: never reorder or remove state variables; only append.
   - Storage gaps (`uint256[50] private __gap`) in upgradeable base contracts.

**4. Selector clashes**

   - If the proxy and the implementation both expose a selector, the proxy wins and the implementation is unreachable.
   - This is a real attack surface — a malicious implementation can shadow admin functions.
   - The Transparent proxy fix: route by `msg.sender` (admin gets admin functions, everyone else gets the implementation).
   - The UUPS fix: put upgrade logic in the implementation, so the proxy has almost no selectors.

**5. The patterns**

   - Transparent (EIP-1967 + ProxyAdmin): simple, but every call pays an extra SLOAD for the admin check.
   - UUPS: upgrade function lives in the implementation — cheaper calls, but you can brick it by deploying an implementation without upgrade logic.
   - Beacon: many proxies point at one beacon; upgrade all of them at once.
   - Diamond (EIP-2535): a selector→facet mapping; powerful, complex, and contentious.

**6. Initializers**

   - Constructors do not run in the proxy's context — state must be set by an `initialize` function.
   - `initializer` and `reinitializer` modifiers; `_disableInitializers()` in the implementation's constructor.
   - **The Parity incident**: an uninitialized library was self-destructed, freezing 513k ETH.
   - Front-running the initializer on a fresh deployment — deploy and initialize atomically.

**7. Upgrade safety tooling**

   - OpenZeppelin Upgrades plugins validate layout compatibility before you deploy.
   - `forge inspect storageLayout` diffing between versions.
   - Timelocks on upgrades so users can exit; multisig or governance as the upgrade authority.
   - Immutability as a feature you can announce by renouncing the admin.

**8. From the Go side**

   - Bind the implementation's ABI to the **proxy's** address (lesson 24).
   - Read the EIP-1967 implementation slot directly with `eth_getStorageAt` — no ABI needed.
   - Monitor `Upgraded(address)` events and alert on any change to a contract you depend on.
   - Pin the implementation address in config and fail closed when it changes unexpectedly.
   - Re-verify your assumptions (decimals, return values, event shapes) after every upgrade.

**9. Reading an unknown proxy**

   - Detect a proxy: read the 1967 implementation and beacon slots, check for code at the result.
   - Detect a Diamond: `facets()` from the loupe interface.
   - Resolve the real ABI from the implementation's verified source.
   - Build this as a reusable Go helper — you will need it constantly.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Adding a state variable in the middle of an upgradeable contract's layout.
- Deploying a UUPS implementation with no upgrade function and permanently bricking the proxy.
- Leaving an implementation uninitialized (Parity).
- Binding your Go client to the implementation address instead of the proxy.
- Integrating an upgradeable contract and never monitoring for upgrades.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Read the EIP-1967 implementation slot of a real proxy from Go.
- Deploy a minimal proxy + implementation on `anvil` and call through it.
- Show that `address(this)` inside the implementation is the proxy's address.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Demonstrate a storage collision between proxy and implementation, then fix it with EIP-1967 slots.
- Demonstrate a selector clash and fix it with the Transparent pattern.
- Upgrade an implementation and show state survives.
- Write the proxy fallback in Yul and verify it forwards calldata and returndata exactly.
- Show `_disableInitializers()` preventing implementation takeover.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- A Go proxy-detector: identify Transparent/UUPS/Beacon/Diamond and resolve the implementation ABI.
- Monitor `Upgraded` events across 10 contracts and alert on change, with a pinned-address fail-closed mode.
- Diff storage layouts between two implementation versions from Go and reject an unsafe upgrade.

### Packages & tools

`github.com/ethereum/go-ethereum/ethclient`, `github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/common`, `os/exec`

### Resources to cite

- EIP-1967 (proxy storage slots): https://eips.ethereum.org/EIPS/eip-1967
- EIP-1822 (UUPS): https://eips.ethereum.org/EIPS/eip-1822
- EIP-2535 (Diamonds): https://eips.ethereum.org/EIPS/eip-2535
- OpenZeppelin Upgrades: https://docs.openzeppelin.com/upgrades-plugins/
- Parity multisig incident: https://www.parity.io/blog/a-postmortem-on-the-parity-multi-sig-library-self-destruct/

---

## 47 — Account Abstraction: ERC-4337 & EIP-7702

**Lesson file:** [../47-account-abstraction.md](../47-account-abstraction.md) · **Examples folder:** `../examples/47-account-abstraction/`

| | |
|---|---|
| Prerequisites | [23](../23-abi-encoding.md), [46](../46-upgradeable-contracts.md) |
| Unlocks | — |
| Examples | **16** — 🟢 4 easy, 🟡 8 medium, 🔴 4 hard |
| Topics | 9 |

*smart accounts, UserOperations, bundlers, paymasters, and what changes for your Go backend*

### Goals

- Explain the EOA's limitations and what account abstraction replaces them with.
- Describe the full ERC-4337 flow from UserOperation to on-chain execution.
- Explain EIP-7702 and how it differs from 4337.
- Build and submit a UserOperation from Go.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. What is wrong with EOAs**

   - One key, one signature scheme, no policy, no recovery, and you must hold ETH to do anything.
   - No batching: an approve+swap is two transactions and two signatures.
   - No session keys, no spending limits, no social recovery.
   - Every UX complaint about crypto traces back to this.

**2. The two approaches**

   - ERC-4337: an entirely separate mempool and pipeline, no protocol change required.
   - EIP-7702: let an EOA temporarily *become* a smart account, at the protocol level (Pectra).
   - They are complementary, not competing — 7702 accounts can use 4337 infrastructure.
   - The history: EIP-2938, EIP-3074 and why 7702 won.

**3. The ERC-4337 pipeline**

   - User signs a **UserOperation** (not a transaction) and sends it to a bundler's alternative mempool.
   - The bundler simulates it, bundles several, and submits `handleOps` to the singleton EntryPoint.
   - EntryPoint validates each op (calling the account's `validateUserOp`), then executes each.
   - Fees are collected from the account's deposit or a paymaster.
   - Draw this pipeline once and it stops being confusing.

**4. UserOperation fields**

   - sender, nonce, initCode, callData, accountGasLimits, preVerificationGas, gasFees, paymasterAndData, signature.
   - v0.7 packed the fields differently from v0.6 — check which EntryPoint you are targeting.
   - The userOpHash: hash(packed op) ‖ entryPoint ‖ chainId — what the user actually signs.
   - Building and hashing one in Go, and verifying against a bundler's `eth_estimateUserOperationGas`.

**5. Smart accounts**

   - `validateUserOp(op, hash, missingFunds)` returns a packed validation result including a time range.
   - Counterfactual deployment: the account address is CREATE2-derived, so it exists before it is deployed.
   - `initCode` deploys it on first use, inside the same UserOperation.
   - Signature schemes become a choice: ECDSA, passkeys/WebAuthn (P-256), multisig, session keys.

**6. Bundlers and the alt mempool**

   - A separate P2P mempool for UserOperations; bundlers are the new searchers.
   - ERC-7562 validation rules: what an account may access during validation, and why (DoS protection).
   - Bundler RPC: `eth_sendUserOperation`, `eth_estimateUserOperationGas`, `eth_getUserOperationReceipt`.
   - Calling these from Go over plain JSON-RPC — there is no `ethclient` wrapper.

**7. Paymasters**

   - Sponsored transactions: someone else pays the gas. This is the killer feature.
   - Verifying paymasters (off-chain signature) vs token paymasters (pay in ERC-20).
   - The deposit/stake model in the EntryPoint and the paymaster's postOp hook.
   - Running a paymaster service in Go: policy, signing, accounting, and abuse prevention.

**8. EIP-7702**

   - A new transaction type carrying an authorization list: `(chainId, address, nonce, y_parity, r, s)`.
   - It sets the EOA's code to a delegation designator (`0xef0100 ‖ implementation`).
   - The EOA keeps its address and history but now executes contract code.
   - Risks: a signed authorization is a blank cheque; chainId 0 authorizations apply to every chain.
   - Building and signing a 7702 authorization in Go.

**9. What changes for your backend**

   - `tx.origin != msg.sender` is now normal; do not use `tx.origin` for anything.
   - The 'from' of a UserOperation is not a transaction sender — indexing by sender breaks.
   - `UserOperationEvent` is where you find success/failure, not the outer transaction receipt.
   - A single transaction can contain many users' operations — receipts are shared.
   - Signature verification must support EIP-1271 (contract signatures) — lesson 50.
   - Your indexer needs to decode `handleOps` calldata to attribute activity correctly.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Indexing 4337 activity from transaction senders — you will attribute everything to bundlers.
- Assuming a successful `handleOps` transaction means every UserOperation succeeded; check `UserOperationEvent.success`.
- Verifying a smart-account signature with `ecrecover` instead of EIP-1271.
- Signing a 7702 authorization with chainId 0 (valid on every chain, forever).
- Targeting the wrong EntryPoint version and getting confusing validation failures.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Compute a UserOperation hash in Go for a given EntryPoint and chain id.
- Call a bundler's `eth_estimateUserOperationGas` over raw JSON-RPC.
- Derive a counterfactual smart-account address from its factory and salt.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build, sign and submit a UserOperation from Go and poll for its receipt.
- Decode `handleOps` calldata and extract every UserOperation in a bundle.
- Decode `UserOperationEvent` logs and attribute success/failure per user.
- Deploy a smart account counterfactually via `initCode` on its first operation.
- Sign an EIP-7702 authorization tuple and inspect the resulting delegation.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- Run a minimal verifying paymaster service in Go: policy check, sign, and account for sponsorship spend.
- Extend an indexer to attribute 4337 activity to smart accounts rather than bundlers.
- Compare gas cost and UX of the same operation as an EOA transaction, a 4337 UserOperation and a 7702 delegation.

### Packages & tools

`github.com/ethereum/go-ethereum/rpc`, `github.com/ethereum/go-ethereum/accounts/abi`, `github.com/ethereum/go-ethereum/crypto`, `github.com/ethereum/go-ethereum/core/types`

### Resources to cite

- ERC-4337: https://eips.ethereum.org/EIPS/eip-4337
- EIP-7702: https://eips.ethereum.org/EIPS/eip-7702
- ERC-7562 (validation rules): https://eips.ethereum.org/EIPS/eip-7562
- eth-infinitism reference implementation: https://github.com/eth-infinitism/account-abstraction
- Ethereum docs — account abstraction: https://ethereum.org/en/roadmap/account-abstraction/

---

## 48 — Foundry: Tests, Fuzzing & Invariants

**Lesson file:** [../48-foundry-testing.md](../48-foundry-testing.md) · **Examples folder:** `../examples/48-foundry-testing/`

| | |
|---|---|
| Prerequisites | [22](../22-solidity-basics.md), [34](../34-testing-blockchain-go.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*the contract-side test suite that complements your Go tests — cheatcodes, fuzzing, invariants, forking*

### Goals

- Write Solidity unit tests with cheatcodes.
- Fuzz functions and find edge cases automatically.
- Write invariant tests that hold across arbitrary call sequences.
- Integrate Foundry into the same CI as your Go tests.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why a second test suite**

   - Go tests verify *your service*; Foundry tests verify *the contract*.
   - Foundry runs the EVM directly — no RPC, no JSON, microsecond tests.
   - Fuzzing and invariant testing have no real Go equivalent for contract state.
   - You will read Foundry tests constantly even if you never write contracts professionally.

**2. Test basics**

   - `forge-std/Test.sol`, `setUp()`, `test_` prefix, `assertEq` family.
   - Naming conventions: `test_`, `testFuzz_`, `test_RevertWhen_`, `invariant_`.
   - `forge test -vvv` verbosity levels and reading a trace.
   - `--match-test`, `--match-contract`, and `--gas-report`.

**3. Cheatcodes**

   - `vm.prank` / `vm.startPrank` — change `msg.sender`.
   - `vm.deal`, `vm.warp`, `vm.roll` — set balance, timestamp, block number.
   - `vm.expectRevert`, `vm.expectEmit`, `vm.expectCall`.
   - `vm.store` / `vm.load` — write and read arbitrary storage slots (the Solidity twin of lesson 22's Go work).
   - `vm.sign` and `vm.addr` for signature tests; `vm.mockCall` for external dependencies.

**4. Fuzz testing**

   - Give a test function parameters and Foundry generates inputs.
   - `vm.assume` and `bound()` to constrain inputs without wasting runs.
   - Reading a counterexample and turning it into a permanent regression test.
   - `forge test --fuzz-runs 10000` and the seed for reproducibility.

**5. Invariant testing**

   - Properties that must hold after *any* sequence of calls: total supply equals sum of balances, solvency, monotonic accounting.
   - Handler contracts to constrain the call space to realistic operations.
   - `targetContract`, `targetSelector`, and ghost variables for tracking expectations.
   - This is where real bugs are found; it is worth the setup cost.

**6. Fork testing**

   - `vm.createSelectFork(rpcUrl, blockNumber)` — real mainnet state inside a Solidity test.
   - Pinning the block for reproducibility; `foundry.toml` RPC endpoints and env vars.
   - Testing an integration against the real token/protocol rather than a mock.
   - The mirror of lesson 34's Go-side anvil forking.

**7. Differential and reference testing**

   - `ffi` to call an external binary — including a Go program — from a Solidity test.
   - Differential testing: compare your Solidity implementation against a Go reference implementation.
   - This is a genuinely good use of your Go skills: write the oracle in Go, fuzz the contract against it.
   - Security implications of `ffi` and why it is off by default.

**8. Coverage and CI**

   - `forge coverage`, its limitations, and why 100% is not the goal.
   - One CI workflow running `forge test`, `forge fmt --check`, `slither`, then `go test ./...`.
   - Caching both the Foundry and Go toolchains.
   - Gas snapshots (`forge snapshot --check`) as a regression gate.

**9. Scripts and deployment**

   - `forge script` with `vm.startBroadcast` for reproducible deployments.
   - Deployment artifacts and how your Go code consumes the resulting addresses and ABIs.
   - Verifying on an explorer from the script.
   - Keeping deployment addresses in a checked-in JSON your Go service reads.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Fuzzing without `bound()`, so 99% of runs are rejected by `vm.assume`.
- Invariant tests with no handler, so the fuzzer only ever hits reverts.
- Unpinned fork blocks making CI nondeterministic.
- Treating coverage percentage as a quality metric.
- Leaving `ffi = true` on in a repo that runs untrusted PRs.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Write a `setUp` and three unit tests for `Counter`.
- Use `vm.prank` to test an `onlyOwner` function.
- Use `vm.expectRevert` on a failing require.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Fuzz a deposit/withdraw pair and find the edge case with `bound()`.
- Use `vm.expectEmit` to assert an event's topics and data.
- Use `vm.store` to force a contract into a specific state and test the recovery path.
- Fork mainnet at a pinned block and test an ERC-20 integration.
- Add a gas snapshot and make CI fail on regression.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Write an invariant test with a handler asserting total supply equals the sum of balances.
- Differential-test a Solidity math function against a Go reference implementation via `ffi`.
- One CI workflow running `forge test`, `slither` and `go test ./...` with caching.

### Packages & tools

`forge-std`, `os/exec`, `testing`

### Resources to cite

- Foundry Book: https://book.getfoundry.sh/
- Foundry cheatcodes reference: https://book.getfoundry.sh/cheatcodes/
- Invariant testing: https://book.getfoundry.sh/forge/invariant-testing
- forge-std: https://github.com/foundry-rs/forge-std

---

## 49 — Vaults, Permits & the Wider Token Standards

**Lesson file:** [../49-token-standards-wider.md](../49-token-standards-wider.md) · **Examples folder:** `../examples/49-token-standards-wider/`

| | |
|---|---|
| Prerequisites | [26](../26-erc-standards.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*ERC-4626, EIP-2612, Permit2, ERC-2981, ERC-6551 and friends — the standards that show up in real integrations*

### Goals

- Integrate an ERC-4626 vault correctly, including share-price accounting.
- Use EIP-2612 permit and Permit2 to remove approval transactions.
- Handle royalties, token-bound accounts and the other standards you will meet.
- Know which standards are widely adopted and which are aspirational.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. ERC-4626 tokenized vaults**

   - A standard interface for yield-bearing vaults: assets in, shares out.
   - `deposit`/`mint`, `withdraw`/`redeem`, and the four `preview*` and `max*` functions.
   - `convertToShares`/`convertToAssets` are *estimates* — `preview*` are the accurate ones.
   - Share price = totalAssets/totalSupply, and why it only goes up (in a well-behaved vault).
   - The **inflation/donation attack** on an empty vault, and the virtual-shares mitigation.
   - Accounting implication: your ledger (lesson 60) must track shares and the asset value separately.

**2. EIP-2612 permit**

   - An EIP-712 signature authorising an allowance — no approval transaction, no gas from the user.
   - Fields: owner, spender, value, nonce, deadline. The nonce is per-owner and on-chain.
   - Building and signing a permit in Go, then submitting `permit` + action in one transaction.
   - Not universally implemented; DAI has its own non-standard variant.

**3. Permit2**

   - Uniswap's universal approval layer: approve Permit2 once, then sign per-transfer permits for any token.
   - Works for tokens that never implemented 2612 — which is most of them.
   - `PermitTransferFrom` vs `AllowanceTransfer` modes and when each is used.
   - Signing a Permit2 message in Go, including the witness variant.
   - The risk: one contract now holds approval for everything you own.

**4. ERC-165 revisited and interface discovery**

   - Building an interface-id computer in Go and a capability probe for unknown contracts.
   - Combining ERC-165 with call-probing heuristics for contracts that do not implement it.
   - Caching capability detection per address.
   - Feeding this into an indexer that must classify unknown tokens.

**5. ERC-2981 royalties**

   - `royaltyInfo(tokenId, salePrice)` → (receiver, amount). Advisory, not enforced.
   - Why marketplaces stopped honouring it and what that means for your accounting.
   - Reading it from Go and handling contracts that do not implement it.

**6. ERC-6551 token-bound accounts**

   - Every NFT gets a smart-contract account, deterministically derived from (chainId, contract, tokenId).
   - The registry and the account implementation; CREATE2 again.
   - Computing a token-bound account address in Go without any RPC call.
   - Why an indexer must treat these as accounts, not just contracts.

**7. ERC-1271 contract signatures**

   - `isValidSignature(hash, signature)` → magic value. How a contract 'signs'.
   - Essential for smart accounts (lesson 47) and Safe multisigs (lesson 43).
   - Verification in Go: try `ecrecover` first, fall back to an `eth_call` of `isValidSignature`.
   - Full application treatment in lesson 50.

**8. Wrapped and rebasing assets**

   - WETH: why it exists, and that it is not a standard ERC-20 mint/burn.
   - Rebasing tokens (stETH) vs wrapped reward-bearing (wstETH) — the accounting difference is enormous.
   - Fee-on-transfer revisited: always measure balance deltas.
   - A checklist your integration must run against any new token.

**9. Standards triage**

   - Widely adopted: 20, 721, 1155, 165, 2612, 1271, 4626, Permit2.
   - Partially adopted: 2981, 4337, 6551, 3156 (flash loans).
   - Aspirational or dead: many. Check adoption before building on one.
   - How to check: block explorers, OpenZeppelin implementations, and actual on-chain usage.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Using `convertToShares` where `previewDeposit` is required — silently wrong by the fee amount.
- Depositing into an empty ERC-4626 vault without checking for the inflation-attack mitigation.
- Assuming every ERC-20 supports `permit`.
- Treating a rebasing token's balance as constant between reads.
- Verifying a smart-account signature with `ecrecover` and rejecting a valid EIP-1271 signature.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Read an ERC-4626 vault's `totalAssets`, `totalSupply` and share price.
- Compute an ERC-6551 token-bound account address in Go.
- Read `royaltyInfo` from an ERC-2981 collection.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Compare `previewDeposit` against `convertToShares` on a vault with fees and explain the difference.
- Sign an EIP-2612 permit in Go and submit `permit` + `transferFrom` in one transaction.
- Sign a Permit2 `PermitTransferFrom` message and execute the transfer.
- Detect and classify an unknown token address (20/721/1155/4626) from Go.
- Track a rebasing token's balance across a rebase and show the event-derived balance drifting.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Demonstrate the ERC-4626 inflation attack on a naive vault and show the virtual-shares fix.
- A universal signature verifier in Go: ECDSA, EIP-1271 and EIP-6492 (pre-deploy) in one function.
- A token-capability probe that produces an integration checklist for any address.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/signer/core/apitypes`, `github.com/ethereum/go-ethereum/crypto`, `math/big`

### Resources to cite

- ERC-4626: https://eips.ethereum.org/EIPS/eip-4626
- EIP-2612 (permit): https://eips.ethereum.org/EIPS/eip-2612
- Permit2: https://github.com/Uniswap/permit2
- ERC-1271: https://eips.ethereum.org/EIPS/eip-1271
- ERC-6551: https://eips.ethereum.org/EIPS/eip-6551
- OpenZeppelin ERC-4626 security: https://docs.openzeppelin.com/contracts/5.x/erc4626

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
