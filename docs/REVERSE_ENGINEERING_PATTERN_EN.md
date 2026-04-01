# Reverse Engineering Pattern

This document describes the workflow for reverse engineering an existing codebase, the design
decisions behind `codebase-analysis`, and how it articulates with the architecture documentation
system.

It is the counterpart of [ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md](ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md)
for the acquisition phase — the phase that precedes documentation.

---

## Why this pattern?

### The problem

Understanding an unknown module before modifying it takes time and produces ephemeral knowledge:
it stays in the developer's head or in the conversation, then disappears.

The next session starts from scratch. Edge cases are rediscovered. The same mistakes are made.

### The solution

```
Separate knowledge acquisition from knowledge capture.
Adapt analysis depth to the size of the scope.
Persist the analysis plan between sessions.
Let the user decide what to memorize.
```

**Core principle**:
> Analyze = explore and understand.
> Document = decide what deserves to be remembered.
> These two acts are distinct and must not be forced together.

---

## Where `codebase-analysis` sits in the ecosystem

```
codebase-analysis          →  /architecture-document     →  /promote-to-memory
(explore and understand)      (capture in memory)            (promote rules)
        ↓                              ↓                              ↓
  analysis-plan-*.md        docs/ai/architecture/<name>/      .claude/rules/
  (session traceability)    full-analysis + summary + decision  systemPatterns.md
        ↓                      (## PROMOTE-CANDIDATE included)
  doc=<file> (option 2)              ↓
  ## PROMOTE-CANDIDATE       architecture-map.md
  (in the file,                      ↓
   with confidence level
   high | medium | low)
        ↑                   architecture-reviewer
        └── /codebase-analysis doc=  (leverage in future tasks)
            (targeted deepening)
        ↓
  /promote-to-memory doc=<file>
  (later session)
```

`codebase-analysis` is **Phase 0 of the architecture documentation workflow**. Without it,
`/architecture-document` can be launched directly from a manual exploratory session.
With it, the exploration is structured, traced, and reproducible.

---

## Which tool to use?

| Situation | Tool |
|-----------|-------|
| Unknown unit, never analyzed | `codebase-analysis` |
| Unit already in `architecture-map.md` | `architecture-reviewer` |
| Free exploratory session, no structure | Direct conversation + `/architecture-document` |
| Convert an existing analysis document | `/architecture-document doc=<path>` |
| Understand an existing rule or constraint | `.claude/rules/` + `systemPatterns.md` |

**Routing rule**: if the unit is in `architecture-map.md`, do not relaunch
`codebase-analysis` — the analysis has already been done. Use `architecture-reviewer`
for a targeted task, or reread `full-analysis.md` for an in-depth review.

---

## Skill design decisions

### Why the output gate exists

`codebase-analysis` does not know what the user wants to do with the results.
Sometimes you analyze to understand, not to document. Imposing `/architecture-document`
at the end would be an anti-pattern — it would force a memorization decision on an
exploration that may not deserve one.

The output gate preserves separation of concerns:
- The skill analyzes
- The user decides
- `/architecture-document` memorizes if requested

### Why 1 conversation per unit on large repos

Claude context accumulates. Analyzing 4 units in the same conversation means
the 4th analysis session runs with the first 3 still present in context —
even compacted, they generate noise and reduce precision.

One conversation per unit guarantees a clean, focused context. Results from
previous sessions are accessible via `architecture-map.md` (if option 1 was chosen)
— not via conversation context.

### Why the plan is an external file

Claude memory does not persist between conversations. A plan stored only in the
conversation disappears when it closes. The `analysis-plan-<repo>.md` file is the
only reliable source of truth between sessions — it is versionable, human-readable,
and queryable by Claude without prior memory.

### Why the review loop precedes the output gate

Phases 1-5 produce a raw analysis — Claude reads the code but does not know
whether the user wants to deepen an aspect, correct an interpretation, or move
directly to capture.

Presenting the analysis in the conversation before proposing the output gate allows:
- correcting interpretation errors before they are persisted
- deepening a targeted aspect without redoing the entire analysis
- validating that quality is sufficient for memorization

**The rule**: the output gate is never proposed without explicit user approval.

### Why the `doc=` mode (iterative refinement)

A complete analysis of a complex unit may require multiple sessions.
The `doc=` mode allows resuming an exported analysis (option 2) in a new
conversation, targeting missing areas, then updating the same file.

This pattern decouples two independent decisions:
- When is the analysis complete enough? (progressive decision)
- When does it deserve to be memorized? (capture decision)

Without `doc=`, you would either have to redo everything, or choose option 1 prematurely.

### Why analysis parameters are persisted in the plan

The analysis objective (debug, modify, understand, refactor, document) determines
which phases are priority and at what depth. The scope and known context determine
what Claude looks for first.

Without persistence, a new session starts from scratch: Claude asks questions again,
the user re-explains, and there is a strong risk that the resumption is not calibrated
the same way — especially if several days separate the sessions.

The plan file stores these parameters in `## Analysis Details`:
scope, freedom (exclusive / indicative), objective, known context.

At the start of the next session, Claude displays the stored parameters and asks
"Confirm or modify?" — which takes 5 seconds. No information is lost,
and the user can adjust if context has changed (new decision, bug identified
in the meantime, revised objective).

The user can also modify these parameters **at any time** during the analysis —
before, during or after the phases — and changes are immediately written to the
plan and applied to remaining phases.

### Why `🚫 ignored` is a persistent status

Some units do not deserve analysis (thin layers, config boilerplate, generated code).
Without a persistent status, the skill proposes them again every session. `🚫 ignored`
is an explicit and durable decision — it expresses "we chose not to document this",
which is itself architectural information.

---

## PROMOTE-CANDIDATE — routing and confidence level

Each PROMOTE-CANDIDATE identified during analysis carries a mandatory destination and
confidence level. The level is determined by the detection mechanism, not by the business
importance of the item.

### Complete routing table

| Detected type | Phase | Destination | Typical confidence |
|---|---|---|---|
| Repo-wide invariant (extracted from a `throw` or grep across 2+ files) | 3 | `.claude/rules/` | `high` |
| Subtree-specific invariant (single module) | 3 | Local `CLAUDE.md` (via registry) | `high` |
| Business policy tied to the analyzed mechanism | 3 | `decision.md` (via `/architecture-document`) | `medium` |
| Cross-cutting decision or already documented mechanism | 3, 4 | `docs/ai/architecture-decisions.md` | `medium` |
| Operational constraint (DB, infra, external system) | 3 | `docs/ai/known-issues.md` | `high` or `medium` |
| Recurring pattern (2+ distinct files) | 2, 3 | `.claude/memory-bank/systemPatterns.md` | `high` |
| Business term (vocabulary with no equivalent in code) | 2 | `docs/ai/domain-glossary.md` | `medium` |
| Shared integration pattern (2+ units) | 4 | `docs/ai/integration-patterns.md` | `medium` |
| Data flow (source, transformation, target) | 4 | `docs/ai/data-sources.md` | `medium` |
| Implicit rule `[IMPLICIT]` (suspicion from a comment) | 3 | Nearest destination | `low` |

### Confidence levels

| Level | Criterion |
|--------|---------|
| `high` | Extracted from a `throw`/exception, or grep-confirmed in 2+ distinct files, or coverage gap found by grep |
| `medium` | Consistent observation but Claude's judgment (business terms, heuristic patterns, integration seen on a single unit) |
| `low` | `[IMPLICIT]` — suspicion from a comment, tacit convention, or indirect evidence |

**Format**: each entry in `## PROMOTE-CANDIDATE` must have the form:
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

**Consequence for promotion**: `/promote-to-memory` and `memory-maintainer` see the confidence
level persisted in `full-analysis.md`. `[low]` items must be verified in the code before the
patch is applied — they must not be treated as established facts.

---

## The complete workflow

### Small repo (< 5k lines)

```
Single session:
  /codebase-analysis
    Phase 0: mapping → confirmation
    Phases 1-5: analysis in one pass
    Review loop → "good to go"
    Output gate → option 1 recommended
    /architecture-document <name> type=<type>
    /promote-to-memory
```

### Medium repo (5k–30k lines)

```
One or two sessions:
  /codebase-analysis
    Phase 0: mapping → plan saved
    Phases 1-5 by layer (domain → services → adapters)
    Review loop per unit → validation → output gate
    Output gate per unit → /architecture-document for each
    /promote-to-memory at end of session
```

### Large repo (> 30k lines)

```
Conversation 1:
  /codebase-analysis
    Phase 0: global mapping → plan saved
    Analyze unit A [HIGH]
    Review loop → "ok"
    Output gate → option 1 → /architecture-document unit-a type=<type>
    Plan updated: unit-a ✅

Conversation 2:
  /codebase-analysis  (plan detected automatically)
    Resumes with unit B [HIGH]
    → Displays stored parameters (objective, scope, context) → "Confirm or modify?"
    Phases 1-5 calibrated according to objective
    Review loop → "ok"
    Output gate → option 1 → /architecture-document unit-b type=<type>
    Plan updated: unit-b ✅

...

Last conversation:
  Cross-cutting pass (transactions, security, end-to-end flows)
  /architecture-document cross-cutting-concerns type=subsystem (if warranted)
  /promote-to-memory global
```

### Iterative analysis across multiple sessions (via doc=)

For complex units requiring multiple passes before capture:

```
Session 1 — initial exploration:
  /codebase-analysis
    Phases 1-5 → review loop → output gate option 2
    → docs/notes/my-unit-analysis.md created
      (includes ## PROMOTE-CANDIDATE — persistent between sessions)

Session 2 — deepening:
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
    → Coverage summary (areas ✅ / ⚠️ / ❌)
    → Targeted deepening on missing areas
    → Review loop → output gate option 2
    → docs/notes/my-unit-analysis.md updated

Session 3 — capture:
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
    → "continue" → review loop → "good to go"
    → Output gate option 1
    → /architecture-document my-unit type=<type>
```

This pattern is suited when the complete analysis exceeds the scope of one conversation,
or when quality must be validated progressively before memorization.

---

## What the system provides once analysis is complete

Once a unit has been analyzed AND documented (option 1):

| Enabled capability | Via |
|-----------------|-----|
| Claude knows the invariants before any modification | `architecture-map.md` @imported |
| Safe modification without re-explaining context | `summary.md` read on trigger |
| Traceable architectural decisions | `decision.md` + ADR |
| Implementation rules active every session | `.claude/rules/` |
| In-depth analysis before complex task | `architecture-reviewer` |

If option 2 or 3 was chosen, the knowledge is not integrated into the system —
it stays in the exported artifacts or disappears. This is a valid choice for a
one-off analysis, but without the benefits above.

---

## ⚠️ Keep the plan synchronized

The `analysis-plan-<repo>.md` file is updated automatically after each output gate.
Two situations require manual intervention:

**1. A unit has been refactored and renamed**
→ Update the name in the plan AND in `architecture-map.md`

**2. A `✅ documented` unit has been significantly modified**
→ Reset to `⬜ to do` in the plan
→ Relaunch `codebase-analysis` on that unit
→ Relaunch `/architecture-document` to regenerate

---

## Warning signals

| Signal | Cause | Action |
|--------|-------|--------|
| Plan absent on a large repo | Sessions cannot be resumed | Create the plan retroactively |
| Stored parameters become obsolete (objective has changed) | Decision made between sessions | Say "Change the objective for <name> to <new>" — plan updated immediately |
| All units in `💬 conversation only` | Analysis without capture | Relaunch with option 1 for HIGH/MEDIUM units |
| PROMOTE-CANDIDATE not promoted, session closed | Rules lost if option 3 | Use option 2 → `/promote-to-memory doc=<file>` in next session |
| PROMOTE-CANDIDATE `[low]` promoted without verification | Suspicion treated as established fact | Verify code before applying the patch in memory-maintainer |
| `architecture-map.md` empty despite analyses | Option 1 never chosen | No active memory — system not serving its purpose |
| Unit in plan but no longer in the code | Untracked refactoring | Mark `🚫 ignored` + update `architecture-map.md` |
| Plan with all units `🚫 ignored` | Scope too broad defined in Phase 0 | Redo the mapping with a reduced scope |

---

## See also

- `.claude/skills/codebase-analysis/SKILL.md` — complete specification
- `.claude/skills/codebase-analysis/references/user-manual.md` — usage guide
- [ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md](ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md) — what follows the analysis
- `.claude/skills/architecture-reviewer/references/user-manual.md` — leveraging the produced docs
