<!--
SYNC IMPACT REPORT
==================
Version change: [unversioned template] → 1.0.0
Bump rationale: MAJOR — initial ratification; all placeholders replaced with concrete values.

Modified principles: N/A (first adoption)

Added sections:
  - Core Principles (5 principles defined)
  - Security & Action Trust Standards
  - Release & Contribution Workflow
  - Governance

Removed sections: N/A

Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Constitution Check section is generic;
     gates align with principles below (no edits required at this time).
  ✅ .specify/templates/spec-template.md — structure compatible; no principle-driven
     mandatory sections added beyond what the template already covers.
  ✅ .specify/templates/tasks-template.md — task categories (setup, foundational,
     user story phases) align with workflow principles; no changes required.
  ✅ No commands/ directory exists yet; nothing to update.

Deferred TODOs:
  - TODO(RATIFICATION_DATE): Using today's date (2026-03-26) as the ratification date
    because no prior adoption date exists. Update if the project was informally started
    on an earlier date.
-->

# homeschoolio-shared-actions Constitution

## Core Principles

### I. Reusable-Action-First

Every deliverable in this repository MUST be a reusable GitHub Actions action or
reusable workflow. Standalone scripts, libraries, or utilities are only permitted
when they directly support an action or workflow defined in this repo. Each action
or reusable workflow MUST be:

- Self-contained within its own directory (e.g., `.github/actions/<name>/`).
- Independently versioned and testable in isolation.
- Accompanied by a `README.md` describing inputs, outputs, and usage examples.

**Rationale**: Downstream homeschoolio projects consume this repo by reference. A
single, unclear action boundary forces consumers to understand the entire repo
rather than just the action they need.

### II. Semantic Versioning & Stability Guarantees

This repository MUST follow [Semantic Versioning 2.0.0](https://semver.org) for all
releases published to consuming projects.

- MAJOR version bump: any breaking change to an action's inputs, outputs, or
  runtime behavior that requires consumer updates.
- MINOR version bump: backward-compatible additions (new optional inputs, new
  outputs, new actions/workflows).
- PATCH version bump: bug fixes and non-breaking internal changes.

Releases MUST be tagged (e.g., `v1.2.3`) and major version tags (e.g., `v1`) MUST
be kept as moving pointers so consumers pinning to `@v1` receive patch/minor
updates automatically without a spec change on their end.

**Rationale**: GitHub Actions consumers pin by tag. Without a disciplined versioning
policy, a merge to `main` silently breaks every downstream project.

### III. Trusted & Auditable Third-Party Actions

Any external GitHub Action or reusable workflow referenced within this repo MUST
meet ALL of the following criteria:

- Sourced from a widely-adopted, well-maintained action (strong star count, active
  maintenance, large community usage).
- Pinned to a specific commit SHA in all workflow files (not a mutable tag like
  `@v3` or `@main`).
- Free of known critical or high security vulnerabilities at the time of adoption;
  re-evaluated on each dependency update.
- From an official publisher or a publisher with a verifiable, reputable track
  record (e.g., `actions/`, `github/`, verified org publishers).

**Rationale**: Supply chain attacks via compromised Actions are a real and documented
threat. Mutable tags allow a compromised upstream to silently execute arbitrary code
in every downstream pipeline.

### IV. Minimal Surface Area (Simplicity)

Each action MUST do one thing well. Avoid bundling unrelated concerns into a single
action. Prefer composition of small, focused actions over monolithic ones.

- Inputs MUST be limited to what is actually needed; no speculative or future-use
  parameters.
- Outputs MUST be documented and stable once published.
- Complexity (e.g., multi-job orchestration, conditional branching) MUST be
  justified in the action's `README.md`.

**Rationale**: Small surface areas are easier to audit, test, and reason about —
critical for security-sensitive CI/CD primitives shared across many projects.

### V. Tested Before Release

Every action or reusable workflow MUST have at least one automated test (e.g., a
workflow under `.github/workflows/test-*.yml`) that exercises its primary use case
before a release tag is created. Tests MUST:

- Run on every pull request targeting `main`.
- Cover the happy path and at least one known failure/error path.
- Pass in CI before the PR is eligible for merge.

**Rationale**: An untested shared action that silently fails will break multiple
downstream projects simultaneously, making the blast radius much larger than a
failure in a single repo.

## Security & Action Trust Standards

All changes touching action definitions, workflow files, or third-party action
references MUST be reviewed with the following checks:

- No use of `pull_request_target` with untrusted code execution unless explicitly
  justified and locked down.
- Secrets MUST never be logged or echoed; use `add-mask` for any dynamically
  derived sensitive values.
- Permissions in workflow files MUST follow the principle of least privilege;
  declare only the scopes required.
- External actions MUST be re-evaluated for security advisories when updating to a
  new commit SHA.

## Release & Contribution Workflow

- All work MUST be done on a feature branch; direct pushes to `main` are
  prohibited.
- Pull requests MUST pass all CI checks (tests + linting) before merge.
- Release tags MUST be created from `main` only after a successful CI run.
- The major version moving pointer (e.g., `v1`) MUST be force-updated to the new
  release commit after each PATCH or MINOR release on that major line.
- The `CHANGELOG.md` (or GitHub Release notes) MUST be updated with every release,
  describing what changed and whether consumers need to take action.

## Governance

This constitution supersedes all informal practices and ad-hoc decisions. It
applies to every action, workflow, script, and documentation artifact in this
repository.

**Amendment procedure**: Any amendment requires a pull request that updates this
file, increments `CONSTITUTION_VERSION` per the semantic versioning rules described
above, and receives at least one approving review before merge.

**Compliance review**: Every pull request that adds or modifies an action or
workflow file MUST verify compliance with Principles I–V and the Security &
Action Trust Standards section. The `Constitution Check` gate in
`.specify/templates/plan-template.md` is the canonical checklist for this review.

**Versioning policy**: Constitution versions follow the same MAJOR.MINOR.PATCH
semantics described in Principle II, applied to governance rather than code.

---

**Version**: 1.0.0 | **Ratified**: 2026-03-26 | **Last Amended**: 2026-03-26
