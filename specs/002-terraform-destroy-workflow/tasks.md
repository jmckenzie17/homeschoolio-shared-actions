# Tasks: Terraform Destroy Workflow

**Input**: Design documents from `/specs/002-terraform-destroy-workflow/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅, quickstart.md ✅

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the two new workflow files at the correct paths, ready to be filled in.

- [x] T001 Create empty `.github/workflows/destroy.yml` with `name: Destroy` and a placeholder `on: workflow_call:` trigger
- [x] T002 [P] Create empty `.github/workflows/test-destroy.yml` with `name: Test destroy workflow` and a placeholder `on: pull_request:` trigger

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Build the reusable `destroy.yml` skeleton — `workflow_call` interface, all inputs/secrets/outputs, and the SHA-pinned tool-setup steps that all three user stories depend on. This phase produces a syntactically valid but not yet functional workflow.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [x] T003 Add `workflow_call` inputs block to `.github/workflows/destroy.yml` — `target-environment` (required string), `confirm-destroy` (required boolean), `opentofu-version` (string, default `"1.6.2"`), `terragrunt-version` (string, default `"0.56.3"`) per `contracts/workflow-call-contract.md`
- [x] T004 Add `workflow_call` secrets block to `.github/workflows/destroy.yml` — `azure-client-id` (required), `azure-tenant-id` (required), `azure-subscription-id` (required), `github-token` (required), `pg_admin_password` (optional) — mirroring `apply.yml`
- [x] T005 Add `workflow_call` outputs block to `.github/workflows/destroy.yml` — `destroyed-sha` wired to the destroy job output
- [x] T006 Add `jobs.destroy` job definition to `.github/workflows/destroy.yml` — `runs-on: ubuntu-latest`, `environment: ${{ inputs.target-environment }}`, `permissions: id-token: write / contents: read`, `outputs: destroyed-sha`
- [x] T007 Add SHA-pinned `actions/checkout` step to `jobs.destroy` in `.github/workflows/destroy.yml` — use same SHA as `apply.yml` (`34e114876b0b11c390a56381ad16ebd13914f8d5`)
- [x] T008 [P] Add SHA-pinned `opentofu/setup-opentofu` step to `jobs.destroy` in `.github/workflows/destroy.yml` — use same SHA as `apply.yml` (`9d84900f3238fab8cd84ce47d658d25dd008be2f`) with `tofu_version: ${{ inputs.opentofu-version }}`
- [x] T009 [P] Add Terragrunt binary-download install step to `jobs.destroy` in `.github/workflows/destroy.yml` — identical `curl` + `chmod` pattern from `apply.yml` using `${{ inputs.terragrunt-version }}`
- [x] T010 [P] Add SHA-pinned `actions/cache` provider-cache step to `jobs.destroy` in `.github/workflows/destroy.yml` — use same SHA as `apply.yml` (`0057852bfaa89a56745cba8c7296529d2fc39830`) and same cache key pattern
- [x] T011 Add `mkdir -p` plugin-cache-dir step to `jobs.destroy` in `.github/workflows/destroy.yml` — identical to `apply.yml`
- [x] T012 Add SHA-pinned `azure/login` step to `jobs.destroy` in `.github/workflows/destroy.yml` — use same SHA as `apply.yml` (`1384c340ab2dda50fed2bee3041d1d87018aa5e8`) with `client-id`, `tenant-id`, `subscription-id` from secrets

**Checkpoint**: `destroy.yml` is a valid YAML file with correct interface and tool-setup steps. `actionlint` should pass at this point.

---

## Phase 3: User Story 1 - Safe Targeted Destroy (Priority: P1) 🎯 MVP

**Goal**: A working destroy workflow that safely tears down a target environment after an explicit confirmation gate, and produces a job summary.

**Independent Test**: Trigger the workflow against a `dev` environment with `confirm-destroy: true`. Verify all resources are destroyed and the job summary shows the resource count. Then trigger with `confirm-destroy: false` (or omitted) and verify the job fails at the guard step before `init` runs.

### Implementation for User Story 1

- [x] T013 [US1] Add guard step to `jobs.destroy` in `.github/workflows/destroy.yml` — shell step that runs `if [ "${{ inputs.confirm-destroy }}" != "true" ]; then echo "::error::confirm-destroy must be true"; exit 1; fi` as the first step after checkout (before any infrastructure interaction)
- [x] T014 [US1] Add `Init` step to `jobs.destroy` in `.github/workflows/destroy.yml` — `cd environments/${{ inputs.target-environment }} && terragrunt run-all init --terragrunt-non-interactive -upgrade -no-color` with same `env` block as `apply.yml` (ARM vars + TF_PLUGIN_CACHE_DIR + TF_VAR_pg_admin_password)
- [x] T015 [US1] Add `Destroy` step (id: `destroy`) to `jobs.destroy` in `.github/workflows/destroy.yml` — `cd environments/${{ inputs.target-environment }} && terragrunt run-all destroy --terragrunt-non-interactive -auto-approve -no-color` with same `env` block; capture `echo "sha=${{ github.sha }}" >> "$GITHUB_OUTPUT"` on success
- [x] T016 [US1] Add job-summary step to `jobs.destroy` in `.github/workflows/destroy.yml` — `if: always()` step that greps Terragrunt destroy output for "Resources:" lines, sums destroyed counts, and writes a markdown table to `$GITHUB_STEP_SUMMARY` showing environment + counts + status
- [x] T017 [US1] Wire `destroyed-sha` job output in `.github/workflows/destroy.yml` — set `outputs.destroyed-sha: ${{ steps.destroy.outputs.sha }}` on the job

**Checkpoint**: User Story 1 complete. A caller can invoke `destroy.yml` with `confirm-destroy: true` and tear down a real environment. Calling with `confirm-destroy: false` fails at the guard step.

---

## Phase 4: User Story 2 - Destroy Mirrors Apply Configuration (Priority: P2)

**Goal**: Verify that default versions and secret names are identical to `apply.yml` so no additional consumer configuration is needed.

**Independent Test**: Call `destroy.yml` passing only `target-environment` and `confirm-destroy: true` (omit version inputs). Confirm the workflow installs OpenTofu 1.6.2 and Terragrunt 0.56.3 — the same defaults as `apply.yml`. Also verify all five secret names match `apply.yml` exactly.

### Implementation for User Story 2

- [x] T018 [US2] Audit `.github/workflows/destroy.yml` inputs against `apply.yml` — confirm `opentofu-version` default is `"1.6.2"`, `terragrunt-version` default is `"0.56.3"`, and all five secret names (`azure-client-id`, `azure-tenant-id`, `azure-subscription-id`, `github-token`, `pg_admin_password`) match exactly; fix any discrepancies
- [x] T019 [US2] Audit `env` blocks in Init and Destroy steps in `.github/workflows/destroy.yml` against `apply.yml` — confirm `ARM_USE_AZUREAD`, `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`, `TF_PLUGIN_CACHE_DIR`, `TF_VAR_pg_admin_password` are all present with identical mappings; fix any discrepancies

**Checkpoint**: User Story 2 complete. A consumer can copy their `apply.yml` caller and change only `destroy.yml` as the workflow path — no other changes needed.

---

## Phase 5: User Story 3 - Destroy Failure Surfaces Clearly (Priority: P3)

**Goal**: Ensure partial failures produce a job summary that identifies which roots succeeded and which failed, and that the job exits non-zero.

**Independent Test**: Simulate a destroy failure by pointing the workflow at an environment root where one resource has a deletion lock. Verify the job exits non-zero and the job summary identifies the failed root. Verify a successful root in the same run is also identified in the summary.

### Implementation for User Story 3

- [x] T020 [US3] Update the `Destroy` step in `.github/workflows/destroy.yml` — add `continue-on-error: false` (explicit) and ensure `run-all destroy` exit code propagates; verify the step `id: destroy` is set so subsequent steps can reference `steps.destroy.outcome`
- [x] T021 [US3] Update the job-summary step in `.github/workflows/destroy.yml` — enhance to parse per-root success/failure from Terragrunt `run-all` output; add a "Failed roots" section to the summary table if any root exits non-zero; keep `if: always()` so the summary writes even on failure

**Checkpoint**: User Story 3 complete. Partial destroy failures surface per-root in the job summary and the workflow exits non-zero.

---

## Phase 6: CI Test Workflow — `test-destroy.yml`

**Purpose**: Fulfill Constitution Principle V (tested before release) by adding a CI gate that runs on every PR to `main`.

- [x] T022 Add `on: pull_request: branches: [main]` trigger to `.github/workflows/test-destroy.yml`
- [x] T023 Add `lint` job to `.github/workflows/test-destroy.yml` — SHA-pinned `actions/checkout` step followed by SHA-pinned `rhysd/actionlint` step with `args: .github/workflows/destroy.yml`; use same SHAs as `test-semver-release.yml`
- [x] T024 [P] Add `sha-pinning` job to `.github/workflows/test-destroy.yml` — SHA-pinned `actions/checkout` step + shell step that greps `destroy.yml` for mutable tag references (`@v[0-9]+|@main|@master|@latest`) and exits 1 if found; identical pattern to `test-semver-release.yml` sha-pinning job

**Checkpoint**: `test-destroy.yml` is wired and will block any PR that introduces mutable action refs or YAML syntax errors in `destroy.yml`.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Documentation and final validation before the PR is merge-ready.

- [x] T025 Run `actionlint .github/workflows/destroy.yml` locally and fix any warnings
- [x] T026 [P] Run `actionlint .github/workflows/test-destroy.yml` locally and fix any warnings
- [x] T027 Validate all external action SHAs in `destroy.yml` against the SHAs used in `apply.yml` — confirm versions are consistent and no SHA drift has occurred
- [x] T028 [P] Verify `destroy.yml` has `permissions` at the job level (not workflow level) with only `id-token: write` and `contents: read` per the Security standards in the constitution

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately; T001 and T002 are independent [P]
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user story phases; T008/T009/T010 are parallel with each other after T007
- **User Story Phases (3–5)**: All depend on Phase 2 completion; US1 (Phase 3) must complete before US2/US3 audits are meaningful but US2/US3 are lightweight and can follow quickly
- **CI Test Workflow (Phase 6)**: Independent of Phase 3–5; can be written in parallel once Phase 2 is done
- **Polish (Phase 7)**: Depends on all phases complete

### User Story Dependencies

- **US1 (P1)**: Depends only on Phase 2 — implements the core destroy logic
- **US2 (P2)**: Depends on US1 — audits the completed workflow for apply-parity
- **US3 (P3)**: Depends on US1 — enhances failure surfacing in the same workflow

### Parallel Opportunities

- T001 and T002 (Phase 1) can run in parallel
- T008, T009, T010 (tool-setup steps, Phase 2) can run in parallel
- Phase 6 (`test-destroy.yml`) can be written in parallel with Phase 3–5
- T025 and T026 (Phase 7 linting) can run in parallel
- T027 and T028 (Phase 7 validation) can run in parallel

---

## Parallel Example: Phase 2 Tool Setup

```text
# After T007 (checkout step added), these three steps can be authored simultaneously:
T008 — Add opentofu/setup-opentofu step
T009 — Add Terragrunt binary-download step
T010 — Add actions/cache provider-cache step
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T002)
2. Complete Phase 2: Foundational (T003–T012)
3. Complete Phase 3: User Story 1 (T013–T017)
4. **STOP and VALIDATE**: Trigger destroy against `dev`, confirm full teardown + guard gate
5. Continue to Phase 4–7 once MVP is validated

### Incremental Delivery

1. Phase 1 + 2 → skeleton workflow with correct interface
2. Phase 3 → working destroy with confirmation gate (MVP)
3. Phase 4 → apply-parity audit (no new YAML, just verification)
4. Phase 5 → enhanced failure surfacing
5. Phase 6 → CI test gate
6. Phase 7 → polish and PR-ready

---

## Notes

- [P] tasks operate on different files or independent sections — no merge conflicts
- All external action SHAs must match those already in `apply.yml` unless a deliberate version upgrade is intended
- `confirm-destroy` is a `boolean` input in the `workflow_call` interface but arrives as the string `"true"` in shell — compare with string equality (`!= "true"`), not boolean
- The `environment:` key on the destroy job is what enables GitHub Environment protection rules for production — do not remove it
- Commit after each phase checkpoint to keep the branch history clean and reviewable
