# /check — Automated Verification Gate

## Purpose

Run all automated checks for the completed epic — unit tests, integration tests, Track B functional/E2E tests, Track C NFR tests, cross-story integration flows, type checking, linting, and regression checks. `/check` does not verify visual correctness — that is the responsibility of `/uat`. Run after all stories in the epic are complete and `/dev` has confirmed all stories done.

---

## Prerequisites

- All stories in the epic are complete — every story section in `TODO.md` is `[x]` with Story Done checklist satisfied
- All commits made (verify with `git log`)
- Currently on the epic feature branch — `feature/epic-N-{slug}`

---

## Repo Setup

The GitHub repo must already exist — it is set up in Sprint 0, not here. Claude verifies the remote is configured and the epic feature branch is pushed. If the remote is missing, stop and tell the user: "No remote configured. Return to Sprint 0 to create the repo before running /check."

```bash
# Verify remote and push epic feature branch
git remote -v          # must show origin — if not, return to Sprint 0
git push origin feature/epic-N-{slug}

# Confirm CI file exists (set up in Sprint 0)
ls .github/workflows/ci.yml
git remote -v
```

---

## Phase 1: Test Suite

### A. Affected Unit Tests (Local)

Run unit tests for all files changed across all stories in the epic — not just the last story.

```bash
vitest related --run [all changed files across epic]
# or
pytest --testpaths [affected]
# or
go test ./[affected]/...
# or
cargo test [affected]
# or
flutter test [affected file]
```

Expected: All affected tests ✅. Any failure must be fixed before proceeding. The full suite across all epics runs in CI (Phase 6) — not repeated in full locally.

### B. Type Safety

```bash
tsc --noEmit | mypy . | go build | cargo check | dart analyze
```

### C. Linting

```bash
npm run lint | flake8 | golangci-lint | cargo clippy | swiftlint | ktlint
```

---

## Phase 2a: Track B — Functional Test Suite

Run all Track B functional tests accumulated across all stories in the epic — per-story tests plus cross-story flow tests. All must be GREEN before proceeding. This is the first time these tests execute — they were written RED during `/dev` and deferred to here.

```bash
npx playwright test e2e/epic-n/          # web — runs all story subdirs + cross-story dir
detox test --configuration ios.sim.release    # mobile (iOS)
detox test --configuration android.emu.release  # mobile (Android)
flutter drive --target=test_driver/app.dart     # Flutter
```

---

## Phase 2b: Track C — NFR Test Suite

Run all Track C NFR tests written during `/prd`. All must be GREEN before proceeding. Commands and pass conditions come from the TC tasks in `task_spec_document.md`. This is the first time these tests execute — deferred from `/dev`.

```bash
# Run each TC task's run command from task_spec_document.md
[TC-1 run command]   # e.g. k6 run tests/performance/epic-n-load.js
[TC-2 run command]   # e.g. npx axe-core tests/accessibility/epic-n.html
# Confirm each meets its declared pass condition (threshold from task_spec_document.md)
```

---

## Phase 2c: Cross-Story Integration Flows

Run the cross-story Track B tests specifically and confirm they exercise transitions between stories correctly. These are the journeys that could not be tested at story level — e.g. sign up → verify email → first login as a single end-to-end flow.

```bash
npx playwright test e2e/epic-n/cross-story/   # web
# equivalent for mobile
```

For each cross-story flow defined in `epic-N.md` Cross-story Notes, confirm:

```markdown
Cross-story flow: Sign up → verify email → first login
  FT-X: e2e/epic-n/cross-story/signup-to-login.spec.ts ✅
  Transitions tested: Story #1 success state → Story #2 trigger → Story #3 completion
```

Any RED cross-story flow is treated as a test failure — handle in Phase 5b.

---

## Phase 3: Acceptance Criteria Check

All gaps and untestable criteria were resolved at `/prd` time. This phase confirms — it does not discover.

For each acceptance criterion across all stories in the epic, verify it maps to a GREEN test or a declared UAT-only item. Group confirmation by story.

```markdown
Epic #N — Acceptance Criteria Confirmation:

Story #[N]: [Title]
  Functional criteria:
  [x] AC-1 (Functional) — FT-1 GREEN ✅
  [x] AC-2 (Functional) — FT-2 GREEN ✅
  Edge cases:
  [x] AC-3 (Edge case) — FT-3 GREEN ✅
  NFR criteria:
  [x] AC-4 (NFR — automatable) — TC-1 GREEN ✅
  [x] AC-5 (NFR — UAT-only, declared at /prd) ✅

Story #[N+1]: [Title]
  ...

Cross-story flows:
  [x] Flow 1: Sign up → verify email → first login — FT-X GREEN ✅
```

**Failure rules — apply per row:**
- FT or TC is RED → test failure, handle in Phase 5b under "Test failure."
- AC has no mapped test and was not declared UAT-only at `/prd` time → `/prd` process failure — log to `HANDOVER.md` as process debt and proceed. Do not write new tests here.
- NFR AC has no TC and was not declared UAT-only or E2E-testable at `/prd` time → `/prd` process failure — log to `HANDOVER.md` as process debt and proceed.

---

## Phase 4: Regression Check

Deterministic — no judgment calls. Four steps, run in order.

**Step 1 — Get the exact change set:**
```bash
git diff main...$(git branch --show-current) --name-only
```
This is the ground truth across all stories in the epic. Every file that changed is in this list.

**Step 2 — Trace dependents and run their test suites:**

For each file in the diff, identify every other module that imports or depends on it, then run those modules' test suites. Report explicitly:

```markdown
File changed: src/services/auth/tokenService.ts
Imported by: src/features/login/hooks/useAuth.ts, src/features/profile/services/profileService.ts
Running: vitest related src/features/login/hooks/useAuth.ts src/features/profile/services/profileService.ts
Result: ✅ 6/6 passing
```

If a changed file has no dependents outside its own module, state that explicitly — do not skip silently.

**Step 3 — Run critical path tests:**

Read Critical Paths from `architecture.md` Section 8. Run each named test file exactly as listed. No interpretation — run the file, report the result.

```bash
# Example — actual paths come from architecture.md Section 8
vitest run tests/auth/login.test.ts
vitest run tests/core/[feature].test.ts
```

```markdown
Critical path: Auth flow — tests/auth/login.test.ts ✅
Critical path: [Core feature] — tests/core/[feature].test.ts ✅
```

**Step 4 — Cross-check story-change-log.md:**

Read `story-change-log.md` entries for all stories in this epic. For any deviation that touched a file not in the `git diff` output, flag it: "story-change-log.md records a change to [file] but it does not appear in git diff — verify this file was committed."

---

## Phase 5: Local Summary

Create a verification report. If anything is ❌ — STOP, fix, re-run `/check` from the failed phase.

```markdown
## Epic #N — /check Report

Unit tests: [N]/[N] passing
Track B functional/E2E: [N]/[N] passing (across all stories)
Track C NFR tests: [N]/[N] passing
Cross-story flows: [N]/[N] passing
Type checking: ✅ No errors
Linting: ✅ No errors
Acceptance audit: ✅ All criteria covered across all stories (Functional / Edge case / NFR)
Regression: ✅ [N] dependent modules tested, [N] critical paths GREEN

Ready for: /uat (UI epic) | Merge (Backend epic)
```

---

## Phase 5b: Failure Recovery (Local)

Only reached if Phase 5 Local Summary has one or more ❌ items. Do not proceed to Phase 6 until all items below are resolved and `/check` re-runs clean from Phase 1.

| Failure Type | Action |
|-------------|--------|
| Test failure (unit / integration / Track B functional) | Return to `/dev` — add a fix task to `TODO.md`, write a failing test that reproduces the bug, fix Red-Green-Refactor, commit, then re-run `/check` from Phase 1. |
| Track C NFR test failure | Return to `/dev` — add a fix task to `TODO.md`, fix the implementation or configuration until the TC passes its declared threshold, commit, then re-run `/check` from Phase 1. Do not change the pass threshold to make the test pass — that defeats the gate. |
| Type error or lint error | Fix inline without a full `/dev` cycle. Re-run Safety Gate (lint + type check), then re-run `/check` from Phase 2a. |
| Acceptance criteria gap — small (missing assertion, wrong test setup, one-line fix) | Return to `/dev` — add the fix as a new Track B or Track C task in `TODO.md`, fix Red-Green-Refactor, commit, then re-run `/check` from Phase 1. |
| Acceptance criteria gap — large (requires new infrastructure, new `/prd` + `/dev` cycle, or affects multiple stories) | **Do NOT fix inline. Stop `/check`.** Create a new story in `sprint.md` for the gap. The new story gets its own `/prd` → `/dev` → `/check` cycle. The current story proceeds to `/uat` or merge without the gap — document it in `HANDOVER.md` as known debt. Never rewrite FTs or TCs to make them pass — that defeats the gate. |
| Same failure after 3 attempts | Trigger `/spike` — do not keep cycling through `/dev` and `/check`. |

**Rules:**
- Update `TODO.md` with the fix task before returning to `/dev` — no silent fixes.
- After the fix is committed, always re-run `/check` from Phase 1 — not from the phase that failed.
- Do not push to CI until local `/check` is fully green.
- Track B and Track C failures here are expected to be first-run failures — they were written RED during `/dev` and have never executed before. Treat them as implementation gaps, not test infrastructure problems, unless the test itself is clearly misconfigured.

---

## Phase 6: Push to CI

Only proceed if the local summary is fully green.

```bash
git push origin feature/epic-N-{slug}
```

After pushing, immediately tail the CI run — do not wait for the user to check back:

```bash
# Get the run ID triggered by this push (wait a moment for it to register)
/opt/homebrew/bin/gh run list --repo [owner/repo] --branch feature/epic-N-{slug} --limit 1

# Watch and block until complete — prints live job status
/opt/homebrew/bin/gh run watch [run-id] --repo [owner/repo]

# If watch is unavailable, poll every 15s:
/opt/homebrew/bin/gh run view [run-id] --repo [owner/repo]
```

Claude runs these commands inline and reports the result directly — no need for the user to check GitHub manually.

> **15-Minute Rule:** If CI fails and cannot be fixed in 15 minutes, rollback immediately: `git reset --hard [last-stable-commit] && git push origin feature/epic-N-{slug} --force`. Document the failure and create a spike task.

### CI Failure Recovery

- If CI fails and the failure is reproducible locally: return to `/dev`, add a fix task to `TODO.md`, fix Red-Green-Refactor, re-run local `/check` fully from Phase 1, then push again.
- If CI fails with an environment-specific issue (flaky test, missing secret, infra problem): fix the CI config directly and push. Do not return to `/dev`.
- After 3 CI failures on the same story: apply the 15-minute rule, rollback, and raise a spike task before continuing.

---

## Phase 7: Handoff

### Backend Epics — Merge Ready

No UAT required. `/check` passing + CI green = merge ready. Generate PR for the epic branch and move to next epic.

### UI Epics — Auto-launch /uat

Do not merge yet. Immediately invoke the `/uat` skill — do not ask the user to run it manually. `/check` hands off directly to `/uat` when all checks are green and the epic contains UI stories.

---

## Archive Before Next Epic

> **Hard Gate:** Required: Archive task spec BEFORE clearing. `mv task_spec_document.md completed/epic-N-tasks.md` → commit → then clear `TODO.md`.

```bash
mv task_spec_document.md completed/epic-N-tasks.md
git add completed/epic-N-tasks.md
git commit -m "chore: archive task spec for epic #N"
echo "# Ready for next epic" > TODO.md

# Mark complete in Sprint.md
# ✅ Epic #N: [name] — all stories merged
```

## Full Safety Net — How the Layers Work Together

| Layer | When | What It Catches |
|-------|------|-----------------|
| PostToolUse hook | Every file save in Claude | Formatting + linting errors instantly |
| Pre-commit hook | Every git commit | Failing unit tests for staged files |
| npm run verify-app | On demand | Smoke test against running dev server |
| CI — Track A (unit) | Every push (all branches incl. main) | All unit + integration tests |
| CI — Track B (web-playwright) | Feature branch push only | All functional/E2E tests (web) |
| CI — Track B (mobile-e2e) | Feature branch push only (mobile) | Full mobile E2E suite |
| CI — Track C (k6 / Lighthouse / axe) | Feature branch push only | Performance, accessibility, NFR thresholds |
| /check | After all stories in epic done | Unit tests + Track B + Track C + cross-story flows + regression |
| /uat | UI epics before merge | Human: look, feel, interaction sign-off across all stories |
| Post-merge CI (main) | After UAT squash merge | Track A only — confirms merge didn't break main |
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
