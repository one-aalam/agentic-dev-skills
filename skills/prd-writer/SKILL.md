---
name: prd-writer
description: Writes codebase-grounded PRDs with BDD acceptance criteria that downstream skills consume directly. Trigger when a user wants to define a feature before building it — from a rough idea, a conversation, or an existing issue. Interviews for missing context, researches the affected codebase area, and flags invariant conflicts before writing.
---

# PRD Writer Skill

Writes structured, codebase-grounded PRDs from feature descriptions, conversations,
or rough ideas. Produces BDD acceptance criteria that downstream skills can act on.

---

## Inputs Expected

The user provides one of:
- A feature description in natural language (any length, any formality)
- A conversation about a problem they want to solve
- An existing rough doc, notes, or bullet list to formalise
- A GitHub issue or `issues/ISS-*.md` file that needs a PRD before implementation
- Just "write a PRD for <feature name>" with no further detail (skill will interview)

---

## Phase Overview

```
Phase 1 — ORIENT      Read project context, existing PRDs, relevant ADRs
Phase 2 — INTERVIEW   Extract the full picture (may need user questions)
Phase 3 — RESEARCH    Understand the codebase area being affected
Phase 4 — DRAFT       Write the full PRD
Phase 5 — REVIEW      Show to user, collect edits
Phase 6 — FINALISE    Write the approved PRD to disk
Phase 7 — HANDOFF     Summarise and link downstream actions
```

---

## Phase 1 — ORIENT

Read project context before writing a single word of the PRD:

```bash
cat CONTEXT.md
ls docs/adr/ 2>/dev/null
ls docs/prd/ 2>/dev/null | grep -v TEMPLATE
ls issues/ 2>/dev/null | head -20
cat AGENTS.md  # layout section
```

Extract from CONTEXT.md:
- **Domain Glossary** — use these exact terms. Do not invent synonyms.
- **Key Invariants** — constraints the PRD must not violate
- **Tech Stack** — ensures Technical Constraints section is accurate
- **External Dependencies** — might affect the feature's feasibility
- **Sharp Edges** — known gotchas to call out in the PRD

---

## Phase 2 — INTERVIEW

The goal is to understand the problem deeply before writing anything.

### 2.1 Core questions (if description is thin)

Ask one at a time:
1. "What is the specific user pain or business gap this solves?"
2. "Who experiences this problem?"
3. "How will you know this feature worked?"
4. "What is explicitly NOT included in this feature?"

Or, if the user is in a hurry, all four at once as "four quick questions".

### 2.2 Conflict checks (internal)

```bash
grep -ri "<feature keyword>" docs/adr/ 2>/dev/null
grep -ri "<feature keyword>" docs/prd/ 2>/dev/null
grep -ri "<feature keyword>" issues/ 2>/dev/null | head -5
```

If overlap found: "I found an existing [ADR/PRD/issue] that may overlap — should I
treat this as an extension, or a separate feature?"

### 2.3 When not to interview

Skip if: description is >3 structured paragraphs, user pasted an existing spec, or
user says "just write a draft" — write it, mark assumptions with `[inferred]`.

---

## Phase 3 — RESEARCH

Understand the codebase area before writing Technical Constraints:

```bash
find src/ -name "*.ts" | xargs grep -l "<feature keyword>" 2>/dev/null | head -10
find . -name "*.sql" -o -name "schema.*" -o -name "models.*" 2>/dev/null | head -5
grep -rn "router\.\|app\.\|@app\.\|@router\." src/ --include="*.ts" | head -20
```

---

## Phase 4 — DRAFT

Write the full PRD using only domain terms from CONTEXT.md.

```markdown
# PRD: <Feature Name>

**Status:** draft
**Author:** <name>
**Date:** <YYYY-MM-DD>
**Codebase snapshot:** <git rev-parse HEAD>
**Related issues:** <ISS-XXX if known>

## Problem Statement
<2–4 sentences. User pain or business gap. No solution language. Domain terms only.>

## Success Metrics
| Metric | Baseline | Target | How Measured |
|--------|----------|--------|--------------|
| <metric> | <current> | <goal> | <method> |

## Users & Context
**Primary user:** <role from CONTEXT.md glossary>
**Trigger:** <what causes the user to need this>
**Current workaround:** <how they do it today>

## User Stories
- As a **<role>**, I want to **<action>** so that **<outcome>**.

## Acceptance Criteria

### Scenario 1: <Happy path name>
```gherkin
Given <a specific precondition>
When <the user takes a specific action>
Then <a specific, verifiable outcome occurs>
```

### Scenario 2: <Error case>
```gherkin
Given <precondition>
When <action>
Then <outcome>
```

## Out of Scope
- <explicit exclusion>

## Technical Constraints
**Invariants that apply:**
- <copy relevant invariant from CONTEXT.md verbatim>

**Architecture constraints:**
- <must use existing X, must go through Y>

## Open Questions
| # | Question | Owner | Due | Resolution |
|---|----------|-------|-----|------------|
| 1 | <question> | <owner> | <date> | — |

## Implementation Plan
(to be created by issue-planner after PRD approval)

## Decision Log
| Decision | Rationale | ADR needed? |
|----------|-----------|-------------|
```

### Writing rules

- **Exact domain terms only** from CONTEXT.md glossary.
- **Scenarios must be falsifiable.** "The user can log in" is not testable. Specific preconditions, actions, and outcomes are.
- **No implementation details** in criteria. "Use a JWT" is implementation; "Authentication persists across page refreshes" is a requirement.
- **Flag invariant conflicts:**
  ```
  ⚠️ INVARIANT CONFLICT: This feature would violate: "<invariant text>". Resolution needed.
  ```

---

## Phase 5 — REVIEW

Show the draft. Ask:
> "Are the acceptance criteria specific enough to write tests from? Anything missing from Out of Scope? Any open questions to add? Tell me what to change, or say 'approved' to write it to disk."

---

## Phase 6 — FINALISE

```bash
mkdir -p docs/prd
SLUG=$(echo "<feature name>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g')
FILENAME="docs/prd/PRD-${SLUG}.md"
```

Update status to `review`. If an architectural decision was made, create an ADR stub.

```bash
git add docs/prd/PRD-${SLUG}.md
git commit -m "docs(prd): add PRD for <feature name>

Status: review
Scenarios: N
Open questions: N blocking / N non-blocking"
```

---

## Phase 7 — HANDOFF

```
✅ PRD written: docs/prd/PRD-<slug>.md
   Status: review
   Scenarios: N | Open questions: N (N blocking)

Next:
  1. Resolve blocking open questions
  2. Change status to 'approved'
  3. Run issue-planner to break into trackable issues
```

---

## Error Cases

**Invariant conflict:** Write PRD with warning, leave status as `draft`.
**Existing PRD:** Offer to update existing file rather than create duplicate.
**No problem context:** Run Phase 2 interview. Don't write a PRD for the wrong problem.

---

## Reference

Reads: `CONTEXT.md`, `docs/adr/`, `docs/prd/TEMPLATE.md`, source code
Writes: `docs/prd/PRD-<slug>.md`, `docs/adr/` (if ADR needed), `DECISIONS.md`
Downstream: `issue-planner` → `tdd-agent`
