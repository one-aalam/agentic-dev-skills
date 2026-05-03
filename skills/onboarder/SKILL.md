---
name: onboarder
description: Synthesises the project's knowledge layer into a personalised onboarding document for a new human or agent joining the codebase. Trigger when someone new is joining, an agent is being introduced to a project, or the user says "onboard me", "explain this codebase", "I'm new here", "what do I need to know", or "create an onboarding guide". Reads CONTEXT.md, AGENTS.md, ADRs, and codebase structure — writes docs/onboarding.md. Asks one question before writing: who is being onboarded?
---

# Onboarder Skill

Every project has implicit knowledge — the why behind the what, the invariants
that must never be broken, the terms that mean something specific here, the sharp
edges that will burn you if you don't know about them. That knowledge lives in
`CONTEXT.md`, in ADRs, in commit history, in the architecture. This skill reads
all of it and synthesises a single, scannable document for whoever is joining.

The output is not a tutorial. It is a map. It tells the reader:
- What this system does and why it exists
- The language of this domain — what words mean here
- The rules that cannot be broken
- Where the bodies are buried (sharp edges)
- What to read next and in what order
- How to get something done on day one

The document is written differently depending on who is being onboarded.
A human engineer needs narrative and context. An agent needs precise rules
and the structure of what it may and may not do. The skill asks first.

---

## Inputs Expected

The user provides one of:
- Nothing — skill will ask one question before proceeding
- A role: "onboard a new frontend engineer"
- An agent declaration: "onboard a new agent to this project"
- A name + role: "create onboarding for Sarah, she's joining as a backend engineer"
- "update the onboarding doc" — refresh existing `docs/onboarding.md`
- "onboard me" — the user themselves is new to the project

---

## Phase Overview

```
Phase 1 — ASK          One question: who and what role?
Phase 2 — INGEST       Read the full knowledge layer
Phase 3 — EXPLORE      Sample the codebase structure
Phase 4 — SYNTHESISE   Build the onboarding document in memory
Phase 5 — WRITE        Produce docs/onboarding.md
Phase 6 — HANDOFF      Surface the document and next actions
```

---

## Phase 1 — ASK

If the role is not already clear from the trigger, ask exactly one question
before doing anything else:

> "Who is being onboarded — a human (and if so, what role: frontend, backend,
> fullstack, devops, PM?) or an agent?"

Wait for the answer. Do not proceed until you know. The document is written
fundamentally differently depending on the answer.

If "update the onboarding doc" is the trigger: skip to Phase 2. The existing
doc determines the persona.

---

## Phase 2 — INGEST

Read the complete knowledge layer. Do not skip any file — the onboarding
document is only as good as what you read here.

```bash
# Core knowledge files
cat CONTEXT.md
cat AGENTS.md
cat DECISIONS.md

# All accepted ADRs
for f in docs/adr/*.md; do
  echo "=== $(basename $f) ==="
  cat "$f"
  echo
done

# Existing PRDs for product context
ls docs/prd/*.md 2>/dev/null | grep -v TEMPLATE | head -10
# Read the most recent 2-3 PRDs for product context

# Existing onboarding doc (if updating)
cat docs/onboarding.md 2>/dev/null

# Recent significant commits (for "what's been happening lately")
git log --oneline --since="30 days ago" | head -20

# Open issues (what's being worked on right now)
ls issues/*.md 2>/dev/null | head -10
grep -h "^title:\|^priority:\|^status:" issues/*.md 2>/dev/null | head -30
```

Extract and hold:
- **Project purpose** — the one-sentence answer to "what does this system do?"
- **Domain Glossary** — every term and definition
- **Key Invariants** — the rules that cannot be broken
- **Sharp Edges** — the gotchas
- **Tech Stack** — what is being worked with
- **Architecture** — how the pieces fit together
- **External Dependencies** — what the system talks to
- **Agent rules** — what agents may and may not do (from AGENTS.md)
- **ADR decisions** — the settled decisions and why
- **Current work** — open issues, recent commits

---

## Phase 3 — EXPLORE

Sample the codebase structure to give the onboardee a map of the code itself.
Do not read every file — sample enough to describe the shape.

```bash
# Top-level structure
ls -la

# Source directory shape (2 levels deep)
find src/ -maxdepth 2 -type d 2>/dev/null | sort
# or: find lib/ app/ internal/ -maxdepth 2 -type d 2>/dev/null | sort

# Entry points
ls src/index.* src/main.* src/app.* src/server.* 2>/dev/null
cat <entry_point_file> | head -40

# Test structure
find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | \
  sed 's|/[^/]*$||' | sort | uniq | head -10

# Recent files touched (what's hot right now)
git log --since="14 days ago" --name-only --format="" | sort | uniq -c | \
  sort -rn | head -15

# Config and environment
ls .env.example Makefile docker-compose.yml 2>/dev/null
cat .env.example 2>/dev/null | head -20
```

Build a mental map of:
- **Where things live** — which directory does what
- **The entry point** — where execution begins
- **What's active** — the hot spots of recent change
- **How to run it** — from AGENTS.md run commands

---

## Phase 4 — SYNTHESISE

Compose the onboarding document in memory. The structure and depth differ
by persona. Choose the right variant.

---

### Variant A — Human Engineer

**Tone:** Narrative and contextual. Answers the "why" before the "what".
Assumes they can read code but don't know *this* codebase or *this* domain.

**Structure:**

```
1. What this system does (2-3 sentences — the honest, jargon-free version)
2. The domain (key concepts and the glossary — definitions they'll encounter everywhere)
3. The architecture (how the pieces fit — a simple map, not a diagram)
4. The rules (key invariants — non-negotiable constraints)
5. The sharp edges (what will bite them if they don't know)
6. The tech stack (what they're working with and any non-obvious choices)
7. How to run it (the commands from AGENTS.md)
8. What's being worked on right now (current issues and recent commits)
9. First things to read (a prioritised reading list — max 5 items)
10. First thing to do (one concrete action to take on day one)
```

**Writing rules for human variant:**
- Lead every section with the most important sentence. Don't bury the lede.
- Use the domain terms from the glossary — but define them on first use.
- Keep sections short. This is a map, not a textbook.
- The "sharp edges" section should read like advice from a teammate, not a warning label.
- The "first thing to do" should be genuinely achievable in 30 minutes.

---

### Variant B — Agent

**Tone:** Precise and operational. Answers "what are my constraints" before
"what does this system do". Agents don't need narrative — they need rules,
structure, and exact terminology.

**Structure:**

```
1. Project identity (name, purpose, stack — one line each)
2. What agents may do autonomously (verbatim from AGENTS.md)
3. What agents must ask a human first (verbatim from AGENTS.md)
4. Key Invariants (verbatim list — agents treat these as absolute constraints)
5. Domain Glossary (full table — agents use these terms exclusively)
6. Codebase map (directory → purpose, one line each)
7. Run commands (exact commands from AGENTS.md)
8. Sharp Edges (operational warnings — what to check before touching each area)
9. Accepted ADRs (summary of settled decisions — what patterns to follow)
10. How to find work (how to read issues/, what agent:ready means)
```

**Writing rules for agent variant:**
- Use the exact terms from CONTEXT.md — no paraphrasing, no synonyms.
- Invariants are listed as hard constraints, not suggestions.
- The "Sharp Edges" section maps each gotcha to the relevant file or module.
- ADR decisions are summarised as rules: "Use X, not Y, because Z."
- Everything must be verifiable from the documents — no inferred claims.

---

### Variant C — Monorepo (either persona)

If the project is a monorepo (multiple packages detected in Phase 3):

- Open with the overall system purpose
- Add a **Package Map** section between Architecture and Rules:
  ```
  | Package | Path | Purpose | CONTEXT.md |
  |---------|------|---------|------------|
  ```
- Note which packages the onboardee will primarily work in
- Link to per-package `CONTEXT.md` files for deeper reading

---

## Phase 5 — WRITE

Write `docs/onboarding.md`. Create the directory if needed.

```bash
mkdir -p docs
```

### Full document format (Human variant)

```markdown
# Onboarding: <Project Name>

**For:** <role — e.g., Backend Engineer>
**Date:** <YYYY-MM-DD>
**Maintained by:** onboarder skill — re-run after significant changes

> This document is generated from the project's knowledge layer (CONTEXT.md,
> AGENTS.md, ADRs). If something here seems wrong, the source of truth is those
> files — and you should run the `context-updater` skill after verifying.

---

## What This System Does

<2–3 sentences. The honest version. What does it do, for whom, and why does it exist?
Written in plain language, not marketing language.>

---

## The Domain

Before reading code, read these terms. They appear everywhere and mean specific
things in this project — not necessarily what they mean elsewhere.

| Term | What it means here |
|------|-------------------|
| <Term> | <Definition from CONTEXT.md glossary> |
| <Term> | <Definition> |

<If any terms have non-obvious relationships or common confusions, call them out
in a short paragraph here. Keep it to the 2-3 most important ones.>

---

## How It's Built

<3–5 sentences describing the architecture. Written as a map, not a diagram.
"The system is structured as X. Requests flow through Y. Data lives in Z.
The main entry point is <path>. The hot spots right now are <areas>.">

**Where things live:**

| Directory | What's in it |
|-----------|-------------|
| `src/<module>/` | <purpose> |
| `src/<module>/` | <purpose> |

---

## The Rules

These are the non-negotiable constraints. They have been established deliberately
and exist for specific reasons. Violating them will be caught in code review.

<For each Key Invariant in CONTEXT.md:>
- **<Invariant>** — <one sentence on why it exists, if known from ADRs>

---

## Sharp Edges

Things that will trip you up if you don't know about them. Learn these before
you touch the relevant code.

<For each Sharp Edge in CONTEXT.md:>
- **<Gotcha>** — <what to watch for and where>

---

## The Tech Stack

| Layer | What | Notes |
|-------|------|-------|
| <layer> | <technology> | <anything non-obvious about how it's used here> |

---

## How to Run It

```bash
# Install
<INSTALL_COMMAND from AGENTS.md>

# Run tests
<TEST_COMMAND>

# Start dev server
<DEV_COMMAND>
```

<If there are any non-obvious setup steps (env vars, seed data, services to start),
list them here. Derive from .env.example and AGENTS.md.>

---

## What's Being Worked On

**Current open issues:**
<List 3-5 open issues with their priority and a one-line description.>

**Recent changes (last 2 weeks):**
<2-3 sentences on what has been happening in the codebase recently, derived from
git log. Written as "the team has been..." not as a list of commit messages.>

---

## What to Read Next

In this order:

1. `CONTEXT.md` — the full project knowledge base (you've seen the highlights above)
2. `AGENTS.md` — how this project works with AI agents, and the commit convention
3. `docs/adr/` — the settled architectural decisions (start with ADR-001)
4. <Most relevant PRD for current work>
5. `<entry_point_file>` — where execution begins

---

## Your First Action

<One concrete, achievable action that will get them oriented in the code:>

> Run `<TEST_COMMAND>` to confirm your setup is working, then open
> `<most_relevant_file>` and read through it. It is the best entry point into
> understanding how <core concept> works in this codebase.
```

---

### Full document format (Agent variant)

```markdown
# Agent Onboarding: <Project Name>

**Date:** <YYYY-MM-DD>
**Source:** Generated from CONTEXT.md, AGENTS.md, and docs/adr/
**Refresh:** Re-run onboarder skill after significant changes to the knowledge layer

---

## Project Identity

**Name:** <project name>
**Purpose:** <one sentence from CONTEXT.md>
**Stack:** <comma-separated list from Tech Stack table>
**Repo root:** <path>

---

## Autonomous Allowlist

You MAY do the following without asking a human:

<Verbatim from AGENTS.md "What Agents May Do Autonomously" section>

---

## Human Approval Required

You MUST ask a human before:

<Verbatim from AGENTS.md "What Agents Must Ask a Human First" section>

---

## Key Invariants

Treat these as absolute constraints. No exceptions without an explicit ADR.

<For each invariant in CONTEXT.md, formatted as:>
- **INVARIANT:** <exact text> — <one-line consequence if violated>

---

## Domain Glossary

Use these terms exclusively. Do not paraphrase, do not invent synonyms.

| Term | Definition |
|------|-----------|
<Full glossary table from CONTEXT.md>

---

## Codebase Map

| Path | Purpose |
|------|---------|
| `src/<module>/` | <purpose> |
| `docs/.agent/` | Internal agent system dir — labels, prompts, workflows |
| `docs/adr/` | Architecture Decision Records — read before proposing new patterns |
| `docs/prd/` | Product Requirement Documents — source of acceptance criteria |
| `issues/` | Tracked issues — pick up `agent:ready` labelled ones |
| `docs/reports/` | Audit and diagnosis reports |

---

## Run Commands

```bash
# Install:  <INSTALL_COMMAND>
# Test:     <TEST_COMMAND>
# Lint:     <LINT_COMMAND>
# Build:    <BUILD_COMMAND>
```

Commit convention: `<type>(<scope>): <description>` — see AGENTS.md for types.
Always reference the issue ID in the commit message: `feat(ISS-042): ...`

---

## Sharp Edges

Know these before touching the relevant code:

<For each Sharp Edge in CONTEXT.md:>
- **<Gotcha>** — affects `<relevant path>`. <What to check / what to avoid.>

---

## Settled Decisions (ADRs)

These have been decided. Do not re-litigate without an explicit ADR.

<For each accepted ADR, one line:>
- **ADR-NNN:** <decision in one sentence> → use `<pattern>`, not `<rejected alternative>`

---

## How to Find Work

1. Check `issues/` for files with `agent:ready` label
2. Verify `depends-on:` issues are merged before starting
3. Read the linked PRD and issue in full before writing any code
4. Run `tdd-agent` workflow: failing tests first, then implementation

**`agent:ready` contract:** An issue carries this label only when it has specific
checkable acceptance criteria, at least one file path hint, no open questions,
priority P1 or lower, and scope bounded to ≤5 files.
```

---

### When updating an existing document

If `docs/onboarding.md` already exists:

1. Read it fully
2. Diff it against the current state of the knowledge layer
3. Identify sections that are stale (outdated terms, wrong stack versions,
   closed issues still listed, etc.)
4. Propose specific updates using the same `[y/e/n]` confirmation model as
   `context-updater`
5. Update the "Date" frontmatter field
6. Add a one-line entry to a `## Revision History` section at the bottom

Do not rewrite the whole document — only the sections that have drifted.

---

## Phase 6 — HANDOFF

```
✅ docs/onboarding.md written

  Persona:   <Human: Backend Engineer | Agent>
  Sections:  <N> — <list section names>
  Sources:   CONTEXT.md, AGENTS.md, <N> ADRs, <N> issues

<If human:>
Share this with <name> and point them to the "First Action" section first.
After their first week, run the onboarder again with "update the onboarding doc"
to refresh anything that felt wrong or incomplete.

<If agent:>
Pass docs/onboarding.md as system context when initialising the agent session.
The agent should read it before any other file in the project.

<If the knowledge layer was thin:>
⚠️ Some sections are sparse because CONTEXT.md or AGENTS.md are incomplete.
Sections affected: <list>
Run project-bootstrap or context-updater to fill these gaps, then re-run onboarder.
```

---

## Quality Checks Before Writing

Before committing the document, verify:

```
□ Every domain term used in the document is defined in the Glossary section
□ Every invariant listed matches CONTEXT.md exactly (no paraphrasing)
□ Run commands are copied from AGENTS.md verbatim (not from memory)
□ Sharp edges are sourced from CONTEXT.md — no invented warnings
□ ADR summaries match the actual ADR decisions (no approximations)
□ "First Action" is genuinely achievable in 30 minutes by the target persona
□ Agent variant: no inferred claims — every statement sourced from a file
```

Fail any check → fix it before writing. An onboarding doc with wrong information
is worse than no onboarding doc.

---

## Refresh Triggers

Re-run this skill when:
- A new human or agent joins the project (obvious)
- `CONTEXT.md` has changed significantly (new invariants, renamed concepts, new stack)
- A major feature ships and the architecture description is outdated
- The "current work" section is more than 4 weeks old
- Someone says "the onboarding doc was wrong about X"

The document should never be more than one significant feature behind reality.

---

## Error Cases

**CONTEXT.md missing or empty:** The onboarding document will be skeletal.
Write what can be derived from AGENTS.md and codebase exploration alone.
Add a prominent warning at the top: "This project has no CONTEXT.md. The
onboarding document is incomplete. Run project-bootstrap to fix this."

**AGENTS.md missing:** Cannot fill in the run commands or agent rules sections.
Leave them as `<fill in>` placeholders with a note. Recommend running
project-bootstrap.

**No ADRs:** Omit the ADR/Settled Decisions section for humans. For agents,
note explicitly: "No ADRs found — architectural decisions are not documented.
Be cautious about introducing new patterns without checking with a human."

**Monorepo with no per-package CONTEXT.md files:** Write the onboarding for the
root level only. Add a note: "This is a monorepo. Per-package context files are
missing — run project-bootstrap on each package you'll be working in."

**Onboarding doc already exists and is recent (< 2 weeks old):** Tell the user
and ask: "An onboarding doc exists from <date>. Should I update it or start fresh?"

---

## Reference

Reads: `CONTEXT.md`, `AGENTS.md`, `DECISIONS.md`, `docs/adr/`, `docs/prd/`,
       `issues/`, git log, codebase structure, `.env.example`
Writes: `docs/onboarding.md`

Position in skill chain:
- Run when a new human or agent joins — no upstream dependency
- Should be run **after** project-bootstrap (requires a knowledge layer to synthesise)
- Re-run after `context-updater` makes significant changes to CONTEXT.md
- The agent variant feeds directly into agent initialisation as system context
