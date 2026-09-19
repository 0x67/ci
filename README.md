# CI Setup

[![License: MIT OR Apache-2.0](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue.svg)](#license)

Shared, reusable GitHub Actions workflows for every repository under `0x67`. Rust only for now; other languages get their own prefixed files (see Naming). One PR here reaches every caller on its next run.

## Layout

```
ci/
  .github/
    workflows/
      rust-ci.yml         # workflow_call: fmt, clippy, cargo nextest, OS × toolchain matrix
      rust-features.yml   # workflow_call: cargo-hack feature powerset, extra commands
      rust-miri.yml       # workflow_call: caller-selected test slice under Miri
      rust-fuzz.yml       # workflow_call: cargo-fuzz corpus replay (check) or time-bounded discovery
      rust-audit.yml      # workflow_call: RustSec scan of the committed Cargo.lock
      pr-title.yml        # workflow_call: Conventional Commits PR title lint
      release-please.yml  # workflow_call: release-please version-bump PR, tag, GitHub Release
      msrv-bump.yml       # workflow_dispatch: fan out MSRV bump PRs to downstream repos
      rust-template.yml   # this repo only: renders templates/rust and checks the result
  templates/
    rust/                 # cargo-generate template for a new standalone Rust repository
  downstream.example.json  # schema reference. Real list lives in vars.DOWNSTREAM_REPOS.
  README.md
```

## Naming

Language-specific workflows carry a language prefix: `rust-*` today, `go-*` and `node-*` when those land. Language-agnostic ones (`pr-title`, `release-please`, `msrv-bump`) stay unprefixed. Templates follow the same split by directory: `templates/rust/` today, `templates/<lang>/` beside it when another language lands.

## Starting a new repository

A new standalone Rust repository starts from `templates/rust/`, whose callers already point at this repo:

```bash
cargo install cargo-generate  # once per machine
cargo generate --git https://github.com/0x67/ci templates/rust --name my-repo
```

cargo-generate prompts for `description`, `author_name`, `author_email`, `year`, `homepage`, `repository`, `msrv`, `ci_owner` (default `0x67`), `ci_ref` (default `main`), and `visibility` (`PUBLIC` or `PRIVATE`). A private repository then sets `os-matrix: '["ubuntu-latest"]'` in its `ci.yml` caller; see the comment there. MCP config (`.mcp.json`) is per machine and does not ship. `templates/rust/TEMPLATE.md` has the maintainer notes.

A change under `templates/rust/`, or to any reusable Rust workflow, runs `rust-template.yml`: it checks every placeholder is declared, renders the template, asserts the output, and runs `actionlint` and `cargo check` on it.

## Consuming from another repo

Reusable workflows are called with `uses: OWNER/REPO/.github/workflows/<name>.yml@REF`. Callers stay thin: one job stanza each.

### `.github/workflows/ci.yml` (caller)

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  rust:
    uses: 0x67/ci/.github/workflows/rust-ci.yml@main
    # optional:
    # with:
    #   workspace-args: "--all-features"
    #   run-ignored: false
    #   extra-toolchains: '["beta"]'
```

MSRV is read from the caller repo's `rust-toolchain.toml` at run time. **No `msrv:` input.** Bump `rust-toolchain.toml`; that is the only source of truth per module.

### `.github/workflows/pr-title.yml` (caller)

```yaml
name: pr-title
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

jobs:
  lint:
    uses: 0x67/ci/.github/workflows/pr-title.yml@main
```

### `.github/workflows/release.yml` (caller)

```yaml
name: release
on:
  push:
    branches: [main]

jobs:
  release:
    uses: 0x67/ci/.github/workflows/release-please.yml@main
    secrets:
      release-token: ${{ secrets.RELEASE_PLZ_TOKEN }}
```

Each caller also carries a `release-please-config.json` (release-type `rust`, `component` = crate name, `include-component-in-tag`) and a `.release-please-manifest.json` (current version) at its repo root.

## What `rust-ci.yml` runs

For every push and PR:

1. `resolve-msrv` — parses `channel = "..."` from `rust-toolchain.toml`, builds toolchain matrix.
2. `fmt` — `cargo fmt --all --check` on ubuntu-latest.
3. `test` matrix: `{ubuntu, macos, windows} × {msrv, stable, nightly}` = 9 jobs.
   - `cargo clippy --workspace --all-targets -- -D warnings`
   - `cargo nextest run --workspace --no-fail-fast`
   - `cargo test --workspace --doc`
4. Optional `cargo nextest --run-ignored ignored-only` when `run-ignored: true`.

Caching via `Swatinem/rust-cache@v2`, keyed on os+toolchain. `cargo-nextest` installed via `taiki-e/install-action` — prebuilt binary, no source build.

## Bumping MSRV globally (automated)

Single source of truth per module = its own `rust-toolchain.toml`. Central authority for what MSRV _should_ be = the `msrv-bump.yml` workflow_dispatch input.

Run from GitHub UI on the `ci` repo → Actions → `msrv-bump` → Run workflow:

- Input: new MSRV (e.g. `1.98.0`).
- Reads target repos from **`vars.DOWNSTREAM_REPOS`** (private Actions variable, never committed to this public repo).
- For each: checks out, updates `rust-toolchain.toml` `channel`, updates `Cargo.toml` `[workspace.package] rust-version`, opens a PR titled `chore(msrv): bump to <version>`.

### One-time setup on the `ci` repo

Neither setting below exists yet, so `msrv-bump` cannot run until both are added.

1. **`vars.DOWNSTREAM_REPOS`** — Settings → Secrets and variables → Actions → **Variables** tab → New repository variable. Paste JSON matching `downstream.example.json`:
   ```json
   {
     "repos": [{ "repo": "0x67/{repo-name}", "base_branch": "main" }]
   }
   ```
   Repository variables are plaintext but not in git and only visible to users with write access to the repo. Perfect for a private list on a public repo. Not a secret — do not use it for tokens.
2. **`secrets.MSRV_BUMP_TOKEN`** — Settings → Secrets and variables → Actions → **Secrets** tab. PAT with `repo` scope, write access to every repo in `DOWNSTREAM_REPOS`. Prefer a fine-grained PAT with `contents: write` + `pull-requests: write` scoped to just those repos.

Merge each generated PR when green.

## Release process

Every module wires `release-please.yml`. Under the hood: `googleapis/release-please-action`.

- Reads Conventional Commits since last tag (already enforced by `pr-title.yml`).
- Opens a release PR bumping `Cargo.toml` version + generating `CHANGELOG.md`.
- On merge: tags `<crate>-vX.Y.Z` and creates a GitHub Release with the changelog section. No `cargo publish` — every crate sets `publish = false` (git-tag distribution).

Single source of truth: `.release-please-manifest.json` version + commit history. No manual `git tag`.

Why not `release-plz`: it runs `cargo package` internally to compute the next version, and `cargo package` cannot resolve an unpublished cross-repo git dependency (`transport_core` is git-tag only, on no registry), so it fails. release-please only edits `Cargo.toml` + `CHANGELOG.md` as text, so unpublished git deps are a non-issue.

### Required secrets per consuming repo

- `RELEASE_PLZ_TOKEN` — PAT with `contents: write` + `pull-requests: write`. Cannot use the default `GITHUB_TOKEN` (its PRs don't trigger downstream workflows).

## Pinning `@ref`

Repositories under `0x67` call `@main`: one PR here reaches all of them on their next run, and a breaking change shows up in those runs. A `v1` tag gets cut the day a repository outside `0x67` depends on this one; outside callers should pin that tag or a commit SHA. Dependabot can bump pinned refs.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
