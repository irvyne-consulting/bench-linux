# uv — facts for `uv.yml`

## Upstream
- Repository `astral-sh/uv`, default branch `main`.
- Pinned `220b3eea7eba1628149c0bf6b880d30f19f678d5`, committed 2026-09-19T19:41:28Z ("ci: watch Dockerfile in trampoline repro check (#21834)"). Workflow files read at that commit.
- Rust toolchain: `rust-toolchain.toml` → `channel = "1.98.1"`. Test Pythons: `.python-versions` → 3.14.0, 3.13.2, 3.12.9, 3.11.11, 3.10.16, 3.9.21, 3.8.20, 3.13.0, 3.14.0rc2. Workspace: 75 crates under `crates/`.

## Job replicated
- `.github/workflows/test.yml` (reusable, `workflow_call`; run by `ci.yml` job `test` on every push to `main` and every PR), job `cargo-test-linux`, name "cargo test on linux". No matrix (one cell). Upstream runner `depot-ubuntu-24.04-16` (16 vCPU), `timeout-minutes: 10`.
- Upstream env (kept verbatim): `CARGO_INCREMENTAL=0`, `CARGO_NET_RETRY=10`, `CARGO_TERM_COLOR=always`, `PYTHON_VERSION=3.12`, `RUSTC_BOOTSTRAP=1` (lets stable cargo take the two `-Z` flags), `RUSTUP_MAX_RETRIES=10`, `UV_LOCKED=1`.
- Upstream steps and commands, in order:
  1. `actions/checkout` (`persist-credentials: false`).
  2. `./scripts/install-mold.sh` — mold 2.40.4 from GitHub releases, SHA-256 `4c999e19…8ec1` checked by the script, `sudo tar -C /usr/local`, `/usr/bin/ld` re-pointed to mold.
  3. `rustup toolchain install --profile minimal` (1.98.1 from the toolchain file).
  4. `Swatinem/rust-cache@v2.9.2` + "Record Cargo cache freshness" — removed.
  5. `astral-sh/setup-uv@v10.1.0` with `version: 0.12.13`, `checksum: 745765a3b6e360ad76743599ae5c42e9278c7edf8bbff9fc76d05bf2623a04dd` — replaced (see changes).
  6. `uv python install`.
  7. "Install secret service": `sudo apt install -y --no-install-recommends gnome-keyring` (after depot-specific `cat /etc/apt/apt-mirrors.txt` and an apt retry/timeout config).
  8. "Start gnome-keyring": `gnome-keyring-daemon --components=secrets --daemonize --unlock <<< 'foobar'`.
  9. btrfs (`sudo apt install -y btrfs-progs`, 1 GiB loop image, `mkfs.btrfs`, mount at `/btrfs`), tmpfs (256 MiB at `/tmpfs`), minix (16 MiB loop image, `mkfs.minix`, mount at `/minix`).
  10. "Cargo test": `uv run --only-dev cargo nextest run --cargo-profile fast-build-nightly -Z panic-abort-tests -Z checksum-freshness --features test-python-patch,test-universal,native-auth,secret-service --workspace --profile ci-linux` with env `UV_HTTP_RETRIES=5`, `RUST_BACKTRACE=1`, `CARGO_UNSTABLE_MTIME_ON_USE=<cache state>`, `UV_INTERNAL__TEST_COW_FS=/btrfs`, `UV_INTERNAL__TEST_NOCOW_FS=/tmpfs`, `UV_INTERNAL__TEST_ALT_FS=/tmpfs`, `UV_INTERNAL__TEST_LOWLINKS_FS=/minix`, `INSTA_UPDATE=new`, `INSTA_PENDING_DIR=<workspace>/pending-snapshots`. `cargo-nextest` comes from the `dev` dependency group (`astral-dev-toolchain-cargo-nextest>=0.9.143`, PyPI). Nextest profile `ci-linux` inherits `ci`: `test-threads = 20`, `fail-fast = false`, JUnit at `target/nextest/ci-linux/junit.xml`, slow-timeout 10 s × 12.
  11. "Prune superseded workspace artifacts" (cache housekeeping) — removed. Upload of pending snapshots (on failure) and JUnit — folded into the evidence artifact.

## Changes versus upstream, and why
- `Swatinem/rust-cache` and both cache-freshness steps removed; `CARGO_UNSTABLE_MTIME_ON_USE` dropped (only meaningful with the cache). Cold by design.
- `astral-sh/setup-uv` replaced by `curl` of `uv-x86_64-unknown-linux-gnu.tar.gz` from release 0.12.13 + `sha256sum -c` against upstream's own pinned checksum (the release's `.sha256` file confirms `745765a3…04dd` is this archive), extracted to `$RUNNER_TEMP/uv-bin` and put on `PATH`. No third-party action, nothing piped into a shell, no setup-uv cache.
- "Install secret service": depot-only `cat /etc/apt/apt-mirrors.txt` (the file does not exist on ICR) and the apt retry/timeout snippet replaced by `sudo apt-get update` before the install, on every leg.
- The test binaries are built in a separate step (`cargo nextest run --no-run`, same flags) before upstream's unchanged command, so GitHub's step timings separate compile time from test time. With `-Z checksum-freshness` the second command rebuilds nothing.
- `rustup --version` in "Runner facts" is run from `$RUNNER_TEMP` so that rustup does not resolve `rust-toolchain.toml` and install the toolchain in that step.
- `echo "$HOME/.cargo/bin" >> "$GITHUB_PATH"` added first (ICR's rustup lives in the runner user's home).
- Nothing else: features, profiles, filesystems, keyring and env are upstream's.

## Third-party actions needed
- none.

## Expected duration
- Upstream's own job is not comparable (16 vCPU, warm rust-cache): 8 consecutive `main` runs of 2026-09-18/19 took 218–248 s (min 218 s run 35367447475, median 241 s, max 248 s run 35386728708); in a 40-run sample the longest successes were 317 s (35277349096), 323 s (35242113458) and 342 s (35246401017), one failure at 324 s (35371230745). No upstream run of this job exists on a 4-vCPU runner or without the cache.
- Estimate for GitHub's 4-vCPU runner, cold: compile of the workspace test binaries (`fast-build-nightly`, opt-level 1, mold) 20–30 min — anchor: upstream's `bench / walltime build` job (a `profiling`-profile build of the `uv` binary on a 4-vCPU Depot arm runner) took 734 s in run 35465124524 —, `uv python install` 1–2 min, tests 10–15 min (thousands of tests spawning Python, bound by PyPI/crates.io/GitHub traffic). Total 30–45 min; `timeout-minutes: 150` covers `icr-2c`.
- Decision: the full job is kept (build + full test run); no upstream-defined subset exists for Linux (upstream partitions only Windows).

## Memory and disk
- Nextest runs 20 tests at a time whatever the core count (`test-threads = 20` in profile `ci`); each test spawns uv and Python and builds venvs. Tightest leg is `icr-2c` (8 GiB); expected to fit, but this is where an OOM would show first.
- rustc peak for the largest crates (`uv`, `uv-resolver` test binaries) is 2–3 GiB per job; 2–8 parallel jobs fit in 8–32 GiB.
- Disk: `target/` for the whole workspace test build is 15–20 GiB, the nine Pythons ~1.5 GiB, the uv cache a few GiB. Needs ≥ 30 GiB free on the runner disk (GitHub-hosted has ~70 GiB).

## May fail on the ICR image, and the fix
- `sudo` (mold to `/usr/local`, `/usr/bin/ld` symlink, apt, mounts): the kernel workflow's "delivered" job already runs `sudo apt-get` on ICR, so the runner user has it. No change.
- `wget` is used by `scripts/install-mold.sh`; it is in Ubuntu's standard set, expected present. If not, add `wget` to the ICR image (do not change the script).
- Kernel modules: `btrfs` (in Ubuntu's `linux-modules`) and `minix` (in `linux-modules-extra`): a virtual/cloud kernel without `linux-modules-extra-$(uname -r)` fails at `sudo mount -o loop /tmp/minix.img /minix`. Fix in the image (install `linux-modules-extra-$(uname -r)`), not in the workflow, so that both legs run the same steps and the same tests. Loop devices (`/dev/loop-control`) are in every Ubuntu kernel.
- `gnome-keyring-daemon` and the `secret-service`/`native-auth` tests need a session D-Bus reachable by libsecret. GitHub's image provides it (upstream passes on Depot's GitHub-derived image). If the ICR leg fails only in `native_auth`/keyring tests, drop `secret-service` from `--features` on every leg (both sides then skip the same tests) — the same kind of edit as laravel's `:imagick`.
- Egress: PyPI (`pypi.org`, `files.pythonhosted.org`), `crates.io`/`static.crates.io`, GitHub (`github.com`, `objects.githubusercontent.com`: mold, uv, python-build-standalone), astral's R2 mirror (`test-r2`), git over HTTPS. The test step measures the network as much as the machine; the build step is the clean CPU number.
- Rust 1.98.1 is already GitHub's shipped Rust (24.04 and 26.04 images, `Rustup 1.29.1`); ICR's rustup `stable` may or may not be 1.98.1 — if not, "Install Rust toolchain" downloads it (~100 MB, minimal profile) on ICR only. Both legs need `rustup` ≥ 1.28 for the argument-less `rustup toolchain install`.
