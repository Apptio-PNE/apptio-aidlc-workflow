# Bug Fix Workflow (Bob) - Implementation Plan

> Bob mode implementation of the Bug Fix Workflow. Generic by design — every team-specific value (repo names, paths, commands, base branches, project keys, GitHub owner) lives in `reference.md`. This file describes the **process**; the reference describes the **environment**.

---

## Workflow Activation

### Trigger Mechanism
This workflow **cannot be triggered independently by the user**. It is activated exclusively by the **core-workflow** when it determines that a bug-fix workflow is appropriate for the user's request.

**How activation works:**
- The core-workflow analyzes the user's request during its Workspace Detection and Requirements Analysis stages.
- If the core-workflow identifies the request as a bug fix (e.g., references a Jira bug ticket, describes a defect to resolve, or explicitly asks to fix a bug), it delegates execution to this Bug Fix Workflow.
- Once activated by the core-workflow, this workflow remains in effect for the duration of the bug-fix conversation.

### Project Reference

All team-specific configuration — repository names, project keys, base branches, API contract paths, PR template paths, build commands, GitHub owner, knowledge base sources — lives in `reference.md` (in this folder).

Throughout this workflow, instructions like "see *Repositories* in `reference.md`" mean: read that exact section heading in the reference file and use the values found there. Do NOT hard-code repository names, branch names, project keys, or file paths in your responses — always pull them from the reference.

**Before starting, fully load the reference file into context**, load the reference file into context using read_file on reference.md. If it is missing, Phase 1 will generate it.

### Initial Setup
When the workflow is activated:
1. **Switch to Plan Mode** - Use `switch_mode` tool to explicitly switch to plan mode before proceeding
2. **Display welcome message** (see below) immediately
3. **Ensure `reference.md` exists** (see *Phase 1* below). This must complete before Phase 2.
4. Load `reference.md` for reference (use `read_file`)
5. Prompt user for Jira ticket information
6. Initialize workflow state tracking

**Important:** Step 1 must be completed before any Jira MCP operations to ensure proper mode context for information gathering and analysis.

### Welcome Message
When the bugfix workflow is triggered, display the following message to the user:

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

Wait for the user's response. If they say "use as-is", proceed to Phase 2. If they say "regenerate", proceed with the generation steps below.

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

This file is the **single source of team-specific configuration** for the Bug Fix Workflow. The workflow files (`.kiro/aws-aidlc-rule-details/bugfix/workflow.md` and `.bob/aws-aidlc-rule-details/bugfix/workflow.md`) read everything they need from here.

> **Auto-generated file.** This was created by the Bug Fix Workflow during Phase 1 (Workflow Reference Setup). To regenerate it, trigger the workflow and choose "Regenerate" when prompted, or start a new session if the file has been deleted.
```

Hardcode:
- Jira base URL: `https://apptio.atlassian.net`
- Knowledge base: IBM Help Docs (`https://www.ibm.com/docs/en`)

**Step 7: Confirm** to the user that the file was generated, then proceed to Phase 2.

**Tools Used:** `execute_command` (git operations), `list_files`, `read_file`, `write_to_file`, `ask_followup_question`

---

## Phase 2: Information Gathering & Analysis

### 2.1 Jira Ticket Information Collection
**Objective:** Gather complete bug information from Jira

**Prerequisites:** Workflow must be in **Plan Mode** (switched during Initial Setup, Step 1)

**Steps:**
1. Ask user for Jira ticket ID. Valid project keys are listed under *Project Keys (Quick Reference)* in `reference.md`.
2. Use Jira MCP `get_issue` tool to fetch ticket details (summary, description, status, priority, assignee, comments, attachments).
3. Display ticket information to user for confirmation.

**Tools Used:** `use_mcp_tool` with Jira server's `get_issue`

---

### 2.2 Search for Similar Issues
**Objective:** Find duplicate or related issues that might provide insights

**Steps:**
1. Extract key terms from bug description.
2. Use Jira MCP `search_issues` tool with focused JQL queries against the project keys listed in *Project Keys (Quick Reference)*. Generic query shape:
   - `project = <PROJECT-KEY> AND summary ~ "<keyword>" AND status in (Closed, Resolved)`
   - `project = <PROJECT-KEY> AND labels = "<label>" AND resolution = Fixed`
3. Present findings to user: duplicates, related closed issues with their resolutions, and any actionable comments.

**Tools Used:** `use_mcp_tool` with Jira server's `search_issues`

---

### 2.3 Knowledge Base Search
**Objective:** Find relevant documentation and known issues

**Steps:**
1. Extract key technical terms from bug description.
2. Use `browser_action` to search the sources listed under *Knowledge Base Sources* in `reference.md` (launch, search, capture screenshots, extract troubleshooting steps).
3. Summarize findings for user.

**Alternative:** If browser automation is not suitable, prompt user to provide relevant documentation links.

**Tools Used:** `browser_action`, or `ask_followup_question`

---

### 2.4 Clarifying Questions
**Objective:** Get complete understanding of the bug

**Question Categories:**

**A. Bug Reproduction:**
- Exact reproduction steps?
- Expected vs. actual behavior?
- Consistent or intermittent?

**B. Environment:**
- Which environment(s) — dev, staging, production?
- Browser/client version?
- Specific user roles or permissions?

**C. Impact:**
- Number of users affected?
- Workaround available?
- Business impact?

**D. Technical Context:**
- Which layers are likely involved? Cross-reference with *Layer-to-Repository Mapping* in `reference.md`.
- Error messages or stack traces?
- When did this start occurring?

**Tools Used:** `ask_followup_question`

---

### 2.5 User Hints & Contextual Information
**Objective:** Gather additional context not in the bug description

**Prompt User For:**
1. **Reference Files:** similar patterns elsewhere, related components that work, prior fixes for similar issues.
2. **Corner Cases:** edge cases, specific data scenarios, configuration differences.
3. **Additional Context:** recent changes that may have caused the regression, related features, known limitations.

**Tools Used:** `ask_followup_question`

---

## Phase 3: Root Cause Analysis

### 3.0 Request/Response Flow Context
**Objective:** Understand which flow path this bug traverses before diving into code

**Before starting code investigation, ask the user which request/response flow(s) are relevant to this bug.** Different bugs traverse different layers — knowing the flow up front narrows the investigation scope significantly.

Use `ask_followup_question`:

```markdown
Before I start analyzing the code, which request/response flow should I focus on for this bug?

For example:
- Does this bug involve the UI (full end-to-end flow)?
- Is this purely a backend issue (no UI involvement)?
- Does it span specific layers only?

Please describe the flow or tell me which repositories/layers are involved.
```

Use the user's answer to scope which repos and code paths to investigate in Steps 2.1 onward. If the user is unsure, investigate broadly but check back frequently.

**Tools Used:** `ask_followup_question`

---

### 3.1 Code Investigation
**Objective:** Identify the root cause of the bug

**Steps:**
1. **Identify Affected Repositories** using the *Request / Response Flow* and *Layer-to-Repository Mapping* sections in `reference.md`.
2. **Read API Contracts.** Each repo's API contract file path is the **API contract file** field of its block under *Repositories* in `reference.md`. Use `read_file` to load each one.
3. **Search for Relevant Code:**
   - `search_files` to find related implementations
   - `list_code_definition_names` for code structure
   - `read_file` to examine specific files
4. **Trace Request Flow** end-to-end as described in *Request / Response Flow* in `reference.md`.

**Tools Used:** `read_file`, `search_files`, `list_code_definition_names`, `list_files`

---

### 3.2 Missing Information Handling
**Objective:** Request any missing files or information

1. If specific files are needed but not accessible, use `ask_followup_question` to request paths and explain why each file is needed.
2. If information is unclear, ask specific technical questions or request logs, error messages, screenshots, or configuration.

**Tools Used:** `ask_followup_question`, `read_file` once paths are provided

---

### 3.3 Root Cause Statement

```markdown
## Root Cause Analysis

### Issue Location:
- Repository: [repo-name]
- File(s): [file-paths]
- Component: [component-name]

### Root Cause:
[Clear explanation of what is causing the bug]

### Why It Happens:
[Technical explanation of the underlying issue]

### Impact:
[What parts of the system are affected]
```

---

## Phase 4: Solution Proposal

### 4.1 Propose Fix

```markdown
## Proposed Solution

### Approach:
[High-level description of the fix]

### Changes Required:

#### Repository: [repo-name]
**Files to Modify:**
1. [file-path-1]
   - Change: [description]
   - Reason: [why this change is needed]

**New Files (if any):**
- [file-path]: [purpose]

### Testing Strategy:
- Unit tests to add/modify
- Integration test scenarios
- Manual testing steps

### Backward Compatibility:
[How this maintains compatibility with existing code]

### Risks & Considerations:
[Any potential issues or edge cases to watch for]
```

### 4.2 User Review & Approval

1. Present the proposed solution.
2. Use `ask_followup_question` with options: Approve / Request changes / Provide additional context / Suggest an alternative.
3. If changes requested, incorporate feedback, re-present, and repeat until approved.

**Tools Used:** `ask_followup_question`

---

## Phase 5: Scope Identification & Ticket Management

### 5.1 Identify Scope of Change

1. Analyze the approved solution.
2. Map every approved change to a repository using the *Layer-to-Repository Mapping* table in `reference.md`. Each row in that table that matches a layer touched by your fix becomes one entry in the scope list.
3. Reference *Repositories* in `reference.md` for project keys, base branches, and repository names.

**Output:**
```markdown
## Scope of Changes

### Repositories Affected:
1. [repo-1] ([project-key]) — Changes: [description]
2. [repo-2] ([project-key]) — Changes: [description]
3. [repo-3] ([project-key]) — Changes: [description]
```

---

### 5.2 Jira Ticket Analysis

```
IF fix_needed_only_in_different_project THEN
    // Special case: Original ticket in wrong project
    request_user_to_move_ticket_to_correct_project
    wait_for_new_ticket_id
    use_new_ticket_for_remaining_workflow
ELSE IF original_ticket_project_key != repository_project_key THEN
    // Standard case: Fix spans multiple projects
    create_ticket_in_repository_project
END IF
```

Cross-reference each affected repository's **Jira project key** (under *Repositories* in `reference.md`) against the original ticket's project key to decide which path applies.

**Scenario A — Ticket Move Required:** The original ticket is in a project that owns no code in the affected scope. Inform user, ask them to move the ticket manually (AI lacks move permissions), wait for the new ticket ID, then continue.

**Scenario B — Multiple Tickets Required:** Create matching tickets in every project that owns a repo with changes; link all of them to the original.

**User Interaction for Ticket Move** — suggested wording (substitute actual project names from `reference.md`):

```markdown
The original ticket [ORIGINAL-TICKET] is in the [ORIGINAL-PROJECT] project, but the fix is only needed in [TARGET-REPO].

Since I don't have permissions to move tickets, please:
1. Move [ORIGINAL-TICKET] from [ORIGINAL-PROJECT] to [TARGET-PROJECT]
2. Provide the new ticket ID after the move

Once you've moved the ticket, please share the new ticket ID so I can continue with the workflow.
```

---

### 5.3 Create Missing Tickets

1. **List Tickets to Create** with project keys pulled from *Repositories* in `reference.md`:
   ```markdown
   ## Tickets to Create

   1. Project: [PROJECT-KEY] ([repo-name])
      - Summary: [derived from original ticket]
      - Description: [link to original + specific changes needed]
      - Link to: [original-ticket]
   ```
2. **Get user approval.** Allow edits to summaries / descriptions.
3. **Create Tickets** using Jira MCP `create_issue` (one call per ticket) and `link_issues` to link to the original.
4. Store created ticket IDs for branch naming.

**Tools Used:** `use_mcp_tool` with Jira server's `create_issue`, `link_issues`

---

### 5.4 Branch Name Generation

1. **Prompt for alias** via `ask_followup_question` (e.g., "john", "sarah", "mike").
2. **Generate one branch name per affected repo** using the format defined in *Branch Naming Convention* in `reference.md`. Look up each repo's base branch from the **Base branch** field under *Repositories*.
3. **Present for approval:**
   ```markdown
   ## Proposed Branch Names

   | Repository | Base Branch | Feature Branch |
   | --- | --- | --- |
   | [repo-1] | [base-1] | [feature-branch-1] |
   | [repo-2] | [base-2] | [feature-branch-2] |

   Please confirm or provide alternative branch names.
   ```
4. **Get explicit approval** via `ask_followup_question`.

**Tools Used:** `ask_followup_question`

---

### 5.5 Create Branches

For each repository with approved branch name, use GitHub MCP `create_branch` tool with:
- `owner` from *GitHub Organization* in `reference.md`
- `repo` from the repo's **Repository name (git)** field under *Repositories*
- `from_branch` from the repo's **Base branch** field
- `branch` set to the feature branch name approved in 4.4

Handle errors (branch exists, permission issues). Confirm successful creation to user.

**Tools Used:** `use_mcp_tool` with GitHub server's `create_branch`

---

### 5.6 Checkout Created Branches

For each repository where a branch was created, use `execute_command` to checkout the branch locally.

**Example Commands:**
```bash
# Fetch the branch from remote
cd /path/to/<repo> && git fetch origin <feature-branch>

# Checkout the branch
cd /path/to/<repo> && git checkout <feature-branch>

# Verify current branch
cd /path/to/<repo> && git branch --show-current
```

Handle uncommitted changes (stash or commit) and missing local copies (fall back to GitHub API for pushing changes).

**Tools Used:** `execute_command` for git operations

---

## Phase 6: Implementation

**MANDATORY: Load bugfix behavioral standards before writing any code:**

#[[file:standards.md]]

**IMPORTANT: Mode Switch Required.** Before starting implementation, you MUST switch to Code mode:

```xml
<switch_mode>
<mode_slug>code</mode_slug>
<reason>Starting implementation phase - need access to file editing tools (apply_diff, write_to_file, insert_content)</reason>
</switch_mode>
```

Code mode provides `apply_diff`, `write_to_file`, `insert_content` — Plan mode does not.

---

### 6.1 Code Changes

1. **For each repo**, verify you're on the correct feature branch, then make changes using:
   - `apply_diff` for targeted edits
   - `write_to_file` for new files
   - `insert_content` for adding lines

**Tools Used:** `apply_diff`, `write_to_file`, `insert_content`, `read_file` (to verify changes)

---

### 6.2 Unit Test Generation

1. **Identify test needs:** new functionality, edge cases, regression for the bug, backward compatibility.
2. **Generate tests** following each repo's testing standards. Pull these from `reference.md` *Repositories*:
   - **Testing framework** field (Jest, JUnit, pytest, etc.)
   - **Test layout** field (where tests live relative to production code)
   - **Coding standards reference** for repo-specific test patterns
3. **Use descriptive test names** that reference the ticket where useful, e.g. `testBugFix_<TICKET-ID>_HandlesNullInput`.

**Tools Used:** `write_to_file`, `apply_diff`

---

### 6.3 Code Review Checkpoint

```markdown
## Implementation Complete

### Changes Summary:

#### Repository: [repo-name] ([feature-branch])
**Modified Files:**
- [file-path]: [summary of changes]

**New Files:**
- [file-path]: [purpose]

**Tests Added/Modified:**
- [test-file]: [test scenarios]

[Repeat per affected repo]

### Next Steps:
Please review and test the changes. Once approved, I will create pull requests.
```

---

## Phase 7: Pull Request Creation

**Mode Switches in This Phase:**
- **Steps 7.1-7.2:** Stay in Code mode (for git operations)
- **Steps 7.3-7.5:** Switch to Plan mode (for GitHub MCP tools)

---

### 7.1 Wait for User Testing

1. Explicitly state: "Waiting for user to review and test changes"
2. Use `ask_followup_question` with options: tested & approved / issues found / need more time.
3. If issues found, return to Phase 6.1 for fixes.

**Tools Used:** `ask_followup_question`

---

### 7.2 Commit and Push Changes

For each repository with changes:

1. **Run any pre-commit step** required by that repo. Pre-commit requirements (e.g., a formatter) are listed under **Pre-commit requirements** in the repo's *Repositories* block in `reference.md`.
2. **Stage specific files** (avoid blanket `git add .`):
   ```bash
   cd [repository-path] && git add path/to/changed-file
   ```
3. **Commit with descriptive message** using the ticket ID as prefix:
   ```bash
   git commit -m "[TICKET-ID]: Brief description of changes"
   ```
4. **Push to remote feature branch:**
   ```bash
   git push origin [branch-name]
   ```

**SSH Authentication:** If push fails with SSH error, prompt user to enter SSH password and retry.

**Tools Used:** `execute_command`, `ask_followup_question`

**IMPORTANT: Mode Switch Required After This Step.** Switch to Plan mode to access GitHub MCP tools:

```xml
<switch_mode>
<mode_slug>plan</mode_slug>
<reason>Completed commit and push - now need access to GitHub MCP tools for creating pull requests</reason>
</switch_mode>
```

---

### 7.3 Read PR Templates

Load each repo's PR template using `read_file`. The template path for each repo is the **PR template file** field under *Repositories* in `reference.md`. The specific concerns each template covers are listed under *Pull Request Template Requirements* in the same file.

**Tools Used:** `read_file`

---

### 7.4 Generate PR Descriptions

Populate each template with:
1. **Ticket reference** — link to Jira ticket(s); include ticket ID in PR title.
2. **Description** — what changed, why, root cause, solution.
3. **Testing** — unit tests added/modified, manual steps performed, integration / UI test status.
4. **Checklist** — mark applicable items, disclose AI assistance (Bob), confirm backward compatibility.

Cross-check the populated description against *Pull Request Template Requirements* in `reference.md` for that repo.

---

### 7.5 Create Pull Requests

For each repository, use GitHub MCP `create_pull_request` tool with:
- `owner` from *GitHub Organization* in `reference.md`
- `repo` from the repo's **Repository name (git)** field
- `title`: "[TICKET-ID] Brief description"
- `body`: Generated PR description
- `head`: Feature branch name
- `base`: Repo's **Base branch** field
- `draft`: false (unless specified)

After creation:
- Cross-reference PRs in descriptions
- Display PR URLs to user

**Tools Used:** `use_mcp_tool` with GitHub server's `create_pull_request`, Jira's `add_comment`

---

## Phase 8: Completion

### 8.1 Workflow Summary

```markdown
## Bug Fix Workflow Complete

### Original Issue:
- Ticket: [original-ticket]
- Summary: [bug summary]

### Root Cause:
[Brief root cause statement]

### Solution Implemented:
[Brief solution description]

### Tickets Created:
1. [original-ticket] (original)
2. [created-ticket-1] (created)
3. [created-ticket-2] (created)

### Branches Created:
[One per affected repo: repo → branch]

### Pull Requests:
[One per affected repo: repo → PR URL]

### Next Steps:
- PRs are ready for review
- Await CI/CD pipeline results
- Address any review comments
- Merge after approval
```

---

## Workflow State Management

Maintain state for: current phase, collected information (ticket details, similar issues, user context, root cause, approved solution), and created artifacts (tickets, branches, files modified, PRs).

**Progress Indicators:**
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

1. **Jira Ticket Not Found:** Verify ticket ID format against *Project Keys (Quick Reference)*, check access, ask user for details manually.
2. **Branch Already Exists:** Suggest alternative, ask whether to reuse, offer to delete and recreate (only with explicit confirmation).
3. **Permission Issues:** Surface the requirement, suggest contacting repo admin, provide manual fallback steps.
4. **API Contract Conflicts:** Highlight conflicts, propose resolution strategies, wait for user guidance.
5. **Test Failures:** Analyze the failure, propose and apply fixes, iterate until passing.
6. **`reference.md` missing or stale:** Ask the user where the reference lives or whether it needs updating. Do not invent repo names, project keys, paths, or commands as a fallback.

### Fallback Mechanisms

1. **MCP Tool Failures:** Manual steps, alternative tools, document workarounds.
2. **Missing Information:** Targeted questions, examples, offer to proceed with explicit assumptions.
3. **Complex Scenarios:** Break into smaller steps, seek user guidance more often, document each decision.

---

## Integration Points

### Jira MCP Server
- `get_issue` - Fetch ticket details
- `search_issues` - Find similar issues
- `create_issue` - Create new tickets
- `link_issues` - Link related tickets
- `add_comment` - Add workflow updates

### GitHub MCP Server
- `create_branch` - Create feature branches
- `create_pull_request` - Create PRs
- `get_file_contents` - Read repository files
- `push_files` - Push code changes (if needed)

### Browser Automation
- Search sources listed under *Knowledge Base Sources* in `reference.md`
- Verify deployed changes
- Capture screenshots for documentation

---

## Best Practices

1. **Always reference `reference.md`** — every repo name, path, project key, base branch, and command must come from there. Do not hard-code in your responses.
2. **User approval gates** — confirmation before creating tickets, branches, commits, or PRs.
3. **Clear communication** — structured output, progress indicators, explained decisions.
4. **Comprehensive testing** — happy path, edge cases, backward compatibility.
5. **Documentation** — update docs, code comments, clear PR descriptions.

---

## Example Workflow Execution (Illustrative — uses concrete sample values)

This walkthrough shows the workflow in action against a sample team configuration. The repo names, tickets, and branches below are **examples** — your team's actual values come from `reference.md`.

### Scenario: Null Pointer Exception in Report Rendering

**Phase 1: Workflow Reference Setup**
```
User: Fix this bug: ETS-1234
Core-workflow: [detects bug-fix intent, delegates to Bug Fix Workflow]
Bot: [displays welcome]
     [checks for reference.md — found]
     [presents reference to user for confirmation]
User: Use as-is
```

**Phase 2: Information Gathering**
```
Bot: [fetches ticket details via Jira MCP]
     [Searches for similar issues]
     [Searches Knowledge Base Sources]
     [Asks clarifying questions]
     [Requests user hints]
```

**Phase 3: Root Cause Analysis**
```
Bot: [Asks user about request/response flow to focus on]
     [Analyzes code]
     [Identifies issue in UI component]
     [Traces through flow per user's answer]

     Root Cause: Missing null check in ReportRenderer.tsx
```

**Phase 4: Solution Proposal**
```
Bot: [Proposes fix with null checks]
User: Approved
```

**Phase 5: Scope & Tickets**
```
Bot: Changes needed in UI repo only
     No additional tickets required
     Proposed branch: wip-sshankar-ETS-1234 (base: master, per Repositories)
User: Approved
Bot: [create_branch] [git checkout]
```

**Phase 6: Implementation**
```
Bot: [Implements fix] [Adds unit tests]
User: Tested and approved
```

**Phase 7: PR Creation**
```
Bot: [pre-commit step from Repositories]
     [git add specific files] [git commit] [git push]
     [reads PR template from Repositories]
     [Creates PR with comprehensive description]
     PR created: https://github.com/.../pull/123
```

**Phase 8: Completion**
```
Bot: [final summary]
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
- [x] AI tools have been used (Bob)
- [x] I have reviewed and understood all changes

#### Testing
- [x] Tests Added/Updated
- [x] UITs are passing
- [x] Verified backward compatible
```

---

## Conclusion

This workflow provides a structured, **team-agnostic** approach to bug fixing that:
- Carries zero hard-coded team details — everything lives in `reference.md`
- Leverages existing Jira and GitHub MCP integrations
- Maintains consistency across any number of repositories
- Provides clear documentation, traceability, and user approval gates at every critical step
- Incorporates user feedback at key decision points
