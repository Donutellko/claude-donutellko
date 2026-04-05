---
name: rdpi-bootstrap
description: >
  Set up the RDPI (Research → Design → Plan → Implement) development workflow for a project.
  Generates 4 project-specific phase skills that guide AI-assisted development through context-engineered phases.
  Use this skill whenever the user wants to set up structured AI development workflow, bootstrap RDPI,
  configure development phases, or mentions "context engineering" / "QRSPI" methodology.
  Also trigger when the user says "set up rdpi", "bootstrap project", "configure AI workflow",
  "rdpi-bootstrap", or wants to organize AI-assisted development into phases with clean context boundaries.
  Also trigger on re-runs: "update rdpi", "reconfigure rdpi", "re-bootstrap".
---

# RDPI Bootstrap

You are setting up the RDPI development workflow for this project. RDPI splits AI-assisted development into 4 phases — Research, Design, Plan, Implement — each running in a fresh context with structured artifact handoff. This prevents context degradation and ensures quality.

The methodology is based on Dex Horthy's "Context Engineering" (QRSPI) and Dmitry Bereznitsky's "Process over Prompts."

## First Run vs Re-run

Check if `./rdpi/RDPI_SKILLS_SPEC.md` exists.

- **Exists** → Re-run flow (jump to "Re-run" section below)
- **Doesn't exist** → First run (continue here)

## First Run — 10 Steps

### Step 1: Discover the environment

Before asking the user anything, gather context silently:

1. **Scan the codebase** — identify tech stack (check `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, etc.), build tools, test framework, CI/CD config, existing CLAUDE.md files
2. **Discover available agents** — glob for `plugins/*/agents/*.md` and note agent names and descriptions
3. **Discover available skills** — check the system-reminder for available skills, glob for `plugins/*/skills/*/SKILL.md` and `.claude/skills/*/SKILL.md`
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

### Step 3: Auto-detect remaining values

For anything not explicitly asked, silently auto-detect from the codebase:
- Branch naming convention (from `git log`)
- Commit message style (from `git log`)
- Quality gates (from CLAUDE.md, CI config)
- Existing coding standards

### Step 4: Reserved (placeholder for future use)

### Step 5: Confirm ALL parameters (D-014 — mandatory)

**Do NOT silently substitute defaults.** Present a single summary of ALL parameters — both interview answers and auto-detected — with provenance. Use this exact format:

```
Here's what I've configured:

  Tech stack:      [value]                          (from [file])
  Build command:   [value]                          (from [file])
  Test command:    [value]                          (from [file])
  Task tracker:    [value]                          (user interview / default)
  PR workflow:     [value]                          (from [file] / user interview)
  Diagrams:        [value]                          (default / user choice)
  Folder format:   ./rdpi/{YYYY-MM-DD}-...          (default / user choice)
  Deploy:          [value]                          (default / from [file])
  Browser testing: [value]                          (default)
  Models:          [summary]                        (spec default / user override)
  Task types:      [value]                          (user interview)
  Multi-phase:     [value]                          (default / user choice)

  Anything to change? (Enter to confirm all)
```

This is the user's last chance to catch auto-detection mistakes before they propagate into generated skills. Wait for explicit confirmation before proceeding.

### Step 6: Save the spec

Write confirmed answers to `./rdpi/RDPI_SKILLS_SPEC.md`. Read the template from `references/spec-template.md` in this skill's directory and fill it in with:
- All confirmed values
- **Source column for every value** (D-015): `auto-detected ({file})`, `user interview`, `default`, `user override`
- Registration details (where skills are placed, method used)
- Model deviations from spec defaults must note the reason

### Step 7: Propose agent assignments

Based on discovered agents, propose which agents handle which RDPI roles. Use these defaults for model selection:

| Role | Default Model | Thinking |
|------|--------------|----------|
| Research: Questions | Opus | standard |
| Research: Blind sub-agent | Sonnet | standard |
| Research: Spec writing | Opus | extended |
| Design: Interactive document | Opus | extended |
| Design: Structure/Outline | Sonnet | extended |
| Plan: Generation | Sonnet | extended |
| Plan: Spec consistency | Sonnet | standard |
| Implement: Orchestrator | Opus | standard |
| Implement: Coders | Sonnet | standard |
| Implement: PR creation | Sonnet | standard |

Present the mapping to the user. If the user has budget constraints or model limits:
- Opus limited → swap to Sonnet + extended thinking
- Sonnet limited → swap to Haiku (warn about quality)

### Step 8: Generate the phase skills

**Default location:** `.claude/skills/` (auto-discovered by Claude Code, zero config).

For each phase, read the corresponding template from this skill's `references/` directory and customize it:

1. Read `references/research-template.md` → write `.claude/skills/rdpi-research/SKILL.md`
2. Read `references/design-template.md` → write `.claude/skills/rdpi-design/SKILL.md`
3. Read `references/plan-template.md` → write `.claude/skills/rdpi-plan/SKILL.md`
4. Read `references/implement-template.md` → write `.claude/skills/rdpi-implement/SKILL.md`
5. Read `references/readme-template.md` → write `.claude/skills/README.md`

When customizing templates, replace placeholders with project-specific values:
- `{{TECH_STACK}}`, `{{BUILD_CMD}}`, `{{TEST_CMD}}`, `{{TASK_TRACKER}}`
- `{{DIAGRAM_FORMAT}}`, `{{ARTIFACT_FOLDER_FORMAT}}`
- `{{MODEL_RESEARCH_MAIN}}`, `{{MODEL_RESEARCH_BLIND}}`, `{{MODEL_DESIGN}}`, etc.
- `{{AVAILABLE_AGENTS}}`, `{{PR_WORKFLOW}}`, `{{DEPLOY_ENABLED}}`, `{{MULTI_PHASE_DEFAULT}}`

Remove template sections that don't apply (e.g., remove Deploy section if deploy is disabled).

### Step 9: Ask about global registration (D-013)

After generating skills, ask: "Want to use these skills in other projects too?"

- **No (default)** → Done. Skills in `.claude/skills/` are auto-discovered, zero config needed.
- **Yes** → Create marketplace structure and register globally:

  1. Create `./rdpi/.claude-plugin/marketplace.json`
  2. Move skills to `./rdpi/rdpi-{project-name}/skills/` structure
  3. Add **absolute path** to `~/.claude/settings.json` under `extraKnownMarketplaces`
  4. Add to `.claude/settings.json` under `enabledPlugins`

  **Constraints (D-013):**
  - `extraKnownMarketplaces` only works in GLOBAL `~/.claude/settings.json`, NOT project-level
  - Marketplace paths must be ABSOLUTE (relative paths don't resolve)
  - Avoid `rdpi/rdpi/rdpi/skills` nesting — use `rdpi/rdpi-{project-name}/skills/`
  - `plugin.json` inside plugin dirs is NOT needed — everything is in `marketplace.json`

### Step 10: Present the result

Show the user what was created. Adapt paths based on Step 9 choice:

**If project-local (default):**
```
Created RDPI workflow:
  ./rdpi/RDPI_SKILLS_SPEC.md                — Your project preferences (with provenance)
  .claude/skills/rdpi-research/SKILL.md     — Research phase
  .claude/skills/rdpi-design/SKILL.md       — Design phase
  .claude/skills/rdpi-plan/SKILL.md         — Plan phase
  .claude/skills/rdpi-implement/SKILL.md    — Implement phase
  .claude/skills/README.md                  — Usage guide

To start using RDPI on a task:
  /rdpi-research <describe your task>

Note: RDPI is designed for medium-to-large tasks where upfront investment
pays off. For small fixes, direct prompts are faster.

If anything in the RDPI workflow doesn't fit your needs, run `/rdpi-bootstrap <description of desired changes>` to refine it.
```

**If global marketplace:**
```
Created RDPI workflow:
  ./rdpi/RDPI_SKILLS_SPEC.md                          — Your project preferences
  ./rdpi/rdpi-{name}/skills/rdpi-research/SKILL.md    — Research phase
  ... (other skills)
  Registered in ~/.claude/settings.json (extraKnownMarketplaces)
  Registered in .claude/settings.json (enabledPlugins)

If anything in the RDPI workflow doesn't fit your needs, run `/rdpi-bootstrap <description of desired changes>` to refine it.
```

---

## Re-run

When `./rdpi/RDPI_SKILLS_SPEC.md` already exists:

1. Read the existing spec and summarize what's currently configured (show the parameters with their sources)
2. Re-discover agents and skills (new ones may have been added)
3. Ask the user what they want to update — don't re-ask everything
4. Update only the changed parts of the spec (preserve unchanged sources)
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

- `references/spec-template.md` — RDPI_SKILLS_SPEC.md format (with provenance tracking)
- `references/research-template.md` — Research phase skill template
- `references/design-template.md` — Design phase skill template
- `references/plan-template.md` — Plan phase skill template
- `references/implement-template.md` — Implement phase skill template
- `references/readme-template.md` — Usage guide template

Read each template when generating the corresponding skill. Do not load all templates at once.
