# RDPI Bootstrap — Backlog

Open questions, unresolved items, and future improvements. Items move to SPEC.md when resolved or to DECISIONS.md when a decision is made.

---

## Unresolved from Spec (Open Questions)

### ~~B-001: Skill Registration Mechanism~~ → Resolved in D-013
Generated skills go to `.claude/skills/` (auto-discovered). Three options: project-local (default), global, existing plugin.

### B-002: Missing Phase Artifacts
What happens when a phase skill is invoked but previous phase artifacts are missing? Options: error with guidance, or offer to run the missing phase.

---

## Unspecified Details

### B-003: MANIFEST.md Format
MANIFEST.md is mentioned as "Navigation index with status of each phase" but the format is undefined. Questions:
- What statuses? (pending / in-progress / complete / skipped)
- Who writes it? (each skill appends its section)
- Is it machine-readable or just for humans?

### B-004: Resume After Interruption
What if a user interrupts a phase midway and restarts? Strategy needed:
- Implement: read `implementation-log.md` and continue from last completed slice
- Design: re-read `design.md` draft if it exists
- Research: re-read `questions.md` if questions were already formulated
- General: each skill should check for existing partial artifacts on start

### B-005: Agent Discovery Mechanism
"Enumerate all agents and skills available" — but how? Options:
- Parse `system-reminder` messages in context for available skills/agents
- Glob `plugins/*/agents/*.md` and `plugins/*/skills/*/SKILL.md`
- Use `ToolSearch` tool
- Read `.claude/settings.json` for registered plugins
- Some combination of the above

### B-006: End-to-End Example
No concrete example of the full flow from ticket to PR. Adding one would make the spec much more tangible. Should cover:
- A medium-complexity feature (standard depth)
- All 4 phases with sample artifacts
- The actual commands the user types

### B-007: Bug Fixing Flow — Concrete Differences
Scope says "Bug fixing workflow (adapted RDPI — lighter Design, focused Research)" but never specifies HOW. Proposed:
- Research: Questions focus on reproduction, root cause, affected areas
- Design: always Light depth (ADR + "why fix it this way")
- Plan: typically single phase
- Complexity assessment: likely recommends skipping Design

---

## Future Improvements

### B-008: Team Workflow
From devil's advocate (D-13): after Design phase, optionally share design with code owners for pre-alignment before proceeding to Plan. Bootstrap asks solo/team but doesn't operationalize the difference.

### B-009: "Go Back and Forth With Me" as Literal Phrase
User requested this phrase be literally present in the meta-skill and in the generated Research skill. Currently captured as Principle 4 but not as a literal instruction in the skill template.

### B-010: Refactoring / Migration Workflows
Currently out of scope. When added:
- Refactoring: Research focused on impact analysis, Design on before/after patterns
- Migration: Research on source/target systems, Design on migration strategy + rollback

### B-011: Big Picture Agent — Related Ticket Context
After reading a ticket by ID, the Research phase should spawn a sub-agent that builds the "big picture":
- Read the current ticket plus related tickets (parent, siblings, children)
- Check the artifacts directory for existing implementations of related tickets
- Summarize: what's already done, what's in progress, how the current ticket fits into the broader initiative
This gives the agent context on dependencies, prior decisions, and potential conflicts before diving into research.

### B-018: Structured Introduction + Closing Pattern for rdpi-bootstrap

The skill execution should follow this pattern:

**1. Opening — show intro immediately (before any questions)**
Show the two-table introduction we formulated:
- Table 1: commands per phase
- Phrase: "Clear context between each phase..."
- One-sentence QRSPI explanation
- Table 2: sub-phases with who/output/user action
- Note: Design and Plan are optional based on complexity
End with: "Now I'll ask you a few questions and scan the codebase to tailor these skills to your project."

**2. Interview + research (Steps 1–4 from skill)**
Run as normal.

**3. Pre-confirm deviation summary**
Before showing the full settings confirmation table (Step 5), output a concise line if anything will differ from the default workflow shown in the intro. Examples:
- "No Deploy/Verify step — not configured for this project."
- "Design phase skipped by default — you selected Sonnet only."
- "Multi-phase delivery enabled — Plan will include slice breakdown."
If nothing differs: skip this entirely.

**4. Closing — show commands table again**
After skill generation is complete (Step 10), repeat Table 1 (commands only).
End with: "Easy to remember — it's in the name: **R**DPI. Always start with **R**: `/rdpi-research`."

**5. README.md**
The same two tables (commands + sub-phases) must also be written to `.claude/skills/README.md` (or the marketplace equivalent). This is the persistent reference — the intro shown during bootstrap is ephemeral, the README stays.

**Why this pattern works:**
- User sees the full picture before committing to the interview
- Deviations are surfaced explicitly, not buried in the settings table
- Closing table is the actionable takeaway — user has it fresh when ready to start

### B-015: Concise Communication Style
The skill must communicate tersely throughout — matching caveman-style rules from user's CLAUDE.md. Specific violations to avoid:
- Long paragraphs explaining what it's about to do
- Verbose confirmations after tool output
- Restating what the user said
Applied to: interview questions, parameter confirmation, status updates, next-step prompts.

### ~~B-016: Workflow Outline After Settings Confirmation~~ → Resolved by B-018
After the user confirms auto-determined settings (D-014 confirm step), the skill should output a concise workflow outline showing what RDPI will look like for THIS project. Not generic docs — personalized to the confirmed settings:
- Commands the user will type per phase
- What each phase produces (artifact names, paths)
- Which phases are optional based on complexity
- Any project-specific notes (task tracker, branch format, deploy step)
Format: table or numbered list, max ~20 lines. Purpose: user sees the full picture before committing.

### B-017: Phase Skills Must Not Invoke Next Phase
Each generated phase skill must end with the next command as plain text — not invoke it. The skill should print `/rdpi-design ./rdpi/{folder}` and stop. It must never call `Skill` tool or otherwise trigger the next phase automatically. User runs the next command in a fresh session.

### B-019: Re-run Prompt After Bootstrap Completion
After skill generation is complete, add this line:
"If anything in the RDPI workflow doesn't fit your needs, run `/rdpi-bootstrap <description of desired changes>` to refine it."

### B-020: Branch Format Missing from Confirm Table
Branch naming convention is auto-detected in Step 3 but never shown in the Step 5 confirmation table. Add it:
```
Branch format:   feature/{ticket-number}-{short-name}    (auto-detected: git log / default)
```
Default if not detectable from git log: `feature/{ticket-number}-{short-name}`. Also write to `RDPI_SKILLS_SPEC.md` under Discovered Conventions and pass into generated phase skills (Implement phase uses it for branch creation).

### B-021: Extract and Surface Useful Instructions from Existing Project Skills
Step 1 already discovers existing skills like `implement-feature`, `implement-story`, `implement-spec`. Currently just "extract useful context" with no further spec. This needs to be concrete:

**What to do:**
- Read each discovered project-specific skill
- Extract: coding conventions, quality gates, agent usage patterns, workflow constraints, project-specific rules
- Write extracted items to `RDPI_SKILLS_SPEC.md` under a new "Inherited from Existing Skills" section
- Show a concise summary to the user in the confirm step (Step 5):
  ```
  Inherited from existing skills:
    implement-feature:   uses wsbaser:code-simplifier after each slice
    implement-story:     requires GHA CI green before PR ready
    ...
  ```
- Incorporate relevant extracted instructions into the generated phase skills

**Why:** Existing skills encode hard-won project conventions. Ignoring them means regenerating rules that are already known.

### B-024: Simpler Command Format — Task ID Instead of Folder Path
Current commands after Research are unfriendly:
```
/rdpi-design ./rdpi/2026-04-06-US1234-fix-login
```
New format — user passes Task ID or short name, skill resolves the folder:
```
/rdpi-research US1234-fix-login
/rdpi-design US1234
/rdpi-plan US1234
/rdpi-implement US1234
```
Each phase skill:
1. Accepts a Task ID or description as argument (not a folder path)
2. Resolves the artifact folder by globbing `./rdpi/*{task-id}*/` — picks the most recent match
3. If ambiguous (multiple matches), lists them and asks the user to confirm
4. If no match found for Design/Plan/Implement, errors with: "No artifact folder found for '{arg}'. Run /rdpi-research first."

Update in: SKILL.md Step 0 intro table, all 4 phase templates (references/), SPEC.md phase descriptions.

### B-025: Generated Phase Skill Descriptions Must Be Human-Facing
RDPI phase skills are invoked by the user, not auto-triggered by LLM. The `description:` field in each generated SKILL.md frontmatter should be short and human-readable — not a trigger-detection prompt.

Proposed descriptions:
- **rdpi-research:** Human part first: "Start your task implementation. Paste the ticket ID or description and answer questions. Produces: `spec.md`." Then LLM trigger part appended (not human-readable): auto-trigger when user pastes issue ID, ticket number, or describes a task to implement.
- **rdpi-design:** "Define what to build. Reviews spec, interactive design session. Produces: `structure-outline.md` (review required)." — human-invoked only, no auto-trigger.
- **rdpi-plan:** "Convert design into execution plan. Produces: `plan.md` + brief summary for you to review." — human-invoked only.
- **rdpi-implement:** "Execute the plan. Runs code, creates Draft PR. Review first slice + final PR." — human-invoked only.

Description format: human-readable sentence first, LLM trigger keywords appended after a separator (e.g. a blank line or `---`) so it doesn't appear in UI but is still parsed by the model.

Also applies to `rdpi-bootstrap` itself — one-time setup skill, no auto-trigger.

Update in: all 4 phase templates (`references/`), and the bootstrap SKILL.md description field.

### B-022: Multi-Service Scope Question
In monorepos with multiple independent services (e.g. 7 agents each with their own Makefile, pyproject.toml, CLAUDE.md), bootstrap must ask: "Which service(s) will you use RDPI for?" before generating skills. Generated skills must be scoped to a specific service root — not repo root — so `make check`, `make tests`, and file paths resolve correctly.
**What to detect:** multiple `pyproject.toml` or `Makefile` files at depth >1 → trigger the question.
**Output:** write chosen service path(s) to RDPI_SKILLS_SPEC.md. Generated skills use that path as working root.

### B-023: Missing Fields in Confirm Table
Add to Step 5 confirmation table:
```
Branch format:   feature/US{id}-{short-name}     (auto-detected: git log / CLAUDE.md)
Commit style:    type: message                    (auto-detected: git log)
Ticket format:   US{id}                           (auto-detected: CLAUDE.md / Rally)
Quality gate:    ruff + mypy + pytest + cov ≥30%  (auto-detected: Makefile)
```
Branch format default if not detectable: `feature/{ticket-id}-{short-name}`.
All four fields written to RDPI_SKILLS_SPEC.md under Discovered Conventions.

### B-026: Pre-Branch Git State Check in Implement

Before creating a feature branch, the generated `rdpi-implement` skill must check git state:

1. **Current branch** — must be on `main`. If not, warn and ask user if they want to switch.
2. **Uncommitted changes** (`git status --short`) — if any, stop and ask: commit, stash, or discard?
3. **Unpushed commits** (`git log @{u}..HEAD --oneline`) — if any, notify and ask: push first or proceed?
4. **Sync with remote** (`git fetch origin main` + compare) — if behind, pull automatically. If ahead or diverged, notify and ask.

Only after all checks pass: `git checkout -b feature/{task-id}-{short-description}`.

Update in: generated `rdpi-implement` skill template (`references/implement-template.md`), Step 1: Branch Setup.

---

### B-013: Artifacts Replace Compaction — Disable Auto-Compaction
Principle 2 says artifacts replace compaction. Should the generated skills explicitly instruct Claude to avoid auto-compaction? Or is this handled by the clean-context-per-phase design?

### ~~B-014: Spec Artifact Template~~ → Partially resolved
`RDPI_SKILLS_SPEC.md` template defined in SPEC.md (Registration + Parameter Provenance sections). Research phase `spec.md` template still needs definition (requirements, acceptance criteria, constraints, out of scope, complexity assessment).

---

## Source Material References

- `DEX_HORTHY_VIDEO.md` — First video summary: RPI, intentional compaction, dumb zone, trajectory
- `DEX_HORTHY_VIDEO_2.md` — Second video summary: QRSPI update, instruction budget, "read the code"
- `DEX_HORTHY_VIDEO_2_original_transcript.txt` — Original transcript of second video
- `DEX_HORTHY_VIDEO_2.pdf` — Presentation slides (47MB)
- `DMIRTY_BEREZNITSKI_VIDEO.md` — Bereznitsky video: R-D-P-I pipeline, multi-agent, quality gates, process over prompts
