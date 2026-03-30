# /spike — Time-boxed Technical Investigation

## Purpose

A spike is triggered by the 3-strike rule in `/dev` when an approach fails repeatedly. A spike is not open-ended research — it is a structured experiment with a hypothesis, a hard time-box, and a mandatory decision record that writes back to `architecture.md`. After a spike, `/dev` resumes with a revised approach.

---

## When to Run /spike

- **`/dev` 3-strike rule:** same task fails 3 times with different approaches.
- **Unknown third-party integration:** external API or library behaviour is unclear and cannot be resolved by reading docs alone.
- **Architectural fork:** two valid approaches with meaningfully different consequences that cannot be resolved without a proof of concept.

---

## Time-Box

Default: **2 hours.** Maximum: **4 hours.**

If the spike cannot produce a decision within the time-box, the outcome is itself a decision: the approach is too risky or complex and must be descoped, simplified, or deferred to a dedicated spike story in the next sprint. Claude states this explicitly — it does not silently extend the investigation.

---

## Spike Phases

### Phase 1: Frame the Hypothesis

Before writing any code, state in one sentence what is being tested and what a successful outcome looks like. This is the hypothesis. If Claude cannot state a clear hypothesis, the problem is not yet well-defined — spend the first 15 minutes narrowing it before proceeding.

### Phase 2: Investigate

Write a minimal proof-of-concept — the smallest possible code that tests the hypothesis. This is throwaway code: it does not follow TDD, does not need tests, and will not be committed. The goal is a yes/no answer to the hypothesis, not a production implementation.

### Phase 3: Decision Record

Write the spike decision record. Append it to `architecture.md` Section 9 (or create Section 9 "Decision Log" if it does not exist). Delete the throwaway PoC code. Resume `/dev` with the revised approach.

```markdown
## Spike Decision Record: [task name] | [YYYY-MM-DD]

Triggered by: 3-strike rule on task [N] | integration unknown | architectural fork

Time-box: [N] hours | Actual: [N] hours

Hypothesis: [one sentence — what was being tested and what success looks like]

Approaches tried:
- [approach]: [outcome — worked / failed / partial, reason]

Decision: [chosen approach] | [descoped — reason] | [deferred to spike story in Sprint N]

architecture.md impact: [section updated] | [none]

Resume /dev: task [N] with [revised approach] | blocked — [reason]
```

---

## architecture.md Integration

Every spike that produces a decision must append its decision record to `architecture.md`. If Section 9 (Decision Log) does not exist, create it. The record is permanent — it explains why the codebase looks the way it does and prevents future developers from re-investigating the same dead ends.

After writing the record, Claude confirms: "architecture.md updated. Spike complete. Resuming /dev task [N] with [approach]."

---

## After /spike

- Delete the throwaway PoC code — do not commit it.
- Append the decision record to `architecture.md` Section 9.
- If the decision requires changes to the task breakdown: update `task_spec_document.md` and `TODO.md` before resuming `/dev`.
- If the spike exceeded the time-box or ended inconclusively: tell the human the outcome, state whether the story should be descoped or deferred, and do not resume `/dev` until the human confirms.
