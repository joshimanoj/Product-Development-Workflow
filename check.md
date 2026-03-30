# /check — Automated Verification Gate

## Purpose

Run all automated checks — unit tests, integration tests, functional/E2E tests, type checking, linting, and regression checks. `/check` does not verify visual correctness — that is the responsibility of `/uat`. Run after all tasks in `TODO.md` are marked `[x]` complete.

---

## Prerequisites

- All tasks in `TODO.md` are `[x]` complete
- All commits made (verify with `git log`)
- Currently on a feature branch — not main

---

## Repo Setup

The GitHub repo must already exist — it is set up in Sprint 0, not here. Claude verifies the remote is configured and the feature branch is pushed. If the remote is missing, stop and tell the user: "No remote configured. Return to Sprint 0 to create the repo before running /check."

```bash
# Verify remote and push feature branch
git remote -v          # must show origin — if not, return to Sprint 0
git push origin [feature-branch-name]

# Confirm CI file exists (set up in Sprint 0)
ls .github/workflows/ci.yml
git remote -v
```

---

## Phase 1: Test Suite

### A. Affected Unit Tests (Local)

```bash
vitest related --run [changed files]
# or
pytest --testpaths [affected]
# or
go test ./[affected]/...
# or
cargo test [affected]
# or
flutter test [affected file]
```

Expected: All affected tests ✅ — tests related to changed files plus any shared modules they touch. Any failure must be fixed before proceeding. The full suite across all stories runs in CI (Phase 6) — not repeated in full locally on every story.

### B. Type Safety

```bash
tsc --noEmit | mypy . | go build | cargo check | dart analyze
```

### C. Linting

```bash
npm run lint | flake8 | golangci-lint | cargo clippy | swiftlint | ktlint
```

---

## Phase 2: Functional Test Suite

Run all E2E and functional tests written during `/prd`. All must be GREEN before proceeding. This validates full user-facing behaviour — not just units.

```bash
npx playwright test e2e/story-x/          # web
detox test --configuration ios.sim.release    # mobile (iOS)
detox test --configuration android.emu.release  # mobile (Android)
flutter drive --target=test_driver/app.dart     # Flutter
```

---

## Phase 3: Acceptance Criteria Check

All gaps and untestable criteria were resolved at `/prd` time. This phase confirms — it does not discover.

For each acceptance criterion in `story-N.md`, verify its mapped FT is GREEN. If every FT is green, all criteria are covered by definition — no further mapping judgment is needed.

```markdown
Story #X — Acceptance Criteria Confirmation:
[x] AC-1 — FT-1 GREEN ✅
[x] AC-2 — FT-2 GREEN ✅
[x] AC-3 (UAT-only, no FT — declared at /prd) ✅
```

If an FT is RED here, that is a test failure — handle it in Phase 5b under "Test failure."
If a criterion has no FT and was not declared UAT-only at `/prd` time, that is a `/prd` process failure — log it to `HANDOVER.md` as process debt and proceed. Do not attempt to write new FTs here.

---

## Phase 4: Regression Check

- Review `task_spec_document.md` Integration Points section for components sharing logic.
- Run specific test suites for affected modules.
- Smoke test critical paths: auth, core feature, checkout if applicable.

---

## Phase 5: Local Summary

Create a verification report. If anything is ❌ — STOP, fix, re-run `/check` from the failed phase.

```markdown
## Story #X — /check Report

Unit tests: [N]/[N] passing
Integration tests: [N]/[N] passing
Functional/E2E: [N]/[N] passing
Type checking: ✅ No errors
Linting: ✅ No errors
Acceptance audit: ✅ All criteria covered
Regression: ✅ Affected components tested

Ready for: /uat (UI story) | Merge (Backend story)
```

---

## Phase 5b: Failure Recovery (Local)

Only reached if Phase 5 Local Summary has one or more ❌ items. Do not proceed to Phase 6 until all items below are resolved and `/check` re-runs clean from Phase 1.

| Failure Type | Action |
|-------------|--------|
| Test failure (unit / integration / functional) | Return to `/dev` — add a fix task to `TODO.md`, write a failing test that reproduces the bug, fix Red-Green-Refactor, commit, then re-run `/check` from Phase 1. |
| Type error or lint error | Fix inline without a full `/dev` cycle. Re-run Safety Gate (lint + type check), then re-run `/check` from Phase 2. |
| Acceptance criteria gap — small (missing assertion, wrong test setup, one-line fix) | Return to `/dev` — add the fix as a new Track B task in `TODO.md`, fix Red-Green-Refactor, commit, then re-run `/check` from Phase 1. |
| Acceptance criteria gap — large (requires new infrastructure, new `/prd` + `/dev` cycle, or affects multiple stories) | **Do NOT fix inline. Stop `/check`.** Create a new story in `sprint.md` for the gap. The new story gets its own `/prd` → `/dev` → `/check` cycle. The current story proceeds to `/uat` or merge without the gap — document it in `HANDOVER.md` as known debt. Never rewrite FTs to make them pass — that defeats the gate. |
| Same failure after 3 attempts | Trigger `/spike` — do not keep cycling through `/dev` and `/check`. |

**Rules:**
- Update `TODO.md` with the fix task before returning to `/dev` — no silent fixes.
- After the fix is committed, always re-run `/check` from Phase 1 — not from the phase that failed. A fix in one area can mask failures elsewhere.
- Do not push to CI until local `/check` is fully green.

---

## Phase 6: Push to CI

Only proceed if the local summary is fully green.

```bash
git push origin [feature-branch-name]
```

After pushing, immediately tail the CI run — do not wait for the user to check back:

```bash
# Get the run ID triggered by this push (wait a moment for it to register)
/opt/homebrew/bin/gh run list --repo [owner/repo] --branch [feature-branch-name] --limit 1

# Watch and block until complete — prints live job status
/opt/homebrew/bin/gh run watch [run-id] --repo [owner/repo]

# If watch is unavailable, poll every 15s:
/opt/homebrew/bin/gh run view [run-id] --repo [owner/repo]
```

Claude runs these commands inline and reports the result directly — no need for the user to check GitHub manually.

> **15-Minute Rule:** If CI fails and cannot be fixed in 15 minutes, rollback immediately: `git reset --hard [last-stable-commit] && git push origin [branch] --force`. Document the failure and create a spike task.

### CI Failure Recovery

- If CI fails and the failure is reproducible locally: return to `/dev`, add a fix task to `TODO.md`, fix Red-Green-Refactor, re-run local `/check` fully from Phase 1, then push again.
- If CI fails with an environment-specific issue (flaky test, missing secret, infra problem): fix the CI config directly and push. Do not return to `/dev`.
- After 3 CI failures on the same story: apply the 15-minute rule, rollback, and raise a spike task before continuing.

---

## Phase 7: Handoff

### Backend Stories — Merge Ready

No UAT required. `/check` passing + CI green = merge ready. Generate PR and move to next story.

### UI Stories — Auto-launch /uat

Do not merge yet. Immediately invoke the `/uat` skill — do not ask the user to run it manually. `/check` hands off directly to `/uat` when all checks are green and the story type is UI.

---

## Archive Before Next Story

> **Hard Gate:** Required: Archive task spec BEFORE clearing. `mv task_spec_document.md completed/story-X-tasks.md` → commit → then clear `TODO.md`.

```bash
mv task_spec_document.md completed/story-X-tasks.md
git add completed/story-X-tasks.md
git commit -m "chore: archive task spec for story #X"
echo "# Ready for next story" > TODO.md

# Mark complete in Sprint.md
# - [x] Story #X: [name]
```

## Full Safety Net — How the Layers Work Together

| Layer | When | What It Catches |
|-------|------|-----------------|
| PostToolUse hook | Every file save in Claude | Formatting + linting errors instantly |
| Pre-commit hook | Every git commit | Failing unit tests for staged files |
| npm run verify-app | On demand | Smoke test against running dev server |
| CI — unit + integration | Every push | All unit + integration tests |
| CI — web-playwright | Every push | All functional/E2E tests (web) |
| CI — mobile-e2e | Every push (mobile) | Full mobile E2E suite |
| /check | Before /uat or merge | Full automated suite + regression audit |
| /uat | UI stories before merge | Human: look, feel, interaction sign-off |
| Mac notification | When Claude stops | Never miss a prompt while stepping away |

---

## Pre-commit Hook Setup

Place at `.git/hooks/pre-commit` (make executable with `chmod +x`):

```bash
#!/bin/sh
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(ts|tsx|js|jsx|py|go|rs|dart)$')

if [ -n "$STAGED_FILES" ]; then
  if [ -f ./node_modules/.bin/vitest ]; then
    npx vitest related --run $STAGED_FILES
    if [ $? -ne 0 ]; then
      echo "❌ Tests failed. Fix before committing."
      exit 1
    fi
  else
    echo "⚠️ Vitest not found. Skipping local test check."
  fi
fi

echo "✅ Safety checks passed."
exit 0
```
