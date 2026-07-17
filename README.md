# github-workflows

Shared [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for the `transformstudios` and `edalzell` packages. This repo is public because the callers span
two different owners (an org and a user account), and a private reusable workflow can only be
called from within its own owner.

## `release.yml`

Publishes the latest draft release: it finds the newest draft, optionally builds and attaches a
compiled asset bundle, updates and commits `CHANGELOG.md`, and finally flips the draft to
published (last, so the "latest release" link never points at a half-built release).

### Usage

```yaml
name: Release
on:
  workflow_dispatch:
jobs:
  release:
    uses: edalzell/github-workflows/.github/workflows/release.yml@v1
    permissions:
      contents: write
```

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `upload_assets` | `false` | Run `npm install && npm run production`, tar `dist/`, and attach it to the release. |
| `composer_install` | `false` | Run `composer install` before the asset build (PHP packages). |
| `asset_working_dir` | `.` | Directory containing the `dist` folder to tar, relative to the repo root. |

`permissions: contents: write` is required on the calling job — the workflow both commits the
changelog and publishes the release. `GITHUB_TOKEN` is passed through automatically.

### Versioning

Callers pin to the moving `@v1` tag. A breaking change gets a new major tag (`v2`); the `v1` tag
is only moved forward for backward-compatible fixes.
