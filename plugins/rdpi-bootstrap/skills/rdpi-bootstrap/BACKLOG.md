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

### B-011: Artifacts Replace Compaction — Disable Auto-Compaction
Principle 2 says artifacts replace compaction. Should the generated skills explicitly instruct Claude to avoid auto-compaction? Or is this handled by the clean-context-per-phase design?

### ~~B-012: Spec Artifact Template~~ → Partially resolved
`RDPI_SKILLS_SPEC.md` template defined in SPEC.md (Registration + Parameter Provenance sections). Research phase `spec.md` template still needs definition (requirements, acceptance criteria, constraints, out of scope, complexity assessment).

---

## Source Material References

- `DEX_HORTHY_VIDEO.md` — First video summary: RPI, intentional compaction, dumb zone, trajectory
- `DEX_HORTHY_VIDEO_2.md` — Second video summary: QRSPI update, instruction budget, "read the code"
- `DEX_HORTHY_VIDEO_2_original_transcript.txt` — Original transcript of second video
- `DEX_HORTHY_VIDEO_2.pdf` — Presentation slides (47MB)
- `DMIRTY_BEREZNITSKI_VIDEO.md` — Bereznitsky video: R-D-P-I pipeline, multi-agent, quality gates, process over prompts
