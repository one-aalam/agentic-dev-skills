---
name: triager
description: Structures and labels issues. Applies the five-dimension label taxonomy from docs/.agent/labels.md and enforces the agent:ready contract. Trigger on: filing a new issue, labelling or triaging an existing one, converting a diagnosis report into a tracked issue, or assessing whether an issue is ready for autonomous agent pickup.
---

# Triager Skill

Turns raw problem descriptions or diagnosis reports into structured, labelled,
trackable issues. Enforces the label taxonomy. Assesses agent-readiness honestly.

---

## Inputs Expected

The user provides one of:
- A raw description of a bug / feature / chore in natural language
- A `docs/reports/diagnosis-<id>.md` file to convert into an issue
- An existing issue file that needs triage (missing labels or metadata)
- A GitHub issue URL or number (if GH issues are in use)
- Just "triage the latest diagnosis" — skill finds the most recent report

---

## Phase Overview

```
Phase 1 — ORIENT       Read labels taxonomy + project context
Phase 2 — INGEST       Parse the input (description, diagnosis, or existing issue)
Phase 3 — CLASSIFY     Assign all label dimensions
Phase 4 — ASSESS       Evaluate agent-readiness honestly
Phase 5 — ENRICH       Add structure missing from the raw input
Phase 6 — WRITE        Create or update the issue file
Phase 7 — HANDOFF      Confirm what was created, suggest next action
```

---

## Phase 1 — ORIENT

```bash
# Read the label taxonomy — this is the source of truth for all labels
cat docs/.agent/labels.md

# Read project context for domain understanding
cat CONTEXT.md | grep -A 50 "## Domain Glossary"
cat CONTEXT.md | grep -A 20 "## Key Invariants"

# Find next available issue ID
ls issues/ 2>/dev/null | grep -oP 'ISS-\d+' | sort -t- -k2 -n | tail -1
mkdir -p issues
```

Extract from `docs/.agent/labels.md`:
- All valid `type:*` labels
- All valid `P*` priority labels
- All valid `domain:*` labels for this project
- Agent-readiness criteria (`agent:ready` contract)

If `docs/.agent/labels.md` is missing: note the gap and use sensible defaults.

---

## Phase 2 — INGEST

### 2a. From a diagnosis report

```bash
cat docs/reports/diagnosis-<id>.md
```

Extract and map:
- **Symptom** → issue title
- **Root Cause** → issue body paragraph 1
- **Recommended Fix** → acceptance criteria seed
- **Affected Paths** → `affects:` frontmatter
- **Severity** → Priority (P0→P0, P1→P1, etc.)
- **Risk of Fix** → Notes section
- **Next Steps** → initial acceptance criteria checklist

### 2b. From a natural language description

Parse for: what is broken/wanted, who it affects, when it happens, any file paths,
urgency signals ("production", "blocking", "critical"). Mark inferences with `[inferred]`.

### 2c. From an existing issue file

```bash
cat issues/ISS-XXX-*.md
```

Identify which frontmatter fields are missing or have invalid values. Only those fields
will be updated.

---

## Phase 3 — CLASSIFY

### Type (exactly one)

| If the input describes... | Assign |
|---|---|
| Something broken, wrong output, crash, data loss | `type:bug` |
| A new user-visible capability | `type:feat` |
| Dependency update, config change, CI fix, cleanup | `type:chore` |
| Research, proof-of-concept, evaluate options | `type:spike` |
| Accumulated code quality issue, no behaviour change | `type:debt` |
| Documentation only | `type:docs` |
| CVE, auth bypass, data exposure, injection | `type:security` |

### Priority (exactly one)

| Signals | Priority |
|---|---|
| Production down, data loss, data exposure, total blocker | `P0` |
| Core feature broken, no workaround, many users affected | `P1` |
| Feature degraded, workaround exists, moderate impact | `P2` |
| Cosmetic, nice-to-have, edge case, future improvement | `P3` |

**P0 rule:** Add `<!-- ESCALATE: P0 -->` as the first line of the issue body.

### Domain (zero or more)

Use only `domain:*` labels defined in `docs/.agent/labels.md`. Do not invent new ones.

### Status: always `status:needs-triage` for new issues.

### Agent readiness — the contract

**Requirements for `agent:ready` (all six must be true):**
1. ✅ Acceptance criteria are written and unambiguous
2. ✅ At least one `affects:` file path is specified
3. ✅ No open questions remain in the issue body
4. ✅ Priority is P1 or lower
5. ✅ Type is NOT `type:security`
6. ✅ Fix is localised — does not require changing more than ~5 files

Partial → `agent:partial` with `missing-for-agent:` array listing gaps.
Any of 4–6 false, or work requires judgement → `agent:human-only`.

---

## Phase 4 — ASSESS

```
Readiness check:
  [ ] Acceptance criteria present and checkable?
  [ ] At least one file path hinted?
  [ ] No unanswered questions in body?
  [ ] Priority P1 or lower?
  [ ] Not a security issue?
  [ ] Scope bounded (≤5 files)?

Result: agent:ready | agent:partial | agent:human-only
Missing: [list any gaps]
```

---

## Phase 5 — ENRICH

Add structure if missing. Never remove or alter what the user wrote.

**Acceptance Criteria** (if absent):
```markdown
## Acceptance Criteria
- [ ] <criterion derived from description>
- [ ] Existing tests continue to pass
- [ ] New behaviour is covered by a test
```

**Affected Files (hint)** — copy from diagnosis report, or make an educated guess
marked `[hint — verify]`.

---

## Phase 6 — WRITE

### Issue file format

```markdown
---
id: ISS-XXX
title: "<title>"
type: <type:X>
priority: <PX>
labels: [<type:X>, <domain:Y>, status:needs-triage, <agent:Z>]
affects: [<path1>, <path2>]
depends-on: []
blocks: []
created: <YYYY-MM-DD>
diagnosis: <path if applicable>
missing-for-agent: <[] or ["list of gaps"]>
---

<!-- ESCALATE: P0 --> <!-- Only if P0 -->

## Description
<2–4 sentences. Use domain terms from CONTEXT.md.>

## Acceptance Criteria
- [ ] <criterion 1 — specific and checkable>
- [ ] Existing tests continue to pass

## Affected Files (hint)
- `<path/to/file>` — <what role it plays>

## Notes
<Risk, edge cases, references. 3–5 bullet points max.>

## References
- Diagnosis: `<path>` (if applicable)
- PRD: `<path>` (if applicable)
```

### For GitHub Issues

```bash
gh issue create \
  --title "<title>" \
  --body "<formatted body>" \
  --label "type:bug,P1,domain:auth,status:needs-triage,agent:ready"
```

Commit:
```bash
git add issues/<NEXT_ID>-<slug>.md
git commit -m "triage(ISS-XXX): add issue <title>"
```

---

## Phase 7 — HANDOFF

```
✅ Issue created: issues/ISS-XXX-<slug>.md

  Title:     <title>
  Type:      type:X
  Priority:  PX
  Readiness: agent:ready | agent:partial | agent:human-only

<If agent:partial>: To make agent:ready, add: <one sentence>
<If P0>: ⚠️ Requires immediate human attention before any agent work.
```

---

## Batch Triage Mode

"Triage all untagged issues" / "clean up the backlog":

```bash
grep -rL "priority:" issues/*.md 2>/dev/null
grep -rL "type:" issues/*.md 2>/dev/null
```

Process each in sequence, print a summary table when done.

---

## Error Cases

**No `docs/.agent/labels.md`:** Use standard defaults, note `[WARNING: labels.md not found]`.
**P0:** Write issue, add sentinel, tell user explicitly — no agent work.
**Ambiguous type:** Choose higher urgency, add comment `<!-- type: could also be X -->`.
**Duplicate:** Check `grep -r "<keywords>" issues/` before writing.

---

## Reference

Reads: `docs/.agent/labels.md`, `CONTEXT.md`, `docs/reports/diagnosis-*.md`
Writes: `issues/ISS-XXX-<slug>.md`
Downstream: `issue-planner`, `tdd-agent`, `arch-auditor`
