# Common Invariant Patterns & Audit Checks

Reference for the arch-auditor skill. Maps common invariant types to
specific grep/search strategies for verification.

---

## Data Integrity Invariants

### "Records are never hard-deleted"
```bash
# TypeScript / JS
grep -rn "\.delete\(\|\.destroy\(\|\.remove\(" src/ --include="*.ts" | grep -v "test\|spec\|mock\|soft"
grep -rn "DELETE FROM" src/ --include="*.ts" --include="*.sql" | grep -v "test\|spec"

# Python (SQLAlchemy)
grep -rn "\.delete()\|session.delete\|db.delete" src/ --include="*.py" | grep -v "test\|soft"

# Go
grep -rn "DELETE FROM\|\.Delete(" internal/ --include="*.go" | grep -v "test\|soft"
```

### "All writes are audited / logged"
```bash
grep -rn "func.*Create\|func.*Update\|func.*Delete" src/ --include="*.ts" | \
  awk -F: '{print $1}' | sort -u | while read f; do
    grep -L "audit\|log\|emit\|publish" "$f" && echo "MISSING AUDIT: $f"
  done
```

### "Monetary amounts stored as integers (cents), never floats"
```bash
grep -rn "amount\|price\|cost\|total\|fee" src/ --include="*.ts" | \
  grep "float\|decimal\|number\|Double\|Float" | grep -v "test"

grep -rn "amount \* \|price \* \|total \+ " src/ --include="*.ts" | grep -v "test"
```

---

## Architecture Invariants

### "All mutations go through the event bus / message queue"
```bash
grep -rn "prisma\.\|knex\.\|repository\." src/ --include="*.ts" | \
  grep -E "\.(create|update|upsert|delete)\(" | \
  grep -v "test\|spec\|mock" | grep -v "event-handler\|consumer\|listener"
```

### "No business logic in route handlers"
```bash
grep -rn "router\.\(get\|post\|put\|delete\|patch\)" src/ --include="*.ts" | \
  awk -F: '{print $1}' | sort -u | while read f; do
    wc -l "$f"
  done | awk '$1 > 100 {print "LARGE ROUTE FILE: "$2}'
```

### "All external API calls go through the service layer"
```bash
grep -rn "fetch(\|axios\.\|http\.\|https\." src/ --include="*.ts" | \
  grep -v "test\|spec\|mock" | grep -v "services/\|adapters/\|clients/"
```

### "Repository pattern — no direct DB access outside repositories"
```bash
# TypeScript
grep -rn "prisma\.\|mongoose\.\|knex\." src/ --include="*.ts" | \
  grep -v "repository\|Repository\|test\|spec\|mock\|migration"

# Python
grep -rn "session\.\|db\.query" src/ --include="*.py" | \
  grep -v "repository\|repo\|test\|spec"
```

---

## Security Invariants

### "No hardcoded secrets"
```bash
grep -rEn "(password|secret|api_key|apikey|token|auth)\s*=\s*['\"][^'\"]{8,}" \
  src/ --include="*.ts" --include="*.py" --include="*.go" | \
  grep -v "test\|spec\|mock\|example\|placeholder\|YOUR_\|<"
```

### "SQL queries are always parameterised"
```bash
grep -rn "\"SELECT\|'SELECT\|\`SELECT" src/ --include="*.ts" | \
  grep "\${" | grep -v "test\|spec" | head -10

grep -rn "f\"SELECT\|f'SELECT" src/ --include="*.py" | grep -v "test\|spec"
```

### "No PII in logs"
```bash
grep -rn "console\.log\|logger\." src/ --include="*.ts" | \
  grep -i "email\|password\|ssn\|phone\|address\|dob\|credit" | \
  grep -v "test\|spec"
```

---

## Dependency & Config Invariants

### "No new npm packages without ADR approval"
```bash
git show HEAD~30:package.json 2>/dev/null | \
  python3 -c "import json,sys; d=json.load(sys.stdin); [print(k) for k in d.get('dependencies',{})]" \
  > /tmp/old-deps.txt

cat package.json | \
  python3 -c "import json,sys; d=json.load(sys.stdin); [print(k) for k in d.get('dependencies',{})]" \
  > /tmp/new-deps.txt

comm -13 /tmp/old-deps.txt /tmp/new-deps.txt  # new packages added
```

### "All environment variables documented in .env.example"
```bash
grep -rn "process\.env\." src/ --include="*.ts" | \
  grep -oP 'process\.env\.\K[A-Z_]+' | sort -u > /tmp/env-used.txt

grep -oP '^[A-Z_]+(?==)' .env.example | sort > /tmp/env-documented.txt

comm -23 /tmp/env-used.txt /tmp/env-documented.txt
```

---

## Code Quality Indicators

### Dead exports (TypeScript)
```bash
npx ts-prune 2>/dev/null | grep -v "used in module" | head -20
```

### Missing error handling
```bash
# Promise chains without catch
grep -rn "\.then(" src/ --include="*.ts" | grep -v "\.catch(\|test\|spec" | head -10

# Async functions without try/catch
grep -rn "async function\|async (" src/ --include="*.ts" | \
  awk -F: '{print $1}' | sort -u | while read f; do
    grep -L "try\s*{" "$f" && echo "MISSING TRY/CATCH: $f"
  done | head -10
```

### Console.log left in production code
```bash
grep -rn "console\.log\|console\.debug\|print(" src/ --include="*.ts" --include="*.py" | \
  grep -v "test\|spec\|logger\|logging"
```

---

## Monorepo-Specific Checks

### "Packages don't import from each other's internals"
```bash
grep -rn "from '\.\./\.\./packages/" packages/ --include="*.ts" | \
  grep -v "/index\|/types" | head -10
```

### "Shared config is used, not duplicated"
```bash
diff <(cat tsconfig.json 2>/dev/null) <(cat packages/*/tsconfig.json 2>/dev/null) | head -20
```
