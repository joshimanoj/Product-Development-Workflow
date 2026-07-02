# /check — Automated Verification Gate

## Purpose

Verify that the completed epic satisfies requirements and does not regress existing behaviour. `/check` runs the current epic's Track B functional tests, cross-story Track B flows, Track C NFR tests, acceptance audit, planned regression targets, planned critical path tests, one final type-check/lint safety gate, and CI handoff. `/check` does not redo `/dev`'s normal unit-test and current-epic technical integration proof, and it does not verify visual correctness — that is the responsibility of `/uat`. Run after `/dev` has completed all stories and the Epic Technical Integration Test Suite is GREEN.

`/check` supports two verification modes:
- `normal verification mode` — verify a completed epic locally, then push to CI
- `recovery mode` — classify a failure, route it to `/prd` or `/dev`, and restart verification from Phase 1 after the repair

`/check` does not invent new product scope or implementation plans. It verifies, classifies failures, and routes work back to the correct owner.

---

## Shared Workflow Model

Use this shared lifecycle model consistently across `/prd`, `/dev`, and `/check`:

- Normal delivery loop: `/prd` → `/dev` → `/check` → `/uat` or merge
- Planning failure loop: `/check` → `/prd` → `/dev` if new implementation work was added → `/check`
- Verification failure loop: `/check` → `/dev` → `/check`

Within this model, `/check` owns verification and failure classification:

- planning artefact failures return to `/prd`
- implementation or verification failures return to `/dev`
- CI runs only after local verification is green

---

## Prerequisites

- All stories in the epic are complete — every story section in `TODO.md` is `[x]` with Story Done checklist satisfied
- All commits made (verify with `git log`)
- Currently on the epic feature branch — `feature/epic-N-{slug}`

---

## Repo Setup

The GitHub repo must already exist — it is set up in Sprint 0, not here. Claude verifies the remote is configured and the CI file exists. If the remote is missing, stop and tell the user: "No remote configured. Return to Sprint 0 to create the repo before running /check."

```bash
# Verify remote
git remote -v          # must show origin — if not, return to Sprint 0

# Confirm CI file exists (set up in Sprint 0)
ls .github/workflows/ci.yml
```

---

## Phase 1: Final Local Safety Gate

Run one lightweight local safety gate before requirement verification begins. `/dev` already proved unit tests, AC integration tests, story integration tests, and the current epic's technical integration suite.

### A. Type Safety

Execute one final compile/type check for the epic.

```bash
tsc --noEmit | mypy . | go build | cargo check | dart analyze
```

### B. Linting

Execute one final lint pass for the epic.

```bash
npm run lint | flake8 | golangci-lint | cargo clippy | swiftlint | ktlint
```

---

## Phase 2: Current-Epic Requirement Verification

### Phase 2a: Track B — Functional Test Suite

Run all Track B functional tests accumulated across all stories in the epic — per-story tests plus cross-story flow tests. All must be GREEN before proceeding. This is the first time these tests execute — they were written RED during `/dev` and deferred to here.

```bash
npx playwright test e2e/epic-n/          # web — runs all story subdirs + cross-story dir
detox test --configuration ios.sim.release    # mobile (iOS)
detox test --configuration android.emu.release  # mobile (Android)
flutter drive --target=test_driver/app.dart     # Flutter
```

---

### Phase 2b: Track C — NFR Test Suite

Run all Track C NFR tests written during `/prd`. All must be GREEN before proceeding. Commands and pass conditions come from the TC tasks in `task_spec_document.md`. This is the first time these tests execute — deferred from `/dev`.

Before running each TC task, verify the tool is available. Apply this resolution logic:

| Situation | Action |
|-----------|--------|
| Tool missing, installable via brew or npm | Install silently, then re-run. Examples: `brew install k6`, `npm install -g lhci` |
| Tool missing, cannot auto-install (needs licence, manual setup, or unknown source) | Stop. Tell user: "TC-N requires [tool] — install it manually and re-run /check from Phase 2b." Do not skip the test. |
| Tool installed but requires a running dev server | Start the dev server first (`npm run dev` or equivalent), run the test, then stop the server. |
| Tool requires env vars or API keys that are not set | Stop. Tell user exactly which vars are missing and where to set them. Do not run the test without them. |

```bash
# For each TC task in task_spec_document.md:
# 1. Check tool availability
which [tool] || command -v [tool]   # system tools (k6, lighthouse)
# npx-based tools (lhci, axe) self-install — no pre-check needed

# 2. Run the declared command
[TC-1 run command]   # e.g. k6 run tests/performance/epic-n-load.js
[TC-2 run command]   # e.g. npx lhci autorun

# 3. Confirm result meets declared pass condition from task_spec_document.md
```

---

### Phase 2c: Cross-Story Integration Flows

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

## Phase 3: Acceptance Audit

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
- FT or TC is RED → test failure, handle in Phase 5b under "Track B functional test failure" or "Track C NFR test failure."
- AC has no mapped Track B test, Track C test, or approved UAT-only declaration from `/prd` time → planning failure (`missing AC verification mapping`). Stop `/check`, write a Planning Defect Report, and return control to `/prd`. Do not proceed.
- NFR AC has no TC and was not declared UAT-only at `/prd` time → planning failure (`missing Track C test`). Stop `/check`, write a Planning Defect Report, and return control to `/prd`. Do not proceed.

---

## Phase 4: Regression And Critical Paths

Deterministic — no judgment calls. Four steps, run in order.

**Step 1 — Execute planned technical regression targets from `/prd`:**

Read the Technical Regression Manifest from `task_spec_document.md` and run every listed suite exactly as planned before doing any live reconciliation. Report each target explicitly:

```markdown
Technical regression target: Story #3 (Epic #1) integration suite
Command: vitest run tests/integration/epic-1/story-3-auth-repository.spec.ts
Result: ✅
```

If the manifest is missing, incomplete, or names a suite without an executable command, that is a planning failure (`missing technical regression target` or `missing epic technical integration test suite`, depending on the missing artefact). Stop, write a Planning Defect Report, and return control to `/prd`.

**Step 2 — Execute planned functional regression targets from `/prd`:**

Read the Functional Regression Manifest from `task_spec_document.md` and run every listed Track B regression target exactly as planned.

```markdown
Functional regression target: FT-3 (Epic #1, Story #2)
Command: npx playwright test e2e/epic-1/story-2/login-flow.spec.ts
Result: ✅
```

If the manifest is missing, incomplete, or names an FT without an executable command, that is a planning failure (`missing functional regression target` or `missing Track B test`, depending on the missing artefact). Stop, write a Planning Defect Report, and return control to `/prd`.

**Step 3 — Execute planned critical path targets from `/prd`:**

Read the Critical Path Manifest from `task_spec_document.md` and run every listed technical critical path suite and functional critical path test exactly as planned.

```markdown
Technical critical path: Auth repository integration
Command: vitest run tests/integration/auth-repository.spec.ts
Result: ✅

Functional critical path: Login happy path
Command: npx playwright test e2e/core/login.spec.ts
Result: ✅
```

If the manifest is missing, incomplete, or names a critical-path target without an executable command, that is a planning failure. Stop, write a Planning Defect Report, and return control to `/prd`.

**Step 4 — Get the exact change set and reconcile additional deterministic dependents:**
```bash
git diff main...$(git branch --show-current) --name-only
```
This is the ground truth across all stories in the epic. Every file that changed is in this list.

For each file in the diff, identify every other module that imports or depends on it, then run those modules' test suites. This live pass may add deterministic coverage beyond what `/prd` predicted, but it does not replace `/prd`'s planning responsibilities and it does not invent a new regression plan by judgment call.

Decision rule:
- If the dependent suite is an additional executable target implied directly by the final diff, run it and report it as extra deterministic coverage.
- If the diff shows that the planned manifests were conceptually missing required coverage, classify that as a planning failure and return to `/prd`.

Report explicitly:

```markdown
File changed: src/services/auth/tokenService.ts
Imported by: src/features/login/hooks/useAuth.ts, src/features/profile/services/profileService.ts
Running: vitest related src/features/login/hooks/useAuth.ts src/features/profile/services/profileService.ts
Result: ✅ 6/6 passing
```

If a changed file has no dependents outside its own module, state that explicitly — do not skip silently.

**Cross-check story-change-log.md:**

Read `story-change-log.md` entries for all stories in this epic. For any deviation that touched a file not in the `git diff` output, flag it: "story-change-log.md records a change to [file] but it does not appear in git diff — verify this file was committed."

---

## Phase 5: Local Summary

Create a verification report. If anything is ❌ — STOP, classify the failure, route it correctly, and re-run `/check` from Phase 1 after the fix.

```markdown
## Epic #N — /check Report

Final type checking: ✅ No errors
Final linting: ✅ No errors
Track B functional/E2E: [N]/[N] passing (across all stories)
Track C NFR tests: [N]/[N] passing
Cross-story flows: [N]/[N] passing
Acceptance audit: ✅ All criteria covered across all stories (Functional / Edge case / NFR)
Technical regression: ✅ [N]/[N] planned targets + [N] extra deterministic dependent suites GREEN
Functional regression: ✅ [N]/[N] planned targets GREEN
Critical paths: ✅ [N]/[N] technical + [N]/[N] functional GREEN

Ready for: /uat (UI epic) | Merge (Backend epic)
```

---

## Phase 5b: Failure Recovery (Local)

Only reached if Phase 5 Local Summary has one or more ❌ items. Do not proceed to Phase 6 until all items below are resolved and `/check` re-runs clean from Phase 1.

This section is `/check` recovery mode. Use it only after a concrete verification or planning failure has been observed during the normal verification run.

### Verification Failure Loop

Verification failures loop through `/dev` by way of `TODO.md`. The handoff is deterministic:

1. `/check` identifies the failing verification target and the most likely owning Track A task.
2. `/check` writes one or more corrective implementation tasks into `TODO.md` at the top of the current story section.
3. Control returns to `/dev`.
4. `/dev` reads `TODO.md` Step 0 and picks the newly inserted fix task first.
5. `/dev` implements the fix, commits it, and preserves all previously completed tasks unchanged.
6. Control returns to `/check`.
7. `/check` re-runs from Phase 1 — never from the middle of the failed phase.

Use this loop for verification failures only:
- Track B functional failures
- Track C NFR failures
- regression target failures
- critical path failures
- type or lint failures if they require implementation work rather than a trivial inline correction

Do not use this loop for planning failures. Planning failures return to `/prd`, not `/dev`.

### Fix Task Schema

When any of the three test-failure rows below fires, `/check` writes a new task into `TODO.md` — not a vague "add a fix task" note, but an object with these fields:

| Field | Meaning |
|-------|---------|
| `target_test` | The failing test's id (e.g. unit test name, `FT-7`, `TC-2`) |
| `suspect_task_id` | The Track A task most likely responsible — found via file/flow overlap with `task_spec_document.md` |
| `raw_failure_output` | The actual assertion diff / stack trace, verbatim — not paraphrased |
| `allowed_files` | Implementation/config files this fix may touch. Never includes the test file itself, its assertions, or (for Track C) its declared threshold |
| `skip_red` | `true` for Track B/C, regression, and critical-path failures when the failing test already exists — `false` or omitted only when a fresh reproduction test is genuinely required |

Insert the task at the **top of the current story's section** in `TODO.md`, ahead of any unstarted Track A work, so `/dev`'s Step 0 "first unchecked task" logic picks it up first.

| Failure Type | Action |
|-------------|--------|
| Track B functional test failure — first execution (Phase 2a / 2c) | Trace suspect Track A task(s) via file/flow overlap with the FT's User Flow steps → `suspect_task_id`. Write a fix task with `skip_red: true`, `target_test` = the FT id. Return to `/dev` — it skips RED (the test already exists and is already failing) and goes straight to GREEN against the existing FT, then refactors. Commit, then re-run `/check` from Phase 1. |
| Track C NFR test failure — first execution (Phase 2b) | Only after the Phase 2b tool/server/env-var resolution table has ruled out an environment cause. If the implementation or config genuinely doesn't meet the threshold: trace suspect Track A task(s) the same way, write a fix task with `skip_red: true`, `target_test` = the TC id. Return to `/dev` — it fixes the implementation/config until the TC's declared threshold passes. Commit, then re-run `/check` from Phase 1. Do not change the pass threshold to make the test pass — that defeats the gate. |
| Technical or functional regression target failure (Phase 4) | Trace suspect Track A task(s) via manifest target overlap and changed-file impact. Write a fix task with `skip_red: true` unless a new reproduction test is needed. Return to `/dev`, fix the implementation, then re-run `/check` from Phase 1. |
| Critical path failure (Phase 4) | Treat exactly like a regression target failure. `/check` does not re-scope the critical path at this stage. |
| Type error or lint error | Fix inline only if the change is trivial and does not alter planned behaviour. Otherwise create a corrective task and return to `/dev`. In either case, re-run `/check` from Phase 1. |
| Planning failure (missing Track A task, missing task unit test spec, missing task integration test spec, missing AC integration test spec, missing story integration test spec, missing epic technical integration test suite, missing Track B test, missing Track C test, missing AC verification mapping, missing technical regression target, missing functional regression target, missing critical path target, missing prerequisite implementation task) | **Do NOT fix inline. Stop `/check`.** Write a Planning Defect Report listing each missing or inconsistent planning artefact, return control to `/prd`, regenerate only the affected planning artefacts, append any new implementation work to `task_spec_document.md` and `TODO.md`, complete that work via `/dev`, then re-run `/check` from Phase 1. Preserve completed tasks exactly as they are. |
| Same `target_test` failing after 3 fix attempts | Trigger `/spike` — do not keep generating fix tasks for the same test. |

**Rules:**
- Update `TODO.md` with the fix task before returning to `/dev` — no silent fixes.
- After the fix is committed, always re-run `/check` from Phase 1 — not from the phase that failed.
- Do not push to CI until local `/check` is fully green.
- Track B and Track C failures here are expected to be first-run failures — they were written RED during `/dev` and have never executed before. Treat them as implementation gaps, not test infrastructure problems, unless the test itself is clearly misconfigured.
- `/check` may extend coverage only by deterministic reconciliation from the final diff. It does not decide "what is worth testing" by open-ended judgment.
- Planning failures never skip ahead to regression, UAT, or CI. `/check` halts immediately until `/prd` and any resulting `/dev` work are complete.

### Planning Defect Report

When `/check` detects a planning failure, create a report with this structure before returning control to `/prd`:

```markdown
## Planning Defect Report — Epic #N

- Defect type: [missing Track A task | missing task unit test spec | missing task integration test spec | missing AC integration test spec | missing story integration test spec | missing epic technical integration test suite | missing Track B test | missing Track C test | missing AC verification mapping | missing technical regression target | missing functional regression target | missing critical path target | missing prerequisite implementation task]
- Story / AC / Target: [exact reference]
- Why this blocks `/check`: [one sentence]
- Required `/prd` repair: [artifact(s) to regenerate]
- Follow-on `/dev` work required: [new task ids or "none yet — regenerate first"]
```

Repeat the five-line defect block once per planning defect. Do not collapse unrelated defects into one generic row.

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
| /check | After all stories in epic done | Final lint/type-check + Track B + Track C + cross-story flows + regression + critical paths |
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
