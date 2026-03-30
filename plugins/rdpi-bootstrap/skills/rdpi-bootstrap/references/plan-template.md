---
name: rdpi-plan
description: >
  Run the Plan phase of RDPI workflow: convert the Design and Structure into a concrete execution plan,
  verify plan covers all Spec requirements, and present a brief summary to the user.
  Use this after Design phase completes, when the user says "/rdpi-plan",
  or when skipping Design for medium-complexity tasks (going straight from Research to Plan).
---

# RDPI Plan

You are running the Plan phase — converting architecture decisions into a concrete, actionable execution plan. The plan is for the implementing agent, not for the human — the human sees only a brief summary.

**Tech stack:** {{TECH_STACK}}
**Build command:** {{BUILD_CMD}}
**Test command:** {{TEST_CMD}}

## Inputs

Read the artifacts from previous phases:
- `01-research/spec.md` — requirements (always present)
- `01-research/research.md` — codebase facts (always present)
- `02-design/design.md` — architecture decisions (may be absent if Design was skipped)
- `02-design/structure-outline.md` — vertical slices (may be absent if Design was skipped)

If Design was skipped (no `02-design/`), that's fine — generate the plan directly from Spec + Research. The plan will be simpler but still structured.

Create the artifact directory: `{{ARTIFACT_FOLDER}}/03-plan/`

## Step 1: Generate the Plan

Read the available artifacts and the relevant source code. Produce a detailed plan in `03-plan/plan.md`:

```markdown
# Implementation Plan: [Task Title]

## Overview
[1-2 sentences: what this plan delivers]

## Vertical Slices

### Slice 1: [Name]
**Files to create/modify:**
- `path/to/file.ext` — [what changes]

**Steps:**
1. [Specific action]
2. [Specific action]

**Verification:**
- Run: `{{TEST_CMD}}`
- Check: [what to verify]

**Commit message:** `feat: [description]`

### Slice 2: [Name]
...

## Execution Notes
- Slices that can run in parallel: [list]
- User review checkpoints: [after slice 1 (mandatory), others as needed]
- Known risks: [things that might go wrong]
```

Be specific: exact file paths, exact commands, exact commit messages. The implementing agent should be able to follow this plan mechanically.

## Step 2: Spec Consistency Check — Sub-agent

Spawn a sub-agent to verify the plan covers all Spec requirements:

```
Agent prompt:
"Compare these two documents and check for consistency:

SPEC (requirements): <read 01-research/spec.md>
PLAN (implementation): <read 03-plan/plan.md>

For each requirement and acceptance criterion in the Spec:
1. Is it addressed by the plan? Which slice?
2. If not addressed — flag it.

For each plan item:
1. Does it trace back to a Spec requirement?
2. If not — is it necessary infrastructure, or scope creep?

Write your findings to: <path>/03-plan/spec-consistency.md"
```

Use model: {{MODEL_PLAN_CONSISTENCY}}

If the consistency check finds gaps, fix the plan before presenting to the user. If there are genuine scope questions (Spec is ambiguous), note them in the summary.

## Step 3: Brief Summary for User

The user does NOT read the full plan. Write a brief summary to `03-plan/summary.md` and present it:

```markdown
# Plan Summary: [Task Title]

## What will be done
[1-2 paragraphs, plain language]

## Scope
- [N] vertical slices
- Estimated files: [N] new, [N] modified
- Key slices: [list the most important ones]

## Parallel opportunities
[Which slices can run simultaneously]

## Spec coverage
[All requirements covered / N issues found — with brief description]

## Risks
[Top 1-2 risks, if any]
```

Tell the user: "Here's a summary of the plan. The full plan is at `03-plan/plan.md` if you want to review the details. Ready to implement?"

## Next Step

```
Next step (clear your context before running):
→ /rdpi-implement {{ARTIFACT_FOLDER}}
```

## Recovery

If the plan seems wrong or the user disagrees with the approach, the issue is likely in the Design, not the Plan. Suggest going back to `/rdpi-design` rather than trying to fix fundamental architecture in the Plan phase.
