# Quickstart & Validation Guide: Semantic Versioning CI/CD Pipelines

**Feature**: 001-semver-cicd-pipelines
**Date**: 2026-03-26

Use this guide to manually validate the feature end-to-end after implementation.
Each scenario maps to an acceptance criterion from the spec.

---

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`)
- A test consumer repository (can be a temporary private repo in the homeschoolio
  org) — referred to as `<consumer-repo>` below
- This repo (`homeschoolio-shared-actions`) has been merged to `main` and tagged
  with at least `v1.0.0` before running consumer tests

---

## Scenario 1: Happy Path — PATCH Release (US1, FR-001, FR-002, FR-003)

**Goal**: Verify a `fix:` commit triggers a PATCH version bump, tag, and release.

### Steps

1. In `<consumer-repo>`, add a workflow file at `.github/workflows/release.yml`:
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
   No config files required — the reusable workflow handles the default configuration.

2. Make a commit with message `fix: correct typo in README` and push to a branch.

3. Open and merge a PR to `main`.

4. **Verify** (semantic-release publishes directly — no Release PR):
   ```bash
   gh release list --repo <consumer-repo> --limit 5
   # Expected: v1.0.1 (or v1.0.0 if first release)
   gh api repos/<consumer-repo>/git/refs/tags/v1 --jq '.object.sha'
   # Expected: SHA matches v1.0.1 tag
   ```

**Pass criteria**: Release exists, tag `v1.0.1` exists, `v1` pointer updated.

---

## Scenario 2: MINOR Release — `feat:` commit (US1, FR-002)

### Steps

1. Commit `feat: add new onboarding section` to a branch in `<consumer-repo>`.
2. Open and merge PR to `main`.

**Verify**:
```bash
gh release view v1.1.0 --repo <consumer-repo>
# Expected: Release v1.1.0 exists with changelog entry for the feat commit
```

---

## Scenario 3: MAJOR Release — Breaking Change (US1, FR-002)

### Steps

1. Commit with message:
   ```
   feat!: redesign configuration format

   BREAKING CHANGE: config.json keys renamed, existing configs must be updated
   ```
2. Open and merge PR to `main`.

**Verify**:
```bash
gh release view v2.0.0 --repo <consumer-repo>
# Expected: Release v2.0.0, release body mentions breaking change
gh api repos/<consumer-repo>/git/refs/tags/v2 --jq '.object.sha'
# Expected: v2 pointer exists and matches v2.0.0
gh api repos/<consumer-repo>/git/refs/tags/v1 --jq '.object.sha'
# Expected: v1 pointer still points to last v1.x.y (unchanged)
```

---

## Scenario 4: No-Release Case — `chore:` only commits (US1, FR-007)

### Steps

1. Commit `chore: update dependencies` and push to a branch in `<consumer-repo>`.
2. Merge PR to `main`.

**Verify**:
```bash
gh run list --repo <consumer-repo> --workflow release.yml --limit 3
# Expected: Workflow ran successfully
# Check the run logs: "release-created" output should be "false"
gh release list --repo <consumer-repo> --limit 1
# Expected: Most recent release is the same as before (no new release created)
```

---

## Scenario 5: First Release — No Prior Tags (US1, FR-008)

### Steps

1. Use a brand new `<consumer-repo>` with zero version tags.
2. Add the consumer workflow (no config files needed).
3. Commit `feat: initial implementation` and merge to `main`.

**Verify**:
```bash
gh release list --repo <consumer-repo> --limit 1
# Expected: v1.0.0 release exists
gh api repos/<consumer-repo>/git/refs/tags/v1.0.0 --jq '.object.sha'
# Expected: tag exists
```

---

## Scenario 6: Consumer Adoption — Single `uses:` Line (US2, SC-001)

**Goal**: Verify a consumer can adopt the workflow with fewer than 20 lines and zero config files.

### Steps

1. Count the lines in the minimal consumer workflow file:
   ```bash
   wc -l <consumer-repo>/.github/workflows/release.yml
   # Expected: < 20 lines
   ```

2. Confirm no config files were required beyond:
   - `.github/workflows/release.yml` (the only required file)

**Pass criteria**: Adoption requires exactly 1 file; workflow file is under 20 lines.

---

## Scenario 7: Major Pointer Update (US3, FR-006)

**Goal**: Verify `v1` pointer is updated atomically in the same pipeline run as the
version tag.

### Steps

1. After any release (e.g., Scenario 1 above), verify the pointer SHA matches the release tag SHA:
   ```bash
   SHA_RELEASE=$(gh api repos/<consumer-repo>/git/refs/tags/v1.0.1 --jq '.object.sha')
   SHA_POINTER=$(gh api repos/<consumer-repo>/git/refs/tags/v1 --jq '.object.sha')
   [ "$SHA_RELEASE" = "$SHA_POINTER" ] && echo "PASS" || echo "FAIL"
   ```

---

## Scenario 8: Self-Versioning — This Repo Versions Itself (US4, FR-011)

**Goal**: Verify that merging to `main` in this repo triggers `release.yml` via
local path and produces the correct version tag.

### Steps

1. Merge a PR with a conventional commit to `main` in `homeschoolio-shared-actions`.

2. **Verify**:
   ```bash
   gh run list --repo jmckenzie17/homeschoolio-shared-actions \
     --workflow release.yml --limit 3
   # Expected: most recent run is "completed / success"

   gh release list --repo jmckenzie17/homeschoolio-shared-actions --limit 1
   # Expected: new version tag (e.g., v1.0.0 on first run)
   ```

3. Confirm the local path reference was used (not an external tag reference):
   ```bash
   grep "uses:" .github/workflows/release.yml
   # Expected: uses: ./.github/workflows/semver-release.yml  (no @tag suffix)
   ```

**Pass criteria**: Release created without any manually published version of this
repo having been required first.

---

## Scenario 9: SHA Pinning Verification (Constitution Principle III)

**Goal**: Verify no mutable tag references exist in the workflow file.

```bash
grep -E "uses:.*@(v[0-9]+|main|master|latest)" \
  .github/workflows/semver-release.yml
# Expected: zero matches (all uses: references are full commit SHAs)
```

---

## Scenario 10: Permissions Verification (FR-010)

**Goal**: Verify only `contents: write` is declared — `pull-requests: write` is not needed.

```bash
grep -A5 "permissions:" .github/workflows/semver-release.yml
# Expected output contains ONLY:
#   contents: write
# pull-requests: write MUST NOT appear.
```

---

## Scenario 11: CI Test Workflow Passes on PR (Constitution Principle V)

**Goal**: Verify the test workflow runs and passes on every PR to `main` in this repo.

### Steps

1. Open any PR to `main` in `homeschoolio-shared-actions`.
2. Wait for CI checks to complete.

**Verify**:
```bash
gh pr checks <PR-number>
# Expected: "test-semver-release" check shows green/passing
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| No release created after merge | Non-releasable commits only (`chore:`, `docs:`, etc.) | Expected behavior — use `fix:` or `feat:` to trigger a release |
| `403 Resource not accessible` on tag push | Tag Ruleset blocking `github-actions[bot]` | Use GitHub App token — see README workaround |
| Major pointer not updated | Release step output not set | Check workflow logs; ensure conditional uses `new_release_published == 'true'` |
| semantic-release fails on first run | Shallow clone missing git tags | Ensure `fetch-depth: 0` in checkout step |
| `v1` pointer triggers downstream workflows | Using PAT instead of GITHUB_TOKEN | Expected behavior for PAT; switch to GITHUB_TOKEN if loops occur |
| actionlint errors in CI | YAML syntax issue in workflow | Fix the flagged lines; run `actionlint` locally to preview |
