# /dev — Implement Each Task Using Red-Green-Refactor TDD

## Purpose

You are a Senior Software Engineer implementing features via Red-Green-Refactor. Every task is driven by a test written first, made to pass with minimal code, then refactored to meet architectural standards. Visual correctness is verified by the human in `/uat` — not here.

`task_spec_document.md` is the complete implementation specification. Do not perform additional planning, re-derive tasks, or generate new task breakdowns — consume the spec as written.

`/dev` supports two execution modes:
- `normal delivery mode` — execute the planned Track A work for the current story in order
- `verification recovery mode` — execute corrective implementation tasks inserted by `/check` during the `/check → /dev → /check` loop

`/dev` does not classify work, reshape scope, or repair planning artefacts. It executes the current plan or the corrective tasks handed back from verification.

---

## Shared Workflow Model

Use this shared lifecycle model consistently across `/prd`, `/dev`, and `/check`:

- Normal delivery loop: `/prd` → `/dev` → `/check` → `/uat` or merge
- Planning failure loop: `/check` → `/prd` → `/dev` if new implementation work was added → `/check`
- Verification failure loop: `/check` → `/dev` → `/check`

Within this model, `/dev` owns implementation only. Planning repairs stay in `/prd`; verification routing stays in `/check`.

---

## Prerequisites

- **`task_spec_document.md`** — epic-scoped task details, sectioned by story (run `/prd` if missing)
- **`TODO.md`** — epic-scoped task checklist, sectioned by story
- **`architecture-dev-summary.md`** — stack, naming conventions, file structure, test runner commands
- **`story-N-ui-context.md`** — story-specific UI context; UI stories only; load the file for the current story

---

## Step 0: Identify the Current Story and Task

Normal delivery mode:
- follow the story-scoped `TODO.md` from top to bottom
- write Track B and Track C tests RED first for the current story
- then execute Track A and the story/epic technical integration gates

Verification recovery mode:
- if `/check` inserted a corrective task at the top of the current story section, execute that task first
- do not skip ahead to later planned work while a `/check`-inserted task remains unchecked
- once the corrective task is committed, hand control back to `/check`

- Read `TODO.md` to identify the current story — the first story section that has unchecked `[ ]` tasks.
- If an argument is provided (e.g. `/dev 3`): use that specific task number within the current story's section.
- Otherwise: pick the first unchecked `[ ]` task in the current story's section.
- Read the task section in `task_spec_document.md` for file paths, responsibilities, public contracts, file changes, test specifications, and architecture constraints.
- Mark the task as In Progress in `TODO.md`.

> **Canonical TODO rule:** `TODO.md` is story-scoped. Each story section contains that story's Track B tasks, Track C tasks, Track A tasks, AC Integration Tests, and Story Integration Test, in that order. `/dev` identifies the current story by finding the first story section with unchecked work.

> **Track ordering rule:** Within the current story section, write that story's Track B and Track C tests RED first — commit them — then begin Track A for that story. Move to the next story only after the current story's section is complete. Do not run Track B or Track C tests at any point during `/dev` — they are written RED and deferred to epic `/check`.

> **Story done rule:** A story is complete when all its Track A tasks are `[x]`, unit tests pass, AC integration tests pass, story integration test passes, type check is clean, lint is clean, and its Track B tests are written RED and committed. Mark the "Story Done When" checklist in `TODO.md` and move to the next story.

> **Epic done rule:** The epic is complete for `/dev` only when every story satisfies the Story Done rule and the Epic Technical Integration Test Suite is GREEN. Only then hand off to `/check`.

> **Fix task rule:** `/check` may insert new tasks at the top of the current story's section after a failure. Two shapes exist:
> - **Regression fix** (`skip_red: false`, or field absent) — a previously-passing test broke. Treat exactly like a normal Track A task: proceed to Step 1 (RED), writing the reproduction test described in the task, then continue through Step 2 and 3 as usual.
> - **Forward fix** (`skip_red: true`) — a Track B or Track C test (`target_test`, e.g. `FT-7` or `TC-2`) is failing for the first time and was never run during `/dev`. Skip Step 1 entirely. Go directly to Step 2 (GREEN), modifying only the files listed in the task's `allowed_files` (never the FT/TC file itself, never its assertions or declared threshold), then run `target_test` directly to confirm it passes, then continue to Step 3 as usual.
>
> Either shape is worked exactly like any other unchecked task in Step 0 — no special invocation needed.
>
> **Verification loop note:** when `/check` inserts one of these fix tasks, `/dev` is in `verification recovery mode` inside the `/check → /dev → /check` loop. Finish the inserted fix task first, commit it, then hand control back to `/check` so it can restart from Phase 1. Do not skip ahead to later story work while a `/check`-inserted fix task remains unchecked.

---

## Step 1: 🔴 RED — Write a Failing Test

> Skip this step entirely for forward fix tasks (`skip_red: true`) — go to Step 2.

1. Open the test file from the task spec.
2. Write the unit test using the test specification from the spec (scenario, given/when/then, data, assertions).
3. Run the test. It MUST fail. Show the failure output.

> **Expected:** ❌ Test fails — the feature is not yet implemented. This is correct.

---

## Step 2: 🟢 GREEN — Minimal Implementation

4. Open the implementation file from the task spec.
5. Write the minimum code to make the test pass. Implement only what the public contract and responsibilities specify — no internal implementation decisions belong here.
6. Run the unit test — it MUST pass.
7. Run the Safety Gate:

```bash
# Lint
npm run lint | flake8 | cargo clippy | go vet | dart analyze | ktlint

# Type check
tsc --noEmit | mypy | go build | cargo check
```

> **Not GREEN if:** test still fails, linting errors introduced, type errors in any file, or build breaks.

---

## Step 3: 🔵 REFACTOR — Clean Up

8. Review code against `architecture-dev-summary.md` (naming conventions, file structure) and `story-N-ui-context.md` (interaction patterns, component behaviour, tokens — UI stories only).
9. Remove duplication. Improve readability.
10. Re-run unit test. Re-run Safety Gate. Both must pass.

---

## Step 4: Commit & Track

```bash
git add [files modified]
git commit -m "[type](scope): [task description from TODO.md]"
# type: feat | fix | refactor | test
# Example: feat(auth): add email validation function
```

- If a new architectural pattern was introduced → update `architecture.md` and regenerate `architecture-dev-summary.md`.
- If an API contract changed → update relevant docs.
- If implementation deviated from `task_spec_document.md` in any way → append an entry to `story-change-log.md` before committing.
- If implementation deviated from an FT spec (different selector, changed API shape, renamed state) → update the FT file to match before committing. Never silently diverge from a spec.
- Mark task `[x]` in `TODO.md`.
- Ask user: "Task complete. Continue to next task?"

---

## Step 5: AC Integration Test

After all Track A tasks for an AC are complete, run the AC Integration Test Spec defined in `task_spec_document.md` for that AC.

```bash
# Run the AC integration test
[test runner command] [ac-integration-test-file]
```

> **Must pass before moving to the next AC.** If it fails: treat as a 3-strike task (Step 6). Do not proceed to the next AC until green.

AC integration test passage is not committed separately — it is a gate. Record result in `TODO.md`:

```
- [x] AC-N Integration Test — green ✅
```

---

## Step 6: Story Integration Test

Only reached when every Track A task and every AC integration test in the current story's section are `[x]` and green.

Run the Story Integration Test Spec defined in `task_spec_document.md` for this story:

```bash
[test runner command] [story-integration-test-file]
```

> **Must pass before story is declared done.** If it fails: treat as a 3-strike task (Step 7). Do not move to the next story until green.

Then verify the full Story Done checklist:

```
Unit tests: passing ✅
AC integration tests: all green ✅
Story integration test: green ✅
Type check: clean ✅
Lint: clean ✅
Track B tests for this story: written RED and committed ✅
```

Tell the user: "Story #N complete. Moving to Story #N+1." Then return to Step 0 for the next story.

---

## Step 7: Epic Integration Test

Only reached when every story in the epic is complete — all stories' Done checklists satisfied.

Run the Epic Technical Integration Test Suite defined in `task_spec_document.md`:

```bash
[test runner command] [epic-integration-test-file]
```

> **Must pass before handing off to /check.** If it fails: treat as a 3-strike task (Step 8). Do not hand off until green.

When green, tell the user: "All stories in Epic #N complete. Epic integration tests green. Ready for /check."

Do not run Track B or Track C tests at any point during `/dev`. Execution is deferred to epic `/check`.
Do not run regression suites from prior epics during normal `/dev`. Only run them if `/check` sends back a corrective task that explicitly requires one.

---

## Step 8: Error Recovery — 3-Strike Rule

After 3 failed attempts on any task or integration test:

1. STOP.
2. ROLLBACK: `git checkout [files]`
3. Propose a spike: identify why the approach failed, investigate alternatives, update task breakdown if needed.

---

## story-change-log.md

A permanent, append-only audit log of every deviation from `task_spec_document.md` across all stories. Never deleted, never overwritten. One entry per deviation, appended at the end of Step 4 before the task commit.

**What counts as a deviation — append an entry if any of these occur:**
- A different file was touched than the spec listed
- An edge case was discovered during implementation that the spec did not anticipate
- The implementation approach differed from the spec's responsibilities or architecture constraints
- Scope was adjusted (task did more or less than specified)
- A public contract was changed from what the spec defined
- An FT spec was updated to match a changed selector, API shape, or renamed state
- A decision was made that affects future tasks in this story

**What does not count — do not append an entry for:**
- Cosmetic variable naming choices within the spec's conventions
- Refactoring that does not change behaviour or file structure
- Test description wording tweaks that don't change what is being tested

**Entry format:**

```markdown
---

## Epic #N | Story #N | Task [A-N / FT-N / TC-N] | [YYYY-MM-DD HH:MM]

Deviation type: Different file | Edge case | Approach change | Scope adjustment | Contract change | FT spec update | Decision

Spec said: [what task_spec_document.md specified]

What happened: [what was actually done and why]

Impact: [effect on remaining tasks in this story or other stories in this epic, if any — or "none"]
```

**Rules:**
- File is created on first entry if it does not exist. Never pre-created empty.
- Entries are appended — never edited or deleted after writing.
- Included in the git commit that contains the deviation: `git add story-change-log.md` alongside the implementation files.
- `/check` does not gate on this file — it is informational. `/handover` reads it as a source of "decisions made" for the checkpoint.

---

## Tech Stack Reference

| Detect | Test Command | Lint | Type Check |
|--------|-------------|------|------------|
| package.json + Jest | npm test -- [file] | npm run lint | tsc --noEmit |
| package.json + Vitest | vitest [file] --run | npm run lint | tsc --noEmit |
| pytest.ini | pytest [file]::[test] | flake8 | mypy |
| go.mod | go test -run [Test] | go vet | go build |
| Cargo.toml | cargo test [test] | cargo clippy | cargo check |
| pubspec.yaml | flutter test [file] | dart analyze | dart analyze |
| build.gradle | ./gradlew test | ktlint | detekt |
| Xcode project | xcodebuild test | swiftlint | xcodebuild |
| Other / unknown | Ask user | Ask user | Ask user |
