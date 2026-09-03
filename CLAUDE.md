# blockchain-learning — working notes for Claude

A course repo: 41 blockchain lessons, **all examples in Go**, beginner → production.

## Layout

| Path | What it is |
|---|---|
| [learning-plan/README.md](learning-plan/README.md) | reader-facing index of all 41 lessons |
| [learning-plan/PLAN.md](learning-plan/PLAN.md) | **the authoring spec** — per-lesson topics, example seeds, prereqs, packages |
| [learning-plan/PROGRESS.md](learning-plan/PROGRESS.md) | what's written and what the user has finished |
| `learning-plan/NN-slug.md` | one lesson each; currently stubs |
| `learning-plan/examples/NN-slug/` | runnable Go programs per lesson, `1-easy` / `2-medium` / `3-hard` |
| `learning-plan/examples/_template/` | copy this to start a lesson's examples |
| `learning-plan/cheatsheets/` | dense reference sheets, written after their lessons |
| `practice/` | the user's exercise answers (gitignored except its bones) |
| `projects/` | bigger multi-lesson builds |

## When asked to write a lesson

1. Read its entry in [PLAN.md](learning-plan/PLAN.md) — it defines the scope. Don't improvise the topic list.
2. Keep the fixed section order: Goals → Concepts → Exercises → Best Practices & Pitfalls → Checklist → Resources.
3. `## Concepts` is the bulk of the file: one `###` per PLAN topic, **prose first** (what problem, what
   breaks without it), then a short Go snippet. This course is explanation-heavy by design.
4. Name the real incident when one exists (the DAO, PS3 nonce reuse, Parity's uninitialized library).
5. Cross-link prerequisites with a relative link to the prerequisite lesson file, e.g. lesson 09 links back to `08-blocks-and-chain.md`.
6. Then build `examples/NN-slug/` from the template, expanding the PLAN's example seeds to the target count.
7. Update the lesson row in PROGRESS.md.

## Non-negotiables for code in this repo

- **Run every example before adding it.** `go build`, `go vet`, `gofmt` clean; the **Output** block is real stdout.
- Each example is a complete `package main` program — never a fragment.
- **No mainnet, no real keys, no `float64` for money** (`math/big` / `uint256` only).
- Chain-dependent examples use `ethclient/simulated` or a local `anvil`.
- Test keys are hardcoded and commented as test-only.

## Style

Match the sibling repo `../golang-learning`: same file shapes, same tier emoji (🟢🟡🔴), same
"retype it into `/tmp`" workflow. Examples are numbered continuously across the three tier files and
are never renumbered once published.
