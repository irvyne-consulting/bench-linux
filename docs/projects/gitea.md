# gitea — facts (icr-bench `gitea.yml`)

**Upstream.** go-gitea/gitea, default branch `main`, pinned `cdf786ce92b9b5d5d3c624408dc92b6261fe4601` (head of `main` on 2026-09-19, committed 2026-09-19T13:04:53Z).

**Job replicated.** `.github/workflows/pull-db-tests.yml` → job `test-unit` (no matrix; `runs-on: ubuntu-latest`; upstream gates it on `files-changed.outputs.backend`). Its steps, reproduced in order:
1. `actions/checkout`
2. `./.github/actions/go-setup` → `./.github/actions/free-disk-space`, then `actions/setup-go` (v7, pinned by commit upstream) with `go-version-file: go.mod` (`go 1.27`, `toolchain go1.27.1`; `check-latest: true` → the latest 1.27.x on the day), `cache: false`; then its go-cache composite — skipped here (`cache: "false"`).
3. `[ -e "/.dockerenv" ] || [ -e "/run/.containerenv" ] || echo "127.0.0.1 minio devstoreaccount1.azurite.local mysql elasticsearch meilisearch smtpimap" | sudo tee -a /etc/hosts`
4. `make deps-backend` (= `go mod download`)
5. `make generate-go` with `TAGS=bindata` (= `CC= GOOS= GOARCH= CGO_ENABLED=0 go generate -tags 'bindata' ./...`)
6. `make test-backend` with `GOTEST_FLAGS=-race -timeout=20m`, `TAGS=bindata` (= `go test -race -timeout=20m -tags='bindata' $(GO_TEST_PACKAGES)`: every package except `modelmigration/...`, `tests`, `tests/integration`, `tests/integration/migration-test`)
7. `make test-backend` with `GOTEST_FLAGS=-race -timeout=20m`, `TAGS=bindata gogit`, `GITEA_TEST_CI_SKIP_EXTERNAL=true`
8. `make test-check` (fails if the tests left files in the source tree)

**Services** (copied verbatim; upstream already pins every tag by digest): `elasticsearch:8.19.15@sha256:8ff844933190136cec601777d633453f4008cc126a652257d5f63cd8e92207ef` (single-node, security and ML off, `-Xms1g -Xmx1g`, 9200), `getmeili/meilisearch:v1@sha256:c94e58ca09662dd6e65e8f1b0fd145767be3da7d5422a863a27b8d2b68e090c9` (`MEILI_ENV=development`, 7700), `redis:8@sha256:298e5b3bc566bade82f46ad5511777a4a07a294097ce16ada2f6a42be5239df5` (health `redis-cli ping`, 6379), `bitnamilegacy/minio:2025.7.23@sha256:8935e75fa5d11295c17171e4aa49efe390a1193cd7f12e4d21b92af9ffef09d7` (root user 123456/12345678, 9000), `mcr.microsoft.com/azure-storage/azurite:latest@sha256:f9791d0a0613e93057c06ae31f7ac0c0222f0530590e67e020b0986c66a36fa6` under the service id `devstoreaccount1.azurite.local` (10000).

**Changes versus upstream, and why.**
- `GITHUB_READ_TOKEN` (an upstream secret) is not passed: `services/migrations/github_test.go` then runs with `liveMode=false` and replays its recorded fixtures instead of calling api.github.com — the behaviour of every fork pull request upstream; deterministic and network-independent.
- No Go module/build cache restore (`go-setup` with `cache: "false"`): cold by design. `free-disk-space` (part of `go-setup`) is kept: it only deletes preinstalled toolchains when `/` has less than 50 GB free.
- The two `make test-backend` outputs are tee'd to files for the evidence artifact (`shell: bash`, whose `pipefail` keeps the exit status). Commands unchanged.
- `timeout-minutes: 120` (upstream: none, i.e. 360). No `files-changed` gate.

**Third-party actions needed.** None (`./.github/actions/go-setup` is upstream's own composite; the `actions/setup-go` inside it is GitHub-owned).

**Toolchain on ICR.** `setup-go` uses the resolved 1.27.x from the runner tool cache when present, otherwise downloads the linux-amd64 tarball from go.dev — distribution-independent, works on Ubuntu 26.04. `-race` needs a C compiler (gcc: build-essential on ICR, present on GitHub's images). Needs passwordless `sudo` (hosts entry; free-disk-space) — GitHub's images have it, ICR's is assumed to.

**Expected duration, GitHub 4-vCPU `ubuntu-latest`, `test-unit`, last 8 successful runs (2026-08-18…20, all with Go module + build caches restored):** 958 s (run 32177984451), 1000 s (32178580406), 1042 s (32179453802), 1093 s (32193986824), 1107 s (32187229995), 1193 s (32178491385), 1406 s (32199471835), 1534 s (32389225359) → min 958 s, median 1100 s, max 1534 s. Cold (this workflow) adds the module download and the full `-race` compilation: expect roughly 20–35 min on 4 vCPU, up to about twice that on `icr-2c`.

**Memory/disk risks.** Elasticsearch ~1.5 GiB RSS (1 GiB heap); Meilisearch, Redis, MinIO, Azurite ~0.5 GiB together; `go test -race` runs `nproc` packages in parallel, each up to ~1–2 GiB → fits 8 GiB on `icr-2c`, comfortable above. Disk: module cache ~1.5 GB, build cache and race test binaries 5–8 GB — at least 15 GB free needed. Elasticsearch with `discovery.type: single-node` skips the `vm.max_map_count` bootstrap check.

**What may fail on the ICR image, fix applied.** (a) Service hostnames (`minio`, `elasticsearch`, `meilisearch`, `devstoreaccount1.azurite.local`) resolve through upstream's own `/etc/hosts` step — passwordless sudo required, nothing else. (b) `go` possibly absent from PATH before `go-setup`: nothing uses it before that step (Runner facts prints "none"). (c) `GITHUB_READ_TOKEN` absent — handled by the test itself (above). (d) Ports 9200, 7700, 6379, 9000, 10000 must be free on the VM (service containers publish them). Nothing Ubuntu-26.04-specific: gitea's unit tests have no distribution-level dependency beyond git and gcc.
