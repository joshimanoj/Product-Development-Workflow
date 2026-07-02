# /prd — Generate Detailed Task List for TDD Implementation

## Purpose

For the active epic, read all its stories and break them into atomic 5–10 minute tasks. For every task produce a technical specification with public contracts, file changes, and test plans. Generate functional test specs (Track B) and NFR test specs (Track C) written RED upfront — these accumulate across all stories and run together at epic-level `/check`. Track B and Track C tests are never run during `/dev` — they are written RED and committed, execution is deferred to `/check`.

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

| AC Type | Description | Output |
|---------|-------------|--------|
| Functional — Happy Path | Core success behaviour | Track A + Track B FT |
| Functional — Edge Case | Boundary / error / alternative path | Track A + Track B FT (own test — never bundled with happy path) |
| Functional — Environmental | Network conditions, offline, slow connection, interrupted state | Track A + Track B FT (with environment conditions e.g. Playwright network throttle) |
| Non-Functional — Tool-driven | Automatable NFR (performance, accessibility, security) | Track C (tool from architecture.md Section 8) |
| Non-Functional — Human/UAT | Subjective / human judgment only | UAT-only — document reason, get human confirmation now |

Also identify cross-story Track B tests at this stage — user journeys that span multiple stories (e.g. sign up → verify email → first login). These are additional Track B tests beyond the per-story ones, written here and run at epic CHECK.

Present the full classification table grouped by story, plus cross-story tests, and wait for human confirmation:

```
AC Classification — Epic #N:

Story #1: [Title]
  AC-1 [Functional — Happy Path]: "User sees confirmation on submit" → Track B FT
  AC-2 [Functional — Edge Case]: "Form rejected when email is blank" → Track B FT
  AC-3 [Functional — Environmental]: "Offline state shows cached data" → Track B FT (offline condition)
  AC-4 [Non-Functional — Tool-driven]: "API response < 200ms p95" → Track C (k6)

Story #2: [Title]
  AC-5 [Functional — Happy Path]: "..." → Track B FT
  ...

Cross-story flows:
  Flow 1: Sign up → verify email → first login → Track B FT (spans stories #1, #2, #3)

Confirm?
```

Human must confirm before proceeding to task generation.

---

## Phase 3: Regression Planning

**Before writing any tasks**, analyse the repository to identify what prior work this epic may affect.

### Dependency Analysis

Scan the planned file changes for this epic (derived from story-N.md and architecture.md) and identify:
- Which files each story intends to create or modify
- Which modules/interfaces those files export or implement
- Which prior story and epic test suites import or depend on those modules

### Technical Regression Manifest

List every prior story-level and epic-level technical integration test suite that touches modules this epic plans to change.

```
Technical Regression Manifest — Epic #N:

Planned file changes:
  Story #1: auth/token.ts, auth/session.ts
  Story #2: api/auth-client.ts

Impacted prior suites:
  - Story #3 (Epic #1) integration suite → imports AuthRepository from auth/token.ts
  - Epic #1 integration suite → depends on AuthState from auth/session.ts
  - Story #2 (Epic #2) integration suite → uses ApiAuthClient from api/auth-client.ts

Action: /check will execute these suites as part of technical regression.
```

### Functional Regression Manifest

List every prior Track B functional test that exercises flows touching modules this epic plans to change.

```
Functional Regression Manifest — Epic #N:

Impacted prior Track B tests:
  - FT-3 (Epic #1, Story #2): login flow → depends on auth/session.ts
  - FT-7 (Epic #2, Story #1): token refresh flow → depends on auth/token.ts

Action: /check will execute these tests as part of functional regression.
```

> Note: These manifests are planning-time estimates based on intended file changes. `/check` performs authoritative live reconciliation via `git diff` and may add additional targets not listed here.

Present both manifests to the human for confirmation before proceeding.

---

## Phase 4: Task Breakdown

Three tracks planned upfront across all stories in the epic. All appear in `task_spec_document.md` and `TODO.md`, sectioned by story.

### Story Structure

Each story section in `task_spec_document.md` follows this structure:

```
## Story #N: [Title]

### Public Contracts

[All interfaces, DTOs, events, error types, and state definitions owned by this story.
Defined once here. ACs reference these by name.]

interface [Name] {
  [field]: [type]
  ...
}

type [ErrorType] = [values]

// Events published
event [EventName]: { [payload fields] }

// Events consumed
consumes [EventName] from [source]

---

### AC: [Name] — [Happy Path | Edge Case | Environmental]

References: [ContractName, OtherContract]

#### Track A

**Implementation Tasks**

[Atomic tasks — what to build, not how]

**Unit Test Spec**

Scenario: [description]
  Given: [precondition]
  When: [action]
  Then: [expected result]
  Data: [test inputs]
  Assertions: [what to verify]

**AC Integration Test Spec**

Boundary: [what module/service boundary is being tested]
Precondition: [required state]
Input: [what is passed across the boundary]
Expected outcome: [what should be returned/emitted]
Failure meaning: [what a failure at this boundary indicates]

#### Track B

**Functional Test Spec**

Criterion (from story-N.md): [exact text]
AC type: Functional — [Happy Path | Edge Case | Environmental]
Test File: e2e/[story-name]/[criterion-slug].spec.ts
Framework: [Playwright / Detox / XCUITest / patrol]

[For Environmental ACs — specify condition]:
Environment condition: [e.g. network offline / throttled to 3G / connection interrupted mid-request]
Setup: [how to apply the condition in the test framework]

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

---

### Story Integration Test Spec

[Tests cross-AC boundaries within this story — verifies ACs work together correctly]

Scenario: [description]
  Boundaries tested: [AC-1 output → AC-2 input, etc.]
  Precondition: [state required]
  Flow: [sequence of calls/events across ACs]
  Expected outcome: [end state]
  Failure meaning: [what a failure here indicates about AC interaction]

### Story Track B Cross-AC Test Spec

[Track B tests that verify user flows spanning multiple ACs within this story]

Flow: [description]
  Spans: [AC-1, AC-2, AC-3]
  User steps: [ordered actions]
  Assertions: [end state verification]
  Initial Status: RED
```

---

### Epic Technical Integration Test Suite

Generated once, after all stories are planned. Covers cross-story technical interactions.

```
## Epic #N: Technical Integration Test Suite

Scenario: [description]
  Stories involved: [#1, #2, #3]
  Boundaries tested: [Story #1 output → Story #2 input, etc.]
  Precondition: [state required before suite runs]
  Flow: [sequence of cross-story calls/events]
  Expected outcome: [end state]
  Failure meaning: [what a failure here indicates about story interaction]

[Repeat per cross-story integration scenario]
```

---

### Track A — Implementation Tasks

Each task = one 5–10 minute Red-Green-Refactor unit. No implementation code. No class internals. Specify what to build, not how.

```markdown
### Task N of M: [Brief Description]

Type: Feature | Refactor | Bug Fix

Files:
  Test: [path/to/test/file]
  Implementation: [path/to/implementation/file]

Responsibilities:
  [What this unit is responsible for — single, clear statement]

Public Contract:
  [Reference the interface/DTO/event from the story's Public Contracts section]
  [If this task introduces a new contract element, define it here]

File Changes:
  [File path]: [what changes — add function X, modify interface Y, create module Z]

Test Specification:
  Input: [sample test data]
  Expected output: [what should happen]
  Test description: "should [do X] when [Y]"
  Scenarios: [list additional scenarios if needed]

Architecture Constraints:
  [Rules from architecture.md that apply — naming, layering, patterns, dependencies this task must not introduce]
```

### Track B — Functional Test Tasks

For every functional criterion and edge case AC across all stories in the epic (UI and backend — see classification table above), define one functional test task. Also include cross-story flow tests and story cross-AC tests identified above. Written RED — committed alongside implementation, executed at epic-level `/check`. `/dev` writes them RED and moves on — it does not run them.

**Hard rule: gaps are resolved at /prd time, not discovered at /check time.** Before writing any FT, apply this resolution logic:

- **Testable, maps cleanly to a user-facing behaviour** → write the FT, proceed.
- **Testable but requires infrastructure that does not yet exist** (e.g. emulator, seed helpers, auth test hooks, mock server) → add a Track A prerequisite task for that infrastructure first, then write the FT.
- **Untestable as written** (too vague, purely subjective, depends on a third party) → stop and resolve with the human right now before generating any tasks. Either sharpen the criterion into something measurable (e.g. "feels fast" → "renders within 500ms") or explicitly mark it UAT-only and document why no FT is possible.

Do not write a stub that can never pass. Do not defer gaps to `/check` — a gap found at `/check` that was not declared here is a `/prd` failure and is logged to `HANDOVER.md` as process debt.

Track B tasks are grouped by AC within each story section (see Story Structure above).

### Track C — NFR Test Tasks

For every NFR AC classified as tool-driven across all stories in the epic, define one Track C test task. Written RED alongside Track B — executed at epic-level `/check`. `/dev` writes them RED and moves on — it does not run them.

**Tooling must come from architecture.md Section 8 Track C tooling — do not invent tooling at this stage.** If the required tool is not listed in architecture.md Section 8, stop and ask the human to update architecture.md before proceeding.

Track C tasks are listed flat, grouped by NFR type (not by AC), in their own section after all stories.

```markdown
### NFR Test Task TC-N: [NFR Criterion]

Criterion (from story-N.md): [exact text]
AC type: Non-Functional — Tool-driven

Required Tool: [from architecture.md Section 8 Track C tooling — e.g. k6 / axe-core / Lighthouse CI]
Installation/Setup Requirements: [version, install command, config file location, env vars required]

Test File: [e.g. tests/performance/story-N-load.js]
Test Data: [specific data required — seed records, fixture files, user accounts, API payloads]
Test Setup: [state required before test runs — e.g. "1000 seeded records", "authenticated session", "staging env"]
Execution Command: [exact command to run]
Pass Criteria: [explicit, measurable threshold — e.g. p95 < 200ms | 0 axe violations | Lighthouse score ≥ 90]

Initial Status: RED (expected — implementation not yet written)
```

---

## Phase 5: Completeness Reverse-Trace

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

For each AC referenced in Story Integration Test Spec:
  → Confirm all referenced ACs have Track A tasks
  → If not → add missing tasks

For Epic Technical Integration Test Suite:
  → Confirm all cross-story boundaries reference ACs with Track A tasks in both stories
  → If not → add missing tasks
```

Only present tasks to the user after the reverse-trace is clean. State explicitly: "Reverse-trace complete — all ACs mapped, no missing tasks found." or list what was added.

---

## Phase 6: TODO.md — Task Ordering

One `TODO.md` covers the entire epic, sectioned by story. Track B and Track C tests are listed first within each story section — written RED by `/dev`, executed at epic CHECK. `/dev` works through stories in the order suggested by `epic-N.md` Cross-story Notes.

```markdown
# Epic #N: [Name] | Stories: #[first]–#[last]

## Track B — Functional Tests (written RED per story, run at epic /check)

- [ ] FT-1: [Story #1 — AC-1 Happy Path] — e2e/epic-n/story-1/criterion-1.spec.ts — 8 min
- [ ] FT-2: [Story #1 — AC-2 Edge Case] — e2e/epic-n/story-1/criterion-2.spec.ts — 6 min
- [ ] FT-3: [Story #1 — AC-3 Environmental] — e2e/epic-n/story-1/criterion-3.spec.ts — 7 min
- [ ] FT-4: [Story #1 — Cross-AC flow] — e2e/epic-n/story-1/cross-ac-flow-1.spec.ts — 8 min
- [ ] FT-5: [Story #2 — AC-4 Happy Path] — e2e/epic-n/story-2/criterion-1.spec.ts — 8 min
- [ ] FT-X: [Cross-story flow] — e2e/epic-n/cross-story/flow-1.spec.ts — 10 min

## Track C — NFR Tests (written RED, run at epic /check)

- [ ] TC-1: [NFR criterion] — tests/performance/epic-n-load.js — 5 min

## Story #[N]: [Title]

### Track A — Implementation Tasks

- [ ] Task 1: [Description] — 8 min
- [ ] Task 2: [Description] — 7 min

### Story Integration Test

- [ ] Story #[N] Integration Test Spec — written RED — 5 min

### Story Done When

- [ ] Unit tests passing
- [ ] Story integration tests written RED and committed
- [ ] Type check clean
- [ ] Lint clean
- [ ] Track B tests for this story written RED and committed

## Story #[N+1]: [Title]

### Track A — Implementation Tasks

- [ ] Task 3: [Description] — 8 min

### Story Integration Test

- [ ] Story #[N+1] Integration Test Spec — written RED — 5 min

### Story Done When

- [ ] Unit tests passing
- [ ] Story integration tests written RED and committed
- [ ] Type check clean
- [ ] Lint clean
- [ ] Track B tests for this story written RED and committed

## Epic Acceptance

- [ ] All stories done
- [ ] Epic Technical Integration Test Suite written RED and committed
- [ ] Ready for /check
```

---

## Phase 7: Output Files

### task_spec_document.md

One document per epic, sectioned by story. Contains all three tracks across all stories, plus Story Integration Test Specs, Epic Technical Integration Test Suite, and Regression Manifests. `/dev` reads only the current story's section when implementing.

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
6. **Run the AC Classification Pass (Phase 2)** — read each `story-N.md` in the epic's suggested order. For every AC across all stories, classify into Functional (Happy Path / Edge Case / Environmental) or Non-Functional (Tool-driven / Human/UAT). Also identify cross-story Track B flows. Present the full classification table grouped by story to the human and wait for confirmation. Resolve all UAT-only declarations now.
7. **Run Regression Planning (Phase 3)** — analyse planned file changes per story. Generate Technical Regression Manifest and Functional Regression Manifest. Present both to the human and wait for confirmation.
8. Generate Track B — one FT per functional AC (Happy Path, Edge Case, Environmental) across all stories, plus cross-story flows and story cross-AC flows. Apply the resolution logic in the Track B section. Edge cases and Environmental conditions each get their own FT — never bundled.
9. Generate Track C — one TC per Non-Functional Tool-driven AC. Tool must come from architecture.md Section 8. Track C listed flat by NFR type.
10. Generate Track A — story by story, reading each `story-N.md` individually. For each story: define Public Contracts first, then for each AC define Implementation Tasks, Unit Test Spec, and AC Integration Test Spec. After all ACs: define Story Integration Test Spec and Story Cross-AC Track B spec. After all stories: define Epic Technical Integration Test Suite. No implementation code — specifications only (Responsibilities, Public Contracts, File Changes, Test Specifications, Architecture Constraints).
11. **Run the Completeness Reverse-Trace (Phase 5)** — verify every AC maps to at least one task, every file exists or is being created, every shared dependency has its own task, every Track C tool is in architecture.md, all integration test specs reference valid ACs. Add any missing tasks. State result explicitly.
12. Write `task_spec_document.md` — one document for the epic, sectioned by story. Contains Public Contracts, AC-grouped Track A + Track B tasks, Story Integration Test Specs, Epic Technical Integration Test Suite, Track C section, and both Regression Manifests.
13. For each UI story, generate `story-N-ui-context.md` — pull only matching entries from `Design.md §4`, `UI_Patterns.md`, Component Behavior Guide, and `theme.json`. This is the only design file `/dev` loads for that story.
14. For UI epics, generate `epic-N-uat.md` — the UAT checklist covering all stories and cross-story flows.
15. Write `TODO.md` — one document for the epic, sectioned by story, Track B and Track C listed at top, Track A tasks and Story Integration Test entry per story section, Epic Technical Integration Test Suite entry at epic acceptance.
16. Tell user: "Tasks ready for Epic #N ([N] stories, [M] Track B tests, [P] Track C tests). Run /dev to start Story #[first]."
