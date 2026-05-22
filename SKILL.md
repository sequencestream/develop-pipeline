---
name: develop-pipeline
description: Hybrid development pipeline. The main agent runs analyst, designer, developer, and documenter inline, and dispatches improver, reviewer, and tester as sub-agents when the complexity assessment requires. Each phase is principle-guided — the agent decides file content and structure based on context.
argument-hint: [requirement description or session directory]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
---

Run the development pipeline with a **hybrid execution model**: the main agent runs most phases inline and delegates the quality-gate phases (`improver`, `reviewer`, `tester`) to sub-agents only when the Complexity Assessment says they're needed.

## Instructions

1. Resolve `<root-doc-dir>` (user-specified, else `doc/`) and `<root-session-dir>` (user-specified, else `changes/`).
2. Resolve `<dev-session-dir>`:
   - If the user specifies one, use it.
   - Else if the user supplies a requirement, create `<root-session-dir>/<YYYY>/<MM>/<DD>/YYYY-MM-DD-<NNN>-<requirement-short-name>/` (`<NNN>` = zero-padded daily sequence starting at 001) and write the raw requirement to `<dev-session-dir>/requirement-raw.md`.
   - Else ask the user for one of the two.
3. Run each phase by its principles in [Phases](#phases). Before starting a phase, write its name to `<dev-session-dir>/STATE`. Inline phases (`analyst`, `designer`, `developer`, `documenter`) run in the main agent; sub-agent phases (`improver`, `reviewer`, `tester`) go through [Sub-Agent Dispatch](#sub-agent-dispatch).
4. Run the pipeline conditionally per the [Complexity Assessment](#complexity-assessment); record the decision in `<dev-session-dir>/auto-plan.md`. `analyst` and `designer` always run. If `tester` produces unfixed `<dev-session-dir>/integrations-error-*.md` files (no `-DONE` suffix), loop back through `developer → [reviewer?] → tester`, capped at 3 iterations.
5. On completion, write `done` to `<dev-session-dir>/STATE` and output a summary, noting which optional phases were skipped and why.

## Phases

### 1. analyst (always, inline)

**Output file:** `<dev-session-dir>/requirement.md`

**Execution Steps:**
- Load relevant context. If `<root-doc-dir>/prd/` (or a similar PRD directory) exists, skim `overview.md`, `architecture/architecture.md`, `architecture/roles.md`, `dictionaries/dictionaries.md`, and any model/process/application docs whose names match the requirement keywords. Skip files that don't exist.
- Ask clarifying questions on unclear points; present options; let the user choose or supply a custom answer. Append Q&A to `<dev-session-dir>/requirement-raw.md`.
- Turn the raw requirement into a structured requirement (background & objectives, user stories with acceptance criteria, in/out of scope, affected roles, models, processes, applications) and write it to the output file.

**Principles:**
- Describe **what**, not **how** — no technical implementation in the requirement doc.
- **Don't assume. Don't hide confusion. Surface tradeoffs.** State assumptions explicitly; if multiple interpretations exist, present them — don't pick silently.
- Define **strong, verifiable success criteria**. Reject vague goals like "make it work."
- Focus only on documents relevant to the requirement scope — do not read everything.
- Note inconsistencies between existing docs for later resolution.

### 2. designer (always, inline)

**Output file:** `<dev-session-dir>/design.md`

**Execution Steps:**
- Design the technical solution for `<dev-session-dir>/requirement.md` and write it to the output file.

**Principles:**
- **Simplicity first.** Ask: "Would a senior engineer say this is overcomplicated?" If yes, simplify.
- Fit the existing world — reuse existing patterns over inventing new abstractions.
- Avoid over-engineering; no speculative extensibility for hypothetical futures.
- Use extended thinking for non-trivial trade-offs, but keep the written design tight.

### 3. improver (conditional, sub-agent)

**Execution:** dispatch via `Agent` (see [Sub-Agent Dispatch](#sub-agent-dispatch)).

**Output files:** `<dev-session-dir>/design-review.md` and an updated `<dev-session-dir>/design.md` (original preserved as `<dev-session-dir>/design-raw.md`).

**Execution Steps (include in the sub-agent prompt):**
- Read `<dev-session-dir>/requirement.md` and `<dev-session-dir>/design.md`.
- Write improvement suggestions to `design-review.md`.
- Rename the current `design.md` to `design-raw.md`, then write a new `design.md` with accepted improvements applied.

**Principles (include in the sub-agent prompt):**
- Simpler approaches win. Reject complexity that doesn't pay for itself.
- Suggest improvements only where they materially help — avoid churn.
- Preserve the original design as `design-raw.md` so the decision trail is auditable.

### 4. developer (always, inline)

**Execution Steps:**
- First check for `<dev-session-dir>/integrations-error-*.md` files **without** a `-DONE` suffix.
  - **Bug Fix Mode** (unfixed files exist): for each, read it, find the root cause, fix the code, re-run the failing tests, then rename `integrations-error-<N>.md` → `integrations-error-<N>-DONE.md`.
  - **Normal Development Mode** (none exist): implement `<dev-session-dir>/design.md`, write/update unit tests and verify they pass, then run lint and fix any issues.

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
- Identify every file the developer created or modified (`git diff` or recent file changes).
- Review the diff for issues or improvements; write findings to `code-review.md`.
- Apply accepted improvements directly to the code.
- Run all unit tests and confirm they still pass.

**Principles (include in the sub-agent prompt):**
- Focus on correctness, design fit, security, concurrency, and error handling — not style nits.
- Apply improvements, don't just list them; the trail is the diff plus `code-review.md`.
- Keep improvements surgical — no opportunistic rewrites of adjacent code.
- Don't regress passing tests; re-run after each change.

### 6. tester (conditional, sub-agent)

**Execution:** dispatch via `Agent` (see [Sub-Agent Dispatch](#sub-agent-dispatch)).

**Output file:** `<dev-session-dir>/test-report-v<N>.md` (`<N>` = sequential run number).

**Execution Steps (include in the sub-agent prompt):**
- Read `<dev-session-dir>/design.md`; review the implementation and existing unit tests.
- Write or update **integration tests** based on the design's integration test plan. Cover every acceptance criterion and its implied edge cases. Add a short comment on each test describing the scenario it verifies.
- Run all integration tests and analyze the results.
- Write `test-report-v<N>.md` summarizing the run.
- On any failure: pick the next sequential `<N>` (check existing `integrations-error-*.md`), then write `<dev-session-dir>/integrations-error-<N>.md` with (a) failing tests, (b) error messages / stack traces, (c) likely root cause, (d) suggested fix. Do **not** add the `-DONE` suffix — the developer adds it after fixing.

**Principles (include in the sub-agent prompt):**
- **Never modify source code** — only tests. Source changes belong to the developer phase.
- Every acceptance criterion and implied edge case must have coverage; missing coverage is a test-phase failure.
- Each test case needs a comment describing the scenario it verifies.
- Failures produce structured error reports so the developer loop can act on them; no silent passes.

### 7. documenter (conditional, inline)

**Execution Steps:**
- Resolve the PRD directory `<prd-dir>` (default `<root-doc-dir>/prd/`, or the repo convention). If it doesn't exist, stop this phase gracefully — nothing to document against.
- Load relevant PRD docs, and any model/process/application docs whose names match the requirement keywords. Skip files that don't exist.
- Update the global files (overview/architecture/roles.md/terms) if product positioning, architecture, or roles are genuinely affected.
- Output a concise summary of what was added or changed across the PRD.

**Principles:**
- Follow the existing PRD format. If none exists, pick a clean, consistent structure.
- Touch global files only when the change genuinely affects them — no cosmetic edits.
- Focus on documents relevant to the requirement scope; do not read or edit everything.
- Note inconsistencies between existing docs for later resolution.

## Sub-Agent Dispatch

When the Complexity Assessment includes `improver`, `reviewer`, or `tester`, run it as an isolated sub-agent to keep the quality-gate judgment independent of the main agent's context.

Call the `Agent` tool with:
- `description`: `"<phase> phase"` (e.g. `"reviewer phase"`).
- `subagent_type`: `"general-purpose"`.
- `prompt`: a self-contained instruction including:
  1. The phase's **role line** (e.g. "You are an expert code reviewer.").
  2. The phase's **Execution Steps** and **Principles** from [Phases](#phases), verbatim.
  3. The actual `<dev-session-dir>` path (substituted, not a placeholder) and the canonical output file path(s).
  4. A note that **output format is the sub-agent's call** — no rigid template, just satisfy the principles and stay consumable by downstream phases.
  5. A final instruction to return a concise summary (files written, key findings, pass/fail for tester).

After the sub-agent returns:
- Verify the artifacts exist on disk — don't trust the summary alone.
- For `tester`, scan `<dev-session-dir>/integrations-error-*.md` (without `-DONE`) to decide whether to loop back to `developer`.
- For `improver` and `reviewer`, continue to the next phase.

## Complexity Assessment

After `designer` finishes, read `<dev-session-dir>/requirement.md` and `<dev-session-dir>/design.md`, then judge.

For each phase below: **include if ANY** of the include bullets hold; **skip only if ALL** of the skip bullets hold.

**`improver`**
- Include: new architectural pattern, framework, or cross-cutting concern (auth, caching, messaging, background jobs); non-trivial algorithm, concurrency, performance, or security decision; newly added or hard-to-reverse data model / API contract; design spans 3+ modules or 5+ ordered tasks; ambiguous requirement with multiple viable solutions; affected surface is core/shared code with many callers.
- Skip: localized change (1–2 files / one component); no schema, API contract, or public interface change; reuses existing patterns with no new abstractions; ≤3 linear tasks. Typical examples: copy tweaks, small UI adjustments, obvious bugfixes, renames, simple CRUD following existing conventions.

**`reviewer`**
- Include: touches core/shared or security-sensitive code, or logic with non-trivial branching; new or reshaped data models / API contracts / public interfaces; concurrency, transactions, error handling, or performance-sensitive code; multiple files modified or ~50+ non-boilerplate changed lines; developer output has uncertainty flags, TODOs, or deviations from the design.
- Skip: trivial and mechanically obvious (copy tweak, style/format, typo, dependency bump without API change); no logic/data/API/interface change; tiny diff that follows an existing pattern verbatim; no security, concurrency, or error-handling implications.

**`tester`**
- Include: backend logic, data layer, API, business rules, or integrations touched; multiple components interact or a new flow is introduced; requirement has integration-testable acceptance criteria; design has an integration test plan; change risks regressions in flows touched by the diff.
- Skip: purely cosmetic/static (copy, styles, typos, comments, formatting, docs-only); cannot be meaningfully covered by integration tests; no backend/data/API/business-rule change; no integration-testable acceptance criteria.

**`documenter`**
- Include: PRD directory exists and the change affects a documented model, dictionary, procedure/rule, or application/page; new or reshaped user-facing behavior, pipeline, or role responsibility; product positioning, architecture, or roles genuinely impacted; new domain concepts, terms, or business rules introduced.
- Skip: no PRD directory exists; change is internal/technical only (refactor, build/CI tweak, dependency bump, code-only bug fix with no semantic change); no in-scope models, dictionaries, procedures, or applications affected; no impact on overview, architecture, or roles.

**Tie-breaker:** when uncertain, include the phase. An unneeded review is cheap; a skipped needed one can mean a regression.

Record the decision to `<dev-session-dir>/auto-plan.md` — phase-by-phase include/skip with one-line rationales, plus the effective pipeline on one line. No rigid template.

## pipeline

```
analyst -> designer -> [improver†] -> developer -> [reviewer†] -> [tester†] -> [documenter]
                                       ^                           |
                                       |     (if tests fail)       |
                                       +---------------------------+
```

`†` = runs as a sub-agent via `Agent` dispatch. Phases without `†` run inline in the main agent. Phases in `[brackets]` run only if the Complexity Assessment marks them as needed. Max 3 fix iterations through `developer → [reviewer?] → tester`; after that, stop and report failure.

## Resume Support

If `<dev-session-dir>/STATE` already exists when starting:
- Read it to determine which phase was active last.
- If `<dev-session-dir>/auto-plan.md` exists, honor its decisions; otherwise re-run the Complexity Assessment before continuing past `designer`.
- Skip completed phases and continue from the next phase in the effective pipeline (see the diagram above).
