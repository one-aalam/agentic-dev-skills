---
name: tdd-agent
description: >
  Implement a feature or fix test-first using strict red → green → refactor discipline.
  Trigger this skill when a user wants to implement an issue, write code for a feature,
  fix a bug with tests first, or says things like "implement ISS-XXX", "build this
  feature", "write the code for", "fix this bug", "make this test pass", "implement
  this agent:ready issue", "start on this ticket", or "code this up". Also trigger
  when a user wants to write tests for existing untested code, add test coverage, or
  practice TDD on any unit of work. This skill requires an issue with acceptance criteria
  — if none exists, it will ask the user to run prd-writer or triager first. Reads
  AGENTS.md and CONTEXT.md religiously. Never skips the failing-test commit.
  Pairs with issue-planner (upstream) and triager / arch-auditor (downstream review).
---

# TDD Agent Skill

Implements issues test-first. Follows strict red → green → refactor. Never writes
implementation before the failing test exists. Never modifies test assertions to make
a test pass.

---

## Inputs Expected

The user provides one of:
- An issue ID: "implement ISS-042"
- A path to an issue file: "issues/ISS-042-auth-refresh.md"
- A GitHub issue number: "implement GH#142"
- A natural language task with clear acceptance criteria pasted inline
- "Write tests for `src/path/to/module.ts`" (test-writing mode, no impl)

If no acceptance criteria are available anywhere, stop:
> "I need acceptance criteria to write meaningful tests. Run issue-planner on the
> PRD, or paste the acceptance criteria and I'll proceed."

---

## Phase Overview

```
Phase 1 — ORIENT        Read AGENTS.md, CONTEXT.md, the issue
Phase 2 — ASSESS        Validate the issue is truly agent:ready
Phase 3 — EXPLORE       Understand the affected code area
Phase 4 — TEST PLAN     Map acceptance criteria → test cases
Phase 5 — RED           Write failing tests, commit
Phase 6 — GREEN         Implement minimum code, commit
Phase 7 — REFACTOR      Clean up, commit
Phase 8 — PR            Open pull request with full context
Phase 9 — UPDATE ISSUE  Mark issue needs-review
```

---

## Phase 1 — ORIENT

Read these files before touching any source code. This is not optional.

```bash
cat AGENTS.md
cat CONTEXT.md
cat issues/ISS-XXX-*.md  # or: gh issue view NNN
cat docs/reports/diagnosis-<id>.md 2>/dev/null
cat docs/prd/PRD-<slug>.md 2>/dev/null
```

Extract from AGENTS.md: run commands, autonomous allowlist, human-first requirements, commit convention.
Extract from CONTEXT.md: Key Invariants (must not be violated), Domain Glossary (use exact terms), Sharp Edges.
Extract from issue: Acceptance Criteria, Affected Files, Implementation Notes, depends-on.

---

## Phase 2 — ASSESS

### Hard stops — pause and ask the user

```
□ depends-on issues all merged?
□ Priority is NOT P0?              (P0 = human oversight required)
□ Type is NOT type:security?       (security = human oversight required)
□ No blocking open questions?
□ Acceptance criteria are specific and falsifiable?
□ At least one file path hinted?
□ Implementation would touch ≤5 files?
□ Within AGENTS.md "may do autonomously" list?
```

If any box is unchecked, stop and tell the user specifically which one failed.

### Soft warnings — note but continue

- `agent:partial`: note what's partial and proceed cautiously
- Issue older than 2 weeks: codebase may have drifted
- CONTEXT.md last updated >30 days ago: invariants might be stale

---

## Phase 3 — EXPLORE

Read the existing code before writing tests:

```bash
cat <affected_file_1>
cat <affected_file_2>
find . -name "*.test.*" -path "*<module>*" 2>/dev/null | head -5
cat <closest_test_file>
<TEST_COMMAND> --testPathPattern "<module>"   # confirm existing tests pass
```

Determine: where new code lives, what test utilities exist, naming conventions, current public interface.

---

## Phase 4 — TEST PLAN

Map every acceptance criterion to specific test cases before writing code:

```
[AC1] "Given valid token, when request made, then authenticated"
  → test: "authenticates request with valid session token"
  → test: "attaches user context to authenticated request"

[AC2] "Given expired token, when request made, then 401 returned"
  → test: "returns 401 for expired session token"

Edge cases (from CONTEXT.md Sharp Edges):
  → test: "returns 401 when token belongs to deleted user"
```

**Test design rules:**
- Test behaviour, not implementation. `expect(result).toBe(401)` not `expect(spy).toHaveBeenCalled()`.
- Test names read as sentences.
- At least one happy-path and one failure-path per acceptance criterion.

---

## Phase 5 — RED (write failing tests)

Write all tests from the plan. They must fail at this stage.

```typescript
describe('<module name>', () => {
  describe('<behaviour group>', () => {
    it('<test name as a sentence>', async () => {
      // Arrange
      const input = <setup>;

      // Act
      const result = await functionUnderTest(input);

      // Assert
      expect(result).<matcher>;
    });
  });
});
```

Run to confirm they fail:
```bash
<TEST_COMMAND> --testPathPattern "<new_test_file>"
```

If a test passes before implementation exists, the test is testing the wrong thing.

**Commit the failing tests — this is mandatory, never skip:**

```bash
git add <test_file_path>
git commit -m "test(ISS-XXX): failing tests for <feature>

Tests cover:
- <AC1 summary>
- <edge cases>

All N tests fail as expected (implementation not yet written)."
```

---

## Phase 6 — GREEN (implement to pass)

Write minimum code to make all tests pass.

**Minimum means:** no extra features, no speculative abstractions, no "while I'm here" changes.

```bash
# Write implementation following CONTEXT.md Domain Glossary naming
# Respect all Key Invariants from CONTEXT.md
<TEST_COMMAND> --testPathPattern "<test_file>"  # run after each logical unit
<TEST_COMMAND>   # full suite — must stay green
<LINT_COMMAND>
```

**If a test is hard to pass:** Fix the implementation, never modify the test assertion.
**If >5 files required:** The issue was scoped wrong — stop and flag to user.

```bash
git add <all_implementation_files>
git commit -m "feat(ISS-XXX): <what was implemented>

Implements acceptance criteria from ISS-XXX.
All N tests pass. Full suite: green."
```

---

## Phase 7 — REFACTOR

With tests green, improve code quality without changing behaviour.

Look for: duplicate logic, functions >20 lines, magic strings, naming inconsistent with Domain Glossary, missing error handling.

- Run tests after every change
- If refactor needs a new test, write it red → green first

```bash
git commit -m "refactor(ISS-XXX): clean up <what>

No behaviour change. Tests: still green."
```

Skip this commit if nothing needed refactoring.

---

## Phase 8 — PR

```markdown
## What
<1–3 sentences>

## Why
Closes ISS-XXX
Related PRD: docs/prd/PRD-<slug>.md

## Changes
- `<file>`: <what changed>

## Tests
N new tests. Full suite: ✅ green. Lint: ✅ clean.

## Invariant Compliance
- "<invariant>": ✅ respected — <how>

## Checklist
- [ ] Tests pass locally
- [ ] Lint passes
- [ ] No hardcoded secrets
- [ ] CONTEXT.md updated if architectural decision made
```

```bash
gh pr create \
  --title "feat(ISS-XXX): <title>" \
  --body "<PR description>" \
  --base main \
  --label "type:feat,status:needs-review"
```

---

## Phase 9 — UPDATE ISSUE

```bash
sed -i 's/status: open/status: needs-review/' issues/ISS-XXX-*.md
git add issues/ISS-XXX-*.md
git commit -m "chore(ISS-XXX): mark as needs-review, link PR"
```

---

## Handoff

```
✅ Implementation complete

  Issue:  ISS-XXX — <title>
  Tests:  N new tests, all green
  Suite:  ✅ full suite passing
  PR:     <branch or GH PR link>
  Status: needs-review
```

---

## Stack-Specific Test Patterns

See `references/testing-patterns.md` for TypeScript/Jest, Python/pytest, Go, and Rust.

---

## Error Cases

**Ambiguous criteria:** Stop at Phase 4. Write interpretation in issue comment, ask user to confirm.
**Test passes before implementation:** Regression test is OK; new-feature test is wrong — review with user.
**>5 files required:** Issue scoped too broadly — stop, list files, ask user.
**Suite fails after implementation:** Regression introduced — diagnose before opening PR.
**AGENTS.md says human oversight required:** Stop. Ask for explicit authorisation.
**Invariant violation required to pass tests:** Stop. Design flaw — describe conflict, ask how to resolve.

---

## Reference

Reads: `AGENTS.md`, `CONTEXT.md`, `issues/ISS-XXX-*.md`, `docs/reports/`, `docs/prd/`
Writes: `src/`, `tests/`, `issues/ISS-XXX-*.md` (status), opens PR
Downstream: human review → `arch-auditor`
