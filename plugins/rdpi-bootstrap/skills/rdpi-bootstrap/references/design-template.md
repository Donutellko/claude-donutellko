---
name: rdpi-design
description: >
  Run the Design phase of RDPI workflow: create an interactive design document collaboratively with the user,
  then generate a Structure/Outline with vertical slices. Use this after Research phase completes,
  when the user says "/rdpi-design", or wants to design the architecture before implementing.
---

# RDPI Design

You are running the Design phase — defining *what* to build and *how*, with architecture decisions locked down before any code is written. This is the most important phase: a good design prevents wasted implementation effort.

**Tech stack:** {{TECH_STACK}}
**Diagram format:** {{DIAGRAM_FORMAT}}

## Resolve Artifact Folder

The argument passed to this skill is a Task ID or short name (e.g., `US1234` or `US1234-fix-login`). Resolve it to the artifact folder before reading any files:

1. Glob `./rdpi/*{arg}*/` to find matching folders
2. **One match** → use it
3. **Multiple matches** → list them and ask the user to pick one
4. **No match** → error: "No artifact folder found for '{arg}'. Run /rdpi-research first."

## Inputs

Read the artifacts from the previous phase (using the resolved folder path):
- `{folder}/01-research/spec.md` — requirements and constraints
- `{folder}/01-research/research.md` — codebase facts
- `{folder}/01-research/questions.md` — original research questions (for context)

If any of these are missing, tell the user and suggest running `/rdpi-research` first.

Create the artifact directory: `{folder}/02-design/`

## Step 1: Interactive Design Document

Brain-dump your initial design into `02-design/design.md` and then **go back and forth with the user** to shape it. This is a collaborative process, not a waterfall — think of it as a shared whiteboard where you write first and the user corrects, shapes, and refines.

The Design document covers:
- **Current state** — how the system works now (from Research)
- **Desired end state** — what it should look like after (from Spec)
- **Patterns to follow** — conventions found in the codebase
- **Architectural approach** — the high-level "how"
- **Key decisions** — each with rationale (ADR-style: decision, alternatives considered, why this one)
- **Open questions** — things you need the user to resolve

What the Design does NOT contain: implementation code. Architecture, contracts, data flow — not code.

### Adaptive depth

Choose depth based on Spec complexity — don't over-engineer simple tasks:

| Depth | When | What to include |
|-------|------|-----------------|
| **Light** | Small fixes, simple bugs | Key decision + brief change description |
| **Standard** | Typical features | Component diagram + API contracts + test strategy + key decisions |
| **Full** | Cross-module, new subsystems | All of Standard + sequence diagrams + data flow + security review |

Keep iterating with the user until all open questions are resolved and you both agree on the approach. Update `design.md` as you go — this file is the living record of your conversation.

## Step 2: Structure/Outline — Sub-agent

Once the Design is stable, spawn a sub-agent in a separate context to generate the Structure/Outline. The sub-agent reads Design + Research + Spec and produces a decomposition into **vertical slices** — end-to-end increments, not horizontal layers.

```
Agent prompt:
"You are a software architect creating a Structure/Outline for implementation.

Read these files:
- <path>/02-design/design.md
- <path>/01-research/spec.md
- <path>/01-research/research.md

Produce a Structure/Outline with:
1. Breakdown into vertical slices (each slice is a complete, demonstrable piece of functionality)
2. Delivery order — start with what the user can see and give feedback on
3. For each slice: what changes, what to test, dependencies on other slices
4. Diagrams in {{DIAGRAM_FORMAT}} format where helpful
5. If the user or design mentions specific types/signatures, include them (like C header files — signatures, not implementation)

Write to: <path>/02-design/structure-outline.md
If there are architecture diagrams, also write: <path>/02-design/c4-diagrams.md"
```

Use model: {{MODEL_DESIGN_STRUCTURE}} with extended thinking.

## User Review — Mandatory

The Structure/Outline is THE synchronization point before code. Present it to the user:

"Here's the Structure/Outline — the breakdown of what we'll build and in what order. Please review `02-design/structure-outline.md`. This is the most important review point in the workflow."

If the user wants changes:
1. Discuss and update the Design if the change is architectural
2. Re-run the Structure/Outline sub-agent with the updated Design
3. Present the revised Structure for review

Repeat until the user approves.

## Next Step

Once the user approves the Structure:

```
Next step (clear your context before running):
→ /rdpi-plan {{TASK_ID}}
```

Print this command as plain text. Do NOT use the Skill tool to invoke the next phase. The user runs it in a fresh session.

## Recovery

If the conversation has gone sideways (user corrected you 2-3+ times in a row), suggest restarting this phase. The Design document on disk preserves your progress — a fresh context will read it and continue from there.
