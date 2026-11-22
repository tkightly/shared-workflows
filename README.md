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
    uses: tkightly/shared-workflows/.github/workflows/compose-config.yml@v1.1.0
```

## go-test

Unit tests for a Go module. Fails if `go mod tidy` would change `go.mod` or
`go.sum`, then runs `go vet ./...` and `go test -race ./...`. Go's version is
read from `go.mod`, and modules and the build cache are cached.

| Input | Default | |
|---|---|---|
| `working-directory` | `.` | Holds `go.mod` |
| `go-version-file` | `<working-directory>/go.mod` | |
| `race` | `true` | Run with the race detector |
| `test-flags` | `""` | Extra `go test` flags, e.g. `-short` or `-tags=integration` |
| `timeout-minutes` | `15` | |

```yaml
jobs:
  go:
    uses: tkightly/shared-workflows/.github/workflows/go-test.yml@v1.2.0
```

For a repo with several modules, use a matrix over `working-directory`.

## go-lint

Static checks for a Go module: `golangci-lint run ./...` with the repo's own
`.golangci.yml` (enable `gosec` there for security linting), then
`govulncheck ./...`, which fails on known vulnerabilities the code can
actually reach, including in the standard library. Go's version is read from
`go.mod` and used exactly (`GOTOOLCHAIN=local`), so keep its `go` line on a
patched release.

| Input | Default | |
|---|---|---|
| `working-directory` | `.` | Holds `go.mod` |
| `go-version-file` | `<working-directory>/go.mod` | |
| `golangci-lint-version` | `v2.15.0` | |
| `govulncheck-version` | `v1.9.0` | |
| `timeout-minutes` | `15` | |

```yaml
jobs:
  lint:
    uses: tkightly/shared-workflows/.github/workflows/go-lint.yml@v1.4.0
    with:
      working-directory: api
```

Run it on a weekly `schedule` too, so newly published vulnerabilities fail the
build even without code changes.

## playwright

Runs a repo's Playwright tests inside the official `mcr.microsoft.com/playwright`
image, so the browsers always match and nothing needs installing.

Pass the image as `image`, pinned to the lockfile's `@playwright/test` version.
The job fails early if the two differ, and `tkightly/renovate-config` bumps
both in the same PR. Without `image`, an extra job reads the version from the
lockfile first; it takes seconds but GitHub bills it as a whole minute on every
run.

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
| `image` | `""` | e.g. `mcr.microsoft.com/playwright:v1.65.0-resolute`. Must match `@playwright/test` in the lockfile. Empty: read it from the lockfile in an extra job |
| `image-flavor` | `resolute` | Ubuntu release of the image when `image` is empty. Snapshots render differently on each, so keep any local Docker runs on the same one |
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
    uses: tkightly/shared-workflows/.github/workflows/playwright.yml@v1.3.0
    permissions:
      contents: write
      statuses: write
    with:
      working-directory: web
      snapshot-path: web/e2e/__screenshots__
      image: mcr.microsoft.com/playwright:v1.65.0-resolute
```

The playwright config should treat `CI` as "rendering matches the baselines":
either skip snapshot comparison locally (`ignoreSnapshots: !process.env.CI`)
or run locally in the same image. Renovate needs
`gitIgnoredAuthors: ["41898282+github-actions[bot]@users.noreply.github.com"]`
(in `tkightly/renovate-config`), or it treats a branch with a snapshot commit
as edited and stops updating it.

## docker-image

Builds a Docker image and pushes it to `ghcr.io/<owner>/<image>`, so servers
pull a finished image instead of building it themselves. PRs only build the
image, to check it still does. Pushes publish it, tagged:

- `latest` from the default branch, which is what a server deploying main pulls
- `sha-<commit>` for every build, to pin or roll back to an exact one
- `<branch>` from other branches
- `1.2.3` and `1.2` from `v1.2.3` git tags, or from the `version` input

Layers are cached in the GitHub Actions cache, one scope per image. The image
is labelled with its source repo, so GHCR links the package to the repo and
it takes the repo's visibility: images from private repos are private.

The calling job needs `packages: write`.

| Input | Default | |
|---|---|---|
| `image` | repo name | Name under `ghcr.io/<owner>/`. Set it for repos with several images |
| `context` | `.` | Build context, from the repo root |
| `dockerfile` | `<context>/Dockerfile` | From the repo root |
| `target` | last stage | |
| `build-args` | `""` | One `KEY=value` per line |
| `version` | `""` | Semver version to tag `1.2.3` and `1.2`, e.g. release-please's `tag_name`. Empty: only on `v*` tag runs |
| `platforms` | `linux/amd64` | Anything else sets up QEMU, which is much slower |
| `timeout-minutes` | `30` | |

Outputs `image` (without a tag) and `digest`.

```yaml
name: Image

on:
  pull_request:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  image:
    uses: tkightly/shared-workflows/.github/workflows/docker-image.yml@v1.1.0
    permissions:
      contents: read
      packages: write
```

For a repo with several images, use a matrix:

```yaml
jobs:
  image:
    strategy:
      matrix:
        include:
          - { image: movie-tracker-api, context: backend }
          - { image: movie-tracker-web, context: web }
          - { image: movie-tracker-migrate, dockerfile: deploy/migrate/Dockerfile }
    uses: tkightly/shared-workflows/.github/workflows/docker-image.yml@v1.1.0
    permissions:
      contents: read
      packages: write
    with:
      image: ${{ matrix.image }}
      context: ${{ matrix.context || '.' }}
      dockerfile: ${{ matrix.dockerfile || '' }}
```

Then in the server's Compose file, replace `build:` with
`image: ghcr.io/tkightly/<image>:latest`. Pulling a private image needs a
classic token with only `read:packages`, as a `ghcr.io` registry in Arcane or
via `docker login ghcr.io`.

## release-please

Automates semver releases from [Conventional Commits](https://www.conventionalcommits.org/).
On every push to main, [release-please](https://github.com/googleapis/release-please)
works out the next version (`fix:` bumps the patch, `feat:` the minor and
`feat!:` or a `BREAKING CHANGE:` footer the major) and opens or updates a
release PR with that version and a `CHANGELOG.md`. When the PR is merged, the
next run tags `vX.Y.Z` and publishes the GitHub release. Commits that don't
follow the convention are left out of the changelog and don't bump anything.

The caller keeps two files at the repo root:

```jsonc
// release-please-config.json
{
  "packages": {
    ".": {
      "release-type": "go",          // or "simple", "node", ...
      "bump-minor-pre-major": true   // before 1.0.0, breaking changes bump the minor
    }
  }
}
```

```jsonc
// .release-please-manifest.json: the current version; release-please updates it
{ ".": "0.0.0" }
```

To pick the first version explicitly, put `Release-As: 0.1.0` in a commit
message footer.

| Input | Default | |
|---|---|---|
| `config-file` | `release-please-config.json` | |
| `manifest-file` | `.release-please-manifest.json` | |

Outputs `release_created`, `tag_name` and `version`, so later jobs can run
only when a release was published.

```yaml
on:
  push:
    branches: [main]

jobs:
  release:
    uses: tkightly/shared-workflows/.github/workflows/release-please.yml@v1.4.0
    permissions:
      contents: write
      pull-requests: write
```

The PRs, tags and releases are made with the workflow's `GITHUB_TOKEN`, and
GitHub doesn't start workflows for events that token causes. So the release
PR gets no CI checks (it only changes the changelog and version files), and
pushing the tag doesn't start `on: push: tags` workflows. To publish versioned
images, build them in the same workflow, passing the released version:

```yaml
jobs:
  release:
    uses: tkightly/shared-workflows/.github/workflows/release-please.yml@v1.5.0
    permissions:
      contents: write
      pull-requests: write

  image:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    uses: tkightly/shared-workflows/.github/workflows/docker-image.yml@v1.5.0
    permissions:
      contents: read
      packages: write
    with:
      version: ${{ needs.release.outputs.tag_name }}
```

## Versions

Releases are semver tags (`v1.2.3`), and a tag is never moved or deleted once
pushed, so a caller's pin always means the same code. Callers pin an exact
version. Bump the patch for fixes, the minor for new workflows or inputs, and
the major for anything that could start failing repos that pass today.

Releases are made by this repo's own `release-please` workflow
(`.github/workflows/release.yml`), so commit messages follow Conventional
Commits: `fix:` for fixes, `feat:` for new workflows or inputs, and `feat!:`
for anything that could start failing repos that pass today. Merging the
release PR it keeps open tags and publishes the version.

The repo has immutable releases turned on, so GitHub locks a released tag
against being moved or deleted. A tag pushed without a release isn't locked,
so always release rather than pushing tags by hand.

`v1` is an older floating tag, frozen at `v1.0.0`. Repos still on it keep
working; move them to an exact version when they're next touched.
