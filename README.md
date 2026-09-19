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
| `linux.yml` | kernel.org `linux-7.2.6.tar.xz` | `x86_64_defconfig`, `make -j$(nproc) bzImage modules`; delivered toolchain or the same `gcc:16.2.0` image everywhere |
| `ripgrep.yml` | BurntSushi/ripgrep `3fce3b5` | `cargo build --release --locked`, `cargo test --all --locked` |
| `laravel.yml` | laravel/framework `1d97271` | upstream's `linux_tests` job, one cell (PHP 8.4, PHPUnit 12.5.8) with MySQL, Redis, Memcached, DynamoDB service containers |
| `petclinic.yml` | spring-projects/spring-petclinic `818c413` | `./mvnw -B verify` on Temurin 17 (format checks, unit and Testcontainers tests, jar) — Java/Maven |
| `seed4j.yml` | seed4j/seed4j `2adf56c` (successor of the archived jhipster-lite) | upstream's `tests-linux` job: `npm ci`, `npm run lint:ci`, `./mvnw -B clean verify` on Temurin 25 + Node 24 (Vue front end built inside Maven, Vitest, Cypress component tests, Spring Boot ITs) — Java + Node full stack |
| `vite.yml` | vitejs/vite `e9078f8` | `pnpm install`, `pnpm run build`, `pnpm run test-unit` on Node 24, pnpm 12.4.2 via corepack; the browser suites are left out — TypeScript monorepo |
| `hono.yml` | honojs/hono `098e119` (4.13.8) | upstream's `Main` job (Node 24.7 + Bun 1.2.19 from `.tool-versions`: install, format, lint, editorconfig, build, tsc + vitest) and its `Bun` job as a second job — TypeScript |
| `fastify.yml` | fastify/fastify `630acd0` (6.0.0-alpha.4) | `npm install --ignore-scripts`, `npm run unit` (borp) on Node 24 — Node.js |
| `hugo.yml` | gohugoio/hugo `51cd9e6` | upstream's toolchain (Go 1.27, Node 22, asciidoctor gems, pandoc, dart-sass, mage), then `mage -v hugo` and `go test -tags extended,withdeploy ./...` — the 60-minute upstream job without staticcheck, the race loop and the cross-build — Go |
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
Pilots of 2026-09-19 (one or two attempts each, quiet node — the campaigns with more attempts follow):

**Linux kernel 7.2.6, `x86_64_defconfig`, `make -j$(nproc) bzImage modules`, build step only, seconds**

| leg | delivered toolchain (2 attempts) | same gcc 16.2.0 image on every leg (1 attempt) |
|---|---|---|
| `ubuntu-latest`, 4 vCPU, gcc 13.3 | 642, 641 | 790 |
| `ubuntu-26.04`, 4 vCPU, gcc 15.2 | 699, 702 | 878 |
| `icr-2c`, 2 vCPU | 448, 445 | 520 |
| `icr-4c`, 4 vCPU | 234, 233 | 271 |
| `icr-8c`, 8 vCPU | 127, 127 | 146 |

Runs: [35461792101](https://github.com/irvyne-consulting/icr-bench/actions/runs/35461792101) (build times from the
evidence artifacts; the job status reads "failure" because of a wrapper defect in the workflow's first version, the builds
exited 0), [35462489984](https://github.com/irvyne-consulting/icr-bench/actions/runs/35462489984),
[35463261826](https://github.com/irvyne-consulting/icr-bench/actions/runs/35463261826). Whole-job turnaround from the
approval, second run: 683 s on `ubuntu-latest`, 745 s on `ubuntu-26.04`, 495 s on `icr-2c`, 280 s on `icr-4c`, 173 s on
`icr-8c`.

**ripgrep, execution / turnaround, seconds, 1 attempt** (runs in the former `bench-ripgrep` repository, archived):
`ubuntu-latest` 70 / 72 · `ubuntu-26.04` 74 / 79 · `icr-2c` 59 / 80 · `icr-4c` 38 / 57 · `icr-8c` 25 / 45.

**laravel, execution, seconds** (former `bench-laravel`, archived): `ubuntu-latest` 173 and 172 · `icr-2c` 152 ·
`icr-4c` 144 · `icr-8c` 145 — ICR installs PHP 8.4 in 21 s that GitHub's image already contains.

The projects are the work of their authors and contributors under their own licences; nothing of them is redistributed
here.
