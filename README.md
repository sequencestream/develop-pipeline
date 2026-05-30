# develop-pipeline

A skill that runs a structured development pipeline as an **agent team**, driven by a single command.

The main agent acts as the **team-lead** — the single orchestration brain and the only agent that talks to the user. It spawns seven persistent phase teammates — `analyst`, `designer`, `improver`, `developer`, `reviewer`, `tester`, `documenter` — and drives them over a shared task board: deciding which conditional phases to run (Complexity Assessment) and running the fix-loop. Each phase writes its output to a session directory (default `changes/YYYY/MM/DD/...`), so progress is auditable and resumable.

See [SKILL.md](./SKILL.md) for the full phase-by-phase specification.

## Why

Ad-hoc "vibe coding" with an AI agent tends to skip the boring-but-load-bearing steps: clarifying the requirement, sketching a design, reviewing the diff, writing integration tests, and keeping the PRD in sync. The result is fast code that's hard to verify and harder to maintain.

This skill enforces a structured pipeline — analyst → designer → developer → tester → documenter — with optional quality gates (improver, reviewer) run only when the Complexity Assessment determines the change warrants it. Each phase writes its artifacts to a session directory, so the work is auditable, resumable, and human-reviewable.

The team model lets each phase run as an independent, isolated teammate whose judgment isn't biased by the others, while staying **persistent and addressable**: the same `developer`, `reviewer`, and `tester` are re-messaged across fix-loop iterations and reuse their own prior context (the test suite they built, the implementation they wrote, the review they did) instead of being rebuilt from a one-shot prompt. Since the team-lead is the only agent talking to the user, any teammate needing a decision routes its question through the team-lead. For genuinely trivial changes the team-lead triages into a **Lightweight Mode** — running analyst → designer → developer itself, serially, with no teammates spawned (footprint: just the team-lead) — and escalates to the full team only if the change turns out non-trivial.

## Install

Installed via [skilladmin](https://github.com/sequencestream/skilladmin).

```bash
skilladmin install https://github.com/sequencestream/develop-pipeline
```

## Usage

```
/develop-pipeline Add a /healthz endpoint that returns build SHA and uptime.

/develop-pipeline changes/2026/05/21/2026-05-21-001-healthz-endpoint/
```

