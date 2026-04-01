# memory-bootstrap — Dependencies on memory-maintainer

For the authoritative file tree and internal structure of every memory file, see:
[MEMORY_ARCHITECTURE_EN.md — Structure Contract](../../../docs/MEMORY_ARCHITECTURE_EN.md#structure-contract)

This document lists every dependency that `memory-bootstrap` has on `memory-maintainer`.
Think of it as a Java `import` declaration: if any of these contracts change in `memory-maintainer`,
`memory-bootstrap` must be reviewed and updated accordingly.

---

## Required files (hard dependencies)

These files must exist for the command to function correctly. If absent, the indicated step degrades.

| File | Used in | Purpose | Degradation if absent |
|------|---------|---------|----------------------|
| `.claude/skills/memory-maintainer/references/templates/activeContext.md` | Step 3c | Section structure for `activeContext.md` | Bootstrap invents its own structure → incompatible with memory-maintainer lifecycle |
| `.claude/skills/memory-maintainer/references/templates/progress.md` | Step 3c | Section structure for `progress.md` | Bootstrap invents its own structure → maintainer may not recognize the format |
| `.claude/skills/memory-maintainer/references/templates/systemPatterns.md` | Step 3c | Section structure for `systemPatterns.md` | Bootstrap invents its own structure → sections may not match maintainer expectations |
| `.claude/skills/memory-maintainer/references/templates/claude-files-registry.md` | Step 3b | Canonical structure for the local CLAUDE.md registry | Bootstrap generates a non-standard format → memory-maintainer cannot reliably sync it |
| `.claude/skills/memory-maintainer/references/promotion-matrix.md` | Step 4 | Routing grid for the critical audit | Step 4 falls back to generic checks — audit quality degrades |

---

## Structural contracts (behavioral dependencies)

These are conventions defined in `memory-maintainer` that `memory-bootstrap` must respect to produce
compatible output. A change in any of these in `memory-maintainer` silently breaks bootstrap output.

### memory-bank file set

```
Contract: exactly three standard files
  activeContext.md
  progress.md
  systemPatterns.md

Owner: memory-maintainer SKILL.md § Bootstrap behavior
Impact: bootstrap must not create any other file in .claude/memory-bank/
Break condition: if memory-maintainer adds or renames a standard file
```

### activeContext lifecycle

```
Contract:
  - max ~15 lines
  - entries expire when work is done → rotate to progress.md
  - structure: ## Current Focus / ## Active Risks / ## Pending Decisions

Owner: memory-maintainer SKILL.md § activeContext lifecycle
Impact: bootstrap must not exceed 15 lines in activeContext.md at generation time
Break condition: if memory-maintainer changes the section names or the size constraint
```

### progress.md append-only rule

```
Contract:
  - append-only log, never edit existing entries
  - one entry per milestone with date and reference (PR, commit, ticket)
  - table format: | Date | Milestone | Reference |

Owner: memory-maintainer SKILL.md § Promotion rules + promotion-matrix.md
Impact: bootstrap populates the table using git log — must match this exact format
Break condition: if memory-maintainer changes the table format or append-only rule
```

### systemPatterns.md two-occurrence rule

```
Contract:
  - promote a pattern only when it appears in 2+ distinct files
  - format: pattern name, when to use it, minimal code example

Owner: memory-maintainer SKILL.md § systemPatterns population
Impact: bootstrap applies this rule in Step 1.5 to determine candidates
Break condition: if memory-maintainer changes the threshold or the entry format
```

### Memory layer routing model

```
Contract:
  CLAUDE.md / local CLAUDE.md  → prescriptive rules
  .claude/rules/               → specialized reusable rules
  docs/ai/                     → explanatory knowledge
  .claude/memory-bank/         → evolving project context

Owner: memory-maintainer SKILL.md § Memory layers and responsibilities
Impact: bootstrap uses this routing in Step 2 (file plan) and Step 4 (audit)
Break condition: if memory-maintainer reassigns a layer or adds a new one
```

### Local CLAUDE.md registry

```
Contract:
  File: .claude/claude-files-registry.md
  Structure: markdown table with columns | File | Routes here when |
  One row per local CLAUDE.md file in the repository.

  Ownership:
    - bootstrap CREATES and INITIALIZES routing criteria (from Step 1 analysis)
    - memory-maintainer MAINTAINS paths (adds missing, removes stale) but never writes criteria
    - human may update routing criteria manually if the module scope changes

Owner: memory-bootstrap.md § Step 3b + memory-maintainer SKILL.md § Local CLAUDE.md discovery
Impact: bootstrap must generate this file for memory-maintainer to route subtree-specific rules
Break condition: if the file location, table format, or column names change
```

### Three-tier loading strategy

```
Contract:
  Tier 1 — @imports in root CLAUDE.md (always loaded):
    @.claude/memory-bank/activeContext.md
    @.claude/memory-bank/systemPatterns.md
    @docs/ai/known-issues.md

  Tier 2 — On-demand references listed in root CLAUDE.md (not imported):
    .claude/memory-bank/progress.md
    docs/ai/architecture-decisions.md
    docs/ai/domain-glossary.md
    docs/ai/integration-patterns.md
    docs/ai/data-sources.md

  Tier 3 — Never imported directly (accessed via architecture-reviewer subagent):
    docs/ai/generators-architecture.md
    docs/ai/validation-pipelines.md
    any full-analysis or summary architecture file

Owner: memory-maintainer SKILL.md § Import hygiene
Impact: bootstrap must generate root CLAUDE.md with exactly this structure.
        Tier 1 @imports only added if the file exists at generation time.
        Tier 2 entries only listed if the file exists at generation time.
Break condition: if memory-maintainer changes the tier file lists or adds/removes a tier
```

---

## Patch ID convention (handoff dependency)

When `memory-maintainer` runs after bootstrap and detects missing memory-bank files, it generates
patches labeled `P-init-1`, `P-init-2`, `P-init-3`. Bootstrap documents this convention in Step 7
(maintenance guidance) so the user knows what to expect on the first maintainer run.

```
Contract: P-init-1 → activeContext.md, P-init-2 → progress.md, P-init-3 → systemPatterns.md
Owner: memory-maintainer SKILL.md § Bootstrap behavior
Break condition: if memory-maintainer changes the patch ID naming scheme
```

---

## What memory-bootstrap does NOT depend on

| Element | Why excluded |
|---------|-------------|
| `memory-maintainer` audit/patch/apply workflow | Bootstrap does not invoke the skill — it runs independently |
| `report-examples.md` | Bootstrap has its own output format |
| `user-manual.md` (memory-maintainer) | Documentation only, no behavioral dependency |
| Auto-memory (`~/.claude/projects/...`) | Bootstrap reads it as optional input, never required |

---

## Compatibility check

Before modifying `memory-maintainer`, verify impact on `memory-bootstrap` by checking:

1. Did the template section names change? → Update Step 3b population instructions
2. Did the standard memory-bank file set change? → Update target architecture section
3. Did the routing model change? → Update Step 2 file plan and Step 4 audit criteria
4. Did the activeContext size or lifecycle rule change? → Update Step 3b constraint
5. Did the progress.md table format change? → Update Step 3b population for progress
6. Did the two-occurrence rule threshold change? → Update Step 1.5 promotion criteria
7. Did the three-tier loading strategy change (tier file lists, new tier, removed tier)? → Update Step 3a loading strategy block in bootstrap
8. Did the registry file location, table format, or ownership rules change? → Update Step 3b (registry) in bootstrap and § Local CLAUDE.md discovery in memory-maintainer
