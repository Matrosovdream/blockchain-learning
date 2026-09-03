# Part 12 — Identity, Wallets & dApp Backends

The half of web3 that lives off-chain: signature-based login, wallet sessions, name resolution and oracles — the pieces a Go backend actually has to implement. Take after [23](../23-abi-encoding.md).

**Extension.** Beyond the core 01–41 spine. Lessons 50–53 · 60 examples planned.

> This is an **authoring spec**, not the lesson. Conventions and the writing rules live in [../PLAN.md](../PLAN.md). The reader-facing index is [../README.md](../README.md).

| # | Lesson | Prereqs | Examples |
|---|---|---|---|
| 50 | [Off-Chain Signatures: EIP-191, EIP-712, EIP-1271 & SIWE](#50-off-chain-signatures-eip-191-eip-712-eip-1271-siwe) | 23, 06 | 16 |
| 51 | [Wallet Integration & dApp Backends](#51-wallet-integration-dapp-backends) | 50 | 15 |
| 52 | [ENS & Name Resolution](#52-ens-name-resolution) | 26 | 14 |
| 53 | [Oracles, Price Feeds & On-Chain Randomness](#53-oracles-price-feeds-on-chain-randomness) | 40 | 15 |

---

## 50 — Off-Chain Signatures: EIP-191, EIP-712, EIP-1271 & SIWE

**Lesson file:** [../50-offchain-signatures-siwe.md](../50-offchain-signatures-siwe.md) · **Examples folder:** `../examples/50-offchain-signatures-siwe/`

| | |
|---|---|
| Prerequisites | [23](../23-abi-encoding.md), [06](../06-keys-signatures.md) |
| Unlocks | 51 |
| Examples | **16** — 🟢 4 easy, 🟡 8 medium, 🔴 4 hard |
| Topics | 9 |

*signature-based login and authorization — the backend half of every dApp*

### Goals

- Verify a personal_sign signature in Go, correctly.
- Implement EIP-712 typed-data signing and verification end to end.
- Verify contract signatures with EIP-1271 (and EIP-6492 for undeployed accounts).
- Build a Sign-In With Ethereum flow with proper nonce and replay protection.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why sign off-chain**

   - Free, instant, and private — no transaction, no gas, no public record.
   - Three uses: authentication (login), authorization (permit, orders), and attestation.
   - The security model: a signature is a bearer credential. Treat it like a password.
   - What can go wrong: replay across time, chains, contracts and users.

**2. EIP-191 personal_sign**

   - `keccak256("\x19Ethereum Signed Message:\n" + len(msg) + msg)`.
   - The 0x19 prefix guarantees the payload can never be a valid RLP transaction — that is its whole job.
   - `accounts.TextHash` in go-ethereum does this for you; implement it by hand once.
   - The v-value trap again: wallets return 27/28, `crypto.Ecrecover` wants 0/1. Normalise it.

**3. EIP-712 typed data**

   - The problem: users cannot read a hex blob, and blind signing is how people get drained.
   - domainSeparator = hashStruct(EIP712Domain{name, version, chainId, verifyingContract, salt?}).
   - typeHash = keccak of the canonical type string, including nested struct definitions in order.
   - digest = keccak256(0x19 0x01 ‖ domainSeparator ‖ hashStruct(message)).
   - Implement all of it in Go with `signer/core/apitypes`, and verify against `cast wallet sign-typed-data`.
   - Nested structs and arrays: the encoding rules that trip everyone up.

**4. Replay protection**

   - chainId in the domain stops cross-chain replay.
   - verifyingContract stops cross-contract replay.
   - A nonce stops the same signature being used twice.
   - A deadline/expiry stops a signature being held and used months later.
   - Omit any one of these and you have a real vulnerability — with real precedents.

**5. EIP-1271 contract signatures**

   - Smart accounts (lesson 47) and multisigs (lesson 43) cannot produce an ECDSA signature.
   - `isValidSignature(bytes32 hash, bytes signature)` returns `0x1626ba7e` when valid.
   - Verification in Go: if the address has code, `eth_call` it; otherwise `ecrecover`.
   - The stateful catch: a contract signature's validity can change — re-verify at use time.
   - EIP-6492 wraps signatures from counterfactual (not-yet-deployed) accounts.

**6. Sign-In With Ethereum (EIP-4361)**

   - A structured, human-readable message: domain, address, statement, URI, version, chainId, nonce, issuedAt, expirationTime.
   - The flow: server issues a nonce → client signs → server verifies and issues a session.
   - **The nonce must be server-generated, single-use and short-lived** — this is the entire security.
   - Bind the session to the address; the signature is not the session.
   - Parsing and validating a SIWE message in Go, field by field, with strict checks.

**7. Session management**

   - After verification you are back in normal web-auth territory: issue a JWT or a server-side session.
   - Cookie hardening: HttpOnly, Secure, SameSite; CSRF protection for state-changing routes.
   - Session expiry vs signature expiry — two different clocks.
   - Logout, revocation, and what happens when a user changes wallets.

**8. Building the Go handlers**

   - `POST /auth/nonce` → generate, store with TTL, return.
   - `POST /auth/verify` → parse SIWE, check domain/chainId/nonce/expiry, verify signature (ECDSA or 1271), issue session.
   - Constant-time nonce comparison; delete the nonce on use.
   - Rate limiting both endpoints; they are unauthenticated by definition.

**9. Signed off-chain orders**

   - The pattern behind Seaport, 0x, and every gasless order book.
   - Sign an order struct with EIP-712; the taker submits it on-chain; the contract verifies and fills.
   - Cancellation: on-chain nonce bumps or an off-chain cancellation registry.
   - Your Go service as the order book: store, index, match, and never accept an expired or filled order.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Accepting a client-supplied nonce — a complete authentication bypass.
- Not checking the SIWE `domain` and `chainId`, allowing a signature from another site to log in.
- Forgetting the v-value 27/28 normalisation and rejecting every valid signature.
- Verifying a smart-account login with `ecrecover` only, locking out every Safe and 4337 user.
- Treating the signature itself as a session token instead of exchanging it for one.
- Caching an EIP-1271 verification result — contract signature validity is not permanent.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 16).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Verify a `personal_sign` signature in Go and recover the address.
- Implement `TextHash` by hand and compare with `accounts.TextHash`.
- Normalise a 27/28 v value to 0/1.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Build an EIP-712 domain separator and type hash in Go.
- Sign typed data in Go and verify it matches `cast wallet sign-typed-data`.
- Show a signature replaying on a second chain when chainId is omitted from the domain.
- Parse and strictly validate a SIWE message, rejecting each malformed field with a distinct error.
- Verify an EIP-1271 signature from a Safe via `eth_call`.

**🔴 Hard — 4 examples** *(real-shaped, multi-concept programs)*

- A complete SIWE HTTP service: nonce issue, verify, session cookie, logout, with tests.
- A universal `VerifySignature(ctx, addr, hash, sig)` handling ECDSA, EIP-1271 and EIP-6492.
- An off-chain order book: sign EIP-712 orders in Go, store, match, and fill one on `anvil`.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts`, `github.com/ethereum/go-ethereum/crypto`, `github.com/ethereum/go-ethereum/signer/core/apitypes`, `net/http`, `crypto/rand`, `crypto/subtle`

### Resources to cite

- EIP-191: https://eips.ethereum.org/EIPS/eip-191
- EIP-712: https://eips.ethereum.org/EIPS/eip-712
- EIP-1271: https://eips.ethereum.org/EIPS/eip-1271
- EIP-4361 (SIWE): https://eips.ethereum.org/EIPS/eip-4361
- EIP-6492: https://eips.ethereum.org/EIPS/eip-6492

---

## 51 — Wallet Integration & dApp Backends

**Lesson file:** [../51-wallet-integration-backends.md](../51-wallet-integration-backends.md) · **Examples folder:** `../examples/51-wallet-integration-backends/`

| | |
|---|---|
| Prerequisites | [50](../50-offchain-signatures-siwe.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 10 |

*the wallet RPC surface, WalletConnect, webhooks, and the Go service behind a dApp*

### Goals

- Describe the wallet RPC methods a frontend calls and what your backend must supply.
- Explain WalletConnect's session model at a working level.
- Build the backend endpoints a dApp actually needs.
- Deliver webhooks reliably with signatures, retries and idempotency.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The division of labour**

   - The wallet holds keys and signs. The frontend builds intents. **Your Go backend supplies data and does the durable work.**
   - What must never be in the backend: user private keys.
   - What must never be in the frontend: trust decisions, pricing authority, or rate limits.
   - Draw the boundary once; most dApp security bugs are boundary confusion.

**2. The wallet RPC surface (EIP-1193)**

   - `eth_requestAccounts`, `eth_accounts`, `eth_chainId`, `personal_sign`, `eth_signTypedData_v4`, `eth_sendTransaction`.
   - `wallet_switchEthereumChain`, `wallet_addEthereumChain`, `wallet_watchAsset`.
   - Events: `accountsChanged`, `chainChanged`, `disconnect`.
   - EIP-6963 multi-injected provider discovery — why `window.ethereum` is no longer enough.
   - You are not writing the frontend, but you must know exactly what it can ask for.

**3. WalletConnect**

   - A relay-based session between a dApp and a mobile wallet, established by QR or deep link.
   - Sessions, namespaces (chains, methods, events) and expiry.
   - Encrypted end-to-end; the relay sees ciphertext only.
   - Backend relevance: session persistence, re-establishment, and mapping sessions to your users.

**4. What the backend must provide**

   - Read APIs over your indexed data (lesson 31) — never make the frontend call RPC directly for lists.
   - Transaction *preparation*: build calldata, estimate gas, compute slippage bounds, return an unsigned transaction.
   - Quote endpoints with a short TTL and a signed quote id so the user cannot replay a stale price.
   - Authentication via SIWE (lesson 50) and per-user rate limits.

**5. Preparing transactions server-side**

   - Why: the backend has the ABIs, the pricing logic and the safety checks; the frontend should not.
   - Return `{to, data, value, gasLimit, maxFeePerGas, maxPriorityFeePerGas, chainId}` and let the wallet sign.
   - Simulate before returning (lesson 40) so you never hand the user a transaction that will revert.
   - Never return a transaction whose destination is not on your allowlist.

**6. Tracking user transactions**

   - The frontend reports a transaction hash; the backend must not trust it blindly.
   - Verify: correct chain, correct `to`, correct sender, correct calldata — then track it.
   - State machine: submitted → pending → mined → confirmed → (reorged).
   - Surfacing status over SSE or WebSocket to the frontend.

**7. Webhooks and notifications**

   - Delivering on-chain events to your customers: at-least-once, ordered per subject, with retries.
   - HMAC-SHA256 signatures over the raw body plus a timestamp; constant-time comparison on the receiver.
   - Replay protection with a timestamp window and a delivery id.
   - Exponential backoff, a dead-letter queue, and a manual redelivery endpoint.
   - Idempotency keys so a customer's retry-safe handler actually is.

**8. Realtime to the browser**

   - Server-Sent Events for one-way updates (simplest, works through proxies).
   - WebSockets when you need bidirectional.
   - Per-user channels, authorization on connect, and backpressure on slow clients.
   - Never proxy raw node subscriptions to browsers — you will get rate-limited into oblivion.

**9. Multi-chain and multi-wallet**

   - Chain configuration as data, not code: id, name, RPC, explorer, confirmations, native token, contracts.
   - Handling `chainChanged` mid-flow, and refusing to act on the wrong chain.
   - Address normalisation: always store checksummed, compare lowercased.
   - Smart accounts change assumptions — see lesson 47.

**10. Security for dApp backends**

   - CORS, CSRF, and why a SIWE session cookie needs both.
   - Never accept a price, amount or address from the client without server-side validation.
   - Rate limit by session *and* by address *and* by IP.
   - Log enough to reconstruct any user's flow, without logging anything sensitive.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Trusting a transaction hash reported by the client without verifying its contents.
- Returning an unsigned transaction whose parameters the client can edit before signing.
- Webhooks without signatures — anyone can forge your events to a customer.
- Storing addresses in mixed case and failing lookups.
- Proxying node WebSocket subscriptions straight to browsers.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- An HTTP endpoint returning an unsigned transaction for an ERC-20 transfer.
- Sign a webhook payload with HMAC-SHA256 and verify it in constant time.
- Normalise and checksum an address for storage and lookup.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Verify a client-reported transaction hash matches the expected `to`, sender and calldata.
- A transaction status endpoint driven by a state machine, with confirmations.
- A webhook deliverer with exponential backoff, a dead-letter queue and idempotency keys.
- Stream transaction status updates to a browser over SSE.
- A chain-config registry driving confirmations and contract addresses per chain.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A complete dApp backend: SIWE auth, indexed read API, transaction preparation with simulation, SSE status.
- A quote endpoint with a signed, expiring quote id that the fill path verifies.
- Load-test the webhook deliverer with a failing consumer and prove no loss and no duplication.

### Packages & tools

`net/http`, `crypto/hmac`, `crypto/subtle`, `encoding/json`, `context`, `golang.org/x/time/rate`, `github.com/ethereum/go-ethereum/ethclient`

### Resources to cite

- EIP-1193 (provider API): https://eips.ethereum.org/EIPS/eip-1193
- EIP-6963 (provider discovery): https://eips.ethereum.org/EIPS/eip-6963
- WalletConnect docs: https://docs.reown.com/
- MDN — Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- OWASP webhook security: https://cheatsheetseries.owasp.org/

---

## 52 — ENS & Name Resolution

**Lesson file:** [../52-ens-name-resolution.md](../52-ens-name-resolution.md) · **Examples folder:** `../examples/52-ens-name-resolution/`

| | |
|---|---|
| Prerequisites | [26](../26-erc-standards.md) |
| Unlocks | — |
| Examples | **14** — 🟢 4 easy, 🟡 7 medium, 🔴 3 hard |
| Topics | 9 |

*forward and reverse resolution, namehash, wildcard/CCIP-Read, and doing it correctly from Go*

### Goals

- Implement namehash and resolve an ENS name to an address in Go.
- Do reverse resolution correctly, including the forward-check.
- Handle avatars, text records and content hashes.
- Understand CCIP-Read and offchain/L2 names.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. Why names**

   - A 42-character hex address is a UX and a safety problem.
   - ENS is the dominant naming system on Ethereum; the mechanism generalises.
   - Names are NFTs (ERC-721) — ownership, expiry and renewal are all on-chain.
   - Where names appear in your service: display, input, and never as a primary key.

**2. Namehash and normalization**

   - namehash is recursive: `namehash("") = 0x00…`, `namehash(label.rest) = keccak(namehash(rest) ‖ keccak(label))`.
   - Implement it in Go and verify against known vectors.
   - **ENSIP-15 normalization** is mandatory before hashing: Unicode normalisation, confusable filtering, emoji rules.
   - Skipping normalization is a homograph-attack vector — `аpple.eth` with a Cyrillic а is a different name.

**3. Forward resolution**

   - Registry (`0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`) → `resolver(node)` → `addr(node)`.
   - Multicoin addresses (ENSIP-9): `addr(node, coinType)` for Bitcoin, Solana and others.
   - Handle the zero resolver and the zero address as distinct 'not set' cases.
   - Caching with a TTL, and why you must not cache forever.

**4. Reverse resolution**

   - `<address>.addr.reverse` → `name(node)` gives a claimed name.
   - **A reverse record is unverified.** You MUST resolve the returned name forward and check it maps back.
   - Skipping the forward-check lets anyone claim to be `vitalik.eth` in your UI.
   - Implement both directions with the check, as one function.

**5. Text records and avatars**

   - `text(node, key)` for `avatar`, `url`, `com.twitter`, `description`, `email`.
   - Avatar URI schemes: `https://`, `ipfs://`, and `eip155:1/erc721:0x…/id` (NFT avatars).
   - Resolving an NFT avatar: fetch `tokenURI`, verify ownership, then fetch the image.
   - All of this is user-supplied data — sanitise and bound it.

**6. Content hashes**

   - `contenthash(node)` encodes an IPFS/Arweave/Swarm pointer for decentralised websites.
   - Decoding the multicodec-prefixed bytes in Go.
   - The link to lesson 55's storage work.

**7. Wildcard resolution and CCIP-Read**

   - ENSIP-10 wildcard resolution: a resolver can answer for every subdomain.
   - EIP-3668 CCIP-Read: the resolver reverts with `OffchainLookup`, telling the client to fetch from a gateway and re-submit with proof.
   - This is how offchain and L2 names (like Base and Linea names) work.
   - Implementing the CCIP-Read client loop in Go — most Go ENS libraries do not do this for you.

**8. Expiry and ownership**

   - Names expire; a resolved address can become stale or hostile after a transfer.
   - Never key a database row on an ENS name — key on the address, display the name.
   - Re-resolve at display time, or cache with a short TTL and re-verify before anything financial.
   - Handling grace periods and premium auctions in your display logic.

**9. Other naming systems**

   - Unstoppable Domains, Space ID, Bonfida (Solana) — the same shape, different registries.
   - L2 name services and the CCIP-Read pattern that unifies them.
   - A resolver abstraction in Go so you can support several.
   - The rule that holds everywhere: verify reverse resolution forward.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Trusting a reverse record without the forward-check — a trivial impersonation vector.
- Hashing a name without ENSIP-15 normalization, enabling homograph attacks.
- Caching a resolution forever, then sending funds to a name that changed hands.
- Storing an ENS name as the identity in your database instead of the address.
- Ignoring CCIP-Read and silently failing to resolve every L2 and offchain name.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 14).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Implement namehash in Go and check it against known vectors.
- Resolve `vitalik.eth` to an address via the registry and resolver.
- Read a text record such as `com.twitter`.

**🟡 Medium — 7 examples** *(concepts combined, and the traps)*

- Do reverse resolution and then the mandatory forward-check; show an unverified claim being rejected.
- Resolve a multicoin (Bitcoin) address with `addr(node, coinType)`.
- Decode a `contenthash` record into an IPFS CID.
- Handle the zero-resolver and unset-address cases distinctly.
- Normalize a name with confusable characters and show it differs from the ASCII lookalike.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- Implement the EIP-3668 CCIP-Read loop in Go and resolve an offchain/L2 name.
- Resolve an NFT avatar end to end: text record → tokenURI → ownership verification → image URL.
- A resolver abstraction supporting ENS plus one other naming system behind one interface.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi/bind`, `github.com/ethereum/go-ethereum/crypto`, `golang.org/x/text/unicode/norm`, `net/http`

### Resources to cite

- ENS docs: https://docs.ens.domains/
- ENSIP-1 (namehash): https://docs.ens.domains/ensip/1
- ENSIP-15 (normalization): https://docs.ens.domains/ensip/15
- ENSIP-10 (wildcard resolution): https://docs.ens.domains/ensip/10
- EIP-3668 (CCIP-Read): https://eips.ethereum.org/EIPS/eip-3668

---

## 53 — Oracles, Price Feeds & On-Chain Randomness

**Lesson file:** [../53-oracles-randomness.md](../53-oracles-randomness.md) · **Examples folder:** `../examples/53-oracles-randomness/`

| | |
|---|---|
| Prerequisites | [40](../40-defi-primitives-mev.md) |
| Unlocks | — |
| Examples | **15** — 🟢 4 easy, 🟡 8 medium, 🔴 3 hard |
| Topics | 9 |

*getting off-chain truth on-chain safely — Chainlink, TWAPs, VRF, RANDAO, and running your own*

### Goals

- Read a price feed correctly, with every safety check.
- Compare push and pull oracle designs and their failure modes.
- Explain VRF and RANDAO, and why naive on-chain randomness is exploitable.
- Build a minimal oracle service in Go.

### Topics

One `###` sub-section in `## Concepts` per numbered topic, in this order. The sub-points are what that section must actually cover.

**1. The oracle problem**

   - A blockchain cannot see outside itself. Every external fact arrives via a trusted reporter.
   - This is where most 'DeFi hacks' actually happen — the contract worked, the input lied.
   - Trust models: single reporter, committee, staked network, optimistic with challenge.
   - The question to ask: who profits if this number is wrong for one block?

**2. Chainlink price feeds**

   - Push model: a decentralised network writes new answers on deviation or heartbeat.
   - `latestRoundData()` → (roundId, answer, startedAt, updatedAt, answeredInRound).
   - **Mandatory checks**: `answer > 0`, `updatedAt` within the feed's heartbeat, `answeredInRound >= roundId`.
   - Decimals are per-feed (usually 8) — read `decimals()`, never hardcode.
   - Deviation thresholds mean the price is *stale by design* between updates.

**3. Push vs pull oracles**

   - Push (Chainlink classic): the oracle pays gas, data is always on-chain, updates are periodic.
   - Pull (Pyth, Chainlink Data Streams): the *user* submits a signed price update with their transaction.
   - Pull is fresher and cheaper for the protocol; it shifts work to the client — which is your Go service.
   - Fetching and submitting a signed price update from Go.

**4. AMM TWAPs**

   - Time-weighted average price from cumulative price accumulators.
   - Manipulation cost scales with the window length and pool depth — compute it.
   - Uniswap v3 observation arrays and `observe()`; cardinality and why it must be increased first.
   - When a TWAP is appropriate (deep pools, long windows) and when it is a liability (thin pools, L2s with fast blocks).

**5. Oracle failure modes**

   - Staleness: the feed stopped updating and the contract kept trusting it.
   - Depeg: the asset moved but the feed's deviation threshold had not triggered.
   - L2 sequencer downtime: prices freeze while the market moves — hence sequencer-uptime feeds.
   - Circuit breakers and min/max answer bounds; the Venus/LUNA incident as the case study.
   - Multi-oracle designs: median of three, and disagree-and-halt.

**6. On-chain randomness**

   - `block.timestamp`, `blockhash`, `prevrandao` are all proposer-influenceable.
   - The concrete attack: a proposer or user reverts the transaction when the outcome is unfavourable.
   - RANDAO: how Ethereum's beacon randomness is produced, and its last-revealer bias.
   - Commit–reveal (lesson 04) and its own last-mover problem.

**7. Verifiable random functions**

   - A VRF produces a random output plus a proof that it was computed correctly from a seed and a key.
   - Chainlink VRF v2.5: subscription, request, callback; the confirmation delay and why it exists.
   - The request/fulfil pattern in your Go service: track requests, handle callbacks, time out.
   - Cost, latency, and when off-chain randomness with an on-chain commitment is enough.

**8. Building your own oracle service in Go**

   - Fetch from N sources, aggregate (median, not mean), sanity-bound, sign, submit.
   - Deviation and heartbeat triggers to control gas spend.
   - The signing boundary (lesson 32) and per-update value limits.
   - Monitoring: last-update age, source disagreement, submission failures — with alerts.
   - This is a genuinely common Go job in this industry.

**9. Other oracle types**

   - Proof of reserve, sports/event outcomes, weather, and identity attestations.
   - Optimistic oracles (UMA): assert, challenge window, dispute resolution.
   - Cross-chain state as an oracle problem (lesson 67).
   - The Schelling-point design and its game theory, in one paragraph.

### Pitfalls to cover

These belong in `## Best Practices & Pitfalls`, each with a one-line *why*.

- Reading `latestRoundData` without a staleness check — the single most common oracle bug.
- Hardcoding 18 decimals for a feed that uses 8.
- Using a spot AMM price as an oracle (flash-loan manipulable within one transaction).
- Ignoring the L2 sequencer-uptime feed and liquidating on frozen prices.
- Using `blockhash` for anything valuable.
- Building a mean-based aggregator that one outlier source can drag.

### Example seeds

Seeds, not the full list — expand each tier to its target count, keeping numbering continuous across the three files (1 → 15).

**🟢 Easy — 4 examples** *(one concept in isolation)*

- Read a Chainlink feed and print the price with correct decimals.
- Add a staleness check and reject an answer older than the heartbeat.
- Read `decimals()` and `description()` from a feed.

**🟡 Medium — 8 examples** *(concepts combined, and the traps)*

- Implement all four Chainlink safety checks and test each rejection path.
- Read an L2 sequencer-uptime feed and gate a price read on it.
- Compute a Uniswap v3 TWAP from `observe()` and compare with spot.
- Estimate the capital cost of moving a pool's TWAP by 10% over a 30-minute window.
- Aggregate three price sources with a median and bound the result.

**🔴 Hard — 3 examples** *(real-shaped, multi-concept programs)*

- A minimal oracle service in Go: fetch, aggregate, deviation/heartbeat trigger, sign, submit, monitor.
- Request Chainlink VRF randomness from Go and handle the fulfilment callback with a timeout.
- Demonstrate the revert-on-unfavourable-outcome attack against naive `blockhash` randomness, then fix it with VRF.

### Packages & tools

`github.com/ethereum/go-ethereum/accounts/abi/bind`, `math/big`, `context`, `time`, `net/http`

### Resources to cite

- Chainlink Data Feeds: https://docs.chain.link/data-feeds
- Chainlink feed API reference: https://docs.chain.link/data-feeds/api-reference
- Chainlink VRF: https://docs.chain.link/vrf
- Uniswap v3 oracle: https://docs.uniswap.org/concepts/protocol/oracle
- Ethereum RANDAO: https://eth2book.info/latest/part2/building_blocks/randomness/

---

*Part index: [../PLAN.md](../PLAN.md) · Reader index: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md)*
