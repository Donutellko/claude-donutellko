---
name: rdpi-implement
description: >
  Run the Implement phase of RDPI workflow: set up a clean branch, execute the plan with sub-agents,
  and create a Draft PR for code review. Use this after Plan phase completes,
  when the user says "/rdpi-implement", or when skipping Design+Plan for simple tasks
  (going straight from Research to Implement).
---

# RDPI Implement

You are running the Implement phase — executing the plan with code. You are the orchestrator: you coordinate sub-agents, track progress, and ensure each vertical slice is complete and verified before moving to the next.

**Tech stack:** {{TECH_STACK}}
**Build command:** {{BUILD_CMD}}
**Test command:** {{TEST_CMD}}

## Resolve Artifact Folder

The argument passed to this skill is a Task ID or short name (e.g., `US1234` or `US1234-fix-login`). Resolve it to the artifact folder before reading any files:

1. Glob `./rdpi/*{arg}*/` to find matching folders
2. **One match** → use it
3. **Multiple matches** → list them and ask the user to pick one
4. **No match** → error: "No artifact folder found for '{arg}'. Run /rdpi-research first."

## Inputs

Read the artifacts from previous phases (using the resolved folder path):
- `{folder}/03-plan/plan.md` — the execution plan (may be absent if Plan was skipped)
- `{folder}/01-research/spec.md` — requirements (always present)
- `{folder}/02-design/design.md` — architecture decisions (may be absent)

If the plan is absent, work directly from the Spec. For simple tasks this is fine — treat the Spec's acceptance criteria as your checklist.

Create the artifact directory: `{folder}/04-implement/`

## Step 1: Branch Setup

Before writing any code, ensure a clean starting point:

1. Check you're on a fresh branch from up-to-date main:
   ```
   git checkout main
   git pull
   git checkout -b {task-id}/{short-description}
   ```
2. If the user already has a branch, confirm it's up to date with main

This is non-negotiable — never implement on main or a stale branch.

## Step 2: Execute the Plan

Work through the plan's vertical slices in order.

For each slice:
1. **Implement** — write the code. For large slices, spawn a sub-agent (model: {{MODEL_CODERS}}) with the specific slice instructions from the plan. For small slices, implement directly.
2. **Verify** — run the verification steps from the plan (build, tests, manual checks)
3. **Commit** — use the commit message from the plan
4. **Log** — append progress to `04-implement/implementation-log.md`:
   ```markdown
   ## Slice N: [Name]
   - Status: complete / partial / blocked
   - Files changed: [list]
   - Tests: passing / failing / N/A
   - Notes: [anything unexpected]
   ```

### First slice — mandatory user review

After the first slice is complete and committed, pause and ask the user to review:

"First slice is done. Please test the result and confirm the direction is right before I continue. Check: [specific thing to verify]"

This catches misalignment early, before more code is written.

### Subsequent slices

{{#if MULTI_PHASE_DEFAULT == "multi"}}
Pause after each slice for user review by default.
{{else}}
Continue without pausing unless the plan marks a specific checkpoint or you hit an unexpected issue.
{{/if}}

If a slice fails verification:
1. Try to fix it (one attempt)
2. If the fix doesn't work, log the issue and ask the user before proceeding

## Step 3: Draft Pull Request

{{#if PR_WORKFLOW != "none"}}
After all slices are complete and verified:

1. Push the branch
2. Create a **Draft PR** with:
   - Title: task ID + short description
   - Body: summary from the Design (or Spec if no Design), list of changes, test results
   - Link to the artifact folder for context

Tell the user: "Draft PR is up at [link]. Please review the code — this is the code review checkpoint. When you're satisfied, promote it to Ready."

Save the PR link to `04-implement/pr-summary.md`.
{{else}}
No PR workflow configured. Tell the user the implementation is complete and suggest they review the changes with `git diff main`.
{{/if}}

{{#if DEPLOY_ENABLED}}
## Step 4: Deploy & Verify (opt-in)

Ask the user if they want to deploy for verification. If yes:
1. Run the deploy command
2. After deploy, check logs for errors
3. If browser testing is available, run a quick verification

This step is always opt-in — never deploy autonomously without explicit user approval.
{{/if}}

## Completion

Summarize what was done:

```
Implementation complete:
- [N] slices delivered
- [N] files created, [N] modified
- All tests passing
- {{#if PR_WORKFLOW != "none"}}Draft PR: [link]{{else}}Changes on branch: [branch-name]{{/if}}
```

## Recovery

If implementation is going sideways:
- If the issue is in the plan (wrong approach) → suggest going back to `/rdpi-plan`
- If the issue is in the design (wrong architecture) → suggest going back to `/rdpi-design`
- If the conversation has degraded → suggest restarting this phase. `implementation-log.md` preserves progress; completed slices are already committed.

## Next Phase

Implement is the final RDPI phase — there is no automatic next step. If you suggest going back to a previous phase (e.g. `/rdpi-plan` or `/rdpi-design`), print the command as plain text only. Do NOT use the Skill tool to invoke any phase automatically.
