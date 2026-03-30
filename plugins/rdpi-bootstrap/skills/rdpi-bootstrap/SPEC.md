# RDPI Bootstrap — Specification

**Date**: 2026-03-30
**Status**: Draft v3
**Authors**: Donat + Claude
**Based on**: Dex Horthy "Context Engineering" (QRSPI update), Dmitry Bereznitsky "Process over Prompts"

---

## Terminology

| Term | Meaning |
|------|---------|
| **QRSPI** | The full methodology: Questions → Research → (Spec) → Design → Structure → Plan → Implement. Originated from Dex Horthy's updated workflow (sometimes transcribed as "CRISPY"). |
| **RDPI** | Our 4 top-level skills: Research, Design, Plan, Implement. Each skill internally implements multiple QRSPI stages. The 4-skill split exists for clean context separation and UX simplicity. |
| **Phase** | One of the 4 RDPI skills. Each phase runs in a fresh Claude session. |
| **Artifact** | A structured markdown file produced by a phase, serving as the contract for the next phase. |
| **Vertical slice** | An end-to-end piece of functionality (not a horizontal layer like "all backend"). |

### Why 4 Skills, Not 8 Stages?

QRSPI defines 8 logical stages. We collapse them into 4 skills because:
1. **Context isolation where it matters most.** The biggest context-pollution risk is between Research→Design→Plan→Implement. Within a phase, sub-agents provide additional isolation (e.g., blind Research sub-agent runs in its own context).
2. **UX simplicity.** 4 commands to remember, not 8. Each command does a coherent chunk of work.
3. **The orchestrator overhead is manageable.** Within Research, the main agent holds interview context + spawns a blind sub-agent. The interview is user input (high-signal), not accumulated LLM output (low-signal noise).

---

## Overview

`rdpi-bootstrap` is a meta-skill that generates a project-specific set of RDPI phase skills when invoked on a new codebase. It ensures that AI-assisted development follows context engineering best practices: clean context per phase, structured artifact handoff, non-biased research, and iterative delivery.

The meta-skill itself follows RDPI: it interviews the user, saves a spec (`./rdpi/RPI_SKILLS_SPEC.md`), then generates the skills. On re-run, it reads the existing spec and discusses updates before refining the generated skills.

---

## Problem Statement

When developers use AI on a new project, they typically fall into the "prompt → code → fix → repeat" anti-pattern. The QRSPI methodology solves this — but setting it up for each project requires significant manual effort: understanding the codebase, choosing the right agents, configuring phase workflows, and defining artifact formats.

`rdpi-bootstrap` automates this setup: one command produces a full set of project-tailored phase skills ready for use.

---

## Core Principles

### 1. Clean Context Between Phases

Each RDPI phase runs in a fresh Claude session (user clears context between phases). The only input is the artifact(s) from the previous phase. This prevents the "dumb zone" problem (context degradation at ~40% window fill).

### 2. Artifacts as the Single Source of Truth

Every phase produces a structured markdown artifact. The artifact is the contract between phases — not the conversation history.

### 3. Runtime Agent Discovery

The meta-skill does NOT hardcode specific agents or skills. At execution time, it discovers what agents and skills are available in the current environment and assigns them to roles. This makes it portable across different plugin setups.

### 4. "Go Back and Forth With Me"

Both the meta-skill interview and the generated Research phase skill must ensure mutual understanding through iterative dialogue. Claude should not assume — it should confirm.

### 5. Recommended Options with Annotations

When using `AskUserQuestion`, always mark the recommended option and annotate choices with tags like `(recommended)`, `(fast)`, `(risky)`, `(safe)`, `(compromise)`.

### 6. Instruction Budget

Each generated SKILL.md must stay under 500 lines. If it grows beyond that, extract content into `references/` files that are read on demand. The underlying principle: ~150-200 actionable instructions is the sweet spot. More than that degrades model performance.

### 7. Non-biased Research

Research sub-agents operate WITHOUT knowledge of the specific task. They receive only questions about the codebase and return facts. This prevents confirmation bias. Note: true blindness is impossible since the questions themselves encode task assumptions. The goal is to prevent the research agent from selectively finding evidence that supports a pre-determined solution. The research agent should also report unexpected findings beyond the questions asked.

### 8. Vertical Slicing

Implementation is organized as end-to-end vertical slices, not horizontal layers. Instead of "all backend, then all frontend", each slice delivers a complete piece of functionality that can be demonstrated and verified.

### 9. No Magic Prompts

Skills should work through pipeline structure and context quality, not through prompt engineering tricks ("think step by step", "you are an expert", "take a deep breath"). If a skill only works with specific phrasing, the design is wrong. The pipeline IS the control mechanism.

### 10. Pipeline as Control

The sequence of phases, context boundaries, and artifact contracts are the control mechanism. Individual skill prompts should be simple and direct. Control through code/pipeline, not through prompt.

### 11. Bad Trajectory Recovery

If the conversation within a phase goes sideways (user has corrected the agent 2-3+ times in a row), the skill should suggest restarting the phase with a fresh context rather than continuing to accumulate a bad trajectory.

### 12. Human Reads Code

The user is expected to read and verify code — not just trust artifacts. During Research, the user should spot-check key code paths identified by the research agent. Code review happens before PR creation (in a Draft PR or locally).

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
├── rdpi-research/SKILL.md         — Research phase skill (Questions + blind Research + Spec)
├── rdpi-design/SKILL.md           — Design phase skill (Design doc + Structure/Outline)
├── rdpi-plan/SKILL.md             — Plan phase skill (parallel plans per role/phase)
├── rdpi-implement/SKILL.md        — Implement phase skill (orchestrator + sub-agents + PR + deploy)
└── README.md                      — Usage guide with phase order and commands
```

Each generated skill:
- Knows how to execute its phase
- Writes its artifact to the RDPI working directory
- Ends with the exact command to invoke the next phase (including `clear` suggestion)
- Is tailored to the project's tech stack, conventions, and available agents

---

## Complexity-Based Phase Skipping

Not every task needs the full 4-phase pipeline. At the end of Research, the agent assesses task complexity and recommends the next step:

| Complexity | Research recommends | Rationale |
|------------|-------------------|-----------|
| **Large / complex** | `/rdpi-design ./rdpi/{folder}` | Cross-module, new subsystem, unfamiliar area — needs full design |
| **Medium** | `/rdpi-plan ./rdpi/{folder}` — skip Design | Well-understood area, clear requirements — design is overkill |
| **Small / simple** | `/rdpi-implement ./rdpi/{folder}` — skip Design + Plan | Bug fix, small change — just implement from the Spec |

The agent presents all applicable options with annotations:

```
Next steps:
→ /rdpi-design ./rdpi/{folder}  (recommended for this task — cross-module changes)
→ /rdpi-plan ./rdpi/{folder}    (fast — skip design, go straight to planning)
→ /rdpi-implement ./rdpi/{folder} (fastest — skip design and planning, implement directly)
```

The user always makes the final decision.

---

## RDPI Phases (Generated Skills)

### Phase 1: Research (`/rdpi-research`)

**Goal:** Understand the task, explore the system without bias, and produce a Spec that locks down requirements.

**Input:** User description of the task (feature or bug) + task ID (Jira/Rally/GitHub issue).

**Internal steps (QRSPI: Q + R + Spec):**

#### Step 1: Questions (with user)

Interview the user using `AskUserQuestion` — go back and forth to ensure mutual understanding:
- What is the task? What problem does it solve?
- If bug: reproduction steps, expected vs actual behavior, affected areas
- If feature: user stories, acceptance criteria, constraints, edge cases
- Clarify ambiguities, challenge assumptions, confirm priorities
- Extract everything needed to formulate research questions

The Questions step produces a list of **codebase research questions** — specific things that need to be understood about the system to implement the task.

#### Step 2: Blind Research (sub-agent)

Spawn a sub-agent that receives ONLY the research questions — NOT the task description. The sub-agent:
- Explores the codebase to answer each question
- Documents: file paths, patterns, conventions, dependencies, data flow
- Returns facts only — no opinions, no recommendations, no "we should refactor this"
- Also reports unexpected findings beyond the questions asked
- Operates in its own clean context

#### Step 3: Spec Writing

With the user's requirements (from Questions) and the codebase facts (from Research), write a structured Spec artifact. The Spec is the contract for all subsequent phases.

#### Step 4: Complexity Assessment & Next Step

Assess the task complexity based on the Spec and Research findings. Present next step options with complexity-based recommendations (see "Complexity-Based Phase Skipping" above).

**Output artifacts:**
```
./rdpi/{folder}/01-research/
├── questions.md                   — Research questions formulated from user interview
├── research.md                    — Blind research results (facts only)
├── spec.md                        — Requirements spec
└── review-{agent}.md              — Reviews by discovered agents
```

**User review point:** The Spec — optional but recommended. The user can review it to confirm mutual understanding of requirements, or trust the interview process and move on.

**Ends with:** Complexity-based next step recommendation (see above).

---

### Phase 2: Design (`/rdpi-design`)

**Goal:** Define what to build and how, with architecture decisions locked down before any code is written.

**Input:** Spec + Research artifacts from Phase 1 (read from disk in fresh context).

**Internal steps:**

#### Step 1: Design Document

The agent reads the Spec and Research results and produces a detailed Design document (~200 lines):
- Current state of the system (from Research)
- Desired state (from Spec)
- Architectural approach and key decisions (ADR)
- No code — only architecture, contracts, data flow

**Adaptive depth** (determined automatically from Spec complexity):

| Depth | When | Includes |
|-------|------|----------|
| **Light** | Small fixes, simple bugs | ADR + brief change description |
| **Standard** | Typical features | C4 component diagram + API contracts + test strategy + ADR |
| **Full** | Cross-module changes, new subsystems | All of Standard + sequence diagrams + data flow + security review |

**Design + Critic review:**

The agent produces a design, then a Critic sub-agent reviews it — challenges assumptions, finds weaknesses, identifies risks. The agent incorporates valid critique into the final design. For Full depth, the Critic review is more thorough.

#### Step 2: Spec Consistency Check (sub-agent)

A sub-agent verifies that the Design is consistent with the original Spec:
- All requirements from the Spec are addressed
- No requirements were silently dropped or altered
- Any deviations are explicitly documented with rationale

#### Step 3: Structure/Outline (sub-agent)

A sub-agent generates a Structure/Outline from the final Design:
- High-level breakdown into vertical slices (end-to-end increments)
- Each slice is demonstrable to the user
- Order of delivery: start with what the user can see and give feedback on
- No code, no file paths — just the logical decomposition
- Diagrams: Mermaid or PlantUML (per project convention, default Mermaid)

**Iteration loop:** User reviews Structure/Outline. If changes needed → agent revises Design → sub-agent regenerates Structure/Outline.

**Vertical slicing principle:** Instead of horizontal layers (all backend, then all frontend), each slice delivers complete functionality. Examples adapted to the project:
- Frontend-first: UI with mocks → feedback → backend + integration
- API-first: Contract + stub server → consumers → real implementation
- Data-first: Schema + seed data → processing → presentation

**Multi-phase workflow:** By default, single-phase delivery for most tasks. Multi-phase (multiple delivery increments) is used ONLY for very large tasks. The user decides during bootstrap setup whether to default to multi-phase.

**Output artifacts:**
```
./rdpi/{folder}/02-design/
├── design.md                      — Final design with critic feedback incorporated
├── structure-outline.md           — Vertical slices breakdown (USER REVIEWS THIS)
├── c4-diagrams.md                 — Architecture diagrams (part of Structure review)
├── critic-review.md               — Critic agent's review of the design
├── spec-consistency.md            — Spec consistency check results
└── review-{agent}.md              — Reviews by discovered agents
```

**User review point:** Structure/Outline + C4 diagrams — mandatory. This is THE synchronization point between human and agent before code. The user does NOT need to read the full Design document — the Structure is the user-facing summary.

**Ends with:** `Next: clear context, then run /rdpi-plan ./rdpi/{folder}`

---

### Phase 3: Plan (`/rdpi-plan`)

**Goal:** Convert Design + Structure into concrete, actionable execution plans.

**Input:** Design + Structure/Outline artifacts from Phase 2 (read from disk in fresh context).

**Internal steps:**

#### Step 1: Generate Plans (parallel sub-agents)

Multiple plans are generated in parallel, one per role/concern:
- **Per vertical slice** — each delivery increment gets its own plan
- **Per role** — frontend, backend, database, tests, documentation, verification
- Plans are generated by parallel sub-agents, each reading the Design + source code

Each plan contains:
- Exact file paths (create/modify)
- Verification steps (build commands, test commands)
- Commit messages
- Dependencies on other plans

#### Step 2: Global Plan

One synthesized global plan with:
- Execution order across slices and roles
- Parallelization opportunities (e.g., frontend + backend after contracts locked)
- Links to individual role/phase plans
- Checkpoints where user review is needed

#### Step 3: Plan Review (internal feedback loop)

Discovered review agents check plans for feasibility and completeness. A Spec Consistency sub-agent verifies that plans cover all Spec requirements.

Review results are used as an internal feedback loop — if issues are found, the agent fixes them before presenting to the user. The user does NOT read review details, only the summary.

#### Step 4: Brief Summary for User

The user does NOT read the full plan. Instead, the agent presents:
- A brief summary of what will be done (1-2 paragraphs)
- Number of phases and estimated scope
- Which parts can run in parallel
- Any risks or issues surfaced by the review (high-level only)

**Output artifacts:**
```
./rdpi/{folder}/03-plan/
├── plan-global.md                 — Master plan with links to sub-plans
├── plan-{slice}-{role}.md         — Individual plans per slice/role
├── summary.md                     — Brief summary for user (NOT the full plan)
├── spec-consistency.md            — Plan vs Spec consistency check
└── review-{agent}.md              — Internal review results (not for user)
```

**Ends with:** Presents the brief summary, then: `Next: clear context, then run /rdpi-implement ./rdpi/{folder}`

---

### Phase 4: Implement (`/rdpi-implement`)

**Goal:** Execute the plan phase by phase with an orchestrator agent managing specialized sub-agents.

**Input:** Plan artifacts from Phase 3 + Design for reference (read from disk in fresh context).

**Internal steps:**

#### Step 1: Worktree Setup

- Create a git branch for the task
- Optionally use git worktree for isolation (if the environment supports it)

#### Step 2: Phase-by-Phase Execution

The orchestrator agent:
1. Reads the global plan and identifies the current phase
2. Spawns specialized sub-agents per role (frontend, backend, tests, etc.) according to the plan
3. Sub-agents can run in parallel when plans allow it
4. After each phase/slice completion:
   - Writes updates to `implementation-log.md`
   - Prompts the user to review the result
   - Waits for user confirmation before proceeding

#### Step 3: Draft Pull Request

After user confirms the implementation:
- Create a **Draft PR** (sub-agent) — allows the user to review code in the PR interface
- Include: summary from Design, changes made, test results
- The user reviews code in the Draft PR (this is the code review step)
- User promotes Draft to Ready when satisfied

#### Step 4: Deploy & Verify (if available)

If deployment capability was discovered during bootstrap:
- Propose deploying for verification (sub-agent)
- After deploy: spawn a monitoring sub-agent to check logs
- If browser testing is available: spawn a testing sub-agent for manual verification

Whether the agent can deploy autonomously is determined during rdpi-bootstrap setup.

**Output artifacts:**
```
./rdpi/{folder}/04-implement/
├── implementation-log.md          — Progress log with updates per phase
├── review-{agent}.md              — Code reviews by agents
└── pr-summary.md                  — PR description and link
```

**User review points:**
- Result after each delivery increment (test it)
- Code in Draft PR (code review)
- PR before merge

---

## Artifact Directory Structure

```
./rdpi/{YYYY-MM-DD}-{task-id}-{short-name}/
├── 01-research/
│   ├── questions.md                   — Research questions from user interview
│   ├── research.md                    — Blind research results
│   ├── spec.md                        ← USER REVIEW (optional)
│   └── review-{agent}.md
├── 02-design/
│   ├── design.md                      — Full design (internal)
│   ├── structure-outline.md           ← USER REVIEW (mandatory)
│   ├── c4-diagrams.md                ← USER REVIEW (mandatory, part of Structure)
│   ├── critic-review.md              — Critic's review
│   ├── spec-consistency.md           — Spec consistency check
│   └── review-{agent}.md
├── 03-plan/
│   ├── plan-global.md                — Master plan (internal)
│   ├── plan-{slice}-{role}.md        — Sub-plans (internal)
│   ├── summary.md                    — Brief summary shown to user
│   ├── spec-consistency.md           — Plan vs Spec check
│   └── review-{agent}.md            — Internal reviews
├── 04-implement/
│   ├── implementation-log.md
│   ├── review-{agent}.md
│   └── pr-summary.md
└── MANIFEST.md                       — Navigation index with status of each phase
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
   - Can the agent deploy autonomously? What's the deploy process?
   - Is browser-based testing available? (for post-deploy verification)
   - Default to multi-phase workflow for all tasks, or only large ones?

5. **Save spec:** Write answers to `./rdpi/RPI_SKILLS_SPEC.md`

6. **Propose agent assignments:** Based on discovered agents, propose which agents handle which RDPI roles:
   - Research: blind research sub-agent, review agents
   - Design: design agent, critic agent, spec-consistency checker, review agents
   - Plan: parallel plan generators per role, spec-consistency checker
   - Implement: orchestrator, specialized coders, code review agents, PR agent, deploy agent, monitoring agent
   - Present the mapping to the user for confirmation

7. **Generate phase skills:** Create the 4 RDPI skills + README in `./rdpi/skills/`
   - Each skill reads its phase template from the meta-skill's bundled references
   - Each skill is customized with project-specific: paths, build commands, conventions, assigned agents

8. **Output:** Display what was created and how to start using it. Include realistic expectations: this workflow is designed for medium-to-large tasks where upfront investment pays off. For small fixes, use direct prompts. Expect 2-3x productivity improvement on suitable tasks.

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
- Feature implementation workflow (RDPI with full QRSPI internally)
- Bug fixing workflow (adapted RDPI — lighter Design, focused Research)
- Generating project-specific RDPI skills
- Runtime discovery and assignment of available agents
- Design + Critic review (single design with critic, not multi-perspective)
- Spec consistency checking across phases
- Vertical slice delivery planning
- Complexity-based phase skipping
- Draft PR creation and code review workflow
- Deployment and post-deploy monitoring (where available)

### Out of Scope (for now)
- Refactoring-only workflows
- Migration workflows
- Incident investigation
- Running the RDPI phases itself (that's what the generated skills do)

---

## Dependencies

### Required
- `AskUserQuestion` tool — for interview flow
- `Agent` tool — for spawning sub-agents (blind research, critic, parallel plans, implementation)
- File system tools (Read, Write, Glob, Grep) — for codebase analysis and skill generation

### Optional (discovered at runtime)
- `implement-spec` or similar — can be delegated to during Implement phase
- `interview` skill — can be used/adapted for Research phase interview
- Any review agents (devil's advocate, architecture, clean-code, regression, linebyline, security)
- Any deployment/CI tools
- Browser testing capabilities (for post-deploy verification)

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
