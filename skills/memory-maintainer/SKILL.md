---
name: memory-maintainer
description: maintain and improve claude code project memory so it stays concise, consistent, unambiguous, and non-contradictory. use when the user wants to audit, refine, patch, or apply changes to project memory files such as claude.md, local claude.md files, .claude/rules, docs/ai, or .claude/memory-bank. also use when the user wants to promote durable learnings from visible auto-memory, /memory output, or the current conversation into the correct versioned memory layer. also use when the user says "promote this", "remember this", or "remember what we just decided". also use to initialize the memory structure on a new project.
---

# Memory Maintainer

Maintain the repository's versioned Claude Code memory with a disciplined, low-noise workflow.

## Inputs

Prefer this semi-structured input format, with optional free text after it:

```text
mode: audit | patch | apply
scope: full | targeted
include_auto_memory: true | false
apply_target: all | patch:<id> | file:<path>
paths:
- optional/path
- optional/path
notes: optional free-text goal
```

Interpret missing fields sensibly:
- default `mode` to `audit`
- default `scope` to `full`
- default `include_auto_memory` to `true` when the user mentions `/memory`, auto-memory, or promotion
- default `apply_target` to `all` only when the user explicitly asks to apply patches

If the user writes free-form requests such as "instead of this memory part..." or "remove it from the plan", treat them as refinements to the latest audit/patch set.

## Memory layers and responsibilities

Use this routing model consistently:

- `CLAUDE.md` root → global, durable, prescriptive repo-wide rules
- local `CLAUDE.md` → directory-specific durable rules tied to that subtree
- `.claude/rules/` → specialized reusable rules, especially cross-cutting or task-specific guidance
- `docs/ai/` → durable explanatory knowledge: architecture, domain, business rules, runbooks, known issues, decisions
- `.claude/memory-bank/` → evolving project context: active work, completed milestones, recurring patterns

**Before routing to `.claude/memory-bank/`, always inspect the actual filenames present in that directory.**
Do not invent new filenames. Route to files that exist. If the project uses the standard structure, expect:
- `activeContext.md` → current work, active risks, pending decisions (temporary, expires when work is done)
- `progress.md` → append-only milestone log (permanent record of completed work)
- `systemPatterns.md` → recurring implementation patterns confirmed in 2+ places in the codebase

## Promotion rules

Promote information only when it is durable and belongs in versioned memory.

Promote to:
- `CLAUDE.md` when it is a stable instruction about how Claude should act
- local `CLAUDE.md` when the rule is only true within a subtree or module
- `.claude/rules/` when it is a reusable specialized rule that should not bloat a `CLAUDE.md`
- `docs/ai/known-issues.md` when it is a recurring operational or debugging pitfall
- `docs/ai/architecture-decisions.md` (or equivalent ADR file) when it is a durable architectural decision or trade-off
- `docs/ai/` (appropriate existing file) for architecture boundaries, domain knowledge, or runbooks
- `.claude/memory-bank/activeContext.md` when it describes current work, near-term risks, or pending decisions — and is expected to expire when the work is done
- `.claude/memory-bank/progress.md` when it is a completed milestone or delivered change
- `.claude/memory-bank/systemPatterns.md` when the same implementation approach appears in 2 or more distinct files and is not yet documented (two-occurrence rule)

Do not promote:
- one-off observations
- temporary frustrations
- redundant restatements of existing rules
- speculative business rules not confirmed by code, docs, or user confirmation
- conversational filler

**Confirming a promotion is durable**: before presenting an item as established fact, ground it in at least one of: current versioned memory files, actual code or repo structure (grep or file read), git log, or explicit user confirmation. Do not treat one memory file as confirmation of another — memory files can themselves be stale.

## Maintenance objectives

Always optimize for:
- concision
- clarity
- non-ambiguity
- non-contradiction
- correct placement by memory layer
- high signal density
- minimal duplication

When auditing or patching, look for:
- duplicate rules across files
- contradictions between root and local rules
- vague language such as "if needed" or "be careful" without specifics
- large generic sections that do not change Claude's behavior
- material that belongs in a different layer
- missing durable promotions from visible auto-memory observations
- stale entries in `activeContext.md` (work that appears completed based on git log)
- undocumented patterns that appear in 2+ places in the codebase

## Bootstrap behavior

**Before doing anything else**, check whether `.claude/memory-bank/` contains the three standard
files: `activeContext.md`, `progress.md`, `systemPatterns.md`.

If one or more are missing, this is a first-run condition. Do not treat it as a generic gap in
Block 2. Instead:

1. Read the corresponding template(s) from `references/templates/`
2. Generate one init patch per missing file:
   - `P-init-1` → create `activeContext.md` from `references/templates/activeContext.md`
   - `P-init-2` → create `progress.md` from `references/templates/progress.md`
   - `P-init-3` → create `systemPatterns.md` from `references/templates/systemPatterns.md`
3. Present them in Block 4 labeled **Bootstrap** and grouped before any other patches
4. Tell the user they can initialize the full structure in one step:
   ```text
   mode: apply
   apply_target: all
   ```

If all three files already exist, skip bootstrap detection entirely and proceed with the normal
audit or patch workflow.

## Conversation-based promotions

If the user explicitly asks to promote something from the current conversation, OR if the
conversation contains explicit statements about any of the following — treat them as promotion
candidates without requiring auto-memory files:

- an architectural decision made or confirmed during this session
- a pattern validated or observed in 2+ places in the codebase
- a known issue discovered or reproduced
- a rule the team agreed on
- a completed milestone worth logging in `progress.md`

Apply the same promotion rules, routing, and confidence levels as for auto-memory items.
Present them in Block 3, clearly labeled **Source: conversation** rather than auto-memory.

Apply the same durable/not-durable filter: do not promote speculation, one-off confusion,
or conversational filler. Promote only what is explicitly stated as a decision, confirmed
pattern, or established fact within this session.

If the user says "promote this" or "remember this" while pointing at a specific exchange,
treat that as an explicit promotion request — extract the durable content and propose the
appropriate destination.

## Finding auto-memory

If `include_auto_memory: true` and no `/memory` output is visible in the conversation:
1. Search `~/.claude/projects/<encoded-project-path>/memory/` for memory files
2. If found, read and use as observational input
3. If not found, report clearly: "No auto-memory files found" — then ask the user to paste `/memory` output if auto-memory promotions are the goal

Never edit auto-memory files directly.

## Verification before patching implementation status

Before any patch that changes implementation status — marking a task done, removing a known issue, closing a debt entry:
1. Check `git log` for relevant commits or merged branches
2. Grep for the affected class, method, or flag in current code to confirm its current state
3. Do not rely solely on memory file content to infer code state — memory files lag behind the code

## activeContext lifecycle

During every audit, for each entry in `activeContext.md`:
- Cross-check against recent git log
- If the associated work appears merged or completed, flag it: "candidate to move to progress.md"

After every apply session that completes work items, offer explicitly:
> "These entries in activeContext.md appear completed — shall I rotate them to progress.md?"

`activeContext.md` must remain short (approximately 15 lines maximum). If it exceeds this, flag it as needing pruning during the audit.

## systemPatterns population

During every audit, scan 3–5 key service and persistence files and compare their recurring approaches against `systemPatterns.md`.

**Two-occurrence rule**: if the same implementation approach appears in 2 or more distinct files in the codebase and is not yet documented in `systemPatterns.md`, propose adding it.

After any session where a non-obvious implementation pattern is confirmed by the user or observed in multiple places, propose adding it to `systemPatterns.md` with a minimal code example.

## Local CLAUDE.md discovery

At the start of every `audit` or `patch` invocation:

1. **Check for `.claude/claude-files-registry.md`**
   - If absent: flag in Block 2 as missing infrastructure. Do not create it — that is bootstrap's
     responsibility. Propose that the user runs `/memory-bootstrap` or creates it manually.
   - If present: read it to discover all local `CLAUDE.md` files and their routing criteria.

2. **Glob for actual local `CLAUDE.md` files** — all `CLAUDE.md` files in the repo except the root one.

3. **Sync registry against filesystem**:
   - File exists on disk but missing from registry → propose adding the path. Do not write routing
     criteria — only bootstrap or a human initializes those.
   - Registry entry exists but file is absent on disk → propose removing the stale entry.

4. **Use routing criteria when dispatching subtree-specific rules**: when a promotion candidate or
   detected issue applies only to a specific module or directory, match it against the registry
   criteria to select the correct local `CLAUDE.md` as destination.

Never write or rewrite routing criteria in the registry — read and use them only.

## Import hygiene

During every audit/patch, verify that root `CLAUDE.md` enforces the three-tier loading strategy.

**Tier 1 — Required @imports** (must be present in root `CLAUDE.md`):
- `@.claude/memory-bank/activeContext.md`
- `@.claude/memory-bank/systemPatterns.md`
- `@docs/ai/known-issues.md`

If any @import is missing: propose a patch to add it.
If the target file does not exist: flag as a bootstrap gap — do not add a broken import.

**Tier 2 — On-demand references** (root `CLAUDE.md` must list these so Claude knows they exist):
- `.claude/memory-bank/progress.md`
- `docs/ai/architecture-decisions.md`
- `docs/ai/domain-glossary.md`
- `docs/ai/integration-patterns.md`
- `docs/ai/data-sources.md`

Root `CLAUDE.md` must contain a section listing these files with a short indication of when to read each.
If the section is absent or any file is unlisted: propose a patch.
If a listed file does not exist: flag it — do not leave a reference to a missing file.

**Tier 3 — Never imported directly** (heavy architecture docs):
These files must not appear as @imports in `CLAUDE.md`. They are accessed via the `architecture-reviewer`
subagent, routed through `docs/ai/architecture-map.md`.
If any of them appear as a direct @import: propose removing it.

## Workflow

### 1. Audit

When in `audit` mode:
1. Inspect the relevant versioned memory files
2. If `include_auto_memory: true`, locate and read auto-memory (see _Finding auto-memory_)
3. Run activeContext lifecycle check against git log
4. Scan 3–5 key service/persistence files for undocumented patterns (two-occurrence rule)
5. Identify all problems and promotion opportunities
6. Produce the four-block report
7. **Block 4 in audit mode: list patch IDs, target files, and short intent only — no unified diffs**
8. Do not apply changes

### 2. Patch proposal

When in `patch` mode:
1. Perform the full audit steps above
2. Incorporate any user refinements from the current conversation
3. Produce the four-block report
4. **Block 4 in patch mode: include full unified diffs for every patch**
5. Do not apply changes unless the user explicitly switches to `apply`

### 3. Apply

When in `apply` mode:
1. Verify there is an existing patch set in the conversation or enough instruction to generate one safely
2. Apply either all patches, a specific `patch:<id>`, or a specific `file:<path>`
3. Keep changes minimal and aligned with the approved plan
4. After applying, report what was changed and what remains unapplied
5. Offer to rotate completed items from `activeContext.md` to `progress.md`

If the user asks to refine the plan or remove an item, update the patch set first instead of applying stale patches.

## Required output format

Always use these four top-level blocks.

### Block 1 — Current memory state
Summarize the relevant memory files, what they currently do, and observed strengths/weaknesses.

### Block 2 — Problems detected
List contradictions, ambiguity, duplication, wrong-layer placement, stale entries, and missing promotions.

### Block 3 — Promotions from live sources
This block covers items sourced from:
- visible `/memory` or auto-memory output
- the current conversation (explicit decisions, confirmed patterns, discoveries, or user promotion requests)

If neither source has anything to promote, write: `No live-source promotions.` — do not repeat versioned-memory items here.

For each candidate:
- Source: `auto-memory` | `conversation` (quote or summarize the relevant exchange)
- Destination file
- Rationale
- Confidence: high | medium | low

### Block 4 — Patch plan
All patches, whether sourced from versioned-memory fixes or auto-memory promotions.

- In **audit** mode: list patch IDs, target files, short explanation. No diffs.
- In **patch** mode: full unified diffs for every patch.
- In **apply** mode: confirm what is being applied and what remains unapplied.

## Patch conventions

- Assign stable IDs like `P1`, `P2`, `P3`
- Support partial application by patch ID or file path
- Keep each patch focused on one theme or file cluster
- Prefer small diffs over broad rewrites
- Preserve user wording when they explicitly rewrite a section

## Iteration rules

Support iterative prompts such as:
- "instead of this memory part, replace it with the following"
- "i don't agree with P2; remove it from the plan"
- "keep the audit, but rewrite the patch for docs/ai/known-issues.md"
- "apply only patch:P3"
- "apply only file:docs/ai/known-issues.md"

When refining, carry forward unchanged accepted patches and clearly mark superseded ones.

## Style constraints

- Be concise and specific
- Do not rewrite large files without cause
- Do not create new memory files unless the user confirms a clear gap; first check actual filenames in the directory
- Do not silently delete important context; explain removals in the patch plan
- Treat the memory system like production code: explicit, reviewable, reversible

## References

For heuristics, examples, and usage patterns:
- `references/promotion-matrix.md` — routing decision table by information type
- `references/report-examples.md` — example audit, patch, and apply responses
- `references/user-manual.md` — concise usage guide with invocation examples
- `references/dependencies.md` — contracts this skill depends on; check before modifying shared structures
