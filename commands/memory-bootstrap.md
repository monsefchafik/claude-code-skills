---
description: bootstrap the full claude code project memory system for an existing repository
argument-hint: [goal or scope]
---

You are bootstrapping the Claude Code memory system for this repository.

User intent:
$ARGUMENTS

## Mission

Initialize a complete, maintainable, enterprise-grade project memory system for this repository.

Target workflow:
1. `/init` should have been run already if possible
2. analyze the repository
3. propose the memory file plan
4. **ask the user to confirm the plan before writing anything**
5. generate the files
6. run a critical audit on the generated memory
7. leave clear sections for human completion
8. propose real-task tests
9. prepare the repository for continuous memory maintenance

If `CLAUDE.md` does not exist yet, say so clearly and continue with a bootstrap-equivalent plan instead of blocking.

## Operating principles

Always optimize for:
- concise, high-signal memory
- low ambiguity
- no contradictions
- correct placement of information by memory layer
- enterprise maintainability
- minimal duplication
- safe defaults for existing codebases

Do not:
- invent hidden business rules
- claim repository facts that are not grounded in visible files
- generate large generic filler sections
- create memory files without a clear purpose
- overuse local `CLAUDE.md` files
- invent filenames in `.claude/memory-bank/` — only use files that already exist or the three standard ones (`activeContext.md`, `progress.md`, `systemPatterns.md`)
- place `claude-files-registry.md` inside `.claude/memory-bank/` — it belongs at `.claude/claude-files-registry.md`

## Required target architecture

Unless the repository strongly suggests a simpler structure, target this memory architecture:

- `CLAUDE.md`
- `docs/ai/`
    - adapt filenames to what already exists in the project — do not invent new names that conflict with existing files
    - typical files: `known-issues.md`, `architecture-decisions.md`, `domain-glossary.md`, `integration-patterns.md`, `runbook-dev.md`
    - only create a file if no existing file already covers that content
- `.claude/rules/`
    - adapt to what already exists — typical files: `database.md`, `security.md`, `java-spring.md`, `hexagonal-architecture.md`
    - only create files for cross-cutting concerns not already covered
- `.claude/memory-bank/`
    - exactly three standard files: `activeContext.md`, `progress.md`, `systemPatterns.md`
    - **bootstrap responsibility**: create these files populated with real project content derived from Step 1 analysis
    - **template structure**: use `.claude/skills/memory-maintainer/references/templates/` as structural skeleton — do not deviate from the sections defined there
    - do not create `projectbrief.md`, `techContext.md`, or any other non-standard file
- `~/.claude/projects/<encoded-project-path>/memory/MEMORY.md` — auto-memory index; mention it exists but never edit it directly
- local `CLAUDE.md` files only where directory-specific durable rules are clearly justified and not already present

**Sequencing with `memory-maintainer`**: bootstrap runs once and populates all layers with real content. After bootstrap, `memory-maintainer` takes over for all ongoing maintenance (audits, patches, promotions, lifecycle). These two tools are sequential, not competing — bootstrap initializes, memory-maintainer maintains.

## Memory layer responsibilities

Use this routing model:

- root `CLAUDE.md` → global durable behavioral rules for Claude in this repo
- local `CLAUDE.md` → durable rules tied to a directory subtree
- `.claude/rules/` → specialized reusable rules that should not bloat `CLAUDE.md`
- `docs/ai/` → durable explanatory knowledge (architecture, domain, known issues, decisions)
- `.claude/memory-bank/` → evolving project context: current work (`activeContext`), milestone log (`progress`), recurring patterns (`systemPatterns`)
- auto-memory (`/memory` or `~/.claude/projects/.../memory/`) → observational input only, never edit directly

## Process

Follow these steps in order.

### Step 1 - Repository analysis

Execute each action below in order. Do not skip any. Mark each finding as **confirmed** (read directly from a file or command output) or **inferred** (deduced from structure or naming).

#### 1.1 — Stack and build
- Read `pom.xml` or `build.gradle` → extract exact language version, framework version, key dependencies
- Read `README.md` if present → extract build, test, and run commands
- If no README, look for `Makefile`, `.mvn/`, `gradlew` → infer commands

#### 1.2 — Repository structure
- List top-level directories
- List `src/main/java` subdirectories (or equivalent) to map backend/domain/infra areas
- Identify high-risk domains by name: auth, billing, payment, admin, migration, security, partner

#### 1.3 — Existing memory files (mandatory — list every file found)
- List all files in `docs/ai/` — read each one to understand current content and avoid duplication
- List all files in `.claude/rules/` — read each one
- List all files in `.claude/memory-bank/` — note which of `activeContext.md`, `progress.md`, `systemPatterns.md` are missing
- Read root `CLAUDE.md` if present
- List any local `CLAUDE.md` files in subdirectories
- Read `.claude/settings.json` and `.claude/settings.local.json` if present — note any hooks

#### 1.4 — Git history (for progress.md population)
- Run `git log --oneline -20` → identify the 5–10 most significant recent milestones
- Run `git branch -r` or `git log --merges --oneline -10` → identify recently merged workstreams
- Note ticket/PR references found in commit messages

#### 1.5 — Code patterns (for systemPatterns.md population)

Use the directory structure found in 1.2 to identify the right paths. For a Java/Spring hexagonal project, typical starting points are `adapter/out/persistence/`, `application/service/`, and `adapter/in/web/`. For other stacks, look for the equivalent persistence, service, and controller layers.

Read 3–5 files across at least two distinct layers. For each file, look explicitly for:

- **Entity ↔ domain mapping**: how are entities converted to/from domain objects? Is there a static factory method pattern (`fromDomainModel`, `toDomainModel`)? Present in 2+ files?
- **Error handling**: how are business errors collected and returned? Is there a shared error type or builder pattern? Same approach in 2+ files?
- **External calls**: how are HTTP clients or external APIs invoked? Retry, timeout, or fallback patterns? Present in 2+ files?
- **Transaction boundaries**: where are `@Transactional` annotations placed? Read-only vs write split? Consistent across services?
- **saveAll / bulk persistence**: any sorting or ordering before batch writes? Present in 2+ files?

For each pattern found in 2+ distinct files: note it as a confirmed candidate for `systemPatterns.md`.
For each pattern found in only 1 file: note it as unconfirmed — do not promote.

Note the dominant language used in existing memory files (French, English, mixed).

#### 1.6 — Memory gaps
- Based on 1.1–1.5, list what is currently undocumented that Claude will need to work effectively on this project

### Step 2 - File plan

Before creating any files, produce a plan with:
- file path
- purpose
- why it is needed (what gap it fills)
- whether content is inferred automatically or needs human completion
- for each `docs/ai/` file: confirm no existing file already covers the same content

Do not skip this step.

**After presenting this plan, ask the user: "Shall I generate these files? (yes / no / adjust)"**
Do not proceed to Step 3 without explicit confirmation.

### Step 3 - Generate memory files

Generate only the files confirmed in Step 2. Work in two phases.

#### Step 3a — Layers 1 and 2 (rules + explanatory knowledge)

Generate: `CLAUDE.md`, `docs/ai/*`, `.claude/rules/*`, local `CLAUDE.md` files.

For every file:
- write directly usable content grounded in Step 1 findings
- keep it concise and high-signal
- include an `To be completed` section when human input is needed
- explain what should be filled in
- add a short realistic example where useful

Writing rules:
- `CLAUDE.md` and `.claude/rules/*` must be prescriptive (instructions to Claude)
- `docs/ai/*` must be explanatory (context and rationale for Claude)
- match the dominant language already used in existing memory files for the same layer
- avoid repeating the same rule across multiple layers

**Three-tier loading strategy — mandatory in root `CLAUDE.md`**:

Root `CLAUDE.md` must enforce the following import structure. Do not deviate from it.

Tier 1 — @imports (always loaded, only if the file exists):
```
@.claude/memory-bank/activeContext.md
@.claude/memory-bank/systemPatterns.md
@docs/ai/known-issues.md
```
If a Tier 1 file does not exist yet at generation time, add a placeholder `To be completed` marker instead
of a broken @import. `memory-maintainer` will add the @import once the file exists.

Tier 2 — On-demand references (root `CLAUDE.md` must list these in a dedicated section):
- `.claude/memory-bank/progress.md` — delivered milestones, avoid re-proposing work already done
- `docs/ai/architecture-decisions.md` — architecture decisions and trade-offs
- `docs/ai/domain-glossary.md` — business/domain terms
- `docs/ai/integration-patterns.md` — patterns for new integrations
- `docs/ai/data-sources.md` — data flows and data sources

List only files that exist. Do not create placeholder entries for missing files.

Tier 3 — Never import directly (heavy architecture docs, accessed via `architecture-reviewer` subagent):
- `docs/ai/generators-architecture.md`
- `docs/ai/validation-pipelines.md`
- Any other full-analysis or summary architecture file

Never add these as @imports in `CLAUDE.md`.

#### Step 3b — Local CLAUDE.md registry

After discovering local `CLAUDE.md` files in Step 1.3 and generating or confirming them in Step 3a:

1. Check whether `.claude/claude-files-registry.md` exists. **This file is at `.claude/`, never inside `.claude/memory-bank/`.**

2. If it does **not** exist — create it with:
   - One row per local `CLAUDE.md` file discovered or generated
   - For each row, a routing criterion derived from the Step 1 analysis:
     *"Route here when rule applies only to [specific module or responsibility]"*
   - The criterion must be specific enough to distinguish this file from root `CLAUDE.md`
     and from `.claude/rules/` (which are cross-cutting)

3. If it already exists — compare against the filesystem:
   - Local `CLAUDE.md` exists but not in registry → add path only (no routing criteria)
   - Registry entry references a file that no longer exists → propose removal

4. Include the registry creation or update in the Step 2 file plan and Step 5 output.

**Do not overload routing criteria** — one concise line per entry. The criterion answers
"what makes a rule belong here instead of root `CLAUDE.md` or `.claude/rules/`?"

#### Step 3c — Layer 3 (memory-bank)

Generate: `activeContext.md`, `progress.md`, `systemPatterns.md`.

**This is the bootstrap's unique value**: `memory-maintainer` only creates empty templates for these files. Bootstrap must populate them with real project content derived from Step 1.

For each file:
- read the canonical template from `.claude/skills/memory-maintainer/references/templates/` to get the exact section structure
- preserve that structure exactly — do not add or remove sections
- populate every section with real project data:
  - `activeContext.md`: fill Current Focus with active work visible in git log or recent branches; fill Active Risks with known issues from docs; leave Pending Decisions if genuinely unknown
  - `progress.md`: populate the log table with significant recent milestones found in git log (last 5–10 commits or merged branches)
  - `systemPatterns.md`: populate each section with patterns confirmed in 2+ places in the codebase (from Step 1 code scan)
- if a section has no grounded content, mark it `<!-- To be completed -->` rather than inventing content
- do not exceed ~15 lines in `activeContext.md`

### Step 4 - Critical audit of generated memory

Immediately audit what you generated using the routing rules from `references/promotion-matrix.md` if available.

Check for:
- generic filler with no behavioral impact
- ambiguity or vague language ("if needed", "be careful")
- contradictions between files or layers
- duplication of content already present in existing files
- wrong-layer placement (e.g., explanatory content in `CLAUDE.md`, prescriptive rules in `docs/ai/`)
- excessive verbosity
- unsupported claims not grounded in code or confirmed facts
- missing sections that should be explicitly marked for human completion
- naming conflicts with existing `docs/ai/` files

Then revise the generated files to improve signal density.

### Step 5 - Human completion checklist

Produce a short checklist of what a human should complete next, especially:
- business invariants not derivable from code
- domain glossary terms
- operational pitfalls
- architectural decisions and their rationale
- security or compliance rules
- current project priorities

### Step 6 - Real-task validation plan

Propose 3 to 5 real tasks to test whether the memory system is working well.

Examples:
- explain how module X works
- propose a safe refactor in Y
- analyze impact of changing Z
- fix a realistic bug in a critical module
- identify the right place to implement a business rule

### Step 7 - Continuous maintenance readiness

Recommend how to maintain this memory system over time:

- **After each significant session**: run `/promote-to-memory` to extract durable decisions into versioned memory
- **After each merged PR or milestone**: rotate the entry from `activeContext.md` to `progress.md` using `memory-maintainer` in apply mode
- **Every 2–4 weeks**: run a full audit with `memory-maintainer` (`mode: audit, scope: full, include_auto_memory: true`) to catch staleness, undocumented patterns, and stale `activeContext` entries
- **When starting a new workstream**: update `activeContext.md` via `memory-maintainer` in patch mode
- **What to promote**: explicit decisions, confirmed patterns (two-occurrence rule), known issues, completed milestones
- **What not to promote**: speculation, one-off observations, content already in versioned memory, content derivable by reading the code

## Output format

Always structure your response using these sections:

# 1. Repository analysis
- confirmed findings
- inferred findings
- existing memory files (listed explicitly)
- major risks
- current memory gaps

# 2. Proposed memory plan
For each file:
- path
- role
- source of truth: confirmed | inferred | human input needed

*(pause and ask for confirmation before continuing)*

# 3. Generated memory summary
Summarize what was created or updated and why.

# 4. Critical audit
List problems found in the initial generation and how you corrected them.

# 5. Human completion checklist
A short actionable checklist.

# 6. Real-task validation plan
List 3 to 5 concrete validation tasks.

# 7. Continuous maintenance guidance
Give concise rules for keeping the memory system healthy, referencing `/promote-to-memory` and `/memory-maintainer` with their specific use cases.

## Important behavior

If the user asks to iterate after generation, treat the memory as code:
- refine specific sections
- move content to the correct layer
- shorten noisy sections
- preserve accepted wording when the user provides replacements
- do not re-bootstrap everything unless asked

If the repository already has a memory system, do not overwrite blindly.
Instead:
- analyze the existing structure
- reuse what is good
- patch what is weak
- only add missing layers where justified
