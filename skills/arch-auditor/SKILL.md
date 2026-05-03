---
name: arch-auditor
description: Backward-looking codebase health check. Reads the codebase against CONTEXT.md invariants and accepted ADRs, scans for doc drift, debt hotspots, and security smells, then writes an audit report and auto-opens P1+ issues. Trigger on: audit requests, post-refactor checks, scheduled reviews, or any question about whether the codebase still matches its documentation.
---

# Arch Auditor Skill

Periodic, read-only architectural health check. Compares the living codebase against
CONTEXT.md, ADRs, and good engineering practices. Produces an audit report and opens
issues for significant findings.

---

## Inputs Expected

- "Audit the codebase" — full audit
- "Audit the auth module" — scoped to a specific area
- "Check for drift since last month" — time-scoped
- "Audit after the billing refactor" — post-change audit
- No input (scheduled/automated invocation) — full audit

---

## Phase Overview

```
Phase 1 — ORIENT         Read CONTEXT.md, ADRs, recent git history
Phase 2 — INVARIANTS     Check every Key Invariant against the codebase
Phase 3 — ADR CHECKS     Verify accepted ADRs are being followed
Phase 4 — DOC DRIFT      Find sections of CONTEXT.md that no longer match code
Phase 5 — DEBT SCAN      Identify accumulating complexity and hotspots
Phase 6 — COVERAGE       Check test coverage for recently changed code
Phase 7 — SECURITY SNIFF Lightweight check for common security smells
Phase 8 — REPORT         Write the full audit report
Phase 9 — ISSUES         Open issues for P1+ findings
Phase 10 — HANDOFF       Summary with severity breakdown
```

---

## Phase 1 — ORIENT

```bash
cat CONTEXT.md
cat DECISIONS.md
cat AGENTS.md
ls docs/adr/ 2>/dev/null
for f in docs/adr/*.md; do echo "=== $f ==="; cat "$f"; done
git log --oneline --since="30 days ago" --stat | head -100
git diff --name-only HEAD~20 HEAD | sort | uniq
find src/ -name "*.ts" -o -name "*.py" -o -name "*.go" 2>/dev/null | sed 's|/[^/]*$||' | sort | uniq | head -40
ls docs/reports/audit-*.md 2>/dev/null | sort | tail -3
```

---

## Phase 2 — INVARIANT CHECKS

For each Key Invariant in CONTEXT.md, search for violations. This is the most important
phase. Invariant violations are always P1 or higher.

See `references/invariant-checks.md` for grep strategies per invariant type.

For each invariant, record:
- **Status:** ✅ No violations | ⚠️ N potential violations | ❌ N confirmed violations
- **Evidence:** file:line references
- **Severity:** P0 (data integrity/security) | P1 (correctness) | P2 (hygiene)

---

## Phase 3 — ADR CHECKS

For each ADR with `Status: accepted`, verify the decision is being followed.

```bash
grep -l "Status.*accepted" docs/adr/*.md | while read f; do
  echo "=== $f ==="; cat "$f"; echo
done
```

Derive a specific codebase check per ADR. Examples:

| ADR says | Check |
|---|---|
| "Use PostgreSQL, not MongoDB" | `grep -rn "mongoose\|mongodb" package.json` |
| "Use JWT for session tokens" | sample token handling code in src/auth/ |
| "All new endpoints require auth middleware" | sample 5 recent endpoint additions |
| "Use the repository pattern" | look for direct DB calls outside repositories |

ADR violations are P1 findings.

---

## Phase 4 — DOC DRIFT

### 4.1 Tech Stack accuracy

```bash
cat package.json | python3 -c "import json,sys; d=json.load(sys.stdin); \
  [print(k,v) for k,v in {**d.get('dependencies',{}),**d.get('devDependencies',{})}.items()]"
```

Compare against CONTEXT.md Tech Stack table.

### 4.2 Architecture Overview accuracy

Verify components described in Architecture Overview exist at the stated paths.

### 4.3 Changelog staleness

```bash
CONTEXT_SHA=$(git log --format="%H" -- CONTEXT.md | head -1)
git log --oneline ${CONTEXT_SHA}..HEAD | wc -l
```

If >50 commits since last CONTEXT.md update without a changelog entry: flag as doc drift.

### 4.4 Domain Glossary drift

```bash
grep -oP '\| \K[A-Z][a-zA-Z]+(?= \|)' CONTEXT.md | sort > /tmp/glossary-terms.txt
grep -rn "[A-Z][a-zA-Z]\+Service\|[A-Z][a-zA-Z]\+Repository\|[A-Z][a-zA-Z]\+Manager" \
  src/ --include="*.ts" | grep -oP '[A-Z][a-zA-Z]+(?=Service|Repository|Manager)' | \
  sort | uniq > /tmp/code-terms.txt
comm -23 /tmp/code-terms.txt /tmp/glossary-terms.txt
```

---

## Phase 5 — DEBT SCAN

### 5.1 Churn hotspots

```bash
git log --since="90 days ago" --name-only --format="" | sort | uniq -c | sort -rn | head -20
```

High churn + no tests = P1 debt.

### 5.2 File size outliers

```bash
find src/ -name "*.ts" -o -name "*.py" | xargs wc -l | sort -n | tail -20
```

Files >500 lines: flag as candidates for splitting.

### 5.3 TODO/FIXME debt

```bash
grep -rn "TODO\|FIXME\|HACK\|XXX\|TEMP" src/ --include="*.ts" --include="*.py" | \
  grep -v test | grep -v spec
```

TODOs >30 days old in high-churn files = P2 debt.

### 5.4 Dependency health

```bash
npm outdated 2>/dev/null | head -20
npm audit --audit-level=high 2>/dev/null | tail -10
```

---

## Phase 6 — COVERAGE CHECK

```bash
git diff --name-only HEAD~30 HEAD | grep -E "\.(ts|py|go|rs)$" | grep -v test | grep -v spec

for f in <changed_files>; do
  TEST=$(echo $f | sed 's/src\//tests\//; s/\.\(ts\|py\)/\.test\.\1/')
  [ -f "$TEST" ] && echo "✅ $f" || echo "❌ No test: $f"
done
```

Changed files with no test coverage = P2 findings.

---

## Phase 7 — SECURITY SNIFF

Lightweight — not a security audit. Flag obvious smells for human review only.

```bash
# Hardcoded secrets
grep -rn "password\s*=\s*['\"][^'\"]" src/ --include="*.ts" | grep -v test

# Unparameterised SQL
grep -rn "query\|execute" src/ --include="*.ts" | grep "WHERE\|INSERT" | \
  grep -v "?" | grep -v "param" | head -10

# eval / dynamic execution
grep -rn "eval(\|exec(\|execSync(\|child_process" src/ --include="*.ts" | grep -v test
```

**Security findings are NEVER auto-issued.** Flag in report as `⚠️ SECURITY SMELL`,
describe the location vaguely (no exploit path), escalate to human.

---

## Phase 8 — REPORT

```bash
mkdir -p docs/reports
```

Report format:

```markdown
# Audit Report: <YYYY-MM-DD>

**Scope:** full codebase | <module>
**Compared against:** CONTEXT.md (last updated: <date>), N ADRs
**Git range:** <from> → HEAD
**Previous audit:** <path>

## Executive Summary
<3–5 sentences. Overall health, most important finding, trend vs. previous audit.>

**Findings by severity:**
| Severity | Count | Auto-issuing |
|----------|-------|--------------|
| P0 | N | No — escalate immediately |
| P1 | N | Yes |
| P2 | N | Yes |
| P3 | N | Documented only |

## Invariant Violations
### <Invariant text>
**Status:** ✅ Clean | ⚠️ warnings | ❌ N violations
- `src/path/file.ts:47` — <what code does and why it violates>

## ADR Compliance
| ADR | Title | Status | Notes |
|-----|-------|--------|-------|
| ADR-001 | <title> | ✅ Followed | |
| ADR-003 | <title> | ❌ Violated | `src/path/file.ts:12` |

## Documentation Drift
- `Tech Stack`: <mismatch>
- `Domain Glossary`: N terms in code not in glossary: TermA, TermB
- `Changelog`: N commits since last entry

### Recommended CONTEXT.md updates
- [ ] <update 1>

## Debt Hotspots
| File | Changes (90d) | Has Tests | Risk |
|------|--------------|-----------|------|
| `src/path/file.ts` | 23 | ❌ | high |

## Security Smells
⚠️ SECURITY SMELL: `src/path/file.ts:88` — possible unparameterised query. Human review required.

## Recommended Issues to Open
| Severity | Title | Type |
|----------|-------|------|
| P1 | Invariant violated: hard delete in user service | type:bug |
| P2 | High-churn file without tests: billing.ts | type:debt |

## Comparison to Previous Audit
| Metric | Previous | Current | Trend |
|--------|----------|---------|-------|
| Invariant violations | 0 | 1 | ↑ worse |
```

---

## Phase 9 — ISSUES

Auto-open for P0 and P1. Show P2 as recommendations (don't auto-create).

Issue frontmatter for audit-sourced issues:
```markdown
---
id: ISS-XXX
title: "<finding title>"
type: type:debt | type:bug
priority: P1
labels: [type:debt, domain:X, status:needs-triage, agent:human-only]
affects: [src/path/to/file.ts]
source: docs/reports/audit-<date>.md
created: <YYYY-MM-DD>
---
```

P0: write issue + `<!-- ESCALATE: P0 -->`, tell user immediately.
Security smells: never auto-open, always require human decision.

```bash
git add issues/ISS-N*.md docs/reports/audit-<date>.md
git commit -m "audit(<date>): arch audit report + N issues opened

Findings: N P0, N P1, N P2, N P3
Auto-opened: N issues (P0+P1)
Report: docs/reports/audit-<date>.md"
```

---

## Phase 10 — HANDOFF

```
✅ Audit complete → docs/reports/audit-<YYYY-MM-DD>.md

  P0  N  ← IMMEDIATE ATTENTION REQUIRED (if >0)
  P1  N  ← N issues opened automatically
  P2  N  ← listed in report, not auto-opened
  P3  N  ← noted in report only

Most critical finding: <one sentence>

Documentation drift found — CONTEXT.md needs updates:
  → <specific update>

Next audit: 2 weeks (if P1 found) | 1 month (otherwise)
```

---

## Error Cases

**CONTEXT.md missing:** Run minimal audit (churn, TODOs, coverage). Note invariant/ADR checks skipped.
**No ADRs:** Note in report. Recommend establishing ADR practice if project >3 months old.
**Security finding:** No exploit details. Vague location reference. Escalate to human first.
**Large codebase (>10k files):** Scope to recently changed files only.
**No previous audit:** Skip comparison table, note "First audit — no baseline."

---

## Reference

Reads: `CONTEXT.md`, `DECISIONS.md`, `docs/adr/`, `AGENTS.md`, source code, `docs/reports/audit-*.md`
Writes: `docs/reports/audit-<YYYY-MM-DD>.md`, `issues/ISS-NXX-*.md`
Reference: `references/invariant-checks.md` — grep strategies per invariant type
Downstream: `diagnoser`, `triager`, `tdd-agent`
