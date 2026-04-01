# memory-maintainer — Dependencies

This document lists every dependency that `memory-maintainer` has on other tools, files, and
structural contracts. Think of it as the counterpart to `memory-bootstrap-dependencies.md`:
if any of these contracts change, `memory-maintainer` must be reviewed and updated accordingly.

---

## Required files (hard dependencies)

These files must exist for the skill to function correctly. If absent, the indicated step degrades.

| File | Used in | Purpose | Degradation if absent |
|------|---------|---------|----------------------|
| `references/templates/activeContext.md` | Bootstrap behavior | Structure reference when creating missing memory-bank file | Skill invents its own structure → incompatible with bootstrap output |
| `references/templates/progress.md` | Bootstrap behavior | Structure reference when creating missing memory-bank file | Same |
| `references/templates/systemPatterns.md` | Bootstrap behavior | Structure reference when creating missing memory-bank file | Same |
| `references/templates/claude-files-registry.md` | Local CLAUDE.md discovery | Canonical structure reference for auditing the registry format | Cannot detect format drift in the registry |
| `references/promotion-matrix.md` | Routing decisions | Routing decision table by information type | Routing falls back to generic heuristics — quality degrades |

---

## Optional files (soft dependencies)

These files improve skill behavior when present. Absence degrades specific capabilities.

| File | Used in | Purpose | Degradation if absent |
|------|---------|---------|----------------------|
| `.claude/claude-files-registry.md` | Local CLAUDE.md discovery | Discover local `CLAUDE.md` files and routing criteria | Cannot route subtree-specific rules to local `CLAUDE.md` files; flags absence in Block 2 |
| `CLAUDE.md` (root) | Import hygiene | Verify Tier 1 @imports and Tier 2 on-demand references are present | Import hygiene check skipped |

---

## Structural contracts (behavioral dependencies)

These are conventions defined or initialized by other tools that `memory-maintainer` must respect.
A change in any of these silently breaks skill behavior.

### Local CLAUDE.md registry format

```
Contract:
  File: .claude/claude-files-registry.md
  Structure: markdown table | File | Routes here when |
  Owner: memory-bootstrap (creates and initializes routing criteria)
  memory-maintainer role: sync paths only — never write routing criteria

Break condition: if the file location or table column names change
```

### Three-tier loading strategy

```
Contract:
  Tier 1 — @imports in root CLAUDE.md:
    @.claude/memory-bank/activeContext.md
    @.claude/memory-bank/systemPatterns.md
    @docs/ai/known-issues.md

  Tier 2 — On-demand references listed in root CLAUDE.md (not imported)
  Tier 3 — Never imported directly (architecture docs via architecture-reviewer)

Owner: memory-bootstrap (initializes in root CLAUDE.md) + memory-maintainer (enforces via import hygiene)
Break condition: if tier file lists change → update § Import hygiene in SKILL.md
```

### memory-bank standard file set

```
Contract: exactly three standard files in .claude/memory-bank/
  activeContext.md, progress.md, systemPatterns.md

Owner: memory-maintainer SKILL.md § Bootstrap behavior
Break condition: if bootstrap adds or renames a standard file
```

### activeContext size constraint

```
Contract: ~15 lines maximum
Owner: memory-maintainer SKILL.md § activeContext lifecycle
Break condition: if the size limit changes → update the lifecycle check
```

---

## What memory-maintainer does NOT depend on

| Element | Why excluded |
|---------|-------------|
| `memory-bootstrap.md` | Memory-maintainer does not invoke bootstrap — they are sequential, not nested |
| `memory-bootstrap-dependencies.md` | Documentation only, no behavioral dependency |
| `report-examples.md` | Self-referential — examples of this skill's own output |
| `user-manual.md` | Documentation only |
| Auto-memory (`~/.claude/projects/...`) | Read as optional input, never required |

---

## Compatibility check

Before modifying contracts that `memory-maintainer` depends on, verify impact:

1. Did the registry file location or column format change? → Update § Local CLAUDE.md discovery in SKILL.md
2. Did the three-tier loading strategy change? → Update § Import hygiene in SKILL.md
3. Did the memory-bank standard file set change? → Update § Bootstrap behavior in SKILL.md
4. Did the activeContext size constraint change? → Update § activeContext lifecycle in SKILL.md
5. Did a template section name change? → Update § Bootstrap behavior (file creation step)
