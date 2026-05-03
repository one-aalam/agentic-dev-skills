# File Templates

Canonical templates for every file the project-bootstrap skill creates.
Substitute `<PLACEHOLDER>` values from project discovery.

---

## AGENTS.md

```markdown
# AGENTS.md
<!-- Instructions for AI agents working in this repository -->
<!-- Update this file whenever tooling, patterns, or constraints change -->

## Run Commands

```bash
# Install dependencies
<INSTALL_COMMAND>

# Run tests
<TEST_COMMAND>

# Lint / typecheck
<LINT_COMMAND>

# Build
<BUILD_COMMAND>

# Start dev server
<DEV_COMMAND>
```

## Repository Layout

```
<PROJECT_LAYOUT>
```

## Constraints & Rules

- Never commit directly to `main` / `master` — always use a branch + PR
- All changes touching `<SENSITIVE_PATH>` require a human review before merge
- Do not modify files in `<GENERATED_DIR>` — they are auto-generated
- Migrations: any schema change must include a migration file in `<MIGRATIONS_DIR>`
- Secrets: never hardcode credentials; use environment variables from `.env.example`

## What Agents May Do Autonomously

- Write and run tests
- Refactor within a single module without changing public interfaces
- Add new files following existing conventions
- Pick up issues labelled `agent:ready` and open PRs
- Update `CONTEXT.md` changelog section after significant changes

## What Agents Must Ask a Human First

- Breaking changes to public APIs or interfaces
- Changes that affect `<AUTH_OR_PAYMENT_PATH>`
- Adding new external dependencies (npm/pip/cargo packages)
- Deleting files or directories
- Any change affecting more than 5 files at once

## Tool Availability

- Git: yes
- GitHub CLI (`gh`): <YES|NO>
- Docker: <YES|NO>
- MCP servers available: listed in `docs/.agent/agents.index.md`

## Commit Convention

Format: `<type>(<scope>): <description>`
Types: feat | fix | chore | docs | test | refactor | perf
Example: `feat(auth): add refresh token rotation (ISS-042)`

Always reference the issue ID in the commit message when applicable.

## Test Strategy

- Write failing tests first (red → green → refactor)
- Test files live alongside source: `<test_colocation_pattern>`
- Coverage target: <COVERAGE_TARGET>%
- Do not skip or mock in ways that defeat the test's purpose
```

---

## CONTEXT.md

```markdown
# CONTEXT.md
<!-- Living knowledge base for this project -->
<!-- Updated: <DATE> -->
<!-- Confidence: sections marked [stale-risk: high] should be verified before use -->

## Project Overview

**Name:** <PROJECT_NAME>
**Purpose:** <ONE_SENTENCE_PURPOSE>
**Status:** <active | maintenance | deprecated>
**Codebase snapshot:** <!-- git sha updated by post-merge hook -->

## Tech Stack

| Layer      | Technology     | Version  | Notes                        |
|------------|---------------|----------|------------------------------|
| Language   | <LANGUAGE>    | <VER>    |                              |
| Framework  | <FRAMEWORK>   | <VER>    |                              |
| Database   | <DATABASE>    | <VER>    |                              |
| Hosting    | <PLATFORM>    |          |                              |
| CI/CD      | <CI_PLATFORM> |          |                              |

## Domain Glossary

> Critical — agents use these definitions. Incorrect terms lead to wrong code.

| Term | Definition | Notes |
|------|------------|-------|
| <TERM> | <DEFINITION> | |

## Architecture Overview

<Describe the major components and how they relate. Include a simple ASCII diagram
if helpful. Focus on decisions that aren't obvious from reading the code.>

## Key Invariants

> Things that must always be true. Agents: treat these as absolute constraints.

- <INVARIANT_1>
- <INVARIANT_2>

## External Dependencies

| Service | Purpose | Rate Limits | Credentials Location |
|---------|---------|-------------|----------------------|
| <SERVICE> | <PURPOSE> | <LIMITS> | `.env` → `<VAR_NAME>` |

## Sharp Edges & Gotchas

<!-- Things that have burned us or will burn an agent that doesn't know -->
- <GOTCHA_1>

## Changelog

<!-- Updated by post-merge hook or manually after significant decisions -->
| Date | Change | Author |
|------|--------|--------|
| <DATE> | Initial bootstrap | project-bootstrap skill |
```

---

## DECISIONS.md

```markdown
# DECISIONS.md
<!-- Index of Architecture Decision Records -->
<!-- Full ADRs live in docs/adr/ -->

| ID      | Title                          | Status   | Date       |
|---------|-------------------------------|----------|------------|
| ADR-000 | ADR template                  | template | <DATE>     |

<!-- Add new rows as ADRs are created. Agents: read docs/adr/<ID>.md for full context. -->
```

---

## docs/adr/ADR-000-template.md

```markdown
# ADR-000: <Decision Title>

**Date:** <YYYY-MM-DD>
**Status:** proposed | accepted | deprecated | superseded-by ADR-XXX
**Deciders:** <names or roles>

## Context

<What situation or problem forced this decision? What constraints existed?
Be specific to this project — a new engineer must understand why this choice
was necessary here, not just generically.>

## Decision

<What was decided, stated clearly and directly. Lead with one sentence.>

## Alternatives Considered

| Option | Pros | Cons | Why Rejected |
|--------|------|------|--------------|
| <OPTION> | | | |

## Consequences

**Positive:** <What gets better>
**Negative:** <What gets worse or harder — be specific>
**Risks:** <What could go wrong>

## Notes

<Any follow-up actions, related issues, or links>
```

---

## docs/prd/TEMPLATE.md

```markdown
# PRD: <Feature Name>

**Status:** draft | review | approved | shipped
**Author:** <name>
**Date:** <YYYY-MM-DD>
**Codebase snapshot:** <!-- git sha at time of writing -->
**Related issues:** <!-- links to ISS-*.md or GH issue numbers -->

## Problem Statement

<What user problem or business need does this address? One paragraph.
No solution language here — just the problem. Use domain terms from CONTEXT.md.>

## Success Metrics

| Metric | Baseline | Target | Measurement Method |
|--------|----------|--------|--------------------|
| <METRIC> | | | |

## Users & Context

**Primary user:** <role from CONTEXT.md glossary>
**Trigger:** <what causes the user to need this feature>
**Current workaround:** <how they do it today, if at all>

## User Stories

- As a **<role>**, I want to **<action>** so that **<outcome>**.

## Acceptance Criteria

<!-- BDD format — TDD agents parse these directly into test scaffolds -->
<!-- Use Given/When/Then strictly. Be specific — avoid "should work" -->

### Scenario 1: <Happy path name>
```gherkin
Given <a specific precondition>
When <the user takes a specific action>
Then <a specific, verifiable outcome occurs>
```

### Scenario 2: <Error case name>
```gherkin
Given <precondition>
When <action>
Then <outcome>
```

## Out of Scope

- <EXPLICITLY_EXCLUDED_1>

## Technical Constraints

<!-- Reference CONTEXT.md invariants that apply here -->
- Must comply with: <INVARIANT>

## Open Questions

| # | Question | Owner | Due | Resolution |
|---|----------|-------|-----|------------|
| 1 | | | | — |

## Implementation Plan

<!-- Planner agent fills this in. Links to issues/ or GH issues. -->
- [ ] ISS-XXX: <task>
```

---

## docs/.agent/agents.index.md

```markdown
# Agent Registry
<!-- Index of all specialist agents available in this project -->
<!-- Each entry: role, prompt location, trigger conditions, output location -->

| Agent        | Prompt                              | Trigger                    | Output            |
|--------------|-------------------------------------|----------------------------|-------------------|
| diagnoser    | docs/.agent/prompts/diagnoser.md    | failing test / bug report  | docs/reports/     |
| triager      | docs/.agent/prompts/triager.md      | new issue filed            | labels + ISS-*.md |
| prd-writer   | docs/.agent/prompts/prd-writer.md   | "write a PRD for X"        | docs/prd/         |
| planner      | docs/.agent/prompts/planner.md      | approved PRD               | issues/ or GH     |
| tdd-agent    | docs/.agent/prompts/tdd-agent.md    | issue labelled agent:ready | src/ + tests/     |
| arch-auditor | docs/.agent/prompts/arch-auditor.md | scheduled / on-demand      | docs/reports/     |

## MCP Servers

<!-- List MCP servers available to agents in this project -->
| Server | URL | Capabilities |
|--------|-----|--------------|
| <!-- add entries --> | | |

## Environment

Agents inherit environment variables from `.env`. Required vars are listed in
`.env.example`. Never hardcode secrets.
```

---

## docs/.agent/workflows/gh-labels-sync.md

```markdown
# Workflow: Sync GitHub Labels

**Run when:** Labels in `docs/.agent/labels.md` change, or on repo init.
**Requires:** `gh` CLI authenticated, run from repo root.

## Steps

1. Optionally delete GitHub's default labels for a clean slate:
```bash
gh label list --json name -q '.[].name' | xargs -I {} gh label delete "{}" --yes
```

2. Run the `gh label create` commands from `docs/.agent/labels.md`.

3. Verify:
```bash
gh label list
```

## Notes

- Use `--force` flag on create to make the operation idempotent
- Customise `domain:*` labels before importing — the defaults are stubs
```

---

## CONTEXT.index.md (monorepo only)

```markdown
# CONTEXT Index
<!-- Root map for monorepo domain knowledge -->
<!-- Each package maintains its own CONTEXT.md; this file is the entry point -->

## Packages

| Package | Path | CONTEXT.md | Domain |
|---------|------|------------|--------|
| <PACKAGE_NAME> | `packages/<name>/` | [CONTEXT.md](packages/<name>/CONTEXT.md) | <DOMAIN> |

## Shared Invariants

> These apply across ALL packages in this monorepo.

- <SHARED_INVARIANT_1>

## Shared Tech

| Layer | Technology | Notes |
|-------|-----------|-------|
| Monorepo tooling | <turborepo/nx/lerna> | |
| Shared config | `packages/config/` | |

## Cross-cutting Concerns

- Auth: handled by `packages/<auth-package>/`
- Logging: `packages/<logging-package>/`
- Feature flags: <where>

## Agent Notes

When working across packages, always read:
1. This file first
2. The relevant package CONTEXT.md
3. Any shared package CONTEXT.md if crossing boundaries
```
