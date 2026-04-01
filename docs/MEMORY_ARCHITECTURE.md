# Claude Code Memory Architecture — Project Standard

This document describes the versioned memory architecture used with Claude Code and recommended
as a company-wide standard. It explains what each memory layer is for, what belongs where, how
the memory-bank is structured, and when and how to use the `memory-maintainer` skill to keep it
accurate over time.

---

## Why a Memory Architecture?

Claude Code starts every conversation without memory of previous sessions. Without a structured
memory system:

- You repeat context every session ("as I told you last time, we use sorted saveAll…")
- Claude makes assumptions that contradict past decisions
- Patterns discovered in one task are forgotten in the next
- The same mistakes are made repeatedly

A versioned memory system solves this by storing durable knowledge in the repository itself —
loaded automatically at conversation start, reviewed like code, and evolved alongside the project.

**The core principle**: memory files are production code. They should be explicit, reviewable,
and reversible. Low-signal content — placeholders, duplicates, stale status boards — is actively
harmful because it dilutes the signal Claude reads at session start. A short, accurate memory is
worth more than a long, stale one.

---

## The Four Memory Layers

There are four distinct layers, each with a different purpose and a different decay rate.

### Layer 1 — Prescriptive Rules

**Files**: `CLAUDE.md` (root), local `CLAUDE.md` per directory, `.claude/rules/*.md`

**Purpose**: Tell Claude *how to behave* in this project. These are instructions, not information.
They directly constrain what Claude writes and how it responds.

**Examples**:
- "Always sort entities by ID before calling `saveAll()`"
- "Never import adapter classes from the application layer"
- "Use `@RequiredArgsConstructor` for dependency injection, never `@Autowired`"

**Decay rate**: Very slow. Changes only when team conventions change.

**Routing**:
- Root `CLAUDE.md` → repo-wide rules
- Local `CLAUDE.md` → rules specific to a directory subtree (placed next to the code they govern)
- `.claude/rules/` → cross-cutting rules too long for `CLAUDE.md` (e.g., a full database safety
  ruleset, a security checklist)

---

### Layer 2 — Explanatory Knowledge

**Files**: `docs/ai/*.md`

**Purpose**: Give Claude durable *understanding* of the project. This layer does not tell Claude
what to do — it tells Claude *why things are the way they are*, so it can make better decisions
when writing new code or investigating problems.

**Examples**:
- Architecture decision records (ADRs)
- Domain glossary
- Known production issues and their root causes
- Integration patterns for adding a new data source
- Generator and pipeline architecture diagrams

**Decay rate**: Slow. Changes when the architecture or domain understanding evolves.

**Key distinction from Layer 1**: a rule says "always sort before saveAll". The explanatory layer
says "unsorted saveAll causes ORA-00060 on the PRODUCT table because Oracle acquires row locks
in insertion order." Both are useful; they serve different purposes.

---

### Layer 3 — Evolving Project Context

**Files**: `.claude/memory-bank/activeContext.md`, `.claude/memory-bank/progress.md`,
`.claude/memory-bank/systemPatterns.md`

**Purpose**: Track what is happening *right now*, what has been completed, and how this specific
codebase tends to solve recurring problems.

This is the most actively maintained layer. Each file has a distinct lifecycle — see the
[Memory Bank Structure](#the-memory-bank-structure) section below.

**Decay rate**: Variable. `activeContext` expires when work is done. `progress` is permanent.
`systemPatterns` grows slowly as patterns are confirmed.

---

### Layer 4 — Live sources (observational only)

Live sources are inputs to the skill, never destinations to be written directly. The skill
reads them, filters for durable content, and promotes into Layers 1–3 after human review.

There are two live sources:

**Auto-memory** — `~/.claude/projects/<project>/memory/` (user-local, not in the repo)
Claude Code's built-in observation layer. It records things Claude notices across sessions —
commands run, patterns observed, feedback given. Requires `autoMemoryEnabled: true` in
`~/.claude/settings.json`. When empty or absent, the skill reports this and moves on.

**The current conversation**
Explicit decisions, confirmed patterns, discovered issues, and completed milestones stated
during the current session are also valid promotion candidates. The skill promotes from the
conversation when the user explicitly asks ("promote this", "remember this"), or when the
conversation contains unambiguous statements of decisions or rules.

The same durable/not-durable filter applies to both sources: speculation, confusion, and
conversational filler are never promoted.

---

## The Memory Bank Structure

The memory-bank is the most actively used layer. It has exactly three files, each with a clear
contract about what goes in, and what comes out when the work is done.

---

### `activeContext.md` — What is happening now

This file answers: *"What should Claude know before starting work today?"*

It holds current work in progress, active risks, and decisions that are still pending. It is
**not a task board** — no assignees, sprint dates, or status columns. It is a focused context
note.

**Structure**:
```markdown
## Current Focus
[What is actively being worked on]

## Active Risks
[Blockers or risks that need resolution before work can proceed]

## Pending Decisions
[Decisions not yet made but needed to move forward]
```

**Lifecycle rule**: When work is done, move the entry to `progress.md`. Do not leave completed
work in `activeContext`. If this file grows beyond ~15 lines, it needs pruning.

**Why this instead of a sprint board?** Sprint boards require knowing sprint boundaries and
assignment columns. They become stale every two weeks by design, and the ritual of maintaining
them fights against accuracy. A flat context note is simpler, stays accurate longer, and gives
Claude exactly what it needs without overhead.

---

### `progress.md` — What has been completed

This file answers: *"What has already been done, so Claude doesn't redo it or contradict it?"*

It is a **strictly append-only log**. One entry per significant milestone, with a date and a
reference (PR number, commit hash, or ticket ID). Never delete or edit existing entries — the
log is the record.

**Structure**:
```markdown
| Date | Milestone | Reference |
|------|-----------|-----------|
| 2026-03-17 | Deadlock fix: sorted saveAll on PRODUCT and CONSISTENCY_ERROR | a1b2c3d4 |
| 2026-03-21 | POC: shared generateAndValidate across CatalogSync and PricingEngine generators | b3c4d5e6 |
```

**Why append-only?** Editable status boards require continuous maintenance and go stale. A log
only grows. Claude reads it to understand what is already done, which prevents suggesting work
that was just completed or designing solutions that contradict delivered architecture.

---

### `systemPatterns.md` — How we implement things here

This file answers: *"What are the recurring implementation approaches specific to this codebase
that Claude should follow when writing new code?"*

It captures patterns that are too implementation-specific for `.claude/rules/` but too recurring
to leave undocumented. **Rules say what to do; patterns say how we do it here.**

**Entry threshold — the two-occurrence rule**: add a pattern when the same approach appears in
2 or more distinct files in the codebase, OR when a non-obvious design decision has been
explicitly validated by the team. Do not promote one-off implementations — they may be
coincidence or experimentation.

**Format**: pattern name, when to use it, minimal code example. No prose essays.

**Example entry**:
```markdown
## Persistence Patterns

### Sort before saveAll
When to use: every saveAll() call on any entity. Required to prevent ORA-00060 deadlocks.

List<XxxEntity> entities = items.stream()
    .map(XxxEntity::fromDomainModel)
    .sorted(Comparator.comparing(XxxEntity::getId, Comparator.nullsLast(naturalOrder())))
    .toList();
repository.saveAll(entities);
```

**Why a separate file?** Rules in `.claude/rules/` are prescriptive ("always sort before
saveAll"). Patterns in `systemPatterns.md` are contextual and include the *how* and a code
example. They complement each other — the rule tells Claude what is required, the pattern tells
Claude exactly how to implement it in this codebase.

---

## What Does Not Belong in Memory

| Item | Why not | Where it belongs instead |
|------|---------|--------------------------|
| Code patterns derivable by reading files | Memory lags behind code | Just read the code |
| Git history | `git log` is authoritative | `git log` |
| Completed sprint tasks | Expire immediately | `progress.md` (milestone only, not task list) |
| Speculative business rules | Unconfirmed, likely wrong | Nowhere until explicitly confirmed |
| Duplicate of an existing rule | Creates contradiction risk | Update the canonical location |
| Placeholder rows (`_À compléter_`) | Zero signal, erodes trust | Remove |
| Stale context from months ago | Actively misleads Claude | Remove or rotate to `progress.md` |

---

## The memory-maintainer Skill

The `memory-maintainer` skill is the tool for maintaining all three versioned layers. It reads
your memory files, identifies problems, and proposes reviewable patches before writing anything.

Full specification: [`.claude/skills/memory-maintainer/SKILL.md`](../.claude/skills/memory-maintainer/SKILL.md)

### What it does automatically

**First**, before any audit or patch work, the skill checks whether the three standard
memory-bank files exist. If any are missing it enters bootstrap mode: it reads the canonical
templates from `references/templates/` and proposes creating the missing files as a labeled
bootstrap patch set (`P-init-1`, `P-init-2`, `P-init-3`). The developer applies them in one
step and the structure is ready. No manual file creation required.

**Then**, during every audit or patch run:

1. **Reads all versioned memory files** and checks for duplication, contradictions, stale content,
   and misplaced material
2. **Checks `activeContext.md` against git log** — flags any entry whose associated work appears
   merged or completed
3. **Scans 3–5 key service/persistence files** and compares against `systemPatterns.md` — proposes
   additions using the two-occurrence rule
4. **Reads auto-memory** (or `/memory` output) and proposes durable promotions into versioned
   memory
5. **Scans the current conversation** for explicit decisions, confirmed patterns, or discoveries
   when the user asks to promote something, or when such statements are clearly present
5. **Verifies implementation status** against code and git before patching — never trusts memory
   files alone to reflect current code state

### The three-mode workflow

Always follow **audit → patch → apply**. Never skip to apply without review.

```
audit  →  patch  →  apply
(read)    (propose)  (write)
```

**Audit**: understand the current state. Block 4 lists patch intents with no diffs. Nothing
is written.

**Patch**: review the actual unified diffs before anything is written. This is the human review
gate — the most important step.

**Apply**: write the approved changes. Can be selective: apply only `P2`, or only the file
`activeContext.md`.

---

## When to Use the Skill During a Project

### At project start (once)

Just invoke the skill — it will detect the missing memory-bank files and propose creating all
three from the standard templates as a bootstrap patch set. Review and apply in one step.

```text
mode: patch
scope: full
include_auto_memory: false
notes: Initialize memory structure for new project.
```

The skill will produce `P-init-1`, `P-init-2`, `P-init-3` for the three memory-bank files, plus
any other patches for existing docs that need cleaning or reorganising. Apply the init patches
first, then review the rest.

After bootstrap, run a second pass to audit any existing `CLAUDE.md` or `docs/ai/` content and
confirm the layer routing fits the project's specific file structure.

---

### At the start of a significant new workstream

Before starting a major refactor, new feature area, or architectural change, update
`activeContext.md` with the new focus. The skill will check whether previous entries should be
rotated to `progress.md` first.

```text
mode: patch
scope: targeted
paths:
- .claude/memory-bank/activeContext.md
notes: Update focus for [workstream name]. Rotate completed items.
```

---

### When a milestone is delivered

After a PR is merged or a significant piece of work is completed, rotate the entry from
`activeContext` to `progress`, and ask the skill whether any pattern discovered during the work
qualifies for `systemPatterns.md`.

```text
mode: apply
apply_target: file:.claude/memory-bank/activeContext.md
notes: Rotate completed item [X] to progress.md.
```

---

### Periodically — every 2 to 4 weeks

Run a full audit to catch accumulating staleness, surface patterns that have become recurrent
but undocumented, and promote any durable auto-memory observations.

```text
mode: audit
scope: full
include_auto_memory: true
```

This is the maintenance cadence that keeps the memory system alive. Without it, memory drifts
toward the same state as a wiki: comprehensive on day one, trusted by no one after six months.

---

### After any session involving a non-obvious implementation decision

If you and Claude resolved a tricky design question and the same pattern is likely to appear
again, run a targeted patch to document it.

```text
mode: patch
scope: targeted
paths:
- .claude/memory-bank/systemPatterns.md
notes: Document the pattern we just validated for [specific scenario].
```

---

## Practical Usage Examples

For invocation syntax, iteration patterns, and apply examples, see:
[`references/user-manual.md`](../.claude/skills/memory-maintainer/references/user-manual.md)

For a decision table on where each type of information belongs, see:
[`references/promotion-matrix.md`](../.claude/skills/memory-maintainer/references/promotion-matrix.md)

For example audit, patch, and apply responses (what to expect), see:
[`references/report-examples.md`](../.claude/skills/memory-maintainer/references/report-examples.md)

---

## Routing Decision Table

| Information type | Destination |
|-----------------|-------------|
| How Claude should behave — repo-wide | `CLAUDE.md` |
| How Claude should behave — subtree only | Local `CLAUDE.md` |
| Reusable specialized rule | `.claude/rules/*.md` |
| Architecture boundaries and ADRs | `docs/ai/architecture-decisions.md` |
| Domain glossary | `docs/ai/domain-glossary.md` |
| Recurring operational pitfall or known bug | `docs/ai/known-issues.md` |
| Integration pattern for a new source/module | `docs/ai/integration-patterns.md` |
| Current work in progress | `.claude/memory-bank/activeContext.md` |
| Completed milestones | `.claude/memory-bank/progress.md` |
| Recurring codebase pattern (2+ occurrences) | `.claude/memory-bank/systemPatterns.md` |
| Auto-memory observation — durable only | Promote via skill to the appropriate layer above |
| Conversation — explicit decision or confirmed pattern | Promote via skill to the appropriate layer above |
| Everything else | Do not store |

---

## Enrichment Sources per File

This table answers: *"How does each file get updated by `memory-maintainer`, and by whom?"*

Three columns need clarification:

- **Code scan (enrichment)**: memory-maintainer actively scans 3–5 service/persistence files to extract
  new content. This applies **exclusively to `systemPatterns.md`** via the two-occurrence rule.

- **Systematic verification**: runs on all entries of the file at every `mode: audit` or `mode: patch`
  invocation, regardless of whether any change is planned. `mode: patch` reruns all audit steps.
  `mode: apply` does not rerun these scans — it only writes a previously proposed patch set.

- **Targeted verification**: triggered only on the specific items being modified or promoted, regardless of
  mode. Applies when a change affects implementation status (marking work done, removing a known issue,
  closing a debt entry). Two targets:
  - *existing entries* — detecting content that has become stale after a refactor
  - *promotion candidates* — confirming validity before writing a new entry

| File | Human (manual edit) | Human (explicit promotion) | Conversation | Auto-memory | Code scan (enrichment) | Systematic verification | Targeted verification |
|------|:-------------------:|:--------------------------:|:------------:|:-----------:|:----------------------:|:-----------------------:|:---------------------:|
| `CLAUDE.md` (root) | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing entries |
| Local `CLAUDE.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing entries (via registry) |
| `.claude/rules/*.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing entries |
| `docs/ai/architecture-decisions.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing + candidates |
| `docs/ai/domain-glossary.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing + candidates |
| `docs/ai/known-issues.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing + candidates |
| `docs/ai/integration-patterns.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing + candidates |
| `docs/ai/data-sources.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | existing + candidates |
| `docs/ai/MEMORY_ARCHITECTURE.md` | ✓ | — | — | — | — | — | — |
| `.claude/memory-bank/activeContext.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | git log — all entries | covered by systematic |
| `.claude/memory-bank/progress.md` | ✓ | ✓ | ✓ reactive | ✓ reactive | | | git log — existing entries |
| `.claude/memory-bank/systemPatterns.md` | ✓ | ✓ | ✓ (confirmed) reactive | ✓ reactive | ✓ systematic | | existing + candidates |
| `references/templates/*.md` | ✓ | — | — | — | — | — | — |

> **reactive**: the update only happens if the content appears in the current conversation or
> auto-memory — no proactive scan. Only two files have a forcing function (systematic verification):
> `activeContext.md` (git log scan) and `systemPatterns.md` (code scan). All other enrichment
> sources are reactive.

> **Note on `domain-glossary.md`**: business terms with no equivalent in the codebase can only be
> provided by a human. Terms derivable from code identifiers or confirmed in conversation can be
> promoted automatically — the audit → patch → apply workflow is the human review gate.

> **Note on `systemPatterns.md` conversation source**: conversation-sourced patterns require explicit
> user confirmation or observation in 2+ places in the codebase before promotion. A pattern mentioned
> once in passing does not qualify.

---

## Structure Contract

This section is the authoritative reference for the complete file tree and internal structure
of every memory file. It serves as a shared contract between `memory-bootstrap` (which generates)
and `memory-maintainer` (which maintains).

Any deviation from this contract detected during a `memory-maintainer` audit will be flagged
and corrected via a patch proposal.

---

### Complete file tree

```
<project-root>/
│
├── CLAUDE.md                                  ← Layer 1: global prescriptive rules (repo-wide)
│
├── docs/ai/                                   ← Layer 2: durable explanatory knowledge
│   ├── architecture-decisions.md              ← ADRs and architectural trade-offs
│   ├── domain-glossary.md                     ← Business and domain terms
│   ├── known-issues.md                        ← Recurring bugs, pitfalls, operational issues
│   ├── integration-patterns.md                ← Patterns for adding data sources or modules
│   ├── MEMORY_ARCHITECTURE.md                 ← This file — memory system contract
│   └── [other project-specific files]         ← Only if no existing file covers the content
│
├── .claude/
│   ├── claude-files-registry.md               ← Local CLAUDE.md discovery and routing registry
│   │     Created by bootstrap, maintained by memory-maintainer (paths only)
│   │
│   ├── rules/                                 ← Layer 1: specialized reusable prescriptive rules
│   │   ├── database-operations.md
│   │   ├── security.md
│   │   ├── java-spring.md
│   │   ├── hexagonal-architecture.md
│   │   ├── error-handling.md
│   │   └── [other cross-cutting rules]
│   │
│   ├── memory-bank/                           ← Layer 3: evolving project context
│   │   ├── activeContext.md                   ← Current work (expires when done)
│   │   ├── progress.md                        ← Append-only milestone log (permanent)
│   │   └── systemPatterns.md                  ← Recurring implementation patterns (2+ rule)
│   │
│   ├── commands/                              ← Slash commands
│   │   ├── memory-bootstrap.md
│   │   ├── promote-to-memory.md
│   │   └── references/
│   │       ├── memory-bootstrap-user-manual.md
│   │       └── memory-bootstrap-dependencies.md
│   │
│   └── skills/memory-maintainer/             ← Maintenance skill
│       ├── SKILL.md
│       └── references/
│           ├── templates/
│           │   ├── activeContext.md           ← Canonical template (structure source of truth)
│           │   ├── progress.md                ← Canonical template
│           │   └── systemPatterns.md          ← Canonical template
│           ├── promotion-matrix.md
│           ├── report-examples.md
│           └── user-manual.md
│
└── [local CLAUDE.md files]                    ← Layer 1: directory-specific rules (only if justified)
    └── src/main/java/.../SomeModule/CLAUDE.md
```

**Layer 4 (auto-memory)** lives outside the repository:
```
~/.claude/projects/<encoded-project-path>/memory/
    MEMORY.md          ← index (never edit directly)
    *.md               ← observational entries (never edit directly)
```

---

### Internal structure of memory-bank files

These structures are enforced by the canonical templates in
`.claude/skills/memory-maintainer/references/templates/`. Bootstrap must generate files that
match these structures exactly. Memory-maintainer audits against them.

#### `activeContext.md`

```markdown
# Active Context

> **Lifecycle rules**
> - Entries here describe CURRENT work only.
> - When done → move the entry to `progress.md`. Do not leave completed work here.
> - When blocked/paused → note the blocker inline with a date.
> - This file must stay short (~15 lines max). If it grows beyond that, prune it.

## Current Focus

<!-- What is actively being worked on right now -->

## Active Risks

<!-- Blockers or risks that need resolution before work can proceed -->

## Pending Decisions

<!-- Decisions not yet made but needed to move forward -->
```

**Constraints**: max ~15 lines of content. Entries rotate to `progress.md` when work is done.

---

#### `progress.md`

```markdown
# Progress Log

> **Append-only.** Add entries when work is completed; never delete or edit existing entries.
> One entry per milestone. Include date and a reference (PR, commit, or ticket).

| Date | Milestone | Reference |
|------|-----------|-----------|
| YYYY-MM-DD | Description of completed milestone | commit/PR/ticket |
```

**Constraints**: append-only — never edit or delete existing rows.

---

#### `systemPatterns.md`

```markdown
# System Patterns

> **Entry threshold**: add a pattern when the same approach appears in 2 or more distinct files
> in the codebase, OR when a non-obvious design decision has been validated by the team.
> Format: pattern name, when to use it, minimal code example. No prose essays.

## Persistence Patterns

<!-- How entities are saved, sorted, and mapped in this codebase -->

## Service Patterns

<!-- Orchestration, error collection, domain model construction -->

## Integration Patterns

<!-- API clients, enrichment generators, data fetch and mismatch handling -->

## Error Handling Patterns

<!-- Error creation, source filtering, status management -->
```

**Constraints**: two-occurrence rule — never promote a pattern seen in only one file.

---

### What bootstrap generates vs what memory-maintainer maintains

| Layer | Generated by bootstrap | Maintained by memory-maintainer |
|-------|----------------------|--------------------------------|
| `CLAUDE.md` | ✓ populated from analysis | ✓ patches on rule changes |
| `docs/ai/*` | ✓ populated from analysis | ✓ patches on content drift |
| `.claude/rules/*` | ✓ populated from analysis | ✓ patches on rule changes |
| `activeContext.md` | ✓ populated from git log + docs | ✓ lifecycle (rotate to progress) |
| `progress.md` | ✓ populated from git log | ✓ append new milestones |
| `systemPatterns.md` | ✓ populated from code scan | ✓ add patterns (two-occurrence rule) |
| Templates in `references/templates/` | ✗ never touched | ✗ never touched (source of truth) |
| Auto-memory (`~/.claude/...`) | ✗ never touched | ✓ reads as input, promotes to layers 1–3 |

---

## Developer Workflow — Who Does What and When

This section describes the healthy usage workflow. Each file in the memory system has a defined
owner and a defined set of actors. Understanding this prevents both under-maintenance (memory
goes stale) and over-engineering (humans doing what tools should do).

---

### Responsibility matrix

| File | Generated by `/memory-bootstrap` | Human responsibility | `memory-maintainer` role |
|------|----------------------------------|---------------------|--------------------------|
| `CLAUDE.md` | Skeleton + rules inferred from code | Complete business rules not derivable from code | Correct misplaced or contradictory rules |
| `docs/ai/architecture-decisions.md` | ADRs visible in code and git | Add the *why* behind decisions (context not in code) | Correct, consolidate, flag stale decisions |
| `docs/ai/domain-glossary.md` | Terms found in code identifiers | Define business terms the code does not explain | Correct ambiguous definitions |
| `docs/ai/known-issues.md` | Issues visible in existing docs | Add production problems not yet written down | **Primary promotion target** — every reproduced bug lands here |
| `docs/ai/integration-patterns.md` | Patterns visible in code | Add edge cases and external constraints | Correct, add newly confirmed patterns |
| `docs/ai/runbook-dev.md` | Commands found in README / pom | Add operational procedures not in docs | Correct when commands change |
| `.claude/rules/*.md` | Rules inferred from code | Validate and refine generated rules | Correct contradictions, consolidate duplicates |
| `activeContext.md` | Populated from git log + existing docs | Keep active focus current | **Owns lifecycle** — rotate to progress when done, prune if > 15 lines |
| `progress.md` | Populated from git log (last 5–10 milestones) | Add milestones not tracked in git | Append new entries at each delivered milestone |
| `systemPatterns.md` | Patterns confirmed in 2+ files (code scan) | Validate patterns proposed by bootstrap | Scan code at each audit, add patterns (two-occurrence rule) |
| Local `CLAUDE.md` | Generated if justified by directory | Complete module-specific rules | Correct if content belongs to another layer |

---

### The three roles of memory-maintainer

Memory-maintainer acts differently depending on the source it is working with.

```
Role 1 — CORRECTOR
  Reads versioned files → detects contradictions, duplicates, wrong-layer placement
  Target: all files in layers 1–3

Role 2 — PROMOTER
  Reads auto-memory + current conversation → extracts durable content → writes to layers 1–3
  Sources:  ~/.claude/projects/.../memory/,  current conversation
  Typical destinations: known-issues, systemPatterns, progress, rules

Role 3 — LIFECYCLE MANAGER
  Manages activeContext.md exclusively
  Actions: rotate (done → progress.md), prune (> 15 lines), flag (stale entries)
```

---

### What only a human can provide

Regardless of how thorough bootstrap or memory-maintainer are, these items **can only come
from a human**. No tool can infer them from code or git history:

- Rationale behind architectural decisions (the *why*, not the *what*)
- Business domain terms with no equivalent in the codebase
- Production problems known to the team but not written anywhere
- Compliance or security rules not derivable from code
- Current organizational priorities and deadlines
- Team agreements made verbally or in meetings

These are the gaps that Step 5 (human completion checklist) of `/memory-bootstrap` surfaces,
and that `memory-maintainer` marks as `<!-- À compléter -->` when it cannot fill them.

---

### Healthy cadence at a glance

```
Project start (once)
  └─ /memory-bootstrap
       → generates all layers with real content
       → human completes the À compléter sections

After a codebase-analysis session (option 1 or option 2 export)
  └─ /promote-to-memory doc=<analysis-file>  (or via /architecture-document option 1)
       → promotes PROMOTE-CANDIDATE items (rules, patterns, domain terms,
         integration patterns, data flows, decisions) to the correct layer
       → prerequisite: destination files must exist (run /memory-bootstrap if not)

After each significant session
  └─ /promote-to-memory
       → extracts decisions and patterns from the conversation
       → promotes to the correct versioned layer

After each merged PR or milestone
  └─ memory-maintainer (apply mode)
       → rotates activeContext entry to progress.md

Every 2–4 weeks
  └─ memory-maintainer (audit → patch → apply)
       → catches staleness, new patterns, stale activeContext entries
       → promotes durable auto-memory observations

When starting a new workstream
  └─ memory-maintainer (targeted patch on activeContext.md)
       → updates focus, rotates completed items
```

---

## Adoption Checklist for a New Project

Use this checklist when setting up the memory architecture on a project for the first time.

- [ ] Copy the `memory-maintainer` skill into `.claude/skills/memory-maintainer/`
- [ ] Run `/memory-bootstrap` — generates all layers populated with real project content
- [ ] Complete the `À compléter` sections (business rules, domain terms, operational pitfalls)
- [ ] Schedule a recurring audit reminder every 2–4 weeks

---

## Silent Drift — What memory-maintainer Cannot Detect Automatically

Some files can become stale without `memory-maintainer` ever detecting it. Understanding this is
critical for calibrating the 2–4 week audit cadence and the human review discipline.

### Fully silent drift

No proactive scan exists for these files. Drift is only caught when that specific item is being
actively patched — which may never happen.

| File | Example of undetected drift |
|------|-----------------------------|
| `CLAUDE.md` (root) | Rule references a class renamed or deleted in a refactor |
| `.claude/rules/*.md` | Rule obsolete after an architectural change |
| `docs/ai/architecture-decisions.md` | ADR invalidated by a subsequent decision never recorded |
| `docs/ai/domain-glossary.md` | Term redefined or removed from the domain |
| `docs/ai/known-issues.md` | Issue documented as active but already fixed in code |
| `docs/ai/integration-patterns.md` | Pattern obsolete after an integration change |
| `docs/ai/data-sources.md` | Data flow modified or removed |
| Local `CLAUDE.md` files | Content drift within the file — discovery gap resolved via registry, but content still reactive |

### Partially detected drift

| File | What is detected | What escapes |
|------|-----------------|--------------|
| `activeContext.md` | Completed entries (git log scan, every audit/patch) | New work started without updating the file |
| `systemPatterns.md` | Missing patterns (code scan, every audit/patch) | Existing patterns that became stale after a refactor |
| `progress.md` | N/A — entries are permanent by design | Milestones delivered without going through memory-maintainer |

### Write access vs detection

`memory-maintainer` can write to all files listed above — the bottleneck is **detection, not
access**. A rule that has become obsolete in `CLAUDE.md`, or an issue fixed in code but still
active in `known-issues.md`, could be corrected by the skill the moment it is identified. The
problem is that without a proactive scan, the skill never sees it.

The only files the skill will never write to regardless of what is detected:
- `docs/ai/MEMORY_ARCHITECTURE.md` — adoption documentation, never updated by the skill
- `references/templates/*.md` — canonical templates, source of truth
- `~/.claude/.../memory/` — auto-memory, read-only by contract

### Mitigation

The 2–4 week full audit cadence is the primary mitigation for fully silent files. During each
audit, manually ask: *"Has anything in this file become false since the last audit?"* for the
high-risk files — especially `known-issues.md` and `CLAUDE.md`.

The `/promote-to-memory` skill run after each significant session is the secondary mitigation:
decisions made and issues discovered during the session are promoted before they are forgotten.

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `activeContext.md` lists finished work | No lifecycle discipline | Run audit; rotate to `progress.md` |
| `systemPatterns.md` is empty after months | No forcing function | During each audit, skill scans for 2+ patterns |
| Two files describe the same bug | Duplication crept in | Audit detects this; consolidate to `known-issues.md` |
| Memory files reference classes that no longer exist | Memory lagged behind refactor | Skill verifies against code before patching |
| Claude ignores memory | Files are too long and low-signal | Prune aggressively; high signal density matters |
| `known-issues.md` documents a fixed bug as active | Silent drift — no proactive scan | Run targeted patch; verify against git log and code |
