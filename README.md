# passiontech-concourse-ci-release

Public release channel for [`passiontech-concourse-ci`](https://github.com/passiontech-global/passiontech-concourse-ci) (private source repo).

GitHub Actions builds the static `x86_64-linux-musl` `ci-tool` binary from a source ref and publishes it as a public GitHub Release here.

## Releases

Each release contains:

| Asset | Contents |
|-------|----------|
| `ci-tool-linux-x86_64` | Raw static binary (`chmod +x` and run) |
| `ci-tool-linux-x86_64.tar.gz` | Same binary, tarred |
| `SHA256SUMS.txt` | Checksums for both |

Install:

```sh
gh release download v0.1.0 \
  --repo passiontech-global/passiontech-concourse-ci-release \
  --pattern ci-tool-linux-x86_64 --clobber
chmod +x ci-tool-linux-x86_64
```

## Triggering a build

### Automatic (schedule)

A cron job polls the source repo every 10 minutes. When `main` HEAD differs
from the source sha recorded in the newest release body, it builds and
publishes a new release — no manual step needed.

### Version (auto semver)

Empty `version` → computed by [git-cliff](https://git-cliff.org) over the
conventional commits since the last **stable** release:

| Commits since last release | Bump |
|----------------------------|------|
| `feat:` | minor |
| `fix:` | patch |
| `BREAKING CHANGE` / `!:` | major |
| anything else (`chore:`, `docs:`, `refactor:`, `perf:`…) | patch |

Each release body embeds a changelog grouped by type (generated from the
source repo git history via `cliff.toml`) plus the exact source sha it was
built from.

### Manual (workflow_dispatch)

```sh
gh workflow run release.yml \
  --repo passiontech-global/passiontech-concourse-ci-release \
  -f ref=main                     # auto semver bump
gh workflow run release.yml \
  --repo passiontech-global/passiontech-concourse-ci-release \
  -f ref=main -f version=v1.0.0   # explicit version
```

| Input | Default | Meaning |
|-------|---------|---------|
| `ref` | `main` | Source branch/tag/SHA to build |
| `version` | *(empty)* | Release tag. Empty → auto semver bump |
| `prerelease` | `false` | Mark release as prerelease |

### Automation (repository_dispatch)

POST `event_type=build-release` — same fields via `client_payload`:

```sh
gh api repos/passiontech-global/passiontech-concourse-ci-release/dispatches \
  -X POST -f event_type=build-release \
  -f 'client_payload[ref]=main' \
  -f 'client_payload[version]=v0.1.0'
```

The caller needs a token with `repo` scope on this repository.

## Secrets

| Secret | Purpose |
|--------|---------|
| `SOURCE_REPO_TOKEN` | PAT with read access to the private source repo (contents only is enough — prefer a fine-grained PAT scoped to `passiontech-concourse-ci`) |

## How it works

`.github/workflows/release.yml` has two jobs:

1. `poll` (schedule only) — compares source `main` HEAD with the source sha in
   the newest release body; skips `release` when unchanged.
2. `release` — checkout this repo (`meta/`) + source repo (`source/`, full
   history) → install Zig (pinned by source `mise.toml`) and git-cliff →
   compute version + changelog → `zig build release` → smoke test → package →
   `gh release create`.

Action references are pinned to commit SHAs. Runs are serialized via a
`concurrency` group. To trigger instantly instead of waiting for the poll,
POST a `repository_dispatch` (above) from a workflow in the source repo.
