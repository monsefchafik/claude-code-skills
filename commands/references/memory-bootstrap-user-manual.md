# Memory Bootstrap — Usage Guide

For the full specification, see [memory-bootstrap.md](../memory-bootstrap.md).

## What this command does

`/memory-bootstrap` initializes the full Claude Code memory system for an existing repository.
It analyzes the codebase, proposes a file plan, waits for your confirmation, then generates all
memory layers populated with real project content.

Use it **once** at project start. After that, use `memory-maintainer` for all ongoing maintenance.

```
/memory-bootstrap                          # full bootstrap
/memory-bootstrap focus on the persistence layer    # scoped intent
```

## When to use it

| Situation | Use |
|-----------|-----|
| New project, no memory system yet | `/memory-bootstrap` |
| Existing project, memory files missing or sparse | `/memory-bootstrap` |
| Memory exists but is stale or inconsistent | `memory-maintainer` (audit mode) |
| Session just ended, want to save decisions | `/promote-to-memory` |

## The workflow at a glance

```
Step 1  Analyze repo        → reads pom.xml, git log, code files, existing memory
Step 2  Propose file plan   → lists every file to create or skip, with rationale
        ↓ YOU CONFIRM ↓
Step 3a Generate rules/docs → CLAUDE.md, docs/ai/, .claude/rules/
Step 3b Generate memory-bank → activeContext, progress, systemPatterns (with real content)
Step 4  Self-audit          → removes filler, fixes routing errors
Step 5  Human checklist     → what only you can fill in
Step 6  Validation tasks    → 3–5 tasks to test the memory system
Step 7  Maintenance plan    → how to keep it healthy going forward
```

The command **stops after Step 2 and asks for confirmation** before writing any file.

## Confirming the file plan

After Step 2, you will see a proposed file list. You can:

```
yes                          # generate everything as proposed
no                           # cancel
adjust: skip docs/ai/runbook-dev.md, rename known-issues to operational-issues
```

Only files you confirm will be generated.

## What gets populated vs left empty

Bootstrap populates memory-bank files with **real project content** — this is its unique value
over `memory-maintainer`, which only creates empty templates.

| File | What bootstrap fills in |
|------|------------------------|
| `progress.md` | Recent milestones from `git log` (last 5–10 significant commits) |
| `activeContext.md` | Active work from recent branches; known risks from existing docs |
| `systemPatterns.md` | Patterns confirmed in 2+ files (mapping, errors, transactions, saveAll) |
| `.claude/claude-files-registry.md` | All local `CLAUDE.md` files discovered + routing criteria from codebase analysis |

If a section has no grounded content, it is marked `<!-- To be completed -->` — never invented.

The registry is bootstrap's **exclusive responsibility** for initialization. `memory-maintainer`
keeps it in sync (adds missing paths, removes stale ones) but never writes routing criteria.

## After bootstrap: handoff to memory-maintainer

Bootstrap runs once. After that:

| When | Command |
|------|---------|
| After each significant session | `/promote-to-memory` |
| After a PR is merged | `memory-maintainer` — rotate activeContext → progress |
| Every 2–4 weeks | `memory-maintainer` — full audit |
| Starting a new workstream | `memory-maintainer` — update activeContext |

## Iterating after generation

If the generated content needs adjustment, treat it like code — refine specific parts rather
than re-running the full bootstrap:

```
the systemPatterns.md section on error handling is too generic, rewrite it based on ConsistencyError
```

```
move the saveAll rule from docs/ai/ to .claude/rules/database.md
```

```
activeContext.md is too long — prune it to the 3 most active items
```

The command will adjust the specific section without regenerating everything.

## Common mistakes to avoid

| Mistake | What to do instead |
|---------|-------------------|
| Running bootstrap again to fix a stale memory | Use `memory-maintainer` in patch mode |
| Adding `projectbrief.md` or `techContext.md` to memory-bank | Only 3 standard files: `activeContext`, `progress`, `systemPatterns` |
| Accepting a `docs/ai/` file that duplicates an existing one | Check Step 2 carefully — reject conflicting files |
| Skipping the confirmation step | Always review Step 2 before proceeding |

## See also

- [memory-bootstrap.md](../memory-bootstrap.md) — full specification
- [memory-maintainer SKILL.md](../../skills/memory-maintainer/SKILL.md) — ongoing maintenance
- [promotion-matrix.md](../../skills/memory-maintainer/references/promotion-matrix.md) — where each type of information belongs
- [MEMORY_ARCHITECTURE_EN.md](../../../docs/MEMORY_ARCHITECTURE_EN.md) — architecture overview and layer responsibilities
