# Implementation Plan: Semantic Versioning CI/CD Pipelines

**Branch**: `001-semver-cicd-pipelines` | **Date**: 2026-03-26 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-semver-cicd-pipelines/spec.md`

## Summary

Deliver a reusable GitHub Actions workflow (`semver-release.yml`) that automates
semantic versioning on every merge to `main` using `googleapis/release-please-action`
(SHA-pinned). This repo versions itself via a local caller workflow (`release.yml`).
Consumers in the homeschoolio ecosystem adopt it with a single `uses:` line.

## Technical Context

**Language/Version**: YAML (GitHub Actions workflow syntax) — no application runtime
**Primary Dependencies**:
- `googleapis/release-please-action` @ `16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0)
- `actions/checkout` @ SHA to be verified at implementation time (v4.x)
- `rhysd/actionlint` @ SHA to be verified at implementation time (v1.7.x) — CI only

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
| III. SHA Pinning | All external actions MUST be commit-SHA pinned | ✅ PASS | FR-009; `release-please-action` SHA documented in research.md Decision 1 |
| IV. Minimal Surface Area | Inputs limited to what's needed; no speculative params | ✅ PASS | 3 inputs (`release-branch`, `tag-prefix`, `config-file`); major pointer step is inline shell |
| V. Tested Before Release | Test workflow MUST cover happy path + one error path on every PR | ✅ PASS | `test-semver-release.yml` runs `actionlint` + dry-run |
| Security | No `pull_request_target`; no secret logging; least privilege | ✅ PASS | `push` trigger only; permissions declared explicitly; no dynamic secret handling |

**Post-design re-check**: All gates still pass. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/001-semver-cicd-pipelines/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions on release-please, SHA pinning, permissions model
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
    ├── release.yml                 # Self-release caller (this repo versions itself)
    └── test-semver-release.yml     # CI test workflow (runs on every PR to main)

docs/
└── semver-release/
    ├── README.md                   # Consumer-facing docs (inputs, outputs, permissions, examples)
    └── examples/
        ├── consumer-workflow.yml           # Minimal consumer example
        └── consumer-workflow-with-deploy.yml  # Example chaining release → deploy
```

**Structure Decision**: Single flat `.github/workflows/` layout — no subdirectories.
Three workflow files serve distinct roles: reusable contract, self-release caller,
and CI test harness. Documentation lives in `docs/semver-release/` per constitution
Principle I (each action accompanied by README).

## Phase 0: Research Summary

All research complete. Key decisions documented in [research.md](research.md).

### Decision Log

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Use `googleapis/release-please-action` v4.4.0 (SHA: `16a9c90...`) | Native conventional commits support; creates tag + release in one run; no PAT required |
| 2 | Major pointer update via inline `git tag -fa` shell step | No external action dependency; official GitHub guidance pattern; `contents: write` sufficient |
| 3 | Reusable workflow via `workflow_call`; not composite action | Multi-job pipeline needs `workflow_call`; composite actions are for single-step logic |
| 4 | SHA-pin all external actions; document in research.md | Constitution Principle III; supply-chain attack mitigation |
| 5 | Test via `actionlint` + `release-please --dry-run` | Can't fully E2E test without real merge; dry-run verifies parsing; lint catches YAML errors |
| 6 | Permissions: caller declares `contents: write` + `pull-requests: write` | `workflow_call` callers control the `GITHUB_TOKEN` ceiling; reusable workflow cannot self-elevate |
| 7 | Self-release via `release.yml` with local path `uses: ./.github/workflows/semver-release.yml` | Avoids bootstrap chicken-and-egg; local path resolves from same commit; supported since Jan 2022 |
| 8 | Concurrency: `group: release`, `cancel-in-progress: false` | Serializes runs; "last pending wins" is acceptable since release-please is idempotent over accumulated commits |

### Critical Findings (affect implementation)

1. **Permissions are caller-controlled**: `release.yml` and consumer workflows MUST
   declare `permissions: contents: write` + `pull-requests: write` on the calling
   job. This must be documented prominently in the README.

2. **`release_created` not `releases_created`**: release-please-action v4 has a
   known bug where `releases_created` (plural) always returns `true`. Conditionals
   gating downstream jobs MUST use `release_created` (singular).

3. **Concurrency is "last pending wins"**: `cancel-in-progress: false` does not
   guarantee every run executes — only one pending slot exists. Acceptable because
   release-please re-reads all commits from the latest tag on each run.

4. **`pull-requests: write` required**: release-please creates and updates a
   "Release PR" that aggregates changes before publishing a release. This permission
   is non-negotiable for the tool to function.

## Phase 1: Design

### Workflow Architecture

```text
On push to main
  └── release.yml (self-release caller)
        ├── concurrency: group=release, cancel-in-progress=false
        ├── permissions: contents=write, pull-requests=write
        └── job: release
              └── uses: ./.github/workflows/semver-release.yml
                    ├── Job: release-please
                    │     ├── googleapis/release-please-action@<SHA>
                    │     │     outputs: release_created, tag_name, major
                    │     └── [if release_created] Update major pointer
                    │           ├── actions/checkout@<SHA>
                    │           └── run: git tag -fa vN && git push --force
                    └── outputs: release-created, tag-name, major-tag

On PR to main
  └── test-semver-release.yml
        ├── job: validate (actionlint)
        ├── job: dry-run (release-please --dry-run, --repo-url=${{ github.repository }})
        └── job: sha-pinning (grep for mutable @tag references)
```

### Key Implementation Details

#### `semver-release.yml` structure

```yaml
on:
  workflow_call:
    inputs:
      release-branch:  { type: string, default: 'main' }
      tag-prefix:      { type: string, default: 'v' }
      config-file:     { type: string, default: '.release-please-config.json' }
    outputs:
      release-created: { value: ${{ jobs.release-please.outputs.release-created }} }
      tag-name:        { value: ${{ jobs.release-please.outputs.tag-name }} }
      major-tag:       { value: ${{ jobs.release-please.outputs.major-tag }} }

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    outputs:
      release-created: ...
      tag-name: ...
      major-tag: ...
    steps:
      - uses: googleapis/release-please-action@16a9c90856f42705d54a6fda1823352bdc62cf38
        id: release
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: simple
          target-branch: ${{ inputs.release-branch }}
          tag-name-format: "${{ inputs.tag-prefix }}${version}"
          config-file: ${{ inputs.config-file }}

      # Only runs when a release was created
      - uses: actions/checkout@<SHA>
        if: steps.release.outputs.release_created == 'true'
        with:
          fetch-depth: 0

      - name: Update major version pointer
        if: steps.release.outputs.release_created == 'true'
        env:
          TAG: ${{ steps.release.outputs.tag_name }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          MAJOR=${TAG%%.*}
          git tag -fa "${MAJOR}" -m "Update ${MAJOR} to ${TAG}"
          git push origin "${MAJOR}" --force
```

#### `release.yml` structure

```yaml
name: Release

on:
  push:
    branches: [main]

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release:
    permissions:
      contents: write
      pull-requests: write
    uses: ./.github/workflows/semver-release.yml
```

#### `test-semver-release.yml` key fix

The existing `dry-run` job must use `--repo-url=${{ github.repository }}` (not
hardcoded `homeschoolio/homeschoolio-shared-actions`). **This fix is already applied**
to the current branch.

### Consumer Adoption Pattern

Minimum viable consumer workflow (≤ 20 lines, satisfying SC-001):

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

Plus a `.release-please-config.json` in the consumer repo root:
```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "simple",
  "packages": { ".": {} }
}
```

### Open Items for Implementation

- [ ] Verify and pin current SHA for `actions/checkout` v4 at implementation time
- [ ] Verify and pin current SHA for `rhysd/actionlint` at implementation time
- [ ] Confirm `tag-name-format` input name in release-please-action v4.4.0 (may differ — check action's action.yml)
- [ ] Confirm exact output name for tag in release-please-action v4.4.0 (`tag_name` vs `tag-name`)

## Complexity Tracking

No constitution violations. No complexity justification required.
