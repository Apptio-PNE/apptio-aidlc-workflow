---
inclusion: always
---

# Bug Fix Workflow (Kiro)

> Kiro-native bug-fix workflow. Generic by design — every team-specific value (repo names, paths, commands, base branches, project keys) lives in `reference.md`. This file describes the **process**; the reference describes the **environment**.

---

## ⚠️ Activation Guard (READ FIRST)

This workflow **cannot be triggered independently by the user**. It is activated exclusively by the **core-workflow** when it determines that a bug-fix workflow is appropriate for the user's request.

**How activation works:**
- The core-workflow analyzes the user's request during its Workspace Detection and Requirements Analysis stages.
- If the core-workflow identifies the request as a bug fix (e.g., references a Jira bug ticket, describes a defect to resolve, or explicitly asks to fix a bug), it delegates execution to this Bug Fix Workflow.
- Once activated by the core-workflow, this workflow remains in effect for the duration of the bug-fix conversation.

**If the core-workflow has NOT delegated to this workflow:** IGNORE the rest of this file entirely. Do NOT mention the workflow, do NOT display the welcome message, and respond to the user as you normally would.

---

## Project Reference

All team-specific configuration — repository names, project keys, base branches, API contract paths, PR template paths, build commands, GitHub owner, knowledge base sources — lives in:

#[[file:reference.md]]

Throughout this workflow, instructions like "see *Repositories* in `reference.md`" mean: read that exact section heading in the reference file and use the values found there. Do NOT hard-code repository names, branch names, project keys, or file paths in your responses — always pull them from the reference.

**Before starting, fully load the reference file into context**, load the reference file into context using read_file on reference.md. If it is missing, Phase 1 will generate it.

---

## Initial Setup

When the workflow activates:

1. **Display the welcome message** (below) immediately.
2. **Ensure `reference.md` exists** (see *Phase 1* below). This must complete before proceeding to Phase 2.
3. **Load `reference.md`** into context (referenced above).
4. **Prompt for the Jira ticket** if the user did not include one with the trigger keyword.
5. **Initialize state tracking** — keep a running mental record of phase, decisions, created artifacts (tickets, branches, PRs), and pending approvals. Surface progress markers at every phase transition.

> **Note on Kiro modes:** Kiro does NOT require switching between Plan and Code modes. All Kiro tools (file edits, MCP calls, shell commands) are available throughout the session. Do not attempt mode switches.

### Welcome Message

```markdown
🐛 **Bug Fix Workflow Activated**

I'll guide you through a comprehensive bug fixing process across multiple repositories. Here's what we'll accomplish together:

**Phase 1: Workflow Reference Setup**
Ensure the team configuration file exists (auto-generate if needed).

**Phase 2: Information Gathering & Analysis**
Collect bug details from Jira, search for similar issues, review documentation, and gather context through clarifying questions.

**Phase 3: Root Cause Analysis**
Investigate the codebase, trace request flows across repositories, and identify the exact cause of the bug.

**Phase 4: Solution Proposal**
Design a fix, present detailed changes needed, and get your approval before proceeding.

**Phase 5: Scope Identification & Ticket Management**
Determine which repositories need changes, create necessary Jira tickets, and set up feature branches.

**Phase 6: Implementation**
Write the code changes, generate unit tests, and present everything for your review and testing.

**Phase 7: Pull Request Creation**
Generate comprehensive PR descriptions following repository templates and create PRs in GitHub.

**Phase 8: Completion**
Provide a complete summary of all work done, tickets created, branches, and PRs.

Let's get started! Please provide the Jira ticket ID for the bug you'd like to fix.
```

---

## Phase 1: Workflow Reference Setup

**This is always the first step when the workflow activates.**

### If `reference.md` already exists:

Present it to the user:

```markdown
I found an existing `reference.md`. Please review it and confirm:

- **Use as-is** — continue with the existing reference
- **Regenerate** — I'll scan the workspace repos and rebuild it from scratch
```

Wait for the user's response. If they say "use as-is", skip to the Welcome Message. If they say "regenerate", proceed with the generation steps below.

### If `reference.md` does NOT exist (or user chose to regenerate):

**Step 1: Scan repositories**

1. List the top-level folders in the workspace.
2. For each folder, run `git remote get-url origin` to get the remote URL.
3. Extract **repository name** (last path segment, strip `.git`) and **GitHub owner** (second-to-last segment).
4. Skip folders that are not git repos. Skip the current repository (the one containing this Bug Fix Workflow) itself.

**Step 2: Detect base branch** for each repo:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

Fallback: check which of `main`, `master`, `develop` exists as a remote branch.

**Step 3: Detect PR template** — check project root and `.github/` folder:

```bash
ls pull_request_template.md PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null | head -1
```

**Step 4: Detect pre-commit** — check for `.husky/pre-commit`, `lint-staged` in `package.json`, `.pre-commit-config.yaml`, or infer `rush prettier` if `rush.json` + `prettier.config.*` both exist. If nothing found, omit.

**Step 5: Prompt the user** for the remaining values:

```markdown
## Repository scan complete

I found [count] repositories:

| # | Repository | Base Branch | PR Template | Pre-commit |
| --- | --- | --- | --- | --- |
| 1 | [repo] | [branch] | [path or —] | [command or —] |

To finish generating `reference.md`, I need:

1. **Jira project key** for each repository:
   - [repo-1]: ?
   - [repo-2]: ?
   - [repo-3]: ?

2. **Your alias** for branch naming (e.g., "john", "sshankar"):
   - Alias: ?
```

Wait for the user's answers.

**Step 6: Write `reference.md`** with all detected + user-provided values. Include the following header at the top of the file:

```markdown
# Bug Fix Workflow — Project Reference

This file is the **single source of team-specific configuration** for the Bug Fix Workflow. The workflow files (`.kiro/steering/aws-aidlc-rules/bugfix-workflow.md` and `.bob/rules/aws-aidlc-rules/bugfix-workflow.md`) read everything they need from here.

> **Auto-generated file.** This was created by the Bug Fix Workflow during Phase 1 (Workflow Reference Setup). To regenerate it, trigger the workflow and choose "Regenerate" when prompted, or start a new session if the file has been deleted.
```

Hardcode:
- Jira base URL: `https://apptio.atlassian.net`
- Knowledge base: IBM Help Docs (`https://www.ibm.com/docs/en`)

**Step 7: Confirm** to the user that the file was generated, then proceed to the Welcome Message.

---

## Phase 2: Information Gathering & Analysis

### 2.1 Jira Ticket Information Collection

**Objective:** Gather complete bug information from Jira.

**Steps:**
1. Ask the user for the Jira ticket ID if not already provided. Valid project keys are listed under *Project Keys (Quick Reference)* in `reference.md`.
2. Call `mcp_jira_get_issue` with `issueKey` set to the ticket. Optionally request `expand: ["changelog", "renderedFields"]` when context demands it.
3. Pull associated comments with `mcp_jira_get_comments` if the description references prior discussion.
4. Display the captured summary, description, status, priority, assignee, labels, components, attachments, and recent comments back to the user for confirmation.

### 2.2 Search for Similar Issues

**Objective:** Find duplicate or related issues that might provide insights.

**Steps:**
1. Extract key terms from the bug description.
2. Use `mcp_jira_search_issues` with focused JQL queries against the project keys listed in *Project Keys (Quick Reference)*. Generic query shape:
   - `project = <PROJECT-KEY> AND summary ~ "<keyword>" AND status in (Closed, Resolved)`
   - `project = <PROJECT-KEY> AND labels = "<label>" AND resolution = Fixed`
3. Present findings to the user: duplicates, related closed issues with their resolutions, and any actionable comments.

### 2.3 Knowledge Base Search

**Objective:** Find relevant documentation and known issues.

**Steps:**
1. Extract key technical terms from the bug description.
2. Use `remote_web_search` with a focused query, then `web_fetch` to pull the most relevant pages. The list of sources to consult is in *Knowledge Base Sources* in `reference.md`.
3. Summarize troubleshooting steps or known limitations for the user.
4. If automated search returns nothing useful, ask the user to share any internal documentation links they know about.

### 2.4 Clarifying Questions

Ask targeted questions across these categories. Group them so the user can answer in one pass:

**A. Bug Reproduction**
- Exact reproduction steps?
- Expected vs. actual behavior?
- Consistent or intermittent?

**B. Environment**
- Which environment(s) — dev, staging, production?
- Browser / client version?
- Specific roles or permissions involved?

**C. Impact**
- Number of users affected?
- Workaround available?
- Business impact?

**D. Technical Context**
- Which layers are likely involved? Cross-reference with *Layer-to-Repository Mapping*.
- Error messages or stack traces?
- When did this start occurring?

### 2.5 User Hints & Contextual Information

Prompt the user for:
1. **Reference files** — similar patterns elsewhere, related components that work correctly, prior fixes for similar issues.
2. **Corner cases** — edge cases, specific data scenarios, configuration differences.
3. **Additional context** — recent changes that may have caused the regression, related features, known limitations.

---

## Phase 3: Root Cause Analysis

### 3.0 Request/Response Flow Context

**Before starting code investigation, ask the user which request/response flow(s) are relevant to this bug.** Different bugs traverse different layers — knowing the flow up front narrows the investigation scope significantly.

Prompt the user:

```markdown
Before I start analyzing the code, which request/response flow should I focus on for this bug?

For example:
- Does this bug involve the UI (full end-to-end flow)?
- Is this purely a backend issue (no UI involvement)?
- Does it span specific layers only?

Please describe the flow or tell me which repositories/layers are involved.
```

Use the user's answer to scope which repos and code paths to investigate in Steps 2.1 onward. If the user is unsure, investigate broadly but check back frequently.

### 3.1 Code Investigation

**Steps:**
1. **Identify affected repositories** using the *Request / Response Flow* and *Layer-to-Repository Mapping* sections in `reference.md`.
2. **Read API contracts.** Each repo's API contract file path is the **API contract file** field of its block under *Repositories* in `reference.md`. Use `read_file` for files in the active workspace, or `mcp_github_get_file_contents` to fetch from repos that aren't checked out locally.
3. **Search for relevant code:**
   - `grep_search` with regex patterns scoped via `includePattern` (e.g., `**/*.ts`, `**/*.java`).
   - `read_code` with a `selector` to jump to a specific class, method, or component.
   - `file_search` when you know part of a filename but not its full path.
   - `list_directory` for shallow exploration.
4. **Trace the request flow** end-to-end as described in *Request / Response Flow*.

> Prefer the `context-gatherer` sub-agent (via `invoke_sub_agent`) when the repository is unfamiliar or the bug spans many files. It preserves main-context budget for the actual fix.

### 3.2 Missing Information Handling

1. If a needed file isn't accessible, ask the user for the path and explain why it's needed.
2. If information is unclear, ask specific technical questions or request logs, error messages, screenshots, or configuration.

### 3.3 Root Cause Statement

Present the conclusion in this exact format:

```markdown
## Root Cause Analysis

### Issue Location
- Repository: [repo-name]
- File(s): [file-paths]
- Component: [component-name]

### Root Cause
[Clear explanation of what is causing the bug]

### Why It Happens
[Technical explanation of the underlying issue]

### Impact
[What parts of the system are affected]
```

---

## Phase 4: Solution Proposal

### 4.1 Propose Fix

Present in this format:

```markdown
## Proposed Solution

### Approach
[High-level description of the fix]

### Changes Required

#### Repository: [repo-name]
**Files to Modify:**
1. [file-path-1]
   - Change: [description]
   - Reason: [why this change is needed]
2. [file-path-2]
   - Change: [description]
   - Reason: [why this change is needed]

**New Files (if any):**
- [file-path]: [purpose]

### Testing Strategy
- Unit tests to add/modify
- Integration test scenarios
- Manual testing steps

### Backward Compatibility
[How this preserves compatibility with existing code]

### Risks & Considerations
[Potential issues or edge cases to watch for]
```

### 4.2 User Review & Approval

Ask the user to choose one of:
- Approve the proposed fix
- Request changes to the solution
- Provide additional context
- Suggest an alternative approach

If changes are requested, incorporate feedback, re-present, and repeat until approved. **Do not proceed to Phase 5 without explicit approval.**

---

## Phase 5: Scope Identification & Ticket Management

### 5.1 Identify Scope of Change

Map every approved change to a repository using the *Layer-to-Repository Mapping* table in `reference.md`. Each row in that table that matches a layer touched by your fix becomes one entry in the scope list.

Output:

```markdown
## Scope of Changes

### Repositories Affected
1. [repo-1] ([project-key]) — Changes: [description]
2. [repo-2] ([project-key]) — Changes: [description]
3. [repo-3] ([project-key]) — Changes: [description]
```

### 5.2 Jira Ticket Analysis

```
IF fix_needed_only_in_different_project THEN
  // Original ticket in wrong project
  request_user_to_move_ticket_to_correct_project
  wait_for_new_ticket_id
  use_new_ticket_for_remaining_workflow
ELSE IF original_ticket_project_key != repository_project_key THEN
  // Fix spans multiple projects
  create_ticket_in_repository_project
END IF
```

Cross-reference each affected repository's **Jira project key** (under *Repositories* in `reference.md`) against the original ticket's project key to decide which path applies.

**Scenario A — Ticket Move Required (fix only in a different project):**
- Inform the user that the ticket is in the wrong project.
- The Jira MCP server does not have a move-ticket operation, so ask the user to move it manually.
- Wait for the new ticket ID and continue with it.

Suggested wording (substitute actual project names from `reference.md`):

```markdown
The original ticket [ORIGINAL-TICKET] is in the [ORIGINAL-PROJECT] project, but the fix is only needed in [TARGET-REPO].

Since I don't have permissions to move tickets, please:
1. Move [ORIGINAL-TICKET] from [ORIGINAL-PROJECT] to [TARGET-PROJECT].
2. Share the new ticket ID once the move is complete.

I'll continue the workflow with the new ticket.
```

**Scenario B — Multiple Tickets Required:**
- Create accompanying tickets in each affected project and link them back to the original.

### 5.3 Create Missing Tickets

1. **List tickets to create** with project keys pulled from *Repositories* in `reference.md`:

```markdown
## Tickets to Create

1. Project: [PROJECT-KEY] ([repo-name])
   - Summary: [derived from original ticket]
   - Description: [link to original + specific changes needed]
   - Link to: [original-ticket]

2. Project: [PROJECT-KEY] ([repo-name])
   - Summary: [derived from original ticket]
   - Description: [link to original + specific changes needed]
   - Link to: [original-ticket]
```

2. **Get user approval.** Allow edits to summary or description before creation.
3. **Create the tickets** with `mcp_jira_create_issue` (one call per ticket).
4. **Link them** to the original with `mcp_jira_link_issues` using `linkType: "Relates"` (or the appropriate type for the relationship).
5. **Optionally add a comment** to the original ticket with `mcp_jira_add_comment` referencing the new tickets.
6. **Store the created ticket IDs** for branch naming.

### 5.4 Branch Name Generation

1. **Ask the user for their alias** (e.g., "john", "sarah", "mike").
2. **Generate one branch name per affected repo** using the format defined in *Branch Naming Convention* in `reference.md`.
3. **Look up each repo's base branch** from the **Base branch** field under *Repositories* (or the *Base Branches (Quick Reference)* table).
4. **Present every branch for approval** in this shape (one row per affected repo):

```markdown
## Proposed Branch Names

| Repository | Base Branch | Feature Branch |
| --- | --- | --- |
| [repo-1] | [base-1] | [feature-branch-1] |
| [repo-2] | [base-2] | [feature-branch-2] |

Please confirm or provide alternative branch names.
```

Wait for explicit approval (or modifications) before proceeding.

### 5.5 Create Branches

For each approved branch, call `mcp_github_create_branch` using:
- `owner` from *GitHub Organization* in `reference.md`
- `repo` from the repo's **Repository name (git)** field under *Repositories*
- `from_branch` from the repo's **Base branch** field

Handle errors:
- **Branch already exists:** ask the user whether to reuse, rename, or recreate.
- **Permission denied:** surface the error and provide manual steps as fallback.

Confirm successful creation to the user, listing each new branch URL where available.

### 5.6 Checkout Created Branches

For each repository, ensure the local base branch is up to date **before** checking out the feature branch. This avoids working on a stale foundation. Use `execute_bash` with the repo's absolute path in `cwd`:

```bash
# 1. Pull the latest base branch (look up base branch from Repositories in reference.md)
git checkout <base-branch>
git pull origin <base-branch>

# 2. Fetch and checkout the feature branch
git fetch origin <feature-branch>
git checkout <feature-branch>

# 3. Verify
git branch --show-current
```

Handle:
- Uncommitted changes — ask the user to stash or commit first.
- Local copy missing — fall back to `mcp_github_push_files` to push edits via the GitHub API.

> **Git safety:** never modify `git config`, never force-push, never `--no-verify`, never `git add .` blindly — stage explicit files. See the Kiro git safety guidelines.

---

## Phase 6: Implementation

> No mode switch needed. Kiro has full file-editing and MCP access throughout the session.

**MANDATORY: Load bugfix behavioral standards before writing any code:**

#[[file:standards.md]]

### 6.1 Code Changes

1. **Verify** you're on the correct feature branch in each repo (`git branch --show-current`).
2. **Apply changes** using the appropriate Kiro tool:
   - **`str_replace`** for targeted edits. Match the existing text exactly, including whitespace.
   - **`fs_write`** for new files or full rewrites where targeted edits would be impractical.
   - **`fs_append`** for adding to the end of an existing file.
   - **`semantic_rename`** for symbol renames across the codebase.
   - **`smart_relocate`** for moving files with automatic import updates.

After each meaningful edit, run `get_diagnostics` on the touched files to catch type/lint errors immediately.

### 6.2 Unit Test Generation

1. **Identify test needs:** new functionality, edge cases, regression tests for the bug, backward compatibility.
2. **Generate tests** following each repo's testing standards. Pull these from `reference.md` *Repositories*:
   - **Testing framework** field — Jest, JUnit, pytest, etc.
   - **Test layout** field — where tests live relative to production code
   - **Coding standards reference** — for repo-specific test patterns (mocking style, query selectors, ordering rules, coverage targets, etc.)
3. **Use descriptive test names** that reference the ticket where useful, e.g. `testBugFix_<TICKET-ID>_HandlesNullInput`.


### 6.4 Code Review Checkpoint

Summarize for the user:

```markdown
## Implementation Complete

### Changes Summary

#### Repository: [repo-name] ([feature-branch])
**Modified Files:**
- [file-path]: [summary]

**New Files:**
- [file-path]: [purpose]

**Tests Added/Modified:**
- [test-file]: [scenarios]

[Repeat per affected repo]

### Verification
- Lint: ✅
- Unit tests: ✅
- Build: ✅

### Next Steps
Please review and test the changes. Once approved, I'll create the pull requests.
```

Wait for the user to review, test locally, and approve before Phase 6.

---

## Phase 7: Pull Request Creation

> No mode switch needed.

### 7.1 Wait for User Testing

Explicitly state: "Waiting for review and local testing." Offer the user these choices:
- Tested and approved — proceed with commit and push
- Issues found — return to Phase 5 for modifications
- Need more time

### 7.2 Commit and Push

For each repository with changes:

1. **Run any pre-commit step** required by that repo. Pre-commit requirements (e.g., a formatter) are listed under **Pre-commit requirements** in the repo's *Repositories* block in `reference.md`.
2. **Stage specific files** (avoid blanket `git add .`).
3. **Commit** with the ticket ID as prefix: `git commit -m "<TICKET-ID>: <brief description>"`.
4. **Push** with upstream tracking: `git push -u origin <feature-branch>`.

Use `execute_bash` and set `cwd` to the repo's absolute path.

**Safety reminders:**
- Never use `git commit --amend` after a pre-commit hook failure; fix and create a new commit instead.
- Never push directly to a base branch listed in *Base Branches (Quick Reference)*.
- Never use `--no-verify` unless the user explicitly asks.
- Flag any file that may contain secrets (`.env`, credentials, tokens) before staging it.

If push fails due to SSH auth, tell the user what's needed and retry once they confirm.

### 7.3 Read PR Templates

Load each repo's PR template with `read_file` (local) or `mcp_github_get_file_contents` (remote). The template path for each repo is the **PR template file** field under *Repositories* in `reference.md`. The specific concerns each template covers are listed under *Pull Request Template Requirements* in the same file.

### 7.4 Generate PR Descriptions

Populate each template with:
1. **Ticket reference** — link to the Jira ticket(s); include the ticket ID in the PR title.
2. **Description** — what changed, why, root cause, solution.
3. **Testing** — unit tests added/modified, manual steps performed, integration / UI test status.
4. **Checklist** — mark applicable items. Disclose AI assistance (Kiro). Confirm backward compatibility.

Cross-check the populated description against *Pull Request Template Requirements* for that repo to make sure nothing required is missing.

Keep the title under ~70 characters; put details in the body.

### 7.5 Create Pull Requests

For each repo, call `mcp_github_create_pull_request` with:
- `owner` from *GitHub Organization* in `reference.md`
- `repo` from the repo's **Repository name (git)** field
- `head` set to the feature branch
- `base` set to the repo's **Base branch**
- `title`, `body` populated from Phase 6.4
- `draft: false` (unless the user requests draft)

After creation:
- Display the PR URLs to the user.

---

## Phase 8: Completion

Provide the final summary:

```markdown
## Bug Fix Workflow Complete

### Original Issue
- Ticket: [original-ticket]
- Summary: [bug summary]

### Root Cause
[Brief root cause statement]

### Solution Implemented
[Brief solution description]

### Tickets
1. [original-ticket] (original)
2. [created-ticket-1] (created)
3. [created-ticket-2] (created)

### Branches
[One per affected repo: repo → branch]

### Pull Requests
[One per affected repo: repo → PR URL]

### Next Steps
- PRs are ready for review
- Await CI/CD pipeline results
- Address any review comments
- Merge after approval
```

---

## Workflow State Management

Throughout the workflow, keep a running record of:

1. **Current phase** — Workflow Reference Setup → Information Gathering → Root Cause → Solution → Scope → Implementation → PR Creation → Completion.
2. **Collected information** — Jira ticket details, similar issues, user context, root cause, approved solution.
3. **Created artifacts** — tickets, branches, modified files, PRs.

Display progress at every phase transition:

```markdown
✅ Phase 1: Workflow Reference Setup — Complete
✅ Phase 2: Information Gathering — Complete
✅ Phase 3: Root Cause Analysis — Complete
🔄 Phase 4: Solution Proposal — In Progress
⏳ Phase 5: Scope Identification — Pending
⏳ Phase 6: Implementation — Pending
⏳ Phase 7: PR Creation — Pending
⏳ Phase 8: Completion — Pending
```

---

## Error Handling

### Common Scenarios

1. **Jira ticket not found**
   - Verify ticket ID format against *Project Keys (Quick Reference)*.
   - Check the user has access.
   - Ask for ticket details manually.

2. **Branch already exists**
   - Suggest an alternative name.
   - Ask if the user wants to reuse it.
   - Offer to delete and recreate (only with explicit confirmation).

3. **Permission issues**
   - Surface the requirement to the user.
   - Suggest contacting the repo admin.
   - Provide manual fallback steps.

4. **API contract conflicts**
   - Highlight the conflict.
   - Propose resolution strategies.
   - Wait for user guidance.

5. **Test failures**
   - Analyze the failure.
   - Propose and apply fixes.
   - Iterate until tests pass.

6. **`reference.md` missing or stale**
   - Ask the user where the reference lives or whether it needs updating.
   - Do not invent repo names, project keys, paths, or commands as a fallback.

### Fallback Mechanisms

1. **MCP tool failure** — provide manual steps, use alternative tools, document workarounds.
2. **Missing information** — ask targeted questions, provide examples, offer to proceed with explicit assumptions.
3. **Complex scenarios** — break into smaller steps, seek user input more often, document each decision.

---

## Best Practices

1. **Always consult `reference.md`** — every repo name, path, project key, base branch, and command must come from there. Do not hard-code in your responses.
2. **Approval gates** — wait for explicit approval before creating tickets, branches, commits, or PRs.
3. **Clear communication** — use structured output, progress markers, and explicit phase transitions.
4. **Test thoroughly** — cover happy path, edge cases, and backward compatibility.
5. **Document well** — update docs, write meaningful PR descriptions, link related tickets.
6. **Honor coding standards** — load each repo's *Coding standards reference* before non-trivial edits.
7. **Use sub-agents wisely** — invoke `context-gatherer` for unfamiliar code exploration to preserve context budget.
8. **Be conservative with destructive actions** — never delete branches, force-push, or rewrite history without explicit confirmation.

---

## Example Walkthrough (Illustrative — uses concrete sample values)

This walkthrough shows the workflow in action against a sample team configuration. The repo names, tickets, and branches below are **examples** — your team's actual values come from `reference.md`.

**Scenario:** Null pointer exception in report rendering.

**Phase 1: Workflow Reference Setup**
```
User: Fix this bug: ETS-1234
Core-workflow: [detects bug-fix intent, delegates to Bug Fix Workflow]
Kiro: [displays welcome]
      [checks for reference.md — found]
      [presents reference to user for confirmation]
User: Use as-is
```

**Phase 2: Information Gathering**
```
Kiro: [calls mcp_jira_get_issue for ETS-1234]
      [calls mcp_jira_search_issues for similar issues]
      [remote_web_search against Knowledge Base Sources]
      [asks clarifying questions]
      [requests user hints]
```

**Phase 3: Root Cause Analysis**
```
Kiro: [grep_search across the UI repo from Layer-to-Repository Mapping]
      [read_files on suspect components]
      [traces flow per Request / Response Flow]
      Root Cause: Missing null check in ReportRenderer.tsx
```

**Phase 4: Solution Proposal**
```
Kiro: [proposes fix with null checks and defensive programming]
User: Approved
```

**Phase 5: Scope & Tickets**
```
Kiro: Changes needed in UI repo only — no additional tickets required.
      Proposed branch: wip-john-ETS-1234 (base: master, per Repositories)
User: Approved
Kiro: [mcp_github_create_branch] [git checkout]
```

**Phase 6: Implementation**
```
Kiro: [str_replace for the fix]
      [fs_write for new test file]
      [get_diagnostics passes]
      [Build & test commands from Repositories all pass]
User: Tested and approved
```

**Phase 7: PR Creation**
```
Kiro: [pre-commit step from Repositories]
      [git add specific files] [git commit] [git push -u]
      [reads PR template from Repositories]
      [mcp_github_create_pull_request with owner from GitHub Organization]
      PR created: https://github.com/.../pull/123
      [mcp_jira_add_comment with PR link]
```

**Phase 8: Completion**
```
Kiro: [final summary]
```

---

## Sample PR Description (Illustrative)

The exact fields you populate will vary per repo (see *Pull Request Template Requirements* in `reference.md`). This is a representative example:

```markdown
### Ticket
Resolves [ETS-1234]
Related: [ODS-5678], [BIIT-9012]

### Contribution Type
- [x] Bug fix
- [x] I have added tests to cover my changes
- [x] I have used an AI tool to review this code

### Description
Fixed null pointer exception in report rendering when ...

**Root Cause:**
The issue occurred because ...

**Solution:**
Implemented null checks and ...

**Testing:**
- Added unit tests for null input scenarios
- Verified backward compatibility with existing reports
- Tested manually in local environment

### Verification Steps
1. Open report with null data
2. Verify no error is thrown
3. Confirm graceful handling

### Developer Done List
#### AI Assistance
- [x] AI tools have been used (Kiro)
- [x] I have reviewed and understood all changes

#### Testing
- [x] Tests Added/Updated
- [x] UITs are passing
- [x] Verified backward compatible
```

---

## Conclusion

This workflow provides a structured, Kiro-native, **team-agnostic** approach to bug fixing that:
- Carries zero hard-coded team details — everything lives in `reference.md`
- Leverages Kiro's first-class Jira and GitHub MCP integrations
- Uses Kiro's full tool surface (file edits, semantic refactors, sub-agents, diagnostics) without mode switches
- Maintains consistency across any number of repositories
- Provides clear documentation, traceability, and user approval gates at every critical step
