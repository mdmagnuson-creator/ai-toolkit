---
id: prd-adhoc-design-decisions
title: Ad-hoc Design Decision Surfacing
status: draft
priority: high
createdAt: 2026-03-06T19:00:00Z
---

# PRD: Ad-hoc Design Decision Surfacing

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
0.1c  → Design Decision Detection & Questions (NEW)
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
- [ ] Decision detection result stored in `builder-state.json` → `activeTask.designDecisions`
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

- [ ] Dashboard includes a new `🎨 DESIGN DECISIONS` section (or similar) showing resolved choices
- [ ] Each decision shows the question, the user's choice, and what it means for implementation
- [ ] If user chose "you decide", show Builder's choice with brief rationale
- [ ] Design decisions section appears between PROBE RESULTS and SCOPE (or after SCOPE, before AFFECTED FILES)
- [ ] Decisions that affect scope are reflected in the SCOPE estimate
- [ ] Validate scripts pass

### US-005: Task Spec Integration

**Description:** As Builder, user's design decision answers are captured in the Task Spec and flow through to generated story acceptance criteria, so implementation agents have clear direction.

**Acceptance Criteria:**

- [ ] Task Spec Analysis section includes a "Design Decisions" subsection listing each resolved decision
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
- [ ] `builder-state.json` records `designDecisions: null` (skipped) vs `designDecisions: { ... }` (detected)
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

## Non-Goals (Out of Scope)

- Pattern-specific question templates or skill files (wizard questions, CRUD questions, etc.)
- Multi-round decision conversations — one round only
- Changing the MEDIUM/LOW clarifying questions mechanism (Step 0.3)
- Modifying the PRD recommendation threshold (Step 0.4)
- Adding decision detection to PRD mode (Planner already has multi-round refinement)

## Technical Considerations

- **Files modified:** Primarily `skills/adhoc-workflow/SKILL.md` (new Step 0.1c, updated Step 0.2 dashboard template, updated Step 0.5 Task Spec template)
- **Possible builder.md change:** May need to note design decision awareness in the analysis delegation context
- **Schema consideration:** `builder-state.json` gets a new `activeTask.designDecisions` field — update `task-state.schema.json` if it exists
- **Ordering sensitivity:** Step 0.1c uses probe results from 0.1b to inform decisions (e.g., "I see you have an existing wizard component — should this new wizard follow the same pattern?")

## Success Metrics

- Builder asks design decision questions for >80% of "in-between" requests (wizards, forms, CRUD, API endpoints with behavior choices)
- Builder skips questions for >90% of simple requests (bug fixes, typos, config changes)
- User answers design decisions in <60 seconds (quick lettered options)
- Rework due to wrong implementation assumptions decreases

## Open Questions

1. Should the `🎨 DESIGN DECISIONS` dashboard section use a different emoji/label? (e.g., `⚙️ DESIGN CHOICES`, `🔧 IMPLEMENTATION DECISIONS`)
2. When Builder detects decisions but the user has already specified some in their original request (e.g., "add a wizard with state persistence"), should those be shown as "pre-resolved" or omitted entirely?
3. Should there be a project.json flag to disable this feature for teams that prefer Builder to just use best judgment? (e.g., `agents.designDecisionQuestions: false`)

## Credential & Service Access Plan

No external credentials required for this PRD.
