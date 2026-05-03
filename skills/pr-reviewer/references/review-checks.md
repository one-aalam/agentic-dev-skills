# Review Checks Reference

Reference for the pr-reviewer skill — phases 4, 5, and 8.
Grep patterns for checking invariants against a diff, verifying ADR compliance,
and assessing test quality. Load during Phase 4 (Invariants) and Phase 8 (Tests).

---

## Invariant Diff Checks

For each invariant type, these patterns detect violations in the PR diff.
Run against the changed files only — not the full codebase.

```bash
# Get changed source files (excluding tests)
CHANGED=$(git diff --name-only main...<branch> | grep -E "^src/|^lib/|^app/|^internal/" | grep -v "test\|spec")
```

---

### "Records are never hard-deleted"

```bash
# Direct delete calls in changed files
echo "$CHANGED" | xargs grep -n "\.delete\(\|\.destroy\(\|\.remove\(" 2>/dev/null | \
  grep -v "soft\|archive\|test\|spec\|mock"

# Raw SQL DELETE
echo "$CHANGED" | xargs grep -n "DELETE FROM" 2>/dev/null | \
  grep -v "test\|spec"

# ORM hard-delete methods by stack:
# TypeScript/Prisma:  prisma.<model>.delete(
# TypeScript/Mongoose: .deleteOne( .deleteMany( .findByIdAndDelete(
# Python/SQLAlchemy:  session.delete( db.session.delete(
# Go/GORM:           db.Delete( db.Unscoped().Delete(
```

---

### "All mutations go through the event bus / message queue"

```bash
# DB write calls that aren't inside event handlers
echo "$CHANGED" | xargs grep -n "\.create\(\|\.update\(\|\.upsert\(\|\.save\(" 2>/dev/null | \
  grep -v "test\|spec\|mock\|event\|handler\|consumer\|listener\|repository\|Repository"

# Verify the callsite is inside a repository or event handler
# (requires reading the file context around the match)
```

---

### "No business logic in route handlers"

```bash
# Route files that are suspiciously long after the PR
echo "$CHANGED" | grep -E "route|controller|handler" | \
  xargs wc -l 2>/dev/null | sort -n | tail -5
# Files >100 lines are candidates for inspection
```

---

### "Monetary amounts stored as integers (cents), never floats"

```bash
# Float literals near money fields
echo "$CHANGED" | xargs grep -n "amount\|price\|cost\|total\|fee\|charge" 2>/dev/null | \
  grep -E "[0-9]+\.[0-9]+" | grep -v "test\|spec\|comment\|//"

# Float type annotations
echo "$CHANGED" | xargs grep -n "amount.*float\|price.*float\|cost.*float" 2>/dev/null | \
  grep -v test

# Division that could produce floats
echo "$CHANGED" | xargs grep -n "amount.*\/\|price.*\/" 2>/dev/null | \
  grep -v "Math.floor\|Math.round\|int(\|round(" | grep -v test
```

---

### "No hardcoded secrets or credentials"

```bash
# Common secret patterns in new lines only
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -iE "(password|secret|api_key|token|apikey)\s*[=:]\s*['\"][^'\"]{8,}" | \
  grep -v "process\.env\|os\.environ\|getenv\|config\.\|placeholder\|example\|test\|YOUR_"

# Base64-looking strings (potential encoded secrets)
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "[A-Za-z0-9+/]{40,}={0,2}" | grep -v "test\|spec\|hash\|sha\|digest"
```

---

### "SQL queries are always parameterised"

```bash
# String concatenation into SQL
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "(SELECT|INSERT|UPDATE|DELETE).*\$\{|\+\s*['\"]|\+ userId|\+ id" | \
  grep -v "test\|spec\|comment\|//"

# Python f-strings with SQL
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E 'f"(SELECT|INSERT|UPDATE|DELETE)' | grep -v test
```

---

### "All API endpoints require auth middleware"

```bash
# New routes in the diff
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "router\.(get|post|put|patch|delete)\(|app\.(get|post|put|patch|delete)\(" | \
  grep -v "test\|spec"

# For each new route, verify auth middleware is present on the same line or adjacent
# (requires reading file context — automated check is a heuristic only)
```

---

### "No PII in logs"

```bash
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "console\.log|logger\.|log\." | \
  grep -iE "email|password|ssn|phone|address|dob|credit|token|secret" | \
  grep -v "test\|spec\|redacted\|masked\|\*\*\*"
```

---

## ADR Compliance Checks

### ADR: "Repository pattern — no direct DB access outside repositories"

```bash
# Non-repository files that call ORM directly
git diff main...<branch> --name-only | grep -v "repository\|Repository\|test\|spec" | \
  while read f; do
    grep -n "prisma\.\|mongoose\.\|knex\.\|db\.query\|session\.query" "$f" 2>/dev/null | \
      grep -v "test\|spec\|mock" && echo "  ^ in $f"
  done
```

### ADR: "Use the event-sourcing pattern for state changes"

```bash
# Direct property mutations instead of events
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "\.(status|state)\s*=" | \
  grep -v "test\|spec\|mock\|event\|emit\|publish\|dispatch"
```

### ADR: "Feature flags for all new user-facing features"

```bash
# New UI-adjacent code without a feature flag check
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "render|return.*<|component|view" | head -10
# Then check if any of these are gated by a feature flag
git diff main...<branch> -- $CHANGED | grep "featureFlag\|isEnabled\|features\." | head -5
```

---

## Test Quality Heuristics

### Detecting phantom tests (assertions that can't fail)

```bash
# Tests with only existence/truthiness checks
git diff main...<branch> -- "*.test.*" "*.spec.*" | grep "^+" | \
  grep -E "expect\(.*\)\.(toBeDefined|toBeTruthy|not\.toBeNull|toBe\(true\)|toExist)" | \
  grep -v "explicitly checking"
```

These pass even if the function returns the wrong value. Flag each one.

### Detecting tests with no assertions

```bash
# Test blocks with no expect() call
git diff main...<branch> -- "*.test.*" "*.spec.*" | grep "^+" | \
  grep -E "it\(|test\(" | while read line; do
    # Check if corresponding test body contains expect
    echo "$line" # manual inspection required for context
  done
```

### Detecting spy-only tests

```bash
# Tests that only assert on mock/spy calls, not observable output
git diff main...<branch> -- "*.test.*" "*.spec.*" | grep "^+" | \
  grep -E "expect\(.*spy|expect\(.*mock|toHaveBeenCalled" | head -10
# Cross-check: does the test ALSO assert on the actual return value/side effect?
```

### Checking for the TDD commit pattern

```bash
# Verify a test commit precedes the implementation commit
git log main...<branch> --oneline | grep -n "^"
# Look for: test(ISS-XXX): commit appearing BEFORE feat(ISS-XXX): commit
# If feat commit appears first, or only feat commit exists → Flag
```

### Coverage of new public functions

```bash
# New exported/public functions added in the diff
git diff main...<branch> -- $CHANGED | grep "^+" | \
  grep -E "^export (function|const|class|async function)|^public " | \
  grep -oP '(?<=function |const |class )\w+' | sort > /tmp/new-exports.txt

# Test descriptions referencing these functions
git diff main...<branch> -- "*.test.*" "*.spec.*" | grep "^+" | \
  grep -oP "(?<=describe\(|it\(|test\()['\"][^'\"]+['\"]" | sort > /tmp/test-descriptions.txt

# Gap — exported functions with no corresponding test description
# (heuristic — names may differ, requires human judgement)
comm -23 /tmp/new-exports.txt /tmp/test-descriptions.txt
```

---

## Scope Check Patterns

### Files changed outside expected domain

```bash
# Get the issue's affected files hint (from issue frontmatter)
EXPECTED_PATHS=$(grep "affects:" issues/ISS-XXX-*.md | sed 's/affects: \[//;s/\]//' | tr ',' '\n' | tr -d ' "')

# Files changed that don't match expected paths
git diff --name-only main...<branch> | while read f; do
  MATCHED=false
  for path in $EXPECTED_PATHS; do
    [[ "$f" == *"$path"* ]] && MATCHED=true && break
  done
  $MATCHED || echo "UNEXPECTED: $f"
done
```

### Detecting bundled unrelated changes

```bash
# Commits in the PR that touch unrelated areas
git log main...<branch> --oneline --stat | grep -v "test\|spec" | \
  grep -E "^\s+[0-9]+ file" | head -10
# Look for commits that touch modules completely unrelated to the issue domain
```

---

## Severity Decision Table

When a finding could be Flag or Block, use this table:

| Finding type | Default | Escalate to Block if... |
|---|---|---|
| AC not implemented | 🚫 Block | Always |
| AC implemented differently | ⚠️ Flag | It's a security or data integrity AC |
| Invariant violated | 🚫 Block | Always |
| ADR violated | 🚫 Block | Always |
| Naming drift from glossary | ⚠️ Flag | The same wrong name is used in >5 places |
| New concept not in glossary | ⚠️ Flag | Never block for this |
| Out-of-scope file | 🚫 Block | Always |
| Zero tests for new behaviour | 🚫 Block | Always |
| Loose test assertions | ⚠️ Flag | All tests are loose (no real coverage) |
| Missing edge case test | ⚠️ Flag | Edge case is a documented Sharp Edge |
| Spy-only tests | ⚠️ Flag | Never block for this alone |
| No failing-test commit | ⚠️ Flag | Project has TDD as a stated invariant |
| Security smell | 🚫 Block + escalate | Always |
| PII in logs | 🚫 Block | Always |
