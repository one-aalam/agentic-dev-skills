---
name: context-updater
description: Keeps the knowledge layer honest. Diffs recent git history against CONTEXT.md and proposes surgical updates — new gotchas, stack changes, shifted invariants, glossary terms that appeared in code but aren't documented. Trigger post-merge, after a significant refactor, or whenever CONTEXT.md feels stale. Never rewrites sections wholesale — only proposes targeted additions and edits, confirmed by the user before writing.
---

# Context Updater Skill

The knowledge layer decays silently. Code moves; `CONTEXT.md` doesn't.
This skill reads what actually changed — in git history, in the codebase, in
dependencies — and produces a minimal, confirmed set of updates that keep
`CONTEXT.md`, `DECISIONS.md`, and `docs/adr/` accurate without noise.

It is the maintenance counterpart to `architect` (which writes decisions as they
form) and `arch-auditor` (which flags large-scale drift). Context-updater runs
the narrow, frequent pass: "what changed since we last looked?"

---

## Inputs Expected

The user provides one of:
- Nothing — skill infers the window from the last `CONTEXT.md` commit
- A time window: "sync context for the last two weeks"
- A git range: "update context since v1.3"
- A specific area: "update context for the auth module"
- A trigger event: "we just merged the billing refactor"

---

## Phase Overview

```
Phase 1 — ANCHOR     Find the last CONTEXT.md update and set the diff window
Phase 2 — DIFF       Read what changed in code, deps, and structure
Phase 3 — COMPARE    Map changes to CONTEXT.md sections; find gaps
Phase 4 — PROPOSE    Show a minimal, specific update proposal to the user
Phase 5 — CONFIRM    User edits, approves, or rejects each item
Phase 6 — WRITE      Apply confirmed updates surgically
Phase 7 — HANDOFF    What was updated, what was skipped, what to watch
```

---

## Phase 1 — ANCHOR

Establish the baseline: when was `CONTEXT.md` last meaningfully updated?

```bash
# When was CONTEXT.md last touched, and what was the commit?
git log --oneline -5 -- CONTEXT.md

# How many commits have landed since then?
CONTEXT_SHA=$(git log --format="%H" -- CONTEXT.md | head -1)
git log --oneline ${CONTEXT_SHA}..HEAD | wc -l

# What does the CONTEXT.md changelog section currently say?
grep -A 20 "## Changelog" CONTEXT.md
```

From this, set the **diff window**: all commits from `CONTEXT_SHA` to `HEAD`.

If the user specified a time window or git range, use that instead. If
`CONTEXT.md` has never been committed (new project), the diff window is all
commits. Note this and proceed.

If the diff window is **zero commits** (CONTEXT.md is current): tell the user
and stop. "CONTEXT.md is up to date — no commits since last update."

---

## Phase 2 — DIFF

Read everything that changed within the diff window. This is purely observational —
no judgements yet.

### 2.1 What files changed

```bash
git diff --name-only ${CONTEXT_SHA}..HEAD | sort
```

Group by area:
- Source files (`src/`, `lib/`, `app/`, `internal/`, etc.)
- Schema / migration files
- Dependency manifests (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`)
- Config files (`.env.example`, CI configs, `Dockerfile`, etc.)
- Test files
- Documentation files (`docs/`, `README.md`)
- Root-level agent files (`AGENTS.md`, `DECISIONS.md`, `docs/adr/`)

### 2.2 What the commits say

```bash
# Commit messages in the window — often contain the best signal
git log --oneline ${CONTEXT_SHA}..HEAD

# Any commits that mention decisions, architecture, or invariants
git log --oneline ${CONTEXT_SHA}..HEAD | \
  grep -i "adr\|decision\|invariant\|breaking\|deprecat\|rename\|restructur\|replac"
```

### 2.3 New terms appearing in code

```bash
# New service/repository/manager names not in the CONTEXT.md glossary
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -oP '[A-Z][a-zA-Z]+(?=Service|Repository|Manager|Handler|Controller)' | \
  sort | uniq > /tmp/new-terms.txt

grep -oP '\*\*[A-Z][a-zA-Z]+\*\*' CONTEXT.md | \
  grep -oP '[A-Z][a-zA-Z]+' | sort > /tmp/existing-terms.txt

comm -23 /tmp/new-terms.txt /tmp/existing-terms.txt
# → terms appearing in code but not documented
```

### 2.4 Dependency changes

```bash
# What packages were added, removed, or had major version bumps?
git diff ${CONTEXT_SHA}..HEAD -- package.json 2>/dev/null | grep "^[+-]" | \
  grep -v "^---\|^+++" | grep '"' | head -30

git diff ${CONTEXT_SHA}..HEAD -- pyproject.toml go.mod Cargo.toml 2>/dev/null | \
  grep "^[+-]" | grep -v "^---\|^+++" | head -30
```

Flag: additions at major version, removals, new packages that imply a stack change.

### 2.5 Structural changes

```bash
# New top-level directories in src/ — new modules or services
git diff ${CONTEXT_SHA}..HEAD --name-only | \
  grep "^src/" | sed 's|src/||' | cut -d/ -f1 | sort | uniq

# Deleted directories — removed modules
git diff ${CONTEXT_SHA}..HEAD --diff-filter=D --name-only | \
  grep "^src/" | sed 's|src/||' | cut -d/ -f1 | sort | uniq
```

### 2.6 New ADRs or DECISIONS.md changes

```bash
git diff ${CONTEXT_SHA}..HEAD -- docs/adr/ DECISIONS.md 2>/dev/null | \
  grep "^+" | grep -v "^+++" | head -30
```

If ADRs were added in this window, they must be reflected in CONTEXT.md.

### 2.7 AGENTS.md changes

```bash
git diff ${CONTEXT_SHA}..HEAD -- AGENTS.md 2>/dev/null
```

Changes to run commands, constraints, or the autonomous allowlist should be
reflected in CONTEXT.md if they represent a significant shift in how the project works.

---

## Phase 3 — COMPARE

Map what was found in Phase 2 against the sections of CONTEXT.md. For each
section, determine: **current**, **stale**, or **missing entry**.

Read CONTEXT.md in full now:

```bash
cat CONTEXT.md
```

Work through each section:

### Tech Stack table
Compare against dependency changes found in 2.4. Flag:
- New major dependency not in the table
- Package listed in table but removed from manifests
- Version number that has changed significantly

### Domain Glossary
Compare against new terms from 2.3. Flag:
- Terms appearing in code not in the glossary
- Terms in the glossary whose definitions may have shifted (check commit messages
  for renames, concept changes, or "this is now called...")

### Architecture Overview
Compare against structural changes from 2.5. Flag:
- New module/service directories not mentioned
- Deleted modules still described
- Component relationships that changed (infer from commit messages and file moves)

### Key Invariants
This is the most sensitive section. Look for:

```bash
# Commits that hint at invariant changes
git log --oneline ${CONTEXT_SHA}..HEAD | \
  grep -i "always\|never\|must\|forbidden\|prohibited\|invariant\|constraint"

# Code that contradicts a stated invariant (spot-check, not exhaustive)
# Load references/drift-heuristics.md for specific patterns
```

Flag: any invariant that code changes suggest may have been relaxed, violated, or
clarified. Do NOT change invariants speculatively — flag them for human confirmation.

### External Dependencies
Compare against new external API calls, service integrations, or env var additions:

```bash
git diff ${CONTEXT_SHA}..HEAD -- .env.example 2>/dev/null | grep "^+" | grep -v "^+++"
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -i "fetch(\|axios\.\|https://\|new.*Client\|new.*SDK" | head -20
```

### Sharp Edges & Gotchas
Look for commits with messages like "fix:", "workaround:", "gotcha", "watch out",
"careful", or "NOTE:" — these often encode gotchas that should be documented:

```bash
git log --oneline ${CONTEXT_SHA}..HEAD | \
  grep -i "fix\|workaround\|gotcha\|careful\|note\|watch"

# Also check for new comments in code that warn about something
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -i "// NOTE\|// WARN\|// GOTCHA\|// CAREFUL\|// HACK\|# NOTE\|# WARN" | head -20
```

### Changelog
The changelog should have an entry for this update window. It almost certainly doesn't.
Note: a changelog entry will always be needed.

---

## Phase 4 — PROPOSE

Produce a **minimal, specific** update proposal. Not a rewrite — individual
targeted changes, each one described precisely.

Format each proposed change as a numbered item:

```
## Proposed CONTEXT.md Updates

N changes identified across M sections.

--- TECH STACK ---

[1] ADD row to Tech Stack table:
    | Redis | Session store | 7.x | Added in ISS-042; replaces in-memory store |
    Reason: redis added to package.json in commit a3f9c21 ("feat: persistent sessions")

[2] UPDATE version in Tech Stack table:
    PostgreSQL: 14 → 15
    Reason: go.mod shows postgres driver bump to v15-compatible in commit b7d1e44

--- DOMAIN GLOSSARY ---

[3] ADD term to Domain Glossary:
    | BillingCycle | A recurring period (monthly/annual) that governs invoice generation | Introduced in src/billing/cycle.ts |
    Reason: BillingCycleService appears in 14 files added this window, not yet documented

[4] UPDATE definition:
    "Session" currently: "A user's authenticated state stored in a cookie"
    Proposed: "A user's authenticated state stored in Redis with a 30-day TTL"
    Reason: session storage changed from cookies to Redis in ISS-042

--- KEY INVARIANTS ---

[5] ADD invariant:
    "Session tokens are always invalidated on password change"
    Reason: commit e2a1f33 ("security: invalidate sessions on password reset") introduced
    this as an explicit requirement. Should be documented so agents don't regress it.

[6] REVIEW — possible invariant relaxation (requires your decision):
    Current invariant: "Records are never hard-deleted"
    Concern: src/admin/cleanup.ts added in this window contains a hard DELETE query.
    Is this an intentional exception for admin cleanup, or a violation?
    → If intentional: update invariant to "Records are never hard-deleted except via
      admin cleanup jobs" and create an ADR for the exception.
    → If a violation: open a type:bug issue.

--- SHARP EDGES ---

[7] ADD to Sharp Edges:
    "BillingCycle.endDate is exclusive (not inclusive) — a cycle from Jan 1 to Feb 1
    does not include Feb 1. Off-by-one errors are common when querying by date range."
    Reason: commit f1c8b20 ("fix: off-by-one in billing date range") + comment in
    src/billing/cycle.ts:47: "// NOTE: endDate is exclusive, not inclusive"

--- ARCHITECTURE OVERVIEW ---

[8] ADD to Architecture Overview:
    "Session management is handled by a Redis-backed SessionService in src/sessions/,
    which wraps all token issuance and validation."
    Reason: new src/sessions/ directory with 8 files added in this window, not mentioned.

--- CHANGELOG ---

[9] ADD changelog entry (always required):
    | <today> | Session storage migrated to Redis; BillingCycle domain concept introduced;
                session invalidation on password change added as invariant | context-updater |

---

Confirm each item:
  [y] accept as-is
  [e] edit before writing
  [n] skip
  [!] flag as needing human decision (invariant changes, deletions)

Or: "accept all" / "accept all except N"
```

### Proposal rules

- **Never propose deleting** a glossary term, invariant, or architecture note
  without explicit user confirmation. Deletion is irreversible; a stale entry
  is merely noisy.
- **Never rewrite a section** — only add rows, append list items, or propose
  specific word changes.
- **Flag, don't decide** on anything that looks like an invariant change. These
  require human judgement.
- **Maximum signal, minimum noise.** If a change is trivial (a version patch bump,
  a test file rename), don't propose it. The changelog entry should capture the
  window; individual items should only be things that meaningfully affect how
  agents understand the project.

---

## Phase 5 — CONFIRM

Wait for the user's response on each item. Three outcomes per item:

**Accepted** — write as proposed.

**Edited** — user pastes a corrected version. Use that instead.

**Skipped** — note it was considered and skipped (don't re-propose on next run).

**Flagged** — user says this needs more thought. Note it in the handoff as
"requires decision" without writing anything.

For invariant changes (item type `[6]`-style): do not proceed until the user
has explicitly chosen a path. Offer the two options clearly, wait for a choice.

---

## Phase 6 — WRITE

Apply confirmed changes surgically. Each change touches exactly the section it
belongs to — nothing else moves.

### Appending to a table (glossary, tech stack, external deps)

Find the last row of the table and append after it. Preserve column alignment.

### Appending to a list (invariants, sharp edges)

Find the section header and append a new `- <item>` at the end of the list.

### Updating a specific value (version number, definition word change)

Make the minimum edit. Show the before/after in the commit message.

### Adding to Architecture Overview

Append to the relevant paragraph or add a new short paragraph. Do not restructure
existing content.

### Changelog entry

Always append to the **top** of the changelog table (newest first):

```markdown
| <YYYY-MM-DD> | <summary of what changed in this window> | context-updater |
```

### Commit

```bash
git add CONTEXT.md
git commit -m "docs(context): sync CONTEXT.md with changes since <CONTEXT_SHA_SHORT>

Window: <N> commits (<date range>)
Updates:
  - Tech Stack: <what changed>
  - Glossary: added <N> terms (<list>)
  - Invariants: added <N>, flagged <N> for review
  - Sharp Edges: added <N>
  - Architecture: <what changed>

Skipped: <N> items (noted in handoff)
Decisions deferred: <N> items (see handoff)"
```

Ask user before committing.

---

## Phase 7 — HANDOFF

```
✅ CONTEXT.md updated

  Window:   <N> commits (<from date> → today)
  Written:  <N> changes across <M> sections
  Skipped:  <N> (too minor or user declined)
  Deferred: <N> items requiring a decision

Deferred items (need your attention):
  → [6] Possible invariant relaxation in src/admin/cleanup.ts — hard DELETE query.
        Decide: document intentional exception, or open a type:bug issue?

CONTEXT.md is now current as of today. Suggested cadence:
  → Run again after any significant merge or refactor
  → Or on a regular schedule (weekly for active codebases, monthly otherwise)

Downstream: arch-auditor will now have accurate invariants to check against.
```

---

## Trigger Cadence

This skill is most useful when run:

- **Post-merge** of a significant feature branch
- **Post-refactor** (especially renames, restructuring, new services)
- **Before running `architect`** — so the domain model is current
- **Before running `arch-auditor`** — so it checks against accurate invariants
- **On a schedule** — weekly for active teams, monthly for slower-moving projects

The skill is lightweight enough to run frequently. The cost of a run with no
updates is low (it exits early in Phase 1). The cost of stale context is high
(every other skill operates on wrong information).

---

## Error Cases

**CONTEXT.md is missing:** This is a project-bootstrap problem. Stop and tell the
user: "No CONTEXT.md found. Run project-bootstrap to create the knowledge layer
before trying to update it."

**No git history:** Can't determine what changed without commits. Ask the user
to describe what changed manually, then help them write the updates directly.

**Diff window is very large (>200 commits):** Warn the user: "This window covers
N commits — the signal-to-noise ratio may be low. Consider narrowing to a specific
time range or module." Then proceed but apply stricter filtering (only structural
changes, major dep bumps, explicit invariant-adjacent commits).

**Proposed invariant deletion:** Never propose. If analysis suggests an invariant
no longer holds, flag it as "REVIEW — possible stale invariant" and require explicit
user decision before any change.

**Conflicting signals:** Two commits in the window suggest opposite things (e.g.,
a feature was added then removed). Note the conflict explicitly in the proposal:
"Commits X and Y give conflicting signals about [topic] — net change may be zero.
Please verify before I write anything for this item."

---

## Reference

- `references/drift-heuristics.md` — patterns for detecting specific types of
  drift (invariant relaxation signals, architecture change signals, glossary drift)

Reads: `CONTEXT.md`, `DECISIONS.md`, `docs/adr/`, `AGENTS.md`, git history, source code, dep manifests
Writes: `CONTEXT.md` (surgical additions/edits only)

Position in skill chain:
- Run **before** `architect` or `arch-auditor` to ensure they operate on current context
- Triggered **after** significant merges, refactors, or dependency changes
- Complements `architect` (writes decisions as they form) and `arch-auditor`
  (flags large-scale drift) — context-updater is the frequent, narrow maintenance pass
