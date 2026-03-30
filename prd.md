# /prd — Generate Detailed Task List for TDD Implementation

## Purpose

For the active story, break it into atomic 5–10 minute tasks. For every task produce a technical blueprint with exact file changes and test plans. Critically, also generate functional test specs (Track B) that are written first in RED, committed alongside the implementation, and must be GREEN before `/check` can proceed.

---

## Pre-Flight Check

- **`story-N.md`** — active story file (full acceptance criteria, user flows, story type, and embedded blueprint Zones tables in Design References)
- **`architecture.md`** — architecture patterns, naming conventions
- **`theme.json`**, **`ui-patterns-[platform].md`**, **`component-behavior-guide.md`** — for UI stories; used to generate `story-N-ui-context.md`. The blueprint content for this story is already embedded in `story-N.md` Design References — do not re-read the full blueprint file.

Show the user a pre-flight summary and wait for confirmation before generating tasks.

```
Branch: feature/story-N-{slug}-{pass} (created/confirmed ✓)
Story #X: [name] | Type: UI | Backend
Will modify: [files] | Will create: [files]
Tech stack: [language / framework / test runner]
~[N] implementation tasks + [M] functional test tasks
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

## Phase 2: Task Breakdown

Two tracks planned upfront. Both appear in `task_spec_document.md` and `TODO.md`.

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

### Track B — Functional Test Tasks (UI stories only)

For every acceptance criterion in `story-N.md`, define one functional test task. Written FIRST in RED before any implementation. Committed alongside feature code. Must be GREEN before `/check` proceeds.

**Hard rule: gaps are resolved at /prd time, not discovered at /check time.** Before writing any FT, read each acceptance criterion from `story-N.md` and apply this resolution logic:

- **Testable, maps cleanly to a user-facing behaviour** → write the FT, proceed.
- **Testable but requires infrastructure that does not yet exist** (e.g. emulator, seed helpers, auth test hooks, mock server) → add a Track A prerequisite task for that infrastructure first, then write the FT.
- **Untestable as written** (too vague, purely subjective, depends on a third party) → stop and resolve with the human right now before generating any tasks. Either sharpen the criterion into something measurable (e.g. "feels fast" → "renders within 500ms") or explicitly mark it UAT-only and document why no FT is possible.

Do not write a stub that can never pass. Do not defer gaps to `/check` — a gap found at `/check` that was not declared here is a `/prd` failure and is logged to `HANDOVER.md` as process debt.

```markdown
### Functional Test Task FT-N: [Acceptance Criterion]

Criterion (from story-N.md): [exact text]
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

---

## Phase 3: TODO.md — Task Ordering

Functional tests (FT-N) are written FIRST and start RED. They go GREEN naturally as implementation tasks complete. No story is ready for `/check` until all FT tasks are also GREEN.

```markdown
# Story #X: [Name] | Type: UI | Backend

## Track B — Functional Tests (write first, start RED)

- [ ] FT-1: [Criterion 1] — e2e/story-x/criterion-1.spec.ts — 8 min
- [ ] FT-2: [Criterion 2] — e2e/story-x/criterion-2.spec.ts — 6 min

## Track A — Implementation Tasks

- [ ] Task 1: [Description] — 8 min
- [ ] Task 2: [Description] — 7 min

## Integration Check

- [ ] Full unit test suite passing
- [ ] All FT tasks GREEN (UI stories only)
- [ ] All acceptance criteria verified

## Story Acceptance

- [ ] Ready for /check
```

---

## Phase 4: Output Files

### task_spec_document.md

> **Note:** Consumed by `/dev` for task implementation. Archived in `/check` Phase 7 (Archive Before Next Story) — not here. Do not add archive logic to `/prd`.

### story-N-ui-context.md (UI stories only)

Generated by `/prd` for UI stories only. Contains only the design context relevant to this specific story, extracted from `Design.md`, `theme.json`, `UI_Patterns.md`, and the Component Behavior Guide. `/dev` loads this file instead of the full design documents, keeping context lean.

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

### story-N-uat.md (UI stories only)

Pre-generated by `/prd` for UI stories only, using the UAT checklist format defined in the `/uat` section. Written at story-planning time when `Design.md`, `story-N.md` acceptance criteria, and user flows are already loaded. `/uat` loads and presents this file directly — no regeneration cost at test time. If the file is missing when `/uat` runs, `/uat` regenerates it and logs a warning.

```markdown
# UAT Checklist: Story #N — [Story Name]

Generated by /prd | Sources: story-N.md (acceptance criteria, user flows), story-N-ui-context.md (components, tokens)

## Setup

Platform: [Web — open local dev server: http://localhost:3000 | iOS — run on simulator | Android — run on emulator]
Test data: [e.g. use account test@example.com / password: Test1234]
Start at: [e.g. Login screen / Home tab]

## Checklist

### Flow 1: [Happy path from story-N.md User Flow]
[ ] 1. [Specific action]
[ ] 2. [What to verify]
[ ] 3. [Next action]

### Flow 2: [Edge case / Error state]
[ ] 1. [Action that triggers error]
[ ] 2. [Verify error state]

### Visual Check
[ ] Colours match design tokens (from story-N-ui-context.md Section 3)
[ ] Typography correct (font, size, weight)
[ ] Spacing consistent
[ ] Interactive states work
[ ] Animations smooth

### Responsive / Device Check
[ ] [Web: 375px mobile width — layout intact]
[ ] [Web: 1280px desktop — layout intact]
[ ] [Mobile: small device — no content cut off]

### Edge Cases
[ ] [Story-specific edge case from story-N.md acceptance criteria]

## Sign-off

Reply with: ✅ APPROVED — all items checked, ready to merge | ❌ CHANGES NEEDED — [describe what needs fixing]
```

---

## Claude Instructions for /prd

0. **Branch check — must be the very first action, before reading any other file:**
   - **Step 0a:** Run `git branch --show-current` AND read `Sprint.md` in parallel — these two reads are safe to parallelise and give you the current branch name plus the active story number and title.
   - **Step 0b:** Derive the expected branch name from the active story:
     - Pass 1 (web): `feature/story-N-{slug}-web` (e.g. `feature/story-3-child-profile-web`)
     - Pass 2 (native): `feature/story-N-{slug}-native`
     - Slug = story title lowercased, spaces → hyphens, special chars stripped (e.g. "Child Profile Setup" → `child-profile-setup`)
   - **Step 0c:** If already on the correct branch → proceed. If on any other branch (e.g. a previous story's branch or `main`): run `git checkout main && git pull && git checkout -b feature/story-N-{slug}-{pass}` and tell the user: "Switched to feature/story-N-{slug}-{pass} off main."
   - Never create the branch silently — always report the branch name to the user.
   - **Only after the branch is confirmed/created**, proceed to read story-N.md, architecture.md, and design files.

1. Read `story-N.md` — active story file, acceptance criteria, user flows, story type (UI / Backend), and Design References section. (`Sprint.md` was already read in step 0a — no need to re-read.)
2. Read `architecture.md` (patterns, naming conventions, tech stack). For UI stories, the blueprint Zones tables for this story's screens are already embedded in `story-N.md` Design References — use these as the authoritative screen-level specification. Do not re-read the full `blueprint-[platform].md` file. Extract the component names, states, and token references from the embedded Zones tables and use them as the filter for reading `ui-patterns-[platform].md`, `component-behavior-guide.md`, and `theme.json` — load only entries referenced in the embedded blueprint content. Do not pass these files to `/dev` directly — use them only to generate `story-N-ui-context.md`.
3. Detect tech stack from config files. Ask user if uncertain.
4. Verify `.github/workflows/ci.yml` exists. If missing: stop and tell the user "CI file not found — return to Sprint 0 to set it up before running /prd." If present: proceed — no changes needed.
5. Show pre-flight summary and wait for confirmation.
6. Generate Track B first (UI stories only) — for each acceptance criterion in `story-N.md`, apply the resolution logic in the Track B section above before writing the FT. Resolve all untestable criteria with the human before proceeding to task generation. No gaps may be deferred past this step.
7. Generate Track A — atomic implementation tasks, 5–10 min each.
8. Write `task_spec_document.md` with both tracks. For UI stories, also generate `story-N-ui-context.md` — using the component, flow, and screen filter list extracted in step 2, pull only the matching entries from `Design.md §4`, `UI_Patterns.md`, the Component Behavior Guide, and `theme.json`. Omit everything not referenced by this story. This is the only design file `/dev` will load. For UI stories, also generate `story-N-uat.md` — the pre-built UAT checklist for this story using the format defined in `/uat`. `/uat` will load and present this file directly rather than regenerating it.
9. Write `TODO.md` — FT tasks first, then implementation tasks.
10. Tell user: "Tasks ready. Run /dev to start FT-1." (UI) or "Run /dev to start Task 1." (Backend).
