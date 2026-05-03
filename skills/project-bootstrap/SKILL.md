---
name: project-bootstrap
description: >
  Bootstrap any new or existing project with a full agentic knowledge architecture.
  Trigger this skill whenever a user mentions starting a new project, setting up a repo,
  "greenfield", "greyfield", wants to add agent-ready structure to an existing codebase,
  asks about AGENTS.md / CONTEXT.md / agentic project setup, wants to set up issue tracking
  or triage labels, wants to establish domain docs or ADRs, or says anything like
  "set up my project for agents", "prepare this repo for Claude Code", "make this repo
  agentic", "initialise project structure", or "bootstrap this project".
  Also trigger when users ask how to track issues locally, set up a PRD system, or
  organise agent prompts. This skill handles the full discovery → confirmation → write
  → handoff cycle, including detecting whether a GitHub repo or local-only project is
  present.
---

# Project Bootstrap Skill

Sets up the complete agentic knowledge architecture for a project — from scratch or by
auditing what already exists. Produces `AGENTS.md`, `CONTEXT.md`, the `docs/` hierarchy,
label taxonomy, ADR scaffolding, and agent prompt stubs, then hands off cleanly.

---

## Phase Overview

```
Phase 1 — DISCOVER   Explore repo/directory, detect what exists
Phase 2 — PRESENT    Show findings + decisions + draft previews to user
Phase 3 — CONFIRM    User edits/approves what will be created
Phase 4 — WRITE      Quietly create internal files, show key files for review
Phase 5 — HANDOFF    Summary of what was created + how to extend it
```

---

## Phase 1 — DISCOVER

**Goal:** Understand the project before making any recommendations.

### 1.1 Find the project root

Ask the user for their project path if not already provided. If they pasted a path or
mentioned a directory in conversation, use that. Then:

```bash
# Check basic structure
ls -la <project_root>
git -C <project_root> rev-parse --is-inside-work-tree 2>/dev/null && echo "GIT_REPO" || echo "NO_GIT"
git -C <project_root> remote -v 2>/dev/null | grep -i github && echo "HAS_GITHUB" || echo "LOCAL_ONLY"
```

### 1.2 Detect existing bootstrap files

Check for presence of each file. Record as EXISTS / MISSING / PARTIAL (has content but
not the expected structure):

**Root-level agent files:**
- `AGENTS.md` — agent instructions
- `CONTEXT.md` — project knowledge base
- `DECISIONS.md` — ADR index
- `.claude/` — Claude-specific config dir

**Docs hierarchy:**
- `docs/adr/` — Architecture Decision Records
- `docs/prd/` — Product Requirement Docs
- `docs/.agent/` — Internal agent system dir (our standard)
- `docs/.agent/labels.md` — Triage label taxonomy
- `docs/.agent/agents.index.md` — Agent prompt registry
- `docs/.agent/workflows/` — Multi-step orchestration flows

**Issue tracking:**
- `.github/` — GitHub integration present
- `issues/` — Local git-native issue dir
- `TODO.md` — Flat todo tracking

**Agent prompts:**
- `agents/` or `docs/.agent/prompts/` — per-role system prompts

### 1.3 Detect project shape

```bash
# Monorepo detection
ls <project_root>/packages <project_root>/apps <project_root>/services 2>/dev/null

# Stack hints
cat <project_root>/package.json 2>/dev/null | head -20
cat <project_root>/pyproject.toml 2>/dev/null | head -20
cat <project_root>/go.mod 2>/dev/null | head -5
cat <project_root>/Cargo.toml 2>/dev/null | head -5

# Existing docs
find <project_root>/docs -name "*.md" 2>/dev/null | head -30

# Any existing README for context extraction
cat <project_root>/README.md 2>/dev/null | head -80
```

Detect:
- **Monorepo** → recommend `CONTEXT.index.md` that maps to per-package `CONTEXT.md` files
- **Single-package** → single root `CONTEXT.md`
- **Stack** → pre-fill stack section of `CONTEXT.md` template

---

## Phase 2 — PRESENT

**Goal:** Show the user a structured findings report and the specific decisions that need
their input. Do NOT ask multiple questions at once. Present findings, then ask for the
one most important fork first.

### 2.1 Findings report format

```
## 🔍 Project Assessment

**Project:** <name from package.json/README or directory name>
**Type:** Greenfield | Greyfield (existing code, no agent setup)
**Repo:** Git repo ✓ | No git (recommend: git init)
**GitHub:** Connected ✓ | Local only
**Monorepo:** Yes (N packages) | No

### What already exists
| File                    | Status   | Notes                          |
|-------------------------|----------|--------------------------------|
| AGENTS.md               | ✅ Found  | Has run commands, missing tool allowlist |
| CONTEXT.md              | ❌ Missing| Will create from template      |
| docs/adr/               | ✅ Found  | 3 existing ADRs                |
| docs/.agent/labels.md   | ❌ Missing| Will create standard taxonomy  |
| issues/                 | ❌ Missing| Decision needed — see below    |

### What will be created
<list of new files>

### What will be updated
<list of existing files that are PARTIAL and need additions>
```

### 2.2 Three decisions that need the user

Present **only these** as choices — everything else is decided by the skill:

**Decision A — Issue Tracking**
```
Issue tracking: where should issues live?

  [A] GitHub Issues  — best if team uses GitHub PRs, gives labels UI,
                       milestone tracking, cross-referencing
  [B] Local (issues/ dir) — git-native markdown files, works offline,
                            agent-readable without API, preferred for
                            solo or air-gapped projects
  [C] Both — local issues/ as source of truth, synced to GH Issues
              via a sync agent (we create the workflow stub)
```

**Decision B — Label Vocabulary**
Show the standard taxonomy from `references/labels-standard.md` and ask:
```
Triage labels: use the standard set, or customise?

  [A] Use standard (recommended for new projects)
  [B] Customise — I'll show you the draft and you can edit inline
  [C] Skip for now — add later
```

**Decision C — Monorepo CONTEXT strategy** (only if monorepo detected)
```
This looks like a monorepo with N packages. Domain docs:

  [A] CONTEXT.index.md at root + CONTEXT.md per package (recommended)
  [B] Single root CONTEXT.md with sections per package
```

### 2.3 Show drafts BEFORE writing

After user answers A/B/C decisions, generate and show draft content for the two
highest-visibility files:

- **AGENTS.md** draft (if missing or partial)
- **CONTEXT.md** draft (always — pre-filled from project detection)

Use a code block with the file path as the title. Say:
> "Here's what I'll write to `AGENTS.md`. Edit anything before I proceed — just paste
> back the changed version or tell me what to change."

Wait for approval or edits. Then proceed to Phase 4.

---

## Phase 3 — CONFIRM

Collect any edits from the user on the shown drafts. Apply them mentally.

Show a compact "write plan" — what will be created vs. updated, not the content again:

```
Write plan:
  CREATE  AGENTS.md
  CREATE  CONTEXT.md
  CREATE  DECISIONS.md
  CREATE  docs/adr/ADR-000-template.md
  CREATE  docs/prd/TEMPLATE.md
  CREATE  docs/.agent/labels.md        (internal)
  CREATE  docs/.agent/agents.index.md  (internal)
  CREATE  docs/.agent/prompts/         (internal, 6 agent stubs)
  UPDATE  .gitignore → no changes (already ignores .env)
  SKIP    issues/    → using GitHub Issues per your choice

Proceed? [yes / make a change]
```

Wait for "yes" before Phase 4.

---

## Phase 4 — WRITE

Write all files. Internal files (`docs/.agent/**`) are created quietly with no per-file
announcement. Show the user only the two or three key files they care about.

### Writing order

1. `docs/.agent/` tree (internal — silent)
2. `docs/adr/ADR-000-template.md` (silent)
3. `docs/prd/TEMPLATE.md` (silent)
4. `DECISIONS.md` (silent)
5. `AGENTS.md` (show after writing)
6. `CONTEXT.md` (show after writing)

### File templates to use

Load templates from `references/` dir. See:
- `references/templates.md` — all file templates

### Monorepo variant

If monorepo detected:
- Write `CONTEXT.index.md` at root (map file)
- Write stub `CONTEXT.md` inside each detected package dir
- Each stub references back to root index

### Git integration

If git repo detected, after writing files:
```bash
cd <project_root>
git add docs/.agent/ docs/adr/ docs/prd/ AGENTS.md CONTEXT.md DECISIONS.md
git commit -m "chore: bootstrap agentic project structure

Adds AGENTS.md, CONTEXT.md, docs/.agent/ system dir, ADR and PRD
scaffolding, triage label taxonomy, and agent prompt stubs.

Bootstrap performed by project-bootstrap skill."
```

Ask user before committing: "Should I commit these as an initial bootstrap commit?"

---

## Phase 5 — HANDOFF

Print a clean summary. No bullet walls. Prose + one reference table.

```
## ✅ Bootstrap complete

Your project is now agent-ready. Here's the living structure:

  AGENTS.md        — edit this when your tooling or run commands change
  CONTEXT.md       — update after any architectural decision
  DECISIONS.md     — add an entry for every non-obvious choice made
  docs/.agent/     — internal system dir; agents read from here automatically

The label taxonomy in docs/.agent/labels.md has N labels across 5 dimensions.
If you chose GitHub Issues, go to github.com/<repo>/labels and import them,
or run the label-sync workflow stub in docs/.agent/workflows/gh-labels-sync.md.

Agent prompt stubs are in docs/.agent/prompts/. Each file is a system prompt
skeleton — fill in project-specific rules as you discover them.

Next natural steps (pick any, in any order):
  • Fill the "Domain Glossary" section of CONTEXT.md with your key business terms
  • Write your first ADR in docs/adr/ using the template
  • Run the `diagnoser` agent on your first real bug to see it in action
  • Try the `prd-writer` agent with: "Write a PRD for <next feature>"

Other skills that read from this structure:
  → issue-planner     breaks a PRD into trackable issues
  → tdd-agent         picks up agent-ready issues and writes tests first
  → arch-auditor      reads CONTEXT.md + codebase, flags drift
  → prd-writer        generates PRDs grounded in CONTEXT.md
```

---

## Error cases

**No git repo found:** Offer to run `git init` before proceeding. Note that local issue
tracking works better with git history.

**Existing AGENTS.md with custom content:** Never overwrite. Show a diff-style merge
proposal. Let user pick: keep theirs, keep template additions, or merge interactively.

**Read-only directory:** Tell user, ask them to specify a writable path or run with
appropriate permissions.

**Monorepo with >10 packages:** Warn that creating per-package CONTEXT.md stubs for all
packages may take a moment. Proceed unless user says to limit scope.

---

## Reference files

Read these when generating file content:

- `references/templates.md` — canonical templates for every file this skill creates
- `references/labels-standard.md` — the standard triage label vocabulary
- `references/agent-prompts.md` — stub system prompts for each specialist agent role
