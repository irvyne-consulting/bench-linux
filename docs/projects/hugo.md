# hugo — facts (drafted 2026-09-19)

## Upstream
- Repository: `gohugoio/hugo`, default branch `master`.
- Pinned: `51cd9e6c7e9b33dc3836e28ac30a3b9260dd1b4e` = head of `master` on 2026-09-19 (commit dated 2026-09-19T16:22:26Z,
  "Revert \"parser/metadecoders: Strip BOM from data prior to unmarshalling\""). `go.mod`: `go 1.27.0`, no `toolchain` line.
  `.gitmodules` exists but is empty (0 bytes) → no submodules, `submodules:` not set (upstream's checkout does not set it either).

## Replicated: `.github/workflows/test.yml`, job `test`, cell `go-version: 1.27.x, os: ubuntu-latest` — bounded
Upstream's Linux cell (matrix `1.27.x` × `[ubuntu-latest, windows-latest]`), in order: `jlumbroso/free-disk-space` →
checkout → `actions/setup-go` (1.27.x, check-latest, cache) → `actions/setup-node` (22) → `ruby/setup-ruby` (3.4.5) →
`gem install asciidoctor -v 2.0.26`, `asciidoctor-diagram -v 3.1.0`, `asciidoctor-html5s -v 0.5.1` → `go install
github.com/blampe/goat/cmd/goat@177de93b…` → `actions/setup-python` (3.x) → `go install github.com/magefile/mage@v1.15.0` →
`go install github.com/gohugoio/gotmplfmt@latest` → `pip install docutils Pygments; rst2html --version` → `sudo apt-get update
-y; sudo apt-get install -y pandoc` → `pandoc -v` → dart-sass 1.80.3 (curl + sha256sum -c + tar, on PATH) → `go install
honnef.co/go/tools/cmd/staticcheck@latest` → `diff <(gotmplfmt -d tpl/tplimpl/embedded/templates) <(printf '')` →
`staticcheck ./...` → `sass --version; mage -v check` with `HUGO_BUILD_TAGS=extended,withdeploy` → `GOOS=dragonfly go install;
go clean -i -cache`. `mage check` (magefile.go) = `Fmt` (`diff <(gofmt -d .) <(printf '')`) + `Vet` (`go vet ./...`) +
`TestRace`, which under `CI` loops over every package: `go clean -testcache`, `UninstallAll`, `go test -p 2 -race <pkg> -tags
extended,withdeploy` (145 directories carry `_test.go` files). The Windows cell runs `mage -v test` instead (same loop, no -race).

Upstream's Linux job takes ~60 min on GitHub's 4-vCPU runner (below), well over the ~40 min bound, so this workflow keeps
upstream's toolchain setup and replaces the long lint/test steps by **the build target plus the tests in one invocation**:
- `mage -v hugo` = `go build -ldflags "-X github.com/gohugoio/hugo/common/hugo.vendorInfo=mage" -tags extended,withdeploy
  github.com/gohugoio/hugo` (upstream's build target; needs cgo/g++ for libsass), then `./hugo version`.
- `go test -json -tags extended,withdeploy ./... > $RUNNER_TEMP/evidence/go-test.json` = mage's `Test` target in its
  non-CI (single-invocation) form `go test ./... -tags <tags>`: every package, same tags, no race detector, default 10-minute
  per-package timeout, package parallelism = cpus. `-json` only changes the output format (and implies -v), not what is tested.

## Changes versus upstream, and why
1. Dropped `jlumbroso/free-disk-space` (third-party; 3 min of GitHub-image housekeeping — removes dotnet/android/tool cache/swap —
   needed by upstream's two full test passes; see disk below).
2. Dropped the lint steps: `staticcheck` install + run (11.7–16.4 min, pure lint), `gotmplfmt` install + template-format check,
   `Fmt`/`Vet` of `mage check` — none is build or test work.
3. `mage -v check` (Fmt, Vet, per-package -race loop: 34–46 min) → `mage -v hugo` + `go test ./...` as above. The `-race`
   per-package loop is what makes upstream's job long and disk-hungry; the single invocation is upstream's own local form.
4. Dropped `Build for dragonfly` (`GOOS=dragonfly go install` + `go clean -i -cache`, ~1 min): a portability check that ends by
   wiping the build cache.
5. `ruby/setup-ruby@v1` (third-party, Ruby 3.4.5) → `sudo apt-get install -y ruby` merged with upstream's pandoc apt step (one
   `apt-get update`); `gem install …` run with `sudo` (system RubyGems dir). Ruby is 3.2.3 on GitHub's 24.04 image, 3.3.8 on
   26.04, distro 3.4 on ICR — all fine for asciidoctor 2.0.26 (pure Ruby; RubyGems ≥ 3 installs without docs by default).
6. `actions/setup-go@v6` with `cache: false` (its default is true), `check-latest: true` kept; `actions/setup-node@v6` with
   `package-manager-cache: false`; `actions/setup-python@v6` (no cache input set). All GitHub-owned, by major tag.
7. `pandoc -v` folded into a `Toolchain facts` step (also prints go/node/ruby/asciidoctor/rst2html/sass/mage/gcc versions).
8. `HUGO_BUILD_TAGS: extended,withdeploy` set at workflow level (upstream sets it on the Check step) — read by mage for the build;
   the test step passes `-tags` explicitly. Upstream's `GOPROXY`, `GO111MODULE`, `SASS_VERSION`, `DART_SASS_SHA_LINUX` kept.
9. `Runner facts`, `Toolchain facts`, `Evidence` (go-test.json, a pass/fail/skip summary and the failing lines printed in the log,
   versions.txt, disk.txt) added. GoAT, Node, Python/docutils, pandoc, dart-sass, asciidoctor kept exactly so that the tests
   guarded by `Supports()` (asciidocext, pandoc, rst, dartsass) run instead of skipping, as upstream.

## Third-party actions needed
None: `jlumbroso/free-disk-space` dropped, `ruby/setup-ruby` replaced (see 1 and 5).

## Expected duration on GitHub's 4-vCPU runner (upstream `test (1.27.x, ubuntu-latest)`, 10 successful `master` runs, 2026-09-12 → 2026-09-19)
- Whole job: min 52.4 min (3142 s), median 60.0 min (3601 s), max 67.6 min (4054 s). Per run (run id: seconds):
  35454762549: 4054 · 35448263875: 3152 · 35360138428: 3889 · 35080049287: 3512 · 35079874360: 3142 · 35079812700: 3215 ·
  35079481144: 3803 · 35079248640: 3690 · 34686199329: 3857 · 34686098781: 3263.
- Steps of it: `Check` (mage -v check) min 33.9 / median 41.4 / max 45.6 min; `Run staticcheck` 11.7 / 14.7 / 16.4 min;
  `Free Disk Space` ≈ 3 min; toolchain installs ≈ 1.5 min in total (setup-go 18 s, python 8 s, pandoc 8 s, goat 7 s, staticcheck
  install 16 s, the rest 0–6 s; run 35454762549). Windows cell's `mage -v test` (no race, per-package loop): 40.8 / 46.0 /
  52.1 min on windows-latest.
- No upstream measurement exists of the bounded form. Estimate on 4 vCPU: setup ≈ 2 min (apt ruby+pandoc and gems add ~40 s
  over upstream), `mage -v hugo` ≈ 2–3 min (libsass C++ via cgo and gocloud/aws dependencies dominate), `go test ./...` ≈ 10–15
  min → ≈ 15–20 min per 4-vCPU leg, ≈ 30–40 min on `icr-2c`, ≈ 8–12 min on `icr-8c`. `timeout-minutes: 90`.

## Memory / disk risks
- Disk: module cache ≈ 1.5 GB, build cache after build + tests ≈ 3–6 GB (test binaries are linked in temp dirs and removed per
  package), dart-sass 30 MB, pandoc 150 MB. GitHub's images leave ~20 GB free; the ICR VMs' disk is not known to me —
  `Runner facts` prints the free space and `disk.txt` records GOCACHE/GOMODCACHE sizes. Upstream's ENOSPC problems came from the
  `-race` + 386 passes that this workflow does not run. If a leg hits ENOSPC, add `sudo rm -rf /usr/share/dotnet
  /usr/local/lib/android` (what free-disk-space does) — GitHub legs only.
- Memory: `go test ./...` runs up to `nproc` test binaries at once; `hugolib` peaks around 2–3 GB without -race, the others are
  small. On `icr-2c` (8 GiB) two packages run at a time — fine. Link steps of the biggest test binaries use ~1–2 GB each.
- Time: the default 10-minute per-package `go test` timeout also applies upstream (GOFLAGS empty under CI); `hugolib` should stay
  under it on 2 vCPU (~5 min expected). If it trips there, `-timeout 20m` is the fix (does not change what is tested).

## What may fail on the ICR image, and the fix applied
- `go` may not be on PATH before setup-go → `Runner facts` guards it; `actions/setup-go` downloads Go 1.27.x (no image has it in
  its tool cache: GitHub caches 1.24/1.25/1.26) on every leg alike.
- No Ruby on the image → apt `ruby` (5). `gem install` without sudo would land in a user dir off PATH → `sudo gem install`.
- `actions/setup-python@v6` on Ubuntu 26.04 without a cached Python downloads a build from actions/python-versions (GitHub ships
  its own ubuntu-26.04 runners, so 26.04 builds exist); `pip install` then works because that Python is not "externally
  managed". If it ever fails on ICR, replace the two Python steps by `sudo apt-get install -y python3-docutils python3-pygments`
  (provides `rst2html`; docutils version then differs from GitHub's 0.22 pip install).
- `extended` tag needs a C++ compiler for libsass: build-essential is on both images (gcc 13 on ubuntu-latest, 15 on 26.04 legs).
- `sudo` without password is assumed, as on GitHub's images (upstream's job itself runs `sudo apt-get`).
- Tests that shell out (`asciidoctor`, `pandoc`, `rst2html`, `sass`, `npm`/`node`) skip when the binary is absent instead of failing,
  so a broken helper install would silently reduce the workload — `Toolchain facts` prints every helper's version to catch that.
