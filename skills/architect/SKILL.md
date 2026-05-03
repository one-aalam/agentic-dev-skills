---
name: architect
description: >
  Stress-test a plan, design, or idea against the project's existing domain model,
  documented invariants, ADRs, and architectural decisions — then sharpen its language
  and write decisions into CONTEXT.md and ADRs as they crystallise during conversation.
  Trigger this skill when a user wants to think through a design before building it,
  challenge an approach, check if a plan fits the existing model, get architectural
  feedback, or says things like "does this design make sense", "stress-test this plan",
  "review my approach", "challenge this idea", "is this the right architecture",
  "does this fit our model", "review this design against our decisions", "help me
  think through the architecture", "is this consistent with our system", "does this
  violate anything", or "should we use X or Y". Also trigger when a user proposes
  a new abstraction, introduces new terminology, or makes a significant design choice
  in conversation and the decision should be captured before it drifts.
  This skill challenges plans conversationally, sharpens domain language, and writes
  crystallised decisions to CONTEXT.md and docs/adr/ inline — turning thinking into
  living documentation. Sits between prd-writer (downstream) and arch-auditor (backward).
---

# Architect Skill

Forward-looking architectural thought partner. Challenges plans against the project's
domain model and decisions. Sharpens language. Writes what crystallises into
`CONTEXT.md` and `docs/adr/` while the conversation is live.

---

## The Core Distinction

| | arch-auditor | architect |
|---|---|---|
| **Direction** | Backward — what drifted | Forward — will this plan work |
| **Input** | Existing codebase | A plan, idea, or design sketch |
| **Mode** | Read-only analysis | Conversational + writing |
| **Output** | Audit report + issues | Challenged plan + updated docs |
| **When** | After the fact | Before the code is written |

The architect is an interlocutor, not a reporter. It asks hard questions, offers
alternatives, flags contradictions — and when the user settles on a decision,
writes it down immediately so it doesn't get lost.

---

## Inputs Expected

The user provides one of:
- A plan, design, or approach in natural language ("I'm thinking of...")
- A sketch — architecture described in text, bullet points, or prose
- A question about approach ("should we use X or Y for this?")
- A PRD or issue to stress-test before implementation starts
- A new concept or abstraction they're introducing to the codebase
- A contested decision that needs resolution ("we can't agree on whether to...")
- "Review this against our docs" with a pasted design

---

## Phase Overview

```
Phase 1 — IMMERSE      Deeply read the project's knowledge layer
Phase 2 — RECEIVE      Understand the plan being proposed
Phase 3 — CHALLENGE    Ask the hard questions — don't rubber-stamp
Phase 4 — SHARPEN      Fix language; align to domain glossary
Phase 5 — SYNTHESISE   Reach decisions together through dialogue
Phase 6 — WRITE        Commit crystallised decisions to CONTEXT.md + ADRs
Phase 7 — HANDOFF      Surface what's decided, what's still open
```

---

## Phase 1 — IMMERSE

Read everything before engaging. An architect who doesn't know the domain speaks
in generic platitudes.

```bash
cat CONTEXT.md
cat DECISIONS.md
cat AGENTS.md

# All existing ADRs — read every accepted one
for f in docs/adr/*.md; do
  STATUS=$(grep -m1 "Status" "$f" 2>/dev/null)
  echo "=== $f ($STATUS) ==="
  cat "$f"
  echo
done

# Existing PRDs for patterns and vocabulary
ls docs/prd/*.md 2>/dev/null | grep -v TEMPLATE
```

From this reading, build an internal model of:

**Domain language** — the exact terms in the glossary. Any new term the user
introduces that isn't here is a signal to stop and name it precisely.

**Settled decisions** — accepted ADRs. Proposals that contradict these need a
very strong reason and must be made explicit.

**Open decisions** — anything marked `proposed` in ADRs, or questions in CONTEXT.md
marked `[stale-risk]` or `[needs decision]`. These are fair game.

**Key invariants** — absolute constraints. Any plan that violates one must be
flagged immediately.

**Architectural patterns** — how the existing system is structured. New things
should be consistent unless there's a good reason to introduce something new.

If `CONTEXT.md` or `docs/adr/` are missing or thin: note this at the start.
Recommend running `project-bootstrap` first if the project has no knowledge layer.

---

## Phase 2 — RECEIVE

Understand the plan before challenging it. Don't jump to critique.

If the input is thin or vague, ask one clarifying question:
> "Before I stress-test this — can you give me the key design decision at the
> heart of it? One sentence: 'We will do X by doing Y.'"

Then extract:
- **The core proposal** — what is actually being suggested
- **The claimed benefits** — why the user thinks this is the right approach
- **The affected area** — which parts of the codebase or domain this touches
- **New concepts being introduced** — any terms not in the Domain Glossary
- **Implicit decisions** — things assumed without being stated

Reflect your understanding back before challenging:
> "As I understand it, you're proposing to [X], which means [Y] and [Z].
> Does that capture it?"

Wait for confirmation. Don't challenge a misunderstood plan.

---

## Phase 3 — CHALLENGE

Work through the seven lenses from `references/challenge-lenses.md`. Load that
file now for the full detail on each lens.

The seven lenses in brief:
1. **Invariant integrity** — does this violate any Key Invariant?
2. **ADR consistency** — does this contradict an accepted ADR?
3. **Domain model coherence** — is the language precise and consistent?
4. **Boundary violations** — does this create undesirable coupling?
5. **Alternatives not considered** — is there a simpler or more consistent solution?
6. **Downstream consequences** — what does this decision lock in?
7. **Operability** — can this be deployed, observed, and debugged?

### Challenge discipline

**Ask one thing at a time.** Surface at most 2 challenges at once. After those
are answered, surface the next. Multiple simultaneous challenges cause the user
to answer the easiest ones and ignore the hard ones.

**Wait for answers.** Don't pre-answer your own questions.

**Be direct about concerns.** "This might be worth considering" is too soft.
"This design couples the billing module to the user module in a way that will
make future user schema changes break billing" is actionable.

**Acknowledge good design.** If part of the plan is solid and consistent with
the domain model, say so. Positive signal is useful too.

For each challenge, use this structure:
1. **The observation** — what you noticed
2. **The concern** — why it matters
3. **The question** — what the user needs to decide
4. (Optionally) **An alternative** — a different approach

---

## Phase 4 — SHARPEN

As the conversation progresses, sharpen the language.

### Terminology alignment

Every time the user uses a term imprecisely or inconsistently, note it and
propose the precise version. Check the Domain Glossary.

The three cases (from challenge-lenses.md):
- In glossary, used consistently → ✅ good
- In glossary, used differently → flag it: "You're using X to mean Y, but the glossary defines it as Z"
- Not in glossary → stop: "What exactly is a [term]? We need a definition."

### Naming new abstractions

If the user introduces a new concept:
> "You've described this a few different ways. Let me try a definition:
> '[Proposed name]': [one-sentence definition]. Does that capture it?"

Good abstraction name tests (from challenge-lenses.md):
1. Distinguishable from adjacent concepts in one sentence?
2. Name suggests what it does AND what it doesn't?
3. Would a domain expert use this term naturally?
4. Still clear when read 6 months from now?

### Invariant candidates

Watch for implicit constraints becoming true of the system:
> "We've now decided that [X] will always [Y]. Should that go into CONTEXT.md
> as an explicit Key Invariant?"

---

## Phase 5 — SYNTHESISE

Periodically gather the threads. Do this when a major challenge is resolved,
or the conversation has been running a while.

```
## Synthesising so far

**Decided:**
- [Decision 1] — [one sentence rationale]
- [Decision 2] — [one sentence rationale]

**Changed from original proposal:**
- [What changed] — [why]

**Still open:**
- [Open question 1] — [what needs to resolve it]
- [Open question 2]

**Glossary additions / changes:**
- "[New term]": [definition]
- "[Existing term]" updated: [what changed]

Ready to write these to docs? Or continue thinking?
```

Don't write prematurely — a decision still being debated shouldn't be in the
ADR yet. Ask the user whether to write now or keep exploring.

---

## Phase 6 — WRITE

When the user signals readiness. Read `references/adr-writing-guide.md` before
writing any ADR.

### What gets written where

| Content type | Destination |
|---|---|
| New architectural decision | New ADR in `docs/adr/` + row in `DECISIONS.md` |
| Reversal of an existing ADR | New superseding ADR + update old ADR status |
| New Key Invariant | `CONTEXT.md` → Key Invariants section |
| New domain term | `CONTEXT.md` → Domain Glossary |
| Updated domain term definition | `CONTEXT.md` → Domain Glossary (edit in place) |
| Architectural pattern clarification | `CONTEXT.md` → Architecture Overview |
| New sharp edge / gotcha discovered | `CONTEXT.md` → Sharp Edges & Gotchas |
| CONTEXT.md changelog entry | `CONTEXT.md` → Changelog |

### Writing ADRs

```bash
# Find next ADR number
NEXT_ADR=$(ls docs/adr/*.md 2>/dev/null | grep -oP '\d+' | sort -n | tail -1 | \
  xargs -I{} printf '%03d\n' $(( {} + 1 )))
[ -z "$NEXT_ADR" ] && NEXT_ADR="001"

TITLE_SLUG=$(echo "<decision title>" | tr '[:upper:]' '[:lower:]' | \
  sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g')
ADR_FILE="docs/adr/ADR-${NEXT_ADR}-${TITLE_SLUG}.md"
mkdir -p docs/adr
```

ADR format (populate from the conversation, not a generic template):

```markdown
# ADR-<NNN>: <Decision Title>

**Date:** <YYYY-MM-DD>
**Status:** accepted
**Deciders:** <from conversation>
**Supersedes:** ADR-<NNN> (if applicable)

## Context

<What situation or constraint in this project forced this decision?
Written in the project's domain language. Include the tension or problem
that required a choice. Reference the plan or feature that precipitated
this discussion. Be specific — a new engineer in 18 months must understand
why this choice was necessary here.>

## Decision

<What was decided, in one clear sentence.>
<Followed by explanation only if not self-evident from Context.>

## Alternatives Considered

| Option | Pros | Cons | Why Rejected |
|--------|------|------|--------------|
| <option from dialogue> | <from dialogue> | <from dialogue> | <from dialogue> |

## Consequences

**Positive:**
- <What improves — drawn from Phase 5 synthesis>

**Negative:**
- <What becomes harder or more constrained — be specific>

**Risks:**
- <Downstream consequences identified in Phase 3>

## Notes

<Link to PRD if applicable, related issues, questions to revisit.>
```

**Show to user before writing:**
> "Here's the ADR for '[decision]'. Does this capture the decision and our
> reasoning accurately? Edit anything before I write it."

Wait for confirmation.

### Updating CONTEXT.md

Make surgical edits — never rewrite whole sections.

**Adding a domain term:** append a new row to the Domain Glossary table.

**Adding a Key Invariant:** append `- <new invariant statement>` to the list.

**Updating an existing term:** show before/after:
> "The current definition of '[Term]' is: '[old]'. Should it become: '[new]'?"

**Always update the Changelog:**
```
| <YYYY-MM-DD> | <what changed> | architect skill |
```

### Updating DECISIONS.md

After writing each ADR, add a row:
```markdown
| ADR-NNN | <Decision Title> | accepted | <YYYY-MM-DD> |
```

If superseding an old ADR, update its row:
```markdown
| ADR-OOO | <Old Title> | superseded-by ADR-NNN | <old date> |
```

### Commit

```bash
git add CONTEXT.md docs/adr/ DECISIONS.md
git commit -m "arch: capture decisions from architectural review of <feature/topic>

Decisions recorded:
  - ADR-NNN: <decision title>

CONTEXT.md updates:
  - Glossary: added/updated <N> terms
  - Invariants: added <N> new invariant(s)
  - Sharp Edges: added <N> item(s)"
```

Ask user before committing.

---

## Phase 7 — HANDOFF

```
## Architectural Review Complete

**Plan status:** Approved ✅ | Approved with changes ⚠️ | Needs rework ❌

**Decisions captured:**
  → ADR-NNN: <title> (docs/adr/ADR-NNN-<slug>.md)

**CONTEXT.md updated:**
  → Glossary: added/updated <N> terms
  → Invariants: added <N> new invariant(s)
  → Sharp Edges: added <N> item(s)

**Still open:**
  → <Open question 1> — suggested next action
  → <Open question 2>

**Recommended next step:**
  <If approved>  → Run prd-writer to formalise the design into a PRD
  <If rework>    → Revisit [specific aspect] before proceeding
  <If open qs>   → Resolve [specific question] — consider a type:spike issue
```

---

## Special Modes

### Contested Decision

When the user presents two competing approaches:
1. State each option as a crisp one-sentence position
2. Apply the challenge lenses to each
3. Name the actual trade-off (there is always one)
4. Make a recommendation — don't stay neutral:
   > "Given [specific context from CONTEXT.md], I'd recommend Option A because
   > [reason]. Option B makes sense if [specific condition] — does that apply here?"
5. Write the chosen option as an ADR once decided

### New Abstraction Review

When the user is introducing a new abstraction (service, entity, pattern):
1. Ask for a one-sentence definition
2. Check against Domain Glossary for collision or overlap
3. Ask: what does this do? What does it NOT do? Who owns it? What is its lifecycle?
4. Name any existing abstraction it resembles — make the distinction explicit
5. Write it to the glossary if it belongs there

### Pre-PRD Alignment

When called before prd-writer to stress-test a concept:
1. Run Phases 1–5 at a lighter pace — identify blockers, not every nuance
2. Focus on: invariant conflicts, terminology, and the one hardest question
3. Produce a short "architectural notes" block the user pastes into the PRD's
   Technical Constraints section
4. Hand off: "These design decisions are resolved. Run prd-writer — pass these
   notes as the Technical Constraints section."

---

## Conversation Style

**Ask one thing at a time.** Multiple challenges at once cause the user to
address the easiest one and ignore the hard ones.

**Wait for answers.** The user's answer often reveals something important.

**Be direct about concerns.** "This might be worth considering" is too soft.

**Acknowledge good design.** Positive signal is useful.

**Don't let perfection block progress.** If 80% of the plan is solid, capture
the 80% and flag the 20% as open questions.

**The goal is a written record.** Every decision that stays in someone's head is
a future point of confusion. The skill succeeds when crystallised thinking gets
written down.

---

## Error Cases

**No CONTEXT.md or ADRs:** Tell the user. Offer lighter review against their
description alone, but note the review will miss project-specific conflicts.
Recommend running project-bootstrap.

**The plan is sound:** Say so clearly and don't manufacture concerns. "This plan
is consistent with the existing architecture. The main decision to capture is [X]."
Then write the ADR for X and move to handoff.

**The plan has a fundamental flaw:** Be direct:
> "This design will [specific consequence] because [specific reason]. I'd
> recommend stepping back and considering [alternative]."

**The user disagrees with a challenge:** Acknowledge their counter-argument. If
it's persuasive, update your view. If you still see the concern:
> "That's fair — [acknowledge their point]. I still think [concern] because
> [reason]. Ultimately it's your call — I'd want that trade-off explicit in the ADR."

**Conversation going in circles:** Synthesise and force a decision:
> "We've been circling around [question]. The two positions are [A] and [B].
> Which one are we going with? I'll write the ADR either way."

---

## Reference files

- `references/challenge-lenses.md` — the seven lenses with priority guide and signal phrases
- `references/adr-writing-guide.md` — what makes ADRs actually useful vs. bureaucratic

Reads from: `CONTEXT.md`, `docs/adr/`, `DECISIONS.md`, `AGENTS.md`, `docs/prd/`, source code
Writes to: `docs/adr/ADR-NNN-<slug>.md`, `CONTEXT.md`, `DECISIONS.md`

Position in the skill chain:
- Called **before** `prd-writer` (resolve architecture before writing requirements)
- Complementary to `arch-auditor` (auditor looks backward; architect looks forward)
- Downstream: `prd-writer` → `issue-planner` → `tdd-agent`
