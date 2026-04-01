# Architecture Reviewer — Usage Guide

For the full specification, see [SKILL.md](../SKILL.md).

## What this subagent does

`architecture-reviewer` reads the documentation for a mechanism in an **isolated context window**
and returns an implementation recommendation in 20 lines maximum.

It does not pollute the main thread with 1500 lines of full-analysis.
It does not modify any files.

```
Main thread               Subagent (isolated context)
      │                           │
      │── "review X for task Y" ─▶│ reads summary.md (50-100 lines)
      │                           │ reads full-analysis.md if needed
      │◀── 20-line recommendation ─│
      │
      │ continues with 20 lines
      │ of targeted context
```

## When to use it

| Situation | Use reviewer? |
|-----------|---------------|
| Simple change in a well-known module | No |
| Change to a documented MEDIUM-risk mechanism | Yes |
| Change to a documented HIGH-risk mechanism | Yes — required |
| Refactoring that touches a complex mechanism | Yes |
| New feature integrating into an existing mechanism | Yes |
| Undocumented mechanism | No — run `/architecture-document` first |

## Invocation

⚠️ **Absolute rule**: `architecture-reviewer` must ALWAYS be invoked via the Agent tool (isolated
subagent). Never read `full-analysis.md` directly in the main thread — this cancels the isolation
benefit and pollutes the context.

### How to invoke it

Tell Claude (in natural language):

```
Use architecture-reviewer. mechanism: generator-pipeline. task: Add a new source type without breaking existing validation.
```

Claude recognizes the skill via its system description ("Invoke via the Agent tool"), loads
`SKILL.md` via the Skill tool, then launches the subagent with the Agent tool.

### How Claude knows it needs a subagent

Reliability relies on three cumulative signals:

| Signal | Content | Scope |
|--------|---------|-------|
| Skill description (system-reminder) | "Invoke via the Agent tool" | Every session |
| `SKILL.md` section "How to invoke" | Shows the Agent tool explicitly | Every invocation |
| Root `CLAUDE.md` | "ALWAYS via Agent tool — never read full-analysis.md directly" | Every session |

**If the request is vague** ("look at the docs for X"), Claude may read files directly
without spawning a subagent. Always use the canonical phrasing to guarantee isolation:

```
Use architecture-reviewer. mechanism: <name>. task: <precise description>
```

## What the reviewer returns

A structured response in 20 lines max:

```
## Analysis — generator-pipeline / New source type

**Risk**: HIGH
**Coverage**: full-analysis consulted (section "Extension points")

### Constraints to respect
- Implement SourceTypeHandler for the new type
- Register in SourceTypeRegistry before use
- The pipeline validates the type before each execution — do not bypass

### Implementation recommendation
Create a class that implements SourceTypeHandler, register it
in SourceTypeRegistry via @Component. Do not modify the main pipeline
— the mechanism is extensible by design.

### Task-specific attention points
- The cache is invalidated by type — test invalidation
- Existing integration tests cover 3 types — add the 4th

### If insufficient
Read full-analysis.md § "Extension points and interface contracts"
```

## What the reviewer does NOT do

- Does not modify any files
- Does not read source code — only architecture docs
- Does not return more than 20 lines
- Does not copy full-analysis.md content into its response
- Does not invent facts not present in the docs

## If the mechanism is not documented

The reviewer responds:
```
No architecture documentation found for `<mechanism>`.
Recommendation: run a reverse-engineering session then
execute `/architecture-document <mechanism>` before continuing.
```

In that case: stop, analyze the mechanism with Claude, then
run `/architecture-document` before the implementation task.

## Typical workflow with the reviewer

```
1. You want to implement a change on a complex mechanism

2. You request a review:
   "Use architecture-reviewer for generator-pipeline.
    Task: [precise description]"

3. Claude invokes the subagent
   → isolated context, no thread pollution

4. The reviewer returns its recommendation (20 lines)

5. You continue implementation with this targeted context
   → Claude knows the constraints without having loaded 1500 lines

6. If implementation reveals new insights:
   → /promote-to-memory to capture them
   → Or update full-analysis.md if it is a correction
```

## See also

- [SKILL.md](../SKILL.md) — full subagent specification
- [architecture-document user-manual](../../architecture-document/references/user-manual.md) — how to generate the documentation the reviewer consults
- [ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md](../../../../docs/ARCHITECTURE_DOCUMENTATION_PATTERN_EN.md) — overview of the pattern
