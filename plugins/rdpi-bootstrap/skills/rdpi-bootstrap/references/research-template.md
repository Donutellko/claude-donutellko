---
name: rdpi-research
description: >
  Start your task implementation. Paste the ticket ID or description and answer questions. Produces: spec.md.
  Trigger: user pastes a ticket/issue ID, describes a feature to build, or reports a bug to fix.
---

# RDPI Research

You are running the Research phase — the first step in the RDPI development workflow. Your goal is to deeply understand the task, explore the codebase without bias, and produce a Spec that anchors all subsequent work.

**Tech stack:** {{TECH_STACK}}
**Task tracker:** {{TASK_TRACKER}}

## Inputs

The user provides a task description (feature or bug) and optionally a task ID from {{TASK_TRACKER}}.

Create the artifact directory: `{{ARTIFACT_FOLDER_FORMAT}}/01-research/`

## Step 1: Questions — Interview the User

Go back and forth with the user to build a complete understanding of the task. Use `AskUserQuestion` and confirm your understanding before moving on.

For **features**, clarify:
- What problem does this solve? Who benefits?
- User stories and acceptance criteria
- Constraints, edge cases, and out-of-scope items
- Integration points with existing system

For **bugs**, clarify:
- Reproduction steps (exact sequence)
- Expected vs actual behavior
- When it started (if known), affected areas
- Severity and impact

Challenge assumptions. If something seems underspecified, ask. The goal is to formulate precise **codebase research questions** — specific things that need to be understood about the system to do this task well.

Write the questions to `01-research/questions.md` before proceeding.

## Step 2: Blind Research — Sub-agent

Spawn a sub-agent that receives ONLY the research questions — not the task description, not the user stories, not your understanding of the goal. This prevents confirmation bias: the researcher reports facts about the codebase, not evidence for a predetermined solution.

```
Agent prompt:
"You are a codebase researcher. Answer each question below by exploring the code.
Report facts only — file paths, patterns, data flow, conventions, dependencies.
No opinions, no recommendations, no 'we should...' statements.
Also report any unexpected findings relevant to the questions.

Questions:
<paste questions.md content>

Save your findings to: <path>/01-research/research.md"
```

Use model: {{MODEL_RESEARCH_BLIND}}

When the sub-agent finishes, read `research.md` to verify it contains useful findings. If it's thin or missed key areas, you can ask targeted follow-up questions (but keep them factual, not leading).

## Step 3: Write the Spec

With the user's requirements (from Step 1) and the codebase facts (from Step 2), write a structured Spec. This is the contract for all subsequent phases — if it's wrong here, everything downstream is wrong.

Write to `01-research/spec.md` using this structure:

```markdown
# Spec: [Task Title]

## Task
[One-paragraph summary]

## Type
Feature / Bug fix

## Requirements
### Functional
- [Requirement 1]
- [Requirement 2]

### Non-functional
- [Performance, security, accessibility constraints]

## Acceptance Criteria
- [ ] [Criterion 1 — testable and specific]
- [ ] [Criterion 2]

## Constraints
- [Technical, timeline, or scope constraints]

## Out of Scope
- [What this task explicitly does NOT include]

## Key Findings from Research
[Summary of the most important codebase facts that shape the approach]

## Complexity Assessment
[Simple / Medium / Complex — with reasoning]
```

Tell the user: "I've written the Spec. You can review it at `01-research/spec.md` if you'd like to verify my understanding, or we can proceed to the next step."

## Step 4: Recommend Next Phase

Based on complexity assessment:

| Complexity | Recommendation |
|------------|---------------|
| **Complex** | `/rdpi-design {{TASK_ID}}` — cross-module, new subsystem, unfamiliar area |
| **Medium** | `/rdpi-plan {{TASK_ID}}` — skip Design, well-understood area |
| **Simple** | `/rdpi-implement {{TASK_ID}}` — skip Design + Plan, just implement |

Present all options with annotations:

```
Next steps (clear your context before running):
→ /rdpi-design {{TASK_ID}}  (recommended — [reason])
→ /rdpi-plan {{TASK_ID}}    (fast — skip design)
→ /rdpi-implement {{TASK_ID}} (fastest — implement directly from spec)
```

The user always decides.

**Important:** Print the next command as plain text only. Do NOT use the Skill tool or any other mechanism to invoke the next phase automatically. The user runs the command in a fresh session.

## Recovery

If the conversation has gone sideways (user corrected you 2-3+ times in a row), suggest: "It seems like we're not aligned. Want to restart this phase with a fresh context? I'll re-read the artifacts we've created so far."
