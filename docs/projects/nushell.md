# nushell — facts for `nushell.yml`

## Upstream
- Repository `nushell/nushell`, default branch `main`.
- Pinned `de33b5407da7bf938bc930f1ab365fc15f942dbf`, committed 2026-09-19T11:21:57Z ("fix(table): don't panic when the terminal is too narrow to draw (#19045)"). Workflow files read at that commit.
- Rust toolchain: `rust-toolchain.toml` → `channel = "1.96.1"`, `profile = "default"` (rustc, cargo, rust-std, rust-docs, rustfmt, clippy); `Cargo.toml` `rust-version = "1.96.1"`. Workspace: 43 members, including every `nu_plugin_*` (polars among them). Profile `ci`: `inherits = "dev"`, `strip = false`, `debug = false`. `.cargo/ci.toml`: `rustflags = ["--cfg", "ci"]`. `[patch.crates-io] reedline = { git = "https://github.com/nushell/reedline", branch = "main" }` (a git dependency, resolved by `Cargo.lock`).

## Job replicated
- `.github/workflows/ci.yml` ("continuous-integration"), job `cargo`, matrix cell `target.name: Ubuntu` (`host: incredibuild-ubuntu-2204`, `target: x86_64-unknown-linux-gnu`, `options: MAIN_OPTIONS`, `steps: [fmt, clippy, build, test, doctest]`) × `workspace.name: root` (`path: ./`, `profile: ci`, `skip: []`). Job name "`cargo` in root (Ubuntu)". Upstream runner `incredibuild-ubuntu-2204` (third-party, core count not published). No `timeout-minutes` upstream.
- Upstream env (kept verbatim): `NU_LOG_LEVEL=DEBUG`, `NU_TEST_SKIP_DEPS_BUILD=true`, `CLIPPY_OPTIONS="-D warnings"`, `NUSHELL_CARGO_PROFILE=ci`, `MAIN_OPTIONS="--workspace --all-targets"`.
- Upstream steps and commands, in order (`${{ env[matrix.target.options] }}` expands to `--workspace --all-targets`, `${{ matrix.workspace.profile }}` to `ci`):
  1. `actions/checkout`.
  2. "Setup Rust Toolchain": `rustup target add x86_64-unknown-linux-gnu` (working directory `./`).
  3. `Swatinem/rust-cache@v2.9.2` (`cache-all-crates`, `cache-workspace-crates`) — removed.
  4. "cargo fmt": `rustup component add rustfmt` then `cargo fmt --all --check`.
  5. "cargo clippy": `rustup component add clippy` then `cargo --config "<workspace>/.cargo/ci.toml" clippy --workspace --all-targets --profile ci -- -D warnings`.
  6. "cargo build": `cargo --config "<workspace>/.cargo/ci.toml" build --workspace --all-targets --profile ci`.
  7. "cargo test": `cargo --config "<workspace>/.cargo/ci.toml" test --workspace --all-targets --profile ci`.
  8. "cargo test --doc": `cargo --config "<workspace>/.cargo/ci.toml" test --workspace --doc --profile ci`.
  9. "Assert Clean Repo": `git diff --quiet && git diff --cached --quiet`.
  (The `cargo check` step is for the WASM cell only and does not run in this cell.)

## Changes versus upstream, and why
- `Swatinem/rust-cache` removed. Cold by design.
- "Setup Rust Toolchain": `rustup toolchain install --profile minimal` added before upstream's `rustup target add`. Upstream's runner either has 1.96.1 preinstalled or lets rustup install it implicitly with the file's `default` profile (which includes `rust-docs`, ~100 MB of HTML nobody uses in CI). The explicit minimal install keeps the download equal on both legs and small; rustfmt and clippy are then added by upstream's own steps. Same compiler, same components used.
- `rustup --version` in "Runner facts" is run from `$RUNNER_TEMP` so that rustup does not resolve `rust-toolchain.toml` and install the toolchain in that step; `echo "$HOME/.cargo/bin" >> "$GITHUB_PATH"` added first (ICR's rustup lives in the runner user's home).
- `${{ github.workspace }}` written as `$GITHUB_WORKSPACE` in the run scripts (same path).
- Nothing else: all five cargo steps, their flags, `.cargo/ci.toml`, the env and the clean-repo assertion are upstream's. No `--fail-on-…` strictness added or removed (`-D warnings` is upstream's own clippy setting and passes at the pin with 1.96.1).

## Third-party actions needed
- none.

## Expected duration
- Upstream's own job is not comparable (Incredibuild runner of unknown size, warm rust-cache with `cache-all-crates`): 8 consecutive `main` runs 2026-09-16..19: min 310 s (run 35407566785), median 397 s (35283829007 / 35123615467), max 456 s (35273915334). No upstream run of this job exists on a 4-vCPU GitHub runner or without the cache.
- Cold anchor on a GitHub 4-vCPU runner: the nightly release build of `nu` (`release` profile, opt-level "s", thin LTO, `ubuntu-22.04`) took 1462 s for its build step in run 35410987217 (job "Nu (x86_64-unknown-linux-gnu)"; that job uses `actions-rust-lang/setup-rust-toolchain`, whose cache is keyed on `Cargo.lock`, so it is at best partly warm).
- Estimate for GitHub's 4-vCPU runner, cold: toolchain download 1 min, fmt < 1 min, clippy over all targets 8–12 min, build of all targets (dev profile without debug info, polars plugin included) 15–25 min, tests 4–6 min, doctests 3–5 min. Total 30–50 min; `timeout-minutes: 150` covers `icr-2c`.
- Decision: the full job is kept (it is one job upstream; no upstream-defined subset exists for this cell). If a pilot shows it well past 45 min on GitHub, the clippy step is the one to drop first (a lint, ~a quarter of the time); say so in the README if done.

## Memory and disk
- rustc peaks of 3–4 GiB for the largest crates (`nu-command` with its tests, the polars crates); with 2 parallel jobs on `icr-2c` (8 GiB) this is the tightest leg; 4 jobs on 16 GiB and 8 on 32 GiB are comfortable.
- Tests run through `cargo test` with the default test-thread count (= cores) and spawn `nu` and plugin binaries; light.
- Disk: `target/ci` for the whole workspace with all targets is 20–30 GiB (dev builds of every plugin and every test binary). Needs ≥ 40 GiB free on the runner disk (GitHub-hosted has ~70 GiB). Check the ICR VM disk size before the first run.

## May fail on the ICR image, and the fix
- No `sudo`, no apt, no containers. Build dependencies are `build-essential` and `pkg-config` (present). Default features use rustls (no OpenSSL) and bundled SQLite; `system-clipboard` (X11) and `static-link-openssl` are off by default.
- Toolchain 1.96.1 is downloaded on every leg (GitHub's images ship 1.98.1): ~120 MB with rustfmt and clippy, equal on both sides. Both legs need `rustup` ≥ 1.28 for the argument-less `rustup toolchain install`.
- Egress: `crates.io`/`static.crates.io` (registry and ~1 GB of crate sources), `github.com` (the `reedline` git dependency, rustup dist server `static.rust-lang.org`). Some tests use the network only when explicitly enabled; none are expected to need it here.
- `NU_LOG_LEVEL=DEBUG` makes the test run verbose; the log is GitHub's, not uploaded. Nothing to fix.
