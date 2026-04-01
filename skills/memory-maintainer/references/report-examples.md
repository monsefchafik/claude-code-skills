# Report Examples

## Example audit response

### Block 1 - Current memory state
- `CLAUDE.md` is concise and prescriptive.
- `docs/ai/business-rules.md` contains two solid invariants.
- `.claude/memory-bank/activeContext.md` is stale and still references a finished migration.

### Block 2 - Problems detected
- Duplicate guidance about minimal changes appears in both `CLAUDE.md` and `.claude/rules/backend.md`.
- `docs/ai/known-issues.md` contains one one-off incident that should be removed.
- Visible auto-memory mentions the actual integration-test command, but that knowledge has not been promoted into `docs/ai/runbook-dev.md`.

### Block 3 - Promotions from live sources
- Source: auto-memory — note about `./gradlew :billing:integrationTest`
  Destination: `docs/ai/runbook-dev.md`
  Rationale: stable operational command likely to be reused
  Confidence: high
- Source: stale note in `activeContext.md`
  Destination: removal only
  Rationale: completed work should not remain in active context
  Confidence: high

### Block 4 - Proposed patches / apply plan
- `P1` -> `docs/ai/runbook-dev.md` -> add confirmed billing integration-test command
- `P2` -> `.claude/memory-bank/activeContext.md` -> remove stale migration focus

```diff
*** Begin Patch P1: docs/ai/runbook-dev.md
@@
 ## Test commands
 - Unit: ./gradlew test
+- Billing integration: ./gradlew :billing:integrationTest
*** End Patch P1
```

```diff
*** Begin Patch P2: .claude/memory-bank/activeContext.md
@@
-Current priority: finish legacy billing migration
*** End Patch P2
```

---

## Example conversation-based promotion

User during a working session:
> "we just confirmed that all PricingEngine calls must go through the retry wrapper — this burned us twice already"

Then later invokes the skill with:
> "promote the decision we just made about PricingEngine retries"

### Block 3 - Promotions from live sources
- Source: conversation — "all PricingEngine calls must go through the retry wrapper — this burned us twice already"
  Destination: `.claude/memory-bank/systemPatterns.md` (two-occurrence rule met: burned twice)
  Rationale: confirmed recurring implementation decision, not speculation
  Confidence: high

### Block 4 - Patch plan (audit mode)
- `P1` → `.claude/memory-bank/systemPatterns.md` → add PricingEngine retry wrapper pattern under Integration Patterns

---

## Example refinement request

User:
"I don't agree with P2. Keep the migration item, but mark it as paused instead of removing it."

Expected behavior:
- keep `P1`
- supersede `P2`
- produce a new patch for `activeContext.md` reflecting the paused status
- clearly label the older `P2` as superseded

---

## Example apply request

User:
`mode: apply`
`apply_target: patch:P1`

Expected behavior:
- apply only the patch labeled `P1`
- report that `P2` remains unapplied
