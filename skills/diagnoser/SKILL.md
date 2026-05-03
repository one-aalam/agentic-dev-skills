---
name: diagnoser
description: >
  Diagnose bugs, failing tests, errors, and unexpected behaviour in a codebase.
  Trigger this skill whenever a user reports a bug, a test is failing, something
  is broken, there is an error or exception, behaviour is wrong or unexpected,
  or they say things like "figure out what's wrong", "why is this failing",
  "investigate this", "root cause", "debug this", "something broke", or pastes
  a stack trace, error log, or failing test output. Also trigger when the user
  wants to understand *why* something behaves the way it does before fixing it.
  This skill performs read-only investigation and produces a structured diagnosis
  report — it never modifies code. Pairs with the triager skill (to open an issue
  from the diagnosis) and the tdd-agent skill (to implement the fix).
---

# Diagnoser Skill

Read-only investigation of bugs, failures, and unexpected behaviour. Produces a
structured diagnosis report grounded in evidence — no guessing, no code changes.

---

## Inputs Expected

The user should provide at least one of:
- A bug description in natural language
- A stack trace or error message (pasted or in a file)
- A failing test name or test output
- A file path and line number where something goes wrong
- A description of expected vs. actual behaviour

If none of these are provided, ask: "What's the symptom? Paste the error, stack
trace, or describe what you expected to happen vs. what actually happened."

---

## Phase Overview

```
Phase 1 — ORIENT      Read project knowledge files
Phase 2 — SCOPE       Understand the reported symptom precisely
Phase 3 — HYPOTHESISE Form ranked hypotheses
Phase 4 — INVESTIGATE Verify/eliminate each hypothesis with evidence
Phase 5 — REPORT      Write the diagnosis report
Phase 6 — HANDOFF     Present findings, recommend next action
```

---

## Phase 1 — ORIENT

Before touching any application code, read the project knowledge layer:

```bash
# Required reads (in order)
cat CONTEXT.md          # invariants, domain glossary, sharp edges, architecture
cat AGENTS.md           # test commands, constraints, sensitive paths
cat DECISIONS.md        # recent architectural decisions that might be relevant

# If a diagnosis has already been attempted:
ls docs/reports/diagnosis-*.md 2>/dev/null | sort | tail -5
```

Key things to extract from CONTEXT.md:
- **Key Invariants** — anything the bug might be violating
- **Sharp Edges & Gotchas** — known problem zones
- **Domain Glossary** — correct terminology to use in the report
- **External Dependencies** — if the bug could be a downstream/API issue

If `CONTEXT.md` is missing: proceed but note in the report that project context
was unavailable. Do not fabricate invariants.

---

## Phase 2 — SCOPE

Precisely characterise the symptom before forming any hypotheses.

### 2.1 Parse the input

From the user's description or pasted output, extract:

| Field | What to find |
|---|---|
| **Symptom** | What is visibly wrong (error message, wrong output, crash) |
| **Trigger** | What action or input causes it |
| **Frequency** | Always / intermittent / only in certain environments |
| **First seen** | When did it start? After a deploy? After a code change? |
| **Environment** | dev / staging / prod; OS; runtime version |
| **Affected users** | All users / specific accounts / specific data |

### 2.2 Find the failure entry point

For stack traces:
```bash
# Identify the topmost frame in application code (not library/framework code)
# That is the entry point — start investigation there
```

For test failures:
```bash
# Run the specific test to confirm it still fails and capture output
<TEST_COMMAND> --testNamePattern "<test name>" 2>&1 | head -60
```

For runtime errors with no trace:
```bash
# Search for the error string in source
grep -r "<error message fragment>" src/ --include="*.ts" --include="*.py" --include="*.go" -l
```

### 2.3 Clarify if still ambiguous

If after reading the input you cannot identify the symptom or entry point, ask ONE
targeted question. Never ask multiple questions at once. Then wait for the answer.

---

## Phase 3 — HYPOTHESISE

Form a ranked list of hypotheses before looking at any more code. This prevents
confirmation bias from anchoring on the first thing that looks suspicious.

Hypothesis ranking criteria (most likely first):
1. **Simplest explanation** — off-by-one, nil/null dereference, wrong variable
2. **Recent change** — did a recent commit touch the affected path?
3. **Known gotcha** — does CONTEXT.md's "Sharp Edges" section point here?
4. **External cause** — API change, environment variable, dependency version
5. **Race condition or state** — intermittent failures often fall here
6. **Design-level issue** — wrong abstraction, violated invariant

```bash
# Check recent commits touching the affected area
git log --oneline -20 -- <affected_path>

# Check if the area was recently modified
git log --since="2 weeks ago" --oneline -- <affected_path>
```

Write down your hypotheses (internally) in ranked order before Phase 4.
Maximum 5 hypotheses. If you can't form even one, the symptom is too vague —
go back to Phase 2 and ask for more information.

---

## Phase 4 — INVESTIGATE

Work through hypotheses in ranked order. For each one:

1. **State the hypothesis clearly** (to yourself)
2. **Find the evidence that would confirm or refute it**
3. **Read the relevant code** — do not modify anything
4. **Mark it confirmed / refuted / inconclusive**
5. **Stop when one hypothesis is confirmed** — do not investigate further

### 4.1 Code reading commands

```bash
# Read a specific file
cat src/path/to/file.ts

# Read a range of lines
sed -n '45,80p' src/path/to/file.ts

# Find all usages of a function/symbol
grep -rn "functionName" src/ --include="*.ts"

# Find where a variable is mutated
grep -rn "variableName\s*=" src/ --include="*.ts"

# Trace imports/dependencies
head -30 src/path/to/file.ts   # imports section

# Check test for the affected unit
find . -name "*.test.*" -path "*<module>*" | head -5
cat <test_file>
```

### 4.2 Git archaeology

```bash
# What changed in this file recently
git log --oneline -10 -- src/path/to/file.ts

# See the actual diff of a commit
git show <commit-sha> -- src/path/to/file.ts

# When was a specific line last changed
git blame src/path/to/file.ts | grep -n "<line content>"
```

### 4.3 Invariant checking

For each Key Invariant in CONTEXT.md, check whether the buggy code path violates it:

```bash
# Example: if invariant is "records are never hard-deleted"
grep -rn "\.delete\(" src/ --include="*.ts" | grep -v "soft"
grep -rn "DELETE FROM" src/ --include="*.sql"
```

### 4.4 Stop conditions

Stop investigating when:
- One hypothesis is confirmed with clear code evidence
- All 5 hypotheses are refuted → write an inconclusive report, expand scope in Phase 2
- You find a security-related issue → stop immediately, escalate (see Error Cases)
- You've spent effort across >10 files without converging → the scope was wrong, reset

---

## Phase 5 — REPORT

Write the diagnosis to `docs/reports/diagnosis-<issue-id>.md`.

If there is no issue ID yet, use the symptom slug: `diagnosis-<YYYY-MM-DD>-<symptom-slug>.md`.

```bash
mkdir -p docs/reports
```

### Report format

```markdown
# Diagnosis: <issue-id or symptom-slug>

**Date:** <YYYY-MM-DD>
**Reported symptom:** <one sentence — the observable failure>
**Severity:** P0 | P1 | P2 | P3
**Status:** confirmed-root-cause | inconclusive | escalated

## Root Cause
<One clear sentence. No hedging.>

## Evidence
<The specific code, log line, or test output that proves the root cause.>

## Hypotheses Considered
| # | Hypothesis | Status | Evidence |
|---|-----------|--------|----------|
| 1 | <hypothesis> | ✅ Confirmed | <what confirmed it> |
| 2 | <hypothesis> | ❌ Refuted | <what refuted it> |

## Affected Paths
- `src/path/to/file.ts:47` — root cause location

## Invariant Violations
- Violates: "<invariant text>" — <how>
- No invariant violations found

## Recommended Fix
<Concrete description of what to change and where.>

## Risk of Fix
<What else could break?>

## Next Steps
- [ ] Open issue referencing this report (triager skill)
- [ ] Implement fix using tdd-agent
- [ ] Update CONTEXT.md "Sharp Edges" with this gotcha
```

---

## Phase 6 — HANDOFF

```
Diagnosis complete → docs/reports/diagnosis-<id>.md

Root cause: <one sentence>
Recommended fix: <one sentence>
Risk: <low | medium | high>

Next: run the triager skill to open a tracked issue, or jump straight to
tdd-agent if you want to implement the fix now.
```

---

## Error Cases

**Security finding:** Stop. Tell the user: "I found what looks like a security issue
in `<path>`. I've stopped the diagnosis. Please review it directly before we proceed."

**No CONTEXT.md:** Proceed but note: `[CONTEXT.md missing — diagnosis performed without project invariants]`

**Intermittent failure:** Focus on race conditions, shared mutable state, time-dependent
logic. Note repro was not confirmed.

**Bug in a dependency:** Document what the dependency does wrong, what version introduced
it, and whether a workaround exists. Recommend pinning the last good version.

---

## Reference

Reads: `CONTEXT.md`, `AGENTS.md`, `DECISIONS.md`, `docs/adr/`
Writes: `docs/reports/diagnosis-<id>.md`
Downstream: `triager` → `tdd-agent`
