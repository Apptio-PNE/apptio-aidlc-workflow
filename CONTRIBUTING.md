# Contributing to aidlc-workflow

This repo is the organization's source of truth for AI-DLC steering content. Pilot teams **fork** this repo and pull updates manually. This document describes:

- What you may change in your fork without an upstream PR (the **customization model**).
- How to open a clean upstream PR from a fork that contains team-generated AIDLC docs.
- Versioning rules and the release process.
- Initial setup prerequisites for maintainers (CODEOWNERS, branch protection).

If you read only one section before opening your first PR, read **["Contributing back from a fork"](#contributing-back-from-a-fork)** — branching from your fork's `main` will pollute the diff with team-generated docs and is the single most common contributor mistake.

---

## Scope

### In-scope (changes ship via this repo)

- `.kiro/steering/aws-aidlc-rules/core-workflow.md`
- `.kiro/aws-aidlc-rule-details/common/**`
- `.kiro/aws-aidlc-rule-details/inception/**`
- `.kiro/aws-aidlc-rule-details/construction/**`
- `.kiro/aws-aidlc-rule-details/operations/**`
- `.kiro/aws-aidlc-rule-details/extensions/**` (when shipped centrally)
- The `.bob/` mirror of the above
- Root governance files: `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `manifest.yaml`, `CODEOWNERS`, `.gitignore`

### Out-of-scope (never PR these to upstream)

- `aidlc-docs/` — generated workflow artifacts inside team forks
- Team-specific specs, design docs, code, or any other team output

If a PR includes any out-of-scope file, the reviewer will reject it. The cleanest way to avoid this is the upstream-branching workflow below.


---

## Contributing back from a fork

**Critical:** Branch your upstream-bound PR from `upstream/main`, not from your fork's `main`. Your fork's `main` contains team-generated AIDLC docs, and a PR opened from it will show those files in the diff even when you only intended to change steering content.

### One-time setup

```
git remote add upstream <upstream-url>
git fetch upstream
```

### Every time you contribute back

```
git fetch upstream
git checkout -b fix/clarify-nfr-rule upstream/main   # branch from upstream, NOT from your fork's main
# make changes only to in-scope files
git add <only the files you intended to change>
git commit -m "Clarify NFR rule wording"
git push origin fix/clarify-nfr-rule
# open PR from your fork's branch -> upstream main
```

### Why this matters

If you instead run `git checkout -b fix/clarify-nfr-rule main` (branching from your fork's `main`), every team-generated file under `aidlc-docs/`, every spec, every design artifact will appear in the PR diff. CODEOWNERS catches some of this socially during review, but it is not a technical block — clean diffs are your responsibility.

### PR checklist

Before opening a PR, confirm:

- [ ] Branch was cut from `upstream/main`, not your fork's `main`.
- [ ] Only **in-scope** paths are modified (see [Scope](#scope)).
- [ ] `CHANGELOG.md` `[Unreleased]` section has a new entry describing the change.
- [ ] If the change is breaking or deprecates something, `manifest.yaml` is updated accordingly (see [Versioning](#versioning)).

---

## Versioning

Releases follow [Semantic Versioning](https://semver.org/) and ship as annotated git tags (`vMAJOR.MINOR.PATCH`).

| Bump  | When                                                                                           | Examples                                                                  |
|-------|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| MAJOR | Breaking change to rule semantics, removed rules, restructured directories consumers reference. | Renaming `common/process-overview.md`; removing a stage from the workflow. |
| MINOR | New rules, new extensions, new `*.opt-in.md` files, additive content.                          | Adding a new extension under `extensions/`; adding a new common rule.     |
| PATCH | Wording fixes, typo corrections, clarifications without semantic change.                       | Fixing a typo; rephrasing a sentence; adding an example.                  |

The current version, release date, shipped paths, rule packs, extensions, and breaking-change/deprecation history all live in **`manifest.yaml`**. This is the **single source of truth** for version metadata. There is no separate `VERSION` file. If a PR bumps the version, it bumps `manifest.yaml`.

---

## Release process

The release process is intentionally lightweight and manual. A maintainer cuts a release when accumulated `[Unreleased]` changes warrant it.

### Prerequisite (one-time)

Before the first release:

- The team referenced in `CODEOWNERS` (e.g., `@your-org/aidlc-maintainers`) **must exist on GitHub**. If the team does not exist, GitHub silently ignores CODEOWNERS rules and review requirements never trigger.
- Branch protection on `main` must be configured per [Branch protection settings](#branch-protection-settings).

### Release checklist

1. Decide the bump type (MAJOR / MINOR / PATCH) based on `[Unreleased]` content in `CHANGELOG.md`.
2. In `CHANGELOG.md`, move all `[Unreleased]` entries into a new `[X.Y.Z] - YYYY-MM-DD` section.
3. In `manifest.yaml`:
   - Bump `version` to `X.Y.Z`.
   - Set `released` to today's date (`YYYY-MM-DD`).
   - If MAJOR: append the breaking change(s) to `breaking_changes` using the schema shown in the manifest's commented example.
   - If applicable: append to `deprecations` using the schema shown in the manifest's commented example.
   - Bump matching `rule_packs[*].version` entries if their content changed.
4. Open a PR titled `Release vX.Y.Z` containing only the CHANGELOG and manifest changes. Wait for CODEOWNERS approval and merge.
5. Tag the merge commit on `main`:
   ```
   git checkout main
   git pull
   git tag -a vX.Y.Z -m "Release vX.Y.Z"
   git push origin vX.Y.Z
   ```
6. Create a GitHub Release from the tag. Paste the new CHANGELOG section (`[X.Y.Z]`) verbatim as the release notes.

That is the complete release. No automation runs; teams pick up the release on their next `git fetch upstream`.

---

## Initial setup prerequisites (maintainers)

Run through this once when the repo is first stood up:

1. **Create the CODEOWNERS team on GitHub.** The handle in `CODEOWNERS` (placeholder: `@your-org/aidlc-maintainers`) must reference a team that actually exists in your GitHub org. Update the handle in `CODEOWNERS` to your real team if it differs from the placeholder.
2. **Configure branch protection on `main`** per [Branch protection settings](#branch-protection-settings).
3. **Verify CODEOWNERS works** by opening a test PR that touches a steering path and confirming the team is auto-requested as a reviewer. If it isn't, the team handle is wrong or the team doesn't exist — fix before announcing the repo to pilot teams.

### Branch protection settings

Configure the following on the `main` branch (Settings → Branches → Branch protection rules):

- **Require a pull request before merging:** enabled.
- **Require approvals:** at least 1.
- **Require review from Code Owners:** enabled.
- **Dismiss stale pull request approvals when new commits are pushed:** enabled (recommended).
- **Do not allow bypassing the above settings:** enabled.
- **Restrict who can push to matching branches:** allow only the maintainers team to push tags and merge commits.
- **Allow force pushes:** disabled.
- **Allow deletions:** disabled.

These settings are configured in the repo UI, not in code. They are listed here so any maintainer can recreate them after a repo migration or audit.
