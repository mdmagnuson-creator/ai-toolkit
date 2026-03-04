# Global Agent Guardrails

## Protected System Resources

> ⛔ **CRITICAL: Never access protected macOS resources**
>
> Do NOT access anything that would trigger a macOS permission prompt.
> These operations block automated workflows and require user intervention.

**Never access:**
- `~/Desktop/`, `~/Documents/`, `~/Downloads/`
- `~/Library/`
- Contacts, Calendar, Reminders, Photos, Mail
- Screen recording or accessibility APIs
- Other applications via AppleScript/osascript
- Clipboard (`pbcopy`/`pbpaste`) in automated scripts
- Keychain or credential stores
- System notifications

**Safe alternatives:**
- Store files in the project directory or `.tmp/`
- Use environment variables for secrets
- Write output to files instead of clipboard
- Log to files instead of system notifications

If a command requires system access, stop and tell the user to run it manually.

## Temporary Files

> ⛔ **CRITICAL: Never use system temp directories**
>
> Do NOT write to `/tmp/`, `/var/folders/`, or any system temporary directory.

Use project-local `.tmp/` for temporary artifacts.

Rules:
- Use `.tmp/` in the project root for all temporary files
- Create subdirectories as needed (`.tmp/screenshots/`, `.tmp/logs/`)
- Ensure `.tmp/` is listed in `.gitignore`
- Clean up temp files when no longer needed

## Global Coding Behavior

- Do not over-comment; add comments only for non-obvious logic
- Exported functions/classes must include language-standard doc comments
- When using external libraries, use a documentation lookup tool if available
- If a docs tool is not available, use primary upstream docs and project source references as fallback

## AWS CLI Sessions

If a command fails due to expired AWS credentials or requires `aws sso login`, stop and tell the user to refresh credentials manually.

## Project Memory

Use `docs/memory/` to capture non-obvious lessons:
- gotchas/footguns
- hidden config constraints
- workarounds and rationale

Do not store obvious code facts, temporary notes, or duplicate docs.

## Cross-Service Blast Radius Checks

Before changing APIs, message schemas, shared contracts, or inter-service integrations:
- Use semantic code search tooling if configured
- If unavailable, fall back to repository-native search (`grep`, project indexes, dependency maps)
- Do not assume blast radius; verify it

## Visual Verification for UI Changes

Any user-visible UI change must be visually verified before committing.

Minimum checks:
1. Capture a screenshot (qa-explorer or screenshot workflow)
2. Review result directly
3. Check key states (hover/active/disabled/loading/error/empty)
4. Check edge cases (long text, dense data, small viewports)

## Git Auto-Commit Enforcement

> ⛔ **CRITICAL: Check `git.autoCommit` setting before ANY commit operation**
>
> **Trigger:** Before running `git commit`, `git add && git commit`, or any commit delegation.
>
> **Check:** Read `project.json` → `git.autoCommit`
>
> **Failure behavior:** Running `git commit` when autoCommit is `manual` or `false` violates a harsh project constraint.

### Auto-Commit Modes

| Value | Behavior |
|-------|----------|
| `onStoryComplete` | (default) Commit after each completed PRD story or ad-hoc task |
| `onFileChange` | Commit after each file modification — more granular history |
| `manual` | Stage changes but do NOT run `git commit` (see protocol below) |
| `true` | (legacy) Same as `onStoryComplete` |
| `false` | (legacy) Same as `manual` |

### When `manual` or `false`:

1. Stage changes: `git add <files>`
2. Report what would be committed:
   ```
   📋 READY TO COMMIT (manual commit required)
   
   Staged files:
     - src/components/Button.tsx
     - src/styles/button.css
   
   Suggested commit message:
     feat: add Button component with hover states
   
   Run: git commit -m "feat: add Button component with hover states"
   ```
3. **Do NOT run `git commit`** — wait for user to commit manually

## Git Workflow Enforcement

> ⛔ **CRITICAL: Validate branch targets before ANY git push or PR operation**
>
> **Trigger:** Before running `git push`, `gh pr create`, or any auto-merge.
>
> **Check:** Read `project.json` → `git.agentWorkflow`
>
> **Failure behavior:** BLOCK the operation and show the appropriate error. No fallbacks, no warnings — hard stop.

### Validation Protocol

**Before `git push [branch]`:**
1. Read `project.json` → `git.agentWorkflow`
2. If `git.agentWorkflow` not defined: BLOCK (Missing Config Error)
3. If target branch in `requiresHumanApproval`: BLOCK (Protected Branch Error)
4. If target branch ≠ `pushTo`: BLOCK (Wrong Target Error)
5. Proceed only if all checks pass

**Before `gh pr create --base [branch]`:**
1. Read `project.json` → `git.agentWorkflow`
2. If `git.agentWorkflow` not defined: BLOCK (Missing Config Error)
3. If `--base` branch ≠ `createPrTo`: BLOCK (Wrong Target Error)
4. If `--base` branch in `requiresHumanApproval`: ALLOW PR creation, but do NOT auto-merge (see below)
5. Proceed only if all checks pass

**Before auto-merge (`gh pr merge`):**
1. If target branch in `requiresHumanApproval`: BLOCK (human must merge)
2. Report: "PR created. Human approval required to merge."

> ⚠️ **Important distinction:**
> - **Direct push** to protected branch → BLOCKED
> - **Create PR** to protected branch → ALLOWED
> - **Auto-merge PR** to protected branch → BLOCKED (human must approve and merge)

### Default Cascade

When fields are not explicitly set:
- `pushTo` defaults to `workBranch`
- `workBranch` defaults to `defaultBranch`
- `createPrTo` defaults to `defaultBranch`

### Error Formats

**Missing Config Error:**
```
⛔ GIT WORKFLOW NOT CONFIGURED

This project requires git workflow configuration before I can perform git operations.

Please describe your git workflow:
1. What branch do you work on? (e.g., main, staging, develop)
2. Where should I push changes? (e.g., same branch, staging branch)
3. Where should PRs be created to? (e.g., main)
4. Which branches require human approval? (e.g., main, production)

I'll generate the configuration for your review.
```

**Protected Branch Error:**
```
⛔ PROTECTED BRANCH — HUMAN APPROVAL REQUIRED

Attempted: git push origin main
Branch 'main' is in requiresHumanApproval — all agent operations are blocked.

This project's workflow:
  - Work on: staging
  - Push to: staging
  - Create PRs to: main (requires human approval)

To proceed:
  1. Push to staging: git push origin staging
  2. Create PR: gh pr create --base main
  3. Get human approval and merge

Source: docs/project.json → git.agentWorkflow.requiresHumanApproval
```

**Wrong Target Error:**
```
⛔ GIT WORKFLOW VIOLATION

Attempted: git push origin develop
Configured target: staging (from git.agentWorkflow.pushTo)

This project's workflow:
  - Work on: staging
  - Push to: staging
  - Create PRs to: main

To proceed correctly:
  git push origin staging

Source: docs/project.json → git.agentWorkflow
```

## Test Failure Output Policy

> ⛔ **IMPORTANT: Never truncate test failure output**
>
> When tests fail, show the **complete failure output** — every failing test, every error message, every stack trace.
> Do not summarize, truncate, or omit failure details.
>
> - Successful test runs: summarize (e.g., "42 tests passed")
> - Failed test runs: show full output, no truncation
>
> Truncating test failures defeats the purpose of running tests.

## Requesting Toolkit Updates

If you discover a needed toolkit change (agent bug, missing capability, etc.), **do not modify toolkit files directly**. Instead:

1. Write a request file to `~/.config/opencode/pending-updates/`:
   ```
   ~/.config/opencode/pending-updates/YYYY-MM-DD-{agent-name}-description.md
   ```

2. Use this format:
   ```markdown
   ---
   requestedBy: {agent-name}
   date: YYYY-MM-DD
   priority: normal
   ---
   
   # Update Request: [Brief Title]
   
   ## What to change
   
   [Describe the change in detail]
   
   ## Files affected
   
   - `agents/{agent-name}.md` — [change description]
   
   ## Why
   
   [Why this change is needed]
   ```

3. Tell the user: "I've queued a toolkit update request. Next time you run @toolkit, it will offer to apply it."

## Instruction Ownership

This root file defines universal guardrails only.

- Specialist implementation rules belong to implementation agents/templates
- Specialist review criteria belong to critic agents
- Project-specific behavior belongs in each project's `docs/CONVENTIONS.md`
