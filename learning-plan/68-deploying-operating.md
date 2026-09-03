# 68 — Deploying & Operating Chain Services in Production

> **Status:** 🦴 bones only — not written yet.  
> **Full spec:** [plan/part-16-cross-chain-production-operations.md](plan/part-16-cross-chain-production-operations.md#68-deploying-operating-chain-services-in-production) — topics, sub-topics, pitfalls, example seeds, resources.

| | |
|---|---|
| **Part** | Part 16 — Cross-Chain & Production Operations |
| **Prerequisites** | [35](35-observability-reliability.md), [32](32-key-management-signing.md) |
| **Unlocks** | — |
| **Examples to build** | 13 (🟢 3 · 🟡 7 · 🔴 3) |
| **Topics in spec** | 10 |

*containers, config, secrets, migrations, rollouts and the incident response that chain services need*

## Goals

- Package a Go chain service into a minimal, secure container.
- Configure and inject secrets without ever baking a key into an image.
- Deploy with health checks, graceful shutdown and zero-downtime rollouts.
- Run the incident playbooks that chain services specifically need.

## Concepts

<!-- WRITE ME — this is the bulk of the lesson.
     One `###` sub-section per numbered topic in the spec, in order.
     Prose first (what problem, what breaks without it), then a short Go snippet. -->

The spec lists **10 topics** with their sub-points: [plan/part-16-cross-chain-production-operations.md](plan/part-16-cross-chain-production-operations.md#68-deploying-operating-chain-services-in-production).

### 1. Packaging

_Not written yet._

### 2. Configuration

_Not written yet._

### 3. Secrets

_Not written yet._

### 4. Health checks that mean something

_Not written yet._

### 5. Graceful shutdown

_Not written yet._

### 6. Database migrations

_Not written yet._

### 7. Deployment topology

_Not written yet._

### 8. Rollouts

_Not written yet._

### 9. Chain-specific incident playbooks

_Not written yet._

### 10. Cost and capacity

_Not written yet._

## Exercises

<!-- WRITE ME — 5–8 numbered tasks the reader writes in `practice/68-deploying-operating/`. -->

_Not written yet._

## Best Practices & Pitfalls

<!-- WRITE ME — the spec lists 6 pitfalls to cover; add the habits alongside them. -->

_Not written yet._

## Checklist

<!-- WRITE ME — one `- [ ]` "I can …" line per goal above. -->

_Not written yet._

## Resources

<!-- WRITE ME — the spec lists 5 starting points; specs/EIPs first. -->

_Not written yet._

---

**Examples:** `examples/68-deploying-operating/` — **13 runnable Go programs** (🟢 3 easy · 🟡 7 medium · 🔴 3 hard). Start from [`examples/_template/`](examples/_template/).

*Progress: [PROGRESS.md](PROGRESS.md) · Plan: [PLAN.md](PLAN.md)*
