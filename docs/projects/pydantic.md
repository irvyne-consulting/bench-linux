# pydantic — facts (icr-bench `pydantic.yml`)

**Upstream.** pydantic/pydantic, default branch `main`, pinned `915896d163835a57fd7987180087409a3229bd71` (head of `main` on 2026-09-19, committed 2026-09-18T08:08:30Z).

**Job replicated.** `.github/workflows/ci.yml` → job `test` (`Test ${{ matrix.os }} / ${{ matrix.python-version }}`), cell `ubuntu-latest / 3.14` — the newest stable CPython in its matrix (`3.15` and `3.15t` are release candidates until October 2026; `pypy3.11` is not CPython). Workflow env `COLUMNS=150 UV_FROZEN=true FORCE_COLOR=1`; job env `OS=ubuntu-latest DEPS=yes UV_PYTHON_PREFERENCE=only-managed`. Steps in order:
1. `actions/checkout` (`persist-credentials: false`)
2. `astral-sh/setup-uv` (v10.0.1 pinned by commit upstream) with `python-version: 3.14`, `enable-cache: true`
3. `uv sync --all-packages --group testing-extra --all-extras` — resolves from `uv.lock` (frozen) and builds the workspace member `pydantic-core` (Rust, PyO3, `maturin>=1.15`, `editable-profile = 'dev'`, `rust-version = "1.88"`) from source
4. `uv run python -c "import pydantic.version; print(pydantic.version.version_info())"`
5. `mkdir coverage`
6. `make test NUM_THREADS=1` = `uv run coverage run -m pytest --durations=10 --parallel-threads 1`, env `COVERAGE_FILE=coverage/.coverage.Linux-py3.14`, `CONTEXT=Linux-py3.14-without-deps`
7. upload of the coverage file as an artifact — replaced by the evidence artifact

**Services.** None.

**Changes versus upstream, and why.**
- `enable-cache: false` (upstream `true`): the uv cache holds the built pydantic-core wheel and every downloaded package; cold by design.
- uv pinned to `0.12.17` (the latest release on 2026-09-19, the one upstream's `required-version = '>=0.8.4'` resolves to today): a uv release published between two campaigns must not change the measurement.
- `PYTEST_ADDOPTS=--junitxml=$RUNNER_TEMP/junit.xml`: a reporter only; `make test` and the test selection are unchanged.
- `$HOME/.cargo/bin` prepended to PATH (rustup's default location on ICR; a no-op on GitHub's images).

**Third-party actions needed.** `astral-sh/setup-uv@v10` (upstream uses it at v10.0.1). Self-hosted Ubuntu 26.04: works — it downloads a static uv binary from GitHub releases and CPython 3.14 from python-build-standalone (glibc ≥ 2.17, no per-distribution builds); its cache is auto-disabled on self-hosted runners and explicitly off here.

**Toolchain on ICR.** Python: uv-managed 3.14 (not the image's Python 3). Rust: rustup's `cargo` from `$HOME/.cargo/bin`, must be ≥ 1.88 (pydantic-core's MSRV). Also `git` (the `pydantic-docs` dependency is a git source) and a C compiler for a few sdists — present.

**Expected duration, GitHub 4-vCPU `ubuntu-latest`, `Test ubuntu-latest / 3.14`, last 8 successful runs (2026-09-17…19, all with the uv cache restored):** 124 s (run 35432157219), 124 s (35322874114), 129 s (35311530993), 122 s (35308841055), 99 s (35295654662), 127 s (35250462229), 111 s (35249906048), 102 s (35223920469) → min 99 s, median 123 s, max 129 s. Cold (this workflow) adds the Python download, all package downloads and the pydantic-core debug build: expect roughly 6–10 min on 4 vCPU, 10–15 min on `icr-2c`.

**Memory/disk risks.** rustc on pydantic-core peaks around 2–3 GiB; pytest itself under 1 GiB. Disk under 5 GB. No risk on any leg.

**What may fail on the ICR image, fix applied.** (a) `cargo` not on PATH → PATH export in Runner facts. (b) rustup stable older than 1.88 → would need `rustup update stable`; not applied (the image is documented as rustup-based, and stable has been past 1.88 since mid-2025). (c) Nothing Ubuntu-26.04-specific: Python and uv are self-contained downloads.
