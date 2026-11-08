# shared-workflows

Reusable GitHub Actions workflows for my repos.

## compose-config

Runs `docker compose config` on every tracked Compose file, so a PR that breaks
a file's syntax, schema or variable interpolation fails CI.

- Each directory's base file (`compose.yaml`, `docker-compose.yml`, ...) is
  checked as a project, which picks up any `*override*` file beside it.
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

Callers pin `@v1`. Move the `v1` tag forward for compatible changes; cut `v2`
for anything that could start failing repos that pass today.
