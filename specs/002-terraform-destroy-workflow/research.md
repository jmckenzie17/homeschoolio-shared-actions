# Research: Terraform Destroy Workflow

**Date**: 2026-03-30 | **Feature**: 002-terraform-destroy-workflow

## Decision 1: Confirmation Gate Mechanism

**Decision**: Use a required boolean input (`confirm-destroy: true`) that must be explicitly set by the caller. The job has a guard step that fails fast if the input is not `"true"`. For protected environments (e.g., `production`), GitHub Environment protection rules (required reviewers) provide an additional gate at the infrastructure level.

**Rationale**: A boolean input gate is the simplest mechanism that works within `workflow_call` semantics — there is no native "pause for approval" inside a reusable workflow job. GitHub Environment protection rules are the correct place to enforce human review for high-stakes environments, but they are a consumer-side concern. The required input ensures the caller must consciously opt in even for unprotected environments (e.g., `dev`).

**Alternatives considered**:
- `workflow_dispatch` with environment protection rules only — rejected because `destroy.yml` is a `workflow_call` reusable workflow, not a dispatch target; callers control when it fires.
- Separate "approval" job using `environment:` on the destroy job — viable for `workflow_call` with environment protection rules, but adds coupling to GitHub-side environment configuration that consumers may not have set up; input gate is self-contained.
- GitHub Deployments approval API — overkill for this use case; only applicable to `workflow_dispatch` or deployment events.

## Decision 2: Destroy Command — `run-all destroy` vs. per-root loop

**Decision**: Use `terragrunt run-all destroy --terragrunt-non-interactive -auto-approve -no-color` in the target environment directory, mirroring the apply step exactly. Terragrunt's `run-all` handles dependency-ordered teardown (destroys dependents before dependencies).

**Rationale**: The `apply.yml` uses `run-all apply` for the same reason — it handles multi-module environments without manual ordering. Using `run-all destroy` keeps the destroy workflow structurally identical to apply, which lowers cognitive overhead and reduces the risk of ordering mistakes.

**Alternatives considered**:
- Per-root loop (detect roots, iterate) — this is what `plan.yml` does for planning, where partial failure is acceptable. For destroy, partial failure is always an error, so a single `run-all` invocation with a fail-fast posture is more appropriate.
- `tofu destroy` directly without Terragrunt — rejected; the apply workflow delegates to Terragrunt for orchestration; destroy must match.

## Decision 3: Job Summary Content

**Decision**: After destroy completes, emit a job summary with: (1) environment name, (2) number of resources destroyed (parsed from Terragrunt output via `grep "Destroy complete"`), and (3) overall status (success/failure).

**Rationale**: The spec requires a summary listing resources destroyed per root. Terragrunt `run-all destroy` outputs "Destroy complete! Resources: N destroyed." per module; a grep-and-sum approach is sufficient without requiring JSON plan output. This matches the lightweight summary approach used in `apply.yml`.

**Alternatives considered**:
- Pre-generate a destroy plan (JSON), upload as artifact, then apply it — adds complexity and an extra job; destroy plans go stale quickly and the confirmation gate already guards against accidents.
- Parse Azure provider output for resource-level detail — too provider-specific; the workflow must remain provider-agnostic at the summary level.

## Decision 4: Test Workflow Coverage

**Decision**: `test-destroy.yml` runs on every PR to `main` with two jobs: (1) `lint` — runs `actionlint` on `destroy.yml`; (2) `sha-pinning` — greps `destroy.yml` for mutable tag references and fails if found. No live Azure destroy is performed in CI (same posture as `test-semver-release.yml`, which also avoids live side effects).

**Rationale**: A live destroy in CI would require real infrastructure and real Azure credentials — impractical for a PR gate. Lint + SHA-pin verification provides meaningful static quality gates without side effects, consistent with how `test-semver-release.yml` tests `semver-release.yml`.

**Alternatives considered**:
- E2E test against ephemeral `dev` environment — desirable long-term but out of scope for this feature; noted as a future enhancement in quickstart.md.
- Dry-run destroy (`tofu plan -destroy`) — requires backend connectivity and Azure credentials in CI; adds secrets management complexity to the shared-actions repo itself, which currently avoids this.

## Decision 5: Output — `destroyed-sha`

**Decision**: Emit a `destroyed-sha` output (the commit SHA that was destroyed) mirroring the `applied-sha` output in `apply.yml`, to give callers a record of the state that was torn down.

**Rationale**: Symmetric with `apply.yml`; enables consumers to log/track the exact SHA that was in effect when infrastructure was destroyed. Low cost, high traceability value.

**Alternatives considered**:
- No output — simpler, but callers lose the ability to correlate the destroy event with the source commit.
