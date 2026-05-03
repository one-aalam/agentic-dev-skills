# agentic-dev-skills

A collection of Claude skills that give any software project a complete **agentic knowledge architecture** — from first commit to production audit. Drop a skill into Claude and it knows how to run a structured workflow grounded in your actual codebase, docs, and domain decisions.

---

## What this is

Modern AI-assisted development works best when agents have a shared, structured knowledge layer: documented invariants, a domain glossary, architectural decision records, a triage taxonomy, and clear instructions for what they may and may not do autonomously. These skills create and maintain that layer — and use it.

Each skill is a standalone Claude skill (a `SKILL.md` plus optional `references/` files). They are designed to be composed: the output of one skill is the input of the next.

---

## The skill chain

```
project-bootstrap          ← run once on any new or existing project
        │
        ├── architect      ← stress-test a plan before writing requirements
        │       │
        │       └── prd-writer     ← formalise the approved design into a PRD
        │               │
        │               └── issue-planner  ← break the PRD into atomic issues
        │                       │
        │                       └── tdd-agent  ← implement each issue test-first
        │
        ├── diagnoser      ← investigate a bug → structured diagnosis report
        │       │
        │       └── triager        ← turn any report/description into a labelled issue
        │
        └── arch-auditor   ← periodic backward audit: drift, debt, violations
```

---

## Skills

### `project-bootstrap`
**Run once. Sets up everything else.**

Explores any repo (or empty directory), detects what exists, and creates the full agentic knowledge layer: `AGENTS.md`, `CONTEXT.md`, `docs/adr/`, `docs/prd/`, `docs/.agent/` (the internal system directory containing the label taxonomy, agent registry, and per-role prompt stubs). Handles monorepos with a `CONTEXT.index.md` map. Confirms before writing anything.

*Triggers on:* "set up my project for agents", "bootstrap this repo", "make this agentic", "prepare for Claude Code", "set up issue tracking"

---

### `architect`
**Forward-looking architectural thought partner.**

Stress-tests a plan against the project's domain model, existing ADRs, and invariants — before any code is written. Works through seven challenge lenses: invariant integrity, ADR consistency, domain model coherence, boundary violations, unconsidered alternatives, downstream consequences, and operability. Sharpens terminology against the Domain Glossary. Writes crystallised decisions directly into `CONTEXT.md` and `docs/adr/` during the conversation, so thinking becomes durable documentation.

*Triggers on:* "does this design make sense", "stress-test this plan", "challenge this approach", "is this consistent with our system", "should we use X or Y", "review my architecture"

*Reference files:*
- `references/challenge-lenses.md` — the seven lenses with priority guidance and signal phrases
- `references/adr-writing-guide.md` — what makes ADRs actually useful vs. bureaucratic

---

### `arch-auditor`
**Backward-looking periodic health check.**

Reads the codebase against `CONTEXT.md` invariants and all accepted ADRs. Checks for doc drift, debt hotspots (churn × no tests), TODOs, dead exports, dependency health, and security smells. Writes a structured audit report and auto-opens issues for P1+ findings. Produces a comparison table against the previous audit for trend tracking.

*Triggers on:* "audit the codebase", "check for drift", "is the codebase healthy?", "find technical debt", "run an audit"

*Reference files:*
- `references/invariant-checks.md` — grep strategies for every common invariant type

---

### `diagnoser`
**Read-only bug investigation.**

Takes a symptom (error, stack trace, failing test, description) and produces a structured diagnosis report in `docs/reports/`. Forms ranked hypotheses before reading any code. Verifies each hypothesis with code evidence. Never modifies source. Stops at security findings and escalates. Hands off cleanly to `triager` or `tdd-agent`.

*Triggers on:* "figure out what's wrong", "why is this failing", "root cause", "investigate this", "something broke" + pasted stack trace

---

### `triager`
**Turns raw descriptions and diagnosis reports into structured, labelled issues.**

Reads the label taxonomy from `docs/.agent/labels.md`. Applies all five label dimensions: Type, Priority, Domain, Status, and Agent Readiness. Enforces the `agent:ready` contract strictly (six required criteria). Adds `<!-- ESCALATE: P0 -->` sentinel for critical findings. Has a batch mode for backlog cleanup.

*Triggers on:* "triage this", "file an issue for", "add this to the backlog", "label this", "is this agent-ready?"

---

### `prd-writer`
**Writes PRDs grounded in the actual codebase — not generic templates.**

Interviews the user for the problem, affected users, success metrics, and scope boundary. Researches the codebase area being touched before writing Technical Constraints. Catches invariant conflicts before the PRD is approved. Writes BDD acceptance criteria in Gherkin format that `issue-planner` and `tdd-agent` consume directly.

*Triggers on:* "write a PRD for", "spec this out", "document requirements", "plan this feature", "define this feature"

---

### `issue-planner`
**Breaks a PRD into atomic, dependency-ordered, agent-labelled issues.**

Decomposes each BDD scenario into independently mergeable work units. Builds a dependency graph and sequences execution order. Size-checks each issue. Labels everything and assesses agent-readiness per issue. Shows the full plan as a table before writing anything. Links back to the PRD's Implementation Plan section after writing.

*Triggers on:* "break this into issues", "plan the implementation", "create tasks for this PRD", "decompose this feature"

---

### `tdd-agent`
**Implements issues test-first. Red → green → refactor. No exceptions.**

Reads `AGENTS.md` and `CONTEXT.md` before touching source. Validates the issue is truly `agent:ready`. Writes the failing test commit *before* any implementation — this commit is mandatory and non-skippable. Runs the full suite after implementation. Opens a PR with invariant compliance notes.

*Triggers on:* "implement ISS-XXX", "build this feature", "write the code for", "fix this bug", "make this test pass"

*Reference files:*
- `references/testing-patterns.md` — stack-specific patterns for TypeScript/Jest, Python/pytest, Go, and Rust

---

## What the skills read and write

```
AGENTS.md              ← read by: all skills (run commands, constraints)
CONTEXT.md             ← read by: all skills | written by: architect, project-bootstrap
DECISIONS.md           ← read by: architect, arch-auditor | written by: architect
docs/adr/              ← read by: architect, arch-auditor, prd-writer | written by: architect
docs/prd/              ← read by: issue-planner, tdd-agent | written by: prd-writer
docs/.agent/labels.md  ← read by: triager, issue-planner | written by: project-bootstrap
docs/.agent/prompts/   ← stub system prompts per agent role
docs/reports/          ← written by: diagnoser, arch-auditor
issues/                ← read by: tdd-agent | written by: triager, issue-planner, arch-auditor
```

---

## The `docs/.agent/` system directory

`project-bootstrap` creates this as the internal agent system layer — separate from human-facing docs:

```
docs/.agent/
├── labels.md           ← triage label taxonomy (5 dimensions, 20+ labels)
├── agents.index.md     ← registry of all agent roles + MCP servers
├── prompts/            ← stub system prompt per agent role
│   ├── diagnoser.md
│   ├── triager.md
│   ├── prd-writer.md
│   ├── planner.md
│   ├── tdd-agent.md
│   └── arch-auditor.md
└── workflows/
    └── gh-labels-sync.md
```

---

## Label taxonomy (summary)

| Dimension | Values |
|-----------|--------|
| **Type** | `type:bug` `type:feat` `type:chore` `type:spike` `type:debt` `type:docs` `type:security` |
| **Priority** | `P0` `P1` `P2` `P3` |
| **Domain** | `domain:auth` `domain:api` `domain:ui` `domain:data` `domain:infra` `domain:dx` `domain:perf` |
| **Status** | `status:needs-triage` `status:in-progress` `status:blocked` `status:needs-review` `status:wontfix` |
| **Agent readiness** | `agent:ready` `agent:partial` `agent:human-only` |

`agent:ready` is a contract: acceptance criteria specific + file path hinted + no open questions + priority P1 or lower + not `type:security` + scope ≤5 files.

---

## Repo structure

```
agentic-dev-skills/
├── README.md
├── CONTEXT.md
└── skills/
    ├── project-bootstrap/
    │   ├── SKILL.md
    │   └── references/
    │       ├── templates.md
    │       ├── labels-standard.md
    │       └── agent-prompts.md
    ├── architect/
    │   ├── SKILL.md
    │   └── references/
    │       ├── challenge-lenses.md
    │       └── adr-writing-guide.md
    ├── arch-auditor/
    │   ├── SKILL.md
    │   └── references/
    │       └── invariant-checks.md
    ├── diagnoser/
    │   └── SKILL.md
    ├── triager/
    │   └── SKILL.md
    ├── prd-writer/
    │   └── SKILL.md
    ├── issue-planner/
    │   └── SKILL.md
    └── tdd-agent/
        ├── SKILL.md
        └── references/
            └── testing-patterns.md
```
