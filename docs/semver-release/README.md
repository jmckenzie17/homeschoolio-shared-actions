# semver-release Reusable Workflow

Automates semantic versioning for any homeschoolio repository. On each push to
`main`, it parses [Conventional Commits](https://www.conventionalcommits.org/),
calculates the next SemVer, creates a Git tag, publishes a GitHub Release, and
updates the moving major version pointer (e.g. `v1`).

---

## Quick Start

Create two files in your consumer repo:

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
      pull-requests: write
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

**`.release-please-config.json`**:
```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "simple",
  "packages": { ".": {} }
}
```

That's it. Two files, under 20 lines of workflow YAML.

---

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `release-branch` | `string` | `"main"` | Branch that triggers release evaluation on push. Override if your default branch is not `main`. |
| `tag-prefix` | `string` | `"v"` | Prefix prepended to the SemVer number. Produces tags like `v1.2.3`. Treat as immutable once set — changing it causes release-please to lose track of prior versions. |
| `config-file` | `string` | `".release-please-config.json"` | Path to the release-please config file in your repo root. |

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
> these permissions on the calling job** or you will get a `403` on tag push or
> Release PR creation:

```yaml
jobs:
  release:
    permissions:
      contents: write       # create/push tags and GitHub Releases
      pull-requests: write  # release-please creates/updates the Release PR
    uses: jmckenzie17/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

---

## Required Consumer File

Your repo must have a `.release-please-config.json` at the root. Minimum content:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "simple",
  "packages": { ".": {} }
}
```

See the [release-please docs](https://github.com/googleapis/release-please) for
advanced configuration (monorepos, changelog sections, etc.).

---

## Common Gotchas

**`release_created` vs `releases_created`**: There is a known bug in
`release-please-action` v4 where the `releases_created` output (plural) always
returns `true` regardless of whether a release was created. Always use
`release_created` (singular) in your conditionals:

```yaml
# ✅ Correct
if: needs.release.outputs.release-created == 'true'

# ❌ Wrong — always true in v4, causes deploy to run on every push
if: needs.release.outputs.releases-created == 'true'
```

Note: the workflow output is named `release-created` (hyphen) but internally uses
the release-please step output `release_created` (underscore). Use the hyphenated
form when referencing workflow outputs from a `needs` context.

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

See also: [examples/](examples/)

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
