# promote-to-memory — Dependencies

This document describes the incoming and outgoing dependencies of the
`/promote-to-memory` command with respect to the other tools in the project.

---

## Incoming dependency: `/architecture-document` and `codebase-analysis`

### The contract

Both skills produce `PROMOTE-CANDIDATE` items via two channels:

| Channel | Source | Persistence |
|---------|--------|-------------|
| **Conversation** | List displayed in Step 4 / output gate | Session only |
| **File** | `## PROMOTE-CANDIDATE` section in `full-analysis.md` or `<name>-analysis.md` | Persists across sessions |

`/promote-to-memory` supports both channels. The file channel (`doc=`) allows
promoting in a later session.

### Expected format in conversation (conversation channel)

```
PROMOTE-CANDIDATE RULES (to be promoted separately):
─────────────────────────────────────────────────────
- "<rule>" → .claude/rules/<file>.md
- "<pattern>" → systemPatterns.md
- "<risk>" → docs/ai/known-issues.md
```

### Expected format in a file (file channel)

```markdown
## PROMOTE-CANDIDATE
- "<rule>" → .claude/rules/<file>.md
- "<pattern>" → systemPatterns.md
(empty if no candidates identified)
```

### Processing rule

`PROMOTE-CANDIDATE` items are treated as **confirmed candidates**.

- `/promote-to-memory` gives them high detection priority
- The durable/non-durable filter is **not re-applied** — the skill has already
  performed that selection
- Items are routed directly to the destinations indicated in their label

### Deduplication rule

Before proposing a promotion, `/promote-to-memory` reads the target file and checks
whether a substantially similar rule already exists. If so: SKIP with reason.

**Why**: `/promote-to-memory` may be run multiple times on the same scope
(multiple sessions, re-runs). Without deduplication, `.claude/rules/` files
accumulate duplicates.

### Possible destinations

| Label in PROMOTE-CANDIDATE | Destination |
|---------------------------|-------------|
| `→ .claude/rules/<file>.md` | Implementation rule or invariant |
| `→ systemPatterns.md` | Confirmed recurring pattern in 2+ places |
| `→ docs/ai/known-issues.md` | Operational risk or known bug |

---

## Outgoing dependency: `memory-maintainer`

`/promote-to-memory` invokes `memory-maintainer` in patch mode. The contract with
`memory-maintainer` is described in `.claude/commands/references/memory-bootstrap-dependencies.md`.

---

## Recommended usage sequence

```
1. /architecture-document <mechanism>
   → 3 files written + 2 map/ADR patches
   → PROMOTE-CANDIDATE list displayed in conversation

2. Human validation of the 3 files (summary.md ≤ 100 lines?)

3. /promote-to-memory
   → Detects PROMOTE-CANDIDATEs first
   → Produces diffs toward .claude/rules/ + systemPatterns.md
   → Approval gate → write after confirmation

4. Wiring (if HIGH risk mechanism)
   → Create local CLAUDE.md with @summary.md
```

**Do not reverse steps 2 and 3**: validate the files before promoting the rules —
a rule derived from an incorrect `summary.md` will itself be incorrect.

---

## See also

- `.claude/commands/promote-to-memory.md` — command specification
- `.claude/skills/architecture-document/SKILL.md` — Step 4 (PROMOTE-CANDIDATE identification)
- `.claude/commands/references/memory-bootstrap-dependencies.md` — memory-bootstrap/memory-maintainer dependencies
- `docs/ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md` — Phase 4 of the complete workflow
