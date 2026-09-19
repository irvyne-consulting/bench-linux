# hono — facts (drafted 2026-09-19)

## Upstream
- Repository: `honojs/hono`, default branch `main`.
- Pinned: `098e11912ab244c5c33931de007f04dc8e3c2929` = head of `main` on 2026-09-19; commit dated 2026-09-15T07:25:06Z,
  message "4.13.8" (release). `.tool-versions`: `nodejs 24.7.0`, `bun 1.2.19`, `deno 2.4.5`; `packageManager: bun@1.2.20`
  (package.json; CI uses .tool-versions). No submodules.

## Replicated: `.github/workflows/ci.yml`, jobs `Main` and `Bun`
`bench` = upstream job `Main` (ubuntu-latest, no matrix), whose steps are: `actions/setup-node` (node-version-file
.tool-versions) → `oven-sh/setup-bun` (bun-version-file .tool-versions) → `bun install --frozen-lockfile` → `bun run format`
(prettier --check) → `bun run lint` (eslint src runtime-tests build perf-measures benchmarks) → `bun run editorconfig-checker
-format github-actions` → `bun run build` (`bun ./build/build.ts` + copy of package.cjs.json, postbuild `publint`) →
`bun run test` = `tsc -p tsconfig.spec.json && vitest --run` → upload `coverage/`. `vitest --run` at the root runs every
project of `vitest.config.ts`: `main`, `jsx-runtime-default`, `jsx-runtime-dom` and `runtime-tests/{node,workerd,fastly,
lambda,lambda-edge}/vitest.config.ts`, all under Node (bun runs the `#!/usr/bin/env node` bins as Node), with v8 coverage
(`coverage/raw/default/coverage-final.json`). Newest Node upstream tests is 24 (its `Node.js v${matrix.node}` job covers 20/22/24
with `test:node` only); `Main` uses Node 24.7.0 exactly.

`bench-bun` = upstream job `Bun` (ubuntu-latest): `oven-sh/setup-bun` → `bun install --frozen-lockfile` → `bun run test:bun` =
`bun test --jsx-import-source ../../src/jsx runtime-tests/bun/*` (bunfig.toml: coverage on, lcov in `coverage/raw/bun`) →
upload `coverage/`. Same `legs` matrix; job name `bun <leg>`.

Not replicated, as asked: `Node.js v20/22/24.x` (test:node only, 14–20 s), `Deno`, `Bun - Windows`, `Fastly Compute`,
`workerd`, `AWS Lambda`, `Lambda@Edge`, `Checking if it's valid for JSR`, `Coverage` (codecov merge), and the PR-only
perf/HTTP-benchmark jobs.

## Changes versus upstream, and why
1. `oven-sh/setup-bun@v2` (third-party) → `curl -fsSL https://bun.sh/install | bash -s "bun-v<.tool-versions>"` into
   `$HOME/.bun/bin`, put first on PATH: the same pinned release (1.2.19) on every leg. GitHub's Ubuntu 24.04/26.04 images ship
   no bun (checked in actions/runner-images readmes of 2026-09-07); the ICR image's system bun is left untouched, the pinned one
   wins on PATH. The install script needs `unzip`: a guard installs it through apt only if absent (present on GitHub's images).
2. `actions/setup-node` by `@v6` tag (upstream pins the same major by SHA) with `package-manager-cache: false` — setup-node v6/v7
   auto-caches npm only when package.json's `packageManager` names npm (hono's says bun), so this is belt-and-braces; no cache.
3. Steps named, `Runner facts` and `Evidence` steps added; the coverage upload of upstream becomes the evidence artifact
   (`coverage-final.json` only, not the html; `lcov.info` for the Bun job) plus `versions.txt`.
4. Upstream's `Main` uploads to codecov through a later job; nothing of that here.
Commands are otherwise verbatim; the quality steps (format, lint, editorconfig-checker) are part of upstream's `Main` and kept.

## Third-party actions needed
None (setup-bun replaced; see change 1).

## Expected duration on GitHub's 4-vCPU runner (upstream, ci.yml on `main`, 12 successful runs, 2026-09-04 → 2026-09-15)
Upstream's jobs run without caches (setup-node with node-version-file only), so these are cold figures.
- `Main`: min 62 s, median 80.5 s, max 89 s. Per run (run id: seconds): 34941604684: 65 · 34828612353: 81 · 34754790412: 86 ·
  34685211551: 78 · 34684652499: 85 · 34684333179: 83 · 34422911556: 68 · 34421847824: 80 · 34212998975: 85 ·
  33910199484: 89 · 33909889617: 80 · 33908918127: 62.
- `Bun`: min 8 s, median 11.5 s, max 14 s (same runs: 13, 12, 14, 12, 10, 10, 13, 8, 9, 11, 13, 8).
- For reference, `Node.js v24.x` (not replicated): 14–20 s.
Expected here: `bench` ≈ 70–100 s per leg, `bench-bun` ≈ 10–20 s; on `icr-2c` roughly twice `Main`'s test/lint time.

## Memory / disk risks
Small: node_modules ≈ 0.5 GB; `tsc` over src ≈ 1–1.5 GB RSS; vitest spawns one worker pool per project sized by cpus (a few
hundred MB each); the workerd project starts Cloudflare's `workerd` binary from node_modules. Fits `icr-2c` (8 GiB) easily.

## What may fail on the ICR image, and the fix applied
- No `unzip` for Bun's install script → guarded apt install (no-op where present).
- `bun` on PATH differs from the pinned 1.2.19 → `$HOME/.bun/bin` prepended via `$GITHUB_PATH` so `bun install --frozen-lockfile`
  reads `bun.lock` with the version that wrote it (a newer bun could want to rewrite the lockfile and fail the frozen check).
- Node 24.7.0 is not in any tool cache (GitHub caches 24.20.0) → setup-node downloads it from nodejs.org on every leg, as
  upstream does (a few seconds, identical work on every leg).
- The `workerd` and `fastly` vitest projects run their runtimes from node_modules (x86-64 glibc binaries): expected fine on
  Ubuntu 26.04; if the workerd binary fails to start on a leg, the `workerd` project fails while the others still run — the log
  and `coverage-final.json` will show it.
