# github-workflows

Shared [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for the `transformstudios` and `edalzell` packages. This repo is public because the callers span
two different owners (an org and a user account), and a private reusable workflow can only be
called from within its own owner.

## Pinning

Pin every `uses:` — including references to the workflows in **this** repo — to a full commit SHA,
with the **specific released version** in a trailing comment:

```yaml
uses: edalzell/github-workflows/.github/workflows/release.yml@d3446ce… # v1.0.0
```

The version in the comment is what lets Dependabot recognise the current version and open a PR
bumping both the SHA and the comment when a new release ships. Enable the `github-actions`
ecosystem in each caller repo's `.github/dependabot.yml` (grouped, so bumps arrive as one PR).

Do not reference a moving major tag like `@v1` — nothing here resolves one.

## Which workflow to use

Every package repo wires up **Release Draft** plus one release flow:

| Piece | Workflow(s) | Trigger |
| --- | --- | --- |
| Keep the draft release current | `release-draft.yml` | every merged PR |
| Release — `main` not protected | `release.yml` (single-phase) | manual |
| Release — `main` protected by a ruleset | `release-prepare.yml` + `release-publish.yml` (PR-gated) | manual + PR merge |

The release flow reads the draft release for the version and notes. Each repo still provides its own
`.github/release-drafter.yml` config.

## `release-draft.yml` (all repos)

Runs [release-drafter](https://github.com/release-drafter/release-drafter) on every merged PR to
keep a draft release current, and **skips the `release` PR** so it never races the (immutable)
release that `release-publish.yml` just published.

```yaml
# .github/workflows/create-draft-release.yml
name: Release Drafter
on:
  pull_request:
    types: [closed]
jobs:
  draft:
    uses: edalzell/github-workflows/.github/workflows/release-draft.yml@<sha> # v1.x.y
    permissions:
      contents: write
      pull-requests: write
```

Add `exclude-labels: ['release']` to the repo's `.github/release-drafter.yml` so the release PR
itself is not rolled into the next draft.

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
    uses: edalzell/github-workflows/.github/workflows/release.yml@<sha> # v1.x.y
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
    uses: edalzell/github-workflows/.github/workflows/release-prepare.yml@<sha> # v1.x.y
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
    uses: edalzell/github-workflows/.github/workflows/release-publish.yml@<sha> # v1.x.y
    permissions:
      contents: write
```

## Inputs (asset-shipping repos)

Both `release.yml` and `release-publish.yml` accept:

| Input | Default | Purpose |
| --- | --- | --- |
| `upload_assets` | `false` | Run `npm install && npm run production`, tar `dist/`, and attach it to the release. |
| `composer_install` | `false` | Run `composer install` before the asset build (PHP packages). |
| `asset_working_dir` | `.` | Directory containing the `dist` folder to tar, relative to the repo root. |

`GITHUB_TOKEN` is passed through automatically; no `secrets:` block is needed.

## Releasing this repo

This repo is a small library, so it releases with a single **Tag Release** workflow rather than the
PR-gated flow it ships (that flow exists to solve branch protection on the package repos, which is
exercised there). To cut a release:

1. Merge your changes to `main` (ordinary PRs).
2. Actions → **Tag Release** → *Run workflow*, and enter the new version (e.g. `v1.1.0`). It tags
   the current commit and publishes a GitHub Release with generated notes.
3. Dependabot opens grouped bump PRs in the caller repos within a week, moving their pinned SHA (and
   the `# vX.Y.Z` comment) to the new version. Squash-merge those.

Use plain semver bumps: patch for fixes/docs, minor for new inputs or workflows, major for breaking
changes to a workflow's interface.
