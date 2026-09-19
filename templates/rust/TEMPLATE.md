# rust-boilerplate

Opinionated cargo-generate template for a new single-crate Rust library. Bootstraps:

- Single-crate library at the repo root (`Cargo.toml` + `src/lib.rs` + `tests/`). Collapse-free: no workspace wrapper. See [Single-crate vs workspace](#single-crate-vs-workspace) to add more crates later.
- MSRV pin via `rust-toolchain.toml` (single source of truth for CI too).
- `.cargo/config.toml` with cross-compilation targets + custom profiles (`production`, `production-with-debug`, `ci`, `profiling`, `profiling-sharp`, `flamegraph`).
- `lat.md/` knowledge-graph scaffolding.
- Agent config for Claude Code, Copilot, and generic Codex agents (`AGENTS.md`, `CLAUDE.md`, `.claude/`, `.agents/`). `AGENTS.md` is the single instruction source: Copilot CLI and Codex read it natively, VS Code via `chat.useAgentsMdFile`, Claude via `CLAUDE.md`'s `@AGENTS.md` import. No `.github/copilot-instructions.md` copy.
- MCP config (`.mcp.json`, `.vscode/mcp.json`) is per machine and gitignored, so it does not ship; add it locally after generation.
- Shared reusable GitHub Actions workflows: `ci`, `pr-title` lint, `release` (`release-please`).
- `release-please-config.json` + `.release-please-manifest.json` conventional-commits changelog config.
- A local `specs/` for the spec-driven workflow (requirements → design → tasks → validation). `.gitignore` ignores `/specs`, so specs stay on the machine and are never committed.
- MIT license + CHANGELOG stub.

## Usage

```bash
cargo install cargo-generate  # once per machine

cargo generate --git https://github.com/0x67/ci templates/rust --name my-module
```

From a local clone of `0x67/ci`, while editing the template:

```bash
cargo generate --path /path/to/ci/templates/rust --name my-module
```

`cargo generate` prompts for each placeholder declared in `cargo-generate.toml`:

| Placeholder             | Purpose                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `project-name`          | Kebab-case module / repo name (e.g. `my-module`).                           |
| `crate_name`            | snake_case crate name (auto-derived from `project-name` by default).        |
| `description`           | One-line description written into `Cargo.toml` + `README.md` + crate docs.  |
| `author_name`           | Author name for `Cargo.toml` + `LICENSE`.                                   |
| `author_email`          | Author email for `Cargo.toml`.                                              |
| `homepage`              | Homepage URL.                                                               |
| `repository`            | Repository URL.                                                             |
| `msrv`                  | MSRV pin. Written to `rust-toolchain.toml` + `[package]`.                   |
| `ci_owner`              | GitHub owner (user or org) hosting the shared `ci` repo.                    |
| `ci_ref`                | Git ref of the shared `ci` repo to pin against (branch, tag, or SHA).       |

## Example crate

The generated `src/lib.rs` ships a trivial `Greeter` trait so the repo compiles and tests green on first `cargo generate`:

```rust
use my_module::{DefaultGreeter, Greeter};

let g = DefaultGreeter;
let msg = g.greet("world").unwrap();
assert_eq!(msg, "hello, world");
```

Replace it with the module's real API.

## After generation

1. `cd <new-module>`.
2. `git init && git add -A && git commit -m "chore: initial commit"`.
3. Install local git hooks: `just hooks` (runs `lefthook install`). Requires `lefthook` + `convco` on PATH (`cargo install lefthook convco` or `brew install lefthook convco`).
4. Set the required secret in the new repo: `RELEASE_PLZ_TOKEN` (PAT with `contents: write` + `pull-requests: write`). No `CARGO_REGISTRY_TOKEN`: crates set `publish = false` (git-tag distribution).
5. Push. First push triggers CI matrix.

## Local git hooks (lefthook)

`lefthook.yml` at the repo root defines three hook stages:

| Stage        | Runs                                                                        |
| ------------ | --------------------------------------------------------------------------- |
| `pre-commit` | branch-name check, `cargo fmt --check`, `cargo clippy -D warnings`, `lat check` (parallel). |
| `commit-msg` | `convco check`: Conventional Commits gate.                                 |
| `pre-push`   | block direct push to `main`/`master`, then `cargo nextest run --workspace`. |

Bypass a single hook with `LEFTHOOK=0 git commit …`. Never bypass in shared work: CI enforces the same gates and will re-fail. To skip only one specific command, use `LEFTHOOK_EXCLUDE=clippy git commit …`.

To disable a check in a specific repo without editing the committed config, copy the block into an untracked `lefthook-local.yml` (already gitignored) and set `skip: true`.

## License

Dual-licensed **`MIT OR Apache-2.0`** (Rust ecosystem convention: matches `serde`, `tokio`, `hyper`). Ships:

- `LICENSE-MIT`: MIT license text with your copyright holder.
- `LICENSE-APACHE`: verbatim Apache 2.0.
- `NOTICE`: Apache 2.0 attribution notice.

Both licenses grant permission; downstream picks either.

### Switching to a different license

If a specific module needs a different license (e.g. proprietary, AGPL, single-MIT), update these places consistently; none are auto-derived:

1. Replace or delete `LICENSE-MIT` / `LICENSE-APACHE` / `NOTICE`. For a single license, drop the extras and add a single `LICENSE`.
2. `Cargo.toml` → `[package] license = "MIT OR Apache-2.0"` → your SPDX identifier (`Apache-2.0`, `MIT`, `AGPL-3.0-only`, `GPL-3.0-or-later`, `MPL-2.0`, `LicenseRef-Proprietary` for closed-source, …).
3. The module `README.md`: swap the "License" section.
4. For a proprietary crate also set `publish = false` in `Cargo.toml` to guarantee it never goes to crates.io by accident.

## Layout

```
<generated-module>/
  .cargo/config.toml        cross-compile targets + custom profiles
  .github/
    workflows/              thin caller stubs (ci, pr-title, release)
  AGENTS.md                 single instruction source (Copilot/Codex native, VS Code via chat.useAgentsMdFile)
  .claude/                  Claude Code local settings + skills
  .agents/                  generic agent skills
  .vscode/mcp.json          MCP servers for VS Code
  .mcp.json                 MCP servers for Claude Code
  src/lib.rs                example lib crate source
  tests/integration.rs      integration test
  docs/                     project docs
  lat.md/                   lat.md knowledge graph
  specs/                    spec-driven workflow artifacts (gitignored, local only)
  AGENTS.md                 agent-facing project rules (memory, search, code comments…)
  CLAUDE.md                 one-line `@AGENTS.md` import
  CHANGELOG.md              maintained by release-please
  Cargo.toml                single-crate manifest (package + profiles)
  LICENSE-MIT / LICENSE-APACHE / NOTICE
  README.md                 module readme (templated with the module name)
  release-please-config.json     release-please config (component, changelog sections)
  .release-please-manifest.json  current version tracked by release-please
  rust-toolchain.toml       MSRV pin (SOT for CI too)
```

`TEMPLATE.md` (this file) documents the template for authors and is excluded from generation; the shipped `README.md` is the module-facing readme.

## Single-crate vs workspace

Default = one library crate at the repo root: `Cargo.toml` (`[package]` + `[profile.release]`), `src/lib.rs`, `tests/`. Most modules stay single-crate: no workspace churn, one `Cargo.toml`, one `README`.

To grow into a multi-crate workspace later (e.g. a bench harness + N backends, like `ipc-spike`):

1. `mkdir -p crates/<name>` and `git mv` `src/` + `tests/` under it, alongside a `[package]` `Cargo.toml`.
2. In the root `Cargo.toml` replace `[package]` with:
   ```toml
   [workspace]
   resolver = "3"
   members = ["crates/*"]

   [workspace.package]
   edition = "2024"
   rust-version = "…"
   # authors/repository/homepage/license…

   [workspace.dependencies]
   # shared dep versions each member inherits via `{ workspace = true }`
   ```
3. Keep `[profile.release]` in the root manifest (profiles only apply from the workspace root).

Do this only when a second crate actually exists. Single-crate is the default because a lone crate needs no workspace wrapper.

## What is excluded from the template

`cargo-generate.toml`'s `ignore` list drops content that must not leak into new modules:

- `specs/**`: example specs. Nothing under `specs/` ships, and the generated `.gitignore` ignores `/specs`, so a new repository keeps its specs local.
- `docs/*.md`, `docs/*.pdf`: reference docs kept in the boilerplate for author convenience only.
- `cargo-generate.toml`, `TEMPLATE.md` (this file). The module-facing `README.md` DOES ship.

`[template] exclude` copies `justfile` verbatim, unrendered: its `{{BIN}}` and `{{VERSION}}` are `just` recipe parameters, and rendering would blank them.
