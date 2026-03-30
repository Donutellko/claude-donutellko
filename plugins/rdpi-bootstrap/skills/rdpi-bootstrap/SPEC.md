# RDPI Bootstrap — Specification

**Date**: 2026-03-30
**Status**: Draft
**Authors**: Donat + Claude

---

## Overview

`rdpi-bootstrap` is a meta-skill that generates a project-specific set of RDPI (Research → Design → Plan → Implement) phase skills when invoked on a new codebase. It ensures that AI-assisted development follows context engineering best practices: clean context per phase, structured artifact handoff, multi-perspective review, and iterative delivery.

The meta-skill itself follows RDPI: it interviews the user, saves a spec (`./rdpi/RPI_SKILLS_SPEC.md`), then generates the skills. On re-run, it reads the existing spec and discusses updates with the user before refining the generated skills.

---

## Problem Statement

When developers use AI on a new project, they typically fall into the "prompt → code → fix → repeat" anti-pattern. The RDPI methodology (from Dex Horthy's "Context Engineering" and Dmitry Bereznitsky's "Process over Prompts") solves this — but setting it up for each project requires significant manual effort: understanding the codebase, choosing the right agents, configuring phase workflows, and defining artifact formats.

`rdpi-bootstrap` automates this setup: one command produces a full set of project-tailored phase skills ready for use.

---

## Core Principles

### 1. Clean Context Between Phases

Each RDPI phase runs in a fresh Claude session. The only input is the artifact(s) from the previous phase. This prevents the "dumb zone" problem (context degradation at ~40% window fill) described by Horthy.

### 2. Artifacts as the Single Source of Truth

Every phase produces a structured markdown artifact. The artifact is the contract between phases — not the conversation history.

### 3. Runtime Agent Discovery

The meta-skill does NOT hardcode specific agents or skills. At execution time, it discovers what agents and skills are available in the current environment and assigns them to roles. This makes it portable across different plugin setups.

### 4. "Go Back and Forth With Me"

Both the meta-skill interview and the generated Research/Spec phase skill must ensure mutual understanding through iterative dialogue. Claude should not assume — it should confirm.

### 5. Recommended Options with Annotations

When using `AskUserQuestion`, always mark the recommended option and annotate choices with tags like `(recommended)`, `(fast)`, `(risky)`, `(safe)`, `(compromise)`.

---

## What rdpi-bootstrap Produces

### Input
- User answers to interview questions
- Codebase analysis (automatic)
- Available agents/skills inventory (automatic)

### Output

```
./rdpi/RPI_SKILLS_SPEC.md          — Spec of user's preferences and project context
./rdpi/skills/
├── rdpi-research/SKILL.md         — Research phase skill
├── rdpi-design/SKILL.md           — Design phase skill
├── rdpi-plan/SKILL.md             — Plan phase skill
├── rdpi-implement/SKILL.md        — Implement phase skill
└── README.md                      — Usage guide with phase order and commands
```

Each generated skill:
- Knows how to execute its phase
- Writes its artifact to the RPI working directory
- Ends with the exact command to invoke the next phase
- Is tailored to the project's tech stack, conventions, and available agents

---

## RDPI Phases (Generated Skills)

### Phase 1: Research (`/rdpi-research`)

**Goal:** Build a compressed map of the relevant parts of the system.

**Input:** User description of the task (feature or bug) + task ID (Jira/Rally/GitHub issue).

**Process:**
1. Interview the user about the task using `AskUserQuestion` — go back and forth to ensure mutual understanding
2. Explore the codebase: find relevant files, patterns, conventions, dependencies
3. If bug: focus on reproduction steps, root cause analysis, affected areas
4. If feature: focus on related existing implementations, integration points, data flow
5. Generate a structured Research artifact (facts only, no recommendations)

**Output artifact:** `./rdpi/{folder}/01-research/research.md`

**Review:** Discovered sub-agents with relevant capabilities review the research for completeness and blind spots.

**Ends with:** `Next: run /rdpi-design ./rdpi/{folder}`

### Phase 2: Design (`/rdpi-design`)

**Goal:** Define what to build and how, with architecture decisions locked down before any code is written.

**Input:** Research artifact from Phase 1.

**Adaptive depth** (determined automatically from Research):

| Depth | When | Includes |
|-------|------|----------|
| **Light** | Small fixes, simple bugs | ADR + brief change description |
| **Standard** | Typical features | C4 component diagram + API contracts + test strategy + ADR |
| **Full** | Cross-module changes, new subsystems | All of Standard + sequence diagrams + data flow + security review |

**Multi-perspective design** (for Standard and Full):

Three agents with different "characters" independently produce design proposals:
1. **Conservative** — minimal changes, maximum safety, proven patterns
2. **Pragmatist** — balanced approach, "good enough" for the timeline
3. **Harsh Critic** — based on devil's advocate, challenges every assumption, finds weaknesses

Then a synthesis step merges the best ideas into a final design. The individual proposals and synthesis rationale are preserved as artifacts.

**Iterative delivery increments:**

For complex tasks, Design must break work into delivery increments where each increment is **demonstrable** to the user. The principle: start with the part that the user can see and give feedback on (with stubs for dependencies), iterate on it, then implement the dependencies.

Example patterns (adapted to the project):
- Frontend-first: UI with mocks → collect feedback → backend + integration
- API-first: Contract + stub server → consumer integration → real implementation
- Data-first: Schema + seed data → processing logic → presentation layer

The specific pattern is chosen based on the project's architecture discovered during Research.

**Diagrams:** Use Mermaid or PlantUML depending on what the project already uses. If neither is established, default to Mermaid.

**Output artifact:** `./rdpi/{folder}/02-design/`
```
02-design/
├── design.md                    — Final synthesized design (USER REVIEWS THIS)
├── design-conservative.md       — Conservative perspective
├── design-pragmatist.md         — Pragmatist perspective
├── design-critic.md             — Harsh critic perspective
├── synthesis-rationale.md       — Why specific choices were made
├── review-{agent-name}.md       — Reviews by discovered agents
└── c4-diagrams.md               — Architecture diagrams (USER REVIEWS THIS)
```

**User review points:**
- The spec/design document — to confirm mutual understanding
- C4 diagrams — to confirm architecture direction
- Delivery increment breakdown — to confirm iteration order

**Ends with:** `Next: run /rdpi-plan ./rdpi/{folder}`

### Phase 3: Plan (`/rdpi-plan`)

**Goal:** Convert Design into a concrete, phased execution plan.

**Input:** Design artifact from Phase 2.

**Process:**
1. Read the design and delivery increments
2. For each increment, create implementation phases with:
   - Exact file paths (create/modify)
   - Verification steps (build commands, test commands)
   - Commit messages
3. Each phase must be independently committable and verifiable
4. Identify which phases can run in parallel (e.g., frontend + backend after contracts are locked)

**Output artifact:** `./rdpi/{folder}/03-plan/plan.md`

**Review:** Discovered review agents check for feasibility and completeness.

**The user does NOT review the full plan.** They already reviewed the Design. The Plan is a mechanical decomposition — it's an implementation detail.

**Ends with:** `Next: run /rdpi-implement ./rdpi/{folder}`

### Phase 4: Implement (`/rdpi-implement`)

**Goal:** Execute the plan, phase by phase.

**Input:** Plan artifact from Phase 3 + Design artifact for reference.

**Process:**
1. If a multi-agent implementation skill (like `implement-spec`) is available, delegate to it
2. Otherwise, execute plan phases sequentially, committing after each
3. After each delivery increment, prompt the user to test/review
4. If the user requests changes, update the spec/design/plan as needed and continue

**Output artifact:** `./rdpi/{folder}/04-implement/implementation-log.md`

**Code review:** Discovered code review agents review the implementation.

**Output:** `./rdpi/{folder}/04-implement/review-{agent-name}.md` (USER REVIEWS THIS)

---

## Artifact Directory Structure

```
./rdpi/{YYYY-MM-DD}-{task-id}-{short-name}/
├── 01-research/
│   ├── research.md
│   └── review-{agent}.md
├── 02-design/
│   ├── design.md                    ← USER REVIEW POINT
│   ├── c4-diagrams.md               ← USER REVIEW POINT
│   ├── design-conservative.md
│   ├── design-pragmatist.md
│   ├── design-critic.md
│   ├── synthesis-rationale.md
│   └── review-{agent}.md
├── 03-plan/
│   ├── plan.md
│   └── review-{agent}.md
├── 04-implement/
│   ├── implementation-log.md
│   └── review-{agent}.md           ← USER REVIEW POINT
└── MANIFEST.md                      — Navigation index with status of each phase
```

The `{folder}` format defaults to `{YYYY-MM-DD}-{task-id}-{short-name}`. The meta-skill asks the user for their preferred format during bootstrap interview.

---

## Meta-Skill Workflow (rdpi-bootstrap itself)

### First Run

1. **Check for existing spec:** Look for `./rdpi/RPI_SKILLS_SPEC.md`
   - If found → go to Re-run flow
   - If not found → continue with First Run

2. **Check for existing project-specific skills:** Look for skills like `implement-feature`, `implement-story`, or any skill that reflects project specifics. Extract useful context from them.

3. **Discover available agents and skills:** Enumerate all agents and skills available in the current environment. Catalog their capabilities for assignment to RDPI phases.

4. **Interview the user** using `AskUserQuestion` — go back and forth:
   - What types of tasks will you use RDPI for? (feature impl, bug fixing, both)
   - Artifact folder format — default: `./rdpi/{YYYY-MM-DD}-{task-id}-{short-name}`
   - Do you create pull requests as part of your workflow? At which point?
   - What's the project's tech stack? (also verify by scanning codebase)
   - What's the team's preferred diagram format? (Mermaid / PlantUML / auto-detect)
   - Is there a CI/CD pipeline? What are the build/test commands?
   - Are there existing coding standards or CLAUDE.md instructions to incorporate?
   - What task tracker do you use? (Jira / Rally / GitHub Issues / none)
   - How many people will review the output? (solo dev / team)

5. **Save spec:** Write answers to `./rdpi/RPI_SKILLS_SPEC.md`

6. **Propose agent assignments:** Based on discovered agents, propose which agents handle which RDPI roles:
   - Research review agents
   - Design multi-perspective agents (conservative, pragmatist, critic)
   - Plan review agents
   - Code review agents
   - Present the mapping to the user for confirmation

7. **Generate phase skills:** Create the 4 RDPI skills + README in `./rdpi/skills/`
   - Each skill reads its phase template from the meta-skill's bundled references
   - Each skill is customized with project-specific: paths, build commands, conventions, assigned agents

8. **Output:** Display what was created and how to start using it

### Re-run

1. **Read `./rdpi/RPI_SKILLS_SPEC.md`**
2. **Ask the user** what they want to update — using `AskUserQuestion`
3. **Re-discover agents** (new ones may have been added)
4. **Update the spec** based on discussion
5. **Regenerate/refine the phase skills** to reflect changes
6. **Show diff** of what changed

---

## Scope

### In Scope
- Feature implementation workflow (RDPI)
- Bug fixing workflow (adapted RDPI — lighter Design, focused Research)
- Generating project-specific RDPI skills
- Runtime discovery and assignment of available agents
- Multi-perspective design review
- Iterative delivery increment planning

### Out of Scope (for now)
- Refactoring-only workflows
- Migration workflows
- Incident investigation
- Running the RDPI phases itself (that's what the generated skills do)

---

## Dependencies

### Required
- `AskUserQuestion` tool — for interview flow
- `Agent` tool — for spawning sub-agents (multi-perspective design, reviews)
- File system tools (Read, Write, Glob, Grep) — for codebase analysis and skill generation

### Optional (discovered at runtime)
- `implement-spec` or similar — delegated to during Implement phase
- `interview` skill — can be used/adapted for Research phase interview
- `devils-advocate` agent — assigned to Harsh Critic role
- `architecture-reviewer`, `clean-code-reviewer`, `regression-reviewer`, `linebyline-reviewer` — assigned to review roles
- Any other agents/skills found in the environment

---

## AskUserQuestion Format Convention

All questions must follow this pattern:

```json
{
  "question": "Clear question text with context",
  "options": [
    {"label": "Option A (recommended)", "description": "Why this is the default choice"},
    {"label": "Option B (fast)", "description": "Trade-off explanation"},
    {"label": "Option C (thorough)", "description": "When this makes sense"}
  ]
}
```

Rules:
- Always mark one option as `(recommended)` — Claude's best judgment given context
- Annotate other options: `(fast)`, `(risky)`, `(safe)`, `(thorough)`, `(compromise)`, `(minimal)`
- The tool automatically adds an "Other" option for custom input
- Never ask about things already clear from the conversation or codebase

---

## Open Questions

- Should the generated skills be registered in a plugin manifest, or is file-based discovery sufficient?
- Should rdpi-bootstrap generate a `.claude/settings.json` entry to auto-register the generated skills?
- What happens when a phase skill is invoked but previous phase artifacts are missing — error, or offer to run the missing phase?
