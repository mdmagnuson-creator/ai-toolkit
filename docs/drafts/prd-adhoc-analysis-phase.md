# PRD: Ad-hoc Analysis Phase with Task Specs

## Introduction

Currently, when a user asks Builder to do something in ad-hoc mode (outside of a PRD), Builder immediately starts implementing. This can lead to:

- **Misunderstood requirements** — Builder assumes intent without clarification
- **Missed context** — Changes that conflict with existing patterns or architecture
- **Scope creep** — Small requests that should be larger planned features
- **Downstream consequences** — Breaking changes, migration needs, or dependency impacts not considered
- **No tracking** — Ad-hoc work isn't documented like PRD work

This PRD adds an **Analysis Phase** to ad-hoc mode and introduces **Task Specs** — lightweight, Builder-generated planning documents that provide PRD-like structure without involving Planner.

## Key Concept: Task Specs vs PRDs

| Aspect | Task Spec (Ad-hoc) | PRD (Formal) |
|--------|-------------------|--------------|
| **Created by** | Builder | Planner |
| **Location** | `docs/tasks/` | `docs/prds/` |
| **Scope** | Single request/session | Multi-story features |
| **Lifecycle** | Generated → Executed → Archived | Draft → Ready → In Progress → Complete |
| **Registry** | `docs/task-registry.json` | `docs/prd-registry.json` |
| **Complexity** | Trivial to Medium | Medium to Large |
| **User involvement** | Confirm understanding | Multi-round refinement |

**Strict separation:** Builder NEVER creates or modifies files in `docs/prds/` or `docs/prd-registry.json`. Planner NEVER creates or modifies files in `docs/tasks/` or `docs/task-registry.json`.

## Goals

- Ensure Builder understands the request before implementing
- Surface potential conflicts with existing code/architecture
- Present alternative approaches when relevant
- Identify downstream consequences (breaking changes, migrations, etc.)
- Track ad-hoc work with Task Specs (like lightweight PRDs)
- Recommend formal PRD creation for larger requests
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
- [ ] Builder incorporates answers into Task Spec

### US-003: Alternative Approaches

**Description:** As a user, I want Builder to present alternative approaches when relevant so that I can choose the best solution.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies when multiple valid approaches exist
- [ ] Builder presents 2-3 alternatives with pros/cons
- [ ] Each alternative includes estimated complexity and impact
- [ ] User can select an approach or ask for more detail
- [ ] Selected approach is recorded in Task Spec
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
- [ ] Consequences are recorded in Task Spec
- [ ] Consequences are shown before implementation, not after
- [ ] User can acknowledge consequences or reconsider the request

### US-005: Task Spec Generation

**Description:** As a user, I want Builder to generate a Task Spec document for non-trivial ad-hoc requests so that my work is tracked and documented.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder generates Task Spec for small/medium scope requests (not trivial)
- [ ] Task Spec is saved to `docs/tasks/task-YYYY-MM-DD-brief-name.md`
- [ ] Task Spec includes: summary, scope, approach, acceptance criteria, consequences
- [ ] Task Spec format is simpler than PRD (no user stories, just tasks)
- [ ] Task Spec is shown to user for confirmation before implementation
- [ ] Trivial requests skip Task Spec generation (inline tracking only)

### US-006: Task Registry

**Description:** As a user, I want ad-hoc tasks tracked in a registry so that I can see history and resume interrupted work.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] `docs/task-registry.json` tracks all Task Specs
- [ ] Registry includes: id, title, status, createdAt, completedAt
- [ ] Status values: `in_progress`, `completed`, `abandoned`
- [ ] Builder can resume in-progress Task Specs from previous sessions
- [ ] Registry is separate from `prd-registry.json` (no cross-contamination)

### US-007: Task Spec Todo Tracking

**Description:** As a user, I want Task Spec todos tracked like PRD stories so that I see progress in the right panel.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Task Spec tasks are shown in OpenCode right panel via `todowrite`
- [ ] Tasks use same status model as PRD stories: pending, in_progress, completed
- [ ] `builder-state.json` tracks current Task Spec in `activeTask` field (separate from `activePrd`)
- [ ] Task progress persists across session interruptions
- [ ] Completing all tasks marks Task Spec as complete in registry

### US-008: PRD Recommendation for Large Requests

**Description:** As a user, I want Builder to recommend creating a formal PRD when my request is too large for ad-hoc so that complex work is properly planned.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder estimates request complexity based on: affected files, scope, consequences
- [ ] For requests estimated as "large" scope, Builder recommends formal PRD
- [ ] Recommendation includes reasoning: "This affects X files, requires Y migration, touches Z components"
- [ ] User can choose: proceed with Task Spec anyway (override) OR switch to Planner
- [ ] If user chooses Planner, Builder provides a summary to pass to @planner
- [ ] Builder does NOT create the PRD itself (strict separation)

### US-009: Fast-Path for Trivial Requests

**Description:** As a user, I want trivial requests to skip the analysis overhead so that quick fixes remain quick.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies "trivial" requests: single-file changes, cosmetic fixes, typos, simple additions
- [ ] Trivial requests show abbreviated analysis: "Quick fix: [summary]. Proceeding..."
- [ ] Trivial requests do NOT generate Task Spec (inline tracking only)
- [ ] User can still say "wait" or "hold on" to get full analysis
- [ ] Trivial threshold is configurable via `project.json` → `agents.adhocAnalysisThreshold`
- [ ] Default threshold: 1 file, no breaking changes, no migrations

### US-010: Analysis Output Format

**Description:** As a user, I want a clear, scannable format for the analysis so that I can quickly review and approve.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Analysis uses a consistent visual format (box/section style matching other Builder outputs)
- [ ] Sections are clearly labeled: Understanding, Questions (if any), Alternatives (if any), Consequences, Task Spec Preview
- [ ] Final prompt asks for confirmation: "[G] Go ahead | [Q] Questions | [P] Create PRD instead | [C] Cancel"
- [ ] Format matches Builder's existing dashboard/prompt style

### US-011: Update adhoc-workflow Skill

**Description:** As a developer, I need the adhoc-workflow skill updated to include the analysis phase and Task Spec workflow.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] `skills/adhoc-workflow/SKILL.md` includes new "Phase 0: Analysis & Task Spec" section
- [ ] Task Spec generation logic is documented
- [ ] Fast-path logic clearly documented
- [ ] Task registry management documented
- [ ] Example flow updated to show full analysis → Task Spec → execution cycle

### US-012: Task Spec Completion and Archival

**Description:** As a user, I want completed Task Specs archived so that I have a record of ad-hoc work.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] When all tasks in a Task Spec are complete, mark as `completed` in registry
- [ ] Completed Task Specs remain in `docs/tasks/` (not moved)
- [ ] Task Spec file is updated with completion timestamp and summary
- [ ] `builder-state.json` → `activeTask` is cleared
- [ ] User sees completion summary matching PRD completion format

## Functional Requirements

- FR-1: Builder MUST analyze ad-hoc requests before implementing (unless trivial fast-path)
- FR-2: Analysis MUST include understanding summary, scope estimate, and affected files
- FR-3: Analysis MUST identify ambiguities and ask clarifying questions when present
- FR-4: Analysis MUST identify downstream consequences (breaking changes, migrations, dependencies)
- FR-5: Builder MUST generate Task Spec for small/medium scope requests
- FR-6: Task Specs MUST be stored in `docs/tasks/` (NEVER in `docs/prds/`)
- FR-7: Task Specs MUST be tracked in `docs/task-registry.json` (NEVER in `docs/prd-registry.json`)
- FR-8: Builder MUST recommend formal PRD for large scope requests
- FR-9: Builder MUST provide fast-path for trivial requests (configurable threshold)
- FR-10: User MUST be able to override recommendations and proceed with Task Spec
- FR-11: Task Spec todos MUST be tracked in right panel and `builder-state.json`

## Non-Goals

- No Builder involvement in formal PRD creation (Planner only)
- No Planner involvement in Task Spec creation (Builder only)
- No blocking of ad-hoc work for large requests (recommendation only, user can override)
- No changes to PRD mode workflow
- No changes to quality checks or ship phase (only adding analysis before implementation)

## Technical Considerations

### File Structure

```
docs/
├── prds/                    # Planner's domain (formal PRDs)
│   └── prd-feature-name.md
├── prd-registry.json        # Planner's registry
├── tasks/                   # Builder's domain (Task Specs)
│   └── task-2026-03-01-add-spinner.md
├── task-registry.json       # Builder's registry
└── builder-state.json       # Tracks activeTask OR activePrd (not both)
```

### Task Spec Format

```markdown
# Task: Add Loading Spinner to Submit Button

**Created:** 2026-03-01T10:30:00Z
**Status:** in_progress
**Scope:** small

## Summary

Add a loading spinner to the submit button that shows during form submission.

## Approach

Use existing Spinner component, disable button during submission.

## Tasks

- [ ] Add loading state to SubmitButton component
- [ ] Show Spinner component when loading
- [ ] Disable button during submission
- [ ] Add unit tests

## Consequences

- None identified

## Completion

**Completed:** [timestamp when done]
**Summary:** [brief completion notes]
```

### Task Registry Format

```json
{
  "schemaVersion": 1,
  "tasks": [
    {
      "id": "task-2026-03-01-add-spinner",
      "title": "Add Loading Spinner to Submit Button",
      "status": "completed",
      "scope": "small",
      "createdAt": "2026-03-01T10:30:00Z",
      "completedAt": "2026-03-01T11:15:00Z"
    }
  ]
}
```

### Builder State Extension

```json
{
  "activePrd": null,
  "activeTask": {
    "id": "task-2026-03-01-add-spinner",
    "currentItem": 1,
    "totalItems": 4
  },
  "uiTodos": { ... }
}
```

**Rule:** `activePrd` and `activeTask` are mutually exclusive. Only one can be non-null at a time.

### Scope Estimation Heuristics

| Scope | Criteria | Task Spec? |
|-------|----------|------------|
| Trivial | 1 file, cosmetic/typo, no tests affected, no dependencies | No (inline) |
| Small | 1-3 files, contained change, minimal test updates | Yes |
| Medium | 4-10 files, OR has breaking changes, OR requires migration | Yes |
| Large | 10+ files, OR multiple breaking changes, OR significant architecture impact | Recommend PRD |

### Configuration

```json
{
  "agents": {
    "adhocAnalysisThreshold": "small",
    "adhocPrdRecommendation": true,
    "adhocTaskSpecs": true
  }
}
```

- `adhocAnalysisThreshold`: Scope at which full analysis is shown (requests below this get fast-path)
- `adhocPrdRecommendation`: Whether to recommend formal PRD for large requests
- `adhocTaskSpecs`: Whether to generate Task Specs (disable for legacy behavior)

## Success Metrics

- Users confirm understanding before implementation (reduced rework)
- Ad-hoc work is tracked and resumable (Task Specs)
- Large requests are flagged for PRD consideration
- Trivial requests remain fast (<3 second overhead)
- Downstream consequences are surfaced before implementation (reduced surprises)
- Clear separation between Builder (tasks) and Planner (PRDs) domains

## Open Questions

1. Should trivial requests be logged anywhere (even without Task Spec)?
2. Should there be a way to "promote" a Task Spec to a formal PRD mid-execution?
3. How long should completed Task Specs be retained before cleanup?

## Credential & Service Access Plan

No external credentials required for this PRD.
