# ADR Writing Guide

Reference for the architect skill — how to write ADRs that are genuinely useful
and not just bureaucratic checkboxes. Read this when writing ADRs in Phase 6.

---

## What makes an ADR useful

An ADR is useful when someone reading it 12 months later can understand:
- **Why** this decision was made (not what — the code shows what)
- **What was rejected** and why (to avoid relitigating settled questions)
- **What the decision commits them to** going forward

An ADR is useless when it:
- Describes the decision but not the context that forced it
- Omits the alternatives (making it look like there was only one option)
- Is so abstract it could apply to any project

---

## The Context section — the most important section

Most ADRs have weak Context sections. A good one answers:

1. **What was happening** — what feature, project, or situation required a decision
2. **What constraint or tension created the need to choose** — the actual forcing function
3. **What couldn't be done without making this decision** — the specific blocker

**Bad context:**
> "We needed to choose a session management approach."

**Good context:**
> "We're adding OAuth login to the billing portal. Sessions need to persist across
> browser restarts (customers complained about logging in every visit) but also
> expire correctly to avoid stale token abuse. The existing cookie-only approach
> from the internal admin panel doesn't support the required 30-day remember-me
> window without security regressions."

The good version is specific to this project and this moment. It will make sense
to a new engineer joining in 18 months. The bad version tells them nothing.

---

## The Decision section — one sentence first

Lead with the decision itself, stated plainly:

> "We will use JWTs with a 15-minute expiry stored in memory, with a refresh
> token in an HttpOnly cookie for persistent sessions."

Then explain the rationale — but only if the decision isn't self-evident from
the Context. Don't explain generic trade-offs; explain why this choice in this context.

---

## The Alternatives Considered table — be honest

The alternatives table is where most ADRs fail. Honest alternatives tables include:
- Options that were **seriously considered** (not just obvious non-starters)
- The **real reason** each was rejected (not "doesn't meet requirements" — what requirement?)
- **Pros for the rejected options** (if there are none, you've built a strawman)

Example of good alternatives:

| Option | Pros | Cons | Why Rejected |
|--------|------|------|--------------|
| Session cookies only | Simple, no token handling, browser manages expiry | Can't support 30-day remember-me without XSS exposure; can't be used in mobile clients | Doesn't meet remember-me requirement; blocks future mobile app |
| JWT only (no refresh) | Stateless, easy to verify | 15-min expiry means frequent login interruption; can't revoke without blocklist | UX impact unacceptable for billing portal users |

---

## The Consequences section — be specific about the negatives

The negative consequences and risks are the most important part of the Consequences
section. Positive consequences are usually obvious. Negative ones are what future
engineers need to know.

**Don't write:**
> "Negative: More complex than cookies alone."

**Write:**
> "Negative: Refresh token rotation requires a token store (Redis) and introduces
> a race condition risk when multiple tabs refresh simultaneously — mitigated by
> a short rotation window but requires careful testing. The mobile client will need
> to implement refresh logic."

Name the specific file, module, or workflow that gets harder. Generic language
helps no one.

---

## When to write a new ADR vs. update an existing one

**Write a NEW ADR when:**
- A genuinely new decision is being made
- An existing ADR is being reversed or significantly modified
  (new ADR supersedes the old one; update old ADR's status to `superseded-by ADR-NNN`)

**Update an EXISTING ADR when:**
- Adding a clarifying note to an accepted decision (add to Notes section)
- Correcting a factual error in the record (mark as `[corrected: <date>]`)

**Never** silently change a decision by editing the Decision section of an accepted
ADR. The history of what was decided must be traceable.

---

## Superseding an ADR

When a new decision reverses a prior one:

New ADR header:
```markdown
**Status:** accepted
**Supersedes:** ADR-003
```

Update the old ADR's status line:
```markdown
**Status:** superseded-by ADR-007
```

Add a note to the old ADR's Notes section:
```markdown
## Notes
Superseded by ADR-007 on <date>. Decision reversed because: <brief reason>.
```

---

## ADR status lifecycle

```
proposed   →  accepted     (decision made)
proposed   →  deprecated   (question became irrelevant before resolution)
accepted   →  superseded   (decision reversed — always replaced by a new ADR)
```

Don't leave ADRs in `proposed` state indefinitely. If a proposed ADR is more than
30 days old with no resolution, it should be accepted, deprecated, or explicitly
deferred with a note explaining why.

---

## Common mistakes

**Writing the ADR after the fact, when the decision is no longer fresh**
Result: vague context, alternatives that weren't actually considered.
Fix: write the ADR while the decision is being made — the architect skill does this inline.

**"We decided to use X" with no alternatives section**
Result: future engineers don't know what was considered and may relitigate.
Fix: always name at least one alternative that was genuinely considered and rejected.

**Generic consequences ("more maintainable", "more scalable")**
Result: useless to future engineers who need to understand the actual trade-offs.
Fix: name a specific file, pattern, or workflow that becomes easier or harder.

**ADR that could apply to any project**
Result: not useful for understanding this project's specific constraints.
Fix: reference specific modules, domain terms, or product requirements.

**Treating ADRs as bureaucracy to be minimised**
ADRs are the cheapest form of institutional memory. A 20-minute conversation that
produces a well-written ADR is worth weeks of archaeology later when someone asks
"why did we do it this way?". The cost of not writing them accumulates silently.
