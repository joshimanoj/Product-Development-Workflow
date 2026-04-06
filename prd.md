# /prd — Generate Detailed Task List for TDD Implementation

## Purpose

For the active epic, read all its stories and break them into atomic 5–10 minute tasks. For every task produce a technical blueprint with exact file changes and test plans. Generate functional test specs (Track B) and NFR test specs (Track C) written RED upfront — these accumulate across all stories and run together at epic-level `/check`. Track B and Track C tests are never run during `/dev` — they are written RED and committed, execution is deferred to `/check`.

---

## Pre-Flight Check

- **`epic-N.md`** — active epic file (story list, AC summaries, cross-story notes, suggested order)
- **`story-N.md` files** — full story detail for each story in the epic (read one at a time during task generation)
- **`architecture.md`** — architecture patterns, naming conventions, Section 8 Critical Paths + Track C tooling
- **`theme.json`**, **`ui-patterns-[platform].md`**, **`component-behavior-guide.md`** — for UI stories; used to generate `story-N-ui-context.md` files

Show the user a pre-flight summary and wait for confirmation before generating tasks.

```
Branch: feature/epic-N-{slug} (created/confirmed ✓)
Epic #N: [name] | Stories: #[first]–#[last] | Type: UI | Backend | Mixed
Stories: [N total — list titles]
Tech stack: [language / framework / test runner]
~[N] implementation tasks + [M] Track B tests + [P] Track C tests across all stories
Proceed?
```

---

## Phase 1: Tech Stack Detection

Auto-detect from project config files. Claude should apply the same detection logic to any stack encountered and ask the user if it cannot determine the stack confidently.

| Detect | Test Command | E2E Framework | Lint / Type Check |
|--------|-------------|---------------|-------------------|
| package.json + Jest | npm test -- [file] | Playwright (.spec.ts) | npm run lint / tsc --noEmit |
| package.json + Vitest | vitest [file] --run | Playwright (.spec.ts) | npm run lint / tsc --noEmit |
| pytest.ini | pytest [file]::[test] | pytest-playwright | flake8 / mypy |
| go.mod | go test -run [Test] | playwright-go / httptest | go vet / go build |
| Cargo.toml | cargo test [test] | playwright-rust / reqwest | cargo clippy / cargo check |
| pubspec.yaml | flutter test [file] | patrol / integration_test | dart analyze |
| build.gradle | ./gradlew test | Detox / Espresso | ktlint / detekt |
| Xcode project | xcodebuild test | Detox / XCUITest | swiftlint |
| Other / unknown | Ask user before proceeding | Ask user before proceeding | Ask user |

---

## Phase 2: AC Classification Pass

**This pass happens before any tasks are written.** Read `epic-N.md` to understand the full epic scope and cross-story notes. Then read each `story-N.md` in the suggested order from `epic-N.md`. For every acceptance criterion across all stories, classify it into exactly one category. No AC may be left unclassified. If classification is unclear, resolve with the human before proceeding.

| AC Type | UI story | Backend story |
|---------|----------|---------------|
| Functional criterion | Track B FT (one per criterion) | Track A unit/integration test (explicit — not assumed) |
| Edge case | Track B FT (own test — never bundled with happy path) | Track A unit test (own test — never bundled) |
| NFR — automatable | Track C (assign tool from architecture.md Section 8 Track C tooling) | Track C (same — story type is irrelevant for NFRs) |
| NFR — E2E testable (e.g. offline, slow network) | Track B with conditions (e.g. Playwright network throttle) | Track B with conditions |
| NFR — subjective / human judgment only | UAT-only — document reason, get human confirmation now | UAT-only — document reason, get human confirmation now |

Also identify cross-story Track B tests at this stage — user journeys that span multiple stories (e.g. sign up → verify email → first login). These are additional Track B tests beyond the per-story ones, written here and run at epic CHECK.

Present the full classification table grouped by story, plus cross-story tests, and wait for human confirmation:

```
AC Classification — Epic #N:

Story #1: [Title]
  AC-1 [Functional]: "User sees confirmation on submit" → Track B FT
  AC-2 [Edge case]: "Form rejected when email is blank" → Track B FT
  AC-3 [NFR — automatable]: "API response < 200ms p95" → Track C (k6)

Story #2: [Title]
  AC-4 [Functional]: "..." → Track B FT
  ...

Cross-story flows:
  Flow 1: Sign up → verify email → first login → Track B FT (spans stories #1, #2, #3)

Confirm?
```

Human must confirm before proceeding to task generation.

---

## Phase 3: Task Breakdown

Three tracks planned upfront across all stories in the epic. All appear in `task_spec_document.md` and `TODO.md`, sectioned by story.

### Track A — Implementation Tasks

Each task = one 5–10 minute Red-Green-Refactor unit.

```markdown
### Task N of M: [Brief Description]

Type: Feature | Refactor | Bug Fix

Files:
  Test: [path/to/test/file]
  Implementation: [path/to/implementation/file]

What to Build:
  Add/modify: [function/class name]
  Behavior: [what it should do]
  Edge cases: [list]

Test Requirements:
  Input: [sample test data]
  Expected output: [what should happen]
  Test description: "should [do X] when [Y]"

Implementation Notes:
  [Patterns from architecture.md / dependencies / integration points]
```

### Track B — Functional Test Tasks

For every functional criterion and edge case AC across all stories in the epic (UI and backend — see classification table above), define one functional test task. Also include cross-story flow tests identified in the Phase 2 classification pass. Written RED — committed alongside implementation, executed at epic-level `/check`. `/dev` writes them RED and moves on — it does not run them.

**Hard rule: gaps are resolved at /prd time, not discovered at /check time.** Before writing any FT, apply this resolution logic:

- **Testable, maps cleanly to a user-facing behaviour** → write the FT, proceed.
- **Testable but requires infrastructure that does not yet exist** (e.g. emulator, seed helpers, auth test hooks, mock server) → add a Track A prerequisite task for that infrastructure first, then write the FT.
- **Untestable as written** (too vague, purely subjective, depends on a third party) → stop and resolve with the human right now before generating any tasks. Either sharpen the criterion into something measurable (e.g. "feels fast" → "renders within 500ms") or explicitly mark it UAT-only and document why no FT is possible.

Do not write a stub that can never pass. Do not defer gaps to `/check` — a gap found at `/check` that was not declared here is a `/prd` failure and is logged to `HANDOVER.md` as process debt.

```markdown
### Functional Test Task FT-N: [Acceptance Criterion]

Criterion (from story-N.md): [exact text]
AC type: Functional | Edge case
Test File: e2e/[story-name]/[criterion-slug].spec.ts
Framework: [Playwright / Detox / XCUITest / patrol]

User Flow:
1. [Navigate / open screen]
2. [User action]
3. [Assert — element visible, URL, API call, state]

Assertions:
- [Element / URL / API response / state]
- [Error state if applicable]

Component Behavior Tests (UI stories — ref Component Behavior Guide):
- State transitions: Idle → Active → Error
- Snapshot per state (tokens from theme.json applied)
- Layout at small + large screen sizes
- Animation timing via virtual clock (ms + easing from behavior spec)

Initial Status: RED (expected — implementation not yet written)
```

### Track C — NFR Test Tasks

For every NFR AC classified as automatable or E2E-testable across all stories in the epic, define one Track C test task. Written RED alongside Track B — executed at epic-level `/check`. `/dev` writes them RED and moves on — it does not run them.

**Tooling must come from architecture.md Section 8 Track C tooling — do not invent tooling at this stage.** If the required tool is not listed in architecture.md Section 8, stop and ask the human to update architecture.md before proceeding.

```markdown
### NFR Test Task TC-N: [NFR Criterion]

Criterion (from story-N.md): [exact text]
AC type: NFR — automatable | NFR — E2E testable
Tool: [from architecture.md Section 8 Track C tooling — e.g. k6 / axe-core / Lighthouse CI]
Test File: [e.g. tests/performance/story-N-load.js | e2e/story-N/offline.spec.ts]
Run command: [exact command]
Pass condition: [explicit, measurable threshold — e.g. p95 < 200ms | 0 axe violations | Lighthouse score ≥ 90]

Setup required:
[Any Track A prerequisite tasks needed before this test can run — e.g. "seed 1000 records", "configure k6 env vars"]

Initial Status: RED (expected — implementation not yet written)
```

---

## Phase 3: Completeness Reverse-Trace

After generating all tracks, Claude performs an explicit reverse-trace before presenting tasks to the user. This is a mechanical check — not a judgment call.

```
For each AC in story-N.md:
  → Confirm it appears in exactly one row of the Phase 2 classification table
  → Confirm at least one Track A, B, or C task directly enables or tests it
  → If no task maps to an AC → add the missing task before presenting

For each Track A task:
  → Confirm every file it touches either exists or is being created by another Track A task
  → If a file doesn't exist and no task creates it → add a prerequisite Track A task

For each shared dependency touched by multiple Track A tasks:
  → Confirm it has its own dedicated Track A task
  → If not → add one before presenting

For each Track C task:
  → Confirm the tool named is listed in architecture.md Section 8 Track C tooling
  → If not → stop and ask the human to update architecture.md first
```

Only present tasks to the user after the reverse-trace is clean. State explicitly: "Reverse-trace complete — all ACs mapped, no missing tasks found." or list what was added.

---

## Phase 4: TODO.md — Task Ordering

One `TODO.md` covers the entire epic, sectioned by story. Track B and Track C tests are listed first within each story section — written RED by `/dev`, executed at epic CHECK. `/dev` works through stories in the order suggested by `epic-N.md` Cross-story Notes.

```markdown
# Epic #N: [Name] | Stories: #[first]–#[last]

## Track B — Functional Tests (written RED per story, run at epic /check)

- [ ] FT-1: [Story #1 — Functional criterion] — e2e/epic-n/story-1/criterion-1.spec.ts — 8 min
- [ ] FT-2: [Story #1 — Edge case criterion] — e2e/epic-n/story-1/criterion-2.spec.ts — 6 min
- [ ] FT-3: [Story #2 — Functional criterion] — e2e/epic-n/story-2/criterion-1.spec.ts — 8 min
- [ ] FT-X: [Cross-story flow] — e2e/epic-n/cross-story/flow-1.spec.ts — 10 min

## Track C — NFR Tests (written RED, run at epic /check)

- [ ] TC-1: [NFR criterion] — tests/performance/epic-n-load.js — 5 min

## Story #[N]: [Title]

### Track A — Implementation Tasks

- [ ] Task 1: [Description] — 8 min
- [ ] Task 2: [Description] — 7 min

### Story Done When

- [ ] Unit tests passing
- [ ] Type check clean
- [ ] Lint clean
- [ ] Track B tests for this story written RED and committed

## Story #[N+1]: [Title]

### Track A — Implementation Tasks

- [ ] Task 3: [Description] — 8 min

### Story Done When

- [ ] Unit tests passing
- [ ] Type check clean
- [ ] Lint clean
- [ ] Track B tests for this story written RED and committed

## Epic Acceptance

- [ ] All stories done
- [ ] Ready for /check
```

---

## Phase 5: Output Files

### task_spec_document.md

One document per epic, sectioned by story. Contains all three tracks across all stories. `/dev` reads only the current story's section when implementing.

> **Note:** Consumed by `/dev` for task implementation. Archived in `/check` Archive phase — not here. Do not add archive logic to `/prd`.

### story-N-ui-context.md (UI stories only)

Generated by `/prd` per story, in batch, for all UI stories in the epic. Format unchanged. `/dev` loads the context file for the current story — `story-N-ui-context.md` — not the full design documents.

```markdown
# UI Context: Story #N — [Story Name]

Generated by /prd | Source: Design.md v[N], theme.json, UI_Patterns.md, Component Behavior Guide

## 1. Relevant Components
[Source: UI_Patterns.md + Component Behavior Guide]
[Selection: include every component named in story-N.md acceptance criteria or user flow steps]

[Component name]: [full pattern snippet from UI_Patterns.md]
  States: [Idle / Active / Error / Disabled from Component Behavior Guide]
  Animations: [timing + easing]
  Layout rules: [from Component Behavior Guide]

## 2. Relevant User Flows
[Source: Design.md Section 4]
[Selection: include only flows triggered or traversed by this story's user flow steps in story-N.md. Omit all other flows.]

[Flow name]: [steps verbatim from Design.md Section 4 — only steps this story touches]
  Success state: [...]
  Error state: [...]

## 3. Design Tokens (story-relevant only)
[Source: theme.json]
[Selection: for each component in Section 1 and each Track A task, extract every token it references. Omit tokens not referenced by included components or tasks.]

[token.name]: [value] — used by: [Component / Task A-N that references it]

## 4. Interaction Rules
[Source: Design.md Section 7]
[Selection: for each Track A task, include every rule from Section 7 that applies to a component in Section 1. Omit rules for components not present in this story.]

[Rule name]: [rule text from Design.md Section 7] — applies to: [Component name from Section 1]
```

### epic-N-uat.md (UI epics only)

One UAT checklist per epic, covering all UI stories and cross-story flows. Generated by `/prd` at epic planning time when all stories, flows, and components are already loaded. `/uat` loads and presents this file directly — no regeneration cost at test time.

```markdown
# UAT Checklist: Epic #N — [Epic Name]

Generated by /prd | Stories: #[first]–#[last] | Sources: story-N.md files, story-N-ui-context.md files

## Setup

Platform: [Web — open local dev server: http://localhost:3000 | iOS — run on simulator | Android — run on emulator]
Test data: [e.g. use account test@example.com / password: Test1234]
Start at: [e.g. Login screen]

## Story #[N]: [Title]

### Flow 1: [Happy path from story-N.md User Flow]
[ ] 1. [Specific action]
[ ] 2. [What to verify]

### Flow 2: [Edge case / Error state]
[ ] 1. [Action that triggers error]
[ ] 2. [Verify error state]

### Visual Check — Story #[N]
[ ] Colours match design tokens
[ ] Typography correct
[ ] Spacing consistent
[ ] Interactive states work
[ ] Animations smooth

## Story #[N+1]: [Title]
[repeat structure per story]

## Cross-story Flows

### Flow: [e.g. Sign up → verify email → first login]
[ ] 1. [Action in story #N]
[ ] 2. [Transition to story #N+1 behaviour]
[ ] 3. [Final assertion]

## Responsive / Device Check (epic-wide)
[ ] [Web: 375px mobile width — all epic screens intact]
[ ] [Web: 1280px desktop — all epic screens intact]
[ ] [Mobile: small device — no content cut off across any screen]

## Sign-off

Reply with: ✅ APPROVED — all items checked, ready to merge | ❌ CHANGES NEEDED — [story #N: describe what needs fixing]
```

---

## Claude Instructions for /prd

0. **Branch check — must be the very first action, before reading any other file:**
   - **Step 0a:** Run `git branch --show-current` AND read `Sprint.md` in parallel.
   - **Step 0b:** Derive the expected branch name from the active epic:
     - `feature/epic-N-{slug}` (e.g. `feature/epic-1-account-creation`)
     - Slug = epic title lowercased, spaces → hyphens, special chars stripped
   - **Step 0c:** If already on the correct branch → proceed. If on any other branch: run `git checkout main && git pull && git checkout -b feature/epic-N-{slug}` and tell the user: "Switched to feature/epic-N-{slug} off main."
   - Never create the branch silently — always report the branch name to the user.
   - **Only after the branch is confirmed/created**, proceed to read epic and story files.

1. Read `epic-N.md` — story list, AC summaries, cross-story notes, suggested story order. This is the planning context for the whole epic. (`Sprint.md` was already read in step 0a — no need to re-read.)
2. Read `architecture.md` (patterns, naming conventions, tech stack, Section 8 Critical Paths + Track C tooling).
3. Detect tech stack from config files. Ask user if uncertain.
4. Verify `.github/workflows/ci.yml` exists. If missing: stop and tell the user "CI file not found — return to Sprint 0 to set it up before running /prd." If present: proceed — no changes needed.
5. Show pre-flight summary and wait for confirmation.
6. **Run the AC Classification Pass (Phase 2)** — read each `story-N.md` in the epic's suggested order. For every AC across all stories, classify into Functional / Edge case / NFR-automatable / NFR-E2E-testable / UAT-only. Also identify cross-story Track B flows. Present the full classification table grouped by story to the human and wait for confirmation. Resolve all UAT-only declarations now.
7. Generate Track B — one FT per functional criterion and edge case AC across all stories, plus cross-story flows. Apply the resolution logic in the Track B section. Edge cases get their own FT — never bundled.
8. Generate Track C — one TC per NFR AC classified as automatable or E2E-testable. Tool must come from architecture.md Section 8.
9. Generate Track A — story by story, reading each `story-N.md` individually. For UI stories, the blueprint Zones tables are embedded in `story-N.md` Design References — use these as the authoritative screen-level specification. Extract component names, states, and token references from the embedded Zones tables as the filter for reading `ui-patterns-[platform].md`, `component-behavior-guide.md`, and `theme.json`.
10. **Run the Completeness Reverse-Trace (Phase 3)** — verify every AC maps to at least one task, every file exists or is being created, every shared dependency has its own task, every Track C tool is in architecture.md. Add any missing tasks. State result explicitly.
11. Write `task_spec_document.md` — one document for the epic, sectioned by story, containing all three tracks.
12. For each UI story, generate `story-N-ui-context.md` — pull only matching entries from `Design.md §4`, `UI_Patterns.md`, Component Behavior Guide, and `theme.json`. This is the only design file `/dev` loads for that story.
13. For UI epics, generate `epic-N-uat.md` — the UAT checklist covering all stories and cross-story flows.
14. Write `TODO.md` — one document for the epic, sectioned by story, Track B and Track C listed at top, Track A tasks per story section.
15. Tell user: "Tasks ready for Epic #N ([N] stories, [M] Track B tests, [P] Track C tests). Run /dev to start Story #[first]."
