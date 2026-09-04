# github-workflows

Shared [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
and the shared [release-drafter](https://github.com/release-drafter/release-drafter) config for
`transformstudios`, `edalzell`, and `silentzco` packages. This repo is public because the callers
span multiple owners, and a private reusable workflow can only be called from within its own owner.

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

The release flow reads the draft release for the version and notes. Drafter config is
[`.github/release-drafter.yml`](.github/release-drafter.yml) in **this** repo; callers must not add
their own copy.

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

The release PR is recognised by its `release/*` branch (not a label), so `release-draft` skips it
automatically. `release-draft` loads this repo's `.github/release-drafter.yml` via `config-name`
pinned to the matching release tag (e.g. `@v1.2.1`). Do not use `github.workflow_sha` — in a
reusable workflow that is the *caller* commit, which is not a ref in this repo. Bump the tag in
`config-name` in the same release that ships the workflow change.

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
   branch and opens a PR to `main`.
2. A maintainer **squash-merges** that PR. GitHub creates the squash commit, so it is signed and
   linear — satisfying `required_signatures` / `required_linear_history` rules with no extra work.
3. **`release-publish.yml`** (`on: pull_request: types: [closed]`, guarded on merge + the `release/*`
   branch) — optionally builds/attaches assets, then publishes the draft with `target_commitish=main`
   so the tag lands on the merged commit (which now includes the CHANGELOG).

`release-prepare` opens the PR with a **fine-grained PAT** (`RELEASE_TOKEN`), not `GITHUB_TOKEN` —
so you do **not** enable the repo-wide "Allow GitHub Actions to create and approve pull requests"
setting. See [Release token](#release-token) for the one-time setup. The caller passes it as a secret:

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
    secrets:
      release_token: ${{ secrets.RELEASE_TOKEN }}
```

```yaml
# .github/workflows/release-publish.yml
name: Release Publish
on:
  pull_request:
    types: [closed]
jobs:
  publish:
    uses: edalzell/github-workflows/.github/workflows/release-publish.yml@<sha> # v1.x.y
    permissions:
      contents: write
```

The merge + `release/*` branch guard lives inside `release-publish.yml`, so the caller just wires
the `pull_request` trigger — no `if` needed (same as `release-draft`).

## Release token

The PR-gated flow opens a PR from a workflow. Rather than enable the repo-wide "Allow GitHub Actions
to create and approve pull requests" setting (which also lets Actions *approve* PRs),
`release-prepare` uses a **fine-grained personal access token** stored as the `RELEASE_TOKEN` secret.

**Create the token** (Settings → Developer settings → Personal access tokens → Fine-grained tokens):
- **Resource owner:** the account/org that owns the repos. A token belongs to **one** owner, so use a
  token owned by `edalzell` for the `edalzell` repos and one owned by `transformstudios` for the org
  repos — same secret name (`RELEASE_TOKEN`), different value. (Orgs must permit fine-grained PATs.)
- **Repository access:** only the repos that use the PR-gated flow.
- **Permissions → Repository:** **Contents: Read and write**, **Pull requests: Read and write**.
  Nothing else.
- **Expiration:** pick a length and set a calendar reminder to renew. When it lapses, `release-prepare`
  fails at the push/PR step with `Bad credentials (HTTP 401)` (GitHub also emails you ~7 days before).

**Add it as a secret** on each repo (or once as an org secret for org-owned repos):

```bash
gh secret set RELEASE_TOKEN --repo <owner>/<repo> --body "<token>"
```

The token only grants those three permissions on the selected repos and cannot approve PRs — so it
doesn't weaken review gates the way the account-wide setting would.

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
