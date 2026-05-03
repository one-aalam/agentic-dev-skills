# Drift Heuristics

Reference for the context-updater skill — Phase 3 (COMPARE).
Patterns that signal specific types of CONTEXT.md drift.
Load this file during Phase 3 to guide the comparison step.

---

## Invariant Relaxation Signals

These commit and code patterns suggest an invariant may have changed or been
intentionally relaxed. Flag for human review — never auto-update an invariant.

### Hard-delete where soft-delete is invariant
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "\.delete\(|\.destroy\(|DELETE FROM" | \
  grep -v "soft|archive|test|spec|mock"
```
Signal phrase in commits: "cleanup", "purge", "hard delete", "permanent removal"

### Direct DB write where event bus is invariant
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "prisma\.|repository\.|db\." | \
  grep -iE "\.(create|update|upsert|delete)\(" | \
  grep -v "test|spec|mock|event|handler|consumer"
```

### Auth bypass where auth-everywhere is invariant
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "skipAuth|noAuth|isPublic|allowAnonymous|bypassAuth"
```

### Float arithmetic where integer-money is invariant
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "(amount|price|cost|total)\s*[\*\/\+\-]\s*[0-9]" | \
  grep -v "test|spec|cents|integer"
```

---

## Architecture Change Signals

### New top-level module or service
```bash
git diff ${CONTEXT_SHA}..HEAD --name-only | \
  grep -E "^src/[^/]+/" | \
  sed 's|src/\([^/]*\)/.*|\1|' | \
  sort | uniq
```
Cross-reference against Architecture Overview. Any new directory not mentioned
is a candidate for an Architecture Overview addition.

### Deleted module
```bash
git diff ${CONTEXT_SHA}..HEAD --diff-filter=D --name-only | \
  grep -E "^src/[^/]+/" | \
  sed 's|src/\([^/]*\)/.*|\1|' | \
  sort | uniq
```
Cross-reference against Architecture Overview. Anything mentioned in overview
but deleted should be removed from description (with user confirmation).

### Renamed module or concept
```bash
# Commits with rename signals
git log --oneline ${CONTEXT_SHA}..HEAD | \
  grep -iE "rename|moved|migrated|now called|replaced by|refactor.*to"

# File moves (renames)
git diff ${CONTEXT_SHA}..HEAD --name-status | grep "^R"
```
Renames in code often mean the Domain Glossary term needs updating.

### New external dependency introduced
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "new .*(Client|SDK|Api|Service)\(|import.*from ['\"]@?[a-z]" | \
  grep -v "test|spec|mock|internal" | head -20
```
Cross-reference against External Dependencies table.

---

## Glossary Drift Signals

### New domain concepts in code (not in glossary)
```bash
# New service/repository/handler/manager class names
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -oP 'class [A-Z][a-zA-Z]+(?=Service|Repository|Manager|Handler|Controller|Provider)' | \
  sed 's/class //' | sort | uniq > /tmp/new-classes.txt

# Terms already in the glossary
grep -oP '\*\*[A-Z][a-zA-Z]+\*\*' CONTEXT.md | \
  grep -oP '[A-Z][a-zA-Z]+' | sort > /tmp/existing-glossary.txt

# Gap
comm -23 /tmp/new-classes.txt /tmp/existing-glossary.txt
```

### Renamed domain term
```bash
# Look for type/interface/class renames
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^-" | \
  grep -oP 'type [A-Z][a-zA-Z]+|interface [A-Z][a-zA-Z]+|class [A-Z][a-zA-Z]+' | \
  sed 's/type \|interface \|class //' | sort > /tmp/removed-types.txt

git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -oP 'type [A-Z][a-zA-Z]+|interface [A-Z][a-zA-Z]+|class [A-Z][a-zA-Z]+' | \
  sed 's/type \|interface \|class //' | sort > /tmp/added-types.txt

# Types removed suggest possible glossary term obsolescence
comm -23 /tmp/removed-types.txt /tmp/added-types.txt
```

---

## Sharp Edge Signals

These patterns in commits and code changes are the most reliable indicators
of new gotchas that should be documented in Sharp Edges.

### Fix commits (often encode gotchas)
```bash
git log --oneline ${CONTEXT_SHA}..HEAD | \
  grep -iE "^[a-f0-9]+ fix" | head -20
```
Read the diff for each fix commit. If it involves a non-obvious condition,
edge case, or platform quirk — that's a sharp edge candidate.

### Workaround comments added to code
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "// (NOTE|WARN|GOTCHA|CAREFUL|HACK|WORKAROUND|FIXME|TODO)|# (NOTE|WARN|GOTCHA)" | \
  grep -v test | head -20
```

### Retry or timeout logic added
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "retry|timeout|backoff|maxAttempts|deadline" | \
  grep -v test | head -10
```
New retry/timeout often signals an external dependency reliability issue worth documenting.

### Explicit ordering constraints added
```bash
git diff ${CONTEXT_SHA}..HEAD -- src/ | grep "^+" | \
  grep -iE "must.*before|always.*after|order.*matters|sequence" | \
  grep -v test | head -10
```

---

## Stack Change Signals

### Major version bump
```bash
git diff ${CONTEXT_SHA}..HEAD -- package.json | grep "^[+-]" | \
  grep -v "^---\|^+++" | \
  awk -F'"' '{
    if (/^\+/) { new[$2] = $4 }
    if (/^\-/) { old[$2] = $4 }
  }
  END {
    for (pkg in new) {
      if (old[pkg] != "" && old[pkg] != new[pkg]) {
        split(old[pkg], ov, "."); split(new[pkg], nv, ".")
        gsub(/[^0-9]/, "", ov[1]); gsub(/[^0-9]/, "", nv[1])
        if (nv[1] > ov[1]) print "MAJOR BUMP: " pkg " " old[pkg] " → " new[pkg]
      }
    }
  }'
```

### New package added
```bash
git diff ${CONTEXT_SHA}..HEAD -- package.json | grep "^+" | \
  grep -v "^+++" | grep '"' | \
  grep -v "version\|description\|author\|license\|name\|main\|scripts" | head -20
```

### Package removed
```bash
git diff ${CONTEXT_SHA}..HEAD -- package.json | grep "^-" | \
  grep -v "^---" | grep '"' | \
  grep -v "version\|description\|author\|license\|name\|main\|scripts" | head -20
```

---

## Significance Filter

Not every signal warrants a CONTEXT.md update. Apply this filter before proposing:

| Signal | Worth updating? |
|--------|----------------|
| New major dependency (e.g. Redis, Kafka, Stripe) | ✅ Yes — Tech Stack + External Deps |
| Patch version bump (1.2.3 → 1.2.4) | ❌ No — too minor |
| New domain concept in 5+ files | ✅ Yes — Glossary |
| New domain concept in 1 file | ⚠️ Maybe — only if it'll spread |
| New top-level service directory | ✅ Yes — Architecture Overview |
| Renamed file within existing module | ❌ No — too minor |
| Fix commit that reveals non-obvious constraint | ✅ Yes — Sharp Edges |
| Fix commit for simple typo/formatting | ❌ No |
| Invariant-adjacent code change | ✅ Always flag — but never auto-update |
| New ENV var in .env.example | ✅ Yes — External Dependencies |
| Deleted ENV var | ✅ Yes — External Dependencies |
| New ADR added | ✅ Yes — DECISIONS.md row + any CONTEXT.md implication |

When in doubt: **flag for human review rather than skip**. The user can always
decline a proposed update. A missed update compounds silently.
