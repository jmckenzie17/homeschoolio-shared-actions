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

**Purpose**: Verify pinned SHAs and confirm output key names before any workflow is written.

- [x] T001 Verify and record the current commit SHA for `actions/checkout` v4 (latest v4.x patch) by checking https://github.com/actions/checkout/releases — update research.md Decision 4 with the confirmed SHA
- [x] T002 [P] Verify and record the current commit SHA for `rhysd/actionlint` v1.7.x by checking https://github.com/rhysd/actionlint/releases — update research.md Decision 4 with the confirmed SHA
- [x] T003 [P] Verify the exact output key names for `googleapis/release-please-action` v4.4.0 (SHA `16a9c90856f42705d54a6fda1823352bdc62cf38`) — confirm `release_created` (singular), `tag_name`, and `major` are correct by reading the action's `action.yml` at that SHA; note any corrections needed in plan.md Open Items

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The reusable workflow (`semver-release.yml`) is the shared dependency for all four user stories. It must exist with correct structure before any caller or consumer can be built.

**⚠️ CRITICAL**: No user story phase can begin until this phase is complete.

- [x] T004 Create `.github/workflows/semver-release.yml` skeleton with `on: workflow_call:` trigger declaring three inputs (`release-branch` string default `main`, `tag-prefix` string default `v`, `config-file` string default `.release-please-config.json`), three outputs (`release-created`, `tag-name`, `major-tag`), and top-level `permissions: contents: write` + `pull-requests: write` — see plan.md "semver-release.yml structure" and contracts/semver-release-workflow.md
- [x] T005 Add the `release-please` job body to `.github/workflows/semver-release.yml`: one step using `googleapis/release-please-action@16a9c90856f42705d54a6fda1823352bdc62cf38` with `token: ${{ secrets.GITHUB_TOKEN }}`, `release-type: simple`, `target-branch: ${{ inputs.release-branch }}`, and `config-file: ${{ inputs.config-file }}`; wire step outputs to job-level outputs using key names confirmed in T003
- [x] T006 Run `actionlint .github/workflows/semver-release.yml` and fix all reported errors before proceeding

**Checkpoint**: `semver-release.yml` skeleton exists, exposes the correct `workflow_call` interface, and passes `actionlint`. User story phases can now proceed.

---

## Phase 3: User Story 1 — Automated Release Versioning on Merge (Priority: P1) 🎯 MVP

**Goal**: Merging a conventional-commit PR to `main` in any repo using this workflow automatically produces the correct SemVer tag and GitHub Release; `chore:`/`docs:`-only merges produce no tag.

**Independent Test** (quickstart.md Scenarios 1–5): Merge a PR with `fix:` commit to a consumer repo referencing this workflow; verify PATCH tag + GitHub Release created. Merge a `chore:`-only PR; verify no tag created.

- [x] T007 [US1] Add the major version pointer update steps to `.github/workflows/semver-release.yml` after the release-please step — step 1: `actions/checkout@<SHA-from-T001>` with `fetch-depth: 0`, conditioned on `steps.release.outputs.release_created == 'true'`; step 2: inline shell extracting `MAJOR=${TAG%%.*}` and running `git config user.name/email` then `git tag -fa "${MAJOR}" -m "Update ${MAJOR} to ${TAG}" && git push origin "${MAJOR}" --force`, using `env: TAG: ${{ steps.release.outputs.tag_name }}` — see research.md Decision 2 and plan.md "semver-release.yml structure"
- [x] T008 [US1] Set the workflow-level outputs in `.github/workflows/semver-release.yml` from the job outputs: `release-created` from `jobs.release-please.outputs.release-created`, `tag-name` from `jobs.release-please.outputs.tag-name`, `major-tag` from `jobs.release-please.outputs.major-tag`; add a comment on the `release-created` output noting it is a string `"true"`/`"false"` not a boolean — see contracts/semver-release-workflow.md Outputs section
- [x] T009 [US1] Run `actionlint .github/workflows/semver-release.yml` after pointer step additions and fix all errors

**Checkpoint**: `semver-release.yml` fully implements US1 — release-please creates tag + release; major pointer updated in same run; no-release commits skip both.

---

## Phase 4: User Story 4 — This Repo Versions Itself (Priority: P1)

**Goal**: Merging to `main` in `homeschoolio-shared-actions` triggers `release.yml`, which calls `semver-release.yml` via local path, producing `v1.0.0` with no prior published release required.

**Independent Test** (quickstart.md Scenario 8): Merge this PR; verify `release.yml` triggers and `v1.0.0` is created; verify `uses:` line in `release.yml` has no `@tag` suffix.

*Depends on Phase 3: the local reusable workflow must be functional before the caller is useful.*

- [x] T010 [US4] Create `.github/workflows/release.yml` with `on: push: branches: [main]`, `concurrency: group: release, cancel-in-progress: false` (add comment: "last-pending-wins — release-please is idempotent over accumulated commits"), and a single job `release` with `permissions: contents: write` + `pull-requests: write` calling `uses: ./.github/workflows/semver-release.yml` — see plan.md "release.yml structure" and data-model.md Entity 8
- [x] T011 [US4] Run `actionlint .github/workflows/release.yml` and fix all errors

**Checkpoint**: `release.yml` exists, passes `actionlint`, uses local path reference. Merging this PR to `main` will trigger `v1.0.0` creation.

---

## Phase 5: User Story 2 — Consumer Repo Adopts the Reusable Workflow (Priority: P2)

**Goal**: Any homeschoolio repo can adopt automated semver with a single `uses:` reference and a config file in under 20 lines. Consumers understand required permissions.

**Independent Test** (quickstart.md Scenarios 1 & 6): Consumer repo using the example workflow and `release-please-config.json` produces correct version tags on merge; workflow file is under 20 lines.

- [x] T012 [P] [US2] Create `docs/semver-release/examples/consumer-workflow.yml` — minimal consumer workflow with `on: push: branches: [main]`, `permissions: contents: write` + `pull-requests: write` on the calling job, `uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1`, and `secrets: inherit`; must be ≤ 20 lines total — see data-model.md Entity 7
- [x] T013 [P] [US2] Create `docs/semver-release/examples/consumer-workflow-with-deploy.yml` — extended example chaining a `deploy` job gated on `needs.release.outputs.release-created == 'true'` using `needs.release.outputs.tag-name` — see contracts/semver-release-workflow.md "Output Usage Example"
- [x] T014 [US2] Create `docs/semver-release/README.md` covering: inputs table, outputs table, required permissions with explicit note that callers must declare them (not inherited automatically), minimal `release-please-config.json` content, `release_created` vs `releases_created` gotcha, tag protection ruleset limitation + workaround reference, and links to example files — see contracts/semver-release-workflow.md and research.md

**Checkpoint**: Consumer documentation and examples are complete. A developer can read `docs/semver-release/README.md` and adopt the workflow without additional context.

---

## Phase 6: User Story 3 — Moving Major Version Pointer Maintained (Priority: P3)

**Goal**: After every release, the `vN` pointer tag is force-updated to the latest release commit in the same pipeline run — no separate step or manual action required.

**Independent Test** (quickstart.md Scenario 7): After a PATCH release, verify `v1` SHA matches `v1.0.1` SHA via `gh api`; after a MAJOR release verify `v2` is created and `v1` is unchanged.

*The pointer update logic was implemented in T007. This phase verifies correctness of the MAJOR bump case and documents it.*

- [x] T015 [US3] Review the `git tag -fa "${MAJOR}"` shell step in `.github/workflows/semver-release.yml` added in T007 — verify the MAJOR bump case is handled correctly (when `v2.0.0` is released, `MAJOR` extracts to `v2`, not `v1`; `v1` is untouched); add an inline comment in the shell step documenting: "MAJOR is extracted from the full tag — each release only updates its own major pointer"
- [x] T016 [US3] Add a "Pointer Tag Behavior" section to `docs/semver-release/README.md` documenting: PATCH/MINOR updates the existing `vN` pointer; MAJOR bump creates new `vN+1` pointer and leaves `vN` frozen at its last `vN.x.y` release

**Checkpoint**: Pointer behavior verified correct for both PATCH/MINOR and MAJOR cases; behavior documented for consumers.

---

## Phase 7: CI Test Workflow (Constitution Principle V)

**Purpose**: Ensure `test-semver-release.yml` is correctly SHA-pinned and uses the dynamic repo URL fix applied in this branch.

- [x] T017 Confirm `.github/workflows/test-semver-release.yml` `dry-run` job uses `--repo-url=${{ github.repository }}` — verify the fix is present (applied earlier in this branch)
- [x] T018 Update `actions/checkout` SHA in `.github/workflows/test-semver-release.yml` to the SHA confirmed in T001 if it differs from the current value
- [x] T019 Update `rhysd/actionlint` SHA in `.github/workflows/test-semver-release.yml` to the SHA confirmed in T002 if it differs from the current value
- [x] T020 Run `actionlint .github/workflows/test-semver-release.yml` and fix all errors

**Checkpoint**: Test workflow is fully SHA-pinned, uses dynamic `${{ github.repository }}`, and passes `actionlint`. CI validates `semver-release.yml` on every PR.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Final consistency pass, inline comments, and end-to-end static validation.

- [x] T021 [P] Add inline comments to `.github/workflows/semver-release.yml`: (a) on the `release_created` conditional — "use singular release_created; releases_created (plural) is a known v4 bug that always returns true"; (b) on `fetch-depth: 0` — "required for git tag operations"; (c) on the pointer step — see T015 comment
- [x] T022 [P] Count lines in `docs/semver-release/examples/consumer-workflow.yml` — must be ≤ 20 (SC-001); trim if needed
- [x] T023 Run quickstart.md static validation scenarios locally: Scenario 9 (`grep -E "uses:.*@(v[0-9]+|main|master|latest)" .github/workflows/semver-release.yml` — expect zero matches), Scenario 10 (`grep -A5 "permissions:" .github/workflows/semver-release.yml` — expect only `contents: write` and `pull-requests: write`)
- [x] T024 Update `CLAUDE.md` Project Structure section to include `.github/workflows/release.yml` as the self-release caller workflow

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 (needs confirmed SHAs from T001–T003) — **blocks Phases 3–6**
- **Phase 3 (US1)**: Depends on Phase 2
- **Phase 4 (US4)**: Depends on Phase 3 (caller depends on functional reusable workflow)
- **Phase 5 (US2)**: Depends on Phase 2; can run in parallel with Phases 3/4
- **Phase 6 (US3)**: Depends on Phase 3 (pointer logic lives in `semver-release.yml`)
- **Phase 7 (CI Tests)**: Depends on Phase 2; T018/T019 depend on Phase 1
- **Phase 8 (Polish)**: Depends on all prior phases

### User Story Dependencies

- **US1 (P1)**: Foundation complete → start
- **US4 (P1)**: US1 complete → start (local path caller needs functional reusable workflow)
- **US2 (P2)**: Foundation complete → can overlap with US1/US4
- **US3 (P3)**: US1 complete → verification only, no new workflow code

### Parallel Opportunities

- T001, T002, T003 — all parallel (independent research)
- T012, T013 — parallel (different example files)
- T021, T022 — parallel (different files)
- Phase 5 tasks can be drafted while Phase 3/4 are in progress

---

## Parallel Example: Phase 1

```
# Launch all three setup tasks in parallel:
Task T001: "Verify actions/checkout v4 SHA"
Task T002: "Verify rhysd/actionlint SHA"
Task T003: "Verify release-please-action v4.4.0 output key names"
```

## Parallel Example: Phase 5 (US2)

```
# Launch both consumer example files in parallel:
Task T012: "Create docs/semver-release/examples/consumer-workflow.yml"
Task T013: "Create docs/semver-release/examples/consumer-workflow-with-deploy.yml"
```

---

## Implementation Strategy

### MVP First (ship `v1.0.0`)

1. Phase 1: Setup — confirm SHAs
2. Phase 2: Foundational — `semver-release.yml` skeleton
3. Phase 3: US1 — pointer update logic
4. Phase 4: US4 — `release.yml` self-caller
5. Phase 7: CI Tests — SHA pins + dry-run fix
6. **Merge PR to `main`** → `release.yml` triggers → `v1.0.0` created automatically

### Incremental Delivery

1. MVP above → `v1.0.0` tagged; consumers can reference `@v1`
2. Phase 5 (US2): consumer docs + examples → `v1.1.0`
3. Phase 6 (US3): pointer docs → included in same PR or follow-up patch
4. Phase 8: Polish → patch release

### Single Developer Sequential Order

T001 → T002 → T003 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T017 → T018 → T019 → T020 → T012 → T013 → T014 → T015 → T016 → T021 → T022 → T023 → T024

---

## Notes

- [P] tasks = different files, no incomplete task dependencies
- [Story] label maps each task to its user story for traceability
- No test tasks generated — spec does not request TDD; `test-semver-release.yml` serves as the automated test harness
- `release_created` (singular) MUST be used in all conditionals — `releases_created` (plural) always returns `true` in release-please-action v4 (known bug)
- `release.yml` local path `uses: ./.github/workflows/semver-release.yml` requires no `@tag` suffix — resolves from the same commit as the caller
- Commit after each phase checkpoint for clean, bisectable git history
