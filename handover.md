# /handover — Session Continuity

## Purpose

Claude's context window is finite. The handover system ensures zero knowledge loss between sessions. The primary mechanism is a lightweight checkpoint appended to `HANDOVER.md` automatically at the end of every successful `/check` pass — when state is cleanest and most complete. The pre-compaction hook acts as a safety net only, firing if no checkpoint has been written yet in the current session.

---

## Session Start Rule (Warm-Start)

At the start of every session, Claude performs a warm-start in five steps:

1. **Check for `.claude/state.json` first.** This is the precise mechanical state — story number, last completed task, FT statuses, next command. If present, read it and hold in memory.
2. **Check for `HANDOVER.md`.** Read only the most recent checkpoint block (the last `## Checkpoint` entry) for narrative context — decisions made, debt carried forward. If `HANDOVER.md` is missing or empty and `state.json` is also missing, refuse to proceed: warn "No session state found — context from your previous session may be lost. Confirm current story and task manually before continuing." Do not proceed silently.
3. **If `state.json` is missing but `HANDOVER.md` exists**, warn: "`state.json` not found — task-level state may be lost. Reconstructing from HANDOVER.md checkpoint. Verify current task before running /dev." Do not silently assume state.
4. **Set active context in memory** — active story number and name, next command, last completed task, FT statuses, warnings and debt. Do not re-read `Sprint.md`, `architecture.md`, or any other project file unless the human explicitly asks.
5. **Greet the human with a one-line status summary and a ready prompt** — for example: "Last checkpoint: Story #3 Sign-up flow — Task 4 complete, FT-1 green, FT-2 red. Ready for /dev 5. Say 'continue' to proceed, or ask me anything."

---

## Primary Mechanism — Two-Layer State

### Layer 1 — `.claude/state.json` (written per `/dev` task)

After every `/dev` task commit, Claude writes `.claude/state.json`. This is the precise mechanical state — updated at task granularity so a mid-story session crash loses at most one task. Claude writes this directly, no script or subprocess.

```json
{
  "story": 3,
  "story_name": "Child Profile Setup",
  "branch": "feature/story-3-child-profile-web",
  "last_completed_task": 4,
  "next_task": 5,
  "track_b_status": {"FT-1": "green", "FT-2": "red", "FT-3": "red"},
  "next_command": "/dev 5",
  "timestamp": "2026-03-20T14:32:00"
}
```

### Layer 2 — `HANDOVER.md` Checkpoint (written per `/check` pass)

After every successful `/check` pass, Claude appends a checkpoint block directly to `HANDOVER.md`. No script, no subprocess — Claude writes from its own current context. This carries narrative context that `state.json` does not: decisions made, debt deferred, architectural notes.

```markdown
---

## Checkpoint: Story #X [name] | [YYYY-MM-DD HH:MM] | /check PASSED

Story complete: [one sentence — what was built]

Files changed: [list]

Decisions made: [read story-change-log.md and extract every entry for Story #X — copy Deviation type, What happened, and Impact fields verbatim. If no entries exist for this story, write "none."]

Warnings / debt: [linting warnings deferred, known issues, anything not to repeat]

Next: [next story number and name] | Ready for: /uat (UI) | /prd (Backend)
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
**Layer 2:** Claude appends a checkpoint block to `HANDOVER.md` at the end of every successful `/check` pass. No script, no subprocess.

The PreCompact hook in `.claude/settings.local.json` acts as a fallback only: it checks whether `HANDOVER.md` was updated this session and spawns a subprocess only if no checkpoint exists yet. Under normal workflow it does nothing — `/check` already appended a checkpoint and `/dev` already updated `state.json`.

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
    # Exit immediately if /check already wrote a checkpoint this session
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
