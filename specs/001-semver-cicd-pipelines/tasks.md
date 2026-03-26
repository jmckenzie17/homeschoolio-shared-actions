---

description: "Task list for 001-semver-cicd-pipelines"
---

# Tasks: Semantic Versioning CI/CD Pipelines

**Input**: Design documents from `/specs/001-semver-cicd-pipelines/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Exact file paths included in all descriptions

## Path Conventions

- Reusable workflow: `.github/workflows/semver-release.yml`
- CI test workflow: `.github/workflows/test-semver-release.yml`
- Consumer docs: `docs/semver-release/README.md`

---

## Phase 1: Setup

**Purpose**: Repository structure and tooling initialization.

- [ ] T001 Create `.github/workflows/` directory structure at repo root
- [ ] T002 [P] Create `docs/semver-release/` directory at repo root
- [ ] T003 [P] Verify `CLAUDE.md` reflects correct tech stack from plan.md (already created by update-agent-context script — review and confirm accuracy)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure shared by all user stories — the reusable workflow
skeleton and permissions model that US2 and US3 depend on.

**⚠️ CRITICAL**: US2 (consumer adoption) and US3 (major pointer) cannot be
implemented or tested until this phase is complete.

- [ ] T004 Create the reusable workflow skeleton in `.github/workflows/semver-release.yml` with `on: workflow_call:` trigger, all three inputs (`release-branch`, `tag-prefix`, `config-file`) with defaults, all three outputs (`release-created`, `tag-name`, `major-tag`), and `permissions: contents: write, pull-requests: write` — no job logic yet, just the interface scaffolding matching `contracts/semver-release-workflow.md`
- [ ] T005 [P] Add the `release-please` job to `.github/workflows/semver-release.yml`: single step using `googleapis/release-please-action` pinned to commit SHA `16a9c90856f42705d54a6fda1823352bdc62cf38`, passing `token: ${{ secrets.GITHUB_TOKEN }}`, `release-type: simple`, `config-file: ${{ inputs.config-file }}`, and `target-branch: ${{ inputs.release-branch }}`
- [ ] T006 Wire the `release-created` and `tag-name` outputs in `.github/workflows/semver-release.yml` from the release-please step's outputs — use `release_created` (singular, NOT `releases_created`) as documented in research.md Decision 1

**Checkpoint**: Reusable workflow interface + release-please job complete — US1 can now begin.

---

## Phase 3: User Story 1 — Automated Release Versioning on Merge (Priority: P1) 🎯 MVP

**Goal**: A merge to `main` with conventional commits automatically produces the
correct SemVer tag and GitHub Release with no manual steps.

**Independent Test**: Run quickstart.md Scenarios 1–5 against a test consumer repo
referencing this workflow.

### Implementation for User Story 1

- [ ] T007 [US1] Add the `checkout` step to the release-please job in `.github/workflows/semver-release.yml` using `actions/checkout` pinned to its current v4 commit SHA (look up at https://github.com/actions/checkout/releases and use the SHA of the latest v4.x tag)
- [ ] T008 [US1] Configure `tag-prefix` input to be forwarded to the release-please step in `.github/workflows/semver-release.yml` via the `tag-prefix` parameter so consumers can override the default `v` prefix
- [ ] T009 [US1] Add conditional logic in `.github/workflows/semver-release.yml` so that the major-pointer update step only runs when `steps.release-please.outputs.release_created == 'true'` — prevents errors on no-release runs
- [ ] T010 [US1] Add the major version pointer update step to `.github/workflows/semver-release.yml`: a `run:` shell step (no external action) that extracts `MAJOR=${GITHUB_REF#refs/tags/}` then `MAJOR=${MAJOR%%.*}`, configures git identity as `github-actions[bot]`, runs `git tag -fa "${MAJOR}" -m "Update ${MAJOR} tag to ${VERSION}"`, and force-pushes via `git push origin "${MAJOR}" --force` — this satisfies FR-006 and US3
- [ ] T011 [US1] Set the `major-tag` output in `.github/workflows/semver-release.yml` to the value of `MAJOR` extracted in T010, so callers can access it via `needs.<job>.outputs.major-tag`
- [ ] T012 [US1] Validate the complete `.github/workflows/semver-release.yml` against the contract in `contracts/semver-release-workflow.md` — confirm all inputs, outputs, permissions, and behavior described in the contract are correctly implemented

**Checkpoint**: US1 complete. Merge a `fix:` commit in a test consumer repo and verify a `v1.0.0` (or next PATCH) tag and GitHub Release are created. Run quickstart.md Scenarios 1–5.

---

## Phase 4: User Story 2 — Consumer Repo Adopts the Reusable Workflow (Priority: P2)

**Goal**: Any homeschoolio repo can adopt automated semver with a single `uses:` line
and a config file — total under 20 lines of new files.

**Independent Test**: Run quickstart.md Scenario 6 — confirm adoption requires only
2 files and the workflow file is under 20 lines.

### Implementation for User Story 2

- [ ] T013 [US2] Create `docs/semver-release/README.md` with the following sections: (1) Overview — what the workflow does, (2) Quick Start — the minimal consumer workflow YAML (< 20 lines as required by SC-001), (3) Inputs table (release-branch, tag-prefix, config-file with types, defaults, descriptions), (4) Outputs table (release-created, tag-name, major-tag), (5) Required consumer file: `.release-please-config.json` minimum content, (6) Versioning — pin to `@v1` for automatic patch/minor updates or `@v1.2.3` for exact pinning, (7) Known Limitations — Ruleset/tag-protection workaround with GitHub App token pattern
- [ ] T014 [US2] Add an example consumer workflow file at `docs/semver-release/examples/consumer-workflow.yml` showing the complete minimal adoption pattern with `uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1` and `secrets: inherit`
- [ ] T015 [US2] Add an example consumer workflow file at `docs/semver-release/examples/consumer-workflow-with-deploy.yml` showing how to use the `release-created` and `tag-name` outputs in a downstream deploy job with `if: needs.release.outputs.release-created == 'true'`
- [ ] T016 [US2] Verify the minimal consumer workflow in `docs/semver-release/examples/consumer-workflow.yml` is under 20 lines (SC-001) and requires no other files beyond `.release-please-config.json`

**Checkpoint**: US2 complete. A developer can follow the README and adopt the workflow in a new repo with 2 files. Verify Scenario 6.

---

## Phase 5: User Story 3 — Moving Major Version Pointer Maintained (Priority: P3)

**Goal**: After each release, the `v1` (or `v2`, etc.) pointer tag is atomically
updated in the same pipeline run — consumers on `@v1` auto-receive patches.

**Independent Test**: Run quickstart.md Scenario 7 — confirm `v1` SHA matches `v1.0.x` SHA after a release.

**Note**: The core pointer-update logic was already implemented in T010 (US1 phase)
because it is part of the reusable workflow. This phase adds robustness and
documentation for the pointer behavior.

### Implementation for User Story 3

- [ ] T017 [US3] Add explicit handling in `.github/workflows/semver-release.yml` for the MAJOR version bump case: when `steps.release-please.outputs.major_tag` indicates a new major version (e.g., `v2.0.0`), ensure the pointer step creates a NEW pointer tag (`v2`) rather than updating the previous major's pointer (`v1`) — verify by checking that `MAJOR` extraction in T010 naturally handles this correctly (it should, since `v2.0.0 → v2` ≠ `v1`)
- [ ] T018 [US3] Add a `major-pointer-updated` section to `docs/semver-release/README.md` explaining the moving pointer behavior: what `v1` means, when it moves, when a new pointer is created (on MAJOR bump), and the gotcha that force-pushing via `GITHUB_TOKEN` does NOT trigger downstream `on: push: tags:` workflows

**Checkpoint**: US3 complete. Run Scenario 7 and Scenario 3 (MAJOR bump) to verify `v1` freezes and `v2` is created.

---

## Phase 6: CI Test Workflow (Constitution Principle V)

**Purpose**: Every action/workflow MUST have automated tests passing in CI before
release. This phase delivers the test workflow.

- [ ] T019 Create `.github/workflows/test-semver-release.yml` with `on: [pull_request]` targeting `main`, a single job `validate` running on `ubuntu-latest`, a step using `actionlint` (look up the `rhysd/actionlint` action SHA for the latest release) to lint `.github/workflows/semver-release.yml` and fail on any errors or warnings
- [ ] T020 [P] Add a second job `dry-run` to `.github/workflows/test-semver-release.yml` that checks out the repo and runs `npx --yes release-please release-pr --dry-run --repo-url=homeschoolio/homeschoolio-shared-actions --token=${{ secrets.GITHUB_TOKEN }} --release-type=simple` to verify release-please can parse the repo without errors
- [ ] T021 [P] Add a SHA-pinning validation step to `.github/workflows/test-semver-release.yml` that greps `.github/workflows/semver-release.yml` for any `uses:` lines referencing a mutable tag (pattern: `uses:.*@(v[0-9]+|main|master|latest)$`) and fails the job if any matches are found — satisfies Constitution Principle III at CI time

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final consistency checks, README at repo root, and release readiness.

- [ ] T022 Create a top-level `README.md` at repo root describing this repository's purpose (shared GitHub Actions for homeschoolio), linking to `docs/semver-release/README.md`, and explaining the SemVer + SHA-pinning policy for contributors
- [ ] T023 [P] Run quickstart.md Scenario 8 (SHA pinning check) and Scenario 9 (permissions check) against the final `.github/workflows/semver-release.yml` to confirm no mutable tag references and no excess permissions
- [ ] T024 [P] Run quickstart.md Scenario 10 — open a PR to `main` and verify the `test-semver-release` CI workflow passes (green checks for `validate`, `dry-run`, and SHA-pinning jobs)
- [ ] T025 Verify the complete feature against `specs/001-semver-cicd-pipelines/quickstart.md` Scenarios 1–10 and confirm all pass before tagging the first release

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Phase 2 — core release logic
- **US2 (Phase 4)**: Depends on Phase 2; US1 should be complete first so the README documents tested behavior
- **US3 (Phase 5)**: Core logic already in US1 (T010); this phase adds robustness + docs
- **CI Test Workflow (Phase 6)**: Can begin after Phase 2 is complete (T004–T006 define the interface being tested)
- **Polish (Phase 7)**: Depends on all prior phases

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational — no story dependencies
- **US2 (P2)**: Can start after Foundational — independent; benefits from US1 being done first
- **US3 (P3)**: Core logic delivered in US1 (T010) — Phase 5 is documentation/hardening only

### Within Each Phase

- T004 → T005 → T006 (sequential: scaffold → add job → wire outputs)
- T007–T012 can be worked through sequentially (all in one file)
- T013–T016 are independent of each other — can run in parallel
- T019 → T020, T021 (T019 creates the file; T020/T021 add jobs to it)
- T022–T025 are independent polish tasks

### Parallel Opportunities

- Phase 1: T002 and T003 can run in parallel with T001
- Phase 4: T013, T014, T015, T016 can all run in parallel (different files)
- Phase 6: T020 and T021 can run in parallel after T019
- Phase 7: T022, T023, T024 can run in parallel

---

## Parallel Execution Examples

### Phase 4 (US2 — documentation)

```bash
# Launch in parallel:
Task: "Create docs/semver-release/README.md with all sections"             # T013
Task: "Create docs/semver-release/examples/consumer-workflow.yml"          # T014
Task: "Create docs/semver-release/examples/consumer-workflow-with-deploy.yml" # T015
```

### Phase 6 (CI test workflow jobs)

```bash
# After T019 (test workflow file created):
Task: "Add dry-run job to .github/workflows/test-semver-release.yml"       # T020
Task: "Add SHA-pinning validation job to .github/workflows/test-semver-release.yml" # T021
```

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (T004–T006)
3. Complete Phase 3: US1 (T007–T012)
4. **STOP AND VALIDATE**: Run quickstart.md Scenarios 1–5 against a test consumer repo
5. If passing: proceed to Phase 6 (CI tests) before merging

### Incremental Delivery

1. Setup + Foundational → workflow interface defined
2. US1 → core release automation working; validate with quickstart
3. US2 → consumer docs; validate Scenario 6 (< 20 lines)
4. US3 → major pointer hardening + docs; validate Scenarios 3 and 7
5. CI test workflow → all PR checks green
6. Polish → repo README, final validation pass

### Notes

- [P] tasks = different files, no blocking dependency on an incomplete peer task
- [Story] label maps each task to its user story for traceability
- No source code files in this project — all deliverables are YAML workflow files and Markdown docs
- SHA values for `actions/checkout` and `rhysd/actionlint` must be looked up at implementation time from their respective GitHub releases pages
- The `googleapis/release-please-action` SHA is already resolved: `16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0)
- Commit after each phase or logical group; each phase is independently deployable
- Run quickstart.md Scenario 10 (CI check) before every merge to `main`
