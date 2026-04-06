# /dev — Implement Each Task Using Red-Green-Refactor TDD

## Purpose

You are a Senior Software Engineer implementing features via Red-Green-Refactor. Every task is driven by a test written first, made to pass with minimal code, then refactored to meet architectural standards. Visual correctness is verified by the human in `/uat` — not here.

---

## Prerequisites

- **`task_spec_document.md`** — epic-scoped task details, sectioned by story (run `/prd` if missing)
- **`TODO.md`** — epic-scoped task checklist, sectioned by story
- **`architecture-dev-summary.md`** — stack, naming conventions, file structure, test runner commands
- **`story-N-ui-context.md`** — story-specific UI context; UI stories only; load the file for the current story

---

## Step 0: Identify the Current Story and Task

- Read `TODO.md` to identify the current story — the first story section that has unchecked `[ ]` tasks.
- If an argument is provided (e.g. `/dev 3`): use that specific task number within the current story's section.
- Otherwise: pick the first unchecked `[ ]` task in the current story's section.
- Read the task section in `task_spec_document.md` for file paths, test requirements, and implementation notes.
- Mark the task as In Progress in `TODO.md`.

> **Track ordering rule:** Track B and Track C tasks are listed at the top of `TODO.md` for the whole epic. When starting the epic for the first time, write all Track B and Track C tests for the current story RED first — commit them — then begin Track A for that story. Move to the next story's Track B/C when you reach it. Do not run Track B or Track C tests at any point during `/dev` — they are written RED and deferred to epic `/check`.

> **Story done rule:** A story is complete when all its Track A tasks are `[x]`, its unit tests pass, type check is clean, lint is clean, and its Track B tests are written RED and committed. Mark the "Story Done When" checklist in `TODO.md` and move to the next story.

---

## Step 1: 🔴 RED — Write a Failing Test

1. Open the test file from the task spec.
2. Write the test using the description, input, and expected output from the spec.
3. Run the test. It MUST fail. Show the failure output.

> **Expected:** ❌ Test fails — the feature is not yet implemented. This is correct.

---

## Step 2: 🟢 GREEN — Minimal Implementation

4. Open the implementation file from the task spec.
5. Write the minimum code to make the test pass.
6. Run the test — it MUST pass.
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

8. Review code against `architecture-dev-summary.md` (naming conventions, file structure) and `story-N-ui-context.md` (interaction patterns, component behaviour, tokens — UI stories only): naming, patterns, documentation.
9. Remove duplication. Improve readability.
10. Re-run test. Re-run Safety Gate. Both must pass.

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
- If implementation deviated from `task_spec_document.md` in any way (different file touched, edge case discovered, scope adjusted, approach changed, FT spec diverged) → append an entry to `story-change-log.md` before committing. See the `story-change-log.md` section below for entry format. Do not commit without logging the deviation first.
- If implementation deviated from an FT spec (different selector, changed API shape, renamed state) → update the FT file to match before committing. Never silently diverge from a spec.
- Mark task `[x]` in `TODO.md`.
- Ask user: "Task complete. Continue to next task?"

---

## Step 5: Story Complete — Handoff to Next Story

Only reached when every Track A task in the current story's section of `TODO.md` is marked `[x]` and the Story Done checklist is satisfied:

```
Unit tests: passing ✅
Type check: clean ✅
Lint: clean ✅
Track B tests for this story: written RED and committed ✅
```

Tell the user: "Story #N complete. Moving to Story #N+1." Then return to Step 0 for the next story.

When all stories in the epic are done — every story section's Track A tasks are `[x]` and every story's Done checklist is satisfied — tell the user: "All stories in Epic #N complete. Ready for /check."

Do not run Track B or Track C tests at any point during `/dev`. Execution is deferred to epic `/check`.

---

## Step 6: Error Recovery — 3-Strike Rule

11. STOP after 3 failed attempts.
12. ROLLBACK: `git checkout [files]`
13. Propose a spike: identify why the approach failed, investigate alternatives, update task breakdown.

---

## story-change-log.md

A permanent, append-only audit log of every deviation from `task_spec_document.md` across all stories. Never deleted, never overwritten. One entry per deviation, appended at the end of Step 4 before the task commit.

**What counts as a deviation — append an entry if any of these occur:**
- A different file was touched than the spec listed
- An edge case was discovered during implementation that the spec did not anticipate
- The implementation approach differed from the spec's Implementation Notes
- Scope was adjusted (task did more or less than specified)
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

Deviation type: Different file | Edge case | Approach change | Scope adjustment | FT spec update | Decision

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
