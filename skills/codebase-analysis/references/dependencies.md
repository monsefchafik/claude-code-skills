# codebase-analysis — Dependencies

This document describes the dependencies of `codebase-analysis` on other tools in the project.

---

## Outgoing dependency: `/architecture-document`

### Nature of the dependency

`codebase-analysis` **does not depend** on `/architecture-document` to function.
The analysis can end with option 2 (file export) or option 3 (conversation only)
without ever invoking `/architecture-document`.

The dependency is **optional and declarative**: at the end of analysis, `codebase-analysis`
proposes to the user to run `/architecture-document` (option 1). It provides the pre-filled
command but does not execute it itself.

### What codebase-analysis provides to /architecture-document

When the user chooses option 1, the conversation contains:

| Content produced by codebase-analysis | Destination in /architecture-document |
|---------------------------------------|---------------------------------------|
| Findings from phases 1-5 | Primary source for `full-analysis.md` |
| Identified rules and invariants | `summary.md` candidates + PROMOTE-CANDIDATE |
| Observed and rejected alternatives | `decision.md` candidates |
| Scope (packages, classes) | `**Scope**` field of the map entry |
| Estimated risk (HIGH/MEDIUM/LOW) | `**Risk**` field of the map entry |
| SE type identified in Phase 0 | `type=` argument of the command |
| `## PROMOTE-CANDIDATE` in the exported file | Read **first** by `/architecture-document` Step 4 (via `doc=`) — not re-filtered |

### Pre-filled command provided

At the end of analysis (option 1 chosen), `codebase-analysis` produces:

```
→ Run: /architecture-document <name> type=<type>
```

The user copies it and runs it in the same conversation or in a new one.

### What codebase-analysis does NOT do

- Never invokes `/architecture-document` automatically
- Never writes to `docs/ai/architecture/`
- Never writes to `docs/ai/architecture-map.md`
- Never writes to `docs/ai/architecture-decisions.md`
- Never decides on behalf of the user

---

## Read dependency: `architecture-map.md`

In Phase 0, `codebase-analysis` reads `docs/ai/architecture-map.md` (if present) to:
- Identify already-documented units → exclude them from the plan by default
- Avoid duplicating an existing analysis
- Inform the user: "for already-documented units, use `architecture-reviewer`"

This read is **non-blocking** — if the file is absent, cartography continues.

---

## Read dependency: `analysis-plan-<repo>.md`

In Phase 0, `codebase-analysis` looks for a `docs/ai/analysis-plan-*.md` file to:
- Resume an in-progress analysis (large repo, multiple sessions)
- Display progress: units already analyzed vs remaining
- Avoid redoing cartography if the plan already exists
- Read `## Analysis details`: parameters stored per unit (objective, scope, freedom,
  context) — displayed before phases 1-5 start for confirmation or modification

This read is **non-blocking** — if the file is absent, Phase 0 starts from scratch.

---

## Read dependency: `.claude/memory-bank/systemPatterns.md`

In Phase 2 (Domain Model), `codebase-analysis` checks `systemPatterns.md` if present:
- Cross-references discovered patterns against already-documented patterns
- Avoids re-reporting what is already captured

---

## Read dependencies: PROMOTE-CANDIDATE destination files

At the output gate, `codebase-analysis` checks the existence of each destination file
referenced by its PROMOTE-CANDIDATE items. These files are not read for their content —
only to confirm their existence.

| File | Role in codebase-analysis | Degradation if absent |
|------|--------------------------|----------------------|
| `docs/ai/domain-glossary.md` | Destination for business terms identified in Phase 2 | Candidates flagged but promotion blocked — absent file flagged at output gate |
| `docs/ai/integration-patterns.md` | Destination for integration patterns identified in Phase 4 | Same |
| `docs/ai/data-sources.md` | Destination for data flows identified in Phase 4 | Same |
| `docs/ai/architecture-decisions.md` | Destination for cross-cutting decisions or already-documented mechanism | Same |
| `.claude/claude-files-registry.md` | Routing to local `CLAUDE.md` for subtree-specific invariants | Local CLAUDE.md candidates not routed — fallback to `.claude/rules/` |

These files are created by `/memory-bootstrap`. If absent: suggest to the user
to run `/memory-bootstrap` or create them manually before re-running `/promote-to-memory`.

---

## No-write contract

`codebase-analysis` does not write to any project file without explicit approval,
except the following exceptions:

| File | Condition | When | Automatic? |
|------|-----------|------|-----------|
| `docs/ai/analysis-plan-<repo>.md` | User answers "yes" in Phase 0 | After cartography | No — proposed |
| `docs/ai/analysis-plan-<repo>.md` — unit status | Existing plan | After output gate | Yes — silent |
| `docs/ai/analysis-plan-<repo>.md` — `## Analysis details` | Existing plan | After parameter confirmation (pre-analysis) | Yes — silent |
| `docs/ai/analysis-plan-<repo>.md` — `## Analysis details` | Existing plan | After user override (objective, scope, context) | Yes — silent |

If the plan does not exist when writing parameters, the skill offers to create it
rather than writing silently.

---

## Compatibility check

Before modifying `/architecture-document`, verify the impact on `codebase-analysis`:

1. Has the `type=` argument changed values? → Update Phase 0 and the output gate
2. Has the `architecture-map.md` entry format changed? → Update the Phase 0 read
3. Has the command name or syntax changed? → Update the pre-filled command in the output gate
4. Has the `## PROMOTE-CANDIDATE` format changed (confidence level, columns)?
   → Update Phase 3e (confidence level table) + `full-analysis.md` template in SKILL.md
   → Update `/promote-to-memory` and `memory-maintainer` if the parsing format changes

---

## See also

- [SKILL.md](../SKILL.md) — full codebase-analysis specification
- [user-manual.md](./user-manual.md) — usage guide
- `.claude/skills/architecture-document/SKILL.md` — /architecture-document specification
- `.claude/commands/references/promote-to-memory-dependencies.md` — PROMOTE-CANDIDATE pipeline
