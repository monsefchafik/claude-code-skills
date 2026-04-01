Invoke the memory-maintainer skill in patch mode focused exclusively on the current conversation
and/or a source file provided via `doc=`.

## Arguments

- `doc=<path>` (optional): path to a `full-analysis.md` or `<name>-analysis.md` file containing
  a `## PROMOTE-CANDIDATE` section. Use this when promoting from a previous session where
  `/architecture-document` or `codebase-analysis` was run but `/promote-to-memory` was not.

## Step 1 — Collect candidates

### Source A — File (if `doc=` provided)

Read the file at `<path>`. Extract items from the `## PROMOTE-CANDIDATE` section.
These are confirmed candidates — do not re-filter for durability.
If the section is absent or empty, note it and continue to Source B.

### Source B — Conversation

Scan the full conversation above for any of the following:
- items explicitly labeled PROMOTE-CANDIDATE (produced by /architecture-document or
  /codebase-analysis) — treat as confirmed candidates, do not re-filter for durability
- architectural decisions made or confirmed
- implementation patterns observed in 2+ places or explicitly validated
- known issues discovered or reproduced
- rules the team agreed on
- completed milestones worth logging

For each candidate from Source B, apply the durable/not-durable filter strictly:
- PROMOTE: explicit decisions, confirmed patterns, established facts
- SKIP: speculation, one-off observations, conversational filler

### Merge

Merge candidates from both sources. Deduplicate: if the same rule appears in both,
keep it once (file source takes precedence for destination mapping).

## Step 2 — Deduplication against existing memory

Before proposing any promotion, read the target file for each candidate and check
whether a substantially similar rule already exists:

```bash
# For each candidate targeting .claude/rules/<file>.md
cat .claude/rules/<file>.md 2>/dev/null

# For systemPatterns.md
cat .claude/memory-bank/systemPatterns.md 2>/dev/null
```

If a similar rule already exists in the target file:
- Mark the candidate as SKIP with reason "already present in <file> (line ~N)"
- Do not propose it for promotion

"Substantially similar" means: same constraint, same table/class, or same pattern —
even if worded differently. When in doubt, flag as potential duplicate rather than
silently skipping.

## Step 3 — Produce the patch report

Produce the four-block report in patch mode:
- Block 1: skip (no need to summarize versioned memory state)
- Block 2: skip (not a maintenance audit)
- Block 3: list every promotion candidate with Source (file or conversation), destination
  file, rationale, confidence — include SKIP items with reason
- Block 4: full unified diffs, ready to apply (only for non-skipped items)

If nothing qualifies, say so in one line. Do not invent promotions.

## Step 4 — Approval gate

After producing the diffs, ask the user:
"Apply these promotions? (yes / no / select numbers)"

Only write files if the user explicitly confirms. Never write without approval.
