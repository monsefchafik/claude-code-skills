# Architecture Document — Usage Guide

For the full specification, see [SKILL.md](../SKILL.md).

## What this skill does

`/architecture-document` transforms one or more analysis sources into a structured documentation
set wired into the project memory.

Accepted sources: **conversation**, **markdown file**, or **both combined**.

It produces **3 files** + **2 patches** on the existing memory:

```
docs/ai/architecture/<name>/
  full-analysis.md   → complete archive (passive, never @imported)
                       includes ## PROMOTE-CANDIDATE at end of file
  summary.md         → action guide 50-100 lines (importable if needed)
  decision.md        → decision context 100-200 lines

docs/ai/architecture-map.md       → routing entry with type (patch)
docs/ai/architecture-decisions.md → short ADR pointing to decision.md (patch)
```

## When to use it

| Situation | Invocation |
|-----------|-----------|
| End of a reverse-engineering session | `/architecture-document <name> type=<type>` |
| After a `codebase-analysis` (option 1) | `/architecture-document <name> type=<type>` |
| Convert an existing document (plan, analysis) | `/architecture-document <name> type=<type> doc=<path>` |
| Existing document + validation session | `/architecture-document <name> type=<type> doc=<path>` |
| Already documented unit, updating | `/architecture-document <name> type=<type>` or without type= |

## Invocation

```bash
# With explicit type (recommended)
/architecture-document generator-pipeline type=component
/architecture-document adapter/out/persistence type=module
/architecture-document domain-model-enrichment type=subsystem
/architecture-document command type=bounded-context

# Without type — Claude infers from code structure
/architecture-document generator-pipeline
/architecture-document

# With source file
/architecture-document deadlock-prevention type=module doc=docs/DEADLOCK_ANALYSIS_AND_FIXES.md

# Combined sources: file + conversation
/architecture-document generator-pipeline type=component doc=docs/GENERATOR_MIGRATION_PLAN.md

# In French
/architecture-document cache type=component doc=notes/cache-analysis.md lang=fr
```

## The three source modes

### Mode A — Conversation only
The skill extracts all content from the current session.
Ideal after a long reverse-engineering session with Claude.

### Mode B — Markdown file only
The skill reads the file and transforms it into the 3-file structure.
Works for two families of documents:

**Retrospective documents** (analysis of an existing system):
```bash
/architecture-document deadlock-prevention type=module doc=docs/DEADLOCK_ANALYSIS_AND_FIXES.md
```

**Prospective documents** (evolution study, RFC, design doc):
```bash
/architecture-document cache-strategy type=subsystem doc=docs/CACHE_EVOLUTION_STUDY.md
```

For prospective documents, `decision.md` will naturally be the richest file
(comparisons, rationale, strategy). Implementation constraints derived from the
decision land in PROMOTE-CANDIDATE → `.claude/rules/`.

### Mode C — File + conversation (hybrid)
Both sources are active. The file provides the background, the conversation provides
recent decisions and validations.

**Merge rule**: in case of conflict, the conversation wins (more recent).

```bash
/architecture-document generator-pipeline doc=docs/GENERATOR_MIGRATION_PLAN.md
# After a session where you discussed and refined the plan
```

## Section-by-section reading (documents ≥ 300 lines)

When a source file is large, the skill does not load it all at once.
It proceeds section by section:

```
1. Reads the first 50 lines → detects structure and ## headings
2. Presents the section list:
   "This document contains 8 sections (~1400 lines). Reading section by section:
    § 1. Context (l.1-80)
    § 2. Current architecture (l.81-250)
    § 3. Identified issues (l.251-480)
    ..."
3. Waits for your confirmation (you can exclude sections)
4. Reads each section individually
5. Classifies each section into the correct destination file
6. Merges at end of reading
```

**Automatic classification by section:**

| Keywords in heading | Primary destination |
|--------------------|---------------------|
| Current architecture, Structure, Components, How it works | `full-analysis` + `summary` candidates |
| Issues, Risks, Edge cases, Limitations, Bugs | `full-analysis` + `known-issues` candidates |
| Options, Alternatives, Comparison, POC, Benchmark | `decision` candidates |
| Retained solution, Decision, Rules, Constraints | `decision` + `.claude/rules/` candidates |
| Performance, Measurements, Metrics | `full-analysis` + `decision` candidates if comparative |
| Strategy, Prioritization, Roadmap, Steps | `decision` + `summary` candidates |
| History, Context, Background | `full-analysis` archive |
| Code, Implementation, Examples | `full-analysis` + `systemPatterns` candidates |
| Vocabulary, Business terms, Glossary | `domain-glossary` candidates |
| Integration, API, Flows between services | `integration-patterns` candidates |
| Data sources, Transformations, Targets | `data-sources` candidates |

You can exclude sections if some are not relevant for the documentation.

## Complete workflow

```
Phase 1a — Work session (Mode A)
   └─ Long reverse-engineering conversation with Claude
   └─ /architecture-document <name>

Phase 1b — Existing document (Mode B)
   └─ You have an existing analysis file
   └─ /architecture-document <name> doc=<path>
   └─ Skill reads section by section if ≥ 300 lines

Phase 1c — Hybrid (Mode C)
   └─ Prepared document + validation/decision session
   └─ /architecture-document <name> doc=<path>
   └─ Merge: doc = background, conversation = decisions

Phase 2 — Generation (common to all 3 modes)
   └─ Skill reads: sources + systemPatterns + architecture-map
   └─ Skill proposes the 3 files + 2 patches
   └─ ↓ HUMAN CONFIRMATION ↓
   └─ Files written

Phase 3 — Human validation
   └─ Re-read summary.md: ≤ 100 lines, "action guide" oriented?
   └─ Re-read decision.md: rejected alternatives documented?
   └─ Verify map + ADR patches

Phase 4 — Rule promotion
   └─ Same session  : /promote-to-memory
   └─ Next session  : /promote-to-memory doc=docs/ai/architecture/<name>/full-analysis.md
   └─ Promotes PROMOTE-CANDIDATE → .claude/rules/ + systemPatterns.md
   └─ Automatic deduplication against existing target files

Phase 5 — Wiring (based on risk level)
   └─ LOW/MEDIUM: map entry is sufficient
   └─ HIGH: local CLAUDE.md with @summary.md
```

## What the skill writes directly

These files are created or patched in Step 6, without going through PROMOTE-CANDIDATE:

| File | Write type |
|------|-----------|
| `docs/ai/architecture/<name>/full-analysis.md` | Created |
| `docs/ai/architecture/<name>/summary.md` | Created |
| `docs/ai/architecture/<name>/decision.md` | Created — includes mechanism-specific business policies |
| `docs/ai/architecture-map.md` | Patched (new mechanism entry) |
| `docs/ai/architecture-decisions.md` | Patched (short ADR) |
| Root `CLAUDE.md` | Patched **once only** (Bootstrap) — adds `@architecture-map.md` block |

> **Root `CLAUDE.md` is not a PROMOTE-CANDIDATE destination.**
> Bootstrap writes it directly once at initialization. Repo-wide rules go into
> `.claude/rules/`, not into `CLAUDE.md`. Putting rules in `CLAUDE.md` bypasses the
> rules system and bloats the context loaded every session.

## What the skill lists as PROMOTE-CANDIDATE

At the end of generation, the skill:
1. Displays candidates in the conversation (Step 5) with their destination and confidence level
2. Writes them into `full-analysis.md` (section `## PROMOTE-CANDIDATE`) — cross-session persistence

**Mandatory format**:
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

**Confidence levels**:

| Level | Criteria |
|-------|----------|
| `high` | Explicit decision or constraint in conversation/doc, or pattern grep-confirmed in 2+ places |
| `medium` | Inferred from content — consistent but Claude's judgment |
| `low` | Prospective item (evolution study, not yet implemented), or indirectly deduced |

> **⚠️ Gap 3 — No grep on code**: unlike `codebase-analysis`, this skill generates its
> PROMOTE-CANDIDATE solely from the conversation and/or the source markdown file.
> It performs **no grep** on actual code. Consequence: the `high` confidence based on
> "grep-confirmed 2+ places" is only available if the source document itself cites code
> evidence. Without explicit code evidence, prefer `medium` over `high` for items identified
> by this skill alone. If the source document is a `codebase-analysis` export (via `doc=`),
> its confidence levels are already based on code evidence — preserve them as-is.

**Rule for items from `doc=` (codebase-analysis)**: their confidence level is preserved
as-is — it was assigned based on code evidence and must not be modified.

**Example**:
```
PROMOTE-CANDIDATE RULES:
- "Always sort by ID before saveAll()"
  → .claude/rules/database-operations.md  [high — constraint stated explicitly in analysis]
- "Never call the external API within a transaction"
  → .claude/rules/database-operations.md  [high — pattern grep-confirmed in 3 service files]
- "XxxEntity.fromDomainModel() pattern present in 4 files"
  → .claude/memory-bank/systemPatterns.md  [high — grep-confirmed in 4 distinct files]
- "Read/write decoupling planned for V2"
  → docs/ai/architecture-decisions.md  [low — prospective, evolution study only]
```

**Possible destinations** (8 PROMOTE-CANDIDATE destinations):

| Type | Destination |
|------|-------------|
| Repo-wide invariant | `.claude/rules/` |
| Subtree-specific invariant | Local `CLAUDE.md` (via registry) |
| Cross-cutting decision | `docs/ai/architecture-decisions.md` |
| Operational constraint | `docs/ai/known-issues.md` |
| Recurring pattern (2+) | `.claude/memory-bank/systemPatterns.md` |
| Business/domain term | `docs/ai/domain-glossary.md` |
| Integration pattern | `docs/ai/integration-patterns.md` |
| Data flow | `docs/ai/data-sources.md` |

> Mechanism-specific business policies go directly into `decision.md` (produced in Step 3)
> — do not duplicate them as PROMOTE-CANDIDATE.

> **Tier 2 — On-demand**: `domain-glossary`, `integration-patterns`, `data-sources` are not
> @imported. They are listed in root `CLAUDE.md` with their "when to read" — Claude knows
> they exist. `memory-maintainer` enforces this and flags any missing file.

Run `/promote-to-memory` after validation to integrate them.
If the session is closed before: `/promote-to-memory doc=docs/ai/architecture/<name>/full-analysis.md`

## Wiring with project memory

### Level 1 — automatic (done by the skill)
Entry in `architecture-map.md`. Sufficient for LOW/MEDIUM risk.

### Level 2 — root CLAUDE.md (once only)
Verify that `@docs/ai/architecture-map.md` is present in `CLAUDE.md`.

### Level 3 — local CLAUDE.md (HIGH risk only)
```markdown
## Mechanism: <Name>
Before any modification, read:
@docs/ai/architecture/<mechanism>/summary.md
```

### Level 4 — architecture-reviewer (complex tasks)
```
Use architecture-reviewer. mechanism: <name>. task: <description>
```

## Iteration after generation

```bash
# summary.md too long
"summary.md is 150 lines, reduce it to 80 keeping the essentials"

# Missing section from source doc
"Re-read section § 5 of the source document and integrate the edge cases into full-analysis.md"

# Exclude a section from source doc
"Section § 7 (Version history) is not relevant, do not include it"

# Wrong language
"Translate summary.md to French"
```

## Common mistakes to avoid

| Mistake | Consequence | Solution |
|---------|-------------|----------|
| @importing `full-analysis.md` in CLAUDE.md | Overloaded context | Only import `summary.md` or `architecture-map.md` |
| Forgetting `/promote-to-memory` after generation | Action rules remain passive | Always run after validation |
| `summary.md` > 100 lines | Diluted signal, too heavy to import | Prune immediately |
| Reading the entire doc on a large file | Context overload during generation | Let the skill read section by section |
| Generating without validating files | Incorrect content in active memory | Always re-read the 3 files before confirming |
| Assigning `[high]` without code evidence | False confidence on unverified item | Use `[medium]` without explicit grep — see Gap 3 |
| Expecting the skill to update `progress.md` | Milestone not tracked | Update `progress.md` manually after generation — see Gap 4 |

## Known skill limitations

### Gap 3 — Confidence without grep

`/architecture-document` does not access code directly. It cannot confirm a pattern
by grep. Consequence on the confidence level of the PROMOTE-CANDIDATE it generates:

- Items from a `codebase-analysis` `doc=` → confidence preserved as-is (based on code evidence)
- Items from a conversation or non-analytical document → at best `[medium]` unless the document itself cites explicit code evidence ("pattern present in 3 files: X, Y, Z")
- Prospective items (RFC, evolution study) → `[low]` systematically

To get `[high]` PROMOTE-CANDIDATE based on grep, run `codebase-analysis` first then
use its export as `doc=` in `/architecture-document`.

### Gap 4 — progress.md and activeContext.md not updated

`/architecture-document` never writes to `progress.md` or `activeContext.md`.
These files represent the current work context — their update is a human decision
that cannot be automated without knowing the project context.

**Required action after each documentation**:
- If the documentation is part of an ongoing workstream → update `activeContext.md`
- If it closes a milestone → move it from `activeContext.md` to `progress.md`
- These updates are done manually or via `/promote-to-memory` if you want the skill
  to handle them alongside the PROMOTE-CANDIDATE

## See also

- [SKILL.md](../SKILL.md) — full specification
- [architecture-reviewer user-manual](../../architecture-reviewer/references/user-manual.md) — using the generated docs in future tasks
- [ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md](../../../../docs/ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md) — pattern overview and complete workflow
