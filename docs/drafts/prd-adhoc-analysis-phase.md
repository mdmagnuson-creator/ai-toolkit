# PRD: Ad-hoc Analysis Phase with Task Specs

## Introduction

Currently, when a user asks Builder to do something in ad-hoc mode (outside of a PRD), Builder immediately starts implementing. This can lead to:

- **Misunderstood requirements** — Builder assumes intent without clarification
- **Missed context** — Changes that conflict with existing patterns or architecture
- **Scope creep** — Small requests that should be larger planned features
- **Downstream consequences** — Breaking changes, migration needs, or dependency impacts not considered
- **No tracking** — Ad-hoc work isn't documented like PRD work

This PRD adds an **Analysis Phase** to ad-hoc mode and introduces **Task Specs** — Builder-generated planning documents that mirror the PRD lifecycle while maintaining strict separation from Planner's domain.

## Key Concept: Task Specs vs PRDs

Task Specs follow the **same lifecycle as PRDs** but in a parallel folder structure:

| Aspect | Task Spec (Builder) | PRD (Planner) |
|--------|---------------------|---------------|
| **Created by** | Builder | Planner |
| **Drafts location** | `docs/tasks/drafts/` | `docs/drafts/` |
| **Ready location** | `docs/tasks/` | `docs/prds/` |
| **Completed location** | `docs/tasks/completed/` | `docs/completed/` |
| **Abandoned location** | `docs/tasks/abandoned/` | `docs/abandoned/` |
| **Registry** | `docs/task-registry.json` | `docs/prd-registry.json` |
| **Scope** | Any size (single request) | Planned features |
| **User involvement** | Confirm understanding | Multi-round refinement |

**Strict separation:** 
- Builder NEVER creates or modifies files in `docs/prds/`, `docs/drafts/`, `docs/completed/`, or `docs/prd-registry.json`
- Planner NEVER creates or modifies files in `docs/tasks/` or `docs/task-registry.json`

## Goals

- Analyze ALL ad-hoc requests consistently (no fast-path exceptions)
- Generate Task Specs for every ad-hoc request
- Track Task Specs with same lifecycle as PRDs (draft → ready → in_progress → completed)
- Surface potential conflicts with existing code/architecture
- Present alternative approaches when relevant
- Identify downstream consequences (breaking changes, migrations, etc.)
- Enable Task Spec promotion to formal PRD when scope grows
- Recommend formal PRD creation for large requests upfront

## User Stories

### US-001: Consistent Request Analysis

**Description:** As a user, I want Builder to analyze every ad-hoc request the same way so that I have consistent visibility regardless of request size.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Every ad-hoc request triggers full analysis (no trivial fast-path)
- [ ] Builder reads `project.json` and relevant source files to understand context
- [ ] Builder outputs an "Understanding" section summarizing the request
- [ ] Understanding includes: affected files/components, scope estimate, approach
- [ ] User confirms understanding before Task Spec is generated
- [ ] Analysis time is optimized but not skipped

### US-002: Clarifying Questions

**Description:** As a user, I want Builder to ask clarifying questions when my request is ambiguous so that it doesn't make wrong assumptions.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder identifies ambiguities in the request (scope, approach, behavior)
- [ ] Builder asks 1-3 focused clarifying questions with lettered options
- [ ] User can respond with letters (e.g., "1A, 2B") for quick answers
- [ ] Builder incorporates answers into Task Spec
- [ ] If no ambiguities exist, Builder states "No clarifying questions needed"

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
- [ ] If only one obvious approach exists, Builder states "Recommended approach: X"

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
- [ ] Consequences are shown before implementation
- [ ] If no consequences identified, Builder states "No downstream consequences identified"

### US-005: Task Spec Generation

**Description:** As a user, I want Builder to generate a Task Spec document for every ad-hoc request so that my work is tracked like PRD work.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder generates Task Spec for every ad-hoc request (no exceptions)
- [ ] Task Spec is initially saved to `docs/tasks/drafts/task-YYYY-MM-DD-brief-name.md`
- [ ] Task Spec includes: summary, scope, approach, tasks (with acceptance criteria), consequences
- [ ] Task Spec format mirrors PRD structure (tasks instead of user stories)
- [ ] Task Spec is shown to user for confirmation
- [ ] On confirmation, Task Spec moves to `docs/tasks/task-YYYY-MM-DD-brief-name.md` (ready)

### US-006: Task Registry with PRD-like Lifecycle

**Description:** As a user, I want Task Specs tracked with the same lifecycle as PRDs so that the experience is consistent.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] `docs/task-registry.json` tracks all Task Specs
- [ ] Registry uses same status values as PRD registry: `draft`, `ready`, `in_progress`, `completed`, `abandoned`
- [ ] Registry includes: id, title, status, scope, createdAt, updatedAt, completedAt
- [ ] Builder can resume `in_progress` Task Specs from previous sessions
- [ ] Registry structure mirrors `prd-registry.json` format

### US-007: Task Spec Todo Tracking

**Description:** As a user, I want Task Spec todos tracked like PRD stories so that I see progress in the right panel.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Task Spec tasks are shown in OpenCode right panel via `todowrite`
- [ ] Tasks use same status model as PRD stories: pending, in_progress, completed
- [ ] `builder-state.json` tracks current Task Spec in `activeTask` field
- [ ] Task progress persists across session interruptions
- [ ] Completing all tasks triggers completion flow (same as PRD)

### US-008: Task Spec Promotion to PRD

**Description:** As a user, I want to promote a Task Spec to a formal PRD when scope grows beyond ad-hoc so that Planner can take over planning.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] User can request promotion at any time: "promote to PRD" or [P] option
- [ ] Builder creates handoff document in `docs/tasks/promotions/`
- [ ] Handoff document includes: original request, analysis, work completed so far, remaining scope
- [ ] Handoff document is named `promote-task-YYYY-MM-DD-name-to-prd.md`
- [ ] Builder updates Task Spec status to `promoted` in registry
- [ ] Builder notifies user: "Promotion request created. Run @planner to continue."
- [ ] Planner can read promotion documents and create formal PRD from them
- [ ] Builder does NOT create the PRD itself (strict separation)

### US-009: PRD Recommendation for Large Requests

**Description:** As a user, I want Builder to recommend creating a formal PRD when my request is large so that complex work is properly planned from the start.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Builder estimates request complexity during analysis
- [ ] For requests estimated as "large" scope (10+ files, multiple breaking changes, significant architecture), Builder recommends formal PRD
- [ ] Recommendation includes reasoning: "This affects X files, requires Y migration, touches Z components"
- [ ] User can choose: proceed with Task Spec anyway (override) OR switch to Planner
- [ ] If user chooses Planner, Builder provides summary for @planner
- [ ] Recommendation is shown during analysis, not after Task Spec is created

### US-010: Analysis Output Format

**Description:** As a user, I want a clear, scannable format for the analysis so that I can quickly review and approve.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Analysis uses consistent visual format matching Builder's existing style
- [ ] Sections: Understanding, Scope, Questions (if any), Alternatives, Consequences, Task Preview
- [ ] Final prompt: "[G] Go ahead | [E] Edit/Clarify | [P] Create formal PRD instead | [C] Cancel"
- [ ] Task Preview shows the tasks that will be in the Task Spec
- [ ] Format is identical regardless of request size

### US-011: Task Spec Completion and Archival

**Description:** As a user, I want completed Task Specs archived like completed PRDs so that I have a consistent record.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] When all tasks complete, Task Spec moves to `docs/tasks/completed/`
- [ ] Registry status updates to `completed` with `completedAt` timestamp
- [ ] Task Spec file is updated with completion timestamp and summary
- [ ] `builder-state.json` → `activeTask` is cleared
- [ ] User sees completion summary matching PRD completion format

### US-012: Task Spec Abandonment

**Description:** As a user, I want to abandon a Task Spec when it's no longer needed so that it's properly tracked.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] User can abandon Task Spec: "abandon" or [A] option during execution
- [ ] Task Spec moves to `docs/tasks/abandoned/`
- [ ] Registry status updates to `abandoned` with reason
- [ ] `builder-state.json` → `activeTask` is cleared
- [ ] Partial work is preserved in the abandoned Task Spec

### US-013: Update adhoc-workflow Skill

**Description:** As a developer, I need the adhoc-workflow skill updated to include the analysis phase and full Task Spec lifecycle.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] `skills/adhoc-workflow/SKILL.md` includes new "Phase 0: Analysis & Task Spec" section
- [ ] Task Spec generation and lifecycle fully documented
- [ ] Promotion flow documented
- [ ] Task registry management documented
- [ ] Example flow updated to show full analysis → Task Spec → execution → completion cycle

### US-014: Planner Promotion Pickup

**Description:** As a developer, I need Planner to recognize and process Task Spec promotion requests.

**Documentation:** No

**Tools:** No

**Acceptance Criteria:**

- [ ] Planner checks for promotion documents in `docs/tasks/promotions/` on startup
- [ ] Planner shows promotion requests in dashboard: "1 Task Spec promotion pending"
- [ ] User can select promotion to review and convert to formal PRD
- [ ] Planner pre-fills PRD draft with information from promotion document
- [ ] After PRD is created, promotion document is deleted
- [ ] Planner adds reference in PRD: "Promoted from Task Spec: task-YYYY-MM-DD-name"

## Functional Requirements

- FR-1: Builder MUST analyze every ad-hoc request before implementing (no exceptions)
- FR-2: Builder MUST generate a Task Spec for every ad-hoc request
- FR-3: Task Specs MUST follow same lifecycle as PRDs (draft → ready → in_progress → completed/abandoned)
- FR-4: Task Specs MUST use parallel folder structure (`docs/tasks/` hierarchy)
- FR-5: Task Specs MUST be tracked in `docs/task-registry.json` (NEVER in `docs/prd-registry.json`)
- FR-6: Builder MUST support Task Spec promotion to PRD via handoff document
- FR-7: Planner MUST recognize and process promotion documents
- FR-8: Builder MUST recommend formal PRD for large scope requests
- FR-9: User MUST be able to override PRD recommendation and proceed with Task Spec
- FR-10: Task Spec todos MUST be tracked in right panel and `builder-state.json`
- FR-11: Completed Task Specs MUST move to `docs/tasks/completed/`
- FR-12: Abandoned Task Specs MUST move to `docs/tasks/abandoned/`

## Non-Goals

- No Builder involvement in formal PRD creation (Planner only)
- No Planner involvement in Task Spec creation (Builder only)
- No automatic promotion (user must explicitly request)
- No changes to PRD mode workflow (PRD execution unchanged)
- No changes to quality checks or ship phase

## Technical Considerations

### File Structure

```
docs/
├── drafts/                      # Planner's PRD drafts
├── prds/                        # Planner's ready PRDs
├── completed/                   # Planner's completed PRDs
├── abandoned/                   # Planner's abandoned PRDs
├── prd-registry.json            # Planner's registry
│
├── tasks/                       # Builder's domain
│   ├── drafts/                  # Task Spec drafts (during analysis)
│   │   └── task-2026-03-01-add-spinner.md
│   ├── completed/               # Completed Task Specs
│   │   └── task-2026-02-28-fix-footer.md
│   ├── abandoned/               # Abandoned Task Specs
│   │   └── task-2026-02-27-refactor-auth.md
│   ├── promotions/              # Promotion handoff documents
│   │   └── promote-task-2026-03-01-name-to-prd.md
│   └── task-2026-03-01-add-spinner.md  # Ready/in-progress Task Specs
│
├── task-registry.json           # Builder's registry
└── builder-state.json           # Tracks activeTask OR activePrd
```

### Task Spec Format

```markdown
---
id: task-2026-03-01-add-spinner
title: Add Loading Spinner to Submit Button
status: in_progress
scope: small
createdAt: 2026-03-01T10:30:00Z
---

# Task: Add Loading Spinner to Submit Button

## Summary

Add a loading spinner to the submit button that shows during form submission.

## Analysis

**Scope:** Small (2 files, no breaking changes)

**Approach:** Use existing Spinner component, disable button during submission.

**Alternatives Considered:**
- CSS-only animation (simpler but less consistent with design system)
- Full-page loading overlay (overkill for single button)

**Consequences:**
- None identified

## Tasks

### T-001: Add loading state to SubmitButton component

**Acceptance Criteria:**
- [ ] Add `isLoading` prop to SubmitButton
- [ ] Pass loading state from parent form
- [ ] Typecheck passes

### T-002: Show Spinner when loading

**Acceptance Criteria:**
- [ ] Import Spinner component
- [ ] Render Spinner when `isLoading` is true
- [ ] Hide button text while loading
- [ ] Verify in browser

### T-003: Disable button during submission

**Acceptance Criteria:**
- [ ] Add `disabled={isLoading}` to button
- [ ] Style disabled state
- [ ] Works in both light and dark mode

### T-004: Add unit tests

**Acceptance Criteria:**
- [ ] Test loading state renders spinner
- [ ] Test disabled during loading
- [ ] Unit tests pass

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
      "status": "in_progress",
      "scope": "small",
      "createdAt": "2026-03-01T10:30:00Z",
      "updatedAt": "2026-03-01T10:45:00Z",
      "completedAt": null,
      "promotedTo": null
    },
    {
      "id": "task-2026-02-28-fix-footer",
      "title": "Fix Footer Alignment",
      "status": "completed",
      "scope": "small",
      "createdAt": "2026-02-28T14:00:00Z",
      "updatedAt": "2026-02-28T14:30:00Z",
      "completedAt": "2026-02-28T14:30:00Z",
      "promotedTo": null
    },
    {
      "id": "task-2026-02-27-refactor-auth",
      "title": "Refactor Auth Flow",
      "status": "promoted",
      "scope": "large",
      "createdAt": "2026-02-27T09:00:00Z",
      "updatedAt": "2026-02-27T10:00:00Z",
      "completedAt": null,
      "promotedTo": "prd-auth-refactor"
    }
  ]
}
```

### Promotion Document Format

```markdown
---
taskId: task-2026-03-01-big-feature
promotedAt: 2026-03-01T15:00:00Z
reason: Scope grew beyond original estimate
---

# Promotion Request: Big Feature Implementation

## Original Request

User asked: "Add user preferences with theme selection"

## Analysis Summary

**Original Scope Estimate:** Small
**Actual Scope:** Large (discovered during implementation)

**Why promotion is needed:**
- Requires database schema changes
- Affects 15+ files
- Needs migration strategy
- Has downstream impacts on 3 other features

## Work Completed

- [x] T-001: Create preferences database table
- [x] T-002: Add theme selection UI

## Remaining Scope

- User preference sync across devices
- Default preference migration for existing users
- Theme application to all components
- Dark mode refinements
- Accessibility considerations

## Recommended PRD Structure

1. Database & API layer (completed work)
2. Theme system implementation
3. Migration strategy
4. Cross-device sync
5. Accessibility audit

## Files Modified So Far

- `src/db/schema/preferences.ts`
- `src/components/settings/ThemeSelector.tsx`
- `src/api/preferences.ts`
```

### Builder State Extension

```json
{
  "activePrd": null,
  "activeTask": {
    "id": "task-2026-03-01-add-spinner",
    "currentItem": "T-002",
    "totalItems": 4,
    "completedItems": 1
  },
  "uiTodos": { ... }
}
```

**Rule:** `activePrd` and `activeTask` are mutually exclusive. Only one can be non-null at a time.

### Scope Estimation Heuristics

| Scope | Criteria |
|-------|----------|
| Small | 1-3 files, contained change, minimal test updates |
| Medium | 4-10 files, OR has breaking changes, OR requires migration |
| Large | 10+ files, OR multiple breaking changes, OR significant architecture impact |

All scopes get full analysis and Task Spec generation. Large scope triggers PRD recommendation.

## Success Metrics

- Every ad-hoc request has a Task Spec (100% coverage)
- Users confirm understanding before implementation (reduced rework)
- Ad-hoc work is tracked and resumable
- Large requests are flagged for PRD consideration
- Downstream consequences are surfaced before implementation
- Clear separation between Builder (tasks) and Planner (PRDs) domains
- Promotion flow enables smooth handoff to Planner when scope grows

## Open Questions

None — all original questions resolved:
1. ✅ All requests handled the same way (no trivial exceptions)
2. ✅ Promotion to PRD defined via handoff documents
3. ✅ Task Spec lifecycle mirrors PRD lifecycle exactly

## Credential & Service Access Plan

No external credentials required for this PRD.
