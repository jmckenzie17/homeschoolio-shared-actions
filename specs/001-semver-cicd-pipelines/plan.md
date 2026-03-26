# Implementation Plan: Semantic Versioning CI/CD Pipelines

**Branch**: `001-semver-cicd-pipelines` | **Date**: 2026-03-26 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-semver-cicd-pipelines/spec.md`

## Summary

Deliver a reusable GitHub Actions workflow (`semver-release.yml`) that automates
semantic versioning for any homeschoolio repository. On each push to `main`, it
uses `googleapis/release-please-action` (pinned to commit SHA) to parse
conventional commit messages, calculate the next version, create a Git tag, and
publish a GitHub Release. A companion step maintains the moving major version
pointer (e.g., `v1`). Consumer repos adopt the full pipeline by adding a single
`uses:` line to their own workflow file.

## Technical Context

**Language/Version**: YAML (GitHub Actions workflow syntax) — no application runtime
**Primary Dependencies**: `googleapis/release-please-action@16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0), `actions/checkout` (v4, SHA pinned at implementation)
**Storage**: N/A — state is tracked via Git tags and release-please manifest files in each consumer repo
**Testing**: `actionlint` (workflow YAML linter), release-please `--dry-run` mode
**Target Platform**: GitHub Actions (GitHub-hosted runners, `ubuntu-latest`)
**Project Type**: Reusable GitHub Actions workflow
**Performance Goals**: End-to-end pipeline completes in under 3 minutes (SC-003)
**Constraints**: `contents: write` + `pull-requests: write` permissions only (FR-010); all external actions pinned to commit SHA (Constitution Principle III)
**Scale/Scope**: Single reusable workflow consumed by all homeschoolio repos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|-----------|-------|--------|
| I. Reusable-Action-First | Deliverable is a reusable workflow in `.github/workflows/`. Accompanied by README with inputs, outputs, and usage examples. | ✅ PASS |
| II. Semantic Versioning | This feature *implements* the versioning policy; the repo's own releases will use this same workflow once merged. | ✅ PASS |
| III. Trusted & Auditable Third-Party Actions | `googleapis/release-please-action` pinned to `16a9c908...` (v4.4.0). `actions/checkout` pinned to SHA at implementation time. No mutable tag references. | ✅ PASS |
| IV. Minimal Surface Area | Workflow has one responsibility: version calculation + tag + release. Major pointer update is a plain shell step — no additional action. Inputs limited to what consumers actually need. | ✅ PASS |
| V. Tested Before Release | `test-semver-release.yml` CI workflow runs `actionlint` + dry-run on every PR to `main`. | ✅ PASS |
| Security: least privilege | `contents: write` + `pull-requests: write` only. No `pull_request_target` usage. No secrets logged. | ✅ PASS |

**Post-design re-check**: All gates pass. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/001-semver-cicd-pipelines/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── semver-release-workflow.md
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
.github/
├── workflows/
│   ├── semver-release.yml          # Reusable workflow (workflow_call) — primary deliverable
│   └── test-semver-release.yml     # CI test workflow (runs on every PR to main)
└── actions/                        # Reserved for future composite actions

docs/
└── semver-release/
    └── README.md                   # Consumer-facing docs: inputs, outputs, usage examples
```

**Structure Decision**: GitHub Actions reusable workflows live under
`.github/workflows/` per platform convention. A `docs/` directory at repo root
holds consumer-facing documentation per Constitution Principle I (README required).
No `src/` or language-specific directories are needed — this is a YAML-only
deliverable.

## Complexity Tracking

> No constitution violations. Table not required.
