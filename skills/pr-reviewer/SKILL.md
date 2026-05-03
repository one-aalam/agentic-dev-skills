---
name: pr-reviewer
description: Reviews a pull request against the project's domain model, invariants, ADRs, and the originating issue's acceptance criteria. Not a style linter — an architectural reviewer. Trigger on: any request to review a PR or diff, "check this against our docs", "does this PR look right", or before merging any significant change. Produces a structured review with pass/flag/block findings. Never approves blindly.
---

# PR Reviewer Skill

The skill chain ends at "open a PR" — nothing reviews it. This skill closes
that gap. It reads a PR diff against the project's knowledge layer and the
originating issue, then produces a structured review focused on what matters:
invariant compliance, acceptance criteria coverage, architectural consistency,
naming, and scope. Not style. Not formatting. Those belong to a linter.

The reviewer has three verdicts per finding:

- **✅ Pass** — deliberate, correct, consistent with the knowledge layer
- **⚠️ Flag** — worth discussing; the author should respond; not a blocker
- **🚫 Block** — must be resolved before merge; clear reason given

---

## Inputs Expected

The user provides one of:
- A PR number: "review PR #42"
- A branch name: "review the feature/billing-cycle branch"
- A raw diff pasted inline
- A GitHub PR URL
- "Review the open PR" — skill finds the most recently opened unmerged PR

If the PR references an issue, the skill reads it automatically. If no issue
is referenced, the skill proceeds but notes the gap.

---

## Phase Overview

```
Phase 1 — ORIENT       Read the knowledge layer and the originating issue
Phase 2 — READ PR      Understand the diff — scope, shape, what changed
Phase 3 — AC CHECK     Does the PR satisfy the issue's acceptance criteria?
Phase 4 — INVARIANTS   Does the PR respect all Key Invariants?
Phase 5 — ADR CHECK    Does the PR follow accepted architectural decisions?
Phase 6 — DOMAIN CHECK Is domain language used correctly? Any naming drift?
Phase 7 — SCOPE CHECK  Is the PR doing exactly what the issue asked — no more?
Phase 8 — TEST CHECK   Are the tests meaningful and sufficient?
Phase 9 — WRITE        Produce the structured review
Phase 10 — HANDOFF     Verdict summary + recommended next action
```

---

## Phase 1 — ORIENT

Read the full knowledge layer before looking at a single line of diff.

```bash
cat CONTEXT.md      # invariants, glossary, architecture, sharp edges
cat AGENTS.md       # commit convention, what agents may do autonomously
cat DECISIONS.md    # ADR index
for f in docs/adr/*.md; do
  grep -l "Status.*accepted" "$f" 2>/dev/null && cat "$f"
done               # all accepted ADRs
```

Extract and hold in mind:
- Every **Key Invariant** — checked one by one in Phase 4
- The **Domain Glossary** — terms the PR must use correctly
- The **accepted ADRs** — patterns and decisions the PR must follow
- **Sharp Edges** — known problem areas that may be relevant to this diff
- **Commit convention** from AGENTS.md — did the PR author follow it?

Then read the originating issue:

```bash
# If PR references an issue number
cat issues/ISS-XXX-*.md 2>/dev/null
# or: gh issue view NNN

# If no issue reference, note it
```

Extract from the issue:
- **Acceptance Criteria** — the ground truth for Phase 3
- **Affected Files hint** — expected scope for Phase 7
- **Implementation Notes** — constraints, relevant ADRs, gotchas
- **Out of Scope** — things the PR must NOT do
- **PRD reference** — read it if the issue links one

If no issue exists: note "No linked issue — acceptance criteria check will be
based on PR description only." Proceed with reduced confidence on Phase 3.

---

## Phase 2 — READ PR

Get the full diff and understand its shape before evaluating it.

```bash
# Via gh CLI
gh pr diff <PR_NUMBER>
gh pr view <PR_NUMBER>

# Or for a branch
git diff main...<branch-name>
git log main...<branch-name> --oneline
```

Build a mental model:

**What changed:**
- List every file modified, created, or deleted
- Note which modules/domains they belong to
- Identify the entry point (the highest-level change that caused the rest)

**Shape of the change:**
- How many files? (>5 in a single PR is a scope flag)
- Are the changes cohesive? (one logical purpose, or multiple?)
- What is the ratio of test code to production code?
- Are any generated files modified directly? (should never happen)

**Commit structure:**
- How many commits?
- Do commit messages follow the project convention from AGENTS.md?
- Is there a failing-test commit followed by an implementation commit?
  (required if `tdd-agent` authored this or TDD is a project standard)

Do not form judgements yet. Just read and map.

---

## Phase 3 — ACCEPTANCE CRITERIA CHECK

The most important phase. Every acceptance criterion from the linked issue must
be satisfied by the diff. Work through them one by one.

For each criterion:

```
AC: "Given a valid session token, when the user makes a request,
     then the request proceeds and user context is attached"

Evidence in diff:
  ✅ src/auth/middleware.ts — verifyToken() called on every request
  ✅ tests/auth/middleware.test.ts — "authenticates request with valid token" passes

Verdict: ✅ Pass
```

```
AC: "Given an expired token, when the user makes a request,
     then a 401 is returned with error code AUTH_TOKEN_EXPIRED"

Evidence in diff:
  ✅ src/auth/middleware.ts:47 — returns 401 for expired tokens
  ⚠️ Error code is "TOKEN_EXPIRED" not "AUTH_TOKEN_EXPIRED" as specified
     → Author should clarify: intentional divergence or typo in AC?

Verdict: ⚠️ Flag — error code mismatch with acceptance criterion
```

```
AC: "Session tokens are invalidated on logout"

Evidence in diff:
  🚫 No evidence found. Neither src/auth/ nor tests/auth/ contains
     logout-related token invalidation. This AC is not addressed.

Verdict: 🚫 Block — acceptance criterion not implemented
```

**Rules:**
- Every AC must have an explicit verdict. "Not checked" is not acceptable.
- A missing implementation is always **Block**.
- An implementation that diverges from the spec is **Flag** unless it's a
  security or data integrity criterion — those are **Block**.
- If the PR description adds or modifies acceptance criteria vs. the issue,
  note the divergence and flag it for author clarification.

---

## Phase 4 — INVARIANT CHECK

For each Key Invariant in CONTEXT.md, check whether the diff respects it.

Load `references/review-checks.md` for specific grep patterns per invariant type.

Work through each invariant explicitly:

```
Invariant: "Records are never hard-deleted"

Diff check:
  grep "\.delete\(\|DELETE FROM" in changed files → src/users/cleanup.ts:23
  git show HEAD:src/users/cleanup.ts | sed -n '20,26p'
  → Line 23: await db.query("DELETE FROM users WHERE id = $1", [userId])

Verdict: 🚫 Block — hard delete of user record violates Key Invariant.
  Either: (a) change to soft-delete, or (b) get an ADR accepted that
  explicitly carves out this exception.
```

```
Invariant: "All mutations go through the event bus"

Diff check:
  grep "prisma\.\|repository\." in changed files → all writes go through
  UserRepository.save() which internally publishes to the event bus

Verdict: ✅ Pass
```

**Rules:**
- Every invariant gets a verdict. Not just the ones that seem relevant.
- An invariant violation is always **Block** — no exceptions.
- If the diff *adds* a new invariant implicitly (a pattern that should
  always hold), note it as a suggestion to run `context-updater`.

---

## Phase 5 — ADR CHECK

For each accepted ADR, check whether the diff follows the decision.

```bash
# Find ADRs relevant to the changed files/domains
grep -l "<domain keyword>" docs/adr/*.md
```

Work through each relevant ADR:

```
ADR-003: "All new API endpoints must require auth middleware"

Diff check:
  New route found: POST /api/billing/cycles in src/routes/billing.ts:14
  Check for auth middleware: grep "authMiddleware\|requireAuth\|verifyToken" src/routes/billing.ts
  → Found: router.post('/cycles', authMiddleware, billingController.createCycle)

Verdict: ✅ Pass
```

```
ADR-007: "Use the repository pattern — no direct DB access outside repositories"

Diff check:
  src/billing/cycle.ts:89 — prisma.billingCycle.create({ ... }) called directly
  This file is not a repository file.

Verdict: 🚫 Block — direct Prisma call outside repository layer violates ADR-007.
```

**Rules:**
- Only check ADRs with `Status: accepted`. Proposed ADRs are not binding.
- An ADR violation is always **Block** unless the PR is explicitly superseding
  the ADR (in which case check that the new ADR is also in the diff).
- If the PR introduces a pattern that should be an ADR but isn't, note it as
  a **Flag** with a suggestion to run `architect` to formalise it.

---

## Phase 6 — DOMAIN LANGUAGE CHECK

Read every new identifier, variable name, function name, class name, and comment
in the diff. Check each against the Domain Glossary in CONTEXT.md.

Three cases:

**In glossary, used correctly:**
```
BillingCycle used as the domain object for recurring billing periods → ✅
```

**In glossary, used inconsistently:**
```
Function named createSubscriptionPeriod() — the glossary term is BillingCycle,
not SubscriptionPeriod. These appear to mean the same thing.
→ ⚠️ Flag: naming drift. Use the canonical glossary term.
```

**New term not in glossary, appearing significantly:**
```
InvoiceRun used in 6 new files but not defined in CONTEXT.md glossary.
→ ⚠️ Flag: new domain concept introduced without documentation.
  Recommend running context-updater after merge to add it to the glossary.
```

**Rules:**
- Naming drift from the glossary is always at least **Flag**.
- A new significant concept with no glossary entry is **Flag** (not Block —
  the code works, but the knowledge layer needs updating).
- Comment language inconsistencies (referring to the right concept by the
  wrong name) are **Flag** — they mislead future agents reading the code.

---

## Phase 7 — SCOPE CHECK

The PR should do exactly what the issue asked — no more, no less.

### Scope creep (doing more than asked)

```bash
# Files changed that weren't in the issue's "Affected Files" hint
git diff --name-only main...<branch> | grep -v "<expected files>"
```

For each unexpected file change:
- Is it a necessary side effect? (e.g., updating an import that moved)
- Is it a "while I was here" improvement?
- Is it a separate feature bundled into this PR?

Necessary side effects → **Pass** with a note.
"While I was here" cleanups → **Flag** (should be a separate PR or issue).
Bundled separate feature → **Block** (must be separated — it obscures the
review surface and makes rollback harder).

### Under-delivery (doing less than asked)

Compared against the AC check from Phase 3. If any ACs are unimplemented,
that's already a **Block** from Phase 3. No double-counting needed here.

### Out-of-scope work

Check the issue's "Out of Scope" section explicitly:

```
Issue Out of Scope: "Does not include invoice generation — only the BillingCycle
data model and CRUD operations"

Diff check: src/billing/invoice-generator.ts added in this PR
→ 🚫 Block — explicitly out-of-scope work included. Separate this into its own issue.
```

---

## Phase 8 — TEST CHECK

Tests are evidence that the implementation is correct. A PR without tests for
new behaviour is a PR without proof.

### Coverage of new behaviour

For every new function, method, or code path in the diff:

```bash
# Find new functions in the diff
git diff main...<branch> -- src/ | grep "^+" | \
  grep -E "function |const .* = |async " | grep -v test

# Find corresponding tests
git diff main...<branch> -- tests/ "*.test.*" "*.spec.*"
```

Map each new behaviour to a test case. Flag anything untested.

### Test quality (not just quantity)

Read the new tests. Apply these checks:

| Check | Pass | Flag | Block |
|---|---|---|---|
| Tests have assertions | All do | Some missing | Tests with no assertions |
| Tests name the behaviour | Clear sentences | Vague names ("it works") | — |
| Tests test behaviour not implementation | Asserting outputs | Asserting spy calls only | — |
| Edge cases covered | AC edge cases tested | Some missing | No edge cases at all |
| Existing tests still pass | Assumed green | — | PR description says tests were skipped |

**The phantom test:** A test that would pass even if the code it covers was deleted
is not a test — it's false confidence. Look for tests where the assertion is so
loose it can't fail:

```typescript
// ⚠️ Flag — this passes regardless of what createBillingCycle does
it('creates a billing cycle', async () => {
  const result = await createBillingCycle(input);
  expect(result).toBeDefined();
});

// ✅ Pass — this would fail if the logic is wrong
it('creates a billing cycle with correct end date', async () => {
  const result = await createBillingCycle({ startDate: '2024-01-01', months: 1 });
  expect(result.endDate).toBe('2024-02-01');
  expect(result.status).toBe('active');
});
```

### The TDD signal

If `tdd-agent` authored this PR, or if the project follows TDD:

```bash
git log main...<branch> --oneline
```

Look for: a `test(ISS-XXX):` commit before the `feat(ISS-XXX):` commit.
If the failing-test commit is missing → **Flag** (process not followed).

---

## Phase 9 — WRITE

Produce the structured review. Format it as a document the PR author can act on.

```markdown
# PR Review: <PR title> (#NNN)

**Branch:** <branch-name>
**Issue:** ISS-XXX — <issue title>
**Reviewer:** pr-reviewer skill
**Date:** <YYYY-MM-DD>

## Verdict

🚫 BLOCK — N blocking finding(s) must be resolved before merge.

<!-- or -->

⚠️ APPROVE WITH FLAGS — No blockers. N flag(s) to address or respond to.

<!-- or -->

✅ APPROVE — All checks pass. Ready to merge.

---

## Acceptance Criteria

| # | Criterion | Verdict | Notes |
|---|-----------|---------|-------|
| 1 | <AC text> | ✅ Pass | Implemented in `src/...` |
| 2 | <AC text> | ⚠️ Flag | Error code mismatch — see below |
| 3 | <AC text> | 🚫 Block | Not implemented |

---

## Invariants

| Invariant | Verdict | Evidence |
|-----------|---------|----------|
| Records never hard-deleted | 🚫 Block | `src/users/cleanup.ts:23` — raw DELETE query |
| All mutations through event bus | ✅ Pass | All writes via UserRepository |

---

## ADR Compliance

| ADR | Verdict | Evidence |
|-----|---------|----------|
| ADR-003 auth on all endpoints | ✅ Pass | authMiddleware present on new route |
| ADR-007 repository pattern | 🚫 Block | `src/billing/cycle.ts:89` — direct Prisma call |

---

## Domain Language

| Finding | Verdict | Recommendation |
|---------|---------|----------------|
| `createSubscriptionPeriod()` | ⚠️ Flag | Glossary term is `BillingCycle` — rename |
| `InvoiceRun` (6 new files) | ⚠️ Flag | New concept not in glossary — run context-updater after merge |

---

## Scope

| Finding | Verdict | Notes |
|---------|---------|-------|
| `src/billing/invoice-generator.ts` | 🚫 Block | Explicitly out of scope in ISS-XXX |
| Minor import reorganisation in `src/auth/` | ✅ Pass | Necessary side effect of move |

---

## Tests

| Finding | Verdict | Notes |
|---------|---------|-------|
| New BillingCycle CRUD covered | ✅ Pass | 8 tests, all with specific assertions |
| `createBillingCycle` end date test | ⚠️ Flag | Assertion is `toBeDefined()` — too loose |
| No test for end date exclusivity | ⚠️ Flag | Sharp edge documented in CONTEXT.md — should be tested |
| Failing-test commit present | ✅ Pass | `test(ISS-XXX):` commit precedes `feat(ISS-XXX):` |

---

## Blocking Findings (must resolve before merge)

### 🚫 B1 — Acceptance criterion not implemented
**AC:** "Session tokens are invalidated on logout"
**Location:** Not found in diff
**Required action:** Implement logout token invalidation, or update the issue if
this AC was intentionally deferred (and link the follow-up issue).

### 🚫 B2 — Key Invariant violated
**Invariant:** "Records are never hard-deleted"
**Location:** `src/users/cleanup.ts:23`
**Required action:** Change to soft-delete, or open an ADR to create a documented
exception for admin cleanup operations before merging this.

### 🚫 B3 — ADR-007 violated
**Location:** `src/billing/cycle.ts:89`
**Required action:** Move DB call into a BillingCycleRepository method.

### 🚫 B4 — Out-of-scope file included
**File:** `src/billing/invoice-generator.ts`
**Required action:** Remove from this PR and open a separate issue for invoice
generation, referencing the parent PRD.

---

## Flagged Findings (respond or resolve)

### ⚠️ F1 — Naming drift
`createSubscriptionPeriod()` should be `createBillingCycle()` per the domain glossary.
Small change but matters for agent-readability going forward.

### ⚠️ F2 — New domain concept undocumented
`InvoiceRun` appears significantly. Not in CONTEXT.md glossary. Run context-updater
after merge to add a definition.

### ⚠️ F3 — Loose test assertion
`src/billing/__tests__/cycle.test.ts:34` — `expect(result).toBeDefined()` would pass
even if `createBillingCycle` returned a completely wrong object. Tighten to assert
specific fields.

### ⚠️ F4 — End date exclusivity not tested
CONTEXT.md Sharp Edges documents that `BillingCycle.endDate` is exclusive.
This edge case is not covered by the new tests — agents may regress it.

---

## Notes for the Author

<Any additional context, suggestions, or references that don't fit the structured
sections. Keep this short — the tables above should do most of the work.>
```

---

## Phase 10 — HANDOFF

After writing the review, surface a compact summary:

```
PR Review complete: #NNN — <title>

Overall verdict: 🚫 BLOCK (N blockers) | ⚠️ APPROVE WITH FLAGS (N flags) | ✅ APPROVE

Blockers (must fix before merge):
  B1: AC not implemented — session logout invalidation
  B2: Invariant violated — hard delete in src/users/cleanup.ts:23
  B3: ADR-007 violated — direct DB call in src/billing/cycle.ts:89
  B4: Out-of-scope file — src/billing/invoice-generator.ts

Flags (respond or resolve):
  F1: Naming drift — createSubscriptionPeriod → createBillingCycle
  F2: InvoiceRun not in glossary — run context-updater after merge
  F3: Loose test assertion at cycle.test.ts:34
  F4: End date exclusivity not tested

Full review: <path if written to file, or inline above>
```

---

## Verdict Escalation Rules

These rules override individual phase judgements:

| Condition | Override verdict |
|---|---|
| Any unimplemented AC | 🚫 Block |
| Any Key Invariant violated | 🚫 Block |
| Any accepted ADR violated | 🚫 Block |
| Out-of-scope work included | 🚫 Block |
| Zero tests for new behaviour | 🚫 Block |
| Security-related finding | 🚫 Block + escalate |
| Naming drift from glossary | ⚠️ Flag (minimum) |
| New concept not in glossary | ⚠️ Flag |
| Loose test assertions | ⚠️ Flag |
| Missing edge case test | ⚠️ Flag |
| Commit convention not followed | ⚠️ Flag |

**There is no "approve with blockers."** A PR with a blocker does not get approved.

---

## What This Skill Does NOT Review

To keep scope clear:

- **Style / formatting** — that's a linter's job (ESLint, Prettier, Black, etc.)
- **Performance** — unless a Key Invariant addresses it or an ADR mandates it
- **Code readability** (subjective opinions) — only naming against the domain glossary
- **Line-by-line logic correctness** — that requires understanding business rules
  not captured in CONTEXT.md; flag it for human review if something looks wrong
- **Infrastructure / deployment** — out of scope unless it violates a stated invariant

If something outside this scope looks wrong, note it briefly as an informational
comment at the end of the review — not as a Flag or Block.

---

## Error Cases

**No linked issue:** Review proceeds but Phase 3 (AC Check) uses PR description
only. Note clearly: "No linked issue found — AC check based on PR description.
Accuracy may be lower. Link the issue and re-run for a complete review."

**PR diff is very large (>20 files):** Warn the user: "This PR touches N files.
Large PRs are hard to review accurately and hard to roll back. Consider asking
the author to split it." Then proceed with the review, noting reduced confidence.

**PR has no tests at all for new behaviour:** This is always 🚫 Block. State it
once clearly and don't enumerate which specific behaviours are untested — the fix
is to add tests, not to debate which ones.

**Security finding in diff:** Stop. Do not document the finding in detail.
Tell the user: "I found what looks like a security concern in `<path>`. I've
stopped the review. A human should examine this before any further action."

**No CONTEXT.md or ADRs:** Review can only check AC coverage and scope. Note
which phases were skipped and why. Recommend running `project-bootstrap`.

**Diff contains only test changes:** Phases 4–7 are largely not applicable.
Focus on Phase 3 (do tests actually cover the stated criteria?) and Phase 8
(are the tests themselves well-written?). Note the reduced scope.

---

## Reference

- `references/review-checks.md` — per-invariant-type diff grep patterns, ADR
  check patterns, and test quality heuristics

Reads: `CONTEXT.md`, `AGENTS.md`, `docs/adr/`, `issues/ISS-XXX-*.md`, `docs/prd/`, PR diff
Writes: review output (inline or to `docs/reports/review-<PR>-<date>.md` if requested)

Position in skill chain:
- Runs **after** `tdd-agent` opens a PR
- Runs **before** human merge approval
- Findings may feed back into: `triager` (new issues from blockers),
  `context-updater` (new glossary terms flagged), `architect` (new ADR needed)
