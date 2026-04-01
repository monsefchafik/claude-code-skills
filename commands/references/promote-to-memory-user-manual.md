# promote-to-memory — Usage Guide

For the full specification, see [`promote-to-memory.md`](../promote-to-memory.md).

## What this command does

`/promote-to-memory` extracts durable learnings and integrates them into the project's
versioned memory. It scans two sources: the current conversation and/or a
`full-analysis.md` / `<name>-analysis.md` file provided via `doc=`.

It automatically deduplicates against existing target files.
It never touches existing memory without explicit approval.

---

## When to use it

| Situation | Use? |
|-----------|------|
| End of an `/architecture-document` session — promoting PROMOTE-CANDIDATEs | Yes — always |
| Architectural decision made during a session | Yes |
| Implementation rule confirmed and validated in session | Yes |
| Pattern observed in 2+ places in the code during a session | Yes |
| Bug or operational risk discovered and reproducible | Yes |
| Simple Q&A exchange with no decision | No |
| Content already present in `.claude/rules/` or `systemPatterns.md` | No |
| Speculative observations or unconfirmed hypotheses | No |

---

## Invocation

```bash
# Conversation mode — scans the current session
/promote-to-memory

# File mode — reads ## PROMOTE-CANDIDATE from a file + conversation
/promote-to-memory doc=docs/ai/architecture/persistence/full-analysis.md
/promote-to-memory doc=docs/notes/my-unit-analysis.md
```

`doc=` is useful when `/architecture-document` or `codebase-analysis` were run
in a previous session — the PROMOTE-CANDIDATEs are in the file, not in the
current conversation.

---

## What happens

```
1. Collect candidates
   → Source A (if doc=) : ## PROMOTE-CANDIDATE section from file — confirmed candidates
   → Source B           : scan conversation — PROMOTE-CANDIDATEs first, then
                          decisions, patterns, rules, confirmed milestones
   → Merge + deduplication between sources A and B

2. Durable/non-durable filter (Source B only)
   → PROMOTE : explicit decisions, confirmed patterns, established facts
   → SKIP    : speculation, one-off observations
   ⚠️ PROMOTE-CANDIDATEs (A and B) are NOT re-filtered — already validated

3. Deduplication against existing memory
   → For each candidate: read the target file
   → If a similar rule already exists → SKIP "already present in <file> (line ~N)"
   → When in doubt: flagged as "potential duplicate" rather than silently skipped

4. Patch report (Block 3 + Block 4)
   → Lists all candidates: PROMOTE or SKIP with reason
   → Unified diffs ready to apply (non-skipped items only)

5. Approval gate
   → "Apply these promotions? (yes / no / select numbers)"
   → No write without explicit confirmation
```

---

## Possible destinations

| Promoted content | Destination |
|-----------------|-------------|
| Implementation rule, invariant, constraint | `.claude/rules/<domain>.md` |
| Confirmed recurring pattern (2+ occurrences) | `.claude/memory-bank/systemPatterns.md` |
| Operational risk, known bug | `docs/ai/known-issues.md` |
| Architectural decision | `docs/ai/architecture-decisions.md` |
| Completed milestone | `.claude/memory-bank/progress.md` |

---

## Typical workflow after `/architecture-document` (same session)

```
1. /architecture-document <mechanism>
   → Files created, PROMOTE-CANDIDATEs in full-analysis.md + displayed in conversation

2. Human validation
   → Review summary.md (≤ 100 lines?)
   → Review decision.md (rejected alternatives documented?)

3. /promote-to-memory
   → Detects PROMOTE-CANDIDATEs first (conversation)
   → Deduplicates against existing .claude/rules/
   → Proposes diffs

4. Approval gate → "yes" / "no" / "select 1,3"

5. Wiring (if HIGH risk mechanism)
   → Create local CLAUDE.md with @summary.md
```

## Typical workflow after `/architecture-document` (later session)

```
1. New session — PROMOTE-CANDIDATEs are no longer in the conversation

2. /promote-to-memory doc=docs/ai/architecture/<mechanism>/full-analysis.md
   → Reads ## PROMOTE-CANDIDATE from the file
   → Deduplicates against existing .claude/rules/
   → Proposes diffs

3. Approval gate → "yes" / "no" / "select 1,3"
```

**Do not reverse validation and promotion**: validate the files before promoting.
A rule derived from an incorrect `summary.md` will itself be incorrect.

---

## Examples of typical promotions

**From `/architecture-document`:**
```
PROMOTE-CANDIDATE:
- "Always sort by ID before saveAll()" → .claude/rules/database-operations.md
- "XxxEntity.fromDomainModel() pattern in 4 files" → systemPatterns.md
```

**From a debugging session:**
```
Discovery: ORA-00060 deadlock reproduced when two threads save
PRODUCT in different orders.
→ Promoted to: docs/ai/known-issues.md
```

**From a decision made in session:**
```
Decision: do not call external API inside a JPA transaction.
→ Promoted to: .claude/rules/database-operations.md
```

---

## What the command does NOT do

- Without `doc=`: does not scan previous sessions — only the visible conversation
- Does not modify `full-analysis.md` or `summary.md` (that is `/architecture-document`'s role)
- Does not duplicate what is already in versioned memory (automatic deduplication)
- Never writes without explicit approval

---

## Common mistakes

| Mistake | Consequence | Solution |
|---------|-------------|----------|
| Running before validating architecture files | Incorrect rule promoted | Always validate summary.md first |
| Answering "yes" without reading the diffs | Promotion of irrelevant content | Read Block 3 before confirming |
| Running multiple times on the same session | Duplicates in target files | Verify that SKIP applies correctly on the 2nd run |

---

## See also

- [`promote-to-memory.md`](../promote-to-memory.md) — full specification
- [`promote-to-memory-dependencies.md`](./promote-to-memory-dependencies.md) — dependencies with `/architecture-document`
- [`memory-bootstrap-dependencies.md`](./memory-bootstrap-dependencies.md) — memory-bootstrap/memory-maintainer dependencies
- `docs/ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md` — Phase 4 of the complete workflow
