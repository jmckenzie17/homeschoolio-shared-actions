# Research: Semantic Versioning CI/CD Pipelines

**Feature**: 001-semver-cicd-pipelines
**Date**: 2026-03-26 (updated for semantic-release engine swap)

---

## Decision 1: Version Bump & Release Engine

**Decision**: Use `cycjimmy/semantic-release-action` (SHA: `b12c8f6015dc215fe37bc154d4ad456dd3833c90`, v6.0.0) as the version calculation and release creation engine.

**Rationale**:
- Natively supports the full Conventional Commits specification: `fix:` → PATCH, `feat:` → MINOR, `feat!:` / `BREAKING CHANGE:` footer → MAJOR.
- Non-release prefixes (`chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:`) accumulate without bumping — satisfying FR-007.
- Publishes a Git tag and GitHub Release **directly** on push to `main` (no Release PR intermediary) — satisfies FR-001, FR-002, FR-003.
- No prior version tags → defaults to `v1.0.0` on first release (FR-008).
- Requires only `contents: write` — no `pull-requests: write` needed (simpler permission model than release-please).

**Pinnable SHA**: `b12c8f6015dc215fe37bc154d4ad456dd3833c90` (v6.0.0, released 2025-11-17). This SHA MUST be used in all `uses:` references per Constitution Principle III.

**Known limitation — CVE-2025-5889**: Low-severity transitive vulnerability in `brace-expansion` (CVSS 3.1). Present in v6 via semantic-release v25's dependency chain. Unfixable in the action itself. Accepted: low severity, transitive only, no direct exposure in CI context.

**Alternatives considered**:
- `googleapis/release-please-action` — previously used; rejected because it requires a "Release PR" model that adds latency and requires `pull-requests: write` permission plus a repo-level "Allow GitHub Actions to create PRs" setting. These were live failure points. semantic-release's direct-publish model is simpler.
- `cycjimmy/semantic-release-action` was previously rejected (Decision 1 original) due to lower adoption (681 stars vs 2,342) and CVE-2025-5889. Re-evaluated at user direction; CVE is low severity/transitive and acceptable.

---

## Decision 2: Moving Major Version Pointer Mechanism

**Decision**: Use a plain shell `run:` step with native `git tag -fa` and `git push origin vN --force` commands — no additional action dependency.

**Rationale**:
- Official GitHub guidance documents this as the standard pattern for Actions repositories.
- A `run:` step has zero external supply-chain dependency.
- `contents: write` permission is sufficient.
- Logic is trivial — does not warrant an external action (Constitution Principle IV).

**Required git configuration before tagging**:
```bash
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
```

**Tag extraction pattern** (semantic-release outputs `new_release_git_tag`, e.g., `v1.2.3`):
```bash
TAG=${{ steps.release.outputs.new_release_git_tag }}  # "v1.2.3"
MAJOR=${TAG%%.*}                                        # "v1"
git tag -fa "${MAJOR}" -m "Update ${MAJOR} tag to ${TAG}"
git push origin "${MAJOR}" --force
```

**Known gotcha — tag protection rulesets**: If a repo has Rulesets matching `v*`, `GITHUB_TOKEN` cannot force-push the major pointer. Workaround: GitHub App token via `actions/create-github-app-token`. Documented in README.

**Known gotcha — force-push does NOT trigger downstream workflows**: Intentional GitHub behavior to prevent loops.

---

## Decision 3: Reusable Workflow Structure (`workflow_call`)

**Decision**: Delivered as a reusable workflow at `.github/workflows/semver-release.yml` with `on: workflow_call:` trigger. Not a composite action.

**Rationale**:
- `workflow_call` is the correct GitHub construct for sharing multi-job pipelines.
- Consumer repos reference it with a single `uses:` line (FR-004).

**Workflow inputs defined**:
| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `release-branch` | string | `main` | Branch to base releases on |
| `tag-prefix` | string | `v` | Prefix for version tags |

**Note**: `config-file` input from the release-please design is removed — semantic-release discovers `.releaserc.json` automatically by convention; no explicit path input is needed.

**Workflow outputs defined**:
| Output | Type | Description |
|--------|------|-------------|
| `release-created` | string (`"true"`/`"false"`) | Whether a release was published this run |
| `tag-name` | string | The full version tag created (e.g., `v1.2.3`) |
| `major-tag` | string | The major pointer tag updated (e.g., `v1`) |

---

## Decision 4: Third-Party Actions — SHA Pinning

Per Constitution Principle III, all external actions MUST be pinned to commit SHAs.

| Action | Purpose | Pinned SHA | Version |
|--------|---------|-----------|---------|
| `cycjimmy/semantic-release-action` | Version calc + tag + release | `b12c8f6015dc215fe37bc154d4ad456dd3833c90` | v6.0.0 |
| `actions/checkout` | Repo checkout | `34e114876b0b11c390a56381ad16ebd13914f8d5` | v4.3.1 |
| `rhysd/actionlint` | Workflow YAML lint in CI test | `393031adb9afb225ee52ae2ccd7a5af5525e03e8` | v1.7.11 |

**Output key names for `cycjimmy/semantic-release-action` v6.0.0**:
- `new_release_published` — `"true"` if a release was created; use this for conditionals
- `new_release_version` — version without prefix, e.g., `1.2.3`
- `new_release_git_tag` — full tag with prefix, e.g., `v1.2.3`
- `new_release_major_version` — major version number only, e.g., `1`

---

## Decision 5: CI Test Workflow

**Decision**: A test workflow at `.github/workflows/test-semver-release.yml` runs on every PR to `main` with:
1. **Lint/validate**: `actionlint` to verify the workflow YAML is syntactically valid.
2. **Dry-run**: `semantic-release --dry-run` to verify commit parsing and version calculation without creating tags.
3. **SHA-pin check**: `grep` to assert no mutable `@tag` references remain.

**Note**: `semantic-release --dry-run` requires `GITHUB_TOKEN` to authenticate with GitHub API even in dry-run mode.

---

## Decision 6: Permissions Model

**Critical finding**: semantic-release does NOT create a Release PR. It publishes directly. Therefore:

- `pull-requests: write` is **not required** (simplification over release-please).
- Only `contents: write` is needed: create/push tags, create GitHub Releases.

**In `semver-release.yml`** (reusable — documents and narrows):
```yaml
permissions:
  contents: write
```

**In `release.yml`** (self-release caller — grants the ceiling):
```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: ./.github/workflows/semver-release.yml
```

Consumer workflows calling via `@v1` must also declare `contents: write` on the calling job.

---

## Decision 7: Self-Release Caller Workflow & Concurrency

**Decision**: This repo versions itself via `.github/workflows/release.yml`, a minimal caller workflow referencing `semver-release.yml` via local path.

**Concurrency**: `group: release`, `cancel-in-progress: false` — last-pending-wins. Acceptable: semantic-release is idempotent.

---

## Decision 8: Zero-Config Adoption via Inline Plugin Config (FR-014)

**Decision**: The reusable workflow configures semantic-release plugins inline via the `extra_plugins` and `branches` inputs of `cycjimmy/semantic-release-action`. No `.releaserc.json` is required for standard single-package repos.

**Inline config**:
```yaml
- uses: cycjimmy/semantic-release-action@b12c8f6015dc215fe37bc154d4ad456dd3833c90
  with:
    semantic_version: 25
    extra_plugins: |
      @semantic-release/commit-analyzer
      @semantic-release/release-notes-generator
      @semantic-release/github
    branches: |
      ['${{ inputs.release-branch }}']
    tag_format: '${{ inputs.tag-prefix }}${version}'
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Consumer override**: If a `.releaserc.json` exists at the repo root, semantic-release picks it up automatically and the inline `with:` configuration is overridden. No explicit `config-file` input is needed — this is semantic-release's standard convention-based discovery.

**Critical finding**: `extra_plugins` only *installs* additional plugins — it does NOT replace semantic-release's default plugin set, which includes `@semantic-release/npm`. On non-Node.js repos with no `package.json`, this causes a hard failure: `SemanticReleaseError: Missing package.json file`. The fix: write a `.releaserc-default.json` at runtime in a `run:` step before the action executes, specifying only the three desired plugins. Semantic-release discovers this file automatically. Consumer repos with their own `.releaserc.json` (or `.releaserc.js`, `release.config.js`, etc.) take precedence automatically — the default file is only written when no consumer config exists.

**Limitation**: The built-in default does not support per-plugin options (e.g., custom changelog presets). Consumers who need advanced configuration must commit their own `.releaserc.json`.

**`fetch-depth: 0` required**: semantic-release reads git tags to determine prior releases. Shallow clones (default `fetch-depth: 1`) may cause incorrect version detection. `fetch-depth: 0` MUST be set on the checkout step.
