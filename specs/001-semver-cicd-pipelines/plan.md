# Implementation Plan: Semantic Versioning CI/CD Pipelines

**Branch**: `001-semver-cicd-pipelines` | **Date**: 2026-03-26 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-semver-cicd-pipelines/spec.md`

## Summary

Deliver a reusable GitHub Actions workflow (`semver-release.yml`) that automates
semantic versioning on every push to `main` using `cycjimmy/semantic-release-action`
v6.0.0 (SHA-pinned). The workflow embeds inline default plugin configuration so
consumers need zero setup files; a consumer-provided `.releaserc.json` overrides
the inline defaults automatically by convention. This repo versions itself via a
local caller workflow (`release.yml`). Consumers adopt with a single `uses:` line.

## Technical Context

**Language/Version**: YAML (GitHub Actions workflow syntax) — no application runtime
**Primary Dependencies**:
- `cycjimmy/semantic-release-action` @ `b12c8f6015dc215fe37bc154d4ad456dd3833c90` (v6.0.0)
- `actions/checkout` @ `34e114876b0b11c390a56381ad16ebd13914f8d5` (v4.3.1)
- `rhysd/actionlint` @ `393031adb9afb225ee52ae2ccd7a5af5525e03e8` (v1.7.11) — CI only

**Storage**: N/A (Git tags and GitHub Releases are the persistent artifacts)
**Testing**: `actionlint` (YAML lint) + `semantic-release --dry-run` (functional smoke test)
**Target Platform**: GitHub Actions (ubuntu-latest runners)
**Project Type**: Reusable GitHub Actions workflow (shared-actions library)
**Performance Goals**: End-to-end PR merge → published GitHub Release under 3 minutes (SC-003)
**Constraints**: `contents: write` only (no `pull-requests: write` needed); `GITHUB_TOKEN` exclusively (FR-010); all external actions SHA-pinned (FR-009, Constitution Principle III)
**Scale/Scope**: Ecosystem-wide (all homeschoolio repos); no volume constraints

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked post-design below.*

| Principle | Gate | Status | Notes |
|-----------|------|--------|-------|
| I. Reusable-Action-First | Deliverable MUST be a reusable workflow or action | ✅ PASS | `semver-release.yml` is a `workflow_call` reusable workflow |
| II. Semantic Versioning | Repo MUST follow SemVer; major pointer tags MUST be maintained | ✅ PASS | FR-006 and User Story 3 address this; self-release via `release.yml` bootstraps it |
| III. SHA Pinning | All external actions MUST be commit-SHA pinned | ✅ PASS | All SHAs confirmed and recorded in research.md Decision 4 |
| IV. Minimal Surface Area | Inputs limited to what's needed; no speculative params | ✅ PASS | 2 inputs (`release-branch`, `tag-prefix`); `config-file` removed — semantic-release uses convention-based discovery |
| V. Tested Before Release | Test workflow MUST cover happy path + one error path on every PR | ✅ PASS | `test-semver-release.yml` runs `actionlint` + `semantic-release --dry-run` |
| Security | No `pull_request_target`; no secret logging; least privilege | ✅ PASS | `push` trigger only; `contents: write` only; `GITHUB_TOKEN` only |

**Post-design re-check**: All gates pass. `pull-requests: write` removed from required permissions — simplification over previous release-please design.

## Project Structure

### Documentation (this feature)

```text
specs/001-semver-cicd-pipelines/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions on semantic-release, SHA pinning, permissions model, inline config
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

docs/
└── semver-release/
    ├── README.md                   # Consumer-facing docs (inputs, outputs, permissions, prerequisites, examples)
    └── examples/
        ├── consumer-workflow.yml           # Minimal consumer example (≤20 lines)
        └── consumer-workflow-with-deploy.yml  # Example chaining release → deploy
```

**Structure Decision**: Single flat `.github/workflows/` layout. Three workflow files
serve distinct roles: reusable contract, self-release caller, CI test harness. No
`.release-please-config.json` or `.release-please-manifest.json` needed — semantic-release
derives version from git tags and uses inline plugin config by default.

## Phase 0: Research Summary

All research complete. Key decisions documented in [research.md](research.md).

### Decision Log

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Use `cycjimmy/semantic-release-action` v6.0.0 (SHA: `b12c8f6...`) | Direct-publish model; no Release PR; only `contents: write` needed; user direction |
| 2 | Major pointer update via inline `git tag -fa` shell step | No external action dependency; official GitHub guidance pattern |
| 3 | Reusable workflow via `workflow_call`; not composite action | Multi-job pipeline needs `workflow_call` |
| 4 | SHA-pin all external actions (confirmed SHAs in research.md) | Constitution Principle III |
| 5 | Test via `actionlint` + `semantic-release --dry-run` | Can't fully E2E test without real merge; dry-run verifies parsing; lint catches YAML errors |
| 6 | Permissions: `contents: write` only | semantic-release does not create a Release PR; `pull-requests: write` not needed |
| 7 | Self-release via `release.yml` with local path `uses: ./.github/workflows/semver-release.yml` | Avoids bootstrap chicken-and-egg |
| 8 | Concurrency: `group: release`, `cancel-in-progress: false` | Serializes runs; last-pending-wins acceptable; semantic-release is idempotent |
| 9 | Zero-config via inline `extra_plugins` + `branches` inputs (FR-014) | Eliminates consumer setup friction; `.releaserc.json` at repo root overrides automatically by convention |

### Critical Findings (affect implementation)

1. **`fetch-depth: 0` required**: semantic-release reads git tags to find prior releases.
   Shallow clone (default) may cause incorrect version detection.

2. **Output key names**: Use `new_release_published` (not `release_created`), `new_release_git_tag`
   (not `tag_name`), `new_release_major_version` (not `major`).

3. **No Release PR model**: semantic-release publishes directly on push to `main`.
   No `pull-requests: write` permission needed. No repo-level "Allow Actions to create PRs" setting required.

4. **Inline config limitation**: `extra_plugins` installs plugins but does not support per-plugin options.
   Consumers needing custom changelog sections or non-standard presets must commit `.releaserc.json`.

5. **`tag_format` input**: Use `tag_format: '${{ inputs.tag-prefix }}${version}'` to honor
   the `tag-prefix` input. Default is `v${version}`.

6. **CVE-2025-5889**: Low severity, transitive, accepted. No direct exposure in CI context.

## Phase 1: Design

### Workflow Architecture

```text
On push to main
  └── release.yml (self-release caller)
        ├── concurrency: group=release, cancel-in-progress=false
        ├── permissions: contents=write
        └── job: release
              └── uses: ./.github/workflows/semver-release.yml
                    ├── Step: checkout (fetch-depth: 0)
                    ├── Step: Run semantic-release-action@<SHA>
                    │     ├── extra_plugins: commit-analyzer, release-notes-generator, github
                    │     ├── branches: ['<inputs.release-branch>']
                    │     ├── tag_format: '<inputs.tag-prefix>${version}'
                    │     └── outputs: new_release_published, new_release_git_tag
                    └── Step: [if new_release_published == 'true'] Update major pointer
                          └── git tag -fa vN && git push --force

On PR to main
  └── test-semver-release.yml
        ├── job: validate (actionlint on semver-release.yml)
        ├── job: dry-run (semantic-release --dry-run)
        └── job: sha-pinning (grep for mutable @tag references)
```

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
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

No config files required. For advanced config, add `.releaserc.json` to repo root.

## Complexity Tracking

No constitution violations. No complexity justification required.
