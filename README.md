# workflows

Reusable GitHub Actions workflows shared across the [manic-systems](https://github.com/manic-systems) organization.

Files are prefixed by language (`rust-…`) because GitHub doesn't allow
subdirectories under `.github/workflows/` — the namespace lives in the
filename. Future languages add their own prefix (`go-…`, `node-…`).

| Workflow            | Use it for |
| ------------------- | ---------- |
| `rust-checks.yml`   | CI: `nix flake check` by default, plus opt-in `cargo test`/`clippy`/`fmt`. |
| `rust-build.yml`    | Build a target with Nix on one runner. Dual-use: pass `upload: true` to attest provenance and ship the binary as a release asset. |
| `rust-release.yml`  | Release bookkeeping (tag + create release; release notes + `SHA256SUMS`). Invoked twice via a `stage` input around the caller-driven build matrix. |

The caller always owns the matrix in plain YAML — GitHub forbids array inputs
to reusable workflows, so callers invoke these once per matrix leg.

---

## CI: `rust-checks.yml`

Runs `nix flake check` by default; toggle `test` / `clippy` / `fmt` for repos
that want additional cargo-driven checks. Rust toolchain is installed only when
any cargo step is enabled.

```yaml
name: Build and Test with Cargo   # rust-release.yml gates on this name

on:
  push:
    branches: [main]
  pull_request:

jobs:
  checks:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, ubuntu-24.04-arm, macos-latest]
    uses: manic-systems/workflows/.github/workflows/rust-checks.yml@main
    with:
      os: ${{ matrix.os }}
      # flake-check is on by default; opt in to cargo checks as needed:
      # test: true
      # clippy: true
      # fmt: true
```

### Inputs

| Input                  | Default                                                      | Description |
| ---------------------- | ------------------------------------------------------------ | ----------- |
| `os`                   | *(required)*                                                 | Runner image. |
| `install-nix`          | `true`                                                       | Install Nix (needed by `nix flake check`). |
| `flake-check`          | `true`                                                       | Run the flake-check command. |
| `flake-check-command`  | `nix flake check`                                            | Command run when `flake-check` is true. |
| `rust-toolchain`       | `stable`                                                     | Toolchain channel passed to `setup-rust-toolchain` (only when any cargo step is enabled). |
| `test`                 | `false`                                                      | Run cargo test. |
| `test-command`         | `cargo test --all-features`                                  | |
| `clippy`               | `false`                                                      | Run cargo clippy with `-D warnings`. |
| `clippy-command`       | `cargo clippy --all-targets --all-features -- -D warnings`   | |
| `fmt`                  | `false`                                                      | Run cargo fmt --check. |
| `fmt-command`          | `cargo fmt --all -- --check`                                 | |

---

## Build: `rust-build.yml`

Builds a project with Nix on one runner. Dual-use: `upload: false` (default) is
the release-build CI check; `upload: true` attests provenance and uploads the
binary as a release asset.

`rust-build.yml` declares **no permissions** of its own, so it inherits the
caller's token. `upload: true` requires the caller to grant
`contents` / `id-token` / `attestations` write; the CI usage above needs none.

### Inputs

| Input             | Default                                | Description |
| ----------------- | -------------------------------------- | ----------- |
| `os`              | *(required)*                           | Runner image. |
| `build-command`   | `nix build`                            | Command that builds the project (flake default package by default). |
| `install-nix`     | `true`                                 | Install Nix before building. |
| `upload`          | `false`                                | Attest provenance and upload the binary to a release. |
| `version`         | *(required when `upload: true`)*       | Release tag to upload to (pass `needs.prepare.outputs.version`). |
| `suffix`          | *(required when `upload: true`)*       | Asset suffix; the asset is `<asset-prefix>-<suffix>`. |
| `binary-name`     | repository name                        | Built binary name under the build output. |
| `asset-prefix`    | repository name                        | Prefix for the asset filename. |
| `artifact-path`   | `result/bin/<binary-name>`             | Path to the built binary (override for non-default layouts). |

---

## Release: `rust-release.yml`

The tag/create-release and notes/checksums bookkeeping, selected by a `stage`
input and **invoked twice** around the build matrix (a reusable workflow is a
single invocation — it can't straddle the caller's matrix). The caller wires
`prepare → build → finalize`:

```yaml
name: Tag and Release

on:
  workflow_dispatch:
  workflow_run:
    workflows: [Build and Test with Cargo] # the CI workflow above
    types: [completed]
    branches: [main]

permissions:
  contents: write      # create the release / push the tag
  id-token: write      # build provenance attestations
  attestations: write  # build provenance attestations

jobs:
  prepare:
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    uses: manic-systems/workflows/.github/workflows/rust-release.yml@main
    with:
      stage: prepare

  build:
    needs: prepare
    if: ${{ needs.prepare.result == 'success' && needs.prepare.outputs.skip != 'true' }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - { os: ubuntu-latest,    suffix: linux-amd64 }
          - { os: ubuntu-24.04-arm, suffix: linux-arm64 }
          - { os: macos-latest,     suffix: macos-arm64 }
    uses: manic-systems/workflows/.github/workflows/rust-build.yml@main
    with:
      upload: true
      version: ${{ needs.prepare.outputs.version }}
      os: ${{ matrix.os }}
      suffix: ${{ matrix.suffix }}

  finalize:
    needs: [prepare, build]
    if: ${{ needs.prepare.outputs.skip != 'true' && needs.build.result == 'success' }}
    uses: manic-systems/workflows/.github/workflows/rust-release.yml@main
    with:
      stage: finalize
      version: ${{ needs.prepare.outputs.version }}
```

See [`examples/`](examples/) for both caller files with overrides annotated.
Pin to a tag (e.g. `@v1`) once this repo is tagged for reproducible releases.

### Inputs

| Input             | Stage    | Default                                                              | Description |
| ----------------- | -------- | -------------------------------------------------------------------- | ----------- |
| `stage`           | —        | *(required)*                                                         | `prepare` or `finalize`. |
| `version`         | finalize | *(required for finalize)*                                            | The tag from the prepare stage. |
| `version-command` | prepare  | `nix run nixpkgs#fq -- -r '.workspace.package.version' Cargo.toml`   | Command printing the bare version (no tag prefix) to stdout. |
| `tag-prefix`      | prepare  | `v`                                                                  | Prepended to the version to form the git tag. |
| `default-branch`  | prepare  | `main`                                                               | Branch used to detect whether the version changed. |
| `install-nix`     | prepare  | `true`                                                               | Install Nix before reading the version. |
| `asset-prefix`    | finalize | repository name                                                      | Prefix of the assets to checksum. Must match what `build` used. |

Outputs (prepare stage): `version` (the tag), `skip` (`true` when the version is unchanged).

---

## Notes

- `build-command`, `version-command`, and the `*-command` inputs are run
  verbatim in the consuming repo's job. Treat them as trusted — only set them
  from workflows you control.
- Build provenance attestations are free for public repositories. Verify an
  asset with `gh attestation verify <file> --repo <owner>/<repo>`.
- If you override `binary-name`/`asset-prefix`, set them on both `build` and
  `finalize` so the checksum step finds the right assets.
- Builds are native per runner — no cross-compilation. `linux-arm64` uses
  GitHub's native `ubuntu-24.04-arm` runner.
- `finalize` only runs when every `build` leg succeeds (`fail-fast: false` lets
  the others finish, but a failed target blocks notes/checksums).
