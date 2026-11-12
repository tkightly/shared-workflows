# shared-workflows

Reusable GitHub Actions workflows for my repos.

## compose-config

Runs `docker compose config` on every tracked Compose file, so a PR that breaks
a file's syntax, schema or variable interpolation fails CI.

- Each directory's base file (`compose.yaml`, `docker-compose.yml`, ...) is
  checked as a project, which picks up any `*override*` file beside it.
- Files under `.github/` are skipped, so a workflow named `compose.yml` is
  not mistaken for one.
- Other Compose files are checked on their own, or as an overlay on the base
  file if they don't stand alone.
- Every tracked `.env.example` is copied to `.env` first, standing in for the
  untracked secrets file. A directory without one gets an empty `.env`.

It doesn't pull images or start containers, so it won't catch problems that
only show at runtime.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  compose:
    uses: tkightly/shared-workflows/.github/workflows/compose-config.yml@v1
```

## playwright

Runs a repo's Playwright tests inside the official `mcr.microsoft.com/playwright`
image, whose tag is read from the lockfile's `@playwright/test` version, so the
browsers always match and nothing needs installing.

With `snapshot-path` set, PRs from the repo's own branches (Renovate's
included) run with `--update-snapshots=changed` and commit any changed
screenshots back to the branch as `github-actions[bot]`, so UI changes show up
as image diffs in the PR. That push can't run workflows itself (GitHub creates
a run for it that waits for approval and never starts; ignore it), so the job
then re-runs the suite against the snapshots it just committed, which catches
flaky screenshots, and reports the result as a `Playwright snapshots` commit
status on the new commit, which shows on the PR. Everything else (pushes to
main, scheduled runs, fork PRs) compares against the committed snapshots and
fails on a difference. On failure the HTML report and test results are
uploaded.

The calling job needs `contents: write` and `statuses: write`. Don't put
`concurrency` on the calling job; keep it at the caller's workflow level.

| Input | Default | |
|---|---|---|
| `working-directory` | (required) | Holds `package.json`, the lockfile and the Playwright config |
| `snapshot-path` | `""` | Committed snapshots, from the repo root. Empty: only compare |
| `node-version` | `26` | |
| `image-flavor` | `resolute` | Ubuntu release of the image. Snapshots render differently on each, so keep any local Docker runs on the same one |
| `go-version-file` | `""` | Set up Go first, for suites whose web servers run Go |
| `setup` | `""` | Shell run from the repo root before the tests; export variables via `$GITHUB_ENV` |
| `timeout-minutes` | `30` | |

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  e2e:
    uses: tkightly/shared-workflows/.github/workflows/playwright.yml@v1
    permissions:
      contents: write
      statuses: write
    with:
      working-directory: web
      snapshot-path: web/e2e/__screenshots__
```

The playwright config should treat `CI` as "rendering matches the baselines":
either skip snapshot comparison locally (`ignoreSnapshots: !process.env.CI`)
or run locally in the same image. Renovate needs
`gitIgnoredAuthors: ["41898282+github-actions[bot]@users.noreply.github.com"]`
(in `tkightly/renovate-config`), or it treats a branch with a snapshot commit
as edited and stops updating it.

Callers pin `@v1`. Move the `v1` tag forward for compatible changes; cut `v2`
for anything that could start failing repos that pass today.
