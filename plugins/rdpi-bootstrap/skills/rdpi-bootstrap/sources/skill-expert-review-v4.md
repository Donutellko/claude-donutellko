# Skill Creation Expert Review — Spec v4

**Date:** 2026-03-30
**Reviewed:** SPEC.md v4 (Draft v4)
**Perspective:** Skill architecture and practical implementability

## Findings

### E-1: Meta-Skill Violates Its Own Instruction Budget | CRITICAL
**Finding:** Spec requires ≤40 instructions per phase skill. But rdpi-bootstrap itself does: discovery, 15-question interview, spec writing, agent mapping, generation of 4 skills + README. This is one of the most complex skills imaginable — and must fit the same budget.
**Recommendation:** Either exempt the meta-skill from the 40-instruction limit (with justification), or split bootstrap into sub-steps with progressive disclosure via `references/`.

### E-2: Generated Skills Are Stale From Birth | CRITICAL
**Finding:** Skills are generated once from templates and immediately start drifting: new agents appear, stack changes, conventions evolve. Re-run is described but:
- No drift detection — user doesn't know when to re-run
- No partial update — one change (e.g., added CI) forces full re-interview
- No artifact migration — what happens to in-progress task artifacts when skills are regenerated?
**Recommendation:** Add lightweight drift detection (hash of discovery results vs. stored hash). Support partial re-run with targeted questions. Define artifact compatibility policy.

### E-3: Template Approach Doesn't Scale to Project Diversity | HIGH
**Finding:** Spec assumes 4 templates with variable substitution cover everything — from Rust CLI to React+Django monorepo to data pipelines. In practice:
- Design for frontend (wireframes) vs backend (API contracts) vs data (schema diagrams) are fundamentally different
- Research for embedded vs web is a different process
- Templates will either be too generic (useless) or need conditional logic that blows past 40 instructions
**Recommendation:** Consider a "flavors" system — 2-3 project archetypes (web app, CLI/library, data pipeline) with different template sets. Or make templates more skeletal and rely on the model's judgment.

### E-4: Interview Is Too Long (15 Questions) | HIGH
**Finding:** Users want "set up AI workflow", not an interrogation. Real users will drop off at question 5-6. Spec mentions "smart defaults" but doesn't define:
- Which questions can be answered from codebase scan?
- Which can be skipped with defaults?
- What's the minimum viable question set?
**Recommendation:** Split into 3 mandatory (task types, stack, model constraints) + 12 optional with auto-detected defaults. Present optional questions as "I've detected X — is that right?" confirmations, not open questions.

### E-5: "Clean Context Between Phases" Is Aspirational | HIGH
**Finding:** The spec's core architecture assumes users clear context between phases. But:
- Users forget or don't want to
- Users may want to reference previous phase context
- No detection mechanism if a phase skill is invoked in dirty context
- No graceful degradation strategy
**Recommendation:** Add a context check at the start of each phase skill (e.g., check if the skill was invoked as the first action in the conversation). If dirty context detected, warn the user and suggest clearing. Don't hard-fail — degrade gracefully.

### E-6: Sub-Agent Failure Strategy Is Absent | HIGH
**Finding:** Full pipeline spawns ~8-10 sub-agents (blind research, structure/outline, spec consistency, coders, PR, deploy). But:
- No timeout handling
- No quality gate on sub-agent output
- No retry strategy
- No fallback if a sub-agent returns garbage
- No guidance on what to do if spec consistency checker finds issues after plan is partially executed
**Recommendation:** Each sub-agent call should have: expected output structure, minimum quality criteria, and a fallback (retry once, then surface to user). Add this to the phase templates.

### E-7: No Feedback Loop From Usage to Skills | MEDIUM
**Finding:** After 3-5 tasks, users will know what works and what doesn't ("Design is always too long", "blind research asks wrong questions"). No mechanism to feed experience back into generated skills without a full re-run of bootstrap.
**Recommendation:** Consider a lightweight "tune" command that reads user feedback and adjusts specific sections of generated skills without full re-interview. Or at minimum, document that users can edit generated skills directly.

### E-8: Bug Workflow Is a Significant Gap | MEDIUM
**Finding:** In-scope: "Bug fixing workflow (adapted RDPI — lighter Design, focused Research)". But the spec contains zero concrete differences from feature flow. B-007 in backlog acknowledges this, but for a skill that claims bug support, this is a spec gap, not a backlog item. Bugs are >50% of developer work.
**Recommendation:** Define at least: Research questions focus (reproduction, root cause, affected areas), Design depth (always Light), default phase skipping recommendation (skip Design for most bugs).

### E-9: Deploy & Verify — Either Spec It or Cut It | MEDIUM
**Finding:** Listed as "speculative, Dex did not cover" but present as Step 4 in Implement. A half-specified step in a template is dangerous — the model will improvise.
**Recommendation:** Either fully spec it (discovery, safety checks, rollback plan) or remove from v4 and add to backlog as a future extension. Don't ship half-baked.

### E-10: No Testing Strategy for Generated Skills | MEDIUM
**Finding:** Bootstrap generates 4 files and says "here, use them." If the research skill is broken (bad blind agent prompts), the user discovers this only on first real use. Even one smoke test would dramatically improve confidence.
**Recommendation:** After generation, offer to run a minimal smoke test: invoke Research on a trivial task (e.g., "add a comment to file X") to verify the pipeline works end-to-end. This also serves as a demo for the user.
