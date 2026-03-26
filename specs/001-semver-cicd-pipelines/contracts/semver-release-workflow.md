# Contract: `semver-release.yml` Reusable Workflow

**Feature**: 001-semver-cicd-pipelines
**Date**: 2026-03-26
**Type**: GitHub Actions `workflow_call` contract

This document defines the complete public interface of the `semver-release.yml`
reusable workflow. This is the contract that all consumer repositories depend on.
Changes to this interface require a MAJOR version bump per the constitution.

---

## Trigger

```yaml
on:
  workflow_call:
    inputs: ...    # see Inputs section
    outputs: ...   # see Outputs section
```

Consumer invocation:
```yaml
jobs:
  release:
    uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

---

## Inputs

All inputs are optional. Defaults are designed so that a consumer with a standard
single-package repo targeting `main` needs zero configuration.

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `release-branch` | `string` | no | `"main"` | The branch name that triggers a release evaluation when a push occurs. Consumer repos using a non-`main` default branch (e.g., `master`) MUST set this. |
| `tag-prefix` | `string` | no | `"v"` | Prefix prepended to the SemVer number. Produces tags like `v1.2.3`. Must be a non-empty string. |

### Input Validation Rules

- `release-branch`: MUST be a valid Git branch name. Empty string is not accepted.
- `tag-prefix`: MUST be a non-empty string. Changing this value in an existing repo
  will cause semantic-release to treat the repo as having no prior releases (existing
  tags with the old prefix are not found). Treat as immutable once set.

---

## Outputs

Outputs are available to calling workflows via
`${{ needs.<job-id>.outputs.<output-name> }}`.

| Output | Type | Description | Empty when |
|--------|------|-------------|------------|
| `release-created` | `string` (`"true"` / `"false"`) | Whether a new release was published in this run. Use `== 'true'` for conditionals (not `== true` — it is a string). | No release was needed (non-release commits only) |
| `tag-name` | `string` | The full version tag created (e.g., `v1.2.3`). | `release-created == 'false'` |
| `major-tag` | `string` | The major pointer tag updated (e.g., `v1`). | `release-created == 'false'` |

### Output Usage Example

```yaml
jobs:
  release:
    uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit

  deploy:
    needs: release
    if: needs.release.outputs.release-created == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.release.outputs.tag-name }}"
```

---

## Required Permissions

**Important**: In `workflow_call`, the `GITHUB_TOKEN` scope is controlled by the
**caller**. The reusable workflow can only narrow what it receives — it cannot
self-elevate. **Calling workflows MUST declare these permissions on the calling job:**

```yaml
jobs:
  release:
    permissions:
      contents: write       # create/push tags, create GitHub Releases
    uses: owner/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

The reusable workflow also declares `permissions: contents: write` internally (to
document requirements and narrow scope if the caller grants more). Without the
caller declaring this permission, the workflow will fail with a 403 on tag push or
Release creation.

**Note**: `pull-requests: write` is NOT required — semantic-release publishes
directly without creating a Release PR.

For repos with Rulesets that block `github-actions[bot]` from creating tags, a
GitHub App token must be passed via secrets — see the README for the workaround
pattern.

---

## Secrets

The workflow uses `secrets: inherit` in the consumer call, which forwards the
consumer's `GITHUB_TOKEN` automatically. No additional secrets are required for
standard operation.

For advanced cases (protected branches, Rulesets), an override token may be passed:

| Secret | Required | Description |
|--------|----------|-------------|
| `RELEASE_TOKEN` | no | Override token (PAT or GitHub App token) when `GITHUB_TOKEN` lacks sufficient permissions to push to a protected branch or bypass tag Rulesets. |

---

## Behavior Contract

### When a release IS created

1. A push to `release-branch` containing at least one releasable commit triggers semantic-release.
2. A new Version Tag (`{tag-prefix}{MAJOR}.{MINOR}.{PATCH}`) is pushed to that commit.
3. A GitHub Release is published with an auto-generated changelog body.
4. The Major Pointer Tag (`{tag-prefix}{MAJOR}`) is force-updated to the same commit.
5. Outputs: `release-created=true`, `tag-name=v1.2.3`, `major-tag=v1`.

### When NO release is created

1. Only non-release commits exist since the last tag (e.g., `chore:`, `docs:`).
2. No tag is created. No GitHub Release is published.
3. Outputs: `release-created=false`, `tag-name=""`, `major-tag=""`.

### Version bump rules

| Commit prefix | Bump type |
|---------------|-----------|
| `fix:`, `perf:` | PATCH |
| `feat:` | MINOR |
| `feat!:`, any commit with `BREAKING CHANGE:` footer | MAJOR |
| `chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:` | No release |

When multiple bump types exist in one release cycle, the highest wins (MAJOR >
MINOR > PATCH).

### First release (no prior tags)

When no version tags matching `{tag-prefix}*` exist, the first release produces
`{tag-prefix}1.0.0` regardless of commit type (PATCH, MINOR, or MAJOR all start
at `1.0.0`).

---

## Breaking Change Policy

Any of the following changes to this contract require a MAJOR version bump of this
repo and a deprecation notice in the release notes:

- Removing or renaming an input.
- Changing the default value of an input in a way that alters behavior for existing consumers.
- Removing or renaming an output.
- Changing the type of an output.
- Changing the required permissions to a broader set.

The following changes are backward-compatible (MINOR or PATCH):

- Adding a new optional input with a sensible default.
- Adding a new output.
- Narrowing the required permissions.
- Updating the pinned SHA of an internal dependency (patch fix).

---

## CI Test Contract (`test-semver-release.yml`)

The test workflow validates this contract on every PR to `main`:

| Test | Method | Pass Criteria |
|------|--------|---------------|
| Workflow YAML is valid | `actionlint` | Zero errors or warnings |
| Inputs are correctly typed | `actionlint` | No type mismatches |
| Dry-run produces expected version | `semantic-release --dry-run` | Exit 0, no tags pushed |
| No-release case exits cleanly | Commits with `chore:` only | `release-created == 'false'` |
| No mutable tag references | `grep` for `@v*`/`@main` patterns | Zero matches |
