# duckdb — facts for `duckdb.yml`

## Upstream
- Repository `duckdb/duckdb`, default branch `v2.0-cyanoptera` (the development branch for 2.0; `main` holds the released line).
- Pinned `3a604ab6d4d66910ad91bf7669c0b5f648a00190`, committed 2026-09-19T14:56:05Z ("5812 fix qualified count star (#25222)"). Workflow, Makefile, `scripts/ci/*` and `.github/config/*` read at that commit.
- Toolchain comes from upstream's CI image `ghcr.io/duckdb/duckdb-ci/manylinux_2_28_amd64_main:20260910-2fc90925` (`scripts/ci/image_version.txt`), pinned by digest `sha256:be3a1ab980091c9267feee386ac89ea94ca9d571c7302e662213e22e4f36565f` (OCI manifest, 30 layers, 1115 MB compressed). Its environment: `CC=clang`, `CXX=clang++` (`/opt/clang/bin`), `GEN=ninja`, gcc-toolset-14 on `PATH` (used by the gcc compatibility checks), Python 3.14, `VCPKG_ROOT=/vcpkg`, `VCPKG_TOOLCHAIN_PATH=/vcpkg/scripts/buildsystems/vcpkg.cmake`, `VCPKG_FORCE_SYSTEM_BINARIES=1`, `CCACHE_DIR=/ccache`. Runs as root.
- vcpkg pin: `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3` (release 2026.06.24) in every `build-duckdb` call of Main.yml.

## Job replicated
- `.github/workflows/Main.yml`, job `linux-release` ("Linux Release (${{ matrix.config.name }})"), matrix cell **"amd64 compatibility"** — the only Linux release cell on pull requests and merge groups (`scripts/ci/job_stages.py`, `compatibility_release_config(arch="amd64", optimized_release=False)`): image `manylinux_amd64_main`, `extension_config: linux_release_extensions.cmake` (icu, tpch, tpcds, fts, json, inet, parquet, autocomplete, httpfs; the out-of-tree fts/inet/httpfs are cloned at pinned commits during configure), `build_jemalloc: 1`, `lto: ""` (no LTO), `extra_cmake_variables: ""`, `is_compatibility_build: true`, `is_canonical_build: true`, `run_smoke: true`. Upstream runner `namespace-profile-linux-x64` (Namespace, core count not published).
- Upstream env (kept verbatim): `GEN=ninja`, `EXTENSION_CONFIGS=<workspace>/.github/config/linux_release_extensions.cmake`, `ENABLE_EXTENSION_AUTOLOADING=1`, `ENABLE_EXTENSION_AUTOINSTALL=1`, `BUILD_BENCHMARK=1`, `BUILD_JEMALLOC=1`, `FORCE_WARN_UNUSED=1`, `SMOKE_RUNNER=build/release/test/run`, plus `DUCKDB_VERSION`/`DUCKDB_COMMIT` from the `prepare` job.
- Upstream steps, in order:
  1. `duckdb/duckdb-ci/.github/actions/checkout@main` (Namespace mirror or `actions/checkout@v6`, then `git config --global --add safe.directory`).
  2. `./.github/actions/ccache-action` (`hendrikmuhs/ccache-action@v1.2.22`, 5 GB cache) — removed.
  3. "Write changed tests file" (list of test files changed by the PR, empty on a push without test changes).
  4. `duckdb/duckdb-ci/build-duckdb@main`: vcpkg at the pinned commit via `lukka/run-vcpkg@v11` (skipped when `$VCPKG_ROOT/vcpkgLastBuiltCommitId` already names it), env `VCPKG_TOOLCHAIN_PATH`, `VCPKG_DEFAULT_BINARY_CACHE=<workspace>/vcpkg-cache`, `VCPKG_BINARY_SOURCES=clear;files,<workspace>/vcpkg-cache,readwrite`, `USE_MERGED_VCPKG_MANIFEST=1`, `actions/cache/restore` of the vcpkg binary cache; **build** `python3 scripts/ci/retry.py -- make release`; **test** `make smoke T="--changed-tests=$RUNNER_TEMP/changed_tests.txt"`; `actions/cache/save`; bundle `make release-artifact`; upload.
  5. "Symbol Checks": `make -j4 --output-sync=target symbol-checks`.
  6. "CLI check": `./build/release/duckdb -c "PRAGMA platform; PRAGMA version;"` (plus a major-version grep on `workflow_dispatch` only).
  7. "Verify build flags": `grep -Fq -- '-march=x86-64-v3'`, `-mtune=generic` in `build/release/compile_commands.json`, and no `-flto` for a compatibility build.
  8. "GCC static library compatibility" and "GCC shared library compatibility": `gcc -std=c11 -Isrc/include examples/embedded-c/main.c …` linked against `libduckdb_static.a` + `libdummy_static_extension_loader.a`, and against `-lduckdb`; both binaries run.
  9. "Deploy" (S3 staging, secrets), three `actions/upload-artifact`, ARM64-only tests — skipped.
- What `make release` does here: `cmake -G Ninja -DDUCKDB_EXTENSION_CONFIGS=… -DENABLE_EXTENSION_AUTOLOADING=1 -DENABLE_EXTENSION_AUTOINSTALL=1 -DBUILD_BENCHMARKS=1 -DENABLE_JEMALLOC=1 -DFORCE_WARN_UNUSED=1 -DDUCKDB_EXPLICIT_VERSION=… -DGIT_COMMIT_HASH=… -DCMAKE_TOOLCHAIN_FILE=<vcpkg> -DVCPKG_BUILD=1 -DVCPKG_MANIFEST_DIR=build/extension_configuration -DCMAKE_BUILD_TYPE=Release ../..` after the merged vcpkg manifest step, then `cmake --build .` with `CMAKE_BUILD_PARALLEL_LEVEL = 80 % of nproc` (Makefile: `CI_BUILD_JOBS`). It builds libduckdb (static and shared), the CLI, the `unittest` binary, the benchmark runner and the extensions; vcpkg compiles OpenSSL for httpfs from source (binary cache empty).
- The follow-up job "Linux Release Tests" (`make allunit` on the artifact, `python -m pytest tools/shell/tests`, examples) is **not** replicated: it is a separate upstream job and adds 15–25 min on 4 vCPU on top of a build that is already past the budget; `make smoke` (test/smoke_tests.list, upstream's own bounded subset run in this very job) is the test part.

## Changes versus upstream, and why
- `container:` uses upstream's image tag pinned by digest (`name:tag@sha256:…`), so every leg builds with the same clang/cmake/ninja/vcpkg — the same toolchain as upstream's CI; no host-toolchain confound.
- The two `duckdb/duckdb-ci` composite actions (unpinned `@main` upstream) are written out as plain steps: `actions/checkout@v7` + `safe.directory`; the vcpkg part of `build-duckdb` as shell (marker check, else `git fetch --depth 1` of the pinned commit + `bootstrap-vcpkg.sh -disableMetrics`, which verifies the vcpkg tool by SHA-512 from `scripts/vcpkg-tool-metadata.txt`); the same five env variables exported; build and test as two steps instead of one composite (per-step timings).
- `lukka/run-vcpkg@v11`, `hendrikmuhs/ccache-action` and `actions/cache` restore/save of the vcpkg binary cache removed. Cold by design: no ccache, OpenSSL built from source by vcpkg on every leg.
- `DUCKDB_VERSION`/`DUCKDB_COMMIT` computed in-job with upstream's `scripts/ci/version.py` instead of the `prepare` job. The checkout is shallow, so the dev iteration is `v2.0.0-dev1` (upstream: `-dev<commit count>`); the version string is cosmetic for the build.
- "Write changed tests file" writes an empty list (a pinned commit changes no tests), as on an upstream push without test changes.
- Deploy, artifact bundling/uploads and the `workflow_dispatch`-only version grep skipped (secrets, release plumbing).
- Smoke test output is `tee`d to `evidence/smoke.log` (`shell: bash`, pipefail).
- Parallelism left as upstream's Makefile rule (80 % of the cpus, minimum 1): 3 jobs on GitHub's 4 vCPU and on `icr-4c`, 6 on `icr-8c`, **1 on `icr-2c`** (2 × 80 / 100 rounds down). Not overridden, to keep upstream's job unchanged; it understates the 2-vCPU leg. The one-line alternative is `CMAKE_BUILD_PARALLEL_LEVEL=$(nproc)` in the build step's env (what CONTRIBUTING.md suggests) — owner's call, to be stated in the README if used.

## Third-party actions needed
- none (`duckdb/duckdb-ci/*`, `lukka/run-vcpkg`, `hendrikmuhs/ccache-action` replaced or removed).

## Expected duration
- Upstream's own job is not comparable (Namespace runner, warm ccache and vcpkg binary cache): "Linux Release (amd64 compatibility)" over 7 Main runs of 2026-09-19: min 82 s (run 35446629833), median 174 s (35458136726), max 365 s (35463123529). "Linux Release Tests" (allunit, not replicated): min 453 s (35450302296), median 557 s (35463123529), max 570 s (35464186044).
- Cold anchors on GitHub's 4-vCPU runners, from NightlyTests run 35411038451 (that workflow disables ccache; host gcc/Ubuntu, 3 build jobs by the same Makefile rule): `make release` of the core with in-tree extensions, "Sqllogic tests" job, build step 1341 s (22 min); `make relassert` with httpfs via vcpkg, "Release Assertions" job, build step 2540 s (42 min), its test step 2414 s; `make debug` 1496 s; `make reldebug`-class builds 1398–1415 s.
- Estimate for this cell on GitHub's 4-vCPU runner, cold: image pull 1–2 min, vcpkg OpenSSL 4–6 min, `make release` with the nine extensions, jemalloc, benchmarks and the unittest binary 40–55 min, smoke tests 2–4 min, checks 1 min. Total 50–65 min. `icr-8c` (6 jobs) 20–25 min; `icr-2c` with a single job about 2 h. `timeout-minutes: 240`.
- The build step is the workload; it cannot be shortened without changing the cell. The bounded test subset is upstream's own smoke list (see above).

## Memory and disk
- CONTRIBUTING.md: "The default number of parallel processes can lock up the system depending on the CPU-to-memory ratio. If this happens, restrict the maximum number of build processes: `CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make`." DuckDB uses unity builds with clang -O3; a compile job peaks at 2–3 GiB, the link of `unittest`/`libduckdb` at 3–4 GiB. With the Makefile rule the legs have 5.3 GiB per job on GitHub (16 GiB / 3), 8 GiB on `icr-2c` (8 / 1), 5.3 GiB on `icr-4c`, 5.3 GiB on `icr-8c` (32 / 6): all fine. With `CMAKE_BUILD_PARALLEL_LEVEL=$(nproc)` it would be 4 GiB per job on every leg — still fine.
- Disk: image ~3–4 GiB extracted, `build/release` 8–12 GiB, vcpkg buildtrees ~1 GiB. Needs ≥ 20 GiB free.

## May fail on the ICR image, and the fix
- Needs Docker on the runner for the job container (ICR ships Docker; laravel's service containers already ran there) and egress to `ghcr.io` (1.1 GB image pull, measured in the job's "Initialize containers" step on both sides; a pre-pulled image under D53 would remove it from the ICR number — say so if done).
- Egress during the build: `github.com` (duckdb-httpfs, duckdb-inet, duckdb-fts at pinned commits; `microsoft/vcpkg` and the `vcpkg-tool` release only if the image's marker does not match), vcpkg port sources for OpenSSL and its dependencies (GitHub release assets; vcpkg falls back to its mirrors), `PRAGMA version` needs nothing.
- Inside the container: root, no `sudo` needed; `free` may be absent in a manylinux image, so "Runner facts" reads `/proc/meminfo`; `nproc` and `/proc/meminfo` report the VM, not a cgroup limit. `actions/checkout@v7` and `actions/upload-artifact@v7` run inside the container with the runner's Node (glibc 2.28 image: supported).
- `retry.py` re-runs `make release` up to twice after a non-zero exit (upstream's wrapper): a transient failure on either side would show as a much longer build step, not as a failed job — read the step log if a number looks off.
- If `vcpkgLastBuiltCommitId` in the image does not name `cd61e1e…`, the fallback fetches vcpkg and bootstraps it (~1 min, both legs alike). No change needed.
