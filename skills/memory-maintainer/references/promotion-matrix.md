# Promotion Matrix

Use this matrix when deciding where information belongs.

## Stable instruction about Claude behavior

**Destination**: `CLAUDE.md` (root) or local `CLAUDE.md` (subtree)

Examples:
- "prefer minimal local changes"
- "propose a short plan before non-trivial refactors"
- "in this subtree, preserve backward compatibility of partner contracts"

## Specialized reusable rule

**Destination**: `.claude/rules/*.md`

Examples:
- testing expectations for endpoint changes
- database migration safety rules
- security rules for admin and secrets handling

## Durable explanatory knowledge

**Destination**: `docs/ai/*.md` (route to the most specific existing file)

| Knowledge type | Target file (typical) |
|----------------|----------------------|
| Architecture boundaries and flows | `docs/ai/architecture-decisions.md` |
| Domain glossary terms | `docs/ai/domain-glossary.md` |
| Recurring operational pitfall or known bug | `docs/ai/known-issues.md` |
| Architectural decision and trade-offs | `docs/ai/architecture-decisions.md` |
| Integration patterns for new data sources | `docs/ai/integration-patterns.md` |

## Current work and near-term risks

**Destination**: `.claude/memory-bank/activeContext.md`

Examples:
- active refactoring effort with its current blocking point
- known risk before an upcoming release
- pending architectural decision not yet resolved

**Lifecycle**: entries here are temporary. When work is done, move to `progress.md`. This file should not exceed ~15 lines.

## Completed milestones

**Destination**: `.claude/memory-bank/progress.md`

Examples:
- milestone delivered and merged
- breaking change shipped and documented
- migration completed

**Rule**: append-only. Never delete or edit existing entries — this file is a log, not a status board.

## Recurring implementation patterns

**Destination**: `.claude/memory-bank/systemPatterns.md`

**Entry threshold (two-occurrence rule)**: promote only when the same implementation approach appears in 2 or more distinct files in the codebase, OR when a non-obvious design decision has been explicitly validated by the team.

Examples:
- the standard way to map entities to domain objects in this project
- the error-collection pattern used across all generators
- the enrichment loop approach used across multiple enrichment services

Do not promote patterns that appear only once — they may be coincidence or experimentation.

## Conversation-sourced item

**Destination**: same routing rules as all other items — determined by the nature of the content, not its source.

**When to promote**: when the user explicitly asks ("promote this", "remember this"), or when the conversation contains an explicit statement of a decision, confirmed pattern, discovered issue, or completed milestone.

**Label in Block 3**: `Source: conversation` with a quote or summary of the relevant exchange.

Examples that qualify:
- "we decided to always wrap RXH calls in a retry decorator" → `systemPatterns.md` or `.claude/rules/`
- "the RXH client has a hard 30-second timeout that causes silent failures on large payloads" → `docs/ai/known-issues.md`
- "DPR-1790 is merged, sorted saveAll is done" → `progress.md`
- "from now on, always filter ConsistencyErrors by source before any update" → `.claude/rules/database-operations.md`

Examples that do not qualify:
- "I think maybe we should look at X" (speculation)
- "that was confusing" (frustration, not a rule)
- "let me check that file" (navigation, not a decision)

## Observational auto-memory item

**Destination**: promote only if durable and useful beyond one session, after verification.

Examples that often qualify:
- the real integration test command confirmed by use
- a recurring startup dependency order
- the module boundary that repeatedly matters in practice

Examples that usually do not qualify:
- one-time confusion during an investigation
- temporary branch or ticket context
- speculative interpretation of business logic

## Heuristics

Promote when the item is:
- reusable across multiple future tasks
- stable enough to survive multiple sprints
- likely to change Claude's future behavior
- better stored in versioned memory than repeated in chat

Do not promote when the item is:
- already stated elsewhere in memory (update the canonical location instead)
- likely to expire within days
- too vague to guide future work
- confirmed only by another memory file (circular — verify against code or git)
