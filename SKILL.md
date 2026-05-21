---
name: metalclaw-dev-flash
description: Fast MetalClaw development pipeline where the main agent runs analyst, designer, developer, and documenter phases inline, and dispatches improver, reviewer, and tester phases as sub-agents when the complexity assessment calls for them. Each phase is guided by its core principles only — the agent decides file content/structure based on context.
argument-hint: [requirement description or session directory]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
---

Run the MetalClaw development pipeline with a **hybrid execution model**: the main agent handles most phases inline, and only delegates the quality-gate phases (`improver`, `reviewer`, `tester`) to sub-agents when the Complexity Assessment says they're needed.

## Instructions

1. Determine the root document directory as `<root-doc-dir>`, use the directory if user specifies, default to `doc/`.
1. Determine the root session directory as `<root-session-dir>`, use the directory if user specifies, default to `changes/`.
2. Determine the current dev session directory as `<dev-session-dir>`:
   - Use the directory if user specifies the session directory.
   - Otherwise, if user provides requirements info, create `<root-session-dir>/<YYYY>/<MM>/<DD>/YYYY-MM-DD-<NNN>-<requirement-short-name>/` (where `<NNN>` is a zero-padded sequential number starting from 001 within that day) and write the raw requirements to `<dev-session-dir>/requirement-raw.md`.
   - Otherwise, require user to specify the session directory or requirements info.
3. Run each phase by following the principles in [Phases](#phases). Before starting each phase, write the phase name to `<dev-session-dir>/STATE`.
   - **Inline phases** (main agent): `analyst`, `designer`, `developer`, `documenter`. Execute these directly using the main agent's tools.
   - **Sub-agent phases** (when included): `improver`, `reviewer`, `tester`. Dispatch via the `Agent` tool — see [Sub-Agent Dispatch](#sub-agent-dispatch).
4. The pipeline is conditional:
   - `analyst` and `designer` always run (inline).
   - After `designer`, perform the [Complexity Assessment](#complexity-assessment) and record the decision to `<dev-session-dir>/auto-plan.md`.
   - `improver` runs (as a sub-agent) only if assessed as needed; otherwise go straight to `developer`.
   - After `developer` (inline), run `reviewer` (as a sub-agent) only if assessed as needed.
   - After the reviewer decision resolves, run `tester` (as a sub-agent) only if assessed as needed. If `tester` surfaces unfixed errors (files matching `<dev-session-dir>/integrations-error-*.md` without a `-DONE` suffix), loop back to `developer` → `[reviewer?]` → `tester`, capped at 3 fix iterations.
   - `documenter` (inline) always runs last.
5. When the workflow is complete:
   - Write `done` to `<dev-session-dir>/STATE`.
   - Output a summary of what was accomplished, including which optional phases were skipped and why.

## Phases

### 1. analyst (always, inline)

**Output file:** `<dev-session-dir>/requirement.md`

**Execution Steps:**
- Load relevant context. If `<root-doc-dir>/prd/` or a similar PRD directory exists, skim `overview.md`, `architecture/architecture.md`, `architecture/roles.md`, `dictionaries/dictionaries.md`, and any model/process/application docs whose names match the requirement keywords. Skip files that don't exist.
- Ask clarifying questions for unclear points; present possible answer options; let the user choose or supply a custom answer. Append the Q&A back to `<dev-session-dir>/requirement-raw.md`.
- Turn the raw requirement into a structured requirement: background & objectives, user stories with acceptance criteria, in-scope / out-of-scope, affected roles, models, processes, and applications.
- Write the structured requirement to `<dev-session-dir>/requirement.md`.

**Principles:**
- Describe **what**, not **how** — no technical implementation in the requirement doc.
- **Don't assume. Don't hide confusion. Surface tradeoffs.** State assumptions explicitly; if multiple interpretations exist, present them — don't pick silently.
- Define **strong, verifiable success criteria**. Reject vague goals like "make it work."
- Focus only on documents relevant to the requirement scope — do not read everything.
- Note inconsistencies found between existing documents for later resolution.

### 2. designer (always, inline)

**Output file:** `<dev-session-dir>/design.md`

**Execution Steps:**
- Design the technical solution for `<dev-session-dir>/requirement.md` and write it to `<dev-session-dir>/design.md`

**Principles:**
- **Simplicity first.** Ask: "Would a senior engineer say this is overcomplicated?" If yes, simplify.
- Fit the existing world — reuse existing patterns over inventing new abstractions.
- Avoid over-engineering; no speculative extensibility for hypothetical futures.
- Use extended thinking for non-trivial trade-offs, but keep the written design tight.

### 3. improver (conditional, sub-agent)

**Execution:** dispatch via `Agent` (see [Sub-Agent Dispatch](#sub-agent-dispatch)).

**Output files:** `<dev-session-dir>/design-review.md` and an updated `<dev-session-dir>/design.md` (preserve the original as `<dev-session-dir>/design-raw.md`).

**Execution Steps (include in the sub-agent prompt):**
- Read `<dev-session-dir>/requirement.md` and `<dev-session-dir>/design.md`.
- Write improvement suggestions to `<dev-session-dir>/design-review.md`.
- Rename the current `design.md` to `design-raw.md`, then produce a new `design.md` with the accepted improvements applied.

**Principles (include in the sub-agent prompt):**
- Simpler approaches win. Reject complexity that doesn't pay for itself.
- Suggest improvements only where they materially help — avoid churn.
- Preserve the original design as `design-raw.md` so the decision trail is auditable.

### 4. developer (always, inline)

**Execution Steps:**
- Check for unfixed error files first: `<dev-session-dir>/integrations-error-*.md` **without** a `-DONE` suffix.
  - **Bug Fix Mode** (any unfixed file exists): read each error, analyze the root cause, fix the code, re-run the failing tests, then rename each fixed file `integrations-error-<N>.md` → `integrations-error-<N>-DONE.md`.
  - **Normal Development Mode** (no unfixed files): implement the design `<dev-session-dir>/design.md`, write/update unit tests and verify they pass, then run lint and fix any issues.

**Principles:**
- **Simplicity first.** Minimum code that solves the problem. Nothing speculative.
- **Surgical changes.** Touch only what the task requires. Don't "improve" adjacent code, don't refactor what isn't broken, don't reflow formatting. Match existing style even if you'd do it differently.
- Clean up orphans **your** changes created (newly-unused imports/variables/functions). Don't remove pre-existing dead code unless asked.
- Every changed line should trace directly back to the design document.
- If you spot unrelated dead code, mention it — don't delete it.

### 5. reviewer (conditional, sub-agent)

**Execution:** dispatch via `Agent` (see [Sub-Agent Dispatch](#sub-agent-dispatch)).

**Output file:** `<dev-session-dir>/code-review.md`

**Execution Steps (include in the sub-agent prompt):**
- Read `<dev-session-dir>/design.md`.
- Identify every file the developer created or modified (use `git diff` or recent file changes).
- Review the diff for issues or improvements; write findings to `<dev-session-dir>/code-review.md`.
- Apply accepted improvements directly to the code.
- Run all unit tests and confirm they still pass.

**Principles (include in the sub-agent prompt):**
- Focus on correctness, design fit, security, concurrency, and error handling — not style nits.
- Apply improvements, don't just list them; the trail is the diff plus `code-review.md`.
- Keep improvements surgical — no opportunistic rewrites of adjacent code.
- Don't regress passing tests; re-run after each change.

### 6. tester (conditional, sub-agent)

**Execution:** dispatch via `Agent` (see [Sub-Agent Dispatch](#sub-agent-dispatch)).

**Output file:** `<dev-session-dir>/test-report-v<N>.md` (where `<N>` is the sequential run number).

**Execution Steps (include in the sub-agent prompt):**
- Read `<dev-session-dir>/design.md`; review the implemented source and existing unit tests.
- Write or update **integration tests** based on the design's integration test plan. Cover every acceptance criterion in the requirement and the edge cases it implies. Add a short comment on each test case describing the scenario it verifies.
- Run all integration tests; analyze the results carefully.
- Write `<dev-session-dir>/test-report-v<N>.md` summarizing the run.
- If any test fails: pick the next sequential `<N>` by checking existing `integrations-error-*.md` files, then write `<dev-session-dir>/integrations-error-<N>.md` with (a) which tests failed, (b) error messages / stack traces, (c) likely root cause, (d) suggested fix approach. Do **not** add the `-DONE` suffix — that's the developer's job once fixed.

**Principles (include in the sub-agent prompt):**
- **Never modify source code** — only tests. Source changes belong to the developer phase.
- Every acceptance criterion and implied edge case must have coverage; missing coverage is a test-phase failure.
- Each test case needs a comment describing the scenario it verifies.
- Failures produce structured error reports so the developer loop can act on them; no silent passes.

### 7. documenter (always, inline)

**Execution Steps:**
- Determine the PRD directory `<prd-dir>` (default `<root-doc-dir>/prd/` or whatever the repo convention is). If it does not exist, stop this phase gracefully — nothing to document against.
- Read `<dev-session-dir>/requirement.md` and `<dev-session-dir>/design.md`.
- Load relevant PRD documents: `overview.md`, `architecture/architecture.md`, `architecture/roles.md`, `dictionaries/dictionaries.md`, and any model/process/application docs whose names match the requirement keywords. Skip files that don't exist.
- Update model docs under `<prd-dir>/models/`, dictionary docs under `<prd-dir>/dictionaries/`, process/rule docs under `<prd-dir>/procedures/`, and application/page docs under `<prd-dir>/applications/` to reflect this change.
- Update the global files (`overview.md`, `architecture/architecture.md`, `architecture/roles.md`) only if the change genuinely impacts product positioning, architecture, or roles.
- Output a concise summary of what was added or changed across the PRD.

**Principles:**
- Follow the existing PRD format. If none exists, pick a clean, consistent structure.
- Touch global files only when the change genuinely affects them — no cosmetic edits.
- Focus on documents relevant to the requirement scope; do not read or edit everything.
- Note inconsistencies between existing docs for later resolution.

## Sub-Agent Dispatch

When a conditional phase (`improver`, `reviewer`, or `tester`) is included by the Complexity Assessment, run it as an isolated sub-agent instead of inline. This keeps the quality-gate judgment independent of the main agent's context.

Call the `Agent` tool with:
- `description`: `"<phase> phase"` — e.g., `"reviewer phase"`.
- `subagent_type`: `"general-purpose"`.
- `prompt`: a self-contained instruction to the sub-agent that includes:
  1. The phase's **role line** (e.g., "You are an expert code reviewer.").
  2. The phase's **Execution Steps** listed under that phase in [Phases](#phases), verbatim.
  3. The phase's **Principles** listed under that phase in [Phases](#phases), verbatim.
  4. The actual `<dev-session-dir>` path (substituted, not a placeholder).
  5. The canonical output file path(s) for that phase.
  6. A note that **output format/structure is the sub-agent's judgment call** — no rigid template, just satisfy the principles and keep downstream phases able to consume the output.
  7. A final instruction to return a concise summary of what it did (files written, key findings, pass/fail for tester).

After the sub-agent returns:
- Read the artifacts it produced (don't trust the summary alone — verify the files exist).
- For `tester`, scan `<dev-session-dir>/integrations-error-*.md` (without `-DONE`) to decide whether to loop back to `developer`.
- For `improver` and `reviewer`, continue to the next phase in the pipeline.

## Complexity Assessment

After `designer` finishes, **Read** `<dev-session-dir>/requirement.md` and `<dev-session-dir>/design.md`, then judge:

**Include `improver` when ANY of these hold:**
- New architectural pattern, framework, or cross-cutting concern is introduced (e.g., new auth flow, caching layer, messaging, background jobs)
- Non-trivial algorithm, concurrency, performance, or security decision is involved
- Data model or API contract is newly added or reshaped in a way that is hard to reverse
- The design spans 3+ modules/components, or the implementation plan lists 5+ ordered tasks
- The requirement has ambiguity or multiple viable solutions that benefit from a senior second-opinion
- The affected surface is core/shared code used by many callers

**Skip `improver` when ALL of these hold:**
- The change is localized (typically 1–2 files or one component)
- No schema, API contract, or public interface change
- The design reuses existing patterns and introduces no new abstractions
- The implementation plan has a small, linear set of tasks (≤ 3)
- Examples: copy tweaks, small UI adjustments, bugfixes with obvious root cause, renames, simple CRUD additions following existing conventions

**Include `reviewer` when ANY of these hold:**
- The change touches core/shared code, security-sensitive paths, or logic with non-trivial branching
- New or reshaped data models, API contracts, or public interfaces
- Concurrency, transactions, error handling, or performance-sensitive code is involved
- Multiple files are modified, or the diff is substantive (roughly 50+ changed lines of non-boilerplate code)
- Developer output has uncertainty flags, TODOs, or deviations from the design

**Skip `reviewer` when ALL of these hold:**
- Change is trivial and mechanically obvious (e.g., copy tweak, style/format, typo fix, dependency bump without API change)
- No logic, data, API, or interface change
- Diff is small (a few lines) and follows an existing pattern verbatim
- No security, concurrency, or error-handling implications

**Include `tester` when ANY of these hold:**
- Backend logic, data layer, API, business rules, or integrations are touched
- Multiple components interact or a new flow is introduced
- Requirement defines acceptance criteria that can be verified via integration tests
- Design lists an integration test plan with runnable scenarios
- The change risks regressions in existing flows touched by the diff

**Skip `tester` when ALL of these hold:**
- Change is purely cosmetic/static (copy, styles, typos, comments, formatting, docs-only)
- Change cannot be meaningfully covered by integration tests (e.g., pure visual tweak)
- No backend, data, API, or business rule is modified
- Requirement has no integration-testable acceptance criteria

**Tie-breaker rule:** When uncertain, include the phase. The cost of an unneeded review is low; the cost of skipping a needed one can be a regression.

Record the decision to `<dev-session-dir>/auto-plan.md`. Keep the file short — phase-by-phase include/skip plus a one-line rationale each, and the effective pipeline on one line. No rigid template.

## Pipeline

```
analyst -> designer -> [improver†] -> developer -> [reviewer†] -> [tester†] -> documenter
                                       ^                           |
                                       |     (if tests fail)       |
                                       +---------------------------+
```

`†` = runs as a sub-agent via `Agent` dispatch. Phases without `†` run inline in the main agent. Phases in `[brackets]` run only if the Complexity Assessment marks them as needed. Max 3 fix iterations through `developer → [reviewer?] → tester`; after that, stop and report failure.

## Resume Support

If `<dev-session-dir>/STATE` already exists when starting:
- Read the state to determine which phase was active last.
- If `<dev-session-dir>/auto-plan.md` exists, honor its phase decisions; otherwise re-run the Complexity Assessment before continuing past `designer`.
- Skip already-completed phases and continue from the next phase in the effective pipeline.
- State progression: `analyst` → `designer` → `[improver]` → `developer` → `[reviewer]` → `[tester]` → `documenter` → `done`.
