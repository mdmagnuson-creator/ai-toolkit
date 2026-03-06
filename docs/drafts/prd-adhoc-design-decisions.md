---
id: prd-adhoc-design-decisions
title: Ad-hoc Design Decision Surfacing & Testing Consolidation
status: draft
priority: high
createdAt: 2026-03-06T19:00:00Z
---

# PRD: Ad-hoc Design Decision Surfacing & Testing Consolidation

## Problem Statement

Builder's ad-hoc mode has two operating speeds that leave a gap for "in-between" tasks:

1. **Small, clear requests** (HIGH confidence, small scope) — Builder runs analysis, shows the dashboard, user hits [G], work starts. No questions asked.
2. **Ambiguous or large requests** (MEDIUM/LOW confidence, or medium/large scope) — Builder asks clarifying questions about *what the user means* or recommends promoting to a full PRD.

The gap: tasks where Builder **understands what the user wants** (HIGH confidence) but the request contains **implicit design decisions** that will significantly affect implementation quality. These decisions aren't about *what* to build — they're about *how* to build it well.

### Example: "Add a 3-step wizard modal to create a new project"

Builder would correctly identify this as HIGH confidence, small-to-medium scope, and proceed directly to [G]. But buried in this request are decisions like:

- Should wizard state persist so users can leave and resume later?
- Per-step validation or validate everything at the final step?
- Can users navigate backward through completed steps?
- What happens on success — redirect, toast, close modal?
- Should there be a confirmation step before submission?

These aren't clarifications (Builder understands the request perfectly). They're **design decisions** that the user has opinions about but didn't think to specify upfront. Today, Builder either guesses silently (often wrong) or the user discovers the wrong choice after implementation.

### Why this matters

- **Rework cost:** Builder implements wizard with no state persistence → user wanted persistence → redo half the work
- **Quality gap:** The difference between "it works" and "it works well" often lives in these decisions
- **User trust:** When Builder asks the right questions upfront, users trust it more — it demonstrates understanding beyond the literal request
- **Not just UI:** This applies to API design decisions (pagination strategy?), data model decisions (soft delete vs hard delete?), scheduling decisions (retry policy?), and any implementation where reasonable people would choose differently

### Why existing mechanisms don't cover this

| Mechanism | Why it misses this |
|-----------|-------------------|
| **MEDIUM/LOW clarifying questions** | Only triggered by ambiguity in *what* to build, not *how* to build it. A wizard request is unambiguous. |
| **PRD promotion** | Overkill — these decisions can be answered in 30 seconds, not a multi-round PRD cycle. |
| **planning.considerations** | Covers project-level concerns (permissions, docs, compliance) but not request-specific implementation decisions. |
| **Alternatives section** | Shows *approach* alternatives (e.g., "modal vs full page") but not the micro-decisions within the chosen approach. |

## Goals

1. **Surface implicit decisions:** Builder identifies design/implementation decisions embedded in requests and asks about them before proceeding
2. **Lightweight interaction:** Single round of quick questions with lettered options — same UX as existing clarifying questions, not a PRD-level process
3. **Autonomous detection:** Builder infers when decisions exist based on code analysis and request characteristics — no hardcoded pattern catalog
4. **Feed into implementation:** User answers flow into both the Task Spec's analysis section AND the generated story acceptance criteria
5. **Consideration-aware:** `planning.considerations` from project.json generates additional decision questions when relevant to the request

## Non-Goals

- Creating pattern-specific skills (wizard-skill, CRUD-skill, etc.) — Builder should reason about decisions generically
- Replacing the existing MEDIUM/LOW clarifying questions flow — that addresses a different problem (ambiguity)
- Requiring questions for every request — trivial tasks ("fix the typo in the header") should still go straight through
- Multi-round question loops — one round of questions, user answers, proceed

## Alignment with Current Architecture

### Current Phase 0 Step Order

```
0.0   → Initialize Analysis Gate
0.0a  → Pre-Analysis Screenshot
0.1   → Time-Boxed Analysis (10 seconds)
0.1a  → Task Type Classification
0.1b  → Playwright Analysis Confirmation (probe)
0.2   → Show ANALYSIS COMPLETE dashboard
0.3   → Clarifying Questions (MEDIUM/LOW only)
0.4   → PRD Recommendation (medium/large scope)
0.5   → Generate Task Spec
```

### Proposed Step Order

Per user direction, design decisions should surface **before the ANALYSIS COMPLETE dashboard** so the dashboard reflects the user's choices:

```
0.0   → Initialize Analysis Gate
0.0a  → Pre-Analysis Screenshot
0.1   → Time-Boxed Analysis (10 seconds)
0.1a  → Task Type Classification
0.1b  → Playwright Analysis Confirmation (probe)
0.1c  → Implementation Decision Detection & Questions (NEW)
0.2   → Show ANALYSIS COMPLETE dashboard (now reflects user's design choices)
0.3   → Clarifying Questions (MEDIUM/LOW only — unchanged)
0.4   → PRD Recommendation (medium/large scope — unchanged)
0.5   → Generate Task Spec (now includes decisions as acceptance criteria)
```

### How 0.1c interacts with 0.3

These are **different concerns** and can both fire:

| Step | Trigger | Purpose | Example |
|------|---------|---------|---------|
| **0.1c** (Design Decisions) | Builder detects implementation decisions regardless of confidence | Surface *how* to build it | "Should wizard state persist?" |
| **0.3** (Clarifying Questions) | Confidence is MEDIUM or LOW | Clarify *what* to build | "Which caching layer do you mean?" |

A request could trigger **both**: MEDIUM confidence (ambiguous what) with design decisions (choices in how). In that case, 0.1c fires first, then the dashboard shows MEDIUM, then 0.3 fires for clarification.

### How `planning.considerations` integrates

The `planning.considerations` array in `project.json` already has `appliesWhen` tags and `scopeQuestions`. Design decision detection should:

1. Check each consideration's `appliesWhen` tags against the request characteristics
2. If a consideration is relevant, include its `scopeQuestions` as additional design decision questions
3. This means project-level concerns (permissions model, audit logging, compliance) automatically surface as questions when they apply

Example: If `planning.considerations` includes `{ id: "permissions", appliesWhen: ["data-access", "crud"], scopeQuestions: ["Which roles can perform this action?", "What happens on unauthorized access?"] }`, and the request involves creating a new record type, those questions appear alongside Builder's inferred questions.

---

## User Stories

### US-001: Design Decision Detection Logic

**Description:** As Builder, after completing code analysis (Step 0.1) and before showing the dashboard (Step 0.2), I detect whether the request contains implicit design/implementation decisions that the user should weigh in on.

**Acceptance Criteria:**

- [ ] New Step 0.1c added to adhoc-workflow skill between 0.1b and 0.2
- [ ] Detection runs after analysis completes (uses analysis results — affected files, scope, request type)
- [ ] Detection considers: request complexity, number of reasonable implementation variants, UX behavior choices, data lifecycle decisions, error handling strategies
- [ ] Detection is autonomous — no hardcoded list of "decision-rich patterns"
- [ ] If no decisions detected, step is silently skipped (no user-visible output)
- [ ] Decision detection result stored in `builder-state.json` → `activeTask.implementationDecisions`
- [ ] Validate scripts pass

### US-002: Design Decision Questions UI

**Description:** As a user, when Builder detects design decisions, I see a focused question prompt before the ANALYSIS COMPLETE dashboard, so I can make choices that inform the analysis.

**Acceptance Criteria:**

- [ ] Questions shown in the same format as existing clarifying questions (numbered with lettered options)
- [ ] Questions are specific and actionable — not vague ("How should it work?")
- [ ] Each question has 2-4 concrete options with brief explanations
- [ ] User can reply with letter codes (e.g., "1A, 2B, 3C") for speed
- [ ] User can reply "you decide" or similar to let Builder use best judgment
- [ ] Single round only — no follow-up questions
- [ ] Maximum 5 questions per request (prioritize highest-impact decisions)
- [ ] Decisions the user already specified in their request are omitted (not shown as "pre-resolved") — Builder detects these from request text and skips them
- [ ] After user answers, proceed to Step 0.2 with answers incorporated into the dashboard
- [ ] Validate scripts pass

### US-003: Planning Considerations Integration

**Description:** As Builder, I include relevant `planning.considerations` from project.json as additional design decision questions, so project-level concerns are surfaced alongside request-specific decisions.

**Acceptance Criteria:**

- [ ] Read `planning.considerations` from project.json during Step 0.1c
- [ ] Match each consideration's `appliesWhen` tags against request characteristics (e.g., CRUD operations, data access, admin features)
- [ ] For matching considerations, include `scopeQuestions` as additional design decision questions
- [ ] Consideration-sourced questions are labeled with the consideration ID for traceability
- [ ] Questions from considerations count toward the 5-question maximum (request-specific questions take priority)
- [ ] If no `planning.considerations` exist in project.json, step proceeds normally with only inferred questions
- [ ] Validate scripts pass

### US-004: Dashboard Reflects Design Choices

**Description:** As a user, the ANALYSIS COMPLETE dashboard I see after answering design decisions reflects my choices, so I can verify Builder understood my preferences before approving.

**Acceptance Criteria:**

- [ ] Dashboard includes a new `⚙️ IMPLEMENTATION DECISIONS` section showing resolved choices
- [ ] Each decision shows the question, the user's choice, and what it means for implementation
- [ ] If user chose "you decide", show Builder's choice with brief rationale
- [ ] Design decisions section appears between PROBE RESULTS and SCOPE (or after SCOPE, before AFFECTED FILES)
- [ ] Decisions that affect scope are reflected in the SCOPE estimate
- [ ] Validate scripts pass

### US-005: Task Spec Integration

**Description:** As Builder, user's design decision answers are captured in the Task Spec and flow through to generated story acceptance criteria, so implementation agents have clear direction.

**Acceptance Criteria:**

- [ ] Task Spec Analysis section includes an "Implementation Decisions" subsection listing each resolved decision
- [ ] Each decision that implies a specific behavior generates an acceptance criterion on the relevant story
- [ ] Example: "Wizard state persistence: Yes" → story acceptance criterion "Wizard progress is saved and restored when user navigates away and returns"
- [ ] Example: "Per-step validation: Yes" → story acceptance criterion "Each wizard step validates inputs before allowing Next"
- [ ] Consideration-sourced decisions also generate acceptance criteria using `acceptanceCriteriaHints` from project.json
- [ ] Validate scripts pass

### US-006: Skip Detection for Simple Requests

**Description:** As Builder, I skip design decision questions for requests that genuinely have no meaningful decisions, so trivial tasks aren't slowed down.

**Acceptance Criteria:**

- [ ] Skip criteria documented in the skill (when to skip)
- [ ] Examples of skip-worthy requests: bug fixes with clear root cause, typo corrections, version bumps, config changes, ops-only tasks
- [ ] Examples of decision-worthy requests: new UI components, new CRUD flows, new API endpoints, workflow changes, anything with UX behavior choices
- [ ] When skipped, no user-visible output — flow proceeds directly from 0.1b to 0.2
- [ ] `builder-state.json` records `implementationDecisions: null` (skipped) vs `implementationDecisions: { ... }` (detected)
- [ ] Validate scripts pass

---

## Functional Requirements

- FR-1: Step 0.1c MUST run after 0.1b (Playwright probe) and before 0.2 (dashboard), using the full analysis context including probe results
- FR-2: Decision detection MUST be autonomous — Builder infers decisions from request + code analysis, not from a hardcoded pattern catalog
- FR-3: Questions MUST be shown in a single round with lettered options, matching the existing clarifying questions UX
- FR-4: Maximum 5 questions per request, prioritized by implementation impact
- FR-5: User answers MUST be incorporated into both the ANALYSIS COMPLETE dashboard and the generated Task Spec
- FR-6: `planning.considerations` with matching `appliesWhen` tags MUST generate additional questions from their `scopeQuestions`
- FR-7: Decision answers that imply specific behaviors MUST generate acceptance criteria on relevant stories
- FR-8: The step MUST be silently skipped when no meaningful decisions are detected
- FR-9: User can opt out of decisions with "you decide" — Builder proceeds with best judgment and shows choices in dashboard
- FR-10: Decisions the user already specified in their request MUST be omitted from questions (not shown as "pre-resolved")

## Non-Goals (Out of Scope)

- Pattern-specific question templates or skill files (wizard questions, CRUD questions, etc.)
- Multi-round decision conversations — one round only
- Changing the MEDIUM/LOW clarifying questions mechanism (Step 0.3)
- Modifying the PRD recommendation threshold (Step 0.4)
- Adding decision detection to PRD mode (Planner already has multi-round refinement)

## Technical Considerations

- **Files modified:** Primarily `skills/adhoc-workflow/SKILL.md` (new Step 0.1c, updated Step 0.2 dashboard template, updated Step 0.5 Task Spec template)
- **Possible builder.md change:** May need to note design decision awareness in the analysis delegation context
- **Schema consideration:** `builder-state.json` gets a new `activeTask.implementationDecisions` field — update `task-state.schema.json` if it exists
- **Ordering sensitivity:** Step 0.1c uses probe results from 0.1b to inform decisions (e.g., "I see you have an existing wizard component — should this new wizard follow the same pattern?")

## Success Metrics

- Builder asks design decision questions for >80% of "in-between" requests (wizards, forms, CRUD, API endpoints with behavior choices)
- Builder skips questions for >90% of simple requests (bug fixes, typos, config changes)
- User answers design decisions in <60 seconds (quick lettered options)
- Rework due to wrong implementation assumptions decreases

## Resolved Questions

1. ~~Should the dashboard section use a different emoji/label?~~ → **Yes: `⚙️ IMPLEMENTATION DECISIONS`**
2. ~~When Builder detects decisions but the user has already specified some in their original request, should those be shown as "pre-resolved" or omitted entirely?~~ → **Omitted entirely.** Builder detects pre-resolved decisions from request text and skips them.
3. ~~Should there be a project.json flag to disable this feature?~~ → **No.** No kill switch. Builder always runs decision detection; it's just silently skipped when no decisions are found.

## Credential & Service Access Plan

No external credentials required for this PRD.

---

# Part 2: Testing & Quality Check Consolidation

## Problem Statement

The toolkit has accumulated significant duplication in its testing and quality-check definitions. The same "quality check pipeline" (typecheck → lint → test → rebuild → critic → Playwright) is defined in **four places**, with subtle differences in each:

| Location | Lines | What it defines |
|----------|-------|-----------------|
| `skills/adhoc-workflow/SKILL.md` | ~213 lines (§Per-Task Quality Checks) | Full quality pipeline for ad-hoc tasks |
| `skills/test-quality-checks/SKILL.md` | 355 lines | Standalone quality check definitions |
| `skills/prd-workflow/SKILL.md` | ~56 lines (§Per-Story Quality Checks) | Quality pipeline for PRD stories |
| `agents/builder.md` | ~56 lines (§Verification Pipeline) | Verification Pipeline Resolution (Steps 1-7) |

Additional duplication exists for:

- **`isUIProject()` detection** — defined in both `test-ui-verification` and `test-activity-resolution`
- **E2E URL resolution** — full implementation in `test-url-resolution`, duplicated inline in `test-e2e-flow` and `tester.md`
- **Architecture-aware rebuild/relaunch** — full implementation in `test-ui-verification`, duplicated in `adhoc-workflow` and summarized in `builder.md`
- **UI verification enforcement with skip patterns** — defined in `test-ui-verification`, duplicated in `adhoc-workflow` and `prd-workflow`
- **Playwright retry strategies** — different implementations in `prd-workflow` (5-attempt) vs `test-verification-loop` (3-attempt) with no shared base
- **Test execution mode (CI/non-watch)** — defined in both `test-activity-resolution` and `tester.md`

### Why this matters

1. **Drift risk:** When a quality check rule is updated in one location, the other three are often missed, creating inconsistent behavior between ad-hoc and PRD modes
2. **Token waste:** Builder loads both `adhoc-workflow` (1,947 lines) and `test-quality-checks` (355 lines), paying for the same pipeline twice
3. **Maintenance burden:** Every quality check change requires editing 4 files and verifying consistency
4. **Onboarding confusion:** Contributors can't determine which definition is canonical

### Consolidation Architecture

The 11 test skills break into two tiers:

**Tier 1 — Always needed (merge into single entry point):**

| Skill | Lines | Why always needed |
|-------|-------|-------------------|
| `test-flow` | 128 | Routing orchestrator — current entry point |
| `test-quality-checks` | 355 | Per-task quality checks, completion prompts |
| `test-activity-resolution` | 316 | Determines what tests to run based on file patterns |

**Tier 2 — Conditional (stay as separate skills, loaded on demand):**

| Skill | Lines | When loaded |
|-------|-------|-------------|
| `test-verification-loop` | 434 | Playwright verification failures |
| `test-e2e-flow` | 422 | E2E-specific execution flows |
| `test-ui-verification` | 733 | UI projects only |
| `test-failure-handling` | 326 | When test failures occur |
| `test-prerequisite-detection` | 428 | Classifying failure causes |
| `test-url-resolution` | 433 | E2E URL resolution |
| `test-user-cleanup` | 557 | Auth-configured projects |
| `test-doc-sync` | 137 | Behavior changes only |

**The consolidation:**

1. **`test-flow`** becomes the single canonical entry point for ALL story/task completion quality checks — both ad-hoc and PRD modes
2. **`test-quality-checks`** and **`test-activity-resolution`** are merged INTO `test-flow` (or absorbed via direct reference) since they're always needed together
3. **`adhoc-workflow`** replaces its ~213-line "Per-Task Quality Checks" section with a single `> 📚 SKILL: test-flow` reference
4. **`prd-workflow`** replaces its ~56-line "Per-Story Quality Checks" section with a single `> 📚 SKILL: test-flow` reference
5. **`builder.md`** Verification Pipeline Resolution references `test-flow` as the canonical source
6. **Shared functions** (`isUIProject()`, URL resolution, CI mode detection) are deduplicated to a single canonical location

## Goals (Part 2)

6. **Single source of truth:** One canonical quality-check pipeline that both ad-hoc and PRD modes reference
7. **Eliminate duplication:** Remove inline quality check definitions from `adhoc-workflow`, `prd-workflow`, and `builder.md`
8. **Preserve behavior:** The consolidated pipeline produces identical results — no functional changes
9. **Reduce token load:** Builder loads one skill instead of duplicated definitions across multiple files
10. **Clear tier separation:** Always-needed skills merged into entry point; conditional skills stay separate and are loaded on demand

## Non-Goals (Part 2)

- Changing test behavior or adding new test types
- Modifying the 8 Tier 2 conditional skills (beyond deduplicating `isUIProject()` and URL resolution if they appear inline)
- Changing how Builder delegates to `@tester` — that agent's orchestration role is unchanged
- Changing how `@developer` runs quality checks via project.json commands

---

## User Stories (Part 2: Testing Consolidation)

### US-007: Merge Always-Needed Skills into test-flow

**Description:** As the test-flow skill, I absorb the content from `test-quality-checks` and `test-activity-resolution` so that a single skill load provides the complete quality-check pipeline. I also absorb the skip-gate logic currently in `builder.md` Step 4, making test-flow the single unconditional entry point that Builder calls for every story/task completion.

**Acceptance Criteria:**

- [ ] `test-flow/SKILL.md` contains a **skip-gate** as its first step: checks changed files against skip patterns (docs-only, config-only, test-only, CI/build config, lockfile-only, user explicit skip) and exits early if all files match — moved from `builder.md` Step 4
- [ ] `test-flow/SKILL.md` contains the full quality-check pipeline (typecheck → lint → test → rebuild → critic → Playwright) currently in `test-quality-checks`
- [ ] `test-flow/SKILL.md` contains the activity resolution logic (file-pattern-based test selection, `isUIProject()`, CI mode) currently in `test-activity-resolution`
- [ ] `test-flow/SKILL.md` is organized with clear sections: Skip Gate, Activity Resolution, Quality Check Pipeline, Completion Prompt, Tier 2 Skill Loading
- [ ] `test-quality-checks/SKILL.md` is deleted — thorough grep audit confirms no remaining references
- [ ] `test-activity-resolution/SKILL.md` is deleted — thorough grep audit confirms no remaining references
- [ ] All existing references to `test-quality-checks` and `test-activity-resolution` in other agents/skills are updated to point to `test-flow`
- [ ] Validate scripts pass

### US-008: Replace adhoc-workflow Inline Quality Checks

**Description:** As the adhoc-workflow skill, my ~213-line "Per-Task Quality Checks" section is replaced with a skill reference to `test-flow`, eliminating the largest source of quality-check duplication.

**Acceptance Criteria:**

- [ ] The "Per-Task Quality Checks" section in `adhoc-workflow/SKILL.md` (approximately lines 1007-1220) is replaced with a `> 📚 SKILL: test-flow` reference block
- [ ] The reference block briefly describes what test-flow does and when it's loaded (1-3 sentences)
- [ ] Any adhoc-specific context that test-flow needs (e.g., ad-hoc task state, task spec reference) is passed as parameters described in the reference
- [ ] Architecture-aware rebuild/relaunch definitions that duplicate `test-ui-verification` are removed (reference that skill instead)
- [ ] UI verification enforcement that duplicates `test-ui-verification` is removed (reference that skill instead)
- [ ] Net reduction of ≥150 lines from `adhoc-workflow/SKILL.md`
- [ ] Validate scripts pass

### US-009: Replace prd-workflow Inline Quality Checks

**Description:** As the prd-workflow skill, my "Per-Story Quality Checks" section is replaced with a skill reference to `test-flow`, and my "UI Verification Enforcement" section references `test-ui-verification` instead of duplicating it.

**Acceptance Criteria:**

- [ ] The "Per-Story Quality Checks (MANDATORY)" section in `prd-workflow/SKILL.md` (approximately lines 234-290) is replaced with a `> 📚 SKILL: test-flow` reference block
- [ ] The "UI Verification Enforcement" section (approximately lines 619-707) is replaced with a `> 📚 SKILL: test-ui-verification` reference block
- [ ] PRD-specific context (story ID, PRD reference, 5-attempt retry strategy) is described as parameters in the reference
- [ ] The 5-attempt vs 3-attempt retry difference between PRD and ad-hoc modes is documented in one canonical location (either `test-flow` or `test-verification-loop`)
- [ ] Net reduction of ≥100 lines from `prd-workflow/SKILL.md`
- [ ] Validate scripts pass

### US-010: Update builder.md Verification Pipeline References

**Description:** As builder.md, my "Verification Pipeline Resolution" section (Steps 1-7) is replaced with a single unconditional call to `test-flow`. Builder no longer decides *when* or *how* to verify — it always calls test-flow, which owns the full decision tree including skip conditions.

**Acceptance Criteria:**

- [ ] "Verification Pipeline Resolution" section in `builder.md` (approximately lines 1135-1191) is replaced with a single mandatory step: "Load `test-flow` skill and execute"
- [ ] The MANDATORY marker is preserved — Builder MUST call test-flow before every commit or task completion declaration
- [ ] All skip-gate logic (Step 4) is removed from builder.md (now lives in test-flow)
- [ ] All pipeline resolution logic (Steps 1-3) is removed from builder.md (now lives in test-flow)
- [ ] Story-scoped Playwright (Step 5), retry strategy (Step 6), and ops-only verification (Step 7) details are removed from builder.md (now lives in test-flow or referenced sub-skills)
- [ ] Builder.md retains only the one-liner: unconditionally call test-flow
- [ ] Net reduction of ≥40 lines from `builder.md`
- [ ] Validate scripts pass

### US-011: Deduplicate isUIProject() and Shared Detection Functions

**Description:** As the toolkit, shared detection functions that are duplicated across skills are consolidated to a single canonical location.

**Acceptance Criteria:**

- [ ] `isUIProject()` detection logic exists in exactly one skill (canonical location: `test-flow` since it's always loaded)
- [ ] `test-ui-verification/SKILL.md` references the canonical `isUIProject()` from `test-flow` instead of redefining it
- [ ] `test-activity-resolution` (now merged into `test-flow`) no longer has a separate `isUIProject()` definition
- [ ] E2E URL resolution logic exists canonically in `test-url-resolution/SKILL.md` only — inline duplicates in `test-e2e-flow` and `tester.md` are replaced with skill references
- [ ] Test execution mode (CI/non-watch) logic exists canonically in `test-flow` only — inline duplicate in `tester.md` is replaced with a skill reference
- [ ] Validate scripts pass

### US-012: Verify Consolidation Preserves Behavior

**Description:** As the toolkit maintainer, I verify that the consolidated testing pipeline produces identical behavior to the pre-consolidation definitions, with no quality checks dropped or added.

**Acceptance Criteria:**

- [ ] Create a verification document (`docs/testing-consolidation-verification.md`) that maps each pre-consolidation quality check step to its post-consolidation location
- [ ] Every quality check in the pre-consolidation `adhoc-workflow` "Per-Task Quality Checks" section has a corresponding step in `test-flow`
- [ ] Every quality check in the pre-consolidation `prd-workflow` "Per-Story Quality Checks" section has a corresponding step in `test-flow`
- [ ] Every step in the pre-consolidation `builder.md` "Verification Pipeline Resolution" is accounted for, including: skip conditions (Step 4), postChangeWorkflow override (Step 1), auto-inference (Step 2), story-scoped Playwright (Step 5), retry strategy (Step 6), and ops-only verification (Step 7)
- [ ] The 5-attempt (PRD) vs 3-attempt (ad-hoc) retry difference is preserved and clearly documented
- [ ] PRD-specific behaviors (story blocking, E2E deferral) are preserved
- [ ] Ad-hoc-specific behaviors (Task Spec reference, single-task scope) are preserved
- [ ] Validate scripts pass

---

## Functional Requirements (Part 2)

- FR-11: `test-flow` MUST be the single canonical skill loaded for quality checks in both ad-hoc and PRD modes
- FR-12: `adhoc-workflow` and `prd-workflow` MUST NOT contain inline quality check pipeline definitions — they reference `test-flow`
- FR-13: `builder.md` Verification Pipeline MUST be reduced to a single unconditional call to `test-flow` — all skip logic, pipeline resolution, and retry strategies live in test-flow or its sub-skills
- FR-14: `isUIProject()` MUST be defined in exactly one location and referenced elsewhere
- FR-15: E2E URL resolution MUST be defined canonically in `test-url-resolution` and NOT duplicated inline in other skills
- FR-16: The consolidated pipeline MUST produce identical behavior to the current separate definitions — no quality checks dropped or added
- FR-17: PRD mode's 5-attempt retry and ad-hoc mode's 3-attempt retry MUST both be supported via a single retry mechanism with configurable attempt count
- FR-18: Tier 2 conditional skills MUST remain separate and loaded on demand — do NOT merge them into `test-flow`

## Technical Considerations (Part 2)

- **File size risk:** Merging 3 skills plus the skip-gate logic into `test-flow` will make it ~850 lines. This is within acceptable range for a skill that is always loaded, but should be monitored. If it grows beyond 1000 lines, consider re-splitting.
- **Skip-gate ownership:** The skip conditions currently in `builder.md` Step 4 move into `test-flow`. This means test-flow is truly unconditional from Builder's perspective — Builder always calls it, test-flow decides whether to actually run.
- **Skill trigger updates:** `opencode.json` skill triggers for `test-quality-checks` and `test-activity-resolution` need to be redirected to `test-flow` or removed
- **Cross-reference audit:** After consolidation, grep for `test-quality-checks` and `test-activity-resolution` references across ALL agents and skills to ensure none were missed
- **Ordering dependency:** US-007 must be completed before US-008, US-009, and US-010 (they depend on the consolidated `test-flow` existing)

## Implementation Order

```
US-007 (merge into test-flow)
  ├── US-008 (adhoc-workflow cleanup)     ─── can run in parallel
  ├── US-009 (prd-workflow cleanup)       ─── can run in parallel
  ├── US-010 (builder.md cleanup)         ─── can run in parallel
  └── US-011 (shared function dedup)      ─── can run in parallel
         └── US-012 (verification)        ─── must run last
```

## Success Metrics (Part 2)

- Quality check pipeline defined in exactly 1 location (down from 4)
- `adhoc-workflow/SKILL.md` reduced by ≥150 lines
- `prd-workflow/SKILL.md` reduced by ≥100 lines
- `builder.md` reduced by ≥40 lines
- `isUIProject()` defined in exactly 1 location (down from 2)
- E2E URL resolution defined in exactly 1 location (down from 3)
- Zero behavioral differences between pre- and post-consolidation quality checks

## Credential & Service Access Plan

No external credentials required for this PRD.
