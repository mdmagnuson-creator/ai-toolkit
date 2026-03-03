# PRD: Related Projects Configuration

## Introduction

Add the ability to define relationships between projects in `project.json` so agents can automatically identify companion projects (e.g., app ↔ marketing website, api ↔ admin dashboard, toolkit ↔ documentation website). This solves the problem of agents needing to guess which "website" project to use when there are multiple, and enables cross-project workflows like syncing documentation updates.

## Goals

- Define explicit relationships between projects in `project.json`
- Enable agents to resolve "the website for this project" unambiguously
- Support common relationship types: documentation site, marketing site, admin dashboard, API backend, mobile app
- Allow bidirectional relationships (project A knows about B, and B knows about A)
- Enable cross-project workflows (e.g., toolkit changes → queue website sync)

## User Stories

### US-001: Add relatedProjects schema to project.json

**Description:** As a developer, I want to define related projects in my project.json so that agents know which companion projects exist.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] New `relatedProjects` section added to project.json schema
- [ ] Schema includes: array of related project objects
- [ ] Each related project has: `projectId` (string, references projects.json id)
- [ ] Each related project has: `relationship` (enum: see list below)
- [ ] Each related project has: optional `description` (string)
- [ ] Schema validation works for all field types
- [ ] JSON schema file updated: `schemas/project.schema.json`

**Relationship types to support:**

| Relationship | Description | Example |
|--------------|-------------|---------|
| `documentation-site` | Documentation/marketing website for this project | yo-go → opencode-toolkit-website |
| `marketing-site` | Public marketing/landing page site | saas-app → saas-marketing |
| `admin-dashboard` | Admin/internal dashboard for this app | api-service → admin-ui |
| `api-backend` | Backend API that this frontend consumes | mobile-app → api-service |
| `mobile-app` | Mobile app companion to this web app | web-app → mobile-app |
| `shared-library` | Shared code library used by this project | app → shared-utils |
| `monorepo-sibling` | Another app in the same monorepo | app-a → app-b |
| `test-harness` | Test/QA project for this system | production-api → api-tests |

---

### US-002: Add relatedProjects to projects.json registry

**Description:** As a system, I need related projects stored in the global registry so cross-project lookups work without reading each project.json.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] `projects.json` schema updated to include `relatedProjects` array per project
- [ ] Format mirrors project.json schema (projectId, relationship, description)
- [ ] Bootstrap/onboarding copies relatedProjects from project.json to registry
- [ ] Registry lookup is O(1) by project ID

---

### US-003: Update project-bootstrap skill to prompt for related projects

**Description:** As a user bootstrapping a project, I want to be asked about related projects so the relationships are configured from the start.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] New step in project-bootstrap skill asks about related projects
- [ ] Shows list of existing projects from registry
- [ ] User can select multiple and assign relationship types
- [ ] Selected relationships saved to both project.json and projects.json
- [ ] Skip option for projects with no related projects

**Example prompt:**

```
═══════════════════════════════════════════════════════════════════════
                       RELATED PROJECTS
═══════════════════════════════════════════════════════════════════════

Does this project have related companion projects?

Existing projects:
  1. helm-ade-website (Next.js marketing site)
  2. opencode-toolkit-website (Next.js documentation site)
  3. helm-api (Go API backend)

Select related projects (comma-separated numbers, or 'skip'):
> 2

Selected: opencode-toolkit-website

What is the relationship?
  A. documentation-site (docs website for this project)
  B. marketing-site (public marketing website)
  C. admin-dashboard (admin UI for this backend)
  D. api-backend (API this frontend consumes)
  E. other (specify)

> A

Added: opencode-toolkit-website as documentation-site

Add more related projects? (y/n)
> n
═══════════════════════════════════════════════════════════════════════
```

---

### US-004: Create helper function to resolve related projects

**Description:** As an agent, I need a reliable way to find related projects by relationship type so I don't have to guess based on project names.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] Helper function/pattern documented for agents
- [ ] Input: current project ID, relationship type
- [ ] Output: related project path (or null if not found)
- [ ] Falls back to projects.json if project.json not available
- [ ] Returns first match if multiple (with warning)

**Example usage in agent:**

```bash
# Find documentation site for current project
DOC_SITE=$(jq -r '.relatedProjects[] | select(.relationship == "documentation-site") | .projectId' docs/project.json)
if [ -n "$DOC_SITE" ]; then
  DOC_PATH=$(jq -r --arg id "$DOC_SITE" '.projects[] | select(.id == $id) | .path' ~/.config/opencode/projects.json)
  echo "Documentation site: $DOC_PATH"
fi
```

---

### US-005: Update toolkit.md to use relatedProjects for website sync

**Description:** As the toolkit agent, I need to use relatedProjects to find the correct documentation website instead of guessing based on project name.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] Toolkit post-change workflow updated to read relatedProjects
- [ ] Looks for `relationship: "documentation-site"` in yo-go's project.json
- [ ] Falls back to current behavior (search for "website" in id) if not configured
- [ ] Clear error message if no documentation site configured

**Updated Step 3a:**

```bash
# Get documentation site for this toolkit
DOC_SITE_ID=$(jq -r '.relatedProjects[] | select(.relationship == "documentation-site") | .projectId' docs/project.json 2>/dev/null)

if [ -n "$DOC_SITE_ID" ] && [ "$DOC_SITE_ID" != "null" ]; then
  WEBSITE_PATH=$(jq -r --arg id "$DOC_SITE_ID" '.projects[] | select(.id == $id) | .path' ~/.config/opencode/projects.json)
else
  # Fallback: search for website in project id (legacy behavior)
  WEBSITE_PATH=$(jq -r '.projects[] | select(.id | contains("toolkit-website")) | .path' ~/.config/opencode/projects.json | head -1)
fi

if [ -z "$WEBSITE_PATH" ] || [ "$WEBSITE_PATH" == "null" ]; then
  echo "⚠️ No documentation-site configured in relatedProjects. Skipping website sync."
  exit 0
fi
```

---

### US-006: Configure yo-go relatedProjects

**Description:** As the first use case, configure the yo-go toolkit to have opencode-toolkit-website as its documentation-site.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] yo-go/docs/project.json updated with relatedProjects section
- [ ] opencode-toolkit-website listed as documentation-site
- [ ] projects.json registry updated to match
- [ ] Toolkit post-change workflow correctly finds the website

**Example addition to yo-go project.json:**

```json
{
  "name": "yo-go",
  "relatedProjects": [
    {
      "projectId": "opencode-toolkit-website",
      "relationship": "documentation-site",
      "description": "Public documentation website for the AI toolkit"
    }
  ]
}
```

---

### US-007: Add bidirectional relationship awareness

**Description:** As a user, I want related projects to be aware of each other bidirectionally so either project can find the other.

**Documentation:** No

**Tools:** No

**Considerations:** none

**Credentials:** none

**Acceptance Criteria:**

- [ ] When adding a relationship, optionally add inverse relationship to the other project
- [ ] Inverse relationship types defined (e.g., documentation-site ↔ documented-project)
- [ ] Bootstrap prompts for bidirectional setup
- [ ] Helper function works from either direction

**Inverse relationship mapping:**

| Relationship | Inverse |
|--------------|---------|
| `documentation-site` | `documented-project` |
| `marketing-site` | `marketed-product` |
| `admin-dashboard` | `managed-service` |
| `api-backend` | `frontend-client` |
| `mobile-app` | `web-counterpart` |
| `shared-library` | `dependent-project` |
| `monorepo-sibling` | `monorepo-sibling` (symmetric) |
| `test-harness` | `tested-system` |

---

## Functional Requirements

- FR-1: Add `relatedProjects` array to project.json schema with projectId, relationship, description
- FR-2: Add `relatedProjects` array to projects.json registry schema
- FR-3: Project-bootstrap skill prompts for related projects during setup
- FR-4: Helper pattern documented for agents to resolve related projects by relationship type
- FR-5: Toolkit agent uses relatedProjects instead of name-based guessing for website sync
- FR-6: Support bidirectional relationships with inverse types
- FR-7: Fallback to legacy behavior (name-based search) when relatedProjects not configured

## Non-Goals

- No automatic relationship detection (requires explicit configuration)
- No cross-project code sharing or imports (just metadata)
- No multi-project workspaces or monorepo tooling
- No relationship-based build ordering

## Technical Considerations

- **Schema location:** `schemas/project.schema.json`
- **Registry location:** `~/.config/opencode/projects.json`
- **Bootstrap skill:** `skills/project-bootstrap/SKILL.md`
- **Toolkit agent:** `agents/toolkit.md`
- Relationships are stored as project IDs, not paths (paths resolved via registry)
- Changes to project.json should sync to projects.json on next bootstrap/refresh

## Success Metrics

- Toolkit correctly syncs to opencode-toolkit-website without name guessing
- New projects can configure related projects during bootstrap
- Agents can find related projects in O(1) lookup time
- Zero ambiguity when multiple "website" projects exist

## Open Questions

- Should we support multiple projects of the same relationship type? (e.g., two documentation sites for different audiences)
- Should relationships be versioned or timestamped?
- Should we add a CLI command to manage relationships post-bootstrap?

## Credential & Service Access Plan

No external credentials required for this PRD.
