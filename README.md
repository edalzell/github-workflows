# github-workflows

Shared [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for the `transformstudios` and `edalzell` packages. This repo is public because the callers span
two different owners (an org and a user account), and a private reusable workflow can only be
called from within its own owner.

## Pinning

Pin every `uses:` — including references to the workflows in **this** repo — to a full commit SHA,
with the version in a trailing comment. Do **not** track the moving `@v1` tag from callers:

```yaml
uses: edalzell/github-workflows/.github/workflows/release.yml@d34db33… # v1
```

`v1` still exists as a human-facing pointer to the latest good commit; bump the pinned SHA
deliberately.

## Which workflow to use

| Repo's `main` | Use |
| --- | --- |
| Not protected against direct pushes | `release.yml` (single-phase) |
| Protected by a ruleset (no direct pushes) | `release-prepare.yml` + `release-publish.yml` (PR-gated) |

All three read the latest draft release (maintained by release-drafter) for the version and notes.

## `release.yml` (single-phase)

Finds the newest draft, optionally builds and attaches a compiled asset bundle, updates and
**commits `CHANGELOG.md` directly to `main`**, then flips the draft to published last (so the
"latest release" link never points at a half-built release). Requires being able to push to `main`.

```yaml
name: Release
on:
  workflow_dispatch:
jobs:
  release:
    uses: edalzell/github-workflows/.github/workflows/release.yml@<sha> # v1
    permissions:
      contents: write
```

## `release-prepare.yml` + `release-publish.yml` (PR-gated)

For repos whose `main` is protected. Splits the release in two so the `CHANGELOG.md` change goes
through a PR instead of a direct push:

1. **`release-prepare.yml`** (`workflow_dispatch`) — updates `CHANGELOG.md` on a `release/<tag>`
   branch and opens a PR to `main` labelled `release`.
2. A maintainer **squash-merges** that PR. GitHub creates the squash commit, so it is signed and
   linear — satisfying `required_signatures` / `required_linear_history` rules with no extra work.
3. **`release-publish.yml`** (`on: pull_request: types: [closed]`, guarded on merge + the `release`
   label) — optionally builds/attaches assets, then publishes the draft with `target_commitish=main`
   so the tag lands on the merged commit (which now includes the CHANGELOG).

```yaml
# .github/workflows/release-prepare.yml
name: Release Prepare
on:
  workflow_dispatch:
jobs:
  prepare:
    uses: edalzell/github-workflows/.github/workflows/release-prepare.yml@<sha> # v1
    permissions:
      contents: write
      pull-requests: write
```

```yaml
# .github/workflows/release-publish.yml
name: Release Publish
on:
  pull_request:
    types: [closed]
jobs:
  publish:
    if: >-
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'release')
    uses: edalzell/github-workflows/.github/workflows/release-publish.yml@<sha> # v1
    permissions:
      contents: write
```

Add `exclude-labels: ['release']` to the repo's `.github/release-drafter.yml` so the release PR
itself is not rolled into the next draft.

## Inputs (asset-shipping repos)

Both `release.yml` and `release-publish.yml` accept:

| Input | Default | Purpose |
| --- | --- | --- |
| `upload_assets` | `false` | Run `npm install && npm run production`, tar `dist/`, and attach it to the release. |
| `composer_install` | `false` | Run `composer install` before the asset build (PHP packages). |
| `asset_working_dir` | `.` | Directory containing the `dist` folder to tar, relative to the repo root. |

`GITHUB_TOKEN` is passed through automatically; no `secrets:` block is needed.

## Versioning

A breaking change to any workflow gets a new major tag (`v2`); `v1` is only moved forward for
backward-compatible changes.
