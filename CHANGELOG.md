# Changelog

All notable changes to `aidlc-workflow` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

The current version, release date, shipped paths, and compatibility metadata live in [`manifest.yaml`](manifest.yaml). When adding entries here, also update the manifest if the change affects version, breaking changes, or deprecations (see [CONTRIBUTING.md](CONTRIBUTING.md)).

## [Unreleased]

### Added

### Changed

### Deprecated

### Removed

### Fixed

### Security

## [1.0.0] - 2026-06-03

Initial release. Establishes `aidlc-workflow` as the organization's source of truth for AI-DLC steering content, with versioning and governance scaffolding to support the fork-and-pull distribution model used by pilot teams.

### Added

#### Core workflow rule pack: `aws-aidlc-rules`

- `core-workflow.md` — adaptive AI-DLC workflow definition covering the inception, construction, and operations phases. Shipped at `.kiro/steering/aws-aidlc-rules/core-workflow.md` and mirrored under `.bob/rules/aws-aidlc-rules/`.

#### Common rules (`common/`)

Shipped at `.kiro/aws-aidlc-rule-details/common/` and mirrored under `.bob/aws-aidlc-rule-details/common/`:

- `process-overview.md` — workflow overview loaded at start of every session.
- `session-continuity.md` — session resumption guidance.
- `content-validation.md` — content validation requirements (Mermaid, ASCII art, escaping).
- `question-format-guide.md` — multiple-choice question formatting rules.
- `terminology.md` — shared vocabulary across phases.
- `error-handling.md` — error recovery and retry guidance.
- `depth-levels.md` — minimal / standard / comprehensive depth definitions.
- `ascii-diagram-standards.md` — ASCII diagram conventions.
- `overconfidence-prevention.md` — guardrails against overconfident model behavior.
- `welcome-message.md` — one-time welcome message displayed at workflow start.
- `workflow-changes.md` — process for proposing changes to the workflow itself.

#### Inception phase rules (`inception/`)

Shipped at `.kiro/aws-aidlc-rule-details/inception/` and mirrored under `.bob/`:

- `workspace-detection.md` — always-execute workspace and brownfield/greenfield detection.
- `reverse-engineering.md` — conditional brownfield reverse-engineering stage.
- `requirements-analysis.md` — adaptive-depth requirements analysis (always executes).
- `user-stories.md` — conditional user-stories stage with planning + generation parts.
- `workflow-planning.md` — always-execute workflow planning.
- `application-design.md` — conditional application-design stage.
- `units-generation.md` — conditional decomposition into units of work.

#### Construction phase rules (`construction/`)

Shipped at `.kiro/aws-aidlc-rule-details/construction/` and mirrored under `.bob/`:

- `functional-design.md` — conditional functional-design stage (per unit).
- `nfr-requirements.md` — conditional NFR requirements stage (per unit).
- `nfr-design.md` — conditional NFR design stage (per unit).
- `infrastructure-design.md` — conditional infrastructure-design stage (per unit).
- `code-generation.md` — always-execute code-generation stage (per unit, planning + generation parts).
- `build-and-test.md` — always-execute build-and-test stage after all units.

#### Operations phase rules (`operations/`)

Shipped at `.kiro/aws-aidlc-rule-details/operations/` and mirrored under `.bob/`:

- `operations.md` — placeholder for future deployment, monitoring, and incident-response workflows.

#### Extensions (opt-in) (`extensions/`)

Shipped at `.kiro/aws-aidlc-rule-details/extensions/` and mirrored under `.bob/`. Each extension provides a lightweight `*.opt-in.md` sentinel that the workflow surfaces during requirements analysis; the full rules file is loaded on-demand only when the user opts in:

- `security/baseline/` — `security-baseline.opt-in.md` and `security-baseline.md`.
- `testing/property-based/` — `property-based-testing.opt-in.md` and `property-based-testing.md`.

#### Governance scaffolding

- `README.md` — purpose, fork-and-pull consumption model, upgrade paths (latest and pinned), repo layout.
- `CONTRIBUTING.md` — scope rules, customization model, mandatory `upstream/main` branching workflow for clean PRs, versioning policy, release checklist, initial-setup prerequisites, branch-protection settings.
- `CHANGELOG.md` — this file; Keep-a-Changelog format.
- `manifest.yaml` — single source of truth for version and compatibility metadata. No separate `VERSION` file.
- `CODEOWNERS` — central-team ownership over `.kiro/`, `.bob/`, and root governance files.
- `.gitignore` — OS and editor hygiene.

[Unreleased]: ../../compare/v1.0.0...HEAD
[1.0.0]: ../../releases/tag/v1.0.0
