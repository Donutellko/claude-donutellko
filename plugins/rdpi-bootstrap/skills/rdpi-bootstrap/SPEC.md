# RDPI Bootstrap — Specification

**Date**: 2026-03-30
**Status**: Draft v4
**Authors**: Donat + Claude
**Based on**: Dex Horthy "Context Engineering" (QRSPI), Dmitry Bereznitsky "Process over Prompts"

---

## Terminology

| Term | Meaning |
|------|---------|
| **QRSPI** | The full methodology from Dex Horthy: Questions → Research → Design → Structure/Outline → Plan → Worktree → Implement → PR. |
| **RDPI** | Our 4 top-level skills: Research, Design, Plan, Implement. Each skill internally implements multiple QRSPI stages. |
| **Phase** | One of the 4 RDPI skills. Each phase runs in a fresh Claude session. |
| **Artifact** | A structured markdown file produced by a phase, serving as the contract for the next phase. Artifacts on disk replace context compaction — the context window is ephemeral, the artifact files are durable state. |
| **Vertical slice** | An end-to-end piece of functionality (not a horizontal layer like "all backend"). |

### Why 4 Skills, Not 8 Stages?

QRSPI defines 8 logical stages. We collapse them into 4 skills because:
1. **Context isolation where it matters most.** The biggest context-pollution risk is between Research→Design→Plan→Implement. Within a phase, sub-agents provide additional isolation (e.g., blind Research sub-agent runs in its own context).
2. **UX simplicity.** 4 commands to remember, not 8. Each command does a coherent chunk of work.
3. **The orchestrator overhead is manageable.** Within Research, the main agent holds interview context + spawns a blind sub-agent. The interview is user input (high-signal), not accumulated LLM output (low-signal noise).

### Our Extensions Beyond QRSPI

The following are our additions, not from Dex's talks:
- **Spec artifact** — output of Research phase, locks down requirements before Design
- **Complexity-based phase skipping** — Research recommends skipping phases for simpler tasks
- **Spec consistency check** — sub-agent in Plan phase verifies plan covers all Spec requirements
- **Draft PR workflow** — code review happens in a Draft PR
- **Deploy & Verify** — speculative, Dex did not cover the implement side in detail

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

### 2. Artifacts Replace Compaction

Every phase produces structured markdown artifacts on disk. These artifacts are the durable state — the context window is ephemeral and disposable. Auto-compaction is unnecessary because everything that matters is in static files. You can always resume from where you left off by reading the artifacts.

### 3. Runtime Agent Discovery

The meta-skill does NOT hardcode specific agents or skills. At execution time, it discovers what agents and skills are available in the current environment and assigns them to roles. This makes it portable across different plugin setups.

### 4. "Go Back and Forth With Me"

Both the meta-skill interview and the generated Research/Design phase skills must ensure mutual understanding through iterative dialogue. Claude should not assume — it should confirm.

### 5. Recommended Options with Annotations

When using `AskUserQuestion`, always mark the recommended option and annotate choices with tags like `(recommended)`, `(fast)`, `(risky)`, `(safe)`, `(compromise)`.

### 6. Instruction Budget

The total instruction-following budget across all sources (system prompt, tools, MCP, CLAUDE.md, skill) is ~150-200 instructions. Each individual phase SKILL.md should aim for **under 40 instructions** — Dex's updated target. If a skill grows beyond that, it should delegate to sub-agents or extract content into `references/` files read on demand.

### 7. Minimal Tool Surface Per Phase

Each phase should only load the agents, MCPs, and tools it actually needs — not all available ones. Discovered agents/MCPs contribute to the instruction budget. Loading unnecessary tools bloats the context window.

### 8. Non-biased Research

Research sub-agents operate WITHOUT knowledge of the specific task. They receive only questions about the codebase and return facts. This prevents confirmation bias. Note: true blindness is impossible since the questions themselves encode task assumptions (our observation). The goal is to prevent the research agent from selectively finding evidence that supports a pre-determined solution. The research agent should also report unexpected findings beyond the questions asked.

### 9. Vertical Slicing

Implementation is organized as end-to-end vertical slices, not horizontal layers. Instead of "all backend, then all frontend", each slice delivers a complete piece of functionality that can be demonstrated and verified.

### 10. No Magic Prompts — Pipeline as Control

Skills should work through pipeline structure and context quality, not through prompt engineering tricks. If a skill only works with specific phrasing, the design is wrong. The sequence of phases, context boundaries, and artifact contracts are the control mechanism. Use code for control flow, not prompts.

### 11. Bad Trajectory Recovery

If the conversation within a phase goes sideways (user has corrected the agent 2-3+ times in a row), the skill should suggest restarting the phase with a fresh context rather than continuing to accumulate a bad trajectory.

### 12. Human Reads Code

The user is expected to read and verify code — not just trust artifacts. Code review happens in the Draft PR. During Research, the user should spot-check key code paths identified by the research agent.

### 13. Right Model for the Right Job

Each agent role should use the appropriate model and thinking level. More capable models cost more and are slower — use them only where the task demands it.

**Default model assignments:**

| Role | Model | Thinking | Rationale |
|------|-------|----------|-----------|
| Research: Questions (main agent) | Opus | standard | Interview needs depth of understanding |
| Research: Blind sub-agent | Sonnet | standard | Factual code exploration |
| Research: Spec writing | Opus | extended | Synthesis of interview + research, key artifact |
| Design: Interactive document | Opus | extended | Architecture decisions — the most important phase |
| Design: Structure/Outline sub-agent | Sonnet | extended | Decomposition requires reasoning about slices |
| Plan: Generation | Sonnet | extended | File paths, ordering, dependencies |
| Plan: Spec consistency checker | Sonnet | standard | Document comparison |
| Implement: Orchestrator | Opus | standard | Coordination, not creative work |
| Implement: Coders | Sonnet | standard | Code from plan — mechanical |
| Implement: PR creation | Sonnet | standard | Straightforward |
| Deploy/Monitor: Log reader | Haiku | standard | Reading logs, simple checks |
| Deploy/Monitor: Browser testing | Sonnet | standard | Navigation logic needed |

**Thinking levels:**
- **standard** — default for most tasks
- **extended** — Design, Spec writing, Plan generation (reasoning required)
- **mega / ultra** — not default; opt-in for Full-depth Design on exceptionally complex tasks

**Bootstrap questions about models:**
- What is the most powerful model allowed? (Opus / Sonnet / Haiku)
- Is extended thinking allowed? Mega? Ultra?
- Are there budget constraints? (if yes — downgrade Opus → Sonnet where possible)

If the user limits to Sonnet: Opus roles switch to Sonnet with extended thinking. If limited to Haiku: everything runs on Haiku (with a warning about quality degradation).

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
├── rdpi-design/SKILL.md           — Design phase skill (interactive Design + Structure/Outline)
├── rdpi-plan/SKILL.md             — Plan phase skill (plan + spec consistency check)
├── rdpi-implement/SKILL.md        — Implement phase skill (orchestrator + sub-agents + PR)
└── README.md                      — Usage guide with phase order and commands
```

Each generated skill:
- Knows how to execute its phase
- Writes its artifact to the RDPI working directory
- Ends with the exact command to invoke the next phase (including `clear` suggestion)
- Is tailored to the project's tech stack, conventions, and available agents

---

## Complexity-Based Phase Skipping

*Our extension — not from QRSPI.*

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

**Internal steps:**

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

*Our extension — QRSPI does not have a separate Spec stage.*

With the user's requirements (from Questions) and the codebase facts (from Research), write a structured Spec artifact. The Spec is the contract for all subsequent phases.

#### Step 4: Complexity Assessment & Next Step

Assess the task complexity based on the Spec and Research findings. Present next step options with complexity-based recommendations (see "Complexity-Based Phase Skipping" above).

**Output artifacts:**
```
./rdpi/{folder}/01-research/
├── questions.md                   — Research questions formulated from user interview
├── research.md                    — Blind research results (facts only)
└── spec.md                        — Requirements spec
```

**User review point:** The Spec — optional. The user can review it to confirm mutual understanding, or trust the interview process and move on.

**Ends with:** Complexity-based next step recommendation.

---

### Phase 2: Design (`/rdpi-design`)

**Goal:** Define what to build and how, with architecture decisions locked down before any code is written.

**Input:** Spec + Research artifacts from Phase 1 (read from disk in fresh context).

**Internal steps:**

#### Step 1: Interactive Design Document

The agent reads the Spec and Research results and produces a Design document (~200 lines). This is an **interactive process** — the agent brain-dumps the design and the human corrects, shapes, and refines it through dialogue. Not a waterfall, but a collaborative back-and-forth.

The Design document contains:
- Current state of the system (from Research)
- Desired end state (from Spec)
- Patterns found in the codebase to follow
- Architectural approach and key decisions (ADR)
- Resolved design decisions (with rationale)
- Open questions (for the human to answer)
- No code — only architecture, contracts, data flow

**Adaptive depth** (determined automatically from Spec complexity):

| Depth | When | Includes |
|-------|------|----------|
| **Light** | Small fixes, simple bugs | ADR + brief change description |
| **Standard** | Typical features | C4 component diagram + API contracts + test strategy + ADR |
| **Full** | Cross-module changes, new subsystems | All of Standard + sequence diagrams + data flow + security review |

The human interacts with the Design until all open questions are resolved and both sides agree on the approach.

#### Step 2: Structure/Outline (sub-agent, separate context)

A sub-agent generates a Structure/Outline from the final Design (in a fresh context, as per Dex's recommendation — Design and Structure in different context windows). The sub-agent reads the Design + Research + Spec.

The Structure/Outline contains:
- High-level breakdown into vertical slices (end-to-end increments)
- Each slice is demonstrable to the user
- Order of delivery: start with what the user can see and give feedback on
- How to test along the way
- Diagrams: Mermaid or PlantUML (per project convention, default Mermaid)

The Structure starts high-level but can include **types, function signatures, and file paths** when the user asks for more detail (like C header files — the signatures and types, not the implementation).

**Iteration loop:** User reviews Structure/Outline. If changes needed → agent revises Design → sub-agent regenerates Structure/Outline.

**Vertical slicing principle:** Instead of horizontal layers (all backend, then all frontend), each slice delivers complete functionality. Examples adapted to the project:
- Frontend-first: UI with mocks → feedback → backend + integration
- API-first: Contract + stub server → consumers → real implementation
- Data-first: Schema + seed data → processing → presentation

**Multi-phase workflow:** By default, single-phase delivery for most tasks. Multi-phase (multiple delivery increments) is used ONLY for very large tasks. The user decides during bootstrap setup whether to default to multi-phase.

**Output artifacts:**
```
./rdpi/{folder}/02-design/
├── design.md                      — Final design (interactive, shaped by human)
├── structure-outline.md           — Vertical slices breakdown (USER REVIEWS THIS)
└── c4-diagrams.md                 — Architecture diagrams (part of Structure review)
```

**User review point:** Structure/Outline + C4 diagrams — **mandatory**. This is THE synchronization point between human and agent before code. The user does NOT need to read the full Design document — the Structure is the user-facing summary. The Design was already shaped by the human during interactive creation.

**Ends with:** `Next: clear context, then run /rdpi-plan ./rdpi/{folder}`

---

### Phase 3: Plan (`/rdpi-plan`)

**Goal:** Convert Design + Structure into a concrete, actionable execution plan.

**Input:** Design + Structure/Outline + Research artifacts (read from disk in fresh context).

**Internal steps:**

#### Step 1: Generate Plan

One agent generates the plan from Design + Structure + Research + source code. The plan contains:
- Exact file paths (create/modify)
- Verification steps (build commands, test commands)
- Commit messages
- Execution order with parallelization opportunities
- Checkpoints where user review is needed

For multi-phase tasks: the plan covers all vertical slices, with per-slice breakdowns.

#### Step 2: Spec Consistency Check (sub-agent)

*Our extension — not from QRSPI.*

A sub-agent verifies that the Plan is consistent with the original Spec:
- All requirements from the Spec are addressed by the plan
- No requirements were silently dropped
- Any deviations are flagged

If issues found, the agent fixes them before presenting to the user.

#### Step 3: Brief Summary for User

The user does NOT read the full plan. The agent presents:
- A brief summary of what will be done (1-2 paragraphs)
- Number of phases and estimated scope
- Which parts can run in parallel
- Any issues surfaced by the spec consistency check (high-level only)

**Output artifacts:**
```
./rdpi/{folder}/03-plan/
├── plan.md                        — Full plan (for the agent, not for human)
├── spec-consistency.md            — Plan vs Spec consistency check
└── summary.md                     — Brief summary shown to user
```

**Ends with:** Presents the brief summary, then: `Next: clear context, then run /rdpi-implement ./rdpi/{folder}`

---

### Phase 4: Implement (`/rdpi-implement`)

**Goal:** Execute the plan with an orchestrator agent managing specialized sub-agents.

**Input:** Plan + Design artifacts (read from disk in fresh context).

**Internal steps:**

#### Step 1: Branch & Worktree Setup

*QRSPI has Worktree as a separate stage. We fold it into Implement for UX simplicity — acknowledged deviation.*

Before writing any code, the orchestrator (or each implementation sub-agent) must ensure a clean branch from up-to-date main:
1. `git checkout main && git pull`
2. Create a new branch: `{task-id}/{short-description}` (e.g., `PROJ-123/user-auth-flow`)
3. Optionally use git worktree for isolation (if the environment supports it)

This is non-negotiable — implementation never starts on a stale branch or directly on main.

#### Step 2: Execution

The orchestrator agent:
1. Reads the plan and identifies phases/slices
2. Spawns specialized sub-agents per role (frontend, backend, tests, etc.) as needed
3. Sub-agents can run in parallel when the plan allows it
4. After each slice completion:
   - Writes updates to `implementation-log.md`
   - **First slice:** mandatory user review (test the result, confirm direction)
   - **Subsequent slices:** configurable — user can opt for review or auto-proceed

#### Step 3: Draft Pull Request

*QRSPI has PR as a separate stage. We fold it into Implement — acknowledged deviation. Our addition: Draft PR for code review.*

After user confirms the implementation:
- Create a **Draft PR** (sub-agent) — the user reviews code in the PR interface
- Include: summary from Design, changes made, test results
- User promotes Draft to Ready when satisfied

#### Step 4: Deploy & Verify (opt-in)

*Speculative — Dex did not cover the implement side in detail.*

If deployment capability was discovered during bootstrap:
- Propose deploying for verification (sub-agent)
- After deploy: spawn a monitoring sub-agent to check logs
- If browser testing is available: spawn a testing sub-agent for manual verification

Whether the agent can deploy autonomously is determined during rdpi-bootstrap setup.

**Output artifacts:**
```
./rdpi/{folder}/04-implement/
├── implementation-log.md          — Progress log with updates per phase
└── pr-summary.md                  — PR description and link
```

**User review points:**
- First slice result (mandatory — test it)
- Subsequent slices (configurable)
- Code in Draft PR (mandatory — code review)
- PR before merge

---

## Artifact Directory Structure

```
./rdpi/{YYYY-MM-DD}-{task-id}-{short-name}/
├── 01-research/
│   ├── questions.md                   — Research questions from user interview
│   ├── research.md                    — Blind research results
│   └── spec.md                        ← USER REVIEW (optional)
├── 02-design/
│   ├── design.md                      — Full design (internal, shaped by human)
│   ├── structure-outline.md           ← USER REVIEW (mandatory)
│   └── c4-diagrams.md                ← USER REVIEW (mandatory, part of Structure)
├── 03-plan/
│   ├── plan.md                        — Full plan (internal, for the agent)
│   ├── spec-consistency.md            — Plan vs Spec check (internal)
│   └── summary.md                     — Brief summary shown to user
├── 04-implement/
│   ├── implementation-log.md
│   └── pr-summary.md
└── MANIFEST.md                        — Navigation index with status of each phase
```

The `{folder}` format defaults to `{YYYY-MM-DD}-{task-id}-{short-name}`. The meta-skill asks the user for their preferred format during bootstrap interview.

---

## Meta-Skill Workflow (rdpi-bootstrap itself)

### First Run

1. **Check for existing spec:** Look for `./rdpi/RPI_SKILLS_SPEC.md`
   - If found → go to Re-run flow
   - If not found → continue with First Run

2. **Check for existing project-specific skills:** Look for skills like `implement-feature`, `implement-story`, or any skill that reflects project specifics. Extract useful context from them.

3. **Discover available agents and skills:** Enumerate all agents and skills available in the current environment. Catalog their capabilities for assignment to RDPI phases. Only assign agents that are actually needed per phase (Principle 7: Minimal Tool Surface).

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
   - What is the most powerful model allowed? (Opus / Sonnet / Haiku)
   - Is extended thinking allowed? Mega? Ultra?
   - Any budget constraints on model usage?

5. **Save spec:** Write answers to `./rdpi/RPI_SKILLS_SPEC.md`

6. **Propose agent assignments:** Based on discovered agents, propose which agents handle which RDPI roles:
   - Research: blind research sub-agent
   - Design: structure/outline sub-agent
   - Plan: spec-consistency checker sub-agent
   - Implement: orchestrator, specialized coders (per role), PR sub-agent, optionally deploy/monitor agents
   - Present the mapping to the user for confirmation

7. **Generate phase skills:** Create the 4 RDPI skills + README in `./rdpi/skills/`
   - Each skill reads its phase template from the meta-skill's bundled references
   - Each skill is customized with project-specific: paths, build commands, conventions, assigned agents
   - Each skill must stay under ~40 instructions

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
- Spec consistency checking in Plan phase (our extension)
- Vertical slice delivery planning
- Complexity-based phase skipping (our extension)
- Draft PR creation and code review workflow (our extension)
- Deployment and post-deploy monitoring (opt-in, speculative)

### Out of Scope (for now)
- Refactoring-only workflows
- Migration workflows
- Incident investigation
- Running the RDPI phases itself (that's what the generated skills do)

---

## Dependencies

### Required
- `AskUserQuestion` tool — for interview flow and interactive Design
- `Agent` tool — for spawning sub-agents (blind research, structure outline, spec consistency, implementation)
- File system tools (Read, Write, Glob, Grep) — for codebase analysis and skill generation

### Optional (discovered at runtime)
- `implement-spec` or similar — can be delegated to during Implement phase
- `interview` skill — can be used/adapted for Research phase interview
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
