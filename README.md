# Product Development Workflow

A complete, opinionated product development workflow built as [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash commands (skills). It takes a project from idea through to deployed, tested code — with built-in quality gates at every stage.

## Workflow

```
/product  →  /design  →  /architecture  →  /sprint  →  /prd  →  /dev  →  /check  →  /uat
                                                                    ↑                   |
                                                                    └───────────────────┘
                                                                      (if changes needed)
```

**Supporting commands** that plug in at any point:

- `/issue` — Entry point for bugs, improvements, and new work in existing codebases
- `/spike` — Time-boxed technical investigation when development gets stuck
- `/handover` — Session continuity between Claude Code conversations

## Commands

| Command | Purpose |
|---|---|
| `/product` | Kick off product definition. Produces `product_note.md` — the foundation document. |
| `/design` | Living UX & interaction design. Iterative prototype reviews producing a design output suite (style guide, theme, blueprints, component specs). |
| `/architecture` | Technical foundation. Platform, stack, data models, APIs, infrastructure, testing strategy. |
| `/sprint` | Transform `product_note.md` into a sprint roadmap with atomic user stories (`story-N.md`). Supports greenfield and incremental roadmap updates. |
| `/prd` | Planning only. Breaks stories into atomic implementation tasks, Track B/Track C specs, verification mapping, regression manifests, and critical-path execution plans. |
| `/dev` | Implementation only. Executes Track A, proves technical correctness with unit and technical integration tests, and writes Track B/Track C tests RED. Supports normal delivery and verification-recovery execution. |
| `/check` | Verification gate. Runs final lint/type-check, current-epic Track B/Track C, acceptance audit, planned regression, planned critical paths, and CI. Gates `/uat` or merge. Supports normal verification and recovery routing. |
| `/uat` | Human acceptance testing for UI stories. Manual sign-off on look, feel, and interaction before merge. |
| `/issue` | Entry point for all new work in existing codebases — classifies as task, bug, improvement, or feature and routes to the correct workflow. |
| `/spike` | Time-boxed (2-4 hr) technical investigation. Triggered by the 3-strike rule or architectural unknowns. Produces a decision record. |
| `/handover` | Session continuity. Maintains `state.json` and `HANDOVER.md` checkpoints so a new Claude session can warm-start. |

### CI Configuration

`ci.yml` is a stack-aware GitHub Actions workflow that auto-activates jobs based on detected config files (Node, Python, Go, Rust, Flutter, Kotlin, iOS). Committed once at Sprint 0.

## Setup

1. Clone this repo
2. Copy the `.md` command files into your Claude Code skills directory (e.g., `~/.claude/commands/` or your project's `.claude/commands/`)
3. Copy `ci.yml` to `.github/workflows/ci.yml` in your project when you reach Sprint 0

## Workflow Details

### New Project (Greenfield)

1. **`/product`** — Define the product vision, users, features, constraints, and risks
2. **`/design`** — Design the UX through iterative prototype review cycles
3. **`/architecture`** — Lock in technical decisions (stack, data models, APIs, infra)
4. **`/sprint`** — Plan sprints and generate user stories from product features
5. **`/prd`** — Plan implementation and verification for the epic
6. **`/dev`** — Implement Track A and prove technical correctness
7. **`/check`** — Verify requirements and regressions, then push to CI
8. **`/uat`** — Human sign-off for UI stories, then merge

### Existing Codebase

Use **`/issue`** as the entry point. It classifies the work and routes to the appropriate workflow:
- **Task** (no user-facing change) → `/dev` directly
- **Bug fix** → attach to existing story/spec → `/prd` incremental → `/dev` → `/check` → `/uat` if UI
- **Improvement** → architecture/design check as needed → usually `/sprint` incremental → `/prd` → `/dev` → `/check` → `/uat` if UI
- **Feature** → architecture/design check as needed → `/sprint` incremental → `/prd` → `/dev` → `/check` → `/uat` if UI

### When You're Stuck

**`/spike`** runs a time-boxed investigation (2-4 hours) and produces a permanent decision record appended to `architecture.md`.

### Between Sessions

**`/handover`** ensures continuity. It writes checkpoints automatically at `/check` pass and `/dev` task commit. New sessions warm-start from the latest checkpoint.

## Core Separation

- `/prd` plans.
- `/dev` builds and proves technical correctness.
- `/check` verifies requirements and protects against regressions.
- `/uat` verifies human-visible UI quality.

## Command Modes

- `/sprint`: greenfield roadmap generation or incremental roadmap updates.
- `/prd`: greenfield epic planning or incremental plan repair/update.
- `/dev`: normal delivery execution or verification-recovery execution.
- `/check`: normal verification or recovery routing after failure.

## License

MIT
