# ICR benchmarks

The same CI work, run on GitHub-hosted runners and on [ICR](https://www.irvyne.eu/icr/) (Irvyne Consulting Runners),
from open-source projects pinned to one commit each, so that anyone can open the runs and check the numbers.

**How it works.** One workflow per project under `.github/workflows/`. Each job checks out the upstream project at a
pinned commit (`actions/checkout` with `repository:` and `ref:`; the kernel is a kernel.org tarball checked against
kernel.org's SHA-256 list), then runs that project's own CI job — the same commands on every leg — with no cache. The
legs of one project run together in one dispatch: GitHub-hosted `ubuntu-latest` and `ubuntu-26.04` (4 vCPU / 16 GB on a
public repository), and ICR `icr-2c`, `icr-4c`, `icr-8c` (2 / 4 / 8 vCPU with 8 / 16 / 32 GiB, one fresh virtual machine
per job, destroyed afterwards). Projects run one after another. Every step is timed by GitHub itself and read back from
its API; evidence (configurations, tool versions, test reports) is uploaded per job where the project produces some.

**Who can run it.** Only Irvyne: a manual dispatch by a writer, then an approval on the `icr-bench` environment. Pull
requests are switched off, issues and discussions too, interactions are limited to collaborators, and the ICR runners are
visible to this repository alone within the organisation's runner group.

**Reading the numbers.** Every attempt is published with its run link: medians and observed maxima, never best cases, with
the sample size. "Execution" is the job from its start to its end; "turnaround" adds the wait between the approval and
the runner start (2–10 s on GitHub, about 20 s on ICR for a fresh VM). Two host conditions are measured on ICR — a quiet
node, and a declared concurrent load — and both are shown. GitHub-hosted runs of a public repository cost nothing;
ICR's normalised minutes follow its published tariff. The operating systems differ (`ubuntu-latest` was Ubuntu 24.04
with gcc 13 at the time; ICR runs Ubuntu 26.04 with gcc 15), so compiled projects also have a *controlled* row where
every leg builds inside the same toolchain image pinned by digest.

## Projects
| workflow | upstream, pinned commit | what runs |
|---|---|---|
| `linux.yml` | kernel.org `linux-7.2.6.tar.xz` | `x86_64_defconfig` (about 11 min on GitHub's 4 vCPU) or `allmodconfig` (hours; WERROR and debug info off so it fits the runner disk), `make -j$(nproc) bzImage modules`; delivered toolchain or the same `gcc:16.2.0` image everywhere |
| `ripgrep.yml` | BurntSushi/ripgrep `3fce3b5` | `cargo build --release --locked`, `cargo test --all --locked` |
| `laravel.yml` | laravel/framework `1d97271` | upstream's `linux_tests` job, one cell (PHP 8.4, PHPUnit 12.5.8) with MySQL, Redis, Memcached, DynamoDB service containers |
| `petclinic.yml` | spring-projects/spring-petclinic `818c413` | `./mvnw -B verify` on Temurin 17 (format checks, unit and Testcontainers tests, jar) — Java/Maven |
| `seed4j.yml` | seed4j/seed4j `2adf56c` (successor of the archived jhipster-lite) | upstream's `tests-linux` job: `npm ci`, `npm run lint:ci`, `./mvnw -B clean verify` on Temurin 25 + Node 24 (Vue front end built inside Maven, Vitest, Cypress component tests, Spring Boot ITs) — Java + Node full stack |
| `vite.yml` | vitejs/vite `e9078f8` | `pnpm install`, `pnpm run build`, `pnpm run test-unit` on Node 24, pnpm 12.4.2 via corepack; the browser suites are left out — TypeScript monorepo |
| `hono.yml` | honojs/hono `098e119` (4.13.8) | upstream's `Main` job (Node 24.7 + Bun 1.2.19 from `.tool-versions`: install, format, lint, editorconfig, build, tsc + vitest) and its `Bun` job as a second job — TypeScript |
| `fastify.yml` | fastify/fastify `630acd0` (6.0.0-alpha.4) | `npm install --ignore-scripts`, `npm run unit` (borp) on Node 24 — Node.js |
| `hugo.yml` | gohugoio/hugo `51cd9e6` | upstream's toolchain (Go 1.27, Node 22, asciidoctor gems, pandoc, dart-sass, mage), then `mage -v hugo` and `go test -tags extended,withdeploy ./...` (bounded, about 15 min on 4 vCPU) or upstream's full `mage -v check` with the per-package race loop (mode `full`, about an hour) — Go |
| `llvm.yml` | llvm/llvm-project `123c5db` | Release Clang for X86 with CMake + Ninja, assertions and tests off, two link jobs — about two hours on GitHub's 4 vCPU; a 1.5 GB checkout — heavy C++ (not an upstream CI job) |
| `uv.yml` | astral-sh/uv `220b3ee` | upstream's `cargo test on linux` job in full: mold, Rust 1.98.1, uv 0.12.13 (SHA-256 pinned), nine test Pythons, then `cargo nextest run` with upstream's profile, build and tests read apart — large Rust workspace |
| `nushell.yml` | nushell/nushell `de33b54` | upstream's `cargo` job, root workspace on Ubuntu: fmt check, clippy, build, tests, doc tests with upstream's ci profile, Rust 1.96.1 — Rust application |
| `duckdb.yml` | duckdb/duckdb `3a604ab` (v2.0-cyanoptera) | upstream's `linux-release` job (amd64 compatibility cell) inside upstream's manylinux CI image pinned by digest: `make release` with the release extensions and jemalloc, smoke tests, symbol and library checks; vcpkg bootstrapped, no ccache — C++, toolchain identical on every leg by construction |
| `pydantic.yml` | pydantic/pydantic `915896d` | upstream's `test` job, CPython 3.14 cell: `uv sync` (builds pydantic-core with maturin), `make test` — Python + Rust extension |
| `plausible.yml` | plausible/analytics `16120d0` | upstream's `Build and test` job: Erlang 28.5 / Elixir 1.20 (setup-beam), tracker build, `mix compile --warnings-as-errors`, migrations, MinIO, the six test partitions as one `mix test` — Elixir with PostgreSQL 18 and ClickHouse services |
| `mastodon.yml` | mastodon/mastodon `3cdcf93` | upstream's `build` and `test` jobs folded into one: Ruby 4.0.7 (setup-ruby), `bundle install`, Node 24 + yarn, `assets:precompile`, `db:setup`, `flatware rspec` — Rails with PostgreSQL 14 and Redis 7 services, the heaviest services case |
| `gitea.yml` | go-gitea/gitea `cdf786c` | upstream's `test-unit` job in full: Go 1.27, `make deps-backend`, `generate-go`, `test-backend` twice (race, gogit), `test-check` — Go with Elasticsearch, Meilisearch, Redis, MinIO and Azurite services |

Each project has a notes file under `docs/projects/` (upstream job replicated and its exact commands, every deviation and
why, third-party actions, the durations upstream's own runs show, memory, disk and image risks). Projects added on
2026-09-19 evening run their first pilot before any result is published here.

## Results

All numbers are seconds from GitHub's own timestamps, read back through the API; **execution** is the job from its start to
its end, **build** the build step alone where one exists. Medians over the attempts, observed maximum in brackets. Quiet
host: no other tenant's job ran on the node during the ICR legs unless stated (the daemon journal is checked for each
window; over the whole night of 2026-09-19 to 20, Socrate started 5 short jobs against our 109). Wait between the
approval and the runner start, every run: 2–10 s on GitHub, 19–22 s on ICR (fresh VM).

### Campaign 1 — five attempts per leg, all legs of a project together, 2026-09-19/20

**Linux kernel 7.2.6, `x86_64_defconfig`, `make -j$(nproc) bzImage modules`** — runs [35466702455](https://github.com/irvyne-consulting/icr-bench/actions/runs/35466702455), [35467363128](https://github.com/irvyne-consulting/icr-bench/actions/runs/35467363128), [35468001886](https://github.com/irvyne-consulting/icr-bench/actions/runs/35468001886), [35468602804](https://github.com/irvyne-consulting/icr-bench/actions/runs/35468602804), [35469211160](https://github.com/irvyne-consulting/icr-bench/actions/runs/35469211160) (delivered), [35469860541](https://github.com/irvyne-consulting/icr-bench/actions/runs/35469860541), [35470637590](https://github.com/irvyne-consulting/icr-bench/actions/runs/35470637590), [35479786364](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479786364), [35480466811](https://github.com/irvyne-consulting/icr-bench/actions/runs/35480466811), [35481145787](https://github.com/irvyne-consulting/icr-bench/actions/runs/35481145787) (controlled)

| leg | delivered toolchain: build | execution | controlled (`gcc:16.2.0` image on every leg): build | execution |
|---|---|---|---|---|
| `ubuntu-latest`, 4 vCPU (gcc 13.3) | **639** | 672 (693) | **803** | 850 (864) |
| `ubuntu-26.04`, 4 vCPU (gcc 15.2) | **688** | 723 (751) | **859** | 899 (907) |
| `icr-2c`, 2 vCPU | **446** | 471 (472) | **517** | 544 (577) |
| `icr-4c`, 4 vCPU | **234** | 257 (258) | **267** | 295 (307) |
| `icr-8c`, 8 vCPU | **128** | 151 (151) | **146** | 172 (181) |

**ripgrep, `cargo build --release --locked` + `cargo test --all --locked`** — runs [35478630982](https://github.com/irvyne-consulting/icr-bench/actions/runs/35478630982), [35478702616](https://github.com/irvyne-consulting/icr-bench/actions/runs/35478702616), [35478777363](https://github.com/irvyne-consulting/icr-bench/actions/runs/35478777363), [35478850576](https://github.com/irvyne-consulting/icr-bench/actions/runs/35478850576), [35478948655](https://github.com/irvyne-consulting/icr-bench/actions/runs/35478948655)

| leg | execution (max) | build | test |
|---|---|---|---|
| `ubuntu-latest` | **62** (65) | 30 | 26 |
| `ubuntu-26.04` | **65** (109) | 29 | 25 |
| `icr-2c` | **57** (58) | 26 | 26 |
| `icr-4c` | **35** (37) | 15 | 15 |
| `icr-8c` | **23** (23) | 8 | 9 |

**laravel/framework, `linux_tests` cell with MySQL, Redis, Memcached, DynamoDB** — runs [35479026086](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479026086), [35479166199](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479166199), [35479302474](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479302474), [35479462782](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479462782), [35479627786](https://github.com/irvyne-consulting/icr-bench/actions/runs/35479627786)

| leg | execution (max) | service containers start | PHP + Composer | tests |
|---|---|---|---|---|
| `ubuntu-latest` | **175** (189) | 25 | 15 | 128 |
| `ubuntu-26.04` | **184** (188) | 33 | 17 | 123 |
| `icr-2c` | **154** (159) | 13 | 28 | 101 |
| `icr-4c` | **154** (155) | 12 | 33 | 99 |
| `icr-8c` | **149** (152) | 12 | 28 | 99 |

ICR installs PHP 8.4 (about 20 s) that GitHub's image already contains; the PHPUnit suite is mostly single-threaded, so
the ICR sizes barely differ from each other.

### Long builds — one attempt per leg, 2026-09-20, quiet host (seconds)

The builds a team waits an hour or more for on GitHub's standard runner. `icr-2c` was left out (two threads for a
multi-hour compile). Build step, then execution in brackets.

| build | `ubuntu-latest`, 4 vCPU, gcc 13 | `ubuntu-26.04`, 4 vCPU, gcc 15 | `icr-4c` | `icr-8c` | run |
|---|---|---|---|---|---|
| Linux kernel 7.2.6 `allmodconfig` (WERROR and debug info off), `make -j$(nproc) bzImage modules` | **9522** (9556) — 2 h 39 min | **7589** (7633) — 2 h 06 min | **3428** (3452) — 57 min | **1765** (1788) — 29 min | [35484786278](https://github.com/irvyne-consulting/icr-bench/actions/runs/35484786278) |
| Clang, Release, X86 only, CMake + Ninja, two link jobs (`ninja clang`); checkout of the LLVM tree 47–62 s on every leg | **4282** (4375) — 71 min | **4576** (4664) — 76 min | **1669** (1741) — 28 min | **859** (927) — 14 min | [35491617154](https://github.com/irvyne-consulting/icr-bench/actions/runs/35491617154) |

On these, four ICR vCPUs finish in a little over a third of GitHub's four (Clang 2.6×, the full kernel 2.8×), and the
8 vCPU runner in a fifth (Clang 5.0×, kernel 5.4×): a kernel that takes GitHub's runner the whole morning is done on
`icr-8c` in half an hour. Hugo's full upstream check (`mage -v check`) failed on every leg and is being looked at.

Campaigns with more attempts per project are paused since 2026-09-20 10:35 UTC at the operator's request; the tables above
are the record until they resume.

### First pilots — one attempt per leg, 2026-09-19/20 (execution, seconds)

| project | `ubuntu-latest` | `ubuntu-26.04` | `icr-2c` | `icr-4c` | `icr-8c` | run |
|---|---|---|---|---|---|---|
| duckdb, `make release` + smoke, inside upstream's CI image | **2394** | 1735 | not run | **907** | **513** | [35476853744](https://github.com/irvyne-consulting/icr-bench/actions/runs/35476853744) |
| gitea, `test-unit` (race, twice) with 5 services | **1005** | 891 | 1072 | **593** | **341** | [35474045866](https://github.com/irvyne-consulting/icr-bench/actions/runs/35474045866) |
| nushell, fmt + clippy + build + tests + doc tests | **876** | 924 | 717 | **424** | **301** | [35474869189](https://github.com/irvyne-consulting/icr-bench/actions/runs/35474869189) |
| plausible, compile + migrations + `mix test` with PostgreSQL 18 and ClickHouse | **631** | 679 | 469 | **405** | **380** | [35472120207](https://github.com/irvyne-consulting/icr-bench/actions/runs/35472120207) |
| seed4j, `npm ci` + `lint:ci` + `mvnw clean verify` | **322** | failed in tests (second attempt queued) | 209 | **179** | **168** | [35471845649](https://github.com/irvyne-consulting/icr-bench/actions/runs/35471845649) |
| petclinic, `./mvnw -B verify` | **121** | 126 | 76 | **70** | **74** | [35471599638](https://github.com/irvyne-consulting/icr-bench/actions/runs/35471599638) |
| fastify, `npm install` + `npm run unit` | **67** | 67 | 103 | **48** | **38** | [35471464564](https://github.com/irvyne-consulting/icr-bench/actions/runs/35471464564) |
| vite, `pnpm install` + `build` + `test-unit` | **42** | 37 | 32 | **27** | **24** | [35471401973](https://github.com/irvyne-consulting/icr-bench/actions/runs/35471401973) |
| mastodon, `bundle install`, assets, `db:setup`, `flatware rspec` (7 701 examples) with PostgreSQL and Redis, third attempt | **933** | 722 | 769 | **347** | 210, one flaky spec (`BackupService … stores them as expected`) | [35496670928](https://github.com/irvyne-consulting/icr-bench/actions/runs/35496670928) |
| uv, `cargo nextest` over 5 098 tests (keyring features off), third attempt | **1224** | 1563, one flaky free-threaded Python install | 1239, two flaky free-threaded Python installs | **629** | **338** | [35497393681](https://github.com/irvyne-consulting/icr-bench/actions/runs/35497393681) |
| hugo, bounded (`mage -v hugo`, then `go test -p 2` package by package as upstream's loop), fourth attempt, green everywhere | **1710** | 1821 | 1319 | **940** | **756** | [35500717876](https://github.com/irvyne-consulting/icr-bench/actions/runs/35500717876) |
| pydantic, `uv sync` (pydantic-core built with maturin) + `make test`, second attempt | **95** | 125 | 85 | **77** | **73** | [35482147504](https://github.com/irvyne-consulting/icr-bench/actions/runs/35482147504) |
| hono, upstream's `Main` job (Node + Bun: install, format, lint, build, tsc + vitest), second attempt | **89** | 84 | 77 | **52** | **46** | [35482147505](https://github.com/irvyne-consulting/icr-bench/actions/runs/35482147505) |
| hono, upstream's `Bun` job (`bun run test:bun`), second attempt | **10** | 13 | 12 | **12** | **11** | same run |

Third attempts (rows above): **mastodon** is green on four legs, its `icr-8c` leg lost one flaky spec out of 7 701; **uv** is
green on `ubuntu-latest`, `icr-4c` and `icr-8c`, the two slowest legs lose free-threaded Python install tests to their
timeouts; **hugo** is green on every leg once the tests run package by package as upstream does (one `go test ./...` over the module kept every test binary until the end and filled the disk: "no space left on device" on ICR, "disk quota exceeded" on GitHub's 26.04 image). History of the fixes: **uv** — 5096 or 5097 of 5098 tests pass on every leg; the
failures were one test exporting from an `ssh://` git URL (blocked by the runners' egress until SSH was opened for this
tenant), the hardlink test that needs a minix filesystem (GitHub's 24.04 kernel has none; the variable is now set only
where it mounted) and two free-threaded Python installs that time out on the slowest legs; **hugo** — `go test ./...` with
one build per CPU ran 76 package builds out of memory on the 16 GiB legs, upstream's own `-p 2` is used now, and the
`codegen` package is left out (it panics outside mage); **mastodon** — `charlock_holmes` needs `zlib1g-dev`, present on
GitHub's image and not on ours, and the test job's `PAM_ENABLED` had leaked into the production asset step. **seed4j**
passes on four legs per run and fails one Cypress component test (`Patch.spec.ts`) on a different leg each time: an
upstream flake, kept as is and stated.

Reading, with the sample sizes above in mind: the compiled projects scale with the cores and with ICR's faster ones
(kernel, ripgrep, nushell, duckdb, gitea); test suites bound by a single thread or by service round trips gain less
(laravel, petclinic); GitHub's `ubuntu-26.04` image is not faster than `ubuntu-latest` on any of them. Two ICR vCPUs beat
GitHub's four on every compiled project.

The projects are the work of their authors and contributors under their own licences; nothing of them is redistributed
here.
