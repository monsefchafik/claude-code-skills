---
name: codebase-analysis
description: Techniques for analyzing existing codebases to reverse-engineer requirements and understand business logic. Use when conducting brownfield analysis or understanding existing system capabilities. Adapts to repo size. Supports iterative refinement via doc= argument. Ends with a review loop then an output gate — never writes to project memory without user confirmation.
allowed-tools: Read, Grep, Glob, Bash
---

# Codebase Analysis Skill

## Overview

Structured process for extracting business requirements, domain knowledge, and technical
specifications from an existing codebase. Adapts to repo size. Supports iterative refinement:
analysis results live in the conversation until the user is ready to produce output.
Never writes to project memory autonomously.

---

## Arguments

- `doc=<path>` (optional): path to an existing analysis file to resume from.
  - The file is read as the current state of the analysis.
  - Claude presents a summary of what is already known and continues from there.
  - Output gate offers to update this same file (not create a new one).
  - Typical use: refine an analysis exported in a previous session.
- `output=` (optional): pre-select output mode at invocation time
  - `output=memory` — user intends to run `/architecture-document` at the end
  - `output=<path>` — export artefacts to this folder
  - omitted — ask at the end of each unit analysis

---

## Phase 0 — Cartography (always, before reading any code)

### Steps

**0.0 — If `doc=` argument provided: resume mode**

Read the file at the given path. Present a summary of what is already documented:

```
EXISTING ANALYSIS DETECTED — <filename>
─────────────────────────────────────────
Unit: <name> [<type>]
Current coverage:
  ✅ Domain model    : [summary]
  ✅ Business rules  : [summary]
  ⚠️  Integrations   : [incomplete or missing]
  ❌ Edge cases      : [not covered]

What would you like to deepen or correct?
(or "continue" to go directly to the review loop)
```

Do not re-run phases already covered unless the user asks.
Skip steps 0.1–0.6 and go directly to the Review Loop.

**0.1 — Read high-level structure only**
```bash
# Folder structure
find src -type d | head -40

# Estimate size per package
find src/main -name "*.java" | xargs wc -l 2>/dev/null | sort -rn | head -30

# Entry points
cat README.md 2>/dev/null | head -60
```

**0.2 — Identify units and classify** (type is optional — use `?` if uncertain)

| Observed scope | Type |
|---------------|------|
| 1 package, cohesive set of classes | `module` |
| Multiple classes around a single functional flow | `component` |
| Multiple packages coordinated toward a function | `subsystem` |
| Domain model boundary (distinct ubiquitous language) | `bounded-context` |

**0.3 — Estimate risk per unit**
- HIGH: core business logic, data integrity, known production issues
- MEDIUM: significant feature, moderate complexity
- LOW: utility, configuration, thin layer

**0.4 — Check existing plan and documentation (PRIORITY)**

Check for an existing analysis plan first — it takes precedence over a fresh cartography:

```bash
# Existing analysis plan — resume from it if present
ls docs/ai/analysis-plan-*.md 2>/dev/null && cat docs/ai/analysis-plan-*.md 2>/dev/null

# Units already documented in project memory
cat docs/ai/architecture-map.md 2>/dev/null
```

**If a plan exists**: present it directly (skip fresh cartography unless user asks).

For `📁` units: look up the export path from the plan and show the `doc=` resume command.
If the export path points to a directory (old multi-file format), look for `full-analysis.md`
or `analysis.md` inside it. If neither exists, mention the path and let the user confirm
which file to use.

```
EXISTING PLAN DETECTED — docs/ai/analysis-plan-<repo-name>.md
──────────────────────────────────────────────────────────────
✅  <name>  [<type>]  HIGH    → documented  (2026-03-23)
📁  <name>  [<type>]  MEDIUM  → exported    (2026-03-23)
    ↳ Resume: /codebase-analysis doc=<export-path>/<name>-analysis.md
⬜  <name>  [<type>]  HIGH    → to do  ⚙️  ← next recommended
⬜  <name>  [<type>]  MEDIUM  → to do
🚫  <name>  [<type>]  LOW     → ignored

⚙️  = saved analysis parameters (objective, scope, context)

Resume with <next-unit> [<type>]?
Options:
  - Confirm (continue with next ⬜ unit)
  - Resume a 📁 exported unit (see ↳ command above)
  - Choose a different unit
  - Mark a unit as 🚫 ignored or ⬜ to do
  - Modify parameters for a unit (objective, scope, context)
  - Redo the full cartography
```

When presenting the plan: for each unit that has an entry in `## Analysis details`,
append `⚙️` to its line.

**If no plan exists**: proceed with fresh cartography (steps 0.1–0.3 above).

**0.5 — Present plan and wait for confirmation**

```
CARTOGRAPHY — <repo-name>
───────────────────────────
Estimated size: ~XXk lines, ~XX classes

Identified units:

  [HIGH]   <name>   [<type|?>]   ~XX classes   <package>
  [HIGH]   <name>   [<type|?>]   ~XX classes   <package>
  [MEDIUM] <name>   [<type|?>]   ~XX classes   <package>
  [LOW]    <name>   [<type|?>]   ~XX classes   <package>

Already documented (skipped unless requested):
  <units from architecture-map.md>

Strategy: <small | medium | large>  (see rules below)

Confirm, adjust types/priorities, mark units (🚫 ignored),
add a missing unit ("Add <name> [<type>] [<risk>]"),
or exclude units before continuing.
```

Do not read any code until the user confirms.

**0.6 — Save or update the analysis plan**

**If plan already exists**: update it silently after user confirmation (no need to ask).

**If no plan exists**: offer to create it:
```
Save this plan to docs/ai/analysis-plan-<repo-name>.md?
(Recommended for large repos — the only way to resume between sessions)
(yes / no)
```

Plan format:

```markdown
# Analysis plan — <repo-name>
Date: <today>

## Statuses

| Status | Meaning |
|--------|---------|
| ⬜ | To do |
| ⏳ | In progress (active session) |
| ✅ | Documented (architecture-map.md updated) |
| 📁 | Exported to a file |
| 💬 | Conversation only (not persisted) |
| 🚫 | Ignored (persistent decision) |

## Units

| Unit | Type | Risk | Status |
|------|------|------|--------|
| <name> | <type> | HIGH | ⬜ to do |
| <name> | <type> | HIGH | ⬜ to do |
| <name> | <type> | MEDIUM | ⬜ to do |
| <name> | <type> | LOW | ⬜ to do |

## Analysis details

<!-- Parameters stored per unit. Only units with defined parameters appear here. -->
<!-- For absent units: scope = auto, objective = to be asked before analysis. -->

### <unit-name>
- **Objective**: b) Debug / investigate an issue
- **Scope**: service/cache, adapter/config/cache
- **Freedom**: indicative
- **Context**: PersistService, nightly scheduler

## Sessions
| Date | Unit analyzed | Output chosen |
|------|--------------|---------------|
```

---

## Sizing Strategy

| Size | Criteria | Approach |
|------|----------|----------|
| **Small** | < 5k lines, ≤ 3 units | All phases in one pass, single conversation |
| **Medium** | 5k–30k lines, 3–8 units | Phase 0 + phases by layer, single conversation |
| **Large** | > 30k lines, > 8 units | Phase 0 + **one conversation per HIGH/MEDIUM unit** |

### Large repo — per-unit session rule

One conversation per unit keeps context clean and focused:

```
Conversation 1 : Phase 0 → plan saved → analyse unit A → output gate
Conversation 2 : resume from analysis-plan → analyse unit B → output gate
Conversation 3 : resume from analysis-plan → analyse unit C → output gate
```

To resume in a new conversation:
```
"Resume analysis of <repo>. The plan is in docs/ai/analysis-plan-<repo>.md.
 Next unit to analyze: <name>."
```

Context management — three rules:

**Rule 1 — Phase 1 = zero reading**
Phase 1 is done entirely by grep. Feeling the need to read a file
during Phase 1 = signal that we are already in Phase 2.

**Rule 2 — Surgical reading (Phases 2-5)**
- Grep first, read only if grep is insufficient
- Files > 300 lines: read in sections (offset+limit), never in full
  except for the pivot class identified in 1.1
- Only read a class body if the phase explicitly requires it

**Rule 3 — Safety net: 70% context threshold**
- If context > 70% (`/context`): export the partial analysis (option 2)
  with a dense and structured file, continue with `doc=` in a new session
- The exported file must be self-contained — readable by Claude without re-reading the code,
  otherwise the next session restarts with the same context problem
- Propose a unit split only if two functionally independent responsibilities
  are identifiable — never based on size or line count

---

## Analysis Phases (run per unit, or globally for small repos)

### Phase 1 — Orientation

4 targeted commands producing a navigation map before any code reading.
Each result directly feeds into a subsequent phase.

```bash
# 1.1 PIVOT — most internally referenced class
# → starting point for Phase 2 and Phase 3
grep -rh "import" src/main/<package>/ --include="*.java" \
  | grep "<package>" \
  | sed 's/.*import //; s/;//' \
  | sort | uniq -c | sort -rn | head -10

# 1.2 ENTRY — external callers of the package
# → starting point for Phase 4 (integration mapping)
grep -rh "import.*<package>" src/main/ --include="*.java" \
  | grep -v "^<package>/" \
  | sed 's/.*import //; s/;//' \
  | sort | uniq -c | sort -rn | head -10

# 1.3 OUTPUT — external business dependencies (JDK/Lombok/Spring/Guava noise filtered)
# → ports, repositories, third-party services consumed — feeds Phase 4
grep -rh "import" src/main/<package>/ --include="*.java" \
  | grep -v "<package>" \
  | sed 's/.*import //; s/;//' \
  | grep -v "^java\.\|^lombok\.\|^org\.springframework\.\|^com\.google\." \
  | sort | uniq -c | sort -rn | head -15

# 1.4 HIERARCHY — abstract skeleton of the package
# → identifies pivot classes (abstract), leaves (concrete), local interfaces
grep -rn "^public abstract\|^public interface\|extends\|implements" \
  src/main/<package>/ --include="*.java" \
  | grep -v "import\|//" | sort | head -30
```

**1.5 PATTERN** — interpretation of 1.4, no dedicated command
- If CLAUDE.md documents the global pattern → confirm it, identify local deviations
- Local pattern (Template Method, Strategy, etc.) recognized by reading 1.4

**Mandatory deliverable** before moving to Phase 2:

```
ORIENTATION MAP — <package>
─────────────────────────────
Pivot            : <class> (X internal refs)
External callers : <class1>, <class2>
Key dependencies : <Port1>, <Port2>
Hierarchy        : <AbstractX> ← <ConcreteA>, <ConcreteB>
Pattern          : <global> + <local pattern if different>

Phase 2 starting point: <pivot>
Phase 4 starting point: <main caller>
```

### Phase 2 — Domain Model Extraction

1. Locate model classes based on the pattern identified in Phase 1:
   - Hexagonal   : application/domain/model/ (pure POJOs) — exclude JPA entities in adapter/
   - Layered     : domain/ or model/ — distinguish persisted entities from pure domain objects
   - MVC         : model/ — watch for classes mixing persistence and business logic
   - Unknown     : look for classes with no @Service/@Component/@Repository annotations
                   and no framework dependencies → likely the domain models

2. Map relationships between entities

3. Identify domain vocabulary

4. Document data types and constraints

5. Identify state markers on entities
   → Look for: enums, String constants, boolean fields, value conventions
     (e.g. date = MAX_DATE meaning "current", status = null meaning "unprocessed")
   → Signal: a field whose value conditions behavior elsewhere in the code
   → Document: `<entity> — marker: <field> — observed values: <v1, v2, v3>`
   → Do not analyze transitions here — just identify and list (transitions = Phase 3)

6. For notable attributes (non-trivial source, non-obvious validation, or transformation
   present): group by entity or logical sub-group, then document per group — factorizing
   common rules, using `↳` only for per-attribute exceptions:

   ```
   Group "<label>" : <attr1>, <attr2>, <attr3>
     → Source         : User | External app (<name>) | Produced internally (computed/enriched)
                        ? if not visible in Phase 2 (completed in Phase 3 or 4)
     → Validation     : <common rule>  [File:line]
                        ? if not visible in Phase 2
       ↳ <attr>       : <specific rule>  [File:line]
     → Transformation : <common transformation>  [File:line]
                        ? if not visible in Phase 2
       ↳ <attr>       : <specific transformation>  [File:line]
   ```

   **`?` = explicit placeholder, not an omission.** Source, Validation and Transformation
   are cross-cutting — they emerge progressively across phases:
   - Visible in constructor or mapper → capture in Phase 2
   - Emerges from business logic → Phase 3 completes
   - Comes from incoming flow → Phase 4 completes

   **Grouping rules:**
   - Group attributes that share source, validation logic, or transformation
   - If an attribute's locations diverge significantly from the group → split the group
     (diverging locations signal a wrong grouping — document it as a finding)
   - Trivial attributes (identity fields, auto-generated, no transformation) → single
     group "Technical fields", no detail needed
   - A group with many `↳` exceptions is itself a finding (fragmented logic)

   **PROMOTE-CANDIDATE trigger:** a group rule that appears in 2+ entities → candidate
   for `systemPatterns.md`

   **PROMOTE-CANDIDATE trigger:** a domain term (business vocabulary appearing consistently
   across entities, with no clear equivalent in code identifiers) → candidate for
   `docs/ai/domain-glossary.md`

**Check existing project patterns** (if `.claude/memory-bank/systemPatterns.md` exists):
- Cross-reference discovered patterns with documented ones
- Do not re-document what is already captured

### Phase 3 — Business Logic Identification

**Before starting:** check Phase 2 output for any `?` on Source, Validation, or
Transformation. Complete them as findings emerge — do not wait for end of phase.

1. Locate service and use-case classes
2. Extract validation rules and constraints
3. Document calculations and workflows
4. Map state machines and lifecycle transitions

**Business rule indicators** (language-agnostic):
`Validate`, `Check`, `Ensure`, `Must`, `Should`, `Calculate`, `Compute`,
`If`, `When`, `Unless`, `Max`, `Min`, `Required`, `Forbidden`

Look for: exception throws, policy classes, specification patterns, rule engines

**For each rule identified, apply the following:**

**a) Exception names → explicit rules**
Each custom exception throw is a rule stated in code. Extract it directly:
```java
throw new CommandException("CTR_COMMAND_MISSING_ID")
→ Rule: "An command must have an ID"
```
The exception name IS the rule — do not paraphrase, capture it verbatim.

**b) Enforcement coverage**
For each rule: is it enforced at ALL write entry points?
- List the entry points that write the affected data (controllers, schedulers, importers)
- Check each one for the rule
- Any entry point missing the rule → **coverage gap** (critical finding)

```bash
# Find all write paths for the affected entity
grep -rn "save\|persist\|update\|insert" src/main/<package>/ --include="*.java" | grep -v test
```

**c) Implicit rules**
Rules not written as explicit validations — the most dangerous kind:
- Magic numbers without named constants (`if (retryCount > 3)` with no explanation)
- Order dependencies (`A must be called before B` by convention, not enforced by code)
- Silent conventions (absence of validation = implicitly accepted)

Document as: `[IMPLICIT] <description> — suspected from <evidence>`

**d) Rule classification → PROMOTE-CANDIDATE routing**

| Type | Description | Destination |
|------|-------------|-------------|
| Invariant (repo-wide) | Always true, never changes, applies to the whole repo | `.claude/rules/` |
| Invariant (subtree-specific) | Always true, but applies to one module only | Local `CLAUDE.md` (via registry) |
| Business policy (mechanism-specific) | Business decision tied to the mechanism being analyzed | `decision.md` |
| Business policy (cross-cutting or mechanism already documented) | Architectural decision or ADR | `docs/ai/architecture-decisions.md` |
| Operational constraint | Imposed by DB, infra, or external system | `docs/ai/known-issues.md` or `summary.md` |
| Business term | Business vocabulary with no equivalent in code identifiers | `docs/ai/domain-glossary.md` |

Apply the classification to every rule. It determines directly where it goes as PROMOTE-CANDIDATE.

**e) Confidence level — mandatory for every PROMOTE-CANDIDATE item**

| Level | Criteria |
|-------|----------|
| `high` | Extracted from a throw/exception statement, OR grep-confirmed in 2+ distinct files, OR coverage gap found by grep |
| `medium` | Observed consistently but based on Claude's judgment (domain terms, heuristic patterns, single-unit integration) |
| `low` | `[IMPLICIT]` — suspected from a comment, naming convention, or indirect evidence; not explicitly stated in code |

Always append the confidence to every PROMOTE-CANDIDATE entry:
```
- "<item>"  → <destination>  [high — extracted from throw]
- "<item>"  → <destination>  [medium — appears consistently across entities]
- "<item>"  → <destination>  [low — implicit, suspected from comment at Scheduler:45]
```

**Flow boundary rule** — when tracing the main flow, stop at any point where the code
exits the read perimeter without being followed. Mark each such boundary with `?` inline:

- abstract / interface method call → implementations not read
- call to a class outside the scope → body not read
- complex parameter whose internals were not explored
- return value that carries more than what is visible at the call site

```
Main flow : <methodA> → <methodB>  [File:line]
              ? <call or type not followed>  — <reason>  [File:line]
            → <methodC>  [File:line]
```

`?` boundaries are the natural agenda for Review Loop deep-dives. Do not follow them
automatically — stay within the pivot class. The human decides what to deepen.

**Mandatory livrable** before moving to Phase 4:

```
BUSINESS LOGIC — <unit>
───────────────────────────────────────────────────
Entry points covered  : <SCHEDULER X [File:line], API Y [File:line]>
Main flow             : <methodA> → <methodB>  [File:line]
                          ? <call or type not followed>  — <reason>  [File:line]
                        → <methodC>  [File:line]
Rules identified      :
  [CODE_CTR_...]  at step <methodB>   [high — extracted from throw]  [File:line]
  [IMPLICIT] ...                      [low — suspected from <evidence>]
State transitions     : <marker> : <v1> → <v2> under condition <C>  [File:line]
PROMOTE-CANDIDATEs    : <count> identified (see classifications above)
```

Each item must be backed by a `[File:line]` reference. Do not include items inferred
without a code anchor.

### Phase 4 — Integration Mapping

**Before starting:** check Phase 2 output for any remaining `?` on Source not yet
resolved by Phase 3. Incoming flows often reveal the origin of attributes — complete
them as integrations are mapped.

1. Find all entry points exposed by the unit:

   ```bash
   # REST endpoints (IN)
   grep -rn "@RestController\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@RequestMapping" \
     src/main/<package>/ --include="*.java"

   # Schedulers (IN)
   grep -rn "@Scheduled" src/main/<package>/ --include="*.java"
   ```

2. Find all outgoing integration calls:

   ```bash
   # Outgoing HTTP calls
   grep -rn "HttpClient\|RestTemplate\|WebClient\|FeignClient" src/main/<package>/ --include="*.java"

   # Message queues
   grep -rn "publish\|subscribe\|send\|consume\|@KafkaListener\|@RabbitListener" src/main/<package>/ --include="*.java"

   # File IN/OUT
   grep -rn "MultipartFile\|Files\.read\|FileInputStream\|FlatFileItemReader\|Files\.write\|FileOutputStream\|FlatFileItemWriter\|BufferedWriter\|BufferedReader" src/main/<package>/ --include="*.java"

   # SFTP / FTP
   grep -rn "JSch\|FTPClient\|SftpClient\|ChannelSftp" src/main/<package>/ --include="*.java"

   # Email
   grep -rn "JavaMailSender\|MimeMessage\|SimpleMailMessage" src/main/<package>/ --include="*.java"

   # Cache externe
   grep -rn "RedisTemplate\|@Cacheable\|@CacheEvict\|@CachePut" src/main/<package>/ --include="*.java"
   ```

   If Phase 1.3 identified a dependency toward an interface with no visible implementation
   in the package: search for the implementation outside the package scope and include it
   in the integration mapping.
   ```bash
   # Find implementations of <InterfaceName> outside the unit package
   grep -rn "implements <InterfaceName>" src/main/ --include="*.java"
   ```

3. For each integration found, document in compact format:

   ```
   [IN|OUT]  <name>  →  <entity/DTO exchanged>  |  failure: <ignore|exception|fallback>  |  [File:line]
   ```

4. **Detect implicit coupling violations** — based on the architectural pattern
   identified in Phase 1:

   | Pattern (Phase 1) | Coupling rule to verify |
   |-------------------|------------------------|
   | Hexagonal | `application/` must not import `adapter/` |
   | Layered (N-tier) | `domain/` must not import `infrastructure/` |
   | MVC | `model/` must not import `controller/` |

   Run the grep corresponding to the detected pattern. Any result = architectural
   violation → critical finding → PROMOTE-CANDIDATE → `.claude/rules/<architecture>.md`

5. **Map outgoing triggers** — what features/flows does this unit activate as output,
   and under what conditions?

   ```bash
   grep -rn "publish\|emit\|trigger\|notify\|schedule\|dispatch\|ApplicationEventPublisher\|@TransactionalEventListener\|@Async\|CompletableFuture" src/main/<package>/ --include="*.java"
   ```

   For direct service-to-service calls: cross-reference the dependency map from Phase 1
   — not duplicated here.

   Document as:
   ```
   Outgoing triggers:
     → <feature or flow name>   always
     → <feature or flow name>   if <condition>
     → <feature or flow name>   on error
   ```

   This produces the **implicit dependency graph between features** — what breaks silently
   when this unit changes. Empty = no outgoing triggers (document explicitly).

**Cross-reference with Phase 1.3 (mandatory):**
Phase 1.3 captured all external imports — it is the ground truth for integrations.
After the mapping is complete: every external import from Phase 1.3 must be explained
by at least one integration entry. Any unexplained import → potential integration not
covered by the grep patterns → investigate manually before closing Phase 4.

**PROMOTE-CANDIDATE triggers (Phase 4):**
- Integration approach shared by 2+ units (same client pattern, same error-handling strategy) → `docs/ai/integration-patterns.md`
- Data flows (source, transformation, target) that are non-obvious or shared across units → `docs/ai/data-sources.md`
- Architectural decision driven by an external integration constraint → `docs/ai/architecture-decisions.md`

**Mandatory livrable** before moving to Phase 5:

```
INTEGRATION MAP — <unit>
─────────────────────────────────────────────────────
IN  : <type> <path/topic/file>  →  <DTO>  |  failure: <...>  [File:line]
IN  : <type> <path/topic/file>  →  <DTO>  |  failure: <...>  [File:line]
OUT : <type> <path/topic/file>  →  <DTO>  |  failure: <...>  [File:line]
Outgoing triggers  : <feature>  always | if <condition> | on error
Coupling violations: none | <finding>  [File:line]
Phase 1.3 coverage : all imports explained | <X> unexplained → investigated
```

### Phase 5 — Risk and Edge Case Documentation

1. **Cross-reference phases 3 and 4 first (no new reads needed)**

   **Phase 3 `[IMPLICIT]` items — validate before promoting:**
   For each `[IMPLICIT]` from Phase 3, attempt confirmation with Phase 5 greps:
   - Confirmed by grep → upgrade confidence to `[high]`, assign P0/P1/P2
   - Still unconfirmed → keep `[low]`, assign P2
   - Contradicted by grep → remove from register, note as invalidated

   A confirmed `[IMPLICIT]` touching shared state or data integrity → **P0**
   regardless of original confidence level.

   **Phase 3 coverage gaps** → promote as P1 (rule absent at entry point)

   **Phase 4 failure modes** → promote using priority scale (data corruption = P0,
   integration exception unhandled = P1, degraded mode = P2)

2. **Targeted greps**

   ```bash
   # Custom exceptions = failure modes named by the team
   grep -rn "throw new [A-Z]" src/main/<package>/ --include="*.java"

   # Swallowed exceptions (catch without log or rethrow)
   grep -rn -A3 "} catch " src/main/<package>/ --include="*.java" \
     | grep -v "log\.\|throw \|return "

   # Concurrency and shared mutable state
   grep -rn "static.*=\|synchronized\|@Async\|CompletableFuture" \
     src/main/<package>/ --include="*.java"

   # Transactions — identify overly broad scope
   grep -rn "@Transactional" src/main/<package>/ --include="*.java"
   ```

3. **Tests — method names only (edge cases documented by the team)**

   ```bash
   grep -rn "@Test\|void test\|void should\|void when" \
     src/test/ --include="*.java" | grep "<ClassName>"
   ```

4. **Git history — past incidents**

   ```bash
   git log --oneline --all -- src/main/<package>/ | head -20
   ```

**Risk priority scale:**

| Priority | Criteria |
|----------|---------|
| P0 | Data corruption risk, known prod incident, concurrency on shared mutable state |
| P1 | Coverage gap, swallowed exception, potential deadlock |
| P2 | Overly broad transaction scope, test edge case, low-impact implicit rule |

**PROMOTE-CANDIDATE triggers (Phase 5):**
- Any P0 risk → `docs/ai/known-issues.md`  [high — if grep-confirmed or prod incident]
- Operational constraint (DB, infra, external) → `docs/ai/known-issues.md` or `summary.md`
- For classification of other findings: apply Phase 3 PROMOTE-CANDIDATE routing table

**Mandatory livrable** before moving to Review Loop:

```
RISK REGISTER — <unit>
────────────────────────────────────────────────────────────────
[P0] <risk>  source: concurrency / prod incident / data corruption  [File:line]
[P1] <risk>  source: coverage gap | swallowed catch | deadlock      [File:line]
[P2] <risk>  source: transaction scope | test edge case             [File:line]
Known issues    : <commit message — date> | none
Test boundaries : <test method names revealing known edge cases>
```

---

## Review Loop (after Phase 5 or after doc= resume)

Before offering the output gate, present the full analysis in the conversation and
enter a refinement loop. The user must explicitly approve before any output is produced.

**Mandatory checks before entering the loop:**

Phase 2 placeholders:
- Resolved by Phase 3 or 4 → confirm it is filled in
- Still `?` → was it deliberately unresolvable (runtime-only, external system)?
  → Replace with `[UNRESOLVED — <reason>]`
- Still `?` with no reason → flag as coverage gap, offer to investigate before closing

Phase 3 flow boundaries:
- `?` markers in BUSINESS LOGIC livrable → list them explicitly as suggested deep-dives:
  ```
  Flow boundaries not followed — deep-dive available on request:
    → <call or type>  (<reason>, hint: <class names from imports if applicable>)
  ```
- If the user says "Deep-dive <boundary>": read the target, extend the flow, update the livrable.

Phase 5 implicites:
- `[IMPLICIT]` invalidated by Phase 5 grep → must be removed from risk register
- `[IMPLICIT]` confirmed → verify it appears in risk register with upgraded confidence and P0/P1/P2

```
ANALYSIS — <name> [<type>]
───────────────────────────
[full findings presented in conversation]

Questions, areas to deepen, corrections?
  → Ask a question                       : Claude explores and answers
  → "Deep-dive <aspect>"                 : Claude identifies which phase covers the aspect,
                                           re-runs that phase only
  → "Correct <point>"                    : Claude updates its understanding
  → "Show me the end-to-end flow
     for <feature>"                      : behavioral synthesis (see below)
  → "Show me the contract for
     <integration>"                      : formal contract (see below)
  → "Show me the risks for <unit>"       : risk register summary (see below)
  → "Done" / "output"                   : proceed to output gate
```

**On-demand behavioral synthesis** ("Show me the end-to-end flow for X") :

Synthesize from phases 1-5 already in context — do not re-read code unless a step is
missing. Start from the Phase 3 BUSINESS LOGIC livrable as skeleton: add Phase 4
outgoing triggers and Phase 5 failure modes to complete the flow. Produce:

```
Flow : <feature name>
──────────────────────────────────────
Entry : <type (SCHEDULER / API / EVENT / TRIGGERED)> — <trigger>

Steps :
  1. <description>
  2. <description>
     → If <condition> : step 3a
     → Otherwise      : step 3b
  3a. ...

Outgoing triggers :
  → <feature>  if <condition>
  → <feature>  always

[Mermaid diagram if ≥ 3 steps or conditions]
```

If a step cannot be inferred from phases 1-5: perform a targeted read, then complete.

**On-demand contract documentation** ("Show me the contract for <integration>") :

Synthesize from phases 2, 3, 4 already in context. If a field or validation is missing,
perform a targeted read of the DTO/entity class body. Produce:

```
Contract : <integration name>
──────────────────────────────────────────────────────────
Type     : <
  REST   : <METHOD> <path>  |  base-url: ${config.key}         [File:line]
  FILE   : <CSV|XML|JSON>  <encoding>  <sep=x if CSV>  |  path: ${config.key}/<pattern>  [File:line]
  MQ     : broker: ${config.key}  topic: <name>  key: <routing-key>            [File:line]
  SFTP   : host: ${config.key}  port: ${config.key}  dir: <path>  pattern: <*.ext>  [File:line]
  EMAIL  : smtp: ${config.key}  to: ${config.key}  subject: <pattern>          [File:line]
>
Direction: IN | OUT

Fields :
  <field>   <type>   MANDATORY | OPTIONAL
    → Domain model : <Phase 2 entity>.<attribute>  [File:line]
    → Validation   : <rule or annotation>          [File:line — Phase 3 if rule, Phase 2 if annotation]
    → Values       : <enum values or constraints if applicable>

  <field>   <type>   MANDATORY | OPTIONAL
    → Domain model : ?  [not resolved — not visible in Phase 2 or 3]
    ...

Failure handling : <ignore | exception <type> | fallback <strategy>>  [File:line]
```

If a field cannot be linked to Phase 2 or Phase 3: mark as `?` with reason.
Do not invent constraints not backed by a `[File:line]` reference.

**On-demand risk register** ("Show me the risks for <unit>") :

Present the Phase 5 RISK REGISTER from context. If Phase 5 was not yet run,
run it first. Produce:

```
RISK REGISTER — <unit>
────────────────────────────────────────────────────────────────
[P0] <risk>  source: <concurrency|prod incident|data corruption>  [File:line]
[P1] <risk>  source: <coverage gap|swallowed catch|deadlock>      [File:line]
[P2] <risk>  source: <transaction scope|test edge case>           [File:line]
Known issues    : <commit message — date> | none
Test boundaries : <test method names revealing known edge cases>
PROMOTE-CANDIDATEs : P0 items → docs/ai/known-issues.md
```

**This loop repeats until the user says the analysis is ready.**
Do not move to the output gate until the user explicitly signals readiness.

Signals accepted as "ready":
- "done", "ok", "output", "go", "ready", "produce output"
- Any explicit output request ("export", "save", "architecture-document")

---

## Output Gate (end of each unit analysis)

**Before presenting the gate**: check existence of each destination file referenced by
PROMOTE-CANDIDATE items. If any are absent, flag them:

```
⚠️  Missing destination files (promotion blocked for these candidates) :
  - docs/ai/domain-glossary.md      → absent — run /memory-bootstrap or create manually
  - docs/ai/integration-patterns.md → absent — same action required
```

If all destination files exist: no warning shown.

Present findings summary, then ask:

```
ANALYSIS COMPLETE — <name> [<type>]
─────────────────────────────────────
Scope       : [packages]
Key classes : [list]

Livrables :
  ORIENTATION     : <pivot>, <callers>, <pattern>
  DOMAIN MODEL    : <X entities>, <X state markers>, <X ? resolved | X unresolved>
  BUSINESS LOGIC  : <X rules>, <X entry points>, <X state transitions>
  INTEGRATION MAP : <X IN>, <X OUT>, coupling violations: <none | X>
  RISK REGISTER   : <X P0>, <X P1>, <X P2>

PROMOTE-CANDIDATEs : <X total>
  - "<item>"  → .claude/rules/<file>.md                [high — extracted from throw]
  - "<item>"  → src/.../SomeModule/CLAUDE.md           [high — grep-confirmed]
  - "<item>"  → .claude/memory-bank/systemPatterns.md  [high — confirmed in N files]
  - "<item>"  → docs/ai/domain-glossary.md             [medium — appears consistently]
  - "<item>"  → docs/ai/integration-patterns.md        [medium — observed in this unit]
  - "<item>"  → docs/ai/data-sources.md                [medium — observed in this unit]
  - "<item>"  → docs/ai/architecture-decisions.md      [medium — business policy]
  - "<item>"  → docs/ai/known-issues.md                [low — implicit, suspected from <evidence>]

What to do with these results?
  1. Capture in project memory
     → /architecture-document <name> type=<type>
  2. Export to folder
     → specify target path (or use output=<path>)
  3. Keep in conversation only
     → nothing written

⚠️  Only option 1 feeds architecture-map.md and makes results available
    for future sessions on this repo.

(No response → option 3 by default)
```

### Option 1 — Memory (via `/architecture-document`)

Do not invoke `/architecture-document` automatically.
Provide the pre-filled command for the user to run:

```
→ Run : /architecture-document <name> type=<type>
```

### Option 2 — Folder export (or file update)

**If `doc=` was not provided**: ask for target path, then write a single file:

```
<path>/<name>-analysis.md     → full findings (phases 1-5) + PROMOTE-CANDIDATE
```

The file must include a `## PROMOTE-CANDIDATE` section — even if empty.
PROMOTE-CANDIDATE items identified during the analysis must be written into this section,
not left only in the conversation.

```markdown
## PROMOTE-CANDIDATE
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
(empty if no candidates identified)
```

The exported file can be used as input for future sessions:
```bash
/codebase-analysis doc=<path>/<name>-analysis.md      # continue refining
/architecture-document doc=<path>/<name>-analysis.md  # capture in memory when ready
```

**If `doc=` was provided**: offer to update the same file. Merge new findings into the
existing content. For `## PROMOTE-CANDIDATE`: merge with the existing section — do not
replace. Add new candidates, keep existing ones intact.

### Option 3 — Conversation only

Nothing is written. Acknowledge:
```
Results kept in conversation only. Not persisted.
```

### Update analysis plan (mandatory if plan exists)

If `docs/ai/analysis-plan-<repo-name>.md` exists, update it **without asking** after
every output gate. This is the only way sessions 2, 3, etc. know what has been done.

Update the unit status:
- Option 1 → `✅ documented`
- Option 2 → `📁 exported`
- Option 3 → `💬 conversation only`

Append to the Sessions table:
```
| <today> | <name> [<type>] | <option chosen> |
```

If plan does not exist, offer to create it (one-time prompt).

### User-controlled status changes

At any point — in Phase 0 or after an output gate — the user can change a unit's status:

```
"Mark <name> as ignored"        → 🚫 ignored   (skip in future sessions)
"Mark <name> as to do"          → ⬜ to do      (reset — will be proposed again)
"Mark <name> as not to process" → 🚫 ignored
```

Units marked `🚫 ignored` are shown in the plan but never proposed for analysis.
The user must explicitly say "analyze <name> anyway" to override.

### User-controlled parameter overrides

The user can change scope, freedom, objective, or context for any unit **at any point**:
during Phase 0, before phases start, during analysis, or at the start of a new session.

```
"Change the objective of <name> to <objective>"
"Change the scope of <name> to <list>"
"Change the freedom of <name> to exclusive/indicative"
"Modify the context of <name>: <text>"
"Add to the context of <name>: <text>"
```

Or without naming the unit when context is clear (current unit in progress):
```
"Change the objective to <objective>"
"Scope: only <list>"
"Context: <text>"
```

On receiving any override:
1. Apply immediately — adjust the current or next phase accordingly.
2. Update `## Analysis details` in the plan **without asking** (same rule as status updates).
3. Acknowledge: "Objective updated: <new>. Adapting phases accordingly."

**If analysis is already in progress**: apply the override to all remaining phases.
Already-completed phases are not re-run unless the user asks.

**Cross-session**: overrides written to `## Analysis details` in the plan are shown
before phases start (pre-analysis questions step) at the start of the next session.

### User-added units

The user can add a unit not discovered during cartography:

```
"Add <name> [<type>] [<risk>]"
```

When a unit is added manually, ask 4 clarifying questions before adding it to the plan:

```
Unit to add: <name> [<type>] <risk>

A few questions to guide the analysis:

1. SCOPE — Packages or classes to analyze?
   → list (e.g. "service/cache, adapter/config/cache")
   → "nothing specific" — I will search the project myself

2. FREEDOM — If scope provided, it is:
   → "exclusive"   : I search only within these packages
   → "indicative"  : starting point, I explore dependencies if relevant

3. OBJECTIVE — Why this analysis?
   a) Modify / extend this unit
   b) Debug / investigate an issue
   c) Understand the architecture before a task
   d) Prepare a refactoring
   e) Document for the team
   → or describe your objective freely

4. KNOWN CONTEXT — What you already know:
   → key classes, expected behavior, suspected issues, entry points...
   → "nothing" if starting from scratch
```

If no scope is provided (question 1 = "nothing specific"): before Phase 1, search for
the unit name in class names and imports to find a natural anchor:
```bash
grep -rn "<name>" src/main/ --include="*.java" -l | head -20
```

### Pre-analysis questions (all units)

Before starting phases 1-5 on **any** unit (discovered or manual):

1. Read `## Analysis details` in the plan for this unit (if plan exists).
2. **If parameters are found**: display them and ask "Confirm or modify?" — do not
   re-ask from scratch.
   ```
   Saved parameters for <name>:
     Objective: b) Debug / investigate
     Scope    : service/cache, adapter/config/cache
     Freedom  : indicative
     Context  : CommandPersistService, nightly scheduler

   Confirm or modify before starting?
   ```
3. **If no parameters stored**: ask questions 3 and 4. For manually added units, all 4
   questions apply.
4. After confirmation: write/update `## Analysis details` in the plan immediately.
   **If no plan exists**: keep parameters in conversation only. Offer to create the plan
   to persist them: "Save the plan to keep these parameters between sessions?"

**How the objective calibrates the phases:**

| Objective | Priority phases | Light phases |
|-----------|----------------|--------------|
| Modify / extend | 2 (domain model) + 5 (risks) | 4 light |
| Debug / investigate | 3 (business logic) + 5 (risks) | 1 light |
| Understand architecture | 1 + 2 | 3, 5 light |
| Prepare a refactoring | 1 + 2 + coupling | 3 light, 5 light |
| Document for the team | All | — |
| Free objective | Adapt based on description | — |

"Light" = run the phase but stop at surface level — do not read individual class bodies.

---

## See also

- [patterns.md](patterns.md) — architectural patterns to identify
- `.claude/skills/architecture-document/SKILL.md` — how to capture results in project memory
- `.claude/rules/` — existing project rules to cross-check against findings
- `docs/ai/architecture-map.md` — units already documented
