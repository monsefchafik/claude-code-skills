# Architecture Documentation Pattern

This document describes the pattern and workflow for documenting complex project mechanisms,
managing architecture knowledge economically in context, and leveraging it effectively
in all future tasks.

It is the counterpart of [MEMORY_ARCHITECTURE_EN.md](MEMORY_ARCHITECTURE_EN.md) for the
architecture documentation layer (distinct from the project memory layer).

---

## ⚠️ Prerequisite — Mandatory wiring in `CLAUDE.md`

**Without this step, the `architecture-document` and `architecture-reviewer` skills do not
work as expected.** Claude does not know that `architecture-map.md` exists and will never
consult it, even after several documented sessions.

### What must be present in the root `CLAUDE.md`

```markdown
## Documented complex mechanisms

@docs/ai/architecture-map.md

If a task touches a mechanism listed in this map, read the corresponding `summary.md`
BEFORE proposing anything. The path is indicated in the mechanism's entry.

For an in-depth analysis before a complex modification, use the subagent:
Use the architecture-reviewer. mechanism: <name>. task: <description>
Specification: `.claude/skills/architecture-reviewer/SKILL.md`

⚠️ `architecture-reviewer` = ALWAYS via Agent tool (isolated subagent).
Never read `full-analysis.md` directly in the main thread.
```

**Why the instruction is imperative ("BEFORE proposing anything"):**
A suggestive phrasing ("consult if needed") lets Claude decide to skip the read.
An imperative phrasing guarantees that `summary.md` is read before any proposal — this is
the difference between ~70% and ~95% reliability on documented mechanisms.

**Why the Agent tool rule is in `CLAUDE.md`:**
Claude knows it must spawn a subagent thanks to 3 cumulative signals: the skill description
in the system-reminder ("Invoke via the Agent tool"), the content of `SKILL.md` loaded on
invocation, and this line in `CLAUDE.md`. Without this third signal, a vague phrasing
("look at the docs for X") can lead Claude to read `full-analysis.md` directly in the main
thread — nullifying the isolation benefit. The canonical phrasing guarantees:
`Use the architecture-reviewer. mechanism: <name>. task: <description>`.

### Why this is critical

| Without wiring | With wiring |
|-------------|-------------|
| Claude ignores `architecture-map.md` | Claude loads the map every session |
| Documented mechanisms never consulted | Claude knows to check before touching |
| `architecture-reviewer` never naturally invoked | Claude knows how to invoke it |
| Skills produce orphan docs | Docs connected to active memory |

### Verification

This wiring is done **once** at pattern initialization.
To verify it is in place, read the root `CLAUDE.md` and confirm that the section
"Documented complex mechanisms" with `@docs/ai/architecture-map.md` is present.

### What must NOT be in `CLAUDE.md`

```markdown
# FORBIDDEN — too costly in context
@docs/ai/architecture/<mechanism>/full-analysis.md
@docs/ai/architecture/<mechanism>/summary.md   # except local CLAUDE.md, HIGH risk mechanism
```

Root `CLAUDE.md` imports only `architecture-map.md` (short hub).
Individual `summary.md` files are imported only in local CLAUDE.md files of HIGH risk modules
— never at the root.

---

## Why this pattern?

### The problem

A reverse-engineering session produces 1500+ lines of valuable content.
If this content is stored in a single file and referenced directly from `CLAUDE.md`,
it is loaded in every conversation — even those that don't need it.

Result: polluted context, diluted signal, high cost, less precise Claude.

### The solution

```
Separate archive from operational reference.
Load only what is useful for the current task.
Delegate reading of archives to an isolated subagent.
```

**Core principle**:
> Active memory = short summaries and action rules.
> Long document = expertise archive consulted on demand.
> Subagent = specialized reader that does not pollute the main thread.

---

## The 3-file structure

For each documented complex mechanism:

```
docs/ai/architecture/
  <mechanism-name>/
    summary.md         → 50-100 lines  — operational action guide
    decision.md        → 100-200 lines — decision context
    full-analysis.md   → no limit      — complete passive archive
```

### What each file contains

#### `summary.md` — The action guide

Answers one question: *"Can I modify this mechanism safely?"*

```markdown
## Role
## Current architecture (diagram or short list)
## Sensitive points (what can break)
## Implementation rules (what must always be respected)
## Evolution rules (how to extend safely)
## References → full-analysis.md, decision.md
```

**Hard limit: 100 lines.** If longer, it is a comprehension document,
not an action guide — shorten it.

#### `decision.md` — The decision context

Answers: *"Why was this choice made and what alternatives were ruled out?"*

```markdown
## Decision made
## Ruled-out alternatives (table with reasons)
## Constraints that guided the choice
## Impact on code
## Rules derived from this decision
```

Consulted when a task questions the architecture or requires understanding the reasoning
behind the choices.

#### `full-analysis.md` — The passive archive

Contains everything discovered, discussed, measured, compared during the reverse-engineering
session. No size limit.

**Never @import.** Consulted only on demand, via the subagent or directly
when the complete analysis is needed.

---

## The routing hub — `architecture-map.md`

`docs/ai/architecture-map.md` is the central index of all documented units.
It is **the only architecture file imported via @ in `CLAUDE.md`**.

Structure of an entry:

```markdown
## <Name> [<type>]

**Type**: component | module | subsystem | bounded-context
**Scope**: [packages or classes involved]
**Risk**: HIGH | MEDIUM | LOW
**Summary**: [one sentence describing the role]
**⚠️ Required**: READ `docs/ai/architecture/<name>/summary.md` before any modification.
**Rule**: [main implementation constraint in one line]
**Full analysis**: `docs/ai/architecture/<name>/full-analysis.md` (do not @import)
```

It stays short (~16 lines per unit) even with 10 documented units.
Claude reads the entry, knows the type, and knows exactly where to go if the task touches this unit.

---

## Memory injection points

### What must be @imported (loaded every session)

| File | Via @ in | Condition |
|---------|-----------|-----------|
| `docs/ai/architecture-map.md` | Root `CLAUDE.md` | As soon as a mechanism is documented |
| `docs/ai/architecture/<name>/summary.md` | Local CLAUDE.md of the module | HIGH risk mechanism only |

### What must never be @imported

| File | Why |
|---------|---------|
| `full-analysis.md` | Too long — loads the entire context unnecessarily |
| `decision.md` | Consulted occasionally, not every session |

### Instruction in root `CLAUDE.md` (added once)

```markdown
## Documented complex mechanisms
Before modifying a risk module, consult:
@docs/ai/architecture-map.md

For an in-depth analysis of a mechanism, use the subagent:
architecture-reviewer (see .claude/skills/architecture-reviewer/SKILL.md)
```

### Instruction in a local CLAUDE.md (for HIGH risk modules)

```markdown
## Documented critical mechanism

This module implements a complex mechanism.
Before any modification:
@docs/ai/architecture/<mechanism>/summary.md

Full analysis (do not @import):
docs/ai/architecture/<mechanism>/full-analysis.md
```

---

## The complete workflow

### Phase 1 — Acquisition (3 modes)

The skill supports three input sources. Choose according to context:

#### Phase 1a — Work session (source: conversation)

```
Developer + Claude explore the code together.
Questions asked, answers found, patterns identified.
Everything stays in the conversation — write nothing yet.

→ /architecture-document <mechanism-name>
  The skill extracts everything from the conversation.
```

#### Phase 1b — Existing document (source: markdown file)

```
You have an existing document: technical analysis, migration plan,
architectural evolution study, RFC, past session notes, etc.

→ /architecture-document <name> type=<type> doc=<path>
  The skill reads the file and extracts its content.
```

This mode works for two families of documents:

**Retrospective documents** (analysis of an existing system): code analyses, post-mortems,
exploration session notes. Content primarily feeds `full-analysis.md` and `summary.md`.

**Prospective documents** (evolution study, RFC): new target architecture,
solution comparison, technical justification, performance notes, development strategy
with prioritization. Content primarily feeds `decision.md` and implementation rules
via PROMOTE-CANDIDATE.

**Section-by-section reading for documents ≥ 300 lines**:

```
1. The skill reads the first 50 lines → detects structure and ## headings
2. Presents the section list:
   "This document contains X sections (~N lines). Reading section by section:
    § 1. Context (l.1-80)
    § 2. Current architecture (l.81-250)
    § 3. Identified problems (l.251-480)
    ..."
3. Waits for confirmation (you can exclude sections)
4. Reads each section individually (offset + limit)
5. Classifies each section into the right destination file
6. Merges buckets → extraction Step 3
```

**Routing by section type:**

| Keywords in heading | Destination |
|------------------------|-------------|
| Architecture, Structure, Components | `full-analysis` + `summary` candidates |
| Problems, Risks, Edge cases, Bugs | `full-analysis` + `known-issues` candidates |
| Options, Alternatives, Comparison, POC, Benchmark | `decision` candidates |
| Chosen solution, Decision, Justification | `decision` + `.claude/rules/` candidates |
| Rules, Constraints, Requirements | `decision` + direct PROMOTE-CANDIDATE |
| Performance, Measurements, Metrics | `full-analysis` + `decision` candidates if comparative |
| Strategy, Prioritization, Roadmap, Steps | `decision` + `summary` candidates |
| History, Context, Background | `full-analysis` archive |
| Code, Implementation, Examples | `full-analysis` + `systemPatterns` candidates |
| Vocabulary, Business terms, Glossary | `domain-glossary` candidates |
| Integration, API, Inter-service flows | `integration-patterns` candidates |
| Data sources, Transformations, Targets | `data-sources` candidates |

**Why this strategy**: reading an 800-line document all at once
overloads the context during generation. Section-by-section reading allows
classifying content as it goes, without risk of loss.

#### Phase 1c — Hybrid (source: file + conversation)

```
You have a prepared document AND a recent validation/decision session.

→ /architecture-document <mechanism-name> doc=<path>
  The skill merges the two sources.
  Rule: the conversation wins in case of conflict (more recent).
```

Typical case: migration plan discussed and refined during the session.

### Phase 2 — Generation (common to all 3 modes)

```
The skill:
  1. Infers or confirms the mechanism name
  2. Reads: systemPatterns.md + architecture-map.md + activeContext.md
  3. Extracts and merges content from active sources
  4. Drafts the 3 files + 2 patches (map + ADR)
  5. Lists PROMOTE-CANDIDATE (rules for .claude/rules/)
  6. Presents drafts → waits for confirmation
  7. Writes only after approval
```

### Phase 3 — Human validation

```
Review summary.md:
  → Does it answer "can I touch this?"?
  → Is it ≤ 100 lines?
  → Are implementation rules clear and actionable?

Review decision.md:
  → Are ruled-out alternatives documented?
  → Is the reasoning reproducible?

Check patches:
  → architecture-map.md: is the entry correct?
  → architecture-decisions.md: is the ADR properly numbered?
```

### Phase 4 — Promotion of action rules

```
# Same session as /architecture-document
/promote-to-memory

# Later session (PROMOTE-CANDIDATE no longer in conversation)
/promote-to-memory doc=docs/ai/architecture/<name>/full-analysis.md
```

`/promote-to-memory` detects PROMOTE-CANDIDATE items as a priority — via the conversation
or via the `## PROMOTE-CANDIDATE` section of `full-analysis.md`. These items are treated
as confirmed candidates — the durable/non-durable filter is not reapplied to them.

Before each promotion, automatic deduplication against existing target files.

**PROMOTE-CANDIDATE format** (persisted in `full-analysis.md`):
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

The confidence level is preserved between sessions. `[low]` items (prospective or
indirect) are flagged to the human at patch time — they are not automatically
rejected but require special attention during validation.

> **⚠️ Gap 3 — Confidence without grep**: `/architecture-document` does not grep the code.
> The PROMOTE-CANDIDATE items it generates itself are at best `[medium]` unless the source
> document cites explicit code evidence. Items from a `codebase-analysis` export via `doc=`
> retain their original level (based on real code evidence).

Complete destinations:
  → Repo-wide invariant          → .claude/rules/
  → Subtree invariant            → local CLAUDE.md
  → Cross-cutting decision       → docs/ai/architecture-decisions.md
  → Operational constraint       → docs/ai/known-issues.md
  → Recurring pattern (2+)       → .claude/memory-bank/systemPatterns.md
  → Business term                → docs/ai/domain-glossary.md
  → Integration pattern          → docs/ai/integration-patterns.md
  → Data flow                    → docs/ai/data-sources.md

DO NOT promote:
  → Content already in decision.md (duplication)
  → Content already in architecture-decisions.md
  → One-off observations without general scope

**Why PROMOTE-CANDIDATE items are persisted in `full-analysis.md`:**
The conversation disappears when the session closes. Without file persistence,
PROMOTE-CANDIDATE items are lost if `/promote-to-memory` is not launched immediately.
`full-analysis.md` is the inter-session handoff point.

Complete pipeline:
```
/architecture-document  →  Step 4 identifies PROMOTE-CANDIDATE
                        →  Step 6 writes them to full-analysis.md (## PROMOTE-CANDIDATE)
                        →  also displays them in conversation (Step 5)
                                    ↓
/promote-to-memory      →  Source A: ## PROMOTE-CANDIDATE from full-analysis.md (if doc=)
                        →  Source B: conversation (if same session)
                        →  deduplication against existing .claude/rules/
                        →  approval gate → write
```

> **⚠️ Gap 4 — progress.md and activeContext.md not updated automatically**:
> `/architecture-document` never writes to these files. If the documentation is part
> of an ongoing workstream, update `activeContext.md` manually. If it closes a
> milestone, move it from `activeContext.md` to `progress.md`. These updates can also
> be included in the next `/promote-to-memory` pass.

### Phase 5 — Wiring (according to risk level)

```
LOW risk   → No additional wiring.
               The entry in architecture-map.md is sufficient.

MEDIUM risk → Verify that root CLAUDE.md @imports architecture-map.md.

HIGH risk  → Create/complete a local CLAUDE.md in the module.
               @import summary.md (not full-analysis).
```

### Phase 6 — Usage in future tasks

```
Simple task on known mechanism
  → Claude reads architecture-map.md (@imported)
  → Follows the entry rules
  → No further consultation

Complex or risky task
  → "Use the architecture-reviewer. mechanism: X. task: Y."
  → Subagent reads summary.md in isolated context
  → If insufficient: reads full-analysis.md (relevant sections)
  → Returns 20-line recommendation to main thread
  → Main thread continues without having loaded 1500 lines
```

---

## The two tools

### `/architecture-document` — The generator

Invoked at the end of an analysis session or to convert an existing document.
Produces the 3 files and the patches.

```bash
# Source: conversation only
/architecture-document generator-pipeline
/architecture-document                          # infers the name
/architecture-document order-cache lang=en

# Source: markdown file only
/architecture-document doc=docs/PRODUCT_PERSISTENCE_ANALYSIS.md
/architecture-document product-persistence doc=docs/PRODUCT_PERSISTENCE_ANALYSIS.md

# Combined sources: file + conversation
/architecture-document product-sync-pipeline doc=docs/PRODUCT_SYNC_MIGRATION_PLAN.md
```

For documents ≥ 300 lines, the skill automatically applies the section-by-section
reading strategy (presents section list → confirmation → section-by-section reading
→ classification → merge).

Documentation: `.claude/skills/architecture-document/references/user-manual.md`

### `architecture-reviewer` — The isolated reader

Invoked before a complex implementation task. Reads docs in a separate context.

⚠️ **Always via Agent tool (isolated subagent).** Never read `full-analysis.md`
directly in the main thread — this nullifies the isolation benefit.

```
Use the architecture-reviewer. mechanism: product-sync-pipeline. task: Add a new source type without breaking validation.
```

**Why this canonical phrasing**: Claude recognizes the reviewer via 3 signals
(skill description, SKILL.md, root CLAUDE.md). A vague phrasing like "look at the
docs for X" can lead Claude to read files directly without spawning a subagent.
The phrasing `Use the architecture-reviewer. mechanism: X. task: Y.` guarantees isolation.

Documentation: `.claude/skills/architecture-reviewer/references/user-manual.md`

---

## Integration with the existing memory system

This pattern integrates into the 4 memory layers defined in
[MEMORY_ARCHITECTURE_EN.md](MEMORY_ARCHITECTURE_EN.md):

| Memory layer | Role in this pattern |
|---------------|----------------------|
| Layer 1 — Rules (`CLAUDE.md`, `.claude/rules/`) | Receives rules promoted from PROMOTE-CANDIDATE |
| Layer 2 — Knowledge (`docs/ai/`) | Contains the 3 files + architecture-map.md + ADRs |
| Layer 3 — Context (`memory-bank/`) | `systemPatterns.md` receives promoted patterns |
| Layer 4 — Auto-memory | Detection source for `/promote-to-memory` |

**The architecture layer (`docs/ai/architecture/`) is an extension of Layer 2.**
It follows the same rules: durable, explicit, reviewable like code.

### What goes where after a session

**Written directly by the skill (Step 6):**

| File | Note |
|---------|---------|
| `docs/ai/architecture/<name>/full-analysis.md` | Passive archive — do not @import |
| `docs/ai/architecture/<name>/summary.md` | Importable if HIGH risk mechanism |
| `docs/ai/architecture/<name>/decision.md` | Passive — includes mechanism business policies |
| `docs/ai/architecture-map.md` | @imported from root `CLAUDE.md` |
| `docs/ai/architecture-decisions.md` | Short ADR |
| Root `CLAUDE.md` | Bootstrap only (once) — add `@architecture-map.md` |

> **Root `CLAUDE.md` is not a PROMOTE-CANDIDATE destination.** Repo-wide rules
> go into `.claude/rules/`. `CLAUDE.md` is an import hub, not a rules file.

**Via PROMOTE-CANDIDATE (deferred promotion, via `/promote-to-memory`):**

| Destination | Item type |
|-------------|-------------|
| `.claude/rules/` | Repo-wide invariant |
| Local `CLAUDE.md` | Subtree-specific invariant |
| `docs/ai/architecture-decisions.md` | Cross-cutting decision (in addition to the ADR) |
| `docs/ai/known-issues.md` | Operational constraint |
| `.claude/memory-bank/systemPatterns.md` | Recurring pattern (2+) |
| `docs/ai/domain-glossary.md` | Business term — Tier 2 on-demand |
| `docs/ai/integration-patterns.md` | Integration pattern — Tier 2 on-demand |
| `docs/ai/data-sources.md` | Data flow — Tier 2 on-demand |

---

## What the system provides once a mechanism is documented

Once a mechanism is documented and wired, Claude has persistent context for
all future tasks related to it — without you having to re-explain it.

### Capabilities enabled by task type

| Task type | What Claude can do | Condition |
|---------------|--------------------------|-----------|
| **Simple fix** | Respects invariants, identifies sensitive points | Automatic via map + summary |
| **MEDIUM evolution** | Proposes an approach compatible with existing architecture | Automatic via map + summary |
| **HIGH evolution** | Analyzes constraints, verifies compatibility with decision.md | Explicit reviewer |
| **Architectural decision** | Draws on already ruled-out alternatives, avoids going through the same reasoning again | Reviewer + decision.md |
| **Reverse engineering** | Starts from full-analysis.md instead of zero — accelerates understanding | On demand |
| **Safe refactoring** | Knows invariants to preserve, contracts to respect | Automatic via summary |
| **Code review** | Verifies compliance with documented implementation rules | Automatic via summary |

### What you will never have to do again

Once documented, you no longer have to:
- re-explain how the mechanism works every session
- remind Claude of constraints to respect before each modification
- rediscover edge cases already analyzed
- reconstruct the reasoning behind architectural choices

### The three limits to know

**1. Recognition depends on naming**
If the task says "modify the CatalogSync client" but the map uses `product-sync-pipeline`,
Claude may not make the connection. The map must list involved packages and classes
precisely. When in doubt, explicitly mention the mechanism name in the task.

**2. The reviewer is not automatic for complex tasks**
For simple modifications: `summary.md` is sufficient and loaded automatically.
For complex modifications: **explicitly invoke the reviewer** — Claude does not spawn
a subagent on its own initiative.

**3. Docs age if code evolves without updates**
An obsolete `summary.md` is **actively dangerous**: Claude will work on an incorrect
basis with unjustified confidence. See the next section.

---

## ⚠️ Keep documentation synchronized with code

**This is the most important rule of the entire pattern.**

An architecture document that no longer reflects real code is worse than having no
documentation. Claude will trust it and propose modifications based on a reality that
no longer exists.

> **Principle**: architecture docs are code. They evolve with it, are reviewed like it,
> and their obsolescence is a bug.

### When to update — mandatory

| Event in code | File(s) to update | Urgency |
|------------------------|---------------------------|---------|
| Refactoring of a documented mechanism | `summary.md` + `architecture-map.md` entry | **Immediate** |
| Change to an implementation rule | `summary.md` + relevant `.claude/rules/` | **Immediate** |
| Addition of a new component in scope | `summary.md` "Current architecture" section | Before next PR |
| Revised architectural decision | `decision.md` + `architecture-decisions.md` | Before next PR |
| New emerging pattern (2+ occurrences) | `full-analysis.md` + `systemPatterns.md` | End of session |
| Bug discovered related to mechanism | `full-analysis.md` edge cases section + `known-issues.md` | End of session |
| Modified external constraint (API, DB, infra) | `full-analysis.md` + `summary.md` if rule impact | Before next PR |

### Update criticality hierarchy

```
CRITICAL — update in the same PR as the code:
  summary.md              → modified implementation rules
  architecture-map.md     → modified scope or risk
  .claude/rules/          → modified invariants

IMPORTANT — update before the next PR:
  decision.md             → revised decision
  architecture-decisions.md → new ADR

USEFUL — update at end of session:
  full-analysis.md        → new discoveries, edge cases
  systemPatterns.md       → newly confirmed patterns
  known-issues.md         → newly identified risks
```

### How to detect obsolescence

**Human signal (most reliable)**:
Claude proposes something that contradicts what you know about the code →
`summary.md` or the rules are obsolete. Update immediately.

**Claude signal**:
During a task, if Claude says "according to documentation, X should work like this"
but you know X has changed → stop, fix the doc, resume.

**`memory-maintainer` signal** (periodic audit):
Every 2-4 weeks, the audit compares memory files with real code.
For architecture files, explicitly add in the audit request:

```text
mode: audit
scope: targeted
paths:
- docs/ai/architecture-map.md
- docs/ai/architecture/<mechanism>/summary.md
notes: Check consistency with current mechanism code.
```

**Git signal**:
After a PR merge touching a documented mechanism, check whether the relevant
architecture files were updated in the same commit or in a follow-up PR.

### Update workflow

```
Code modified in a documented mechanism
  │
  ├─ Minor change (adding a class, internal refactor)
  │    → Edit summary.md directly
  │    → Update architecture-map.md entry if scope changes
  │
  ├─ Significant change (new rule, new pattern)
  │    → Edit summary.md + decision.md if needed
  │    → /promote-to-memory for new rules → .claude/rules/
  │    → memory-maintainer to validate consistency
  │
  └─ Major refactoring (revised architecture)
       → New analysis session if needed
       → /architecture-document <mechanism> to regenerate
       → Keep old full-analysis.md as archive (rename with date)
       → New full-analysis.md for current state
```

### What must NOT be updated

`full-analysis.md` is an **archive** — do not rewrite history.
If the architecture changes significantly, create a new `full-analysis.md` and
archive the old one (e.g.: `full-analysis-2025-03.md`). The history of reasoning
has value.

---

## Evolution complexity levels

The reviewer adapts its analysis depth according to the announced complexity of the task:

| Complexity | Reviewer behavior | Minimum documentation needed |
|------------|--------------------------|-----------------------------------|
| Simple | Reads summary.md only | summary.md |
| Medium | Reads summary.md + targeted sections of full-analysis | summary.md + full-analysis.md |
| Complex | Reads summary.md + decision.md + full full-analysis.md | All 3 files |
| Very complex | Same as complex + recommends a new analysis session | All 3 files + new session |

---

## Warning signals

### Structure signals (wiring and format)

| Signal | Cause | Action |
|--------|-------|--------|
| `summary.md` > 100 lines | Not focused enough | Prune — remove what belongs in `full-analysis.md` |
| `architecture-map.md` not @imported in `CLAUDE.md` | Missing wiring | Add the instruction in `CLAUDE.md` |
| PROMOTE-CANDIDATE not promoted after generation | Rules remain in passive docs | Run `/promote-to-memory` |
| HIGH risk mechanism without local `CLAUDE.md` | Claude does not load `summary.md` automatically | Create the local `CLAUDE.md` |

### Obsolescence signals (⚠️ most dangerous)

| Signal | Cause | Action |
|--------|-------|--------|
| Claude proposes something that contradicts real code | Obsolete `summary.md` | Update immediately before continuing |
| PR merged without updating architecture docs | Missing discipline | Update in follow-up PR |
| Class or package listed in map that no longer exists | Undocumented refactoring | Update `architecture-map.md` + `summary.md` |
| Rule in `summary.md` that no longer applies | Code evolved without update | Fix `summary.md` + check `.claude/rules/` |
| `decision.md` describes a "ruled-out" alternative that was finally implemented | Untracked revised decision | Update `decision.md` + new ADR |
| `full-analysis.md` describes behavior contradicted by current tests | Architecture changed | Create new `full-analysis.md`, archive old one with date |
| `memory-maintainer` audit detects non-existent classes in docs | Docs preceded or followed a refactoring | Run `/architecture-document` to regenerate |

---

## Appendix A — Performance and context economy

### Base assumptions

```
full-analysis.md  : ~800 lines  → ~16,000 tokens
summary.md        : ~75 lines   → ~1,500 tokens
decision.md       : ~150 lines  → ~3,000 tokens
map entry         : ~15 lines   → ~300 tokens
reviewer response : 20 lines    → ~400 tokens
```

### Scenario comparison (per session)

| Scenario | Tokens in main thread | Notes |
|----------|---------------------------------|-------|
| **Without system** — developer re-explains context | ~3,000 | Risk of forgetting invariants |
| **Anti-pattern** — `@full-analysis.md` in CLAUDE.md | ~16,000 per mechanism | Loaded even when useless |
| **Anti-pattern × 5 mechanisms** | ~80,000 (fixed) | 40% of Sonnet context saturated |
| **With system** — map + reviewer | ~1,900 | 1,500 (map) + 400 (reviewer) |

### Measured reduction

| Comparison | Token reduction | Context reduction |
|-------------|------------------|--------------------|
| vs @import anti-pattern (1 mechanism) | −14,100 tokens | **−88%** |
| vs @import anti-pattern (5 mechanisms) | −78,100 tokens | **−97.6%** |
| vs manual re-explanation | −1,100 tokens | **−37%** but full coverage |

### Behavior at scale

```
Anti-pattern: cost ∝ number of mechanisms → context saturated at 10-12 mechanisms
With system: fixed cost (map) + on-demand cost (reviewer)
→ architecture memory cost stays constant regardless of growth
```

At 10 documented mechanisms:
- Anti-pattern: ~160,000 fixed tokens → practical impossibility on Sonnet (200K)
- With system: ~3,000 fixed tokens + occasional reviewer

### Summary

| Axis | Score | Detail |
|-----|-------|--------|
| Visibility | ★★★★★ | 3 graduated access levels — nothing is lost |
| Context isolation | ★★★★★ | −97% vs anti-pattern, stable at scale |
| Robustness | ★★★★☆ | 1 residual weak point (obsolescence = human discipline) |
| Determinism | ★★★★☆ | ~95% on reviewer with canonical phrasing |

---

## Appendix B — Planned improvements

### Known residual flaws

**Flaw 1 — Naming (MEDIUM risk)**

If the task uses a different term than the map name ("modify the CatalogSync client"
vs entry `product-sync-pipeline`), Claude may not make the connection and not consult
`summary.md`.

Current mitigation: list involved packages and classes in the map entry.
Possible improvement: add an `aliases:` field in the map entry format.

**Flaw 2 — Non-automatic reviewer (LOW risk)**

For complex tasks, Claude does not spawn the reviewer on its own initiative — the
developer must use the canonical phrasing. If the request is vague ("look at the docs
for X"), Claude may read files directly without spawning a subagent.

Current mitigation: 3 cumulative signals (system-reminder + SKILL.md + CLAUDE.md).
Possible improvement: add a `reviewer: required` field in the map entry for HIGH
mechanisms, which Claude could interpret as an automatic trigger.

**Flaw 3 — Silent obsolescence (HIGH risk if discipline is absent)**

A `summary.md` that no longer reflects the code is actively dangerous: Claude works
with unjustified confidence on a reality that no longer exists.

Current mitigation: "Keep documentation synchronized" section + warning signals.
Possible improvement: add in local CLAUDE.md files of HIGH risk modules a line
indicating the last update date of `summary.md`, visible every session:

```markdown
> ⚠️ Last summary.md update: 2026-01-15 — check if still up to date.
```

### Possible evolutions

| Evolution | Value | Complexity |
|-----------|--------|------------|
| `aliases` field in map entries | Resolves naming flaw | Low |
| `reviewer: required` field for HIGH risk | Makes reviewer semi-automatic | Medium |
| Update date in local CLAUDE.md | Detects silent obsolescence | Low |
| Periodic `memory-maintainer` audit with architecture scope | Detects non-existent classes | Already possible |
