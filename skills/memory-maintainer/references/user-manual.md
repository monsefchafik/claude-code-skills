# Memory Maintainer — Usage Guide

For the full specification, see [SKILL.md](../SKILL.md).

## Quick start

```text
mode: audit
scope: full
include_auto_memory: true
notes: Full memory audit with auto-memory promotions.
```

## Modes at a glance

| Mode | What you get | When to use |
|------|-------------|-------------|
| `audit` | Four-block report with patch plan (no diffs) | Understand the state before acting |
| `patch` | Four-block report with full unified diffs | Ready to review and approve changes |
| `apply` | Changes written to files | After approving the patch set |

Always follow the sequence: **audit → patch → apply**. Never skip to apply without review.

## Common invocations

**Full audit with auto-memory:**
```text
mode: audit
scope: full
include_auto_memory: true
```

**Targeted patch on one file:**
```text
mode: patch
scope: targeted
paths:
- .claude/memory-bank/activeContext.md
notes: Rotate completed items to progress.md.
```

**Apply one patch:**
```text
mode: apply
apply_target: patch:P2
```

**Apply one file:**
```text
mode: apply
apply_target: file:docs/ai/known-issues.md
```

## Promoting from the current conversation

You can ask the skill to promote something said during the current session:

```text
promote the decision we just made about sorted saveAll
```

```text
remember that the CatalogSync client has a 30-second timeout — add it to known-issues
```

```text
mode: patch
notes: Promote the patterns we confirmed during this session.
```

The skill will extract the durable content, determine the correct destination, and present it
in Block 3 labeled `Source: conversation`.

## Iterating on a proposal

After an audit or patch response, refine conversationally before applying:

```text
I don't agree with P3. Remove it from the plan.
```

```text
Keep the audit, but rewrite the patch for activeContext.md to be shorter.
```

```text
Apply only P1 and P4.
```

The skill updates the plan first, then waits for explicit apply instructions.

## Local CLAUDE.md files — discovery and routing

At every `audit` or `patch` invocation, the skill automatically:

1. Reads `.claude/claude-files-registry.md` (if present) to discover local `CLAUDE.md` files
2. Globs for actual local `CLAUDE.md` files and syncs against the registry:
   - File exists but not in registry → flagged in Block 2, path addition proposed
   - Registry entry references a missing file → removal proposed
3. Uses the routing criteria in the registry to dispatch subtree-specific rules to the correct local `CLAUDE.md`

**If the registry is absent**: the skill flags it in Block 2 and cannot route to local `CLAUDE.md` files reliably. Run `/memory-bootstrap` to create it, or create it manually using the format in `.claude/claude-files-registry.md`.

The skill never writes routing criteria — only `memory-bootstrap` or a human initializes those.

## What the skill cannot detect automatically

The skill has write access to all versioned memory files. The bottleneck is **detection, not
access**. These files can drift silently — the skill will only correct them if the stale content
surfaces in the current conversation or auto-memory:

- `CLAUDE.md` (root) and `.claude/rules/*.md` — rules referencing renamed or deleted code
- `docs/ai/known-issues.md` — issues documented as active but already fixed
- `docs/ai/architecture-decisions.md`, `domain-glossary.md`, `integration-patterns.md`, `data-sources.md` — any content that changed without going through a session

**Practical consequence**: the 2–4 week full audit is not optional — it is the only forcing
function for these files. Between audits, run `/promote-to-memory` after each significant session
to minimize the drift window.

## See also

- [promotion-matrix.md](promotion-matrix.md) — where does each type of information belong?
- [report-examples.md](report-examples.md) — example responses for audit, patch, and apply
- [SKILL.md](../SKILL.md) — full specification including lifecycle rules and verification steps
