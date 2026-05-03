# Onboarding Reference

Reference for the onboarder skill — loaded during Phase 4 (Synthesise).
Contains persona-specific writing guidance, section prioritisation by role,
reading-order templates, and quality checks.

---

## Persona Decision Tree

```
Who is being onboarded?
│
├── Human
│   ├── Engineer (frontend / backend / fullstack)
│   │   → Variant A — full narrative document
│   │   → Lead with "what this does and why", then domain, then rules
│   │
│   ├── Engineering Manager / Tech Lead
│   │   → Variant A, compressed
│   │   → Emphasise: architecture overview, current work, ADR decisions
│   │   → De-emphasise: run commands, sharp edges (delegate to engineers)
│   │
│   ├── Product Manager / Designer
│   │   → Variant A, but skip tech stack and run commands
│   │   → Lead with: what it does, domain glossary, current PRDs
│   │   → Add: user-facing features currently available (derive from PRDs)
│   │
│   └── New contributor (open source or contractor)
│       → Variant A + add a "How to Contribute" section
│       → Source contribution rules from AGENTS.md commit convention
│       → Point to issues labelled agent:ready as good first tasks
│
└── Agent
    → Variant B — operational rules document
    → Lead with constraints, then glossary, then map
    → Every claim must be verifiable from a source file
```

---

## Section Priority by Role

Not all sections matter equally to all personas. Weight accordingly:

| Section | Backend Eng | Frontend Eng | PM/Designer | Agent |
|---------|-------------|--------------|-------------|-------|
| What it does | High | High | High | Medium |
| Domain glossary | High | High | High | Critical |
| Architecture | High | Medium | Low | High |
| Key invariants | High | Medium | Low | Critical |
| Sharp edges | High | High | Low | Critical |
| Tech stack | High | High | Low | Medium |
| Run commands | Critical | Critical | Skip | Critical |
| Current work | Medium | Medium | High | Low |
| ADR decisions | Medium | Low | Skip | High |
| First action | High | High | Medium | N/A |

For lower-priority sections: keep them brief (2-3 lines) or omit if the knowledge
layer doesn't have relevant content.

---

## Reading Order Templates

### For a backend engineer

> Read in this order — each one builds on the last:
>
> 1. This document (you're reading it)
> 2. `CONTEXT.md` — the full knowledge base; pay special attention to Key Invariants
> 3. `AGENTS.md` — commit convention, what you can do without asking
> 4. `docs/adr/ADR-001-*.md` through `ADR-003-*.md` — the foundational decisions
> 5. `src/<core-domain-module>/` — the heart of the business logic
> 6. `src/<core-domain-module>/__tests__/` — the tests tell you what the code guarantees

### For a frontend engineer

> 1. This document
> 2. `CONTEXT.md` — especially the Domain Glossary (terms appear in API responses)
> 3. `AGENTS.md` — commit convention
> 4. `src/<ui-module>/` — the UI entry point
> 5. Any ADRs tagged with `domain:ui`
> 6. Open issues labelled `domain:ui` — what's being worked on right now

### For an agent

> Load these in order as system context before any other action:
>
> 1. This document (docs/onboarding.md) — your operating constraints
> 2. `CONTEXT.md` — invariants and glossary are absolute
> 3. `AGENTS.md` — your allowlist and run commands
> 4. `docs/.agent/labels.md` — the issue taxonomy
> 5. `docs/.agent/agents.index.md` — the other agents you may encounter

---

## "Sharp Edges" Writing Guide

The sharp edges section is often the most valuable part of the document and the
most poorly written. Rules for writing it well:

**Write it as advice, not as warnings.**

❌ "WARNING: BillingCycle.endDate is exclusive."
✅ "BillingCycle.endDate is exclusive — a cycle from Jan 1 to Feb 1 does not
   include Feb 1. This trips up everyone. When querying by date range, always
   subtract one day from the end boundary."

**Name the file or module it affects.**

❌ "Date handling can be tricky."
✅ "Date handling in `src/billing/cycle.ts` — endDate is exclusive (see line 47).
   The test in `tests/billing/cycle.test.ts:89` demonstrates the correct pattern."

**Say what goes wrong, not just what to watch for.**

❌ "Be careful with the event bus."
✅ "The event bus does not guarantee delivery order. If you write two events in
   quick succession and both update the same record, the second may arrive first.
   `src/events/ordering.ts` contains the idempotency guard — always read it before
   adding new event handlers."

**Limit to the 5 most important.** More than 5 sharp edges means the person reading
will remember none of them. Curate ruthlessly. The rest belong in CONTEXT.md where
they can be discovered when relevant.

---

## "First Action" Design Guide

The "First Action" section only works if it's genuinely achievable and genuinely
teaches something about the system. Rules:

**It must be completable in 30 minutes by someone who just set up their environment.**

**It must touch the core of the system, not the periphery.**

❌ "Read all the ADRs."
❌ "Set up your editor."
✅ "Run the test suite (`npm test`), then open `src/auth/middleware.ts` and read it
   end-to-end. It's the most-touched file in the codebase and understanding it gives
   you a map of how authentication works throughout the system."

**For agents, the first action is not a task — it's a verification:**

✅ "Before picking up any issue, confirm you can run `<TEST_COMMAND>` and all
   tests pass. If any fail, stop and report to a human before proceeding."

---

## Deriving "Current Work" from Git + Issues

This section should read like a teammate briefing, not a log dump.

**Raw material:**
```bash
git log --oneline --since="14 days ago"
grep -rh "^title:\|^priority:\|^status:" issues/*.md | head -30
```

**Transform it:**

From:
```
a3f9c21 feat(ISS-042): persistent sessions via Redis
b7d1e44 fix(ISS-044): off-by-one in billing date range
e2a1f33 chore: bump postgres driver to v15
```

To:
> The team recently migrated session storage from in-memory to Redis (ISS-042),
> which enables the 30-day remember-me feature. A date range bug in billing was
> fixed (ISS-044) — worth knowing about if you're working near BillingCycle.
> The open priority work right now is [ISS-047: invoice generation, P1] and
> [ISS-049: audit logging, P2].

The key transformation: connect the commits to their purpose, and name the
current priorities explicitly.

---

## Staleness Markers

When writing the document, add inline staleness markers for sections that are
likely to change fast:

```markdown
<!-- STALE-RISK: This section reflects the state as of <date>.
     Re-run onboarder after any significant change to <area>. -->
```

Sections that warrant staleness markers:
- "Current Work" (changes every sprint)
- "Tech Stack" if a migration is in progress
- Architecture overview if a major refactor is planned
- Any section that says "coming soon" or "being replaced"

---

## Verification Checklist

Before writing `docs/onboarding.md`, verify each claim:

### Domain terms
```bash
# Every term used in the doc must be in the glossary section
# Cross-reference: terms in doc body vs. terms in glossary table
```

### Invariants
```bash
# Invariant text must match CONTEXT.md verbatim
grep "## Key Invariants" CONTEXT.md -A 20
# Compare against what you're writing — no paraphrasing
```

### Run commands
```bash
# Commands must be copied from AGENTS.md, not from memory
grep -A 20 "## Run Commands" AGENTS.md
# Test: do these commands actually work?
```

### ADR summaries
```bash
# Each ADR summary must match the ADR's Decision section
for f in docs/adr/*.md; do
  echo "=== $(basename $f) ==="
  grep -A 3 "## Decision" "$f"
done
```

### Current work accuracy
```bash
# Issue titles and priorities must match actual issue files
grep "^title:\|^priority:\|^status:" issues/*.md
```

---

## Monorepo Addendum

If the project is a monorepo, the Package Map section is the most important
addition. Structure it as:

```markdown
## Package Map

This is a monorepo. Each package has its own CONTEXT.md with domain-specific
knowledge. Always read the relevant package's CONTEXT.md before working in it.

| Package | Path | Purpose | Context |
|---------|------|---------|---------|
| `<name>` | `packages/<name>/` | <one-line purpose> | [CONTEXT.md](packages/<name>/CONTEXT.md) |

**Cross-package rules:**
- Packages import from each other via public index files only — not from internal paths
- Shared config lives in `packages/config/` — do not duplicate it
- <Any other cross-cutting invariants from CONTEXT.index.md>

**You'll primarily be working in:** `packages/<relevant-package>/`
Start with that package's CONTEXT.md before reading the root one.
```

---

## Output Quality Bar

The onboarding document is passing quality if:

1. A new engineer could get their environment running from the "How to Run It"
   section alone — no Slack messages needed
2. A new agent could pass the invariant check in `pr-reviewer` without reading
   anything else first
3. Every domain term used in code reviews will have been defined before the
   person encounters it
4. The "sharp edges" section would have prevented at least one of the bugs
   documented in `docs/reports/diagnosis-*.md`
5. The "first action" has been tested — someone actually did it in ~30 minutes

If the document fails any of these, it needs more work before it ships.
