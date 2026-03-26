# Implementation Plan: Semantic Versioning CI/CD Pipelines

**Branch**: `001-semver-cicd-pipelines` | **Date**: 2026-03-26 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-semver-cicd-pipelines/spec.md`

## Summary

Deliver a reusable GitHub Actions workflow (`semver-release.yml`) that automates
semantic versioning on every merge to `main` using `googleapis/release-please-action`
(SHA-pinned). The workflow embeds a built-in default release-please config so
consumers need zero setup files; a consumer-provided config overrides the default.
This repo versions itself via a local caller workflow (`release.yml`). Consumers
adopt it with a single `uses:` line.

## Technical Context

**Language/Version**: YAML (GitHub Actions workflow syntax) — no application runtime
**Primary Dependencies**:
- `googleapis/release-please-action` @ `16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0)
- `actions/checkout` @ `34e114876b0b11c390a56381ad16ebd13914f8d5` (v4.3.1)
- `rhysd/actionlint` @ `393031adb9afb225ee52ae2ccd7a5af5525e03e8` (v1.7.11) — CI only

**Storage**: N/A (Git tags and GitHub Releases are the persistent artifacts)
**Testing**: `actionlint` (YAML lint) + `release-please --dry-run` (functional smoke test)
**Target Platform**: GitHub Actions (ubuntu-latest runners)
**Project Type**: Reusable GitHub Actions workflow (shared-actions library)
**Performance Goals**: End-to-end PR merge → published GitHub Release under 3 minutes (SC-003)
**Constraints**: `contents: write` + `pull-requests: write` only; `GITHUB_TOKEN` exclusively (FR-010); all external actions SHA-pinned (FR-009, Constitution Principle III)
**Scale/Scope**: Ecosystem-wide (all homeschoolio repos); no volume constraints

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked post-design below.*

| Principle | Gate | Status | Notes |
|-----------|------|--------|-------|
| I. Reusable-Action-First | Deliverable MUST be a reusable workflow or action | ✅ PASS | `semver-release.yml` is a `workflow_call` reusable workflow |
| II. Semantic Versioning | Repo MUST follow SemVer; major pointer tags MUST be maintained | ✅ PASS | FR-006 and User Story 3 address this; self-release via `release.yml` bootstraps it |
| III. SHA Pinning | All external actions MUST be commit-SHA pinned | ✅ PASS | FR-009; all SHAs confirmed and recorded in research.md Decision 4 |
| IV. Minimal Surface Area | Inputs limited to what's needed; no speculative params | ✅ PASS | 3 inputs (`release-branch`, `tag-prefix`, `config-file`); config resolution is inline shell |
| V. Tested Before Release | Test workflow MUST cover happy path + one error path on every PR | ✅ PASS | `test-semver-release.yml` runs `actionlint` + dry-run |
| Security | No `pull_request_target`; no secret logging; least privilege | ✅ PASS | `push` trigger only; permissions declared explicitly; no dynamic secret handling |

**Post-design re-check**: All gates still pass. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/001-semver-cicd-pipelines/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions on release-please, SHA pinning, permissions model, config resolution
├── data-model.md        # Phase 1 — workflow entities, inputs/outputs, relationships
├── quickstart.md        # Phase 1 — end-to-end validation scenarios
├── contracts/
│   └── semver-release-workflow.md   # Phase 1 — public interface contract
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
.github/
└── workflows/
    ├── semver-release.yml          # Reusable workflow (workflow_call) — primary deliverable
    ├── release.yml                 # Self-release caller (this repo versions itself via local path)
    └── test-semver-release.yml     # CI test workflow (runs on every PR to main)

.release-please-config.json         # This repo's own release-please config (overrides built-in default)
.release-please-manifest.json       # This repo's version manifest (managed by release-please)

docs/
└── semver-release/
    ├── README.md                   # Consumer-facing docs (inputs, outputs, permissions, prerequisites, examples)
    └── examples/
        ├── consumer-workflow.yml           # Minimal consumer example (≤20 lines)
        └── consumer-workflow-with-deploy.yml  # Example chaining release → deploy
```

**Structure Decision**: Single flat `.github/workflows/` layout. Three workflow files
serve distinct roles: reusable contract, self-release caller, CI test harness. This
repo carries its own `.release-please-config.json` which serves as both a working
example and the override for the built-in default.

## Phase 0: Research Summary

All research complete. Key decisions documented in [research.md](research.md).

### Decision Log

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Use `googleapis/release-please-action` v4.4.0 (SHA: `16a9c90...`) | Native conventional commits support; creates tag + release in one run; no PAT required |
| 2 | Major pointer update via inline `git tag -fa` shell step | No external action dependency; official GitHub guidance pattern; `contents: write` sufficient |
| 3 | Reusable workflow via `workflow_call`; not composite action | Multi-job pipeline needs `workflow_call`; composite actions are for single-step logic |
| 4 | SHA-pin all external actions (confirmed SHAs in research.md) | Constitution Principle III; supply-chain attack mitigation |
| 5 | Test via `actionlint` + `release-please --dry-run` | Can't fully E2E test without real merge; dry-run verifies parsing; lint catches YAML errors |
| 6 | Permissions: caller declares `contents: write` + `pull-requests: write` | `workflow_call` callers control the `GITHUB_TOKEN` ceiling; reusable workflow cannot self-elevate |
| 7 | Self-release via `release.yml` with local path `uses: ./.github/workflows/semver-release.yml` | Avoids bootstrap chicken-and-egg; local path resolves from same commit |
| 8 | Concurrency: `group: release`, `cancel-in-progress: false` | Serializes runs; last-pending-wins is acceptable since release-please is idempotent |
| 9 | Built-in default config with consumer override (FR-014) | Eliminates zero-config prerequisite friction; `release-type` inline and config-file mode are mutually exclusive in v4 — write default to temp file, substitute if consumer file exists |

### Critical Findings (affect implementation)

1. **Permissions are caller-controlled**: `release.yml` and consumer workflows MUST
   declare `permissions: contents: write` + `pull-requests: write` on the calling
   job. Documented prominently in README.

2. **Repo setting required**: "Allow GitHub Actions to create and approve pull
   requests" must be enabled in Settings → Actions → General. Documented in README
   Prerequisites. Without it: `GitHub Actions is not permitted to create or approve
   pull requests.`

3. **`release_created` not `releases_created`**: release-please-action v4 bug —
   `releases_created` (plural) always returns `true`. Use `release_created`
   (singular) in all conditionals.

4. **`release-type` inline and `config-file` are mutually exclusive**: When
   `release-type` is set in `with:`, the config file is completely ignored. To
   support both default config and consumer override, the workflow writes a default
   config to a temp file, checks if the consumer has their own file, and passes
   whichever exists to the action via `config-file` (no `release-type` in `with:`).

5. **`pull-requests: write` required**: release-please creates and updates a
   "Release PR" that aggregates changes before publishing a release.

## Phase 1: Design

### Workflow Architecture

```text
On push to main
  └── release.yml (self-release caller)
        ├── concurrency: group=release, cancel-in-progress=false
        ├── permissions: contents=write, pull-requests=write
        └── job: release
              └── uses: ./.github/workflows/semver-release.yml
                    ├── Step: checkout (fetch-depth: 0)
                    ├── Step: Resolve release-please config
                    │     └── writes default config to temp file
                    │         checks if consumer's config-file input exists
                    │         outputs: config path to use
                    ├── Step: Run release-please-action@<SHA>
                    │     └── config-file: steps.config.outputs.config
                    │         outputs: release_created, tag_name
                    └── Step: [if release_created] Update major pointer
                          └── git tag -fa vN && git push --force

On PR to main
  └── test-semver-release.yml
        ├── job: validate (actionlint on semver-release.yml)
        ├── job: dry-run (release-please --dry-run --repo-url=${{ github.repository }})
        └── job: sha-pinning (grep for mutable @tag references)
```

### Config Resolution Logic

```yaml
- name: Resolve release-please config
  id: config
  env:
    CONFIG_FILE: ${{ inputs.config-file }}
  run: |
    DEFAULT_CONFIG='.release-please-config-default.json'
    cat > "${DEFAULT_CONFIG}" <<'EOF'
    {
      "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
      "release-type": "simple",
      "packages": { ".": {} }
    }
    EOF
    if [ -f "${CONFIG_FILE}" ]; then
      echo "config=${CONFIG_FILE}" >> "${GITHUB_OUTPUT}"
    else
      echo "config=${DEFAULT_CONFIG}" >> "${GITHUB_OUTPUT}"
    fi
```

The action then uses `config-file: ${{ steps.config.outputs.config }}` with no
`release-type` in `with:` (file-based mode, not inline mode).

### Consumer Adoption — Zero Config

Minimum viable consumer workflow (≤ 20 lines, SC-001 satisfied):

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    permissions:
      contents: write
      pull-requests: write
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

**One-time repo setup**: Settings → Actions → General → enable "Allow GitHub Actions
to create and approve pull requests".

No config files required. The built-in default (`release-type: simple`, root package)
handles standard single-package repos. For advanced config, add
`.release-please-config.json` to the repo root.

## Complexity Tracking

No constitution violations. No complexity justification required.
