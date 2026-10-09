<!--markdownlint-disable MD024-->

# workflows

[manic-systems]: https://github.com/manic-systems

Reusable GitHub Actions workflows shared across the [manic-systems]
organization. This lets us reuse the same workflow code with optional parameters
to fine-grain their behaviour.

> [!NOTE]
> Language-specific files are prefixed by language (`rust-...`, `deno-...`)
> because GitHub doesn't allow subdirectories under `.github/workflows/`.
> Language-neutral checks use descriptive names such as `changelog-check.yml`.

<!--markdownlint-disable MD013-->

| Workflow              | Use it for                                                                                                                                                            |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rust-checks.yml`     | CI: `nix flake check` by default, plus opt-in `cargo test`/`clippy`/`fmt`.                                                                                            |
| `rust-build.yml`      | Build a target with Nix on one runner. Dual-use: pass `upload: true` to attest provenance and ship the binary as a release asset.                                     |
| `rust-release.yml`    | Release bookkeeping (tag + create release; release notes + `SHA256SUMS`; optional crates.io publish). Invoked once per `stage` around the caller-driven build matrix. |
| `deno-fmt.yml`        | Check Deno formatting of Markdown files without running language-specific CI.                                                                                         |
| `changelog-check.yml` | Require pull requests to update their changelog, with a label-based exemption.                                                                                        |
| `nix-checks.yml`      | Language-neutral Nix flake checks and optional package or named-check builds.                                                                                         |
| `rust-audit.yml`      | Cargo dependency advisories, licenses, bans, and sources; standalone or Nix-backed.                                                                                   |

<!--markdownlint-enable MD013-->

The caller owns any build or language-check matrix in plain YAML, as GitHub
forbids array inputs to reusable workflows. Run documentation checks once,
outside that matrix.

---

## Markdown formatting - `deno-fmt.yml`

Runs `deno fmt --check '**/*.md'` in the caller's repository. The quoted glob
includes root and nested Markdown files without checking standalone TypeScript,
JSON, YAML, or other files. Deno can also format supported code fences inside
Markdown. Formatting errors fail the job; files are never rewritten.

The caller's `deno.json` / `deno.jsonc` formatting options and exclusions are
honored. Set `working-directory` to check a documentation subtree. To fix
formatting locally, run `deno fmt '**/*.md'` in that same directory. A directory
with no matching files fails rather than silently passing.

### Inputs

<!--markdownlint-disable MD013-->

| Input               | Default         | Description                                                          |
| ------------------- | --------------- | -------------------------------------------------------------------- |
| `os`                | `ubuntu-latest` | Runner image.                                                        |
| `working-directory` | `.`             | Directory containing Markdown files and optional Deno configuration. |
| `deno-version`      | `v2.x`          | Deno version or semver range passed to `setup-deno`.                 |

<!--markdownlint-enable MD013-->

## Changelog check - `changelog-check.yml`

Requires a pull request to add or modify `CHANGELOG.md`. The check compares the
PR head against its merge base with the PR base SHA, so unrelated changes on the
target branch cannot satisfy it. Deleting or renaming the changelog away does
not count. This enforces a file update, not a changelog schema or the contents
of an entry.

The job runs only on `pull_request` events and is skipped on pushes, dispatches,
and merge-group events. A PR carrying `skip-changelog` is exempt by default; set
`skip-label` to an empty string to disable exemptions. Callers should include
`labeled` and `unlabeled` PR activity types so changing a label reruns the
check. Both documentation checks need only `contents: read`, including for fork
PRs; no secrets or write permissions are required. Do not use
`pull_request_target`.

### Inputs

<!--markdownlint-disable MD013-->

| Input            | Default          | Description                                         |
| ---------------- | ---------------- | --------------------------------------------------- |
| `os`             | `ubuntu-latest`  | Runner image.                                       |
| `changelog-path` | `CHANGELOG.md`   | Repository-relative literal file path (not a glob). |
| `skip-label`     | `skip-changelog` | Exemption label; empty disables the exemption.      |

<!--markdownlint-enable MD013-->

### Caller example

```yaml
name: Documentation Checks

on:
    push:
        branches: [main]
    pull_request:
        types: [opened, synchronize, reopened, labeled, unlabeled]

permissions:
    contents: read

jobs:
    markdown:
        uses: manic-systems/workflows/.github/workflows/deno-fmt.yml@v1
    changelog:
        uses: manic-systems/workflows/.github/workflows/changelog-check.yml@v1
        # with:
        #   changelog-path: docs/CHANGELOG.md
        #   skip-label: ""
```

These jobs belong outside the Rust OS matrix: each check needs only one runner.
See [`examples/ci.yml`](examples/ci.yml) for a combined caller. New workflows
become available at `@v1` after this repository's next release; use a commit SHA
to try them before then.

---

## Nix checks - `nix-checks.yml`

Runs `nix flake check --print-build-logs` by default, with an optional package
build afterward. This is the language-neutral alternative to `rust-checks.yml`
for repositories that need only Nix. Call it once per runner in the caller's
platform matrix; see [`examples/nix.yml`](examples/nix.yml).

All command steps run in `working-directory`. Commands are shell scripts, so
quoting, pipelines, and compound commands work; errors fail the job. The Nix
installer is Cachix's action, matching the existing Rust workflows. For a
preconfigured runner, set `install-nix: false`; `extra-nix-config` applies only
when installation is enabled. No Rust toolchain is installed.

### Inputs

<!--markdownlint-disable MD013-->

| Input                 | Default                              | Description                                                       |
| --------------------- | ------------------------------------ | ----------------------------------------------------------------- |
| `os`                  | `ubuntu-latest`                      | Runner image.                                                     |
| `working-directory`   | `.`                                  | Directory containing the flake; used for every command step.      |
| `install-nix`         | `true`                               | Install Nix before running commands.                              |
| `extra-nix-config`    | _(empty)_                            | Additional settings passed to the installer's `extra_nix_config`. |
| `setup-command`       | _(empty)_                            | Optional shell script run before checks and builds.               |
| `flake-check`         | `true`                               | Run the flake-check command.                                      |
| `flake-check-command` | `nix flake check --print-build-logs` | Flake-check shell script.                                         |
| `build`               | `false`                              | Run a package or named-check build after the flake check.         |
| `build-command`       | `nix build --print-build-logs`       | Build shell script.                                               |

<!--markdownlint-enable MD013-->

To run a named check (including a VM test) independently of the whole flake:

```yaml
jobs:
    vm-test:
        uses: manic-systems/workflows/.github/workflows/nix-checks.yml@v1
        with:
            flake-check: false
            build: true
            build-command: nix build .#checks.x86_64-linux.eval --no-link -L
```

Keep check discovery, matrices, dependencies, caching, and KVM preparation in
the caller. Select a runner that supports the check's requirements. A failing
flake check prevents the subsequent build.

## Dependency audit - `rust-audit.yml`

Runs `cargo-deny` against the caller's Cargo dependency graph and policy.
Provide the project's `Cargo.toml` and `deny.toml`; this workflow does not
invent an organization-wide license or dependency policy. By default it checks
all features and all four categories: advisories, licenses, bans, and sources.
Violations or advisory-database fetch errors fail the job.

The default standalone path uses SHA-pinned `EmbarkStudios/cargo-deny-action`
v2.1.1 (cargo-deny 0.20.2) with stable Rust. Its Docker image is x86_64-only, so
this workflow runs once on `ubuntu-latest`, outside the build matrix.
`manifest-path` is relative to the repository root. Put the policy alongside
that manifest, or select a custom policy with `arguments`, for example
`--all-features --config policy/deny.toml`.

Set `install-nix: true` to use the repository's dev shell instead of the Docker
action. The dev shell must provide Cargo and cargo-deny. Both Nix command steps
run in `working-directory`; standalone-only inputs do not affect these scripts.
The default Nix audit runs all categories with all features. For a reproducible
policy derivation followed by a fresh advisory audit, use:

```yaml
jobs:
    audit:
        uses: manic-systems/workflows/.github/workflows/rust-audit.yml@v1
        with:
            install-nix: true
            policy-command: nix build .#checks.x86_64-linux.cargo-deny --no-link -L
            audit-command: nix develop --command cargo deny check advisories
```

The optional policy command runs first; a failure prevents the advisory command
from running. A cached policy derivation is not a substitute for fetching fresh
advisories, so keep the second step when splitting checks this way.

### Inputs

<!--markdownlint-disable MD013-->

| Input               | Default                                                 | Description                                                                    |
| ------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `rust-toolchain`    | `stable`                                                | Standalone: Rust version installed inside the audit container.                 |
| `manifest-path`     | `Cargo.toml`                                            | Standalone: repository-relative manifest path.                                 |
| `arguments`         | `--all-features`                                        | Standalone: global cargo-deny arguments, before `check`.                       |
| `command-arguments` | _(empty)_                                               | Standalone: check arguments; empty checks all categories.                      |
| `install-nix`       | `false`                                                 | Select Nix-backed auditing and install Nix instead of using the Docker action. |
| `working-directory` | `.`                                                     | Nix: directory containing the flake and audit configuration.                   |
| `policy-command`    | _(empty)_                                               | Nix: optional dependency-policy shell script before the audit.                 |
| `audit-command`     | `nix develop --command cargo deny --all-features check` | Nix: audit shell script.                                                       |

<!--markdownlint-enable MD013-->

Both new workflows require only `contents: read`; they do not publish artifacts
or modify the repository. Command inputs are trusted caller-owned shell scripts,
not PR-provided text. [`examples/ci.yml`](examples/ci.yml) includes a standalone
audit alongside Rust checks. Callers own all triggers; consider a scheduled
audit as well as PR/push checks, since advisories can appear without dependency
changes. These workflows become available at `@v1` after the next release; pin a
commit SHA to use them before then.

---

## Check - `rust-checks.yml`

Runs `nix flake check` by default; toggle `test` / `clippy` / `fmt` for repos
that want additional cargo-driven checks. Rust toolchain is installed only when
any cargo step is enabled.

```yaml
name: Build and Test with Cargo # rust-release.yml gates on this name

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
        uses: manic-systems/workflows/.github/workflows/rust-checks.yml@v1
        with:
            os: ${{ matrix.os }}
            # flake-check is on by default; opt in to cargo checks as needed:
            # test: true
            # clippy: true
            # fmt: true
```

### Inputs

<!--markdownlint-disable MD013-->

| Input                 | Default                                                    | Description                                                                               |
| --------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `os`                  | _(required)_                                               | Runner image.                                                                             |
| `working-directory`   | `.`                                                        | Directory the cargo steps run in (the flake check stays at the repo root).                |
| `install-nix`         | `true`                                                     | Install Nix (needed by `nix flake check`).                                                |
| `flake-check`         | `true`                                                     | Run the flake-check command.                                                              |
| `flake-check-command` | `nix flake check`                                          | Command run when `flake-check` is true.                                                   |
| `rust-toolchain`      | `stable`                                                   | Toolchain channel passed to `setup-rust-toolchain` (only when any cargo step is enabled). |
| `rust-targets`        | _(empty)_                                                  | Comma-separated extra targets installed with the toolchain, e.g. `wasm32v1-none`.         |
| `setup-command`       | _(empty)_                                                  | Shell script run before the cargo steps, with Nix available.                              |
| `test`                | `false`                                                    | Run cargo test.                                                                           |
| `test-command`        | `cargo test --all-features`                                |                                                                                           |
| `clippy`              | `false`                                                    | Run cargo clippy with `-D warnings`.                                                      |
| `clippy-command`      | `cargo clippy --all-targets --all-features -- -D warnings` |                                                                                           |
| `fmt`                 | `false`                                                    | Run cargo fmt --check.                                                                    |
| `fmt-command`         | `cargo fmt --all -- --check`                               |                                                                                           |

<!--markdownlint-enable MD013-->

---

## Build - `rust-build.yml`

Builds a project with Nix on one runner. Dual-use: `upload: false` (default) is
the release-build CI check; `upload: true` attests provenance and uploads the
binary as a release asset.

`rust-build.yml` declares **no permissions** of its own, so it inherits the
caller's token. `upload: true` requires the caller to grant `contents` /
`id-token` / `attestations` write; the CI usage above needs none.

### Inputs

<!--markdownlint-disable MD013-->

| Input               | Default                          | Description                                                              |
| ------------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `os`                | _(required)_                     | Runner image.                                                            |
| `working-directory` | `.`                              | Directory the build runs in; `artifact-path` is resolved relative to it. |
| `build-command`     | `nix build`                      | Command that builds the project (flake default package by default).      |
| `install-nix`       | `true`                           | Install Nix before building.                                             |
| `upload`            | `false`                          | Attest provenance and upload the binary to a release.                    |
| `version`           | _(required when `upload: true`)_ | Release tag to upload to (pass `needs.prepare.outputs.version`).         |
| `suffix`            | _(required when `upload: true`)_ | Asset suffix; the asset is `<asset-prefix>-<suffix>`.                    |
| `binary-name`       | repository name                  | Built binary name under the build output.                                |
| `asset-prefix`      | repository name                  | Prefix for the asset filename.                                           |
| `artifact-path`     | `result/bin/<binary-name>`       | Path to the built binary (override for non-default layouts).             |

<!--markdownlint-enable MD013-->

---

## Release: `rust-release.yml`

The tag/create-release and notes/checksums bookkeeping, selected by a `stage`
input and **invoked twice** around the build matrix (a reusable workflow is a
single invocation, so it can't straddle the caller's matrix). The caller wires
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
    contents: write # create the release / push the tag
    id-token: write # build provenance attestations
    attestations: write # build provenance attestations

jobs:
    prepare:
        if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
        uses: manic-systems/workflows/.github/workflows/rust-release.yml@v1
        with:
            stage: prepare

    build:
        needs: prepare
        if: ${{ needs.prepare.result == 'success' && needs.prepare.outputs.skip != 'true' }}
        strategy:
            fail-fast: false
            matrix:
                include:
                    - { os: ubuntu-latest, suffix: linux-amd64 }
                    - { os: ubuntu-24.04-arm, suffix: linux-arm64 }
                    - { os: macos-latest, suffix: macos-arm64 }
        uses: manic-systems/workflows/.github/workflows/rust-build.yml@v1
        with:
            upload: true
            version: ${{ needs.prepare.outputs.version }}
            os: ${{ matrix.os }}
            suffix: ${{ matrix.suffix }}

    finalize:
        needs: [prepare, build]
        if: ${{ needs.prepare.outputs.skip != 'true' && needs.build.result == 'success' }}
        uses: manic-systems/workflows/.github/workflows/rust-release.yml@v1
        with:
            stage: finalize
            version: ${{ needs.prepare.outputs.version }}
```

See [`examples/`](examples/) for both caller files with overrides annotated. Pin
to a tag: `@v1` tracks the latest stable release (updated on each release), or
`@vX.Y.Z` for an exact version.

## Releasing this repo

Bump the [`VERSION`](VERSION) file (bare semver) and push to `main`. The
`Self-release` workflow then tags `vX.Y.Z` and moves the floating `vX` tag to
the release commit, creates the GitHub release, and marks pre-release versions
(e.g. `1.2.0-rc.1`) as prerelease. Releases can also be triggered manually via
`workflow_dispatch` at the current `VERSION`.

### Inputs

<!--markdownlint-disable MD013-->

| Input                       | Stage            | Default                                                            | Description                                                                                      |
| --------------------------- | ---------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `stage`                     | -                | _(required)_                                                       | `prepare`, `finalize`, or `publish`.                                                             |
| `version`                   | finalize/publish | _(required for finalize/publish)_                                  | The tag from the prepare stage.                                                                  |
| `version-command`           | prepare          | `nix run nixpkgs#fq -- -r '.workspace.package.version' Cargo.toml` | Command printing the bare version (no tag prefix) to stdout.                                     |
| `tag-prefix`                | prepare          | `v`                                                                | Prepended to the version to form the git tag.                                                    |
| `default-branch`            | prepare          | `main`                                                             | Branch used to detect whether the version changed.                                               |
| `install-nix`               | prepare          | `true`                                                             | Install Nix before reading the version.                                                          |
| `asset-prefix`              | finalize         | repository name                                                    | Prefix of the assets to checksum. Must match what `build` used.                                  |
| `publish-command`           | publish          | `cargo publish`                                                    | Command that publishes to crates.io (`CARGO_REGISTRY_TOKEN` is injected).                        |
| `publish-working-directory` | publish          | `.`                                                                | Directory the publish command runs in.                                                           |
| `publish-environment`       | publish          | _(none)_                                                           | GitHub Environment to run the publish job in; must match the crates.io Trusted Publisher config. |
| `rust-toolchain`            | publish          | `stable`                                                           | Rust toolchain channel installed before publishing.                                              |

<!--markdownlint-enable MD013-->

Outputs (prepare stage): `version` (the tag), `skip` (`true` when the version is
unchanged).

### Publishing to crates.io (Trusted Publishing)

[Trusted Publishing]: https://crates.io/docs/trusted-publishing
[`rust-lang/crates-io-auth-action`]: https://github.com/rust-lang/crates-io-auth-action

The `publish` stage ships the crate to crates.io using [Trusted Publishing].
Under this setup, GitHub's OIDC identity is exchanged for a short-lived registry
token via [`rust-lang/crates-io-auth-action`], so **no `CARGO_REGISTRY_TOKEN`
secret is stored**.

One-time setup (per crate):

1. Publish the crate manually once (`cargo publish`). <https://crates.io> only
   lets you add a Trusted Publisher to an existing crate.
2. On `crates.io` go to the crate's **Settings -> Trusted Publishing** and add a
   GitHub publisher: owner, repository, the release **workflow filename** (e.g.
   `release.yml`), and optionally an **environment** name. If you set one, pass
   the same value as `publish-environment`.

The publish job declares `id-token: write` itself, but a reusable workflow can't
grant a permission the caller withheld. The caller's release workflow must also
grant `id-token: write` (the example already does). Wire it after `finalize`:

<!--markdownlint-disable MD013-->

```yaml
publish:
    needs: [prepare, finalize]
    if: ${{ needs.prepare.outputs.skip != 'true' && needs.finalize.result == 'success' }}
    uses: manic-systems/workflows/.github/workflows/rust-release.yml@v1
    with:
        stage: publish
        version: ${{ needs.prepare.outputs.version }}
        # publish-environment: release   # set if the Trusted Publisher config uses one
        # publish-command: cargo publish -p my-crate   # e.g. a specific workspace member
```

<!--markdownlint-enable MD013-->

## Notes

- `build-command`, `version-command`, and the `*-command` inputs are run
  verbatim in the consuming repo's job. Treat them as trusted. Only set them
  from workflows you control.
- `stage` is validated. A value other than `prepare`, `finalize`, or `publish`
  fails rather than succeeding with every release job skipped.
- Build provenance attestations are free for public repositories. Verify an
  asset with `gh attestation verify <file> --repo <owner>/<repo>`.
- `cachix/install-nix-action` is pinned to a floating major tag (`@v31`) so
  consumers pinning this repo to `@v1` don't float on the action's `master`.
- crates.io Trusted Publishing currently supports GitHub Actions only, and the
  publish job runs on `ubuntu-latest`. Publishing is host-independent.
- If you override `binary-name`/`asset-prefix`, set them on both `build` and
  `finalize` so the checksum step finds the right assets.
- Builds are native per runner. There is no cross-compilation. `linux-arm64`
  uses GitHub's native `ubuntu-24.04-arm` runner.
- `finalize` only runs when every `build` leg succeeds (`fail-fast: false` lets
  the others finish, but a failed target blocks notes/checksums).
