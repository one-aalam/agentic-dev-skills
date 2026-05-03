# Challenge Lenses — Quick Reference

Used in Phase 3 of the architect skill. Each lens is a line of questioning
to apply to the plan under review. Work through them in order; skip only those
that genuinely don't apply.

---

## Lens 1 — Invariant Integrity

**Question:** Does this plan violate any Key Invariant?

**What to check:**
- Read every invariant in `CONTEXT.md → Key Invariants`
- For each: does the plan require doing the thing the invariant forbids?
- For each: does the plan assume something the invariant guarantees?

**Signal phrases from the user that trigger this lens:**
- "we'd need to delete..."
- "we'd bypass the..."
- "we could skip the validation for..."
- "just this once we'd..."

**If a violation is found:**
> "This design would violate the invariant: '[exact text]'. Either the invariant
> needs to change via an ADR, or the design does. Which?"

**Severity:** Hard stop. Cannot proceed without resolution.

---

## Lens 2 — ADR Consistency

**Question:** Does this plan contradict an accepted ADR?

**What to check:**
- Read all ADRs with `Status: accepted`
- Does the plan propose what an ADR explicitly rejected?
- Does the plan assume a different technology or pattern than one an ADR mandated?

**Signal phrases:**
- "we could use [technology X]" — when an ADR chose Y over X
- "we wouldn't need the [pattern]" — when an ADR mandated the pattern
- "we'd go direct" — when an ADR chose an indirection

**If a contradiction is found:**
> "ADR-XXX decided '[decision]'. This plan goes in a different direction.
> Is this intentional? If so, we should supersede ADR-XXX with a new one."

**Severity:** High. Reversals are valid — they just need to be explicit.

---

## Lens 3 — Domain Model Coherence

**Question:** Does this plan use domain language consistently?

**The three cases:**
```
In glossary, used consistently   → ✅ good
In glossary, used differently    → 🟡 flag: "You're using X to mean Y, but the glossary defines it as Z"
Not in glossary                  → 🔴 stop: "What exactly is a [term]? We need a definition."
```

**The naming quality test (for new concepts):**
1. Can it be distinguished from adjacent concepts in one sentence?
2. Does the name suggest what it does AND what it doesn't?
3. Would a domain expert use this term naturally?
4. Read 6 months from now — still clear?

---

## Lens 4 — Boundary Violations

**Question:** Does this plan create undesirable coupling?

**Coupling patterns to look for:**
- Module A reading directly from Module B's internal data
- A component taking on a responsibility architecturally owned by another
- A new abstraction that straddles a domain boundary
- Circular dependencies being introduced
- A thin layer that becomes a fat layer (e.g., route handler with business logic)

```bash
# Where does similar functionality live today?
grep -rn "<concept>" src/ --include="*.ts" | head -10
```

**Signal questions:**
- "Who owns the [concept]? Which module is responsible for it?"
- "When [X] changes in the future, what else has to change?"

---

## Lens 5 — Alternatives Not Considered

**Question:** Is there a simpler or more consistent solution?

**What to look for:**
- An existing pattern in the codebase that already solves this
- A library/dependency already in use that handles it
- An over-engineered solution to a small problem

```bash
grep -rn "<concept or pattern>" src/ --include="*.ts" | head -10
ls docs/adr/ | grep -i "<related keyword>"
```

**The YAGNI check:** "Is this complexity needed now, or are we building for a future that may not come?"

---

## Lens 6 — Downstream Consequences

**Question:** What does this decision lock in, and what does it make harder?

**Think through:**
- What changes when this is built?
- What becomes harder once this is in place?
- What gets locked in (hard to undo)?
- At 10x scale — does this design hold?

**The reversal cost question:** "If in 6 months we decide this was wrong — how hard is it to undo?"

**The data question:** "What does a migration look like if we need to change this data's shape?"

---

## Lens 7 — Operability

**Question:** Can this be deployed, observed, and debugged in production?

**Questions to ask:**
- "How is this rolled out progressively? Is there a feature flag?"
- "If this has a bug in production at 3am — what do you look at?"
- "What logs, metrics, or traces does this emit?"
- "What does a bad deploy look like, and how do you roll it back?"

**Common omissions:** no rollback plan for DB migrations, no observability for async
processes, feature flag not user-scoped, no timeout for new external calls.

---

## Priority Guide

| Proposal type | Lead with |
|---|---|
| New data entity or schema | Lenses 1, 3, 6 |
| New service or module | Lenses 4, 2, 5 |
| New external integration | Lenses 7, 1, 6 |
| New abstraction or pattern | Lenses 3, 4, 5 |
| Replacing an existing component | Lenses 2, 1, 6 |
| Performance optimisation | Lenses 6, 7, 1 |
| Security-adjacent change | Lens 1, then stop and escalate |

---

## The One-Challenge Rule

Surface at most 2 challenges at once. After those are answered, surface the next.
If you dump 7 challenges at once, the user addresses the easiest 2 and ignores
the hardest ones — which are usually the ones that matter.

Order by: (1) hardest to undo, (2) broadest impact, (3) most likely to be wrong.
