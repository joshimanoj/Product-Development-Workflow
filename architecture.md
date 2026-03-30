# /architecture — Define the technical foundation

## Purpose

Run `/architecture` after `product_note.md` and `Design.md` are Approved. Claude leads a technical decision-making conversation covering platform, stack, data models, APIs, infrastructure, and architectural patterns — informed by the UX and interaction requirements captured in `Design.md`. The output — `architecture.md` — is the authoritative technical reference that `/sprint` and `/prd` depend on in full.

`/dev` loads a focused summary (`architecture-dev-summary.md`) generated at `/sprint` time, containing only what is needed for implementation: stack, naming conventions, file structure, and test runner commands. It must be detailed enough that Claude can make correct implementation decisions without asking basic technical questions during development.

---

## How the Conversation Works

1. Claude reads `product_note.md` in full — constraints, platform, integrations, scale expectations — before asking anything.
2. Claude asks targeted technical questions: "Do you have a preferred stack or are we choosing from scratch?", "What are your scaling expectations at launch vs 6 months?", "Are there existing systems we must integrate with?", "Do you have authentication preferences?", "What are your offline / performance requirements?"
3. Claude proposes a recommended stack with explicit rationale and trade-offs, then asks for confirmation or alternatives.
4. Claude works through each section of `architecture.md` collaboratively, confirming decisions before moving on.
5. Claude generates the full document and asks for approval before committing.

---

## What architecture.md Must Contain

> **Quality Gate:** architecture.md must be detailed enough that a developer joining the project cold can understand every technical decision, why it was made, and what the constraints are. Vague entries like "REST API" or "PostgreSQL" without rationale, structure, or constraints are not acceptable. Claude must challenge and expand thin entries before marking the document Approved.

---

## Prescribed Format

```markdown
# Architecture: [Product Name]

Version: 1.0 | Status: Draft | Approved | Date: [YYYY-MM-DD]

## Version History

| Version | Date | Changes | Status |
|---------|------|---------|--------|
| 1.0 | YYYY-MM-DD | Initial architecture definition | Approved |

## 1. Platform & Technical Stack

Platform: [iOS / Android / Web / Cross-platform]
Primary Language: [Swift / Kotlin / TypeScript / Python / etc.]
Framework: [SwiftUI / Jetpack Compose / React / Next.js / etc.]
Rationale: [Why this stack for this product and team]

State Management: [Redux / MobX / Zustand / ViewModel / etc.]
Rationale: [Why — complexity level, team familiarity, scale needs]

Backend: [REST / GraphQL / Firebase / Supabase / custom]
Rationale: [Why — real-time needs, simplicity, existing infra]

Database: [PostgreSQL / SQLite / Firestore / DynamoDB / etc.]
Rationale: [Why — relational needs, scale, offline support]

Auth: [Firebase Auth / Auth0 / Supabase Auth / custom]
Rationale: [Why — SSO needs, compliance, cost]

Hosting / Infrastructure: [Vercel / AWS / GCP / Railway / etc.]
Rationale: [Why — cost at scale, team familiarity, CI/CD integration]

## 2. Third-Party Dependencies

| Library / Service | Purpose | Version | Risk if removed |
|-------------------|---------|---------|-----------------|
| [name] | [what it does] | [x.x.x] | [High/Med/Low] |

## 3. Data Models

### [Model Name]

Purpose: [What this model represents in the domain]

Fields:
- [field]: [type] — [description, constraints, defaults]
- [field]: [type] — [description, constraints, defaults]

Relationships:
- [e.g. User has many Tasks via user_id]

Indexes:
- [field or compound index — why it exists]

Notes:
- [Soft delete strategy / audit trail / versioning if applicable]

## 4. API Structure

Style: REST | GraphQL | tRPC | etc.
Base URL: [endpoint]
Auth: [Bearer token / API key / session cookie / etc.]
Versioning: [/v1/ prefix / header-based / none]

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| GET | /api/v1/[resource] | [what it returns] | Yes / No |
| POST | /api/v1/[resource] | [what it creates] | Yes / No |
| PATCH | /api/v1/[resource]/:id | [what it updates] | Yes / No |
| DELETE | /api/v1/[resource]/:id | [what it removes] | Yes / No |

Error format:
{ "error": { "code": "VALIDATION_FAILED", "message": "...", "fields": {} } }

## 5. Architecture Patterns

Pattern: [MVVM / Clean Architecture / MVC / Hexagonal / etc.]
Rationale: [Why this pattern for this product]

Naming conventions:
- Files: [camelCase / snake_case / PascalCase]
- Components: [PascalCase]
- Variables: [camelCase]
- Database columns: [snake_case]

File structure:
src/
  features/          # Feature-based modules (one folder per feature)
    [feature-name]/
      components/    # UI components for this feature
      hooks/         # Feature-specific hooks / view models
      services/      # Feature-specific API calls
      types.ts       # Feature-specific types
  shared/
    components/      # Shared UI components
    hooks/           # Shared hooks / utilities
    types/           # Global types
  services/
    api/             # API client and interceptors
    auth/            # Auth service

## 6. Infrastructure & DevOps

Environments: [local / staging / production]
CI/CD: [GitHub Actions / CircleCI / etc.]
Branch strategy: [trunk-based / gitflow / feature branches]
Deployment: [auto-deploy on merge to main / manual approval gate]
Monitoring: [Sentry / Datadog / LogRocket / etc.]
Secret management: [env vars / Vault / AWS Secrets Manager]

## 7. Non-Functional Requirements

Performance: [e.g. API p95 response < 200ms, initial load < 2s]
Accessibility: [e.g. WCAG 2.1 AA]
Security: [e.g. OWASP Top 10, input sanitisation, rate limiting]
Offline support: [supported / not supported — if supported, how?]
Internationalisation: [supported / not supported]
Browser / OS targets: [e.g. iOS 16+, Android 12+, Chrome/Safari/Firefox latest]

## 8. Testing Strategy

Unit tests: [Jest / Vitest / pytest / XCTest / etc.]
Integration tests: [approach and tooling]
E2E / Functional: [Playwright / Detox / XCUITest / patrol]
Coverage target: [e.g. 80% unit test coverage minimum]

Mobile targets (confirmed here — used by /sprint, /prd, and CI from this point on):
iOS Simulator: [Yes / No] — [device name, OS version e.g. iPhone 15, iOS 17]
Android Emulator: [Yes / No] — [device name, API level e.g. Pixel 7, API 34]

Test data strategy (required — used by /prd when writing task test cases):
Approach: [factories / fixtures / seeds / inline mocks — choose one primary approach]
Location: [e.g. tests/fixtures/ | tests/factories/ | test/support/]
Isolation: [e.g. each test resets DB state via transaction rollback / truncation]
External services: [real / mocked / stubbed — state your policy]

## 9. Open Architectural Decisions

Unresolved decisions that may affect implementation.
(Clear this section when Status = Approved)
- [Decision needed | Options considered | Owner | Resolve by]
```

---

## Claude Instructions for /architecture

1. Read `product_note.md` and `Design.md` fully before asking any questions. Identify constraint-driven decisions first: platform from `product_note.md` Section 6, integrations from Section 4, compliance from Section 6. Use `Design.md` to understand UX and interaction requirements that should shape technical decisions (e.g. navigation patterns, offline needs, animation complexity).
2. Propose a recommended stack with explicit rationale and trade-offs — do not just ask "what do you want?".
3. Work through each section collaboratively. Confirm decisions before moving on.
4. Pay special attention to Section 8 (Testing Strategy) — confirm emulator targets here. These are locked in and used by `/sprint`, `/prd`, and CI config from this point forward. They are not asked again.
5. Challenge thin entries: "You've listed PostgreSQL but haven't defined your data models or indexing strategy — let's work through that now."
6. Only mark Status: Approved when the human explicitly confirms every section.
