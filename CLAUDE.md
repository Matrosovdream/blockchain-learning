# blockchain-learning — working notes for Claude

A course repo: **68 blockchain lessons, all examples in Go**, beginner → production.
Nothing is written yet; the repo holds the structure, the plan and 68 stubs.

## Layout

| Path | What it is |
|---|---|
| [learning-plan/README.md](learning-plan/README.md) | reader-facing index of all 68 lessons |
| [learning-plan/PLAN.md](learning-plan/PLAN.md) | authoring conventions + **how to extend the course** |
| [learning-plan/plan/](learning-plan/plan/) | **the per-lesson spec**, one file per part — topics, sub-points, pitfalls, example seeds |
| [learning-plan/plan/backlog.md](learning-plan/plan/backlog.md) | unscheduled and rejected topics |
| [learning-plan/PROGRESS.md](learning-plan/PROGRESS.md) | what's written, and what the user has finished |
| `learning-plan/NN-slug.md` | one lesson each; currently stubs with the section skeleton pre-filled |
| `learning-plan/examples/NN-slug/` | runnable Go programs per lesson: `1-easy` / `2-medium` / `3-hard` |
| `learning-plan/examples/_template/` | copy this to start a lesson's examples |
| `learning-plan/cheatsheets/` | dense reference sheets, written after their lessons |
| `practice/` | the user's exercise answers (gitignored except its bones) |
| `projects/` | bigger multi-lesson builds |

## When asked to write a lesson

1. **Read its spec first** — the stub links to it (`plan/part-NN-*.md#nn--title`). It defines the goals,
   the numbered topic list with sub-points, the pitfalls, and the example seeds. Don't improvise the scope.
2. Keep the fixed section order: Goals → Concepts → Exercises → Best Practices & Pitfalls → Checklist → Resources.
3. `## Concepts` is the bulk of the file: one `###` per spec topic, in order, **covering every sub-point**.
   Prose first (what problem, what breaks without it), then a short Go snippet.
4. Name the real incident when one exists (the DAO, Parity, PS3 nonce reuse, Ronin, Nomad).
5. Cross-link prerequisites with a relative link to the prerequisite lesson file.
6. Then build `examples/NN-slug/` from the template, expanding the spec's seeds to the tier counts.
7. Update the lesson's row in PROGRESS.md.

## When asked to extend the course

Follow the checklist in [PLAN.md § Extending the course](learning-plan/PLAN.md). The short version:
numbers are never reused or renumbered; a new lesson needs a spec entry, a stub, a README line and a
PROGRESS row; ideas without a number go in the backlog with a reason.

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
