---
name: architecture-document
description: Generate the 3-file architecture documentation structure (full-analysis, summary, decision) from a reverse-engineering conversation and/or a markdown file source, with wiring into architecture-map and architecture-decisions.
argument-hint: [name] [type=component|module|subsystem|bounded-context] [doc=<path>] [lang=fr|en]
---

# Architecture Document

## Mission

Produce the structured documentation set for a software unit from one or more sources: a passive
archive (`full-analysis.md`), an operational reference (`summary.md`), and a decision record
(`decision.md`). Sources can be the current conversation, a markdown file, or both combined.
Wire the result into the project memory routing hub.

This skill never writes anything without explicit user approval.

---

## Arguments

- `name` (optional): kebab-case name of the unit (e.g. `generator-pipeline`, `cache`).
  If omitted, infer from the dominant subject of the active sources.
- `type=` (optional): SE type of the unit being documented. Allowed values:
  - `component` — cohesive functional unit (e.g. `cache`, `generator-pipeline`)
  - `module` — set of classes with a shared responsibility (e.g. `adapter/out/persistence`)
  - `subsystem` — group of components fulfilling a function (e.g. `domain-model-enrichment`)
  - `bounded-context` — domain model boundary in DDD sense (e.g. `command`)
  - If omitted, infer using the rule in Step 1.
- `doc=<path>` (optional): path to a markdown file to use as source, in addition to or
  instead of the conversation (e.g. `doc=docs/DEADLOCK_ANALYSIS_AND_FIXES.md`).
- `lang=en` (default) or `lang=fr`: language for generated content.

---

## Context to read before generating

Execute in order. Do not skip.

### Source resolution (Step 0 — before anything else)
Determine active sources from arguments and conversation. See Step 0 below.

### Bootstrap behavior

**Before reading any context file**, check whether `docs/ai/architecture-map.md` exists.

If it does **not** exist:
1. Create `docs/ai/architecture-map.md` using the canonical structure defined in the
   `## Output formats` section below
2. Check whether root `CLAUDE.md` contains both the `@import` and the usage guidance block
   for `architecture-map.md`. If absent, add the following block:

```markdown
## Documented complex mechanisms

@docs/ai/architecture-map.md

If a task touches a mechanism listed in this map, read the corresponding `summary.md`
BEFORE proposing anything. The path is indicated in the mechanism entry.

For an in-depth analysis before a complex change, use the subagent:
\```
Use architecture-reviewer. mechanism: <name>. task: <description>
\```
Specification: `.claude/skills/architecture-reviewer/SKILL.md`

⚠️ `architecture-reviewer` = ALWAYS via Agent tool (isolated subagent).
Never read `full-analysis.md` directly in the main thread.
```

3. Include both the file creation and the `CLAUDE.md` patch in the Step 5 approval output,
   labeled **Bootstrap** and listed before the mechanism patches

If it already exists: proceed normally — patch it with the new mechanism entry only.

### Project context (read if present)
- `.claude/memory-bank/systemPatterns.md` — align generated patterns with existing ones;
  avoid duplicating patterns already documented
- `docs/ai/architecture-map.md` — check if this mechanism is already partially mapped;
  detect naming conflicts
- `.claude/memory-bank/activeContext.md` — understand current workstream context
- `docs/ai/architecture-decisions.md` — check existing ADR numbering for the new entry

---

## Process

Follow these steps in order.

### Step 0 — Source resolution

Determine active sources before doing anything else.

**Case A — conversation only** (no `doc=` argument):
- Source: current conversation
- Continue to Step 1

**Case B — `doc=` only** (no substantive conversation content):
- Source: the specified markdown file
- Apply section-by-section reading strategy (see below)
- Continue to Step 1

**Case C — `doc=` + conversation**:
- Both sources are active
- Markdown file = primary source for background, analysis, history
- Conversation = primary source for recent decisions, validations, refinements
- Conflict resolution: conversation takes precedence (more recent)
- Merge both in Step 3

**If neither source has relevant content**: ask the user before proceeding.

---

#### Section-by-section reading strategy (applied when `doc=` is used)

Do not read the entire document at once if it is large.

1. **Read the first 50 lines** to get document title, structure, and section headings
2. **If document < 300 lines**: read entirely in one pass — no section splitting needed
3. **If document ≥ 300 lines**:
   - Extract all `##` headings and their approximate line positions
   - Present the section list to the user:
     ```
     This document contains X sections (~N lines). Reading section by section:
     [list of sections with line numbers]
     Confirm or indicate sections to exclude.
     ```
   - Wait for confirmation before reading sections
   - Read each section individually using offset + limit
   - After each section, classify its content into destination buckets:

| Section type (keywords) | Destination bucket |
|------------------------|-------------------|
| Current architecture, Structure, Components, How it works | `full-analysis` + `summary` candidates |
| Issues, Risks, Edge cases, Limitations, Bugs | `full-analysis` + `known-issues` candidates |
| Options, Alternatives, Comparison, POC, Benchmark | `decision` candidates |
| Retained solution, Decision, Rules, Constraints | `decision` + `rules` candidates |
| History, Context, Background | `full-analysis` archive |
| Code, Implementation, Examples | `full-analysis` + `systemPatterns` candidates |
| Vocabulary, Business terms, Glossary | `domain-glossary` candidates |
| Integration, API, Flows between services | `integration-patterns` candidates |
| Data sources, Transformations, Targets | `data-sources` candidates |

   - After all sections are read: merge buckets into Step 3 extraction

---

### Step 1 — Identify name, type and scope

**Name**: if provided as argument, use as-is. Otherwise:
- Scan the conversation for the dominant subject (class names, package names, repeated terms)
- Propose a kebab-case name
- **Ask the user to confirm**: "Inferred name: `<name>`. Confirm or provide a different name."
- Do not proceed until confirmed.

**Type**: if `type=` is provided as argument, use as-is. Otherwise infer:

| Observed scope | Inferred type |
|---------------|---------------|
| 1 Java package, cohesive set of classes | `module` |
| Multiple classes around a single functional flow | `component` |
| Multiple packages coordinated toward a function | `subsystem` |
| Domain model boundary (distinct ubiquitous language) | `bounded-context` |

Confirm inferred type with the user if uncertain.

Determine the output folder: `docs/ai/architecture/<name>/`

### Step 2 — Read context files

Read the files listed in "Context to read" above. Note:
- patterns already in `systemPatterns.md` → do not duplicate in the generated docs
- existing mechanisms in `architecture-map.md` → do not conflict
- next ADR number from `architecture-decisions.md`

### Step 3 — Extract and merge content from active sources

Extract from each active source determined in Step 0. When both sources are active,
merge after extraction — conversation takes precedence for decisions and rules.

**From conversation** (if active): scan the full conversation thread.
**From markdown file** (if active): use the section buckets built in Step 0.

Extract:

**For full-analysis.md:**
- all reverse-engineering findings
- technical observations and measurements
- options studied and compared
- edge cases, failure modes, production risks
- code snippets and examples discussed
- questions raised and answers given

**For summary.md:**
- role and responsibility of the mechanism (1 paragraph)
- current architecture (diagram or structured list, max 20 lines)
- sensitive points: what breaks if touched carelessly
- implementation rules: what must always be respected
- evolution rules: how to extend safely
- links to full-analysis.md and decision.md

**For decision.md:**
- the retained decision (what was chosen)
- rejected alternatives and why
- constraints that shaped the choice
- impact on the codebase
- rules that derive from this decision

**For architecture-map.md entry:**
- affected packages/classes (inferred from conversation)
- risk level: HIGH | MEDIUM | LOW (based on sensitivity findings)
- entry point: path to summary.md
- one-line implementation rule

**For architecture-decisions.md entry:**
- ADR number (next in sequence)
- title
- status: Accepted
- one-sentence decision
- pointer to decision.md for detail

### Step 4 — Identify PROMOTE-CANDIDATE rules

**Priority: if `doc=` is active and the source file contains a `## PROMOTE-CANDIDATE`
section, read it first and use it as the base list.** These items are already validated —
do not re-filter them. **Preserve their existing confidence level as-is** — they were
assigned by `codebase-analysis` based on code evidence and must not be downgraded.

**Option 1 flow (same conversation as `codebase-analysis`, no `doc=` export):** if
PROMOTE-CANDIDATE items from `codebase-analysis` are visible in the conversation thread
(identifiable by their `[high | medium | low — <reason>]` format and codebase-analysis
context), treat them with the same rule: preserve their confidence level as-is.
They are code-grounded regardless of the delivery channel.

Then scan the extracted content for additional items not already in the list:

| What to detect | Destination | Typical confidence |
|----------------|-------------|-------------------|
| Mandatory implementation constraint (always/never/must/forbidden) | `.claude/rules/` | `high` if stated explicitly |
| Subtree-specific invariant (single module only) | Local `CLAUDE.md` (via registry) | `high` if stated explicitly |
| Cross-cutting architectural decision (affects 2+ units) | `docs/ai/architecture-decisions.md` | `medium` |
| Operational constraint (DB, infra, external system, known limit) | `docs/ai/known-issues.md` | `high` if explicit, `medium` otherwise |
| Recurring code pattern (2+ locations mentioned in doc/conversation) | `.claude/memory-bank/systemPatterns.md` | `medium` — no grep available |
| Domain term recurring without direct code equivalent | `docs/ai/domain-glossary.md` | `medium` |
| Integration pattern (API, inter-service exchange, shared protocol) | `docs/ai/integration-patterns.md` | `medium` |
| Data flow (identified source, transformation, target) | `docs/ai/data-sources.md` | `medium` |

Note: mechanism-specific business policies go directly into `decision.md` (already produced
in Step 3) — do not duplicate them as PROMOTE-CANDIDATE.

**Confidence assignment for new items (items not from `doc=`):**

| Level | Criteria |
|-------|----------|
| `high` | Explicit decision or constraint stated unambiguously in conversation or doc, OR pattern grep-confirmed in 2+ places |
| `medium` | Inferred from doc content or conversation context — consistent but judgment-based |
| `low` | Prospective item (evolution study, not yet implemented), OR inferred indirectly from structure or naming |

Always append confidence to every item:
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

Merge: deduplicate between the file-sourced items and the newly discovered ones.
List the final set explicitly. Do NOT write them to their destinations — that is the user's
decision, handled separately via `/promote-to-memory` after validation.

### Step 5 — Draft and present for approval

**Before presenting**: check existence of each destination file referenced by PROMOTE-CANDIDATE
items. If any are absent, flag them:

```
⚠️  Missing destination files (promotion blocked for these candidates):
  - docs/ai/domain-glossary.md      → missing — run /memory-bootstrap or create manually
  - docs/ai/integration-patterns.md → missing — same action required
```

If all destination files exist: no warning shown.

Present the full draft of all outputs:

```
FILES TO CREATE:
────────────────
docs/ai/architecture/<mechanism-name>/full-analysis.md   (~XXX lines)
docs/ai/architecture/<mechanism-name>/summary.md         (~XX lines)
docs/ai/architecture/<mechanism-name>/decision.md        (~XX lines)

PATCHES:
────────
docs/ai/architecture-map.md      → new entry "<mechanism-name>"
docs/ai/architecture-decisions.md → ADR-XXX

PROMOTE-CANDIDATE RULES (to promote separately):
─────────────────────────────────────────────────
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
- ...

Generate these files? (yes / no / adjust)
```

Do not write anything until the user confirms.

### Step 6 — Write files after approval

Write only the files and patches confirmed by the user.

**PROMOTE-CANDIDATE must be written into `full-analysis.md`** as a `## PROMOTE-CANDIDATE`
section at the end of the file — even if the list is empty. This ensures they persist
beyond the current session and are available to `/promote-to-memory doc=` in future sessions.

After writing, output:
```
✓ Files created:
  docs/ai/architecture/<mechanism-name>/full-analysis.md
  docs/ai/architecture/<mechanism-name>/summary.md
  docs/ai/architecture/<mechanism-name>/decision.md
  docs/ai/architecture-map.md (updated)
  docs/ai/architecture-decisions.md (updated)

Recommended next steps:
1. Review and adjust summary.md (50-100 lines max, action-guide oriented)
2. Run /promote-to-memory to promote PROMOTE-CANDIDATE rules
   or /promote-to-memory doc=docs/ai/architecture/<mechanism-name>/full-analysis.md
   (if in a new session)
3. Consider a local CLAUDE.md in the relevant module (see user-manual)
```

---

## File specifications

### `full-analysis.md`

No length limit. Contains everything from the reverse-engineering conversation.
This file is never @imported. It is read on demand by the `architecture-reviewer` subagent
or by a developer who needs deep context.

Structure:
```markdown
# Full Analysis — <Mechanism Name>

> Passive archive. Do not @import. Consult via architecture-reviewer or on demand.
> Operational summary: docs/ai/architecture/<name>/summary.md

## Context and scope
## Detailed current architecture
## Reverse engineering: observations and measurements
## Options studied
## Edge cases and failure modes
## Technical notes
## Analysis history

## PROMOTE-CANDIDATE
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
(empty if no candidates identified)
```

The `## PROMOTE-CANDIDATE` section must always be present — it is the handoff point
to `/promote-to-memory doc=<this file>` in future sessions.

---

### `summary.md`

**Hard limit: 50-100 lines.** This is the operational reference — answers one question:
*"What do I need to know before modifying this mechanism?"*

Do not use it to document the mechanism comprehensively. Use it to make changes safe.

Structure:
```markdown
# Summary — <Mechanism Name>

> Operational reference. Limit: 100 lines.
> Full analysis: docs/ai/architecture/<name>/full-analysis.md

## Role
[1-2 sentences]

## Current architecture
[diagram or list, max 15 lines]

## Sensitive points
[what can break if modified carelessly, max 10 lines]

## Implementation rules
[what must always be respected, list format]

## Evolution rules
[how to extend this mechanism safely]

## References
- Full analysis: [full-analysis.md](./full-analysis.md)
- Decision: [decision.md](./decision.md)
- Map: [architecture-map.md](../../architecture-map.md)
```

---

### `decision.md`

**Target: 100-200 lines.**

Structure:
```markdown
# Decision — <Mechanism Name>

## Retained decision
## Rejected alternatives
| Alternative | Reason for rejection |
|-------------|----------------------|

## Constraints that shaped the choice
## Impact on the codebase
## Rules derived from this decision
## References
```

---

### `architecture-map.md` entry

```markdown
## <Name> [<type>]

**Type**: component | module | subsystem | bounded-context
**Scope**: [affected packages or classes]
**Risk**: HIGH | MEDIUM | LOW
**Summary**: [one sentence describing the role]
**⚠️ Required**: READ `docs/ai/architecture/<name>/summary.md` before any modification.
**Rule**: [main implementation constraint in one line]
**Full analysis**: `docs/ai/architecture/<name>/full-analysis.md` (do not @import)
```

---

### `architecture-decisions.md` entry

```markdown
## ADR-XXX: <Title>

**Status**: Accepted
**Decision**: [one sentence]
**Detail**: [docs/ai/architecture/<name>/decision.md](architecture/<name>/decision.md)
```

---

## Style constraints

- Match the language of `$ARGUMENTS lang=` parameter (default: English)
- `summary.md` and `decision.md`: concise, structured, no prose essays
- `full-analysis.md`: no style constraint — preserve the richness of the conversation
- Do not invent facts not grounded in the conversation or read files
- Mark uncertain content with `[TO VERIFY]` rather than presenting it as confirmed
