# Projects

Bigger builds that use several lessons at once. Each gets its own folder and its own Go module,
with a `README.md` stating what it demonstrates and which lessons it draws on.

Nothing here yet. The planned progression:

| Project | After lesson | What it is |
|---|---|---|
| `toy-chain` | 08–15 | the from-scratch blockchain from Part 3: blocks, PoW, UTXO, wallet, storage, P2P, fork choice |
| `mini-evm` | 18 | a bytecode interpreter for a useful subset of EVM opcodes, with gas metering |
| `token-watcher` | 24–26 | a CLI that reads ERC-20/721 state and streams Transfer events |
| `erc20-indexer` | 31 | reorg-safe log ingestion into Postgres, with a query API |
| `signer-service` | 32 | a policy-enforcing signing service with a nonce allocator |
| `capstone` | 41 | the end-to-end system from lesson 41 |
