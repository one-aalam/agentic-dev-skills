# CONTEXT.md
<!-- Living knowledge base for the agentic-dev-skills repository -->
<!-- Updated: 2025-05-03 -->

## Project Overview

**Name:** agentic-dev-skills
**Purpose:** A composable library of Claude skills that give any software project a complete agentic knowledge architecture — structured knowledge files, issue tracking discipline, triage taxonomy, and specialised agent workflows for diagnosing, planning, building, and auditing code.
**Status:** active

---

## Domain Glossary

> These definitions are used precisely throughout all skills. Agents: do not paraphrase or invent synonyms.

| Term | Definition | Notes |
|------|-----------|-------|
| **Skill** | A self-contained Claude capability defined by a `SKILL.md` file with YAML frontmatter (name + description) and structured markdown instructions. May include `references/` files loaded on demand. | `.skill` files are zip archives of a skill directory |
| **SKILL.md** | The primary file of a skill. Frontmatter contains the triggering description; body contains phased instructions for Claude to follow. | Keep under 500 lines; use references/ for overflow |
| **references/** | A subdirectory within a skill containing supporting documents loaded on demand. Never loaded automatically; explicitly referenced from SKILL.md. | |
| **Knowledge layer** | The set of structured files a project maintains for agents: `AGENTS.md`, `CONTEXT.md`, `DECISIONS.md`, `docs/adr/`, `docs/prd/`, `docs/.agent/`. Created by `project-bootstrap`. | |
| **AGENTS.md** | Root-level file containing instructions for AI agents: run commands, constraints, what agents may do autonomously vs. must ask a human first. | Required reading for all skills before touching code |
| **CONTEXT.md** | Root-level living knowledge base: tech stack, domain glossary, key invariants, architecture overview, external dependencies, sharp edges, and changelog. | Agents treat Key Invariants as absolute constraints |
| **ADR** | Architecture Decision Record. A structured document capturing a significant architectural decision: context, decision, alternatives, consequences. Lives in `docs/adr/`. | |
| **DECISIONS.md** | Index file mapping ADR IDs to titles and statuses. Quick lookup; full ADRs live in `docs/adr/`. | |
| **Key Invariant** | A constraint that must always be true of the target project's codebase. Skills treat these as hard constraints — violations are always P1 or higher findings. | |
| **Domain Glossary** | The controlled vocabulary of a project's domain, defined in `CONTEXT.md`. All skills use only these terms when writing code, tests, or docs. | Terminology drift is a bug before it is a bug in code |
| **`docs/.agent/`** | The internal system directory created by `project-bootstrap`. Contains the label taxonomy, agent registry, per-role prompt stubs, and workflow runbooks. | Not user-facing docs; readable by agents |
| **Label taxonomy** | Five orthogonal label dimensions applied to every issue: Type, Priority, Domain, Status, Agent Readiness. Defined in `docs/.agent/labels.md`. | |
| **`agent:ready`** | An issue label that is a contract: specific checkable acceptance criteria, at least one file path hint, no open questions, priority P1 or lower, not `type:security`, bounded to ≤5 files. | P0 and type:security must never carry this label |
| **`agent:partial`** | An issue that an agent can start but needs a human checkpoint. Must include `missing-for-agent:` frontmatter listing exactly what is missing. | |
| **Phase** | A named stage within a skill's execution (ORIENT, CHALLENGE, WRITE, etc.). Skills are structured as explicit phases to make the workflow transparent and resumable. | |
| **Diagnosis report** | Output of the `diagnoser` skill. Structured markdown in `docs/reports/` containing root cause, evidence, affected paths, recommended fix, and risk. | Feed directly into triager to open an issue |
| **Audit report** | Output of the `arch-auditor` skill. Structured markdown in `docs/reports/` covering invariant compliance, ADR compliance, doc drift, debt hotspots, recommendations. | |
| **BDD** | Behaviour-Driven Development. Format used for acceptance criteria in PRDs: Given/When/Then scenarios in Gherkin syntax. Parsed directly by `issue-planner` and `tdd-agent`. | |
| **Red → green → refactor** | The mandatory TDD discipline enforced by `tdd-agent`. The failing-test commit (red) must exist before any implementation. Never skipped. | |
| **Challenge lens** | One of seven structured lines of questioning the `architect` skill applies to a plan: invariant integrity, ADR consistency, domain model coherence, boundary violations, unconsidered alternatives, downstream consequences, operability. | See `architect/references/challenge-lenses.md` |
| **Crystallise** | The moment a design decision under discussion becomes settled enough to write down. The `architect` skill writes ADRs and CONTEXT.md updates at the moment of crystallisation. | |

---

## Architecture Overview

This is a **skill library repository** — it contains no application code, only Claude skill definitions.

```
agentic-dev-skills/
├── README.md                     ← human-facing overview
├── CONTEXT.md                    ← this file; domain knowledge for agents
└── skills/                       ← one directory per skill
    ├── <skill-name>/
    │   ├── SKILL.md              ← required: frontmatter + phased instructions
    │   └── references/           ← optional: supporting lookup files
```

Each skill is **self-contained and independently usable**. Skills are also **composable** — the output artifact of one skill is the natural input of the next.

### Skill data flow

```
project-bootstrap
  └─ produces: AGENTS.md, CONTEXT.md, docs/.agent/, docs/adr/, docs/prd/
       └─ consumed by: all other skills

architect
  └─ produces: updated CONTEXT.md, new ADRs in docs/adr/
       └─ consumed by: prd-writer (Technical Constraints section)

prd-writer
  └─ produces: docs/prd/PRD-<slug>.md (with BDD acceptance criteria)
       └─ consumed by: issue-planner

issue-planner
  └─ produces: issues/ISS-NXX-*.md (one per atomic work unit)
       └─ consumed by: tdd-agent

diagnoser
  └─ produces: docs/reports/diagnosis-<id>.md
       └─ consumed by: triager

triager
  └─ produces: issues/ISS-NXX-*.md (structured, labelled)
       └─ consumed by: tdd-agent

tdd-agent
  └─ produces: source code + tests + PR
       └─ consumed by: humans (review), arch-auditor (periodic audit)

arch-auditor
  └─ produces: docs/reports/audit-<date>.md + auto-opened issues
       └─ consumed by: triager, diagnoser
```

---

## Key Invariants

- **Skills never act before reading AGENTS.md and CONTEXT.md** — every skill reads these in its ORIENT phase before any other action.
- **Skills never write without user confirmation** — every skill that writes files shows a write plan or draft and waits for approval. No silent writes to user-visible files.
- **`agent:ready` is a contract, not a suggestion** — all six criteria must be met. Partial satisfaction → `agent:partial`.
- **`tdd-agent` never skips the failing-test commit** — the red commit is mandatory.
- **The `architect` skill writes decisions at the moment of crystallisation** — not after the conversation ends.
- **Skills that find security issues stop and escalate** — never auto-open a security issue or describe an exploit path.
- **Skills never overwrite existing user content without showing a diff** — existing AGENTS.md, CONTEXT.md, or ADR content is shown as a merge proposal.
- **P0 issues are never `agent:ready`** — P0 findings always require human oversight.

---

## Skill Inventory

| Skill | Role | Direction | Writes |
|-------|------|-----------|--------|
| `project-bootstrap` | Initiator | Setup | AGENTS.md, CONTEXT.md, docs/.agent/, docs/adr/, docs/prd/ |
| `architect` | Forward reviewer | Before code | CONTEXT.md updates, docs/adr/ |
| `arch-auditor` | Backward reviewer | After the fact | docs/reports/audit-*.md, issues/ |
| `diagnoser` | Bug investigator | Read-only | docs/reports/diagnosis-*.md |
| `triager` | Issue structurer | Filing | issues/ISS-NXX-*.md |
| `prd-writer` | Requirements author | Before planning | docs/prd/PRD-*.md |
| `issue-planner` | Work decomposer | Before implementation | issues/ISS-NXX-*.md |
| `tdd-agent` | Implementer | Code-writing | src/, tests/, issues/ status |

---

## Sharp Edges & Gotchas

- **Skill descriptions are the triggering mechanism** — Claude decides whether to consult a skill based on the `description` frontmatter field. If a skill isn't triggering, the description needs more natural-language phrasings.
- **`references/` files are not auto-loaded** — they must be explicitly referenced from SKILL.md with guidance on when to read them.
- **Skills work best with a bootstrapped project** — `architect`, `triager`, and `arch-auditor` degrade significantly without CONTEXT.md and docs/.agent/.
- **`agent:ready` requires honest pessimism** — a missing file path hint or ambiguous acceptance criterion must be resolved before the label applies.
- **ADRs should be written during the decision, not after** — the `architect` skill is designed for this. ADRs written from memory have weak Context sections.
- **The `tdd-agent` has hard stops** — it pauses rather than proceeds if: a dependency is unmerged, priority is P0, type is security, criteria are ambiguous, or scope exceeds 5 files.

---

## Changelog

| Date | Change | Author |
|------|--------|--------|
| 2025-05-03 | Initial skill library: project-bootstrap + 7 specialist skills | one-aalam |
