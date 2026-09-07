# Step 08 — Blocks & the Chain · Examples

A library of **18 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/bc-ex && cd /tmp/bc-ex
go mod init scratch       # first time only
# paste the example into main.go, then:
go run .
```

**Standard library only** — no chain, no node, no dependencies. Every timestamp is a fixed
constant, so all output reproduces exactly.

Every example was compiled, `gofmt`-checked, `go vet`-ed, and run before being added — the **Output**
under each one is real stdout, and all 18 were run twice to confirm they are reproducible.

| Tier | File | Examples | What it covers |
|------|------|----------|----------------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 | the header, hashing, genesis, linking, header vs body |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–13 | serialization, tampering, validation, timestamps |
| 🔴 Hard | [3-hard.md](3-hard.md) | 14–18 | the test fixture, a real Chain, fuzzing, versioning, assembly |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md)

- [1. The header, field by field](1-easy.md#1-the-header-field-by-field)
- [2. Hashing a header](1-easy.md#2-hashing-a-header)
- [3. The genesis block](1-easy.md#3-the-genesis-block)
- [4. Chaining three blocks](1-easy.md#4-chaining-three-blocks)
- [5. Header vs body: what a light client downloads](1-easy.md#5-header-vs-body-what-a-light-client-downloads)

### 🟡 [Medium](2-medium.md)

- [6. Encode, decode, encode](2-medium.md#6-encode-decode-encode)
- [7. Why not JSON or gob](2-medium.md#7-why-not-json-or-gob)
- [8. Tampering breaks every later block](2-medium.md#8-tampering-breaks-every-later-block)
- [9. Height is convenient, not authoritative](2-medium.md#9-height-is-convenient-not-authoritative)
- [10. Stateless and stateful validation](2-medium.md#10-stateless-and-stateful-validation)
- [11. The cached-hash bug](2-medium.md#11-the-cached-hash-bug)
- [12. Timestamps are int64, not time.Time](2-medium.md#12-timestamps-are-int64-not-timetime)
- [13. Median-time-past](2-medium.md#13-median-time-past)

### 🔴 [Hard](3-hard.md)

- [14. A chainBuilder for the next seven lessons](3-hard.md#14-a-chainbuilder-for-the-next-seven-lessons)
- [15. A Chain that validates before it mutates](3-hard.md#15-a-chain-that-validates-before-it-mutates)
- [16. Fuzzing the decoder](3-hard.md#16-fuzzing-the-decoder)
- [17. Changing the format without splitting the chain](3-hard.md#17-changing-the-format-without-splitting-the-chain)
- [18. The whole thing, assembled](3-hard.md#18-the-whole-thing-assembled)

## Part 3 starts here

This is the first lesson of the eight in which **one program grows into a working blockchain**.
[Example 18](3-hard.md#18-the-whole-thing-assembled) is that program's first version — header,
Merkle root, genesis, both validation passes and a `Chain` type — and it ends with a list of
exactly what each following lesson changes about it.

[Example 14](3-hard.md#14-a-chainbuilder-for-the-next-seven-lessons) is the other one to keep:
a test fixture where valid chains are the default and invalidity is requested explicitly. Every
test in lessons 09–15 starts from it.

## The arc

1–2 — the header's fixed layout, and the hash that is the block's identity.  
3–4 — genesis, and the links that turn blocks into a chain.  
5 — why the header/body split makes light clients possible.  
6–7 — round-tripping the format, and why no reflection-based encoder can be used.  
8–9 — what tampering actually breaks, and why height is not authority.  
10–13 — the two validation passes, the caching trap, and how timestamps are bounded.  
14–18 — the fixture, the Chain, the fuzzer, format versioning, and the assembled result.

---

*Lesson: [../../08-blocks-and-chain.md](../../08-blocks-and-chain.md) · Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
