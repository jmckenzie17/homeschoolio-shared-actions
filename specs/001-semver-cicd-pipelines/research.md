# Research: Semantic Versioning CI/CD Pipelines

**Feature**: 001-semver-cicd-pipelines
**Date**: 2026-03-26

---

## Decision 1: Version Bump & Release Engine

**Decision**: Use `googleapis/release-please-action` (the active maintained fork,
NOT the archived `google-github-actions/release-please-action`) as the version
calculation and release creation engine.

**Rationale**:
- Natively supports the full Conventional Commits specification including `fix:` →
  PATCH, `feat:` → MINOR, `feat!:` / `BREAKING CHANGE:` footer → MAJOR.
- Non-release prefixes (`chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:`)
  accumulate without bumping a version — cleanly satisfying FR-007.
- Creates both the Git tag and a GitHub Release with auto-generated release notes in
  a single action run — satisfying FR-001, FR-002, FR-003.
- Uses a "Release PR" model: on every push to `main`, it either creates or updates a
  "Release PR" that aggregates all pending changes. Merging that PR creates the tag
  and release. This gives teams a human review gate before publishing.
- Works with `secrets.GITHUB_TOKEN` using `contents: write` and
  `pull-requests: write` permissions — no PAT required for basic operation.
- No prior version tags → defaults to `v1.0.0` on first release (FR-008).
- Actively maintained (2,342 action stars, 6,622 underlying library stars as of
  March 2026). No known CVEs specific to the action.

**Pinnable SHA**: `16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0, released
2025-10-23). This SHA MUST be used in all `uses:` references per Constitution
Principle III.

**Alternatives considered**:
- `semantic-release` (direct via `npx`) — 23,476 stars, most popular overall.
  Rejected because it requires `.releaserc.json` configuration in every consuming
  repo, adding boilerplate for consumers. The release-please model keeps all
  configuration in `release-please-config.json` in the target repo, which is more
  discoverable. Additionally, the Angular convention default (vs. Conventional
  Commits) requires extra config to get `feat!:` and `BREAKING CHANGE:` footer
  parsing.
- `cycjimmy/semantic-release-action` (681 stars) — lower adoption, CVE-2025-5889
  (low, transitive) present, rejected in favor of higher-adoption option.
- `mathieudutour/github-tag-action` — maintenance stalled since March 2024, does
  not create GitHub Releases natively. Rejected.

**Known limitation**: The `releases_created` output in release-please-action v4 has
a known bug (Issue #965) where it returns `true` regardless of whether a release was
published. The correct output to use for conditionals is `release_created` (singular,
not plural).

---

## Decision 2: Moving Major Version Pointer Mechanism

**Decision**: Use a plain shell `run:` step with native `git tag -fa` and
`git push origin vN --force` commands — no additional action dependency.

**Rationale**:
- Official GitHub guidance (`actions/toolkit/docs/action-versioning.md`) documents
  this exact pattern as the standard for Actions repositories.
- A `run:` step has no external supply-chain dependency (no SHA to pin, no upstream
  to compromise).
- `contents: write` permission is sufficient to create and force-push tag refs when
  no tag protection rules/rulesets are in place.
- Logic is simple (extract major from `vMAJOR.MINOR.PATCH`, force-push) — does not
  warrant an external action (Constitution Principle IV: Minimal Surface Area).

**Required git configuration before tagging**:
```bash
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
```

**Tag extraction pattern**:
```bash
VERSION=${GITHUB_REF#refs/tags/}   # "v1.2.3"
MAJOR=${VERSION%%.*}               # "v1"
git tag -fa "${MAJOR}" -m "Update ${MAJOR} tag to ${VERSION}"
git push origin "${MAJOR}" --force
```

**Alternatives considered**:
- `nowactions/update-majorver` action — wraps the same shell commands. Rejected
  because adding an external action dependency for a 3-line shell script violates
  Constitution Principle IV and Principle III (requires SHA vetting for minimal
  gain).
- `actions/github-script@ed597411...` (v8.0.0) — appropriate for REST API calls
  but unnecessary overhead for a pure git ref operation. Rejected.

**Known gotcha — tag protection rulesets**: If a consuming repo has GitHub Rulesets
with tag name patterns matching `v*`, `GITHUB_TOKEN` (as `github-actions[bot]`)
cannot be added to the bypass list. Force-pushing the major pointer will fail with
403. Workaround: use a GitHub App token via `actions/create-github-app-token`. This
is documented in the workflow's README as a known limitation.

**Known gotcha — force-push does NOT trigger downstream workflows**: A tag
force-pushed via `GITHUB_TOKEN` will not trigger other workflows listening on
`on: push: tags:`. This is intentional GitHub behavior to prevent loops.

---

## Decision 3: Reusable Workflow Structure (`workflow_call`)

**Decision**: The semver release pipeline is delivered as a reusable workflow at
`.github/workflows/semver-release.yml` with `on: workflow_call:` trigger. It is NOT
a composite action.

**Rationale**:
- `workflow_call` is the correct GitHub construct for sharing multi-job pipelines
  across repos. Composite actions are suited for single-step logic, not full release
  pipelines with multiple jobs.
- Consumer repos reference it with a single `uses:` line (FR-004), e.g.:
  ```yaml
  uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
  ```
- Inputs exposed via `workflow_call.inputs` allow consumers to override release
  branch and tag prefix (FR-005).
- The reusable workflow is self-contained and ships with its own `permissions:`
  block, so consumers do not need to declare permissions themselves.

**Workflow inputs defined**:
| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `release-branch` | string | `main` | Branch to base releases on |
| `tag-prefix` | string | `v` | Prefix for version tags |
| `config-file` | string | `.release-please-config.json` | Path to release-please config |

**Workflow outputs defined**:
| Output | Type | Description |
|--------|------|-------------|
| `release-created` | boolean | Whether a release was published this run |
| `tag-name` | string | The full version tag created (e.g., `v1.2.3`) |
| `major-tag` | string | The major pointer tag updated (e.g., `v1`) |

---

## Decision 4: Third-Party Actions — SHA Pinning

Per Constitution Principle III, all external actions MUST be pinned to commit SHAs.

| Action | Purpose | Pinned SHA | Version |
|--------|---------|-----------|---------|
| `googleapis/release-please-action` | Version calc + tag + release | `16a9c90856f42705d54a6fda1823352bdc62cf38` | v4.4.0 |
| `actions/checkout` | Repo checkout for pointer update step | Verify current SHA at implementation time | v4.x |

`actions/checkout` SHA must be verified at implementation time from
`https://github.com/actions/checkout/releases` to get the latest v4 patch.

---

## Decision 5: CI Test Workflow

**Decision**: A test workflow at `.github/workflows/test-semver-release.yml` runs
on every PR to `main` and validates the reusable workflow structure using a dry-run
or lint check.

**Rationale**: Constitution Principle V requires every action/reusable workflow to
have an automated test covering the happy path and at least one error path before
release. Since the release workflow can't be fully end-to-end tested without an
actual merge (it would create real tags), the test strategy is:

1. **Lint/validate**: Use `actionlint` to verify the workflow YAML is syntactically
   valid and free of common mistakes.
2. **Dry-run**: Pass `--dry-run` flag to release-please to verify it can parse
   commits and calculate a version without creating tags.
3. **Input validation**: Test that invalid inputs (e.g., unsupported `tag-prefix`)
   fail with a clear error message.

`actionlint` is available as a GitHub Action:
`rhysd/actionlint` — verify SHA at implementation time.

---

## Decision 6: Permissions Model

The reusable workflow declares the minimum permissions required (FR-010):

```yaml
permissions:
  contents: write      # create/push tags, create releases
  pull-requests: write # release-please creates/updates the Release PR
```

No other permissions are needed. `contents: write` covers tag creation, tag
force-push (for major pointer), and GitHub Release creation. `pull-requests: write`
is required by release-please to create and update its Release PR.
