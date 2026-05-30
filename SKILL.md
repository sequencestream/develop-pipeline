---
name: develop-pipeline
description: Team-based development pipeline. The main agent acts as team-lead — the single orchestration brain and human interface — and spawns seven persistent, isolated phase teammates (analyst, designer, improver, developer, reviewer, tester, documenter) that it drives over a shared task board. Each phase is principle-guided — the teammate decides file content and structure based on context.
argument-hint: [requirement description or session directory]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskList, TaskUpdate, TaskGet
---

Run the development pipeline as an **agent team**. The main agent is the **team-lead**: the single orchestration brain (path resolution, Complexity Assessment, shared task board, fix-loop) and the only agent that talks to the user. It spawns the seven **phase teammates** (`analyst`, `designer`, `improver`, `developer`, `reviewer`, `tester`, `documenter`) on demand.

**Phase teammates are persistent and isolated.** Each is spawned at most once, then re-messaged across fix-loop reruns so it reuses its own prior context (its built test suite, its implementation, its earlier review). Their judgment stays independent of each other and of the team-lead.

## Team Topology

```
                 user
                  ↑↓                ← only team-lead talks to the user
            ┌───────────┐
            │ team-lead │  (main agent: orchestration brain + human interface + teardown)
            └─────┬─────┘
        spawns + messages (by name)
   ┌───────┬───────┬────────┬──────────┬─────────┬──────────┐
 analyst designer improver developer reviewer  tester   documenter
```

- The team-lead owns the pipeline: Complexity Assessment, the shared task board, the fix-loop, and spawning each teammate.
- **Only the team-lead may ask the user anything.** A teammate needing an `AskUserQuestion`-style clarification sends it to `team-lead` bundled with full context — see [Human-Question Protocol](#human-question-protocol).
- Teammates communicate **only** via `SendMessage`, addressing each other by **name** (`team-lead`, `analyst`, …). Plain-text output is invisible to other agents.

## Instructions (team-lead / main agent)

1. Resolve `<root-doc-dir>` (user-specified, else `specs/`, then `doc/`) and `<root-session-dir>` (user-specified, else `changes/`).
2. Resolve `<session-dir>`:
   - If the user specifies one, use it.
   - Else if the user supplies a requirement, create `<root-session-dir>/<YYYY>/<MM>/<DD>/YYYY-MM-DD-<NNN>-<requirement-short-name>/` (`<NNN>` = zero-padded daily sequence from 001) and write the raw requirement to `<session-dir>/requirement-raw.md`.
   - Else ask the user for one of the two.
3. Create the team with `TeamCreate`: `team_name = "devpipe-<session-slug>"` (slug = the `YYYY-MM-DD-<NNN>-<short-name>` leaf of `<session-dir>`, lowercased), `agent_type = "team-lead"`. This also creates the shared task board.
4. Determine the **Execution Mode** (see [Execution Mode](#execution-mode-lightweight-vs-full-team)). In **Lightweight Mode**, run `analyst → designer → developer` yourself, serially, in your own context, and skip the rest. In **Full Team Mode**, orchestrate the phase teammates per the steps below.
5. Drive the pipeline (Full Team Mode):
   - Read `<session-dir>/STATE` if present to resume (see [Resume Support](#resume-support)); otherwise write `analyst` to `STATE` and begin.
   - Write each phase's name to `STATE` before starting it.
   - Seed the task board: one task per phase you intend to run, with `blockedBy` dependencies mirroring the [pipeline](#pipeline). Create `analyst` and `designer` up front; create conditional-phase tasks when the Complexity Assessment selects them; create fix-loop tasks on demand.
   - Spawn each teammate on demand (at most once), assign its task (`TaskUpdate owner=<name>`), and send a self-contained work message (see [Phase Dispatch Message](#phase-dispatch-message)).
   - After a teammate reports done: **verify the artifact exists on disk** (don't trust the message), mark the task `completed`, and advance.
   - Run the [Complexity Assessment](#complexity-assessment) after `designer`, recording the decision to `<session-dir>/auto-plan.md`.
   - Drive the [fix-loop](#fix-loop).
6. Throughout, service the [Human-Question Protocol](#human-question-protocol) and surface phase progress to the user.
7. On completion, write `done` to `STATE` and output a summary (which optional phases were skipped and why, plus test pass/fail). On unrecoverable failure, write the failure state and report it. Then [tear the team down](#teardown).

`analyst` and `designer` always run. The fix-loop and conditional phases follow their sections' rules.

## Execution Mode (Lightweight vs Full Team)

The full team has a fixed cost (one teammate per phase). For trivial changes that overhead isn't worth it, so the team-lead triages **once at start**.

**Triage (before spawning any teammate):** read `requirement-raw.md` and skim the touched code. The change is **trivial** when **ALL four** conditional phases would `skip` under the [Complexity Assessment](#complexity-assessment) — i.e. a localized (1–2 files / one component), pattern-following change with no schema / API contract / public-interface change, no security/concurrency/error-handling implications, no integration-testable acceptance criteria, and no documented model/module/glossary/role impact. Typical: copy tweaks, small UI adjustments, obvious bugfixes, renames, simple CRUD following existing conventions.

**Lightweight Mode (trivial):**
- The team-lead spawns **no teammate**. It performs `analyst → designer → developer` itself, serially, following each phase's Execution Steps and Principles verbatim and writing the same artifacts (`requirement.md`, `design.md`, the code + unit tests).
- All conditional gates (`improver`, `reviewer`, `tester`, `documenter`) are skipped; there is **no fix-loop**.
- It still writes `STATE` per phase and records the decision to `auto-plan.md` (one line: "Lightweight Mode — trivial change, all gates skipped" + rationale). It can still ask the user directly if genuinely needed.
- Net footprint: just the **team-lead** — no teammates, no task-board fan-out.

**Full Team Mode (everything else):** the default flow — spawn `analyst`/`designer`, run the post-`designer` Complexity Assessment, then proceed with conditional gates and the fix-loop.

**Escalation escape hatch:** Lightweight Mode is a bet. If at any point the team-lead finds the change is actually non-trivial (a hidden API/schema change, branching logic, a security/concurrency concern, an emergent integration-test need), it **escalates to Full Team Mode**: keep the artifacts already written, run the Complexity Assessment now, and spawn the gate teammates it selects (re-using prior work). This honors the tie-breaker — when uncertain, prefer the gated path.

## Phases

Each phase lists the **role line**, **Execution Steps**, and **Principles** the team-lead must put, verbatim, into that teammate's dispatch message (with `<session-dir>` substituted). Output format is each teammate's own call — no rigid template — but artifacts must stay consumable by downstream phases.

### 1. analyst (always) — teammate `analyst`

**Role line:** "You are an expert requirements analyst."

**Output file:** `<session-dir>/requirement.md`

**Execution Steps:**
- Load relevant context. If `<root-doc-dir>` exists, skim `overview.md`, `architecture.md`, and any model/process/module docs matching the requirement keywords. Skip files that don't exist.
- Ask clarifying questions on unclear points via the [Human-Question Protocol](#human-question-protocol); do not invent answers. The team-lead appends the Q&A to `requirement-raw.md`.
- Turn the raw requirement into a structured requirement covering: background, **intent** (business/product/engineering), **functional scenarios** with acceptance criteria, **constraints** (business/technical/compliance, non-functional, dependencies), in/out of scope, and affected roles, models, processes, applications.

**Principles:**
- Describe **what**, not **how** — no implementation in the requirement doc.
- **Don't assume. Don't hide confusion. Surface tradeoffs.** State assumptions explicitly; present multiple interpretations rather than picking silently.
- Define **strong, verifiable success criteria**. Reject vague goals like "make it work."
- Focus only on documents relevant to the requirement scope.
- Note inconsistencies between existing docs for later resolution.

### 2. designer (always) — teammate `designer`

**Role line:** "You are an expert software designer."

**Output file:** `<session-dir>/design.md`

**Execution Steps:**
- Read `requirement.md`, design the technical solution, and write it to the output file.

**Principles:**
- **Simplicity first.** Ask: "Would a senior engineer say this is overcomplicated?" If yes, simplify.
- Fit the existing world — reuse existing patterns over inventing new abstractions.
- Avoid over-engineering; no speculative extensibility.
- Use extended thinking for non-trivial trade-offs, but keep the written design tight.

### 3. improver (conditional) — teammate `improver`

**Role line:** "You are an expert design reviewer."

**Output files:** `<session-dir>/design-review.md` and an updated `<session-dir>/design.md` (original preserved as `<session-dir>/design-raw.md`).

**Execution Steps:**
- Read `requirement.md` and `design.md`.
- Write improvement suggestions to `design-review.md`.
- Rename the current `design.md` to `design-raw.md`, then write a new `design.md` with accepted improvements applied.

**Principles:**
- Simpler approaches win. Reject complexity that doesn't pay for itself.
- Suggest improvements only where they materially help — avoid churn.
- Preserve the original as `design-raw.md` so the decision trail is auditable.

### 4. developer (always) — teammate `developer`

**Role line:** "You are an expert software developer."

**Execution Steps:**
- First check for `<session-dir>/integrations-error-*.md` files **without** a `-DONE` suffix.
  - **Bug Fix Mode** (unfixed files exist): for each, read it, find the root cause, fix the code, re-run the failing tests, then rename `integrations-error-<N>.md` → `integrations-error-<N>-DONE.md`.
  - **Normal Development Mode** (none exist): implement `design.md`, write/update unit tests and verify they pass, then run lint and fix any issues.

**Principles:**
- **Simplicity first.** Minimum code that solves the problem. Nothing speculative.
- **Surgical changes.** Touch only what the task requires. Don't "improve" adjacent code, refactor what isn't broken, or reflow formatting. Match existing style even if you'd do it differently.
- Clean up orphans **your** changes created (newly-unused imports/variables/functions). Don't remove pre-existing dead code unless asked.
- Every changed line should trace back to the design document.
- If you spot unrelated dead code, mention it — don't delete it.

### 5. reviewer (conditional) — teammate `reviewer`

**Role line:** "You are an expert code reviewer."

**Output file:** `<session-dir>/code-review.md`

**Execution Steps:**
- Read `design.md`.
- Identify every file the developer created or modified (`git diff` or recent changes).
- **First pass:** review the full diff against the design.
- **Fix-loop reruns:** review **only the incremental fix diff** since your last review — your earlier review (retained in your context) still stands for untouched code.
- Write/append findings to `code-review.md` and apply accepted improvements directly to the code.
- Run all unit tests and confirm they still pass.

**Principles:**
- Focus on correctness, design fit, security, concurrency, and error handling — not style nits.
- Apply improvements, don't just list them; the trail is the diff plus `code-review.md`.
- Keep improvements surgical — no opportunistic rewrites of adjacent code.
- Don't regress passing tests; re-run after each change.

### 6. tester (conditional) — teammate `tester`

**Role line:** "You are an expert integration tester."

**Output file:** `<session-dir>/test-report-v<N>.md` (`<N>` = sequential run number).

**Execution Steps:**
- Read `design.md`; review the implementation and existing unit tests.
- **First run:** write/update **integration tests** from the design's integration test plan, covering every acceptance criterion and its implied edge cases. Add a short comment per test describing the scenario it verifies.
- **Fix-loop reruns:** do **not** rebuild the suite. Reuse the existing tests (in your context); re-run the previously failing tests (plus any sharing the touched code path), and add new cases only if the fix exposed an uncovered scenario.
- Run the relevant integration tests and analyze the results.
- Write `test-report-v<N>.md` summarizing the run.
- On any failure: pick the next sequential `<N>` (check existing `integrations-error-*.md`), then write `<session-dir>/integrations-error-<N>.md` with (a) failing tests, (b) error messages / stack traces, (c) likely root cause, (d) suggested fix. Do **not** add `-DONE` — the developer adds it after fixing.
- In your report to `team-lead`, state pass/fail explicitly and list any `integrations-error-<N>.md` you wrote.

**Principles:**
- **Never modify source code** — only tests. Source changes belong to the developer.
- Every acceptance criterion and implied edge case must have coverage; missing coverage is a failure.
- Each test case needs a comment describing the scenario it verifies.
- Failures produce structured error reports; no silent passes.

### 7. documenter (conditional) — teammate `documenter`

Runs last, after the `reviewer → tester` chain (and its fix-loop) has settled.

**Role line:** "You are an expert technical documenter."

**Output file:** `<session-dir>/doc-changes.md`

**Execution Steps:**
- Resolve `<root-doc-dir>`. If it doesn't exist, stop gracefully and report nothing-to-document (safety net; the Complexity Assessment normally skips documenter in this case).
- Load relevant docs under `<root-doc-dir>`, plus any model/process/module docs matching the requirement keywords. Skip files that don't exist.
- Update the global files (overview/architecture/roles/term/glossary) only if product positioning, architecture, or roles are genuinely affected.
- Write a concise summary of what changed across the documents to `doc-changes.md`.

**Principles:**
- Follow the existing document format; if none exists, pick a clean, consistent structure.
- Touch global files only when the change genuinely affects them — no cosmetic edits.
- Focus on documents relevant to the requirement scope.
- Note inconsistencies between existing docs in `doc-changes.md` for later resolution.

## Spawning Teammates

The team-lead spawns phase teammates with the `Agent` tool:
- `subagent_type`: `"general-purpose"` (all phases need read/write/bash).
- `team_name`: the team from Instruction 3.
- `name`: the role name (`analyst`, `designer`, …).
- `prompt`: a self-contained briefing establishing the teammate's standing role. Per-run work is delivered later as a [Phase Dispatch Message](#phase-dispatch-message). Tell each teammate: *"You are a persistent teammate. After finishing a unit of work, mark your task completed with TaskUpdate, send a concise report to `team-lead`, then go idle and await further messages — do not shut down until you receive a shutdown request."*

## Phase Dispatch Message

When the team-lead activates a phase, it sends a `SendMessage` containing:
1. The phase's **role line**.
2. Its **Execution Steps** and **Principles** from [Phases](#phases), verbatim.
3. The real `<session-dir>` path and the canonical output file path(s).
4. The current mode where relevant (developer: Normal vs Bug-Fix; reviewer/tester: first pass vs fix-loop rerun).
5. A note that **output format is the teammate's call** — satisfy the principles, stay consumable downstream.
6. A final instruction: write the artifact(s), `TaskUpdate` the task to `completed`, then reply to `team-lead` with a concise report (files written, key findings, pass/fail for tester).

After the reply, the team-lead **verifies the artifact on disk** before advancing.

## Human-Question Protocol

Only the team-lead can ask the user. Any teammate needing a user decision:
1. Teammate → `team-lead`: send a **self-contained question package**, not just the bare question and options. The team-lead and the user cannot see the teammate's working context, so include:
   - The **question** and **options** (with what each implies / its trade-offs).
   - The **background**: why this decision is needed now, what the teammate is doing, and which file/component/requirement it concerns.
   - The teammate's **recommendation** and reasoning, if any.
   - The **impact** of each choice (what changes downstream).
   Keep working on anything not blocked by the answer.
2. `team-lead` relays the full package to the user (`AskUserQuestion` for choices, carrying the context into the question/options text; plain text otherwise), appends the Q&A to `requirement-raw.md`, then replies to the asking teammate with the answers.
3. The teammate resumes with the answers.

Batch questions where possible to minimize round-trips.

## Complexity Assessment

After `designer` finishes, the team-lead reads `requirement.md` and `design.md`, then judges which conditional phases to run. For each phase: **include if ANY** include bullet holds; **skip only if ALL** skip bullets hold.

**`improver`**
- Include: new architectural pattern, framework, or cross-cutting concern (auth, caching, messaging, background jobs); non-trivial algorithm, concurrency, performance, or security decision; newly added or hard-to-reverse data model / API contract; design spans 3+ modules or 5+ ordered tasks; ambiguous requirement with multiple viable solutions; affected surface is core/shared code with many callers.
- Skip: localized change (1–2 files / one component); no schema, API contract, or public interface change; reuses existing patterns with no new abstractions; ≤3 linear tasks. Examples: copy tweaks, small UI adjustments, obvious bugfixes, renames, simple CRUD.

**`reviewer`**
- Include: touches core/shared or security-sensitive code, or logic with non-trivial branching; new or reshaped data models / API contracts / public interfaces; concurrency, transactions, error handling, or performance-sensitive code; multiple files or ~50+ non-boilerplate changed lines; developer output has uncertainty flags, TODOs, or design deviations.
- Skip: trivial and mechanically obvious (copy tweak, style/format, typo, dependency bump without API change); no logic/data/API/interface change; tiny diff following an existing pattern verbatim; no security, concurrency, or error-handling implications.

**`tester`**
- Include: backend logic, data layer, API, business rules, or integrations touched; multiple components interact or a new flow is introduced; requirement has integration-testable acceptance criteria; design has an integration test plan; change risks regressions in touched flows.
- Skip: purely cosmetic/static (copy, styles, typos, comments, formatting, docs-only); cannot be meaningfully covered by integration tests; no backend/data/API/business-rule change; no integration-testable acceptance criteria.

**`documenter`**
- Include: `<root-doc-dir>` exists and the change affects a documented model/module, glossary, procedure/rule, or application/page; new or reshaped user-facing behavior, pipeline, or role responsibility; product positioning, architecture, or roles genuinely impacted; new domain/module concepts, glossary terms, or business rules introduced.
- Skip: `<root-doc-dir>` does not exist; change is internal/technical only (refactor, build/CI tweak, dependency bump, code-only fix with no semantic change); no in-scope models, glossary, procedures, or modules affected; no impact on overview, architecture, or roles.

**Tie-breaker:** when uncertain, include the phase. An unneeded review is cheap; a skipped needed one can mean a regression.

The team-lead records the decision to `<session-dir>/auto-plan.md` — phase-by-phase include/skip with one-line rationales, plus the effective pipeline on one line. It then creates task-board entries for the included conditional phases.

## Fix-loop

If `tester` produces unfixed `integrations-error-*.md` files (no `-DONE` suffix), the team-lead loops back through `developer → [reviewer?] → tester`, capped at **3 iterations**.

- The team-lead re-messages the **existing** `developer`, `reviewer`, and `tester` teammates (never fresh ones) so each reuses its prior context. It creates new task entries per iteration (`developer-fix-<i>`, `reviewer-<i>`, `tester-<i>`) for visibility.
- In each iteration, `reviewer` reviews only the incremental fix diff and `tester` reruns only the failing/affected tests rather than rebuilding the suite.
- `reviewer` is in the loop only if the Complexity Assessment included it, and runs before `tester` (it may edit code the tester then exercises).
- After 3 iterations with tests still failing, stop, write the failure state to `STATE`, and report failure to the user.

## pipeline

```
analyst -> designer -> [improver] -> developer -> [reviewer] -> [tester] -> [documenter]
                                        ^                            |
                                        +------(if tests fail)-------+
                                                (≤3 iterations)
```

All boxes are teammates messaged by the team-lead, run in order. Phases in `[brackets]` run only if the Complexity Assessment marks them needed. Max 3 fix iterations through `developer → [reviewer?] → tester`; after that, stop and report failure. In fix-loop iterations the persistent `reviewer`/`tester` teammates reuse their context — `reviewer` reviews only the incremental fix diff, `tester` reruns only the failing/affected tests. `documenter` runs last, once the chain has settled.

## Resume Support

If `<session-dir>/STATE` exists when the skill starts, the team-lead resumes:
- Read `STATE` for the last active phase, then re-create the team and re-spawn the teammates whose downstream work remains (a fresh process has no live teammates).
- Persistent fix-loop context is lost across a process restart — reconstruct state from the on-disk artifacts (`code-review.md`, `test-report-v<N>.md`, `integrations-error-*.md`) and treat the next `reviewer`/`tester` activation as it naturally falls in the pipeline.
- If `auto-plan.md` exists, honor its decisions; otherwise re-run the Complexity Assessment before continuing past `designer`.
- Skip completed phases and continue from the next phase in the effective pipeline.

## Teardown

When the pipeline completes or fails unrecoverably, the team-lead:
1. Confirms `<session-dir>/STATE` reads `done` (or the failure state) and that expected artifacts exist.
2. Outputs the final summary to the user.
3. Sends each live teammate a `SendMessage` `{type: "shutdown_request"}` and waits for shutdown.
4. Calls `TeamDelete` to remove the team and task board.

Leaving the team un-deleted is acceptable mid-session (e.g. pausing for resume), but a completed run should be torn down. In Lightweight Mode no teammates were spawned, so teardown is just confirming `STATE` and (if a team was created) `TeamDelete`.
