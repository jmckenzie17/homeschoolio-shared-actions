# semver-release Reusable Workflow

Automates semantic versioning for any homeschoolio repository. On each push to
`main`, it parses [Conventional Commits](https://www.conventionalcommits.org/),
calculates the next SemVer, creates a Git tag, publishes a GitHub Release, and
updates the moving major version pointer (e.g. `v1`).

Powered by [`cycjimmy/semantic-release-action`](https://github.com/cycjimmy/semantic-release-action)
v6.0.0 (SHA-pinned). Releases are published **directly** on push — no intermediate
Release PR required.

---

## Quick Start

Create one file in your consumer repo:

**`.github/workflows/release.yml`**:
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

That's it. No config files required — the workflow provides built-in default plugin
configuration for standard single-package repos.

---

## Prerequisites

Only one one-time setup step is required:

### Enable `contents: write` on the calling job

Declare `permissions: contents: write` on the calling job (shown in the Quick Start
above). GitHub Actions `workflow_call` callers control the `GITHUB_TOKEN` permission
scope — the reusable workflow cannot self-elevate.

> **Note**: `pull-requests: write` is **not** needed. semantic-release publishes
> releases directly without creating a Release PR.

---

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `release-branch` | `string` | `"main"` | Branch that triggers release evaluation on push. Override if your default branch is not `main`. |
| `tag-prefix` | `string` | `"v"` | Prefix prepended to the SemVer number. Produces tags like `v1.2.3`. Treat as immutable once set — changing it causes semantic-release to lose track of prior versions. |

---

## Outputs

Available to calling workflows via `${{ needs.<job-id>.outputs.<name> }}`.

| Output | Type | Description | Empty when |
|--------|------|-------------|------------|
| `release-created` | `string` | `"true"` if a release was published; `"false"` otherwise. Always use `== 'true'` (string comparison). | No release needed |
| `tag-name` | `string` | Full version tag created (e.g. `v1.2.3`). | `release-created == 'false'` |
| `major-tag` | `string` | Major pointer tag updated (e.g. `v1`). | `release-created == 'false'` |

---

## Required Permissions

> **Important**: GitHub Actions `workflow_call` callers control the `GITHUB_TOKEN`
> permission scope. The reusable workflow cannot self-elevate. **You must declare
> `contents: write` on the calling job** or you will get a `403` on tag push or
> Release creation:

```yaml
jobs:
  release:
    permissions:
      contents: write       # create/push tags and GitHub Releases
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

`pull-requests: write` is **not required** — semantic-release does not create a
Release PR.

---

## Custom Configuration (Advanced)

The workflow uses built-in inline defaults for standard single-package repos
(`@semantic-release/commit-analyzer`, `@semantic-release/release-notes-generator`,
`@semantic-release/github`). No config file is needed.

For advanced configuration (custom changelog sections, non-standard bump strategies,
monorepos), add `.releaserc.json` to your repo root — semantic-release discovers it
automatically and it takes precedence over the inline defaults:

```json
{
  "branches": ["main"],
  "plugins": [
    ["@semantic-release/commit-analyzer", { "preset": "conventionalcommits" }],
    ["@semantic-release/release-notes-generator", { "preset": "conventionalcommits" }],
    ["@semantic-release/github"]
  ]
}
```

See the [semantic-release docs](https://semantic-release.gitbook.io/semantic-release/)
for full configuration options.

---

## Version Bump Rules

| Commit prefix | Bump type |
|---------------|-----------|
| `fix:`, `perf:` | PATCH |
| `feat:` | MINOR |
| `feat!:`, `BREAKING CHANGE:` footer | MAJOR |
| `chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:` | No release |

---

## Chaining a Deploy Job

```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit

  deploy:
    needs: release
    if: needs.release.outputs.release-created == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.release.outputs.tag-name }}"
```

See also: [examples/consumer-workflow-with-deploy.yml](examples/consumer-workflow-with-deploy.yml)

---

## Moving Major Version Pointer

After each release the `v1` (or `v2`, etc.) pointer tag is force-updated to the
same commit as the version tag — in the same pipeline run, atomically.

- **What `v1` means**: the latest published `v1.x.y` release.
- **When it moves**: every time a `v1.x.y` tag is created.
- **When a new pointer is created**: on a MAJOR bump. `v2.0.0` creates a new `v2`
  pointer; `v1` freezes at the last `v1.x.y` it pointed to.
- **Gotcha — no downstream workflow trigger**: force-pushing via `GITHUB_TOKEN`
  does **not** trigger other workflows listening on `on: push: tags:`. This is
  intentional GitHub behavior to prevent loops.

---

## Versioning

Pin consumers to a major tag for automatic patch/minor updates:
```yaml
uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
```

Pin to an exact version for locked-down environments:
```yaml
uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1.2.3
```

---

## Known Limitations

**Tag protection rulesets**: If your repo has GitHub Rulesets with tag name patterns
matching `v*`, `GITHUB_TOKEN` (as `github-actions[bot]`) cannot bypass them to
force-push the major pointer. You'll see a `403` error.

**Workaround** — use a GitHub App token:
```yaml
jobs:
  release:
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets:
      RELEASE_TOKEN: ${{ secrets.MY_GITHUB_APP_TOKEN }}
```

Generate the token with
[`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
in a preceding job and pass it as `RELEASE_TOKEN`.

**Inline config limitation**: The built-in inline plugin configuration does not
support per-plugin options (e.g., custom changelog presets). Add `.releaserc.json`
to your repo root for advanced configuration.
