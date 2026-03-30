# /product — Kick off the product definition conversation

## Purpose

Run `/product` at the very start of a new project. Claude leads a structured conversation to draw out your product vision, goals, users, features, and constraints — then produces a rich `product_note.md` that serves as the single source of truth for all downstream commands. This is not a form to fill in — it is a collaborative thinking session. Claude asks probing questions and challenges assumptions until the picture is clear.

---

## How the Conversation Works

1. Claude opens with: "Tell me about your product idea — what problem are you solving and for whom?"
2. Claude listens, then asks clarifying questions across: user pain points, market context, competing solutions, key differentiators, must-have vs nice-to-have features, constraints, and success metrics.
3. Claude reflects back a structured summary and asks: "Does this capture it correctly, or have I missed anything?"
4. Iterate until the human confirms the picture is complete.
5. Claude generates `product_note.md` and asks for approval before committing.

---

## What product_note.md Must Contain

**product_note.md is the foundation everything else is built on.** It must be rich enough that `/architecture` can make real technical decisions, `/design` can make real UX decisions, and `/sprint` can generate real user stories without guessing. Every section below is required. Thin sections — one-liners, vague platitudes, placeholder text — must be challenged and expanded before Claude marks the document Approved.

> **Quality Gate:** Every section is required. Claude must challenge and expand any thin or vague section before marking the document Approved.

---

## Prescribed Format

```markdown
# Product Note: [Product Name]

Version: 1.0 | Status: Draft | Approved | Date: [YYYY-MM-DD]

## 1. Vision

One to two paragraphs. What is this product and why does it exist?
What core problem does it solve? For whom? Why now?
What does success look like in 12 months?

## 2. Target Users

### Primary User

Who they are — role, context, technical level.
What they do today to solve this problem (workarounds, existing tools).
What frustrates them most. What they value most.
Frequency of use: daily / weekly / occasionally.

### Secondary Users (if any)

[Repeat structure for each secondary persona]

## 3. Market & Competitive Context

What alternatives exist today (tools, processes, competitors)?
Where do those alternatives fall short?
What is our key differentiator — what do we do that they do not?

## 4. Features

### Feature 1: [Name]

Description: What the user can do. Be specific — avoid vague verbs.
User value: The concrete outcome for the user (not just "saves time").
Key interactions: How the user triggers and completes this feature.
Priority: Must Have | Should Have | Nice to Have

### Feature 2: [Name]

[Repeat structure for each feature]

## 5. Out of Scope (v1)

Explicitly list what this product will NOT do in v1.
For each item, briefly note WHY (future version, third-party, complexity, etc.).
This is used by /sprint to reject creeping stories.

## 6. Constraints

Timeline: [Hard deadlines — launch date, demo date, etc.]
Budget: [Rough budget if relevant to tech choices]
Platform: [iOS / Android / Web / Cross-platform]
Compliance: [GDPR / HIPAA / WCAG / SOC2 — any applicable]
Existing systems: [Must integrate with X / must NOT replace Y]
Team: [Solo / small team / agency — affects complexity decisions]

## 7. Success Metrics

How will we know this product is working? Be specific and measurable.
- [Metric 1: e.g. 500 active users within 60 days of launch]
- [Metric 2: e.g. core task completion in under 2 minutes]
- [Metric 3: e.g. < 3% weekly churn in first quarter]

## 8. Open Questions

Things not yet decided that may affect architecture or scope.
Each question should have an owner and a deadline for resolution.
- [Question 1 | Owner | Resolve by]
- [Question 2 | Owner | Resolve by]

## 9. Risks

Known risks that could derail the project.
- [Risk 1: description | Likelihood: H/M/L | Mitigation]
- [Risk 2: description | Likelihood: H/M/L | Mitigation]

## 10. Assumptions

Things we are assuming to be true that have not been validated.
- [Assumption 1 — what happens if this turns out to be wrong?]
- [Assumption 2 — what happens if this turns out to be wrong?]

---

Note for /sprint: Section 4 (Features) is the direct source for user story generation.
Each feature becomes one or more stories. The priority field maps to sprint ordering.
Section 5 (Out of Scope) is used to reject stories that creep in during planning.
```

---

## Claude Instructions for /product

1. Open with a warm, open question — do not show the template yet.
2. Ask probing follow-ups: "Who specifically feels that pain?", "What do they use today?", "Why hasn't this been solved already?", "What would make this a failure?"
3. Challenge vague answers: "Can you be more specific about the user value?" or "That sounds like a nice-to-have — is it truly must-have for v1?"
4. Once you have enough, draft `product_note.md` and present it section by section.
5. Flag any section that is thin and ask the human to expand before marking Approved.
6. Only mark Status: Approved when the human explicitly confirms.
7. Remind the human: "product_note.md is the foundation. Weak sections here mean weak stories, wrong architecture, and poor design decisions downstream."
