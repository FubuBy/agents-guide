# Implement Jira Task — Harness Workflow

A controlled, paradigm-agnostic pipeline for implementing a task from Jira.
This rule describes **the process and the gates**. The **commands and conventions** (how to build, lint, test, what rules to follow) are read from the project's own rules — `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `README.md`, and any `*.mdc` / `*.rules` files the repository provides.

The pipeline:

```
JIRA fetch → G1 → Planning → G2 → Implement
           → Build (fix loop) → Linter (fix loop) → Tests (fix loop)
           → Code Review → G3 → FIX → G4
           → Build (fix loop) → Linter (fix loop) → Tests (fix loop) → END
```

Each step produces a machine-readable artifact under the artifacts directory.
Each `G` is a guard rail (quality gate / control point): stop the pipeline if entry criteria are not met.

**Input to this rule**: either a Jira key (e.g. `PROJ-123`) or a pasted task description.

## Step 0 — Resolve input

- If the input matches a Jira key pattern (e.g. `PROJ-123`, `ABC-4567`) → treat as Jira ID, go to JIRA fetch.
- If the input is a multi-line description → skip the fetch step, use it as the task body.
- If the input is empty → **stop and ask**: "Provide a Jira key (e.g. `PROJ-123`) or paste the task description."

**Artifacts directory** — in this order:
1. `$JIRA_TASK_ARTIFACTS_DIR/<KEY-OR-SLUG>/` if the env var is set.
2. `.jira-tasks/<KEY-OR-SLUG>/` under the repository root.

Use `unknown-<timestamp>` if no key is available. Suggest adding the chosen path to `.gitignore`.

Do NOT touch git — no branches, no commits, no PRs. Artifacts live on disk only.

## Step 0.5 — Read project rules

Before doing anything else, read the project's own instructions. Produce `00-rules.md` listing:

- **Sources read**: every rule/instruction file found in the repository (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `README.md`, any `*.mdc` / `*.rules` files, CI files like `.github/workflows/*` if they document canonical commands).
- **Project commands** to use during this run, extracted verbatim from those sources: `build`, `typecheck`, `lint`, `test`, `test-single` (and any others the project mandates). If a category is not defined — record `not defined by project rules`.
- **Conventions** that constrain implementation (architecture, layering, naming, error handling, security, banned constructs).

Do not invent commands or conventions that are not in the project's rules. If something is ambiguous or missing — **stop and ask** the user; record the answer in `00-rules.md` and continue.

## Step 1 — JIRA fetch

Goal: produce `01-task.md` containing the **full task context**, including everything around the task that helps understand it deeper.

Order of attempts:
1. **Any Jira integration available in the environment** — pick whatever works: an MCP server / connector / plugin exposing Jira tools, the Jira REST API (with credentials from env vars like `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`), or a CLI like `jira` / `acli`.
2. **Manual text** — if the user pasted the description, use it as-is. Explicitly ask the user whether comments / linked issues / subtasks / checklist / test cases exist and request them if so.
3. **Ask** — if nothing else is available, stop and ask the user to paste the description (or to enable a Jira integration).

### What to fetch — not only the task itself

If you have a Jira URL or key, recursively gather **all** of the following for context understanding:

- **The task itself**: title, key, status, assignee, reporter, priority, labels, components, fix versions, custom fields, description, attachments, links (external URLs in the description).
- **Acceptance criteria** (may live in a custom field or in the description — find it).
- **Comments** — every comment, in chronological order, with author + timestamp. Discussions in comments often contain the real spec / agreed scope changes / "why we are doing this".
- **Subtasks** — for each subtask: key, title, status, description, AC. Fetch their comments too if they look relevant.
- **Linked issues** — for context, fetch every relationship type:
  - **Parent / Epic** — fetch the parent issue fully (it usually holds strategic context, scope, definition of done for the whole epic).
  - **`blocks` / `is blocked by`** — fetch title + description summary so you understand surrounding work.
  - **`depends on` / `is dependency of`** — same.
  - **`relates to`** — fetch title + short summary.
  - **`duplicates` / `is duplicated by`** — fetch description to understand whether their AC overlaps.
  - **`clones` / `is cloned by`** — fetch to understand origin.
  - **`causes` / `is caused by`** — bug context.
- **Checklist** (e.g. Smart Checklist / Issue Checklist add-on): every item, with its `done` state. Unchecked items are required deliverables.
- **Test cases** (e.g. Zephyr / Xray / Tempo): name, steps, expected results.
- **Testing instructions** ("How to test" / "Testing instructions" — often a custom field or a separate comment from QA).
- **Definition of Done** if the project uses one (often a comment or custom field).
- **Sprint / Release** the task belongs to (may imply deadline constraints).

> Note: linked-issue **status** is informational only. It is the user's decision to take the task — do not block the pipeline on linked-issue states.

Save `01-task.md` with the structure:

```
# <KEY> — <Title>

## Meta
status / assignee / reporter / priority / labels / components / sprint / fix versions

## Description
<full description, markdown-converted>

## Acceptance Criteria
<list>

## Checklist
- [x] / [ ] items

## Test cases / Testing instructions
<text>

## Comments (chronological)
- <date> <author>: ...

## Subtasks
### <KEY-1> — <title>
description, AC, key comments

## Linked issues
### Parent/Epic: <KEY> — <title>
short summary + AC
### Blocks: ...
### Blocked by: ...
### Relates to: ...
### Other relations: ...

## External links / attachments
- ...
```

Do not skip sections — write "n/a" if a section genuinely has no content, so the absence is explicit and visible to G1.

## Step 2 — Gate G1 (input quality gate)

Read `01-task.md` and verify ALL of:

- [ ] Title is present and non-trivial (not just a key).
- [ ] Description is non-empty and informative (not "fix it" / "todo").
- [ ] Acceptance criteria or expected behavior is stated.
- [ ] Affected area is identifiable in this codebase.
- [ ] No obvious missing context (unspecified entities, undefined fields, missing IDs, ambiguous "this" references).
- [ ] **Related-context fetched**: explicit sections for Comments, Subtasks, Linked issues, Checklist, Test cases / Testing instructions (with `n/a` where genuinely empty — not silently omitted).
- [ ] **Parent / Epic context present** if such a link exists (we read it, not just listed it).
- [ ] **Checklist items make sense**: every unchecked item is either covered by the description/AC or explicitly flagged as out-of-scope.
- [ ] **Comments reviewed for scope**: latest comments do not silently change the scope of work compared to the description; if they do, the artifact reflects the updated scope.

This gate checks **content sufficiency**, not workflow status. If the user is running this rule, the task is assumed to be accepted by them — do not gate on Jira workflow states.

If ANY criterion fails — **STOP**. Write `01-task-gate-FAILED.md` listing what is missing and tell the user exactly what to add. Do not proceed.

If all pass — write `01-task-gate-OK.md` (brief reasoning) and continue.

## Step 3 — Planning

Goal: produce `02-plan.md` — a concrete implementation plan grounded in this codebase **and the project's own rules** (read in Step 0.5).

Fill the sections that apply; mark the rest `n/a` (don't silently skip).

- **Affected surface** — real files / modules / packages / scripts / assets / configs / infra resources (verify by reading the codebase, not invented).
- **Behavior changes** — public contracts that change.
- **Data & persistence** — schema / migrations / config / file format / queue / topic / feature flag changes.
- **Integrations** — external services touched, contract changes, new dependencies.
- **Cross-cutting concerns** — security, performance, observability, i18n, accessibility, error handling.
- **Tests** — what to add and at what level.
- **Docs & comms** — what to update, who to notify.
- **Rollout & rollback** — feature flag? gradual rollout? backwards compat?
- **Out of scope** — explicitly list what this task does NOT do.
- **Risks & open questions** — explicit list.
- **Rules alignment** — for each plan item, reference the project rule(s) from `00-rules.md` that constrain it.

## Step 4 — Gate G2 (plan quality gate)

Verify the plan satisfies ALL of:

- [ ] Every acceptance criterion from `01-task.md` is mapped to at least one plan item.
- [ ] All file/module paths are real (verified, not invented).
- [ ] No project conventions or rules violated (cross-checked against `00-rules.md`).
- [ ] Public contract changes are backwards-compatible OR there is an explicit migration/rollout plan.
- [ ] Test plan covers the public surface and the acceptance criteria.
- [ ] Risks and open questions are listed (not silently swept under the rug).
- [ ] Scope-creep check: every plan item traces to AC, checklist, or comments — nothing extra.

On failure — write `02-plan-gate-FAILED.md`, revise the plan, re-run G2. Max 3 iterations, then stop and ask the user.
On success — write `02-plan-gate-OK.md` and continue.

## Step 5 — Implement

Make the code changes following `02-plan.md` and the conventions in `00-rules.md`. Surgical edits only — every line traceable to the plan.

Save `03-implement.md` with: list of changed/created files, short rationale per file, deviations from the plan (with reasons).

## Step 6 — Build loop

Run the **build / typecheck command(s) declared in `00-rules.md`**. If the project defines none, write `04a-build-NONE.md` and skip this step.

If failures:
- Apply **FIX** in code → re-run → loop.
- Hard cap: 5 iterations. On cap → write `04a-build-STUCK.md`, stop, ask the user.

Save `04a-build.md` (commands used, iterations, fixes applied).

## Step 7 — Linter loop

Run the **lint command(s) declared in `00-rules.md`**. If the project defines none, write `04-lint-NONE.md` and continue (suggest adding lint in `08-summary.md` as a follow-up — do not introduce it as part of this task).

If errors:
- Apply **FIX** in code (do not suppress without justification — suppressions require an explicit comment with reasoning).
- Re-run → loop.
- Hard cap: 5 iterations. On cap → write `04-lint-STUCK.md`, stop, ask the user.

Save `04-lint.md` (commands used, iterations, fixes applied).

## Step 8 — Tests loop

Run the **test command(s) declared in `00-rules.md`**. Prefer the targeted command (single file / single test) when applicable; fall back to the full suite otherwise.

If failures:
- Apply **FIX** (in code or in test, whichever is correct — not the test if the test reflects the spec).
- Re-run → loop.
- Hard cap: 5 iterations. On cap → write `05-tests-STUCK.md`, stop, ask the user.

If the project has no test framework defined in its rules — **stop and ask**: continue without tests, or bootstrap a test framework first? Don't silently skip. Record the answer in `05-tests-NONE.md`.

Save `05-tests.md` (commands used, iterations, last output tail, count of tests run/passed).

## Step 9 — Code Review (self-review)

Re-read the full diff against:
1. The project rules captured in `00-rules.md`.
2. The plan in `02-plan.md`.
3. General hygiene: simplicity, surgical scope, naming, error handling, security, performance, accessibility, observability, doc updates.

Produce `06-review.md` with findings categorized as: **must-fix** / **should-fix** / **nit** / **out-of-scope**.
Each finding: file:line, rule or principle violated, suggested change.

## Step 10 — Gate G3 (review actionability gate)

Verify:
- [ ] Review covered every changed file.
- [ ] Every `must-fix` has a concrete fix proposal.
- [ ] No `out-of-scope` items are silently included in the planned fixes.
- [ ] If review found 0 `must-fix` and 0 `should-fix` — sanity-check: did the reviewer actually look, or was the diff trivial?

On failure — redo the review (max 2 attempts).
On success — write `06-review-gate-OK.md` and continue.

## Step 11 — FIX (apply review findings)

Apply all `must-fix` and accepted `should-fix` items. Save `07-fix.md` listing each finding and its resolution (applied / deferred + reason).

## Step 12 — Gate G4 (post-fix integrity gate)

Verify:
- [ ] Every `must-fix` from `06-review.md` is resolved in `07-fix.md`.
- [ ] No unrelated changes were introduced.
- [ ] No regressions in scope of `02-plan.md`.

On failure — loop back to Step 11. Max 2 iterations.
On success — write `07-fix-gate-OK.md` and continue.

## Step 13 — Build loop (round 2)

Same as Step 6.

## Step 14 — Linter loop (round 2)

Same as Step 7.

## Step 15 — Tests loop (round 2)

Same as Step 8. If the project's rules define a full-suite test command in addition to the targeted one, run the full suite here for confidence.

## Step 16 — END

Produce `08-summary.md`:
- Jira key + title.
- Project rules sources used (one-line summary).
- List of changed/created files.
- Build: clean (iterations) or n/a.
- Lint: clean (iterations) or none.
- Tests: green (iterations, count) or none.
- Review: N findings (M must-fix, all resolved).
- Open follow-ups (deferred items, future TODOs from `02-plan.md`).
- Suggested commit message (do **not** commit — leave to user).

Report the summary and stop.

## Hard rules for the whole pipeline

- **Stop at any failed gate.** Don't push through.
- **Don't touch git.** No `git commit`, no `git checkout -b`, no PR creation.
- **Don't suppress build/lint errors** to "make it pass" — fix the code.
- **Don't modify tests** to make them green if the test reflects the spec — fix the code.
- **The project's rules are the source of truth.** All commands and conventions come from them — do not invent or guess.
- **Surgical changes only** — every changed line must trace to `02-plan.md` or a review finding.
- **Artifacts are append-only within a run** — never delete prior step artifacts; if you have to redo a step, suffix `-v2`, `-v3`.
- **No new dependencies** without an explicit plan item — they require user approval.
- **No new tooling** introduced silently — if missing in project rules, ask.
- **No Jira workflow gating** — the user has chosen to take this task; status of the task or its links is informational only.
