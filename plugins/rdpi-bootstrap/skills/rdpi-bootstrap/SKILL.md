---
name: rdpi-bootstrap
description: >
  Set up the RDPI (Research → Design → Plan → Implement) development workflow for a project.
  Generates 4 project-specific phase skills that guide AI-assisted development through context-engineered phases.
  Use this skill whenever the user wants to set up structured AI development workflow, bootstrap RDPI,
  configure development phases, or mentions "context engineering" / "QRSPI" methodology.
  Also trigger when the user says "set up rdpi", "bootstrap project", "configure AI workflow",
  or wants to organize AI-assisted development into phases with clean context boundaries.
---

# RDPI Bootstrap

You are setting up the RDPI development workflow for this project. RDPI splits AI-assisted development into 4 phases — Research, Design, Plan, Implement — each running in a fresh context with structured artifact handoff. This prevents context degradation and ensures quality.

The methodology is based on Dex Horthy's "Context Engineering" (QRSPI) and Dmitry Bereznitsky's "Process over Prompts."

## First Run vs Re-run

Check if `./rdpi/RDPI_SKILLS_SPEC.md` exists.

- **Exists** → Re-run flow (jump to "Re-run" section below)
- **Doesn't exist** → First run (continue here)

## First Run

### Step 1: Discover the environment

Before asking the user anything, gather context silently:

1. **Scan the codebase** — identify tech stack, build tools, test framework, CI/CD config, existing CLAUDE.md files
2. **Discover available agents** — glob for `plugins/*/agents/*.md` and note agent names and descriptions
3. **Discover available skills** — check the system-reminder for available skills, glob for `plugins/*/skills/*/SKILL.md`
4. **Check for existing project-specific skills** — look for skills like `implement-feature`, `implement-story`, `implement-spec` that reflect project conventions. Extract useful context from them.

### Step 2: Interview the user

Use `AskUserQuestion` for each question. Go back and forth — confirm understanding before moving on. Present auto-detected values as confirmations ("I detected X — is that right?") rather than open questions.

**3 mandatory questions:**

1. **Task types** — "What will you use RDPI for?"
   - Feature implementation (recommended)
   - Bug fixing
   - Both features and bugs

2. **Model constraints** — "What's the most powerful model allowed?"
   - Opus (recommended — full power)
   - Sonnet (good balance of speed and quality)
   - Haiku (fast and cheap, quality tradeoff)
   - Follow up: "Is extended thinking allowed?" if Opus or Sonnet selected

3. **Confirmation of detected stack** — present what you found in Step 1 and ask the user to confirm or correct: tech stack, build/test commands, conventions.

**Optional questions** — ask only if not auto-detectable or if the defaults seem wrong:

4. Artifact folder format — default: `./rdpi/{YYYY-MM-DD}-{task-id}-{short-name}`. Confirm or customize.
5. PR workflow — "Do you create pull requests? Draft PRs?" Default: yes, Draft PR.
6. Task tracker — Jira / GitHub Issues / Linear / none. Check if there are clues in the repo (e.g., `.jira`, issue templates).
7. Diagram format — Mermaid (default) / PlantUML. Check existing docs for conventions.
8. Multi-phase delivery — "Default to single-phase for most tasks, multi-phase only for large ones?" Default: single-phase.
9. Deploy capability — "Can the agent deploy? What's the deploy process?" Default: no autonomous deploy.
10. Browser testing — "Is browser-based testing available for verification?" Default: no.

Keep the interview conversational. If the user says "just use defaults", accept that and move on.

### Step 3: Save the spec

Write the interview results to `./rdpi/RDPI_SKILLS_SPEC.md`. Read the template from `references/spec-template.md` in this skill's directory and fill it in with the user's answers and discovered context.

### Step 4: Propose agent assignments

Based on discovered agents, propose which agents handle which RDPI roles. Read the model selection table from `references/model-defaults.md` to determine the right model for each role.

Present the mapping to the user:

```
Agent Assignments:
- Research blind sub-agent: Explore agent (Sonnet)
- Design structure/outline: general-purpose agent (Sonnet + extended thinking)
- Plan spec-consistency: general-purpose agent (Sonnet)
- Implement orchestrator: main session (Opus)
- Implement coders: general-purpose agents (Sonnet)
- Implement PR: general-purpose agent (Sonnet)
```

If the user has budget constraints or model limits, adjust per the fallback rules:
- Opus limited → swap to Sonnet + extended thinking
- Sonnet limited → swap to Haiku (warn about quality)

### Step 5: Generate the phase skills

Create `./rdpi/skills/` directory. For each phase, read the corresponding template from this skill's `references/` directory and customize it:

1. Read `references/research-template.md` → write `./rdpi/skills/rdpi-research/SKILL.md`
2. Read `references/design-template.md` → write `./rdpi/skills/rdpi-design/SKILL.md`
3. Read `references/plan-template.md` → write `./rdpi/skills/rdpi-plan/SKILL.md`
4. Read `references/implement-template.md` → write `./rdpi/skills/rdpi-implement/SKILL.md`
5. Read `references/readme-template.md` → write `./rdpi/skills/README.md`

When customizing templates, replace placeholders with project-specific values:
- `{{TECH_STACK}}` — detected/confirmed tech stack
- `{{BUILD_CMD}}` — build command
- `{{TEST_CMD}}` — test command
- `{{TASK_TRACKER}}` — Jira/GitHub Issues/etc.
- `{{DIAGRAM_FORMAT}}` — Mermaid/PlantUML
- `{{ARTIFACT_FOLDER_FORMAT}}` — folder naming pattern
- `{{MODEL_RESEARCH_MAIN}}` — model for research main agent
- `{{MODEL_RESEARCH_BLIND}}` — model for blind research sub-agent
- `{{MODEL_DESIGN}}` — model for design
- `{{MODEL_PLAN}}` — model for plan
- `{{MODEL_IMPLEMENT}}` — model for implement orchestrator
- `{{MODEL_CODERS}}` — model for implementation coders
- `{{AVAILABLE_AGENTS}}` — list of discovered agents with descriptions
- `{{PR_WORKFLOW}}` — Draft PR / regular PR / none
- `{{DEPLOY_ENABLED}}` — yes/no
- `{{MULTI_PHASE_DEFAULT}}` — single/multi

Remove template sections that don't apply (e.g., remove Deploy section if deploy is disabled, remove bug-specific sections if task type is features only).

### Step 6: Present the result

Show the user what was created:

```
Created RDPI workflow:
  ./rdpi/RDPI_SKILLS_SPEC.md          — Your project preferences
  ./rdpi/skills/rdpi-research/SKILL.md — Research phase
  ./rdpi/skills/rdpi-design/SKILL.md   — Design phase
  ./rdpi/skills/rdpi-plan/SKILL.md     — Plan phase
  ./rdpi/skills/rdpi-implement/SKILL.md — Implement phase
  ./rdpi/skills/README.md              — Usage guide

To start using RDPI on a task:
  /rdpi-research <describe your task>

Note: RDPI is designed for medium-to-large tasks where upfront investment
pays off. For small fixes, direct prompts are faster.
```

Remind the user to register the generated skills if needed (add the `./rdpi/skills/` path to their plugin configuration or `.claude/settings.json`).

---

## Re-run

When `./rdpi/RDPI_SKILLS_SPEC.md` already exists:

1. Read the existing spec and summarize what's currently configured
2. Re-discover agents and skills (new ones may have been added)
3. Ask the user what they want to update — don't re-ask everything
4. Update only the changed parts of the spec
5. Regenerate only the affected phase skills
6. Show a diff summary of what changed

---

## Key Principles (pass these into generated skills)

These principles come from the source methodology and shape every generated skill:

1. **Clean context between phases** — each phase runs in a fresh session. Artifacts on disk are the only state transfer.
2. **Artifacts replace compaction** — everything important is in files, not in the context window.
3. **"Go back and forth with me"** — never assume, always confirm understanding through dialogue.
4. **Recommended options with annotations** — mark choices as (recommended), (fast), (risky), (safe).
5. **Under 40 instructions per skill** — delegate to sub-agents or references if approaching the limit.
6. **Non-biased research** — blind sub-agent gets questions only, not task context.
7. **Vertical slicing** — end-to-end slices, not horizontal layers.
8. **Bad trajectory recovery** — if the user has corrected you 2-3 times in a row, suggest restarting the phase.
9. **Human reads code** — code review happens in the PR, not just via artifacts.
10. **Right model for the right job** — Opus for decisions, Sonnet for execution, Haiku for monitoring.

---

## Reference Files

Templates for generated skills are in `references/` within this skill's directory:

- `references/spec-template.md` — RDPI_SKILLS_SPEC.md format
- `references/research-template.md` — Research phase skill template
- `references/design-template.md` — Design phase skill template
- `references/plan-template.md` — Plan phase skill template
- `references/implement-template.md` — Implement phase skill template
- `references/readme-template.md` — Usage guide template

Read each template when generating the corresponding skill. Do not load all templates at once.
