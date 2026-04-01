# Codebase Analysis — Usage Guide

For the full specification, see [SKILL.md](../SKILL.md).

## What this skill does

`codebase-analysis` analyses an existing codebase to extract its domain model,
business rules, integrations, and technical risks. It adapts to the size of the
repo and never persists anything without explicit approval.

It produces a structured analysis and proposes at end of session:
- capturing it in project memory (via `/architecture-document`)
- exporting it to a folder
- keeping it in the conversation only

---

## When to use it

| Situation | Use it? |
|-----------|---------|
| Discovering an unknown repo | Yes — start with cartography |
| Understanding a module before modifying it | Yes |
| Preparing an `/architecture-document` session | Yes |
| Auditing the risks of a component | Yes |
| Module already documented in `architecture-map.md` | No — use `architecture-reviewer` |
| Simple question about a class | No — read directly |

---

## Invocation

```bash
# Interactive mode — analysis + review loop + output gate
/codebase-analysis

# With intent to document in memory
/codebase-analysis output=memory

# With export to a folder
/codebase-analysis output=docs/notes/

# Resume from an existing analysis file (iterative refinement)
/codebase-analysis doc=docs/notes/pipeline-analysis.md

# Resume an in-progress analysis (large repo, new session)
"Resume the analysis of back.
 The plan is in docs/ai/analysis-plan-back.md.
 Next unit: pipeline."
```

---

## Unit types

The skill classifies each discovered unit by its SE type:

| Type | Description | Example |
|------|-------------|---------|
| `component` | Cohesive functional unit | `pipeline`, `cache` |
| `module` | Package with shared responsibility | `adapter/out/persistence` |
| `subsystem` | Group of coordinated components | `domain-model` |
| `bounded-context` | Domain model boundary | `command`, `crm` |

The type is optional in Phase 0 — use `?` if uncertain. It can be specified
or corrected before running `/architecture-document`.

---

## Complete workflow

### Small repo (< 5k lines)

```
1. /codebase-analysis
   → Phase 0: global cartography → confirmation
   → Phases 1-5: analysis in one pass

2. Output gate
   → Option 1: /architecture-document <name> type=<type>
   → Option 2: folder export
   → Option 3: conversation only
```

### Medium repo (5k–30k lines)

```
1. /codebase-analysis
   → Phase 0: cartography → plan saved (recommended)
   → Phases 1-5: by layer (domain first)

2. Output gate per identified unit
   → /architecture-document <name> type=<type>  (once per unit)
```

### Large repo (> 30k lines) — one conversation per unit

```
Conversation 1:
  /codebase-analysis
  → Phase 0: global cartography → plan saved in analysis-plan-<repo>.md
  → Analyse unit A (HIGH priority)
  → Output gate → option 1 → /architecture-document unit-a type=module

Conversation 2:
  "Resume the analysis of <repo>. Plan: docs/ai/analysis-plan-<repo>.md.
   Next unit: unit-b."
  → Phases 1-5 on unit-b only
  → Output gate → /architecture-document unit-b type=component

...
```

**⚠️ Only option 1 (→ `/architecture-document`) feeds `architecture-map.md`
and makes results available for subsequent sessions.**

---

## The plan file (large repo)

`docs/ai/analysis-plan-<repo-name>.md` is proposed for creation if the repo is large.
It is the **source of truth between sessions** — without it, a new session does not know
what has already been processed.

### Available statuses

| Status | Meaning | Persistent |
|--------|---------|------------|
| ⬜ | To do — will be proposed at next session | Yes |
| ⏳ | In progress — active session | Yes |
| ✅ | Documented — `architecture-map.md` updated | Yes |
| 📁 | Exported to a folder — resumable via `doc=` | Yes |
| 💬 | Conversation only — not persisted | Yes |
| 🚫 | Ignored — will no longer be proposed | Yes |

For `📁` units, the skill automatically displays the resume command:

```
📁  domain-model  [bounded-context]  HIGH  → exported (2026-03-23)
    ↳ Resume: /codebase-analysis doc=docs/ai/analysis/domain-model/domain-model-analysis.md
```

### Updating the plan

- After each output gate: **automatic and mandatory** update (without asking)
- If the plan does not yet exist: one-time proposal to create it

### Changing a unit's status

At any time, the user can change a unit's status:

```
"Mark <name> as ignored"     → 🚫 (will no longer be proposed)
"Mark <name> as to do"       → ⬜ (will be re-proposed)
"Analyse <name> anyway"      → override of an ignored 🚫
```

`🚫 ignored` units appear in the plan but are never proposed
for analysis unless explicitly instructed.

### Adding a unit manually

If cartography missed a topic, the user can add it:

```
"Add <name> [<type>] [<risk>]"
```

The skill asks 4 questions before registering the unit in the plan:

```
1. SCOPE    — packages or classes to analyse (or "nothing specific")
2. FREEDOM  — exclusive or indicative scope
3. OBJECTIVE — why this analysis?
4. CONTEXT  — what you already know
```

**Proposed objectives:**

| | Objective | Effect on phases |
|-|-----------|-----------------|
| a | Modify / extend | Priority: domain model + risks |
| b | Debug / investigate | Priority: business logic + risks |
| c | Understand architecture | Priority: structure + domain model |
| d | Prepare a refactoring | Priority: structure + coupling (phases 3, 5 light) |
| e | Document for the team | All phases |
| — | Free objective | Describe in free text |

### Pre-analysis questions (all units)

Before starting phases 1-5 on **any** unit, the skill checks
whether parameters are already stored in `## Analysis details` of the plan:

- **If yes**: displays the parameters and asks "Confirm or modify?"
- **If no**: asks questions 3 (objective) and 4 (known context).
  For manually added units, all 4 questions apply.

Confirmed parameters are **immediately written** to the plan.
If no plan exists, parameters remain in conversation only — the skill
offers to create the plan to persist them.

### Modifying a unit's parameters

The user can change scope, freedom, objective, or context **at any time**:
during cartography, before phases start, during analysis,
or at the start of a new session.

```
"Change the objective of <name> to <objective>"
"Change the scope of <name> to <list>"
"Change the freedom of <name> to exclusive/indicative"
"Modify the context of <name>: <text>"
"Add to the context of <name>: <text>"
```

When context is clear (unit currently being analysed), without naming the unit:

```
"Change the objective to <objective>"
"Scope: only <list>"
"Context: <text>"
```

The skill applies the change immediately and updates `## Analysis details`
in the plan without asking. If analysis is in progress, remaining phases are
recalibrated. Already completed phases are not replayed unless explicitly requested.

**Persistence between sessions**: parameters written to the plan are displayed
at the start of the next session (resume) and before launching the phases.

---

## Review loop and iterative refinement

After phases 1-5, the skill **does not go directly to the output gate**.
It first presents the analysis in the conversation and enters a review loop:

```
ANALYSIS — <name> [<type>]
──────────────────────────
[complete findings in the conversation]

Questions, areas to explore further, corrections?
  → Ask a question                           : Claude explores and responds
  → "Explore <aspect> further"               : Claude re-runs the relevant phases
  → "Correct <point>"                        : Claude updates its understanding
  → "Show me the end-to-end flow
     for <feature>"                          : behavioural synthesis (see below)
  → "That's good" / "output"                 : proceed to output gate
```

**Behavioural synthesis on demand**: on "Show me the end-to-end flow for X",
Claude synthesises from phases 1-5 already in context and produces:
- Entry point (type + trigger)
- Feature steps with conditions and branches
- Triggered outputs (features/flows activated on exit + conditions)
- Mermaid diagram if ≥ 3 steps or conditions

No re-reading of code unless a step is missing from the existing analysis.

The loop continues until the user signals the analysis is ready.
**The output gate is only proposed after explicit approval.**

Accepted signals for "ready": `that's good`, `ok`, `output`, `ready`, `go`,
`produce the output`, `export`, `save`, `/architecture-document`.

---

## Iterative refinement from an existing file (`doc=`)

If an analysis has already been exported to a file (option 2), it can
be resumed and refined in a new session:

```bash
/codebase-analysis doc=docs/notes/pipeline-analysis.md
```

The skill reads the file and presents a coverage summary:

```
EXISTING ANALYSIS DETECTED — pipeline-analysis.md
─────────────────────────────────────────────────────────────
Unit: service/domain_model [component]
Current coverage:
  ✅ Domain model    : 5 entities, relations mapped
  ✅ Business rules  : 4 rules identified
  ⚠️  Integrations   : incomplete (outgoing flows not covered)
  ❌ Edge cases      : not covered

What would you like to explore further or correct?
(or "continue" to go directly to the review loop)
```

The user can then target missing areas without redoing the entire analysis.
At end of session, option 2 proposes to **update the same file**
rather than creating a new one.

### Complete iterative pipeline

```
Session 1 — exploration:
  /codebase-analysis
  → Phases 1-5 → review → output gate option 2
  → docs/notes/my-unit-analysis.md created (single file)

Session 2 — deepening:
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
  → Coverage summary → targeted deepening → review → output gate option 2
  → docs/notes/my-unit-analysis.md updated

Session 3 — capitalisation:
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
  → "That's good" → output gate option 1
  → /architecture-document my-unit type=component
    (generates full-analysis.md + summary.md + decision.md)
```

---

## Output gate — the 3 options

```
What to do with these results?
  1. Document in project memory → /architecture-document <name> type=<type>
  2. Export artefacts to a folder → <path>
  3. Keep in conversation only → nothing is written
```

| Option | What is written | PROMOTE-CANDIDATE | Persisted between sessions? |
|--------|----------------|-------------------|-----------------------------|
| 1 | `docs/ai/architecture/<name>/` + map patches + ADR | In `full-analysis.md` (via `/architecture-document`) | ✅ Yes |
| 2 | `<path>/<name>-analysis.md` (single file) | `## PROMOTE-CANDIDATE` section in the file | ⚠️ Yes but outside project memory |
| 3 | Nothing | Lost when session closes | ❌ No |

To promote from the exported file (option 2) in a later session:
```bash
/promote-to-memory doc=<path>/<name>-analysis.md
```

---

## Resuming between sessions

Resume phrase to use at the start of a new conversation:

```
"Resume the analysis of <repo-name>.
 The plan is in docs/ai/analysis-plan-<repo-name>.md.
 Next unit to analyse: <name> [<type>]."
```

Claude reads the plan, sees what is already documented in `architecture-map.md`,
and starts directly on the next unit without redoing the cartography.

---

## PROMOTE-CANDIDATE — destinations and verification

`codebase-analysis` identifies candidates for promotion to all memory layers,
not just rules and patterns. The complete routing table:

| Detected type | Phase | Destination |
|---|---|---|
| Repo-wide invariant | 3 | `.claude/rules/<file>.md` |
| Subtree-specific invariant | 3 | Local `CLAUDE.md` of the module (via registry) |
| Business policy (analysed mechanism) | 3 | `decision.md` (via `/architecture-document`) |
| Cross-cutting decision or already-documented mechanism | 3, 4 | `docs/ai/architecture-decisions.md` |
| Operational constraint | 3 | `docs/ai/known-issues.md` |
| Recurring pattern (2+ places) | 2, 3 | `.claude/memory-bank/systemPatterns.md` |
| Business term | 2 | `docs/ai/domain-glossary.md` |
| Integration pattern (2+ units) | 4 | `docs/ai/integration-patterns.md` |
| Data flow | 4 | `docs/ai/data-sources.md` |

**Confidence level**: each PROMOTE-CANDIDATE carries a mandatory confidence level,
suffixed in brackets:

| Level | Criterion |
|-------|-----------|
| `high` | Extracted from a `throw`/exception, or grep-confirmed in 2+ distinct files, or coverage gap found by grep |
| `medium` | Consistent observation but Claude's judgement (business terms, heuristic patterns, integration observed on a single unit) |
| `low` | `[IMPLICIT]` — suspicion from a comment, naming convention, or indirect evidence |

```
- "rule"   → .claude/rules/db.md             [high — extracted from throw CTR_MISSING_IDR]
- "term"   → docs/ai/domain-glossary.md      [medium — appears consistently across entities]
- "order"  → docs/ai/known-issues.md         [low — implicit, suspected from comment Scheduler:45]
```

The confidence level is persisted in `## PROMOTE-CANDIDATE` of `full-analysis.md` and
visible by `/promote-to-memory` in a later session — which avoids treating a suspicion
as an established fact during promotion.

**Destination file verification**: before the output gate, the skill checks whether
each referenced destination file exists. If a file is absent, it is flagged
and promotion of the corresponding candidates is blocked. Solution: run
`/memory-bootstrap` or create the file manually, then re-run `/promote-to-memory`.

---

## Common mistakes

| Mistake | Consequence | Solution |
|---------|-------------|----------|
| Running without Phase 0 on a large repo | Context saturates quickly | Always start with cartography |
| Not saving the plan | Cannot resume between sessions | Accept plan save in Phase 0 |
| Choosing option 3 for all units | No persistent trace | Prefer option 1 for HIGH/MEDIUM units |
| Analysing a unit already in `architecture-map.md` | Duplicate work | Use `architecture-reviewer` instead |
| Exceeding 10 classes per session | Context saturated | Split the unit into two sub-units |

---

## See also

- [SKILL.md](../SKILL.md) — full specification
- [patterns.md](../patterns.md) — architectural patterns to identify
- [dependencies.md](./dependencies.md) — dependencies towards `/architecture-document`
- [report-examples.md](./report-examples.md) — output examples
- `docs/ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md` — workflow overview
