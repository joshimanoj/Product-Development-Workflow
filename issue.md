# /issue — Create and Develop an Issue in an Existing Codebase

## Purpose

The `/issue` commands are the entry point for all user-facing work in an existing codebase built with this workflow. `/issue` is responsible for classifying the work first, then routing it into the correct downstream planning and delivery path. Claude owns the full execution chain from that point, including artifact checks, pre-flight gates, and downstream command invocations. The human's only inputs after the initial command are the approval gates already defined in those downstream commands.

Pure internal implementation tasks sit outside this classifier and should be handled directly inside the active planning and execution flow rather than introduced through `/issue`.

---

## Command Format

```
/issue-bugfix      "description"
/issue-improvement "description"
/issue-feature     "description"
```

The command name is the human's initial signal, but `/issue` still performs a classification pass and surfaces a mismatch if the description conflicts with the declared type.

---

## Classification First

Before entering any type-specific flow, `/issue` must classify the work into one of these three delivery shapes:

1. `new feature`
2. `improvement to existing feature`
3. `bug fix`

Claude proposes the classification using the rules below, then asks for confirmation only if the case is ambiguous or conflicts with the invoked command.

| Classification | Meaning | Routing Shape |
|------|-------------------|--------------|
| New feature | A new user capability, workflow, or outcome the product could not do before | architecture/design check → `/sprint` incremental → `/prd` → `/dev` → `/check` → `/uat` if UI |
| Improvement to existing feature | Existing capability remains the same in purpose, but UX, content, speed, behavior, or scope improves | architecture/design check → usually `/sprint` incremental → `/prd` → `/dev` → `/check` → `/uat` if UI |
| Bug fix | Restores intended behavior that already exists in story/design/spec history | locate affected story/spec → reopen planning context → `/prd` incremental → `/dev` → `/check` → `/uat` if UI |

### Classification Rules

- If the request introduces a capability the product could not previously perform → `new feature`
- If the request changes or improves an existing capability without changing its core purpose → `improvement to existing feature`
- If the request restores expected behavior that was already intended and documented → `bug fix`

`/dev` is not the classification surface for user-facing bug fixes. `/issue` must attach every bug fix to the appropriate story and planning context before handing work to `/dev`.

---

## Command Declarations

| Command | Typical classification | User-facing change? | Needs story? | Entry point after classification |
|------|-------------------|--------------|-------------|
| `/issue-bugfix` | Bug fix | Usually yes | Yes — reopen existing story | `/prd` incremental |
| `/issue-improvement` | Improvement to existing feature | Yes | Yes — existing story or new story in existing roadmap | usually `/sprint` incremental, then `/prd` |
| `/issue-feature` | New feature | Yes | Yes — new epics/stories in existing roadmap | `/sprint` incremental, then `/prd` |

**Mismatch Detection:** If the description implies a different classification than the invoked command, Claude flags it before proceeding: "You've invoked [command], but this looks like [classification]. Confirm before I continue."

---

## Pre-flight: All Issue Types

Before any type-specific steps, Claude always:

1. Reads `HANDOVER.md` — most recent checkpoint only — to establish current project state.
2. Reads `architecture-dev-summary.md` — stack, naming conventions, file structure, test commands.
3. Reads `Sprint.md` — current story index and active sprint status.

---

## /issue-bugfix

**Definition:** Restores behaviour that was intended but is broken. The bug must relate to an existing story. A bug that requires a new UI element or new scope is **not** a bugfix — see Mismatch Detection.

### Execution Chain

```
1. Pre-flight (HANDOVER.md + architecture-dev-summary.md + Sprint.md)
2. Identify affected story-N.md
3. Propose bug type → human confirms
4. Architecture impact check
5. [If UI] Design spec currency check
6. Update story-N.md AC in place + log to bugs.md + reopen in Sprint.md
7. /prd (incremental) → /dev → /check → [if UI] /uat
```

### Step-by-Step

**Step 1 — Identify affected story:**
Claude reads the description and matches it to the most likely `story-N.md` based on the feature area. Presents: "This appears to relate to Story #N: [name]. Confirm?"

If the human disagrees, Claude asks which story it belongs to before proceeding.

**Step 2 — Propose bug type:**
Claude reads the affected `story-N.md` and the description, then proposes:

```
Bug type: UI | Logic/Backend | Combination
Affected AC: [which acceptance criterion is failing]
Proposed fix scope: [one sentence]
Confirm?
```

Human confirms or corrects. Claude does not proceed until confirmed.

**Step 3 — Architecture impact check:**
Does this bug reveal a gap or wrong decision in `architecture.md`?
- If yes → update `architecture.md` + regenerate `architecture-dev-summary.md`. Tell the user what changed.
- If no → proceed.

**Step 4 — Design spec check (UI or Combination bugs only):**
Are `Design.md` and design specs current with the actual codebase?
- If yes → proceed.
- If drift detected → flag to human: "Design specs appear out of sync with [area]. Resolve before /prd?" Human confirms before proceeding.

**Step 5 — Update artifacts:**

*Update `story-N.md` AC in place:*
Rewrite the failing acceptance criterion to capture both the original intent and the correct behaviour. Do not append — replace the criterion so the story reflects its true definition of done going forward.

Add a `## Bug History` section at the bottom of `story-N.md` if one does not exist:
```markdown
## Bug History

### Bug #[N] — [YYYY-MM-DD]
Description: [what failed]
AC rewritten: [which criterion was updated]
bugs.md ref: Bug #[N]
```

*Log to `bugs.md`:*
```markdown
---

## Bug #N: [description] | [YYYY-MM-DD] | Story #X

Status: Open
Type: UI | Logic/Backend | Combination
Description: [what failed and in what context]
AC rewritten: [exact criterion that was updated in story-N.md]
Fix committed: [filled in after /check passes]
Resolved: [filled in after /check passes]
```

*Update `Sprint.md`:*
Change the story's status from `✅` (or current status) to `🔄 Reopened — Bug #N`.

**Step 6 — Execute:**
Run `/prd` in incremental mode → `/dev` → `/check` → `/uat` if UI or Combination type.

**Step 7 — Close the bug:**
After `/check` passes (and `/uat` if UI), update `bugs.md`:
- `Fix committed: [commit ref]`
- `Resolved: [YYYY-MM-DD]`
- `Status: Resolved`

Update `Sprint.md` story status back to `✅`.

---

## /issue-improvement

**Definition:** Changes something the user experiences — a label, error message, interaction, visual, or performance characteristic. The product already does the thing; this makes it better. Does **not** introduce new capabilities.

If a new UI element or interaction pattern is required that does not exist in `Design.md`, this is not an improvement — it is a feature. See Mismatch Detection.

### Execution Chain

```
1. Pre-flight (HANDOVER.md + architecture-dev-summary.md + Sprint.md)
2. Identify affected story-N.md (or confirm new story needed)
3. Confirm UI or Backend
4. Architecture impact check
5. [If UI] Design spec currency check + new pattern check
6. Update product/design scope only as needed, then run `/sprint` in incremental mode to update the affected epic/story set
7. `/prd` (greenfield for new epic, incremental for changed existing epic) → `/dev` → `/check` → [if UI] `/uat`
```

### Step-by-Step

**Step 1 — Identify affected scope:**
Claude matches the improvement to the most likely existing feature, epic, and story set. Presents either:
- "This improvement relates to Story #N: [name]. I'll update that story path."
- "This improvement extends Feature/Epic [name] and needs one or more new stories in the existing roadmap."

If the improvement spans multiple stories or is sufficiently distinct, Claude expands the scope through `/sprint` incremental mode instead of directly hand-authoring only one story file.

**Step 2 — Confirm type:**
Claude states: "This improvement affects [UI / Backend / both]. Confirm?"

**Step 3 — Architecture impact check:**
Does this improvement require changes to data models, API contracts, or architectural patterns?
- If yes → update `architecture.md` + regenerate `architecture-dev-summary.md` first.
- If no → proceed.

**Step 4 — Design spec check (UI improvements only):**

*Currency check:* Are `Design.md` and design specs current?
- If drift → flag and resolve with human before proceeding.

*New pattern check:* Does this improvement require a UI element or interaction that doesn't exist in `Design.md`?
- If follows existing pattern → proceed.
- If new pattern needed → **stop**. This is feature scope. Tell the human: "This improvement requires a new UI pattern not in Design.md. Treat as /issue-feature instead, or descope to fit existing patterns."

**Step 5 — Update planning inputs and route through `/sprint`:**
Update the relevant product/design context first:
- If the improvement changes feature-level scope or language, update `product_note.md`
- If the improvement changes screens, flows, or interactions, update `Design.md` and related design outputs first

Then run `/sprint` in incremental mode so `Sprint.md`, `epic-N.md`, and `story-N.md` stay consistent.

If the improvement only sharpens an existing story without changing epic/story structure, `/sprint` incremental may update just that story and its parent epic metadata.

When an existing story is updated in place, add or update the story's `## Improvement History` section:
```markdown
## Improvement History

### Improvement — [YYYY-MM-DD]
Description: [what was improved]
AC rewritten: [which criterion was updated]
```

**Step 6 — Execute:**
Run `/prd` using the scope produced by `/sprint` incremental → `/dev` → `/check` → `/uat` if UI.

---

## /issue-feature

**Definition:** A new capability not previously in scope. Extends `product_note.md`, assumes an existing approved design baseline is already present, re-checks architecture and design impact as needed, and then goes through `/sprint` in incremental mode to add properly structured epics and stories into the existing roadmap.

### Execution Chain

```
1. Pre-flight (HANDOVER.md + architecture-dev-summary.md + Sprint.md)
2. Update product_note.md
3. Architecture impact assessment → update architecture.md if needed
4. [If UI] Design impact assessment → update Design.md + specs if needed
5. `/sprint` incremental → generates only the new or changed epic/story files
6. Per story: `/prd` → `/dev` → `/check` → [if UI] `/uat`
```

### Step-by-Step

**Step 1 — Update `product_note.md`:**
Claude updates the relevant sections directly — no full `/product` conversation. Specifically:
- Add the new feature to Section 4 (Features) using the standard feature template (Description, User value, Key interactions, Priority).
- Update Section 5 (Out of Scope) if this feature was previously listed there — remove it and note why it moved in scope.
- Update Section 8 (Open Questions) if the feature raises unresolved decisions.
- Update Section 9 (Risks) if relevant.

Claude presents the changes and asks: "product_note.md updated. Confirm before proceeding to architecture assessment?"

**Step 2 — Architecture impact assessment:**
Claude reads current `architecture.md` against the new feature and asks targeted questions only where the feature creates genuine uncertainty:
- Does it require new data models or changes to existing ones?
- Does it require new API endpoints or changes to existing contracts?
- Does it require new third-party dependencies?
- Does it change infrastructure or hosting requirements?

If changes needed → update `architecture.md` collaboratively, human approves, regenerate `architecture-dev-summary.md`.
If no changes needed → state explicitly: "No architecture changes required. Proceeding."

**Step 3 — Design impact assessment (if UI):**
- Does the feature fit within existing Design.md patterns and components?
  - If yes → proceed to `/sprint` incremental.
  - If new screens or patterns needed → run mini design loop: update `Design.md`, update relevant design specs, prototype if new patterns are significant. Human approves before proceeding.

**Step 4 — Run `/sprint`:**
`/sprint` runs in incremental mode. It reads the updated `product_note.md`, `architecture.md`, and existing roadmap files. It generates only the new or changed epics/stories for this feature — it does not regenerate unrelated existing stories. New `story-N.md` files are added and `Sprint.md` is updated in place with the new entries.

**Step 5 — Execute per story:**
For each new story: `/prd` → `/dev` → `/check` → `/uat` if UI.

---

## Approval Gates Summary

These are the only points where the human must respond. Claude drives everything else automatically.

| Gate | Applies to | What human decides |
|------|-----------|-------------------|
| Bug type confirmation | bugfix | Confirms type and affected AC |
| Affected story confirmation | bugfix, improvement | Confirms which story-N.md is affected |
| Architecture update approval | all types (if triggered) | Approves changes to architecture.md |
| Design update approval | improvement, feature (if triggered) | Approves changes to Design.md + specs |
| product_note.md update approval | feature, improvement (if triggered) | Confirms changed feature-level scope |
| `/sprint` scope confirmation | improvement, feature | Confirms the incremental epic/story changes before `/prd` |
| /prd pre-flight | bugfix, improvement, feature | Confirms task breakdown before /dev |
| /uat sign-off | UI bugfix, UI improvement, UI feature | Approves look, feel, and interaction |

---

## bugs.md

Initialized at Sprint 0 as an empty file. Never deleted. Append-only — one block per bug, in order of discovery.

```markdown
# bugs.md — Bug Log

Initialized: [YYYY-MM-DD]
Format: one block per bug, append only, never delete entries.

---

## Bug #N: [short description] | [YYYY-MM-DD] | Story #X

Status: Open | Resolved
Type: UI | Logic/Backend | Combination
Description: [what failed and in what context]
AC rewritten: [exact criterion updated in story-N.md]
Fix committed: [commit ref — filled in after /check passes]
Resolved: [YYYY-MM-DD — filled in after /check passes]
```

---

## story-change-log.md

A single project-scoped log that records every deviation from the original story definition — bugs, improvements, and scope changes — across all stories. Append-only. Referenced by `HANDOVER.md` checkpoint generation.

Initialized at Sprint 0 as an empty file alongside `bugs.md`.

```markdown
# story-change-log.md — Story Change Log

Initialized: [YYYY-MM-DD]
Format: one block per change, append only, never delete entries.

---

## Change #N | Story #X: [story name] | [YYYY-MM-DD]

Type: Bug fix | Improvement | Scope change
Triggered by: /issue-bugfix | /issue-improvement | /issue-feature
Description: [what changed and why]
AC affected: [which acceptance criteria were rewritten]
bugs.md ref: Bug #[N] (bug fixes only)
```
