---
name: spec-extractor
description: Reverse-engineers implicit requirements from an existing codebase — reading source, tests, git history, and API shapes to produce explicit PRDs and CONTEXT.md updates. Use when a project has no PRDs, before a major refactor, when joining a legacy codebase, or when the user says "document what this does", "reverse-engineer the requirements", "what does this module actually do", or "extract the spec from the code". Outputs verified PRDs to docs/prd/ and proposes CONTEXT.md additions. Never invents — only documents what the code provably does.
---

# Spec Extractor Skill

Code is the only specification that is guaranteed to be accurate — it is what
the system actually does, not what someone intended. This skill reads it as a
specification document and works backward to the requirements.

The typical use case is a greyfield project: code exists, behaviour exists,
but no PRDs, no domain glossary, no documented invariants. The spec-extractor
reads tests (what the code guarantees), source (how it does it), git history
(why decisions were made), and API shapes (what the system promises externally)
— then writes what it finds as explicit, reviewable specification documents.

The output is never invented. If the code does X, the spec says X. If the code
does X in some cases and Y in others, the spec says both, flags the inconsistency,
and asks the user which behaviour was intended. Inferences are always marked
`[inferred — verify]`. Contradictions are surfaced, not resolved.

---

## Inputs Expected

The user provides one of:
- Nothing — skill scopes the whole codebase
- A module or directory: "extract the spec for src/billing/"
- A concept: "extract the spec for how authentication works"
- A trigger: "we're about to refactor payments — document it first"
- A file: "what does src/auth/middleware.ts actually do?"
- An issue: "ISS-042 is about BillingCycle — extract the spec before we change it"

If no scope is given, the skill starts with the highest-value module (the one
most touched in git history — highest churn = most business logic) and asks
the user to confirm before proceeding with the whole codebase.

---

## Phase Overview

```
Phase 1 — ORIENT      Read the knowledge layer; understand what's already documented
Phase 2 — SCOPE       Decide what to extract and in what order
Phase 3 — READ TESTS  Extract guarantees from the test suite
Phase 4 — READ SOURCE Understand the implementation and extract behaviour
Phase 5 — READ GIT    Mine commit history for intent and decisions
Phase 6 — READ API    Extract the external contract from routes and schemas
Phase 7 — SYNTHESISE  Assemble findings into draft PRD(s) and CONTEXT.md proposals
Phase 8 — VERIFY      Show to user — flag inferences, surface contradictions
Phase 9 — WRITE       Write confirmed PRDs and CONTEXT.md additions
Phase 10 — HANDOFF    What was extracted, what needs decisions, what's next
```

---

## Phase 1 — ORIENT

Before reading any source code, check what is already documented. The goal is
to find gaps — not to re-extract what already exists.

```bash
# What knowledge layer already exists?
cat CONTEXT.md 2>/dev/null | head -80
ls docs/prd/*.md 2>/dev/null | grep -v TEMPLATE
ls docs/adr/*.md 2>/dev/null
cat DECISIONS.md 2>/dev/null

# What modules/domains are already documented in PRDs?
grep -h "^# PRD:" docs/prd/*.md 2>/dev/null | sed 's/^# PRD: //'

# What terms are already in the glossary?
grep -A 100 "## Domain Glossary" CONTEXT.md 2>/dev/null | grep "^|" | head -30
```

Build a map of what's already known vs. what needs extraction:

```
Already documented (skip or update only):   <list>
Partially documented (extract and augment): <list>
Completely undocumented (primary targets):  <list>
```

If CONTEXT.md is missing entirely: this is a `project-bootstrap` problem — note
it, but proceed with extraction anyway. The output will seed a new CONTEXT.md.

---

## Phase 2 — SCOPE

Decide the extraction order. Work in priority sequence:

### 2.1 Find the highest-value targets

```bash
# Most-touched files in git history (= most business logic)
git log --name-only --format="" | grep -E "\.(ts|py|go|rs|js)$" | \
  grep -v "test\|spec\|mock\|fixture\|generated" | \
  sort | uniq -c | sort -rn | head -20

# Largest source files (= most behaviour to capture)
find src/ -name "*.ts" -o -name "*.py" -o -name "*.go" 2>/dev/null | \
  grep -v "test\|spec\|mock" | \
  xargs wc -l | sort -rn | head -15

# Files with the most test coverage (= most documented behaviour)
find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | \
  sed 's/\.test\.\|\.spec\./\./' | \
  sed 's/tests\//src\//' | sort | head -20
```

### 2.2 Group by domain concept

Don't extract file by file. Group related files into one coherent extraction
that will produce one PRD per business concept:

```
auth/           → "Authentication and Session Management" PRD
billing/        → "Billing and Subscription" PRD
notifications/  → "Notification Delivery" PRD
```

Ask the user to confirm this grouping before proceeding if it's non-obvious.

### 2.3 Set a scope limit

Extracting a whole codebase in one pass produces overwhelming output.
Default: **one domain concept per run**.

If the user wants the whole codebase: proceed concept by concept, committing
a PRD after each before moving to the next.

---

## Phase 3 — READ TESTS

Tests are the most reliable source of specification. A passing test is a
documented guarantee. Start here before reading source.

```bash
# Find all test files for the target module
find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | \
  grep "<module_keyword>" | sort

# Read each test file
cat <test_file>
```

For each test, extract:

**The guarantee format:**
```
Given: <precondition from test setup / beforeEach>
When:  <action from the test (the function called, the event triggered)>
Then:  <assertion — what the test verifies>
```

Work through every `describe`, `it`, `test`, `def test_`, `func Test` block.

**Classification per test:**
- `[happy path]` — normal operation works correctly
- `[error case]` — system behaves correctly when things go wrong
- `[boundary]` — behaviour at the edges of valid input
- `[invariant]` — a constraint that is always enforced
- `[regression]` — a bug that was fixed and should not recur

**Flag patterns that indicate implicit invariants:**

```bash
# Tests that check "should never" or "must always"
grep -rn "never\|always\|must\|forbidden\|prohibited\|invariant" \
  <test_files> | grep -v "// " | head -20

# Tests with "throws" or "rejects" — error guarantees
grep -rn "throws\|rejects\|raises\|assertRaises\|expect.*toThrow" \
  <test_files> | head -20

# Tests for idempotency (calling twice = calling once)
grep -rn "idempotent\|called.*twice\|duplicate" <test_files> | head -10
```

Build a **test coverage map**:
- What behaviours are tested?
- What behaviours appear to have no tests?
- Which tests are so loose they might not be guaranteeing anything?
  (See `references/extraction-patterns.md` for phantom test detection)

---

## Phase 4 — READ SOURCE

With the test guarantees established, read the source to understand the
implementation and find behaviour that tests don't cover.

```bash
# Read the main module files in dependency order
# Start from entry points
cat src/<module>/index.ts 2>/dev/null || cat src/<module>/__init__.py 2>/dev/null

# Read each file in the module
for f in src/<module>/*.ts; do
  echo "=== $f ==="
  cat "$f"
  echo
done
```

For each source file, extract:

### 4.1 Public interface

```bash
# TypeScript — exported functions, classes, types
grep -n "^export " src/<module>/*.ts | head -30

# Python — public functions/classes (not prefixed with _)
grep -n "^def \|^class " src/<module>/*.py | grep -v "def _\|class _" | head -30

# Go — exported identifiers (capitalised)
grep -n "^func [A-Z]\|^type [A-Z]\|^var [A-Z]" src/<module>/*.go | head -30
```

Each exported symbol is a contract. Document: name, inputs, outputs, what it does.

### 4.2 Data shapes

```bash
# TypeScript interfaces and types
grep -n "interface \|type \|schema\|Schema" src/<module>/*.ts | head -20
# Read the full type definitions

# Database schema (if present)
find . -name "*.sql" -o -name "schema.*" -o -name "models.*" 2>/dev/null | head -5
cat <schema_file> | head -100

# Validation rules (often encode requirements)
grep -rn "z\.string\|z\.number\|Joi\.\|yup\.\|validate\|@IsString\|@IsNumber" \
  src/<module>/ | head -20
```

Validation rules are implicit requirements. `z.string().min(8)` means "passwords
must be at least 8 characters" — that's a requirement worth documenting.

### 4.3 Business logic

Look for conditionals that encode rules:

```bash
# If/else blocks — each branch is a requirement
grep -n "if \|else \|switch \|case " src/<module>/*.ts | \
  grep -v "test\|spec\|comment\|//" | head -30

# Error throws — each throw encodes a rule violation
grep -n "throw\|raise\|Error(\|Exception(" src/<module>/*.ts | \
  grep -v "test\|spec" | head -20

# Constants — often encode business rules
grep -n "const \|readonly \|final \|UPPER_CASE" src/<module>/*.ts | \
  grep -v "test\|import" | head -20
```

### 4.4 Implicit invariants in code

```bash
# Guard clauses at function tops — "always check X before proceeding"
grep -n "if (!.*) throw\|if (!.*) return\|if (!.*) reject" \
  src/<module>/*.ts | head -20

# Assertions in source (not tests)
grep -n "assert\.\|console\.assert\|invariant(" src/<module>/*.ts | head -10

# Transaction boundaries — "these operations must be atomic"
grep -rn "transaction\|BEGIN\|COMMIT\|ROLLBACK\|atomic" src/<module>/ | head -10
```

---

## Phase 5 — READ GIT

Git history encodes intent. Commit messages, PR descriptions, and the sequence
of changes often explain *why* the code is the way it is — the missing context
that source alone can't provide.

```bash
# Commits touching the target module
git log --oneline --follow -- src/<module>/ | head -30

# Full commit messages for significant ones
git log --format="%H %s%n%b%n---" -- src/<module>/ | head -200

# Find the commit that introduced a specific function or pattern
git log -S "functionName" --oneline -- src/<module>/

# When was each file last significantly changed?
git log --oneline -1 -- src/<module>/*.ts | head -10
```

Extract from git history:

**Decision signals:**
- Commits starting with `fix:` → a bug was fixed; the fix encodes what the correct behaviour is
- Commits starting with `feat:` → a requirement was added; the message describes it
- Commits with "workaround", "hack", "temp" → likely a sharp edge to document
- Commits that revert another commit → there was a failed attempt worth understanding
- Commits referencing issue numbers → read the linked issue for the original requirement

```bash
# Find commits that reference issues
git log --oneline -- src/<module>/ | grep -oE "ISS-[0-9]+" | sort -u

# Read those issues
for id in <issue_ids>; do
  cat issues/${id}-*.md 2>/dev/null
done
```

**Stability signals:**
```bash
# Lines that have changed frequently (unstable logic)
git log --follow -p -- src/<module>/key-file.ts | \
  grep "^[-+]" | grep -v "^---\|^+++" | \
  sort | uniq -c | sort -rn | head -20
```

Frequently changing lines often encode requirements that are still evolving —
flag them as `[unstable — may not reflect final intent]` in the spec.

---

## Phase 6 — READ API

The external API is the system's public contract. It's often the most reliable
source of requirements because breaking it has immediate, visible consequences.

```bash
# Find route definitions
grep -rn "router\.\|app\.\(get\|post\|put\|patch\|delete\)\|@Get\|@Post\|@Put" \
  src/ --include="*.ts" | grep -v test | head -30

# Read each route handler
cat <route_file>

# OpenAPI / Swagger specs (often the most explicit requirements)
find . -name "openapi*.yml" -o -name "swagger*.yml" -o -name "api.yml" 2>/dev/null
cat <openapi_file> 2>/dev/null | head -100

# GraphQL schemas
find . -name "*.graphql" -o -name "schema.gql" 2>/dev/null
cat <graphql_schema> 2>/dev/null | head -80
```

For each endpoint, extract:

```
Endpoint: POST /api/billing/cycles
Input:    { startDate: string, planId: string }
Output:   { id: string, startDate: string, endDate: string, status: 'active' }
Auth:     Required (inferred from auth middleware presence)
Errors:   400 if startDate is in the past [inferred from validation]
          404 if planId not found [inferred from service code]
          409 if active cycle already exists [inferred from guard clause]
```

---

## Phase 7 — SYNTHESISE

Assemble findings from Phases 3–6 into a coherent draft PRD and CONTEXT.md
proposals. This is the skill's core intellectual work.

### 7.1 Construct the PRD

Group extracted behaviours into a narrative specification:

```markdown
# PRD: <Domain Concept Name> [EXTRACTED]

**Status:** extracted — needs verification
**Extracted from:** src/<module>/, git log, tests/
**Extraction date:** <YYYY-MM-DD>
**Confidence:** high | medium | low
<!-- Confidence is based on test coverage and commit clarity -->

> ⚠️ This PRD was reverse-engineered from the codebase, not written from
> requirements. Items marked [inferred] have not been confirmed by a human.
> Items marked [contradiction] indicate the code behaves inconsistently.
> Review all findings before treating this as authoritative.

---

## What This Does

<2–4 sentences describing what the module/feature does, synthesised from
source reading and git history. Written in business language, not technical.>

## Confirmed Behaviours

Behaviours verified by passing tests — the system provably does these:

### <Behaviour group name>

```gherkin
Given <test setup>
When  <action under test>
Then  <assertion>
```
**Source:** `tests/<path>:line` — `<test description>`
**Confidence:** High (explicit test)

### <Another behaviour group>

```gherkin
Given <precondition>
When  <action>
Then  <outcome>
```
**Source:** `src/<path>:line` — <description of guard clause or validation>
**Confidence:** Medium [inferred from source — no direct test]

## Inferred Behaviours

Behaviours evident from source or git but not covered by tests.
Each must be verified with a human or confirmed by writing a test.

- [inferred] <behaviour> — derived from `src/<path>:line`
- [inferred] <behaviour> — derived from commit <sha>: "<commit message>"

## Contradictions Found

The code behaves inconsistently in these cases. A decision is needed.

### Contradiction 1
- `src/<path>:line` suggests: <behaviour A>
- `src/<other_path>:line` suggests: <behaviour B>
- These contradict each other. Which is correct?

## Implicit Invariants

Patterns in the code that appear to be enforced consistently:

- [candidate invariant] <statement> — enforced in <N> places:
  `src/<path>:line`, `src/<other_path>:line`

These should be added to CONTEXT.md Key Invariants if confirmed.

## Untested Behaviours

Code paths with no corresponding test — risk of regression:

- `src/<path>:line` — <what this code does> — no test found
- `src/<other_path>:line` — <what> — no test found

## External Contract

API endpoints this module exposes:

| Endpoint | Method | Input | Output | Auth | Notes |
|----------|--------|-------|--------|------|-------|
| `/api/<path>` | POST | `{...}` | `{...}` | Required | |

## Sharp Edges Discovered

```
- <Gotcha from git blame / NOTE comments> — `src/<path>:line`
- <Non-obvious behaviour> — discovered in commit <sha>
```

## Open Questions

Things the extraction could not determine — require human knowledge:

| # | Question | Why unclear | Where to look |
|---|----------|-------------|---------------|
| 1 | Is <behaviour> intentional or a bug? | Contradicts <other behaviour> | Original author / issue tracker |
| 2 | What was the original requirement for <edge case>? | No test, commit message unclear | Git blame on `src/<path>:line` |
```

### 7.2 Construct CONTEXT.md proposals

From the extraction, identify what should be added to CONTEXT.md:

**New glossary terms** — domain concepts appearing in code not yet in the glossary:
```
[propose] | <Term> | <Definition inferred from usage> | [inferred — verify] |
```

**Candidate invariants** — patterns enforced consistently across the module:
```
[propose] "<invariant statement>" — enforced in N places (see PRD)
```

**Sharp edges** — NOTE comments and fix commits that encode gotchas:
```
[propose] "<gotcha>" — src/<path>:line
```

**Architecture notes** — if the module's role in the system isn't described:
```
[propose] Add to Architecture Overview: "<module> handles <role> in the system"
```

---

## Phase 8 — VERIFY

Show the draft PRD and CONTEXT.md proposals to the user. Do not write anything
until they have reviewed.

### 8.1 Present the findings

```
## Extraction complete: <Domain Concept Name>

Confidence breakdown:
  ✅ High confidence (test-backed):      N behaviours
  ⚠️ Medium confidence (inferred):       N behaviours
  ❓ Contradictions requiring decision:  N items
  🔴 Untested code paths:               N paths

Draft PRD ready for review: see above
CONTEXT.md proposals: N additions

Before I write anything, please:
1. Confirm the [inferred] items are correct (or correct them)
2. Resolve the contradictions — which behaviour was intended?
3. Mark any behaviours as intentional vs. bugs

You can:
  [accept all]        — write everything as-is
  [accept, except N]  — skip specific items
  [correct N]         — paste the correction for item N
  [it's a bug]        — mark a contradiction as a bug to be fixed
```

### 8.2 Handle contradictions explicitly

For each contradiction, present as a choice:

```
Contradiction 1 — Password validation:

  src/auth/validator.ts:23 enforces minimum 8 characters
  src/auth/legacy-api.ts:89 accepts passwords of any length

  Which is the intended behaviour?
    [A] 8 character minimum — legacy API should be updated (open an issue)
    [B] No minimum — validator is wrong (open an issue)
    [C] Both are intentional — different rules for different contexts (document both)
```

Wait for the user's answer before writing.

### 8.3 Confidence threshold

Do not write a PRD that is predominantly `[inferred]`. If fewer than 60% of
behaviours are test-backed:

> "This extraction has low confidence — N% of behaviours are inferred from source
> rather than verified by tests. I recommend writing tests to confirm the inferred
> behaviours before formalising this as a spec. Should I generate a list of tests
> to write, or proceed with the low-confidence PRD?"

---

## Phase 9 — WRITE

Write confirmed items after user approval.

### 9.1 Write the PRD

```bash
mkdir -p docs/prd
SLUG=$(echo "<concept name>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g')
FILENAME="docs/prd/PRD-${SLUG}-extracted.md"
```

Mark the PRD status as `extracted` (not `draft` or `approved`) to distinguish
it from human-authored PRDs. Add the extraction warning at the top.

Update confirmed status:
- Remove `[inferred]` markers from confirmed items
- Add `[verified by <name>]` where the user confirmed something
- Leave contradictions that were resolved as `[was: <X>, corrected to: <Y>]`
- Leave unresolved items as `[open question — see Open Questions section]`

### 9.2 Write CONTEXT.md additions

Use the same surgical update approach as `context-updater` — append to tables
and lists, never rewrite sections.

Mark each addition with `[spec-extractor — verify]` so future readers know
it came from extraction, not from a live architectural conversation.

### 9.3 Write test stubs for unverified behaviours (optional)

If the user requests it, generate test stubs for the behaviours that have no
tests — the "Untested Behaviours" section of the PRD:

```typescript
// STUB: Generated by spec-extractor — implement and verify
describe('<module name>', () => {
  describe('<behaviour group>', () => {
    it.todo('<behaviour that needs a test>');
    // Inferred from: src/<path>:line
    // Expected: <what the code appears to do>
  });
});
```

Write stubs to `tests/<module>/extracted-behaviours.test.ts` with `.todo` markers
so they don't pollute the test run.

### 9.4 Commit

```bash
git add docs/prd/PRD-${SLUG}-extracted.md CONTEXT.md
# (and test stubs if generated)
git commit -m "docs(spec): extract spec for <concept name>

Source: src/<module>/, git history, test suite
Confidence: N% test-backed behaviours
Contradictions resolved: N
Open questions: N

Verified by: <user>
Extraction tool: spec-extractor skill"
```

Ask user before committing.

---

## Phase 10 — HANDOFF

```
✅ Extraction complete

  Concept:       <Domain Concept Name>
  PRD:           docs/prd/PRD-<slug>-extracted.md
  Confidence:    N% (N test-backed / N inferred / N contradictions)

  CONTEXT.md additions proposed:
    → Glossary:    N new terms
    → Invariants:  N candidate invariants
    → Sharp Edges: N gotchas
    → Architecture: N notes

  Still needs attention:
    → N open questions (listed in PRD)
    → N unverified [inferred] items
    → N untested code paths (risky before refactor)

<If low confidence:>
  ⚠️ This spec has low confidence. Before using it as the basis for a refactor,
  write tests for the N inferred behaviours to confirm them. The test stubs
  are in tests/<module>/extracted-behaviours.test.ts.

Next steps:
  → Run prd-writer to formalise this into a proper approved PRD once verified
  → Run context-updater to add the proposed CONTEXT.md items (after review)
  → Run tdd-agent to add tests for untested behaviours before the refactor
  → Run the extractor again on the next domain concept: <suggested_next>
```

---

## Confidence Scoring

Every extracted item gets a confidence level. Use this rubric:

| Evidence | Confidence | Label |
|---|---|---|
| Explicit test with specific assertion | High | ✅ (no marker) |
| Multiple tests across different scenarios | High | ✅ |
| Source code guard clause with clear intent | Medium | `[inferred]` |
| Commit message describing intent | Medium | `[inferred from commit <sha>]` |
| Consistent pattern across multiple files | Medium | `[pattern — not explicitly tested]` |
| Single code path, no test, unclear intent | Low | `[inferred — low confidence]` |
| Contradicts other evidence | None | `[contradiction — decision needed]` |

The overall PRD confidence = (High items) / (Total items). Display this clearly.

---

## Scope Guidance

### When to extract one module vs. a full codebase

**One module (recommended):** Always start with one. It's fast, produces a
reviewable output, and the user can correct calibration before running more.

**Full codebase:** Only when the user explicitly requests it and the codebase
is small (<20 source files). Anything larger: run concept by concept.

### How to sequence a full codebase extraction

```bash
# Suggested order (most value first):
# 1. Core domain objects (the nouns: User, Order, Payment, etc.)
# 2. Core workflows (the verbs: checkout, authenticate, notify, etc.)
# 3. Integration points (external APIs, webhooks, queues)
# 4. Infrastructure (auth middleware, error handling, logging)
```

### Pre-refactor extraction

When called "before a refactor", the goal changes: capture current behaviour
comprehensively, including bugs and quirks, so the refactor can make conscious
decisions about what to preserve and what to fix. Mark all discovered bugs with
`[current behaviour — may be a bug]` rather than removing them from the spec.

---

## Error Cases

**No tests exist at all:** All extractions will be `[inferred]`. Tell the user:
"This module has no tests. The extraction will be entirely inferred from source —
confidence will be low across the board. Strongly recommend writing tests before
treating this as a spec." Proceed if the user confirms.

**Tests exist but are all phantom tests** (see `references/extraction-patterns.md`):
Tell the user: "The existing tests are mostly existence checks (`toBeDefined()`)
that don't verify actual behaviour. They don't provide reliable spec evidence.
Treating source as primary evidence instead." Proceed with source-based extraction.

**Massive module (>1000 lines):** Scope to the public interface only. Read the
exported functions and their signatures, not the full implementation. Note clearly:
"This module is large. Extracted the public interface only — internal behaviour
was not examined. Run again with a specific function name to go deeper."

**Contradictions that can't be resolved without domain knowledge:** Document them
clearly in the Open Questions section and move on. Never resolve a contradiction
by picking the "more likely" option without user confirmation.

**Significant git history (>500 commits on module):** Sample rather than read all.
Read the first 5 commits (origin intent), the last 10 (current intent), and any
commits with "breaking", "refactor", "redesign" in the message.

---

## Reference

- `references/extraction-patterns.md` — language-specific patterns for extracting
  guarantees from tests, finding implicit invariants in source, detecting phantom
  tests, and reading API contracts from different frameworks

Reads: source code, test files, git log, issues/, CONTEXT.md, docs/prd/, API specs
Writes: `docs/prd/PRD-<slug>-extracted.md`, `CONTEXT.md` (proposed additions, confirmed)

Position in skill chain:
- Run **before** a major refactor to capture current behaviour
- Run **on greyfield projects** that have code but no requirements docs
- Output feeds **prd-writer** (to formalise the extracted spec)
- Output feeds **context-updater** (proposed CONTEXT.md additions)
- Untested behaviours feed **tdd-agent** (to write confirming tests)
- Contradictions may feed **architect** (to resolve design questions)
