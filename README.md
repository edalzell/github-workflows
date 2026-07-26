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

## Releasing this repo

This repo releases itself with its own PR-gated workflows (`cut-release-prepare.yml` /
`cut-release-publish.yml`), pinned to the previously released SHA — so the flow the package repos
depend on is exercised here first. You never type a version number; it comes from PR labels.

### Steps

1. **Land your change via a labelled PR.** Open a PR and apply a label (see the table below), then
   merge it. **Release Drafter** runs on merge and updates the draft release with the next version
   and a notes entry.
2. **Run *Cut Release*** — Actions → *Cut Release* → *Run workflow* (on `main`). It reads the draft
   and opens a `release/<tag>` PR that updates `CHANGELOG.md`. The PR is labelled `release`.
3. **Squash-merge the release PR.** *Cut Release Publish* fires on the merge, publishes the draft
   release, and creates the `vX.Y.Z` tag on the merged commit.
4. **(Automatic) Dependabot** opens grouped bump PRs in the caller repos within a week, moving them
   to the new version. Squash-merge those too.

### Label → version bump

Release Drafter computes the next version from the labels on the merged PRs since the last release
(the highest bump wins):

| Label(s) | Bump | Use for |
| --- | --- | --- |
| `major` | `v1.4.2 → v2.0.0` | breaking change to a workflow's inputs or behaviour |
| `feature`, `enhancement`, `change`, `improve`, `improvement` | minor `→ v1.5.0` | new input or capability |
| `fix`, `bugfix`, `bug` | patch `→ v1.4.3` | bug fix |
| `chore` | patch (the default) | docs, tooling, dependency bumps |

The `release` label is reserved for the PR that *Cut Release* opens and is excluded from the draft.

**Bootstrapping:** a release is cut using the *previous* release of these workflows. If a broken
`release-prepare`/`release-publish` ever ships, fix the following release by hand.
