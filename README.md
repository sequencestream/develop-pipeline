# develop-pipeline

A skill that runs a structured development pipeline as a single command.

The main agent runs `analyst`, `designer`, `developer`, and `documenter` inline, and dispatches `improver`, `reviewer`, and `tester` as sub-agents when a Complexity Assessment determines they are needed. Each phase writes its output to a session directory (default `changes/YYYY/MM/DD/...`) so progress is auditable and resumable.

See [SKILL.md](./SKILL.md) for the full phase-by-phase specification.

## Why

Ad-hoc "vibe coding" with an AI agent tends to skip the boring-but-load-bearing steps: clarifying the requirement, sketching a design, reviewing the diff, writing integration tests, and keeping the PRD in sync. The result is fast code that's hard to verify and harder to maintain.

This skill enforces a structured pipeline — analyst → designer → developer → tester → documenter — with optional quality gates (improver, reviewer) dispatched as isolated sub-agents when the change warrants it. Each phase writes its artifacts to a session directory, so the work is auditable, resumable, and reviewable by a human.

The hybrid execution model exists to keep the common case fast (inline phases reuse the main agent's context) while the judgment-heavy phases (improver, reviewer, tester) run in fresh sub-agent contexts so their verdicts aren't biased by the implementation they're checking. The Complexity Assessment is what decides — trivial changes skip the gates; risky ones get them.

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

