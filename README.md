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
| `/sprint` | Transform `product_note.md` into a sprint roadmap with atomic user stories (`story-N.md`). |
| `/prd` | Generate a detailed task list for TDD implementation. Breaks stories into 5-10 min red-green-refactor units with functional tests. |
| `/dev` | Implement each task using TDD (red-green-refactor). |
| `/check` | Automated verification gate — tests, types, linting, acceptance criteria, regression, CI. Gates `/uat` or merge. |
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
5. **`/prd`** — Break each story into atomic TDD tasks with functional tests
6. **`/dev`** — Implement tasks using red-green-refactor
7. **`/check`** — Run the full verification gate (tests, CI, acceptance criteria)
8. **`/uat`** — Human sign-off for UI stories, then merge

### Existing Codebase

Use **`/issue`** as the entry point. It classifies the work and routes to the appropriate workflow:
- **Task** (no user-facing change) → `/dev` directly
- **Bug fix** → `/prd` → `/dev` → `/check`
- **Improvement** → `/prd` → `/dev` → `/check` → `/uat`
- **Feature** → `/sprint` → full workflow

### When You're Stuck

**`/spike`** runs a time-boxed investigation (2-4 hours) and produces a permanent decision record appended to `architecture.md`.

### Between Sessions

**`/handover`** ensures continuity. It writes checkpoints automatically at `/check` pass and `/dev` task commit. New sessions warm-start from the latest checkpoint.

## License

MIT
