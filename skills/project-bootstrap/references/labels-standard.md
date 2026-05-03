# Standard Triage Label Vocabulary

Canonical label set for the project-bootstrap skill.
Labels are grouped into five orthogonal dimensions. Every issue should
have exactly one label from Type and Priority, and any number from the rest.

---

## Dimension 1 — Type (one per issue)

| Label           | Colour  | Meaning                                           |
|-----------------|---------|---------------------------------------------------|
| `type:bug`      | #d73a4a | Something is broken or behaving incorrectly       |
| `type:feat`     | #0075ca | New capability or user-facing enhancement         |
| `type:chore`    | #e4e669 | Maintenance, dependency bumps, CI, config         |
| `type:spike`    | #8b5cf6 | Time-boxed research or proof-of-concept           |
| `type:debt`     | #f97316 | Technical debt that doesn't change user behaviour |
| `type:docs`     | #0e8a16 | Documentation only                                |
| `type:security` | #b91c1c | Security-related fix or hardening                 |

## Dimension 2 — Priority (one per issue)

| Label | Colour  | Meaning                                                  |
|-------|---------|----------------------------------------------------------|
| `P0`  | #b91c1c | Critical — production down or data loss risk             |
| `P1`  | #f97316 | High — major feature broken, no workaround               |
| `P2`  | #f59e0b | Medium — degraded experience, workaround exists          |
| `P3`  | #6b7280 | Low — cosmetic, nice-to-have, backlog                    |

## Dimension 3 — Domain (zero or more, project-specific)

These are stubs. Replace with your actual domain names after bootstrapping.

| Label           | Colour  | Meaning                        |
|-----------------|---------|--------------------------------|
| `domain:auth`   | #93c5fd | Authentication & authorisation |
| `domain:api`    | #93c5fd | API layer                      |
| `domain:ui`     | #93c5fd | Frontend / UI                  |
| `domain:data`   | #93c5fd | Data layer / storage           |
| `domain:infra`  | #93c5fd | Infrastructure / DevOps        |
| `domain:dx`     | #93c5fd | Developer experience           |
| `domain:perf`   | #93c5fd | Performance                    |

## Dimension 4 — Status (zero or one)

| Label                  | Colour  | Meaning                                  |
|------------------------|---------|------------------------------------------|
| `status:needs-triage`  | #e2e8f0 | Newly filed, not yet assessed            |
| `status:in-progress`   | #bfdbfe | Actively being worked on                 |
| `status:blocked`       | #fecaca | Waiting on external dep or decision      |
| `status:needs-review`  | #fef08a | PR open or work ready for review         |
| `status:wontfix`       | #f1f5f9 | Acknowledged, not addressing             |

## Dimension 5 — Agent Readiness (zero or one)

| Label               | Colour  | Meaning                                       |
|---------------------|---------|-----------------------------------------------|
| `agent:ready`       | #4ade80 | Issue has enough context for autonomous agent |
| `agent:partial`     | #fbbf24 | Agent can start but needs human checkpoint    |
| `agent:human-only`  | #f87171 | Requires human judgement or credentials       |

---

## Usage rules

- Every issue filed must have `status:needs-triage` until a human or triage agent
  reviews it and assigns Type + Priority.
- `agent:ready` is a contract — the issue must contain: clear acceptance criteria,
  at least one affected file path hint, no open questions, priority P1 or lower,
  not `type:security`, and scope bounded to ≤5 files. If any of these are missing,
  use `agent:partial` with a `missing-for-agent:` array instead.
- `P0` issues must never be labelled `agent:ready` — they require human oversight.
- Domain labels are project-specific. Edit the `domain:*` entries to match your
  actual domains. Delete any that don't apply.

---

## GitHub import

To import these labels into a GitHub repository:

```bash
# Install gh CLI first: https://cli.github.com
# Then from your repo root:
gh label create "type:bug"      --color "d73a4a" --description "Something is broken"
gh label create "type:feat"     --color "0075ca" --description "New capability"
gh label create "type:chore"    --color "e4e669" --description "Maintenance"
gh label create "type:spike"    --color "8b5cf6" --description "Research / PoC"
gh label create "type:debt"     --color "f97316" --description "Technical debt"
gh label create "type:docs"     --color "0e8a16" --description "Documentation"
gh label create "type:security" --color "b91c1c" --description "Security"
gh label create "P0"            --color "b91c1c" --description "Critical"
gh label create "P1"            --color "f97316" --description "High priority"
gh label create "P2"            --color "f59e0b" --description "Medium priority"
gh label create "P3"            --color "6b7280" --description "Low priority"
gh label create "status:needs-triage" --color "e2e8f0" --description "Needs triage"
gh label create "status:in-progress"  --color "bfdbfe" --description "In progress"
gh label create "status:blocked"      --color "fecaca" --description "Blocked"
gh label create "status:needs-review" --color "fef08a" --description "Needs review"
gh label create "status:wontfix"      --color "f1f5f9" --description "Won't fix"
gh label create "agent:ready"         --color "4ade80" --description "Agent can tackle autonomously"
gh label create "agent:partial"       --color "fbbf24" --description "Agent needs checkpoint"
gh label create "agent:human-only"    --color "f87171" --description "Human required"
```

A workflow stub that does this automatically lives in:
`docs/.agent/workflows/gh-labels-sync.md`
