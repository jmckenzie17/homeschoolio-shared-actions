# Tasks: Semantic Versioning CI/CD Pipelines

**Input**: Design documents from `/specs/001-semver-cicd-pipelines/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/semver-release-workflow.md, quickstart.md

**Organization**: Tasks are grouped by user story to enable independent implementation
and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1–US4)
- Exact file paths are included in all descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Confirm all pinned SHAs before writing any workflow code.

- [x] T001 Verify commit SHA for `cycjimmy/semantic-release-action` v6.0.0 is `b12c8f6015dc215fe37bc154d4ad456dd3833c90` by checking https://github.com/cycjimmy/semantic-release-action/releases — record in research.md Decision 4
- [x] T002 [P] Verify commit SHA for `actions/checkout` v4.3.1 is `34e114876b0b11c390a56381ad16ebd13914f8d5` — record in research.md Decision 4
- [x] T003 [P] Verify commit SHA for `rhysd/actionlint` v1.7.11 is `393031adb9afb225ee52ae2ccd7a5af5525e03e8` — record in research.md Decision 4

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The reusable workflow (`semver-release.yml`) is the shared dependency for
all four user stories. It must exist with correct structure before any caller or consumer
can be built.

**⚠️ CRITICAL**: No user story phase can begin until this phase is complete.

- [x] T004 Create `.github/workflows/semver-release.yml` with `on: workflow_call:` trigger declaring two inputs (`release-branch` string default `"main"`, `tag-prefix` string default `"v"`), three outputs (`release-created`, `tag-name`, `major-tag`), and top-level `permissions: contents: write` — see plan.md "Workflow Architecture" and contracts/semver-release-workflow.md Inputs/Outputs sections
- [x] T005 Add the `release-please` job body to `.github/workflows/semver-release.yml`: checkout step using `actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5` with `fetch-depth: 0` (required for git tag history), then `cycjimmy/semantic-release-action@b12c8f6015dc215fe37bc154d4ad456dd3833c90` step with `semantic_version: 25`, `extra_plugins` listing `@semantic-release/commit-analyzer`, `@semantic-release/release-notes-generator`, `@semantic-release/github`, `branches: ['${{ inputs.release-branch }}']`, and `tag_format: '${{ inputs.tag-prefix }}${version}'`; set `env: GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` — see research.md Decision 8
- [x] T006 Run `actionlint .github/workflows/semver-release.yml` and fix all reported errors before proceeding

**Checkpoint**: `semver-release.yml` skeleton exists, passes `actionlint`, exposes the correct `workflow_call` interface. User story phases can now proceed.

---

## Phase 3: User Story 1 — Automated Release Versioning on Merge (Priority: P1) 🎯 MVP

**Goal**: Merging a conventional-commit PR to `main` in any repo using this workflow automatically produces the correct SemVer tag and GitHub Release directly (no Release PR intermediary); `chore:`/`docs:`-only merges produce no tag.

**Independent Test** (quickstart.md Scenarios 1–5): Merge a PR with `fix:` commit to a consumer repo referencing this workflow; verify PATCH tag + GitHub Release created. Merge a `chore:`-only PR; verify no tag created.

- [x] T007 [US1] Wire the `cycjimmy/semantic-release-action` step outputs to the job-level outputs in `.github/workflows/semver-release.yml`: `release-created` from `steps.semantic-release.outputs.new_release_published`, `tag-name` from `steps.semantic-release.outputs.new_release_git_tag`; add a comment noting `new_release_published` is a string `"true"`/`"false"` — see contracts/semver-release-workflow.md Outputs and research.md Decision 4 output key names
- [x] T008 [US1] Add the major version pointer update step to `.github/workflows/semver-release.yml` after the semantic-release step, conditioned on `steps.semantic-release.outputs.new_release_published == 'true'`: extract `MAJOR` from `VERSION=${TAG%%.*}` where `TAG` is `steps.semantic-release.outputs.new_release_git_tag`; run `git config user.name/email`, `git tag -fa "${MAJOR}" -m "Update ${MAJOR} to ${TAG}"`, `git push origin "${MAJOR}" --force`; output `major_tag` — add inline comment: "MAJOR extracted from full tag — each release only updates its own major pointer (e.g. v2.0.0 → v2, leaving v1 untouched)" — see research.md Decision 2
- [x] T009 [US1] Set the workflow-level outputs in `.github/workflows/semver-release.yml` from job outputs: `release-created`, `tag-name`, `major-tag` from `jobs.release-please.outputs.*`; wire `major-tag` from `steps.update-major-pointer.outputs.major_tag`
- [x] T010 [US1] Run `actionlint .github/workflows/semver-release.yml` after all steps added and fix all errors

**Checkpoint**: `semver-release.yml` fully implements US1 — semantic-release creates tag + GitHub Release on push; major pointer updated in same run; no-release commits skip both.

---

## Phase 4: User Story 4 — This Repo Versions Itself (Priority: P1)

**Goal**: Pushing to `main` in `homeschoolio-shared-actions` triggers `release.yml`, which calls `semver-release.yml` via local path, producing `v1.0.0` with no prior published release required.

**Independent Test** (quickstart.md Scenario 8): Merge this PR; verify `release.yml` triggers and `v1.0.0` is created; verify `uses:` line has no `@tag` suffix.

*Depends on Phase 3: the local reusable workflow must be functional before the caller is useful.*

- [x] T011 [US4] Create `.github/workflows/release.yml` with `on: push: branches: [main]`, `concurrency: group: release, cancel-in-progress: false` (add comment: "last-pending-wins — semantic-release is idempotent over accumulated commits"), and a single job `release` with `permissions: contents: write` calling `uses: ./.github/workflows/semver-release.yml` with no inputs (all defaults apply) — see plan.md "release.yml" and data-model.md Entity 7
- [x] T012 [US4] Run `actionlint .github/workflows/release.yml` and fix all errors

**Checkpoint**: `release.yml` exists, passes `actionlint`, uses local path reference. Merging this PR to `main` will trigger `v1.0.0` creation.

---

## Phase 5: User Story 2 — Consumer Repo Adopts the Reusable Workflow (Priority: P2)

**Goal**: Any homeschoolio repo can adopt automated semver with a single `uses:` reference in under 20 lines — no config files required. Consumers understand required permissions.

**Independent Test** (quickstart.md Scenarios 1 & 6): Consumer repo using only the minimal workflow file produces correct version tags on merge; workflow file is under 20 lines; no `.releaserc.json` required.

- [x] T013 [P] [US2] Create `docs/semver-release/examples/consumer-workflow.yml` — minimal consumer workflow with `on: push: branches: [main]`, `permissions: contents: write` on the calling job, `uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1`, and `secrets: inherit`; must be ≤ 20 lines total — see data-model.md Entity 6
- [x] T014 [P] [US2] Create `docs/semver-release/examples/consumer-workflow-with-deploy.yml` — extended example chaining a `deploy` job gated on `needs.release.outputs.release-created == 'true'` using `needs.release.outputs.tag-name` — see contracts/semver-release-workflow.md "Output Usage Example"
- [x] T015 [US2] Create `docs/semver-release/README.md` covering: inputs table, outputs table, required `contents: write` permission with explicit note that callers must declare it; note that `pull-requests: write` is NOT needed; zero-config adoption (no `.releaserc.json` required); optional `.releaserc.json` override for advanced config; `new_release_published` output key name; tag protection ruleset limitation + GitHub App token workaround; links to example files — see contracts/semver-release-workflow.md and research.md

**Checkpoint**: Consumer documentation and examples are complete. A developer can read `docs/semver-release/README.md` and adopt the workflow without additional context.

---

## Phase 6: User Story 3 — Moving Major Version Pointer Maintained (Priority: P3)

**Goal**: After every release, the `vN` pointer tag is force-updated to the latest release commit in the same pipeline run — no separate step or manual action required.

**Independent Test** (quickstart.md Scenario 7): After a PATCH release, verify `v1` SHA matches `v1.0.1` SHA via `gh api`; after a MAJOR release verify `v2` is created and `v1` is unchanged.

*The pointer update logic was implemented in T008. This phase verifies correctness and documents it.*

- [x] T016 [US3] Review the `git tag -fa "${MAJOR}"` shell step in `.github/workflows/semver-release.yml` added in T008 — verify the MAJOR bump case is handled correctly (when `v2.0.0` is released, `MAJOR` extracts to `v2`, not `v1`; `v1` is untouched); confirm the inline comment documents this
- [x] T017 [US3] Add a "Moving Major Version Pointer" section to `docs/semver-release/README.md` documenting: PATCH/MINOR updates the existing `vN` pointer; MAJOR bump creates new `vN+1` pointer and leaves `vN` frozen at its last `vN.x.y` release; force-push via `GITHUB_TOKEN` does NOT trigger downstream workflows (intentional GitHub behavior)

**Checkpoint**: Pointer behavior verified correct for both PATCH/MINOR and MAJOR cases; behavior documented for consumers.

---

## Phase 7: CI Test Workflow (Constitution Principle V)

**Purpose**: Ensure `test-semver-release.yml` validates the reusable workflow on every PR to `main`.

- [x] T018 Create (or rewrite) `.github/workflows/test-semver-release.yml` with three jobs: (1) `validate` — runs `actionlint@393031adb9afb225ee52ae2ccd7a5af5525e03e8` on `.github/workflows/semver-release.yml`; (2) `dry-run` — checks out repo and runs `npx semantic-release --dry-run --no-ci --repository-url=${{ github.repositoryUrl }}` to verify commit parsing without pushing tags; (3) `sha-pinning` — `grep -E 'uses:.*@(v[0-9]+|main|master|latest)' .github/workflows/semver-release.yml` asserting zero matches — workflow triggers on `pull_request: branches: [main]` — all external actions SHA-pinned
- [x] T019 Run `actionlint .github/workflows/test-semver-release.yml` and fix all errors

**Checkpoint**: Test workflow is fully SHA-pinned, validates semver-release.yml on every PR, passes `actionlint`.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Final consistency pass, inline comments, and static validation.

- [x] T020 [P] Add inline comments to `.github/workflows/semver-release.yml`: (a) on `fetch-depth: 0` — "required: semantic-release reads full git tag history to determine prior releases"; (b) on `new_release_published` conditional — "use new_release_published string output; compare with == 'true' not == true"; (c) on the pointer step — "MAJOR extracted from full tag — only updates its own major pointer"
- [x] T021 [P] Count lines in `docs/semver-release/examples/consumer-workflow.yml` — must be ≤ 20 (SC-001); trim if needed
- [x] T022 Run quickstart.md Scenario 9 static check locally: `grep -E "uses:.*@(v[0-9]+|main|master|latest)" .github/workflows/semver-release.yml` — expect zero matches
- [x] T023 Run quickstart.md Scenario 10 static check locally: `grep -A5 "permissions:" .github/workflows/semver-release.yml` — expect only `contents: write`; confirm `pull-requests: write` is absent
- [x] T024 Update `CLAUDE.md` to reflect the new primary dependency (`cycjimmy/semantic-release-action` v6.0.0) and remove references to release-please

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 (confirmed SHAs) — **blocks Phases 3–6**
- **Phase 3 (US1)**: Depends on Phase 2
- **Phase 4 (US4)**: Depends on Phase 3 (caller depends on functional reusable workflow)
- **Phase 5 (US2)**: Depends on Phase 2; can run in parallel with Phases 3/4
- **Phase 6 (US3)**: Depends on Phase 3 (pointer logic lives in `semver-release.yml`)
- **Phase 7 (CI Tests)**: Depends on Phase 2
- **Phase 8 (Polish)**: Depends on all prior phases

### User Story Dependencies

- **US1 (P1)**: Foundation complete → start
- **US4 (P1)**: US1 complete → start (local path caller needs functional reusable workflow)
- **US2 (P2)**: Foundation complete → can overlap with US1/US4
- **US3 (P3)**: US1 complete → verification only, no new workflow code

### Parallel Opportunities

- T001, T002, T003 — all parallel (independent SHA verification)
- T013, T014 — parallel (different example files)
- T020, T021 — parallel (different files)
- Phase 5 tasks can be drafted while Phase 3/4 are in progress

---

## Parallel Example: Phase 1

```
# Launch all three setup tasks in parallel:
Task T001: "Verify cycjimmy/semantic-release-action SHA"
Task T002: "Verify actions/checkout SHA"
Task T003: "Verify rhysd/actionlint SHA"
```

## Parallel Example: Phase 5 (US2)

```
# Launch both consumer example files in parallel:
Task T013: "Create docs/semver-release/examples/consumer-workflow.yml"
Task T014: "Create docs/semver-release/examples/consumer-workflow-with-deploy.yml"
```

---

## Implementation Strategy

### MVP First (ship `v1.0.0`)

1. Phase 1: Setup — confirm SHAs
2. Phase 2: Foundational — `semver-release.yml` skeleton
3. Phase 3: US1 — semantic-release step + major pointer logic + outputs
4. Phase 4: US4 — `release.yml` self-caller
5. Phase 7: CI Tests
6. **Merge PR to `main`** → `release.yml` triggers → `v1.0.0` created automatically

### Incremental Delivery

1. MVP above → `v1.0.0` tagged; consumers can reference `@v1`
2. Phase 5 (US2): consumer docs + examples → `v1.1.0`
3. Phase 6 (US3): pointer docs → included in same PR or follow-up patch
4. Phase 8: Polish → patch release

### Single Developer Sequential Order

T001 → T002 → T003 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T012 → T018 → T019 → T013 → T014 → T015 → T016 → T017 → T020 → T021 → T022 → T023 → T024

---

## Notes

- [P] tasks = different files, no incomplete task dependencies
- [Story] label maps each task to its user story for traceability
- No test tasks generated — spec does not request TDD; `test-semver-release.yml` serves as the automated test harness
- `new_release_published` MUST be used in all conditionals — it is a string; compare with `== 'true'` not `== true`
- `release.yml` local path `uses: ./.github/workflows/semver-release.yml` requires no `@tag` suffix — resolves from the same commit as the caller
- `pull-requests: write` is NOT required — semantic-release publishes directly without a Release PR
- No `.release-please-config.json` or `.release-please-manifest.json` needed — semantic-release uses inline plugin config and derives version from git tags
- Commit after each phase checkpoint for clean, bisectable git history
