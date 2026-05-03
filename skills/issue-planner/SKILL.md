---
name: issue-planner
description: >
  Break a PRD, large feature, or complex task into atomic, trackable, dependency-ordered
  issues that humans or agents can pick up independently. Trigger this skill when a user
  has an approved PRD and wants to create issues from it, wants to break down a large
  feature into tasks, wants to plan the implementation of something, says things like
  "break this into issues", "create tasks for this", "plan the implementation",
  "make this agent-ready", "decompose this feature", "what are the sub-tasks", or
  "turn this PRD into tickets". Also trigger when a user wants to sequence work,
  identify dependencies between tasks, or estimate the scope of a feature.
  This skill reads PRDs and CONTEXT.md, produces structured issue files (or GitHub
  Issues), identifies inter-issue dependencies, and labels each issue correctly
  including agent-readiness. Pairs with prd-writer (upstream) and tdd-agent (downstream).
---

# Issue Planner Skill

Breaks PRDs or feature descriptions into atomic, ordered, agent-labelled issues.
Each issue is independently testable, mergeable, and correctly labelled for the
triager/tdd-agent pipeline.

---

## Inputs Expected

The user provides one of:
- A path to an approved PRD (`docs/prd/PRD-<slug>.md`)
- A natural language feature description (skill will structure it internally)
- "Break down ISS-XXX" — a large existing issue to decompose
- "Plan the implementation of <feature>" — no prior doc needed

---

## Phase Overview

```
Phase 1 — ORIENT       Read PRD, CONTEXT.md, labels, existing issues
Phase 2 — DECOMPOSE    Identify atomic work units from acceptance criteria
Phase 3 — SEQUENCE     Order units, identify dependencies
Phase 4 — SIZE CHECK   Validate each unit is appropriately scoped
Phase 5 — LABEL        Apply full label taxonomy to each issue
Phase 6 — PREVIEW      Show the plan to the user before writing
Phase 7 — WRITE        Create all issue files
Phase 8 — LINK BACK    Update the PRD's Implementation Plan section
Phase 9 — HANDOFF      Summary table + next action
```

---

## Phase 1 — ORIENT

```bash
cat docs/prd/PRD-<slug>.md
cat CONTEXT.md
cat docs/.agent/labels.md
ls issues/ 2>/dev/null | grep -oP '\d+' | sort -n | tail -1   # next ID
grep -h "^title:" issues/*.md 2>/dev/null | head -20
cat AGENTS.md | grep -A 20 "Repository Layout"
ls src/ 2>/dev/null | head -20
```

Extract from the PRD:
- **Acceptance Criteria scenarios** — each is a decomposition seed
- **Out of Scope** — hard boundary, don't create issues for these
- **Open Questions** — if any are blocking, stop before planning
- **Technical Constraints** — which issues need extra care
- **Implementation Plan** — rough ordering suggested (if present)

**Blocking questions check:** If the PRD has unresolved blocking questions, stop:
> "The PRD has N unresolved blocking questions. Resolve them first, or tell me to
> proceed and I'll flag assumptions."

---

## Phase 2 — DECOMPOSE

**One issue = one independently mergeable unit of work.**

Atomic if: testable independently, PR doesn't depend on in-progress work, completable
in ~0.5–2 days, touches ≤5 files.

### Decomposition patterns

| PRD element | How to decompose |
|---|---|
| BDD scenario with 1 clear action | → 1 issue |
| BDD scenario spanning multiple layers (API + DB + UI) | → 2–3 issues, one per layer |
| Data model change implied | → Always separate issue (migrations first) |
| New API endpoint | → 1 issue for endpoint, 1 for client if applicable |
| Auth/permission requirement | → Separate issue if non-trivial |
| Configuration / feature flag | → 1 issue, comes first |
| Documentation update | → 1 issue `type:docs`, comes last |

### Common dependency order

```
ISS-N01: Data model / migration
ISS-N02: Core business logic (depends on N01)
ISS-N03: API layer (depends on N02)
ISS-N04: Client/UI layer (depends on N03)
ISS-N05: Integration / E2E tests (depends on N03, N04)
ISS-N06: Configuration / feature flag
ISS-N07: Docs update (last)
```

---

## Phase 3 — SEQUENCE

For each pair: "Can B start before A is merged?" If no → A precedes B.

- No dependencies = `agent:ready` candidate
- Blocked by in-progress work = `depends-on: [ISS-NXX]`
- Circular dependencies = decomposition error, revisit Phase 2

---

## Phase 4 — SIZE CHECK

| Signal | Action |
|---|---|
| Issue touches >5 files | Split |
| Can't describe the PR in one sentence | Too big |
| "Fix typo in comment" | Too small — merge with adjacent issue |
| >5 acceptance criteria | Consider splitting |

---

## Phase 5 — LABEL

**agent:ready requires ALL of:**
1. ✅ Acceptance criteria: specific, checkable, written in the issue
2. ✅ At least one `affects:` file path
3. ✅ No open questions
4. ✅ Priority P1 or lower
5. ✅ Not `type:security`
6. ✅ Scope bounded (≤5 files)

Common `agent:partial` reasons: unclear file paths, design depends on preceding issue,
implementation approach not yet decided.

---

## Phase 6 — PREVIEW

Show plan as a table before writing anything:

```
## Implementation Plan for: <PRD name>

N issues | N agent:ready, N agent:partial, N human-only

| # | ID       | Title                  | Type       | P  | Depends on | Readiness     |
|---|----------|------------------------|------------|----|------------|---------------|
| 1 | ISS-N01  | <title>                | type:chore | P1 | —          | agent:ready   |
| 2 | ISS-N02  | <title>                | type:feat  | P1 | ISS-N01    | agent:ready   |
| 3 | ISS-N03  | <title>                | type:feat  | P1 | ISS-N02    | agent:partial |

agent:partial reasons:
  ISS-N03: <what's missing>

Proceed with writing these issues? [yes / adjust]
```

---

## Phase 7 — WRITE

```bash
mkdir -p issues
```

Issue file format:

```markdown
---
id: ISS-NXX
title: "<title>"
prd: docs/prd/PRD-<slug>.md
type: <type:X>
priority: <PX>
labels: [<type:X>, <domain:Y>, status:needs-triage, <agent:Z>]
affects: [<path1>, <path2>]
depends-on: [<ISS-NXX or []>]
blocks: [<ISS-NXX or []>]
created: <YYYY-MM-DD>
missing-for-agent: <[] or ["gap 1"]>
---

## Description
<2–4 sentences. Reference the PRD scenario by name. Domain terms only.>

## Acceptance Criteria
- [ ] <criterion 1 — specific and testable>
- [ ] Existing tests continue to pass
- [ ] New behaviour covered by tests

## Affected Files (hint)
- `<path/to/file>` — <role>
- `<path/to/test>` — test to create or update

## Implementation Notes
- Invariant: <relevant>
- ADR: <relevant>
- Note: <gotcha from Sharp Edges>

## References
- PRD: `docs/prd/PRD-<slug>.md` (Scenario: <name>)
```

Commit all at once:

```bash
git add issues/ISS-N*.md
git commit -m "plan(<feature-slug>): break PRD into N issues (ISS-N01..ISS-N0N)

N agent:ready, N agent:partial, N human-only
Source PRD: docs/prd/PRD-<slug>.md"
```

---

## Phase 8 — LINK BACK

Update the PRD's Implementation Plan section with actual issue links.
Ask: "Should I also mark the PRD status as 'approved'?"

---

## Phase 9 — HANDOFF

```
✅ Implementation plan created: N issues

  agent:ready   N  — ready for tdd-agent immediately
  agent:partial N  — gaps listed below
  human-only    N  — require human involvement

agent:partial gaps:
  ISS-N03: <what's needed>

→ Run tdd-agent on any agent:ready issue to start
→ Full list in docs/prd/PRD-<slug>.md under Implementation Plan
```

---

## Error Cases

**Blocking open questions:** Stop. Don't plan around unresolved design questions.
**Vague acceptance criteria:** Rewrite to testable form, show user before planning.
**No PRD:** Run 2–3 question mini-interview, mark `planned without PRD` in frontmatter.
**>15 issues:** Feature is too large — split into two PRDs.

---

## Reference

Reads: `docs/prd/PRD-<slug>.md`, `CONTEXT.md`, `docs/.agent/labels.md`, `AGENTS.md`, `issues/`
Writes: `issues/ISS-NXX-<slug>.md`, `docs/prd/PRD-<slug>.md` (Implementation Plan)
Downstream: `tdd-agent`, `triager`
