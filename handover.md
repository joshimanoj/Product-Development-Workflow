# /handover — Session Continuity

## Purpose

Claude's context window is finite. The handover system ensures zero knowledge loss between sessions. The primary mechanism is a lightweight checkpoint appended to `HANDOVER.md` automatically at the end of every successful `/check` pass — when state is cleanest and most complete. The pre-compaction hook acts as a safety net only, firing if no checkpoint has been written yet in the current session.

---

## Session Start Rule (Warm-Start)

At the start of every session, Claude performs a warm-start in five steps:

1. **Check for `.claude/state.json` first.** This is the precise mechanical state — epic number, active story, last completed task, track B statuses, next command. If present, read it and hold in memory.
2. **Check for `HANDOVER.md`.** Read only the most recent checkpoint block (the last `## Checkpoint` entry) for narrative context — decisions made, debt carried forward. If `HANDOVER.md` is missing or empty and `state.json` is also missing, refuse to proceed: warn "No session state found — context from your previous session may be lost. Confirm current epic and story manually before continuing." Do not proceed silently.
3. **If `state.json` is missing but `HANDOVER.md` exists**, warn: "`state.json` not found — task-level state may be lost. Reconstructing from HANDOVER.md checkpoint. Verify current epic and story before running /dev." Do not silently assume state.
4. **Set active context in memory** — active epic number and name, active story number, next command, last completed task, track B statuses, warnings and debt. Do not re-read `Sprint.md`, `architecture.md`, or any other project file unless the human explicitly asks.
5. **Greet the human with a one-line status summary and a ready prompt** — for example: "Last checkpoint: Epic #1 Account Creation — Story #3 complete, FT-1 red, FT-2 red. Ready for /dev task 5 on Story #3. Say 'continue' to proceed, or ask me anything."

---

## Primary Mechanism — Two-Layer State

### Layer 1 — `.claude/state.json` (written per `/dev` task)

After every `/dev` task commit, Claude writes `.claude/state.json`. This is the precise mechanical state — updated at task granularity so a mid-epic session crash loses at most one task. Claude writes this directly, no script or subprocess.

```json
{
  "epic": 1,
  "epic_name": "Account Creation",
  "branch": "feature/epic-1-account-creation",
  "active_story": 3,
  "last_completed_task": 4,
  "next_task": 5,
  "track_b_status": {"FT-1": "red", "FT-2": "red", "FT-3": "red"},
  "next_command": "/dev 5",
  "timestamp": "2026-03-20T14:32:00"
}
```

### Layer 2 — `HANDOVER.md` Checkpoint (written per epic merge)

After every epic merges (at the end of `/uat` After Approval, or at the end of `/check` Phase 7 for backend epics), Claude appends a checkpoint block directly to `HANDOVER.md`. No script, no subprocess — Claude writes from its own current context. This carries narrative context that `state.json` does not: decisions made, debt deferred, architectural notes.

```markdown
---

## Checkpoint: Epic #N [name] | Stories #[first]–#[last] | [YYYY-MM-DD HH:MM] | MERGED ✅

Epic complete: [one sentence — what the user can now do]

Files changed: [list]

Decisions made: [read story-change-log.md and extract every entry for stories in this epic — copy Deviation type, What happened, and Impact fields verbatim. If no entries exist, write "none."]

Warnings / debt: [linting warnings deferred, known issues, anything not to repeat]

Next: Epic #[N+1] [name] | Ready for: /prd
```

---

## Full Handover Document Format

Used when a full session handover is needed (e.g. end of a long session, before handing off to another developer, or when the pre-compaction hook fires).

```markdown
# Session Handover: [YYYY-MM-DD HH:MM]

## 1. Executive Summary

Objective: [High-level goal of this session]
Outcome: Success | Partial | Blocked
Momentum: [How urgent is the next step?]

## 2. Progress & Completed Work

Files Created: [list]
Refactors: [list]
Logic Verified: [specific tests / checks that passed]

## 3. Technical Debt & Debugging Log

Resolved Bugs: [error → fix → root cause]
Failed Approaches: [what didn't work — do NOT repeat]
Current Warnings: [linting errors or deferred issues]

## 4. Architectural & Strategic Decisions

Decision: [e.g. Hook instead of Cron job]
Rationale: [Why this over alternatives]
Future Impact: [What next dev must keep in mind]

## 5. Environment & Context

New Dependencies: [npm install / pip install commands run]
File Map:
  path/to/file.py: [why this file matters]
  .claude/hooks/...: [custom logic added this session]

## 6. Unresolved Threads & The Wall

Blocked By: [missing key, rate limit, complexity]
Mental Context: [in-flight logic not yet coded]

## 7. Immediate Next Steps (priority order)

1. [First command or task for next session]
2. [Second]
```

---

## Triggering Handovers

### Automatic — Primary: Two-Layer State

**Layer 1:** Claude writes `.claude/state.json` at the end of every `/dev` task commit. No script, no subprocess.
**Layer 2:** Claude appends a checkpoint block to `HANDOVER.md` at the end of every epic merge — at the end of `/uat` After Approval (UI epics) or `/check` Phase 7 (backend epics). No script, no subprocess.

The PreCompact hook in `.claude/settings.local.json` acts as a fallback only: it checks whether `HANDOVER.md` was updated this session and spawns a subprocess only if no checkpoint exists yet. Under normal workflow it does nothing — the epic merge already appended a checkpoint and `/dev` already updated `state.json`.

### Manual — /handover Command

```bash
python3 .claude/hooks/pre-compact-handover.py
# or via Claude shortcut: /handover
```

---

## Automation — pre-compact-handover.py

Place at `.claude/hooks/pre-compact-handover.py`. Acts as a safety net only — fires before context compaction and checks whether `HANDOVER.md` already contains a checkpoint written this session. If yes, exits immediately (no cost). If no checkpoint exists yet, spawns a Claude subprocess to generate one from the session transcript.

Under normal workflow this script does nothing, because `/check` already appended a checkpoint.

```python
import datetime, os, subprocess

def safety_net_handover():
    # Exit immediately if epic merge already wrote a checkpoint this session
    handover = "HANDOVER.md"
    if os.path.exists(handover):
        mtime = os.path.getmtime(handover)
        age_hours = (datetime.datetime.now().timestamp() - mtime) / 3600
        if age_hours < 8:  # written within this session
            print("Checkpoint exists — skipping subprocess.")
            return

    date_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    filename = "HANDOVER.md"

    prompt = f'''
You are a technical lead doing a session handover.

Using the full session transcript, generate {filename} with:
1. Executive Summary
2. Progress
3. Debugging Log
4. Architectural Decisions
5. Environment
6. Unresolved Threads
7. Immediate Next Steps (priority ordered)
'''

    try:
        result = subprocess.run(
            ['claude', '-p', prompt],
            capture_output=True, text=True, check=True
        )
        with open(filename, 'w') as f:
            f.write(result.stdout)
        print(f'Saved: {filename}')
    except Exception as e:
        print(f'Error: {e}')

if __name__ == '__main__':
    safety_net_handover()
```

---

## .claude/settings.local.json Configuration

```json
{
  "mcpServers": { },
  "hooks": {
    "PreCompact": "python3 .claude/hooks/pre-compact-handover.py",
    "Stop": "osascript -e 'display notification \"Claude needs your input\" with title \"Claude Code\" sound name \"Ping\"'",
    "PostToolUse": {
      "matcher": "edit_file|write_to_file",
      "script": "npm run format && npm run lint:fix"
    }
  },
  "commands": [
    {
      "name": "handover",
      "description": "Manually trigger a session handover document",
      "script": "python3 .claude/hooks/pre-compact-handover.py"
    }
  ]
}
```

---

## Mac Desktop Notifications

Get notified when Claude needs your input — so you can step away while long tasks run without missing a prompt. The Stop hook above fires a native Mac notification every time Claude stops and waits for a response.

### Targeted Notifications from Scripts

For more specific alerts — for example when `/check` fails or `/uat` is ready — add a `notify()` helper to any Python script:

```python
import subprocess

def notify(message, title="Claude Code"):
    subprocess.run([
        "osascript", "-e",
        f'display notification "{message}" with title "{title}" sound name "Ping"'
    ])

# Usage examples:
notify("UAT checklist ready — please review")
notify("/check failed — CI is red, rollback needed")
notify("Handover saved — safe to close session")
```

---

# Session Handover: 2026-07-02 15:08 IST

## 1. Executive Summary

Objective: Tighten the core `/prd`, `/dev`, and `/check` workflow docs around planning failures, verification failures, shared lifecycle flow, and `TODO.md` structure.
Outcome: Partial
Momentum: High — the repo is in the middle of a workflow-doc cleanup, with the next clear task being point `2` (restart-rule cleanup and deciding whether `/check` should re-run Track A coverage).

## 2. Progress & Completed Work

Files Modified:
- `prd.md`
- `dev.md`
- `check.md`

Logic Updated:
- Added a Planning Defect Intake flow to `/prd`.
- Expanded Planning Defect Report taxonomy in `/check` and aligned `/prd` intake handling with it.
- Added explicit verification failure loop documentation for `/check -> /dev -> /check`.
- Added a shared workflow model section to `/prd`, `/dev`, and `/check`.
- Made `TODO.md` structure canonical and story-scoped in `/prd`, and aligned `/dev`’s current-story interpretation to that structure.

Logic Verified:
- Text consistency pass only.
- Confirmed the moved working repo is `/Users/manojjoshi/Desktop/Personal/Projects/Product-Development-Workflow`.
- Confirmed current branch state: `main...origin/main` with modified `prd.md`, `dev.md`, and `check.md`.

## 3. Technical Debt & Debugging Log

Resolved / clarified:
- Planning failures now route explicitly from `/check` to `/prd`.
- Verification failures now route explicitly from `/check` to `/dev`.
- `TODO.md` is no longer described as an epic-top-level FT/TC list in the active model; it is now story-scoped in the latest edits.

Failed / blocked approaches:
- An earlier patch attempt targeted the old repo location under `echo` after the repo had been moved. Work continued successfully after switching to the moved repo path.

Current warnings:
- Point `2` has not been implemented yet: restart semantics are still contradictory in `/check`.
- `/check` still re-runs affected unit tests and technical integration coverage, even though `/dev` already runs AC integration, story integration, and epic technical integration gates.
- `HANDOVER.md` in this repo doubles as the command spec document; this session handover was appended to the bottom rather than replacing that content.
- No `.claude/state.json` exists in the moved repo.

## 4. Architectural & Strategic Decisions

Decision: Use a shared lifecycle model across `/prd`, `/dev`, and `/check`.
Rationale: The workflow logic had become correct in pieces but hard to read as a system. A shared model reduces drift and makes failure routing easier to follow.
Future Impact: Any further edits to restart rules or CI flow should preserve the same canonical loops in all three docs.

Decision: Make `TODO.md` story-scoped and canonical.
Rationale: `/prd`, `/dev`, and `/check` were interpreting `TODO.md` inconsistently. A story-scoped structure makes current-story selection and fix-task insertion deterministic.
Future Impact: Any future `TODO.md` examples or rules should keep the same ordering: Track B -> Track C -> Track A -> AC Integration Tests -> Story Integration Test.

Decision: Keep AC classification separate from the QA scenario layer used to plan Track B.
Rationale: Requirements categorization and functional test-design serve different purposes. Keeping them separate allows `/prd` to preserve the current Functional / Edge / Environmental model while still planning derived Track B scenarios when needed.
Future Impact: Any future edits to `/prd`, `/check`, `/dev`, or `/uat` should preserve the boundary: AC classification defines requirement type, QA scenarios define Track B coverage, Track C owns automatable NFRs, and UAT owns human-judgment verification.

## 5. Environment & Context

Repo Path:
- `/Users/manojjoshi/Desktop/Personal/Projects/Product-Development-Workflow`

Important Files:
- `prd.md`: planning flow, planning defect intake, canonical `TODO.md` generation
- `dev.md`: task execution, fix-task handling, story completion gates
- `check.md`: verification flow, failure routing, local/CI gate behavior

Git State:
- Modified but uncommitted: `prd.md`, `dev.md`, `check.md`

## 6. Unresolved Threads & The Wall

Blocked By:
- Nothing external; next work is a doc-logic cleanup decision.

Mental Context:
- The next major task is “point `2`”: remove contradictory restart rules and decide the intended boundary between `/dev` Track A verification and `/check` re-verification.
- Specifically, verify and likely resolve:
  - `Repo Setup` in `/check` pushes before local verification, which conflicts with the “push to CI only after local green” intent.
  - Phase 5 text in `/check` says rerun from failed phase, while Phase 5b rules say rerun from Phase 1.
  - Static-fix row in `/check` says rerun from Phase 2a, which conflicts with the broader “rerun from Phase 1” rule.
  - `/dev` step references for AC/story integration test failures appear wrong and should likely point to Step 8, not Steps 6/7.
- Also, before implementing point `2`, the user explicitly asked to confirm whether `/check` was supposed to avoid rerunning unit/integration tests already covered by `/dev`.
- Current conclusion: `/check` still re-runs affected unit tests and technical integration coverage, so the docs are not yet aligned with a strict “no Track A reruns in `/check`” model.

## 7. Immediate Next Steps (priority order)

1. Re-open point `2` and decide the canonical restart policy:
   - preferred default discussed: rerun `/check` from Phase 1 after any repaired failure, with only a narrow inline Phase 1 exception for immediate static re-checks.
2. Decide whether `/check` should continue running affected unit tests and technical integration coverage, or whether final local verification should be limited to Track B, Track C, and regression-only deltas.
3. Update `check.md` to remove contradictory restart rules and align Repo Setup / CI timing with the chosen policy.
4. Update `dev.md` step references for integration failure recovery if confirmed incorrect.
5. After that, optionally do a final clarity pass on structure if needed, but only after point `2` is settled.

---

# Session Handover: 2026-07-02 17:05 IST

## 1. Executive Summary

Objective: Finish the workflow-doc alignment beyond `/prd`, `/dev`, and `/check`, especially around `/issue`, `/sprint`, incremental planning, and top-level README consistency.
Outcome: Completed for the workflow docs; continuity note recorded here only.
Momentum: High — the command set now has a much more coherent ownership model, and the next session can focus on residual polish instead of foundational restructuring.

## 2. Progress & Completed Work

Files Modified:
- `README.md`
- `issue.md`
- `sprint.md`
- `prd.md`
- `dev.md`
- `check.md`
- `handover.md`

Logic Updated:
- Made `/issue` the classifier for existing-codebase work and explicitly split work into:
  - new feature
  - improvement to existing feature
  - bug fix
- Reworked `/issue-feature` and `/issue-improvement` so they route through architecture/design checks and then `/sprint` incremental mode instead of treating `/sprint` as greenfield-only.
- Clarified that bug fixes should not go directly to bare `/dev`; they must first be attached to the relevant story/planning context and then flow through `/prd` incremental mode.
- Added explicit `greenfield` and `incremental` modes to `/sprint`.
- Formalized `/prd` incremental mode and clarified that Planning Defect Intake is a special case of it.
- Added explicit execution/verification mode language to:
  - `/dev` — normal delivery mode vs verification recovery mode
  - `/check` — normal verification mode vs recovery mode
- Updated `README.md` so the top-level workflow summary matches the new classification and mode model.

Logic Verified:
- Text consistency pass only.
- Created commit `1556196` with message:
  - `docs(workflow): align issue routing and command modes`
- Commit intentionally includes only:
  - `README.md`
  - `issue.md`
  - `sprint.md`
  - `prd.md`
  - `dev.md`
  - `check.md`

## 3. Technical Debt & Debugging Log

Resolved / clarified:
- `/issue` is now the correct classification and routing surface for existing-codebase work.
- `/sprint` is no longer implicitly single-shot; it now has explicit incremental behavior.
- `/prd` incremental behavior is now recognized as a first-class planning mode rather than only a `/check` recovery special case.
- `handover.md` is not part of the repo workflow redesign itself; it remains session-continuity infrastructure for Codex.

Current warnings:
- `handover.md` is modified locally for session continuity and intentionally not included in commit `1556196`.
- No tests were run because all changes in this pass were documentation-only.
- The docs are now structurally aligned, but a future pass may still want to simplify wording or remove any remaining repetitive explanation.

## 4. Architectural & Strategic Decisions

Decision: `/issue` owns work classification for existing-codebase flows.
Rationale: The system needed one surface that decides whether the work is a new feature, improvement, or bug fix before downstream commands are invoked.
Future Impact: Any new existing-codebase entrypoints should either defer to `/issue` or stay strictly internal/non-user-facing.

Decision: `/sprint`, `/prd`, `/dev`, and `/check` all now expose explicit mode language.
Rationale: The workflow had already developed fresh/incremental/recovery behavior in practice, but only parts of the docs admitted it. Making the modes explicit reduces ambiguity.
Future Impact: Future command changes should preserve the same pattern:
- scope-shaping commands get greenfield/incremental language
- execution/verification commands get normal/recovery language

Decision: `handover` is session continuity infrastructure, not part of the product-delivery ownership model.
Rationale: It records and restores context across Codex sessions; it should not be treated as another planner or verifier.
Future Impact: Keep `handover.md` out of workflow-doc commits unless it is intentionally being updated as a separate continuity note.

## 5. Environment & Context

Repo Path:
- `/Users/manojjoshi/Desktop/Personal/Projects/Product-Development-Workflow`

Important Files:
- `issue.md`: classification and routing for existing-codebase work
- `sprint.md`: roadmap generation/update in greenfield or incremental mode
- `prd.md`: epic planning in greenfield or incremental mode
- `dev.md`: execution in normal delivery or verification recovery mode
- `check.md`: verification in normal or recovery mode
- `README.md`: top-level summary of the updated workflow
- `handover.md`: session continuity only, not part of the committed workflow-doc change set

Git State:
- Commit created on `main`: `1556196` — `docs(workflow): align issue routing and command modes`
- `handover.md` remains modified and uncommitted by design

## 6. Unresolved Threads & The Wall

Blocked By:
- Nothing external.

Mental Context:
- The big conceptual cleanup is done.
- The remaining work, if any, is now likely refinement rather than architecture:
  - wording simplification
  - making examples more concrete
  - checking whether any adjacent docs beyond the core set still reference outdated flows
- Be careful not to accidentally fold `handover.md` into a repo commit unless explicitly requested again.

## 7. Immediate Next Steps (priority order)

1. If requested, inspect residual docs for stale examples or contradictory references outside the core set.
2. If the user wants to publish the workflow-doc changes, push commit `1556196` to the remote and create a PR or direct push as requested.
3. Keep `handover.md` as a local continuity note unless the user explicitly asks to commit it.
