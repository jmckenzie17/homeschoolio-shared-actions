# Feature Specification: Semantic Versioning CI/CD Pipelines

**Feature Branch**: `001-semver-cicd-pipelines`
**Created**: 2026-03-26
**Status**: Draft
**Input**: User description: "implement ci cd pipelines that use semantic versioning. the templates used for semantic versioning should also be available to other repos to use."

## Clarifications

### Session 2026-03-26

- Q: Does this repo (`homeschoolio-shared-actions`) use the semver workflow to version itself? → A: Yes.
- Q: How does this repo reference its own reusable workflow to avoid a bootstrap chicken-and-egg problem? → A: Option B — a local caller workflow (`.github/workflows/release.yml`) references the reusable workflow via local path `uses: ./.github/workflows/semver-release.yml`, requiring no prior published release.
- Q: How should concurrent pipeline runs (two PRs merged in rapid succession) be handled? → A: Option C — enforce serialization via a `concurrency` group on `release.yml` with `cancel-in-progress: false` (queue, not cancel).
- Q: Should the reusable workflow accept an optional PAT token input, or rely solely on `GITHUB_TOKEN`? → A: Option A — `GITHUB_TOKEN` with `contents: write` only; no PAT input; consumers must not have tag protection rules that block force-push on pointer tags.
- Q: How are release pipeline failures surfaced to operators/consumers? → A: Option A — rely on GitHub's built-in failure email/notification system; no custom alerting step in the workflow.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automated Release Versioning on Merge (Priority: P1)

A developer on a homeschoolio project merges a pull request to `main`. The CI/CD
pipeline automatically determines the correct next semantic version (MAJOR, MINOR,
or PATCH) based on the nature of the merged changes, creates a version tag, and
publishes a GitHub Release — without any manual tagging or version-file editing.

**Why this priority**: This is the core value of the feature. Without automated
versioning on merge, teams must manually tag every release, which is error-prone
and creates bottlenecks. All other stories depend on this working correctly.

**Independent Test**: Merge a PR with a conventional commit message to a test
repository that references the reusable workflow; verify that a new SemVer tag and
GitHub Release are created automatically with the correct version number.

**Acceptance Scenarios**:

1. **Given** a PR with only `fix:` commits is merged to `main`, **When** the
   pipeline runs, **Then** a PATCH version tag (e.g., `v1.0.1`) and GitHub Release
   are created automatically.
2. **Given** a PR with a `feat:` commit is merged to `main`, **When** the pipeline
   runs, **Then** a MINOR version tag (e.g., `v1.1.0`) is created.
3. **Given** a PR with a `feat!:` or `BREAKING CHANGE:` commit is merged to `main`,
   **When** the pipeline runs, **Then** a MAJOR version tag (e.g., `v2.0.0`) is
   created.
4. **Given** a PR with only `chore:` or `docs:` commits is merged, **When** the
   pipeline runs, **Then** no new version tag is created (no-release commits are
   ignored).
5. **Given** no prior version tag exists in the repo, **When** the first release
   pipeline runs, **Then** versioning starts at `v1.0.0`.

---

### User Story 2 - Consumer Repo Adopts the Reusable Workflow (Priority: P2)

A developer on another homeschoolio project (e.g., `homeschoolio-web`) wants to
adopt automated semantic versioning without copying workflow files. They reference
the reusable workflow from this repo with a single `uses:` line and get the full
versioning behavior.

**Why this priority**: Reusability is a stated requirement. If workflows cannot be
consumed by other repos, the feature delivers value only to this repo, not the
ecosystem.

**Independent Test**: Create a minimal test repository that references the reusable
workflow via `uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1`;
merge a commit and verify the release pipeline runs correctly in the consumer repo.

**Acceptance Scenarios**:

1. **Given** a consumer repo adds a workflow file with a single `uses:` reference
   to the reusable semver workflow, **When** a PR is merged to `main` in that repo,
   **Then** the semver release pipeline runs and creates the correct version tag.
2. **Given** the consumer repo pins to a major version pointer (e.g., `@v1`),
   **When** a PATCH update is released in this repo, **Then** the consumer receives
   the fix automatically without changing their workflow file.
3. **Given** a consumer passes optional inputs (e.g., custom release branch name),
   **When** the workflow runs, **Then** those inputs override the defaults correctly.

---

### User Story 3 - Moving Major Version Pointer Maintained (Priority: P3)

After each release of this repo itself, the major version pointer tag (e.g., `v1`)
is automatically updated to point to the latest release on that major line, so
consumers pinning to `@v1` always get the latest stable patch/minor without
changing their workflow files.

**Why this priority**: Required by the constitution's versioning policy but is
independent of the core release behavior. Consumers can still use exact tags
(`@v1.2.3`) without this story, making it lower priority.

**Independent Test**: After a PATCH release (e.g., `v1.0.0` → `v1.0.1`), verify
that the `v1` tag now points to the same commit as `v1.0.1`.

**Acceptance Scenarios**:

1. **Given** a new `v1.2.3` tag is created, **When** the release pipeline completes,
   **Then** the `v1` pointer tag is force-updated to reference the same commit.
2. **Given** a MAJOR bump creates `v2.0.0`, **When** the release pipeline completes,
   **Then** a new `v2` pointer is created and `v1` is left unchanged.

---

### User Story 4 - This Repo Versions Itself (Priority: P1)

When a PR is merged to `main` in `homeschoolio-shared-actions`, the repo's own
release pipeline runs automatically and creates the correct SemVer tag and GitHub
Release — using the reusable workflow defined in this same repo via a local path
reference, with no prior published version required.

**Why this priority**: The repo must eat its own cooking. Consumers reference this
repo by version tag; if this repo doesn't self-version, there are no tags to
reference.

**Independent Test**: Merge a conventional-commit PR to `main` in this repo; verify
that `.github/workflows/release.yml` triggers, calls `semver-release.yml` via local
path, and produces a new version tag and GitHub Release.

**Acceptance Scenarios**:

1. **Given** a PR with a `feat:` commit is merged to `main` in this repo, **When**
   the `release.yml` caller workflow runs, **Then** a MINOR version tag and GitHub
   Release are created via the local reusable workflow.
2. **Given** no prior version tag exists in this repo, **When** the first
   `release.yml` run completes, **Then** versioning starts at `v1.0.0` with no
   manual bootstrap step required.

---

### Edge Cases

- **Concurrent merges**: Two PRs merged in rapid succession are serialized by a
  `concurrency` group (`cancel-in-progress: false`) on `release.yml`; runs queue
  rather than overlap, so each gets a release attempt in order.
- How does the pipeline behave if the most recent commit message does not follow
  conventional commit format (no recognizable prefix)?
- What happens if a human manually creates a version tag out-of-sequence — does the
  pipeline correctly base the next version off the latest tag?
- How does initial versioning work in a brand-new repo with zero existing tags?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The semver release workflow MUST automatically calculate the next
  version from the highest existing version tag and the conventional commit messages
  in the merged PR.
- **FR-002**: The workflow MUST support the full conventional commits specification
  for bump determination: `fix:` → PATCH, `feat:` → MINOR, breaking change footer
  or `!` → MAJOR.
- **FR-003**: The workflow MUST create a Git tag and a GitHub Release on every
  version bump, with auto-generated release notes summarizing the included commits.
- **FR-004**: The workflow MUST be a reusable workflow (`workflow_call` trigger) so
  other repositories can reference it with a single `uses:` line.
- **FR-005**: The workflow MUST expose documented inputs allowing consumers to
  override defaults such as release branch name and tag prefix.
- **FR-006**: The workflow MUST update the major version pointer tag (e.g., `v1`)
  after every PATCH or MINOR release on that major line.
- **FR-007**: The workflow MUST skip version creation (no tag, no release) when all
  commits since the last tag carry non-release prefixes (e.g., `chore:`, `docs:`,
  `style:`).
- **FR-008**: The workflow MUST work correctly in a repository with no prior version
  tags, starting at `v1.0.0`.
- **FR-009**: All third-party actions used within the workflow MUST be pinned to
  specific commit SHAs (not mutable tags).
- **FR-010**: The reusable workflow MUST declare `contents: write` and
  `pull-requests: write` permissions and no broader scopes. The self-release caller
  workflow (`release.yml`) and consumer workflows MUST also declare these same
  permissions on the calling job (GitHub Actions `workflow_call` callers control the
  `GITHUB_TOKEN` scope ceiling; the reusable workflow cannot self-elevate).
  Authentication MUST use `GITHUB_TOKEN` exclusively; no PAT input is accepted or
  required.
- **FR-012**: The self-release caller workflow (`release.yml`) MUST declare a
  `concurrency` group (`group: release`, `cancel-in-progress: false`) to serialize
  runs. Note: this implements "last pending wins" — at most one run executes and one
  is pending at a time; a third trigger replaces the pending run. This is acceptable
  because release-please is idempotent and calculates the correct version from all
  accumulated commits when it runs.
- **FR-011**: This repo MUST version itself using the reusable workflow via a local
  caller workflow (`.github/workflows/release.yml`) that references it with
  `uses: ./.github/workflows/semver-release.yml` (local path), avoiding any
  dependency on a prior published release of this repo.

### Key Entities

- **Reusable Workflow**: The `.github/workflows/semver-release.yml` file in this
  repo; defines the `workflow_call` trigger, inputs, and all versioning logic.
- **Version Tag**: A Git tag in `vMAJOR.MINOR.PATCH` format created on the release
  commit; the primary artifact consumed by downstream projects.
- **Major Pointer Tag**: A mutable Git tag (e.g., `v1`) that always points to the
  latest release on a given major version line.
- **GitHub Release**: A GitHub Release record attached to the version tag, including
  auto-generated notes describing changes since the previous release.
- **Consumer Workflow**: A minimal workflow file in a consuming repo that references
  this repo's reusable workflow via `uses:`.
- **Self-Release Caller Workflow**: `.github/workflows/release.yml` in this repo;
  triggers on `push` to `main` and calls the reusable workflow via local path
  (`uses: ./.github/workflows/semver-release.yml`) so this repo versions itself
  without requiring a prior published release.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can enable automated semantic versioning in a new
  homeschoolio repo by adding fewer than 20 lines to a workflow file.
- **SC-002**: The correct version bump type is produced for 100% of PRs that use
  standard conventional commit prefixes.
- **SC-003**: The end-to-end time from PR merge to published GitHub Release is under
  3 minutes under normal conditions.
- **SC-004**: Zero manual steps are required to create a version tag or GitHub
  Release once the reusable workflow is configured in a consumer repo.
- **SC-005**: The major version pointer tag is updated within the same pipeline run
  that creates the version tag, with no separate manual step.

## Assumptions

- Consuming repositories use conventional commits (or a compatible commit message
  convention) as their standard commit format.
- All homeschoolio repositories target GitHub (not GitLab or Bitbucket), so
  GitHub-native features (Actions, Releases, tags) are available.
- The release branch in consumer repos defaults to `main`; consumers can override
  via a workflow input if they use a different default branch name.
- This feature covers release tagging and changelog generation only; it does not
  include building, publishing packages, or deploying artifacts — those are
  separate concerns for future actions in this repo.
- Repositories that consume this workflow already have at least one commit on `main`
  before the first pipeline run.
- Consumer repositories do not enforce tag protection rules that would block
  `GITHUB_TOKEN` from force-pushing the major pointer tag (e.g., `v1`).
- Release pipeline failure visibility relies on GitHub's built-in notification
  system (email to commit author and repo watchers); no custom alerting is
  implemented within the workflow.
