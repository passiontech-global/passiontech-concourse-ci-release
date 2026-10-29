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

### Manual (workflow_dispatch)

```sh
gh workflow run release.yml \
  --repo passiontech-global/passiontech-concourse-ci-release \
  -f ref=main -f version=v0.1.0
```

| Input | Default | Meaning |
|-------|---------|---------|
| `ref` | `main` | Source branch/tag/SHA to build |
| `version` | *(empty)* | Release tag. Empty → `dev-<sha8>` prerelease |
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

`.github/workflows/release.yml`: checkout source repo → install Zig (version pinned by the source repo's `mise.toml` via `jdx/mise-action`) → `zig build release` → smoke test (`ci-tool version`) → package → `gh release create`.

Action references are pinned to commit SHAs. Runs are serialized via a `concurrency` group.
