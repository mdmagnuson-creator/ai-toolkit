# PRD: Ad-hoc Analysis Phase

## Introduction

Currently, when a user asks Builder to do something in ad-hoc mode (outside of a PRD), Builder immediately starts implementing. This can lead to:

- **Misunderstood requirements** — Builder assumes intent without clarification
- **Missed context** — Changes that conflict with existing patterns or architecture
- **Scope creep** — Small requests that should be larger planned features
- **Downstream consequences** — Breaking changes, migration needs, or dependency impacts not considered

This PRD adds an **Analysis Phase** to ad-hoc mode that mirrors the PRD planning process, giving users visibility into what Builder understands, what alternatives exist, and whether the request warrants a full PRD.

## Goals

- Ensure Builder understands the request before implementing
- Surface potential conflicts with existing code/architecture
- Present alternative approaches when relevant
- Identify downstream consequences (breaking changes, migrations, etc.)
- Recommend PRD creation for larger requests
- Maintain speed for truly quick fixes (provide fast-path for trivial requests)

## User Stories

### US-001: Request Analysis

**Description:** As a user, I want Builder to analyze my ad-hoc request before implementing so that I can confirm it understands what I want.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] When user enters ad-hoc mode with a request, Builder does NOT immediately start implementing
- [ ] Builder reads `project.json` and relevant source files to understand context
- [ ] Builder outputs a brief "Understanding" section summarizing what it thinks the user wants
- [ ] Understanding includes: affected files/components, scope estimate (trivial/small/medium/large)
- [ ] User can confirm understanding or clarify before implementation begins

### US-002: Clarifying Questions

**Description:** As a user, I want Builder to ask clarifying questions when my request is ambiguous so that it doesn't make wrong assumptions.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies ambiguities in the request (scope, approach, behavior)
- [ ] Builder asks 1-3 focused clarifying questions with lettered options (like PRD mode)
- [ ] Questions are skipped for unambiguous requests (fast-path)
- [ ] User can respond with letters (e.g., "1A, 2B") for quick answers
- [ ] Builder incorporates answers into implementation plan

### US-003: Alternative Approaches

**Description:** As a user, I want Builder to present alternative approaches when relevant so that I can choose the best solution.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies when multiple valid approaches exist
- [ ] Builder presents 2-3 alternatives with pros/cons
- [ ] Each alternative includes estimated complexity and impact
- [ ] User can select an approach or ask for more detail
- [ ] For trivial requests with one obvious approach, alternatives section is skipped

### US-004: Downstream Consequences

**Description:** As a user, I want Builder to warn me about downstream consequences so that I understand the full impact of my request.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies breaking changes (API changes, schema changes, etc.)
- [ ] Builder identifies migration needs (database, config, etc.)
- [ ] Builder identifies dependency impacts (other components that use affected code)
- [ ] Builder identifies test impacts (tests that will need updating)
- [ ] Consequences are shown before implementation, not after
- [ ] User can acknowledge consequences or reconsider the request

### US-005: PRD Recommendation

**Description:** As a user, I want Builder to recommend creating a PRD when my request is larger than a quick fix so that complex work is properly planned.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder estimates request complexity based on: affected files, scope, consequences
- [ ] For requests estimated as "medium" or "large" scope, Builder recommends PRD
- [ ] Recommendation includes reasoning: "This affects X files, requires Y migration, touches Z components"
- [ ] User can choose to proceed with ad-hoc anyway (override) or switch to Planner
- [ ] If user chooses Planner, Builder provides a summary to pass to @planner

### US-006: Fast-Path for Trivial Requests

**Description:** As a user, I want trivial requests to skip the analysis overhead so that quick fixes remain quick.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies "trivial" requests: single-file changes, cosmetic fixes, typos, simple additions
- [ ] Trivial requests show abbreviated analysis: "Quick fix: [summary]. Proceeding..."
- [ ] User can still say "wait" or "hold on" to get full analysis
- [ ] Trivial threshold is configurable via `project.json` → `agents.adhocAnalysisThreshold`
- [ ] Default threshold: 1 file, no breaking changes, no migrations

### US-007: Analysis Output Format

**Description:** As a user, I want a clear, scannable format for the analysis so that I can quickly review and approve.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Analysis uses a consistent visual format (box/section style matching other Builder outputs)
- [ ] Sections are clearly labeled: Understanding, Questions (if any), Alternatives (if any), Consequences, Recommendation
- [ ] Each section is collapsible/skippable for trivial requests
- [ ] Final prompt asks for confirmation: "[G] Go ahead | [Q] Questions | [P] Create PRD instead | [C] Cancel"
- [ ] Format matches Builder's existing dashboard/prompt style

### US-008: Update adhoc-workflow Skill

**Description:** As a developer, I need the adhoc-workflow skill updated to include the analysis phase so that Builder loads the correct workflow.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] `skills/adhoc-workflow/SKILL.md` includes new "Phase 0: Analysis" section
- [ ] Analysis phase comes before current "Phase 1: Task Execution"
- [ ] Current phases renumbered (1→2, 2→3, 3→4)
- [ ] Fast-path logic clearly documented
- [ ] Example flow updated to show analysis phase

## Functional Requirements

- FR-1: Builder MUST analyze ad-hoc requests before implementing (unless trivial fast-path)
- FR-2: Analysis MUST include understanding summary, scope estimate, and affected files
- FR-3: Analysis MUST identify ambiguities and ask clarifying questions when present
- FR-4: Analysis MUST identify downstream consequences (breaking changes, migrations, dependencies)
- FR-5: Analysis MUST recommend PRD creation for medium/large scope requests
- FR-6: Builder MUST provide fast-path for trivial requests (configurable threshold)
- FR-7: User MUST be able to override PRD recommendation and proceed with ad-hoc
- FR-8: Analysis output MUST use consistent visual format matching Builder style

## Non-Goals

- No automatic PRD generation (user must switch to Planner manually)
- No blocking of ad-hoc work for large requests (recommendation only, user can override)
- No changes to PRD mode workflow
- No changes to quality checks or ship phase (only adding analysis before implementation)

## Technical Considerations

- **Performance:** Analysis should complete in <5 seconds for most requests
- **Context:** Analysis uses existing project context loading (project.json, CONVENTIONS.md)
- **File scanning:** May need to scan source files to identify affected components
- **Scope estimation:** Heuristics based on file count, change type, consequence severity

### Scope Estimation Heuristics

| Scope | Criteria |
|-------|----------|
| Trivial | 1 file, cosmetic/typo, no tests affected, no dependencies |
| Small | 1-3 files, contained change, minimal test updates |
| Medium | 4-10 files, OR has breaking changes, OR requires migration |
| Large | 10+ files, OR multiple breaking changes, OR significant architecture impact |

### Configuration

```json
{
  "agents": {
    "adhocAnalysisThreshold": "small",  // "trivial" | "small" | "medium" | "none"
    "adhocPrdRecommendation": true      // Show PRD recommendations
  }
}
```

- `adhocAnalysisThreshold`: Scope at which full analysis is shown (requests below this get fast-path)
- `adhocPrdRecommendation`: Whether to recommend PRD for larger requests

## Success Metrics

- Users confirm understanding before implementation (reduced rework)
- Medium/large requests are flagged for PRD consideration
- Trivial requests remain fast (<3 second overhead)
- Downstream consequences are surfaced before implementation (reduced surprises)

## Open Questions

1. Should analysis results be persisted to `builder-state.json` for session continuity?
2. Should there be a "always skip analysis" user preference for power users?
3. How should analysis handle requests that span multiple unrelated changes?

## Credential & Service Access Plan

No external credentials required for this PRD.
