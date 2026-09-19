# vite — facts

- Upstream: `vitejs/vite`, default branch `main`, pinned `e9078f865cdff6bed77cd729214a7e2868f126b5`
  (2026-09-18, "docs: size the version check input to its content (#23505)").
- Build: pnpm monorepo — `packageManager: pnpm@12.4.2`, `engines.node ^20.19.0 || >=22.12.0`, TypeScript ~6.0, Vitest ^4.1,
  rolldown ~1.2 / rollup ^4.59; workspaces `packages/*` (`vite`, `create-vite`, `plugin-legacy`), `playground/**`, `docs`.
  `pnpm-workspace.yaml` patches three packages and allows native builds (esbuild, sharp, workerd, @parcel/watcher, …).

## Upstream job replicated
`.github/workflows/ci.yml` "CI", job `test` — "Build&Test: node-${{ matrix.node_version }}, ubuntu-latest", matrix
`node_version: [20, 22, 24, 26]` on `ubuntu-latest` (+ macOS 24, Windows 24.15.0), `timeout-minutes: 20`, workflow env
`NODE_OPTIONS=--max-old-space-size=6144`, `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1`, `VITEST_SEGFAULT_RETRY=3`. Steps:
`actions/checkout` (`persist-credentials: false`) · `pnpm/action-setup` v6.1.0 (version from `packageManager`) ·
`actions/setup-node` (`cache: pnpm`) · `pnpm install` · Playwright path + `actions/cache` · `pnpm playwright install chromium` ·
`pnpm run build` · `pnpm run test-unit` · `pnpm run test-serve` · `pnpm run test-serve-bundled` · `pnpm run test-build`.
The job is skipped when only docs/templates changed (`changed` job with `tj-actions/changed-files`).

Cell replicated: **Node 24, `ubuntu-latest`** — `pnpm install` → `pnpm run build` (= `pnpm -r --filter='./packages/*' run
build`) → `pnpm run test-unit` (= `vitest run`; `vitest.config.ts`: `**/__tests__/**/*.spec.[tj]s` outside `playground/`,
`isolate: false`, 20 s timeout). Not replicated, per the brief: `pnpm playwright install chromium` and the three playground
suites `test-serve`, `test-serve-bundled`, `test-build` (`vitest.config.e2e.ts`, Playwright + Chromium).

## Changes versus upstream
| change | why |
|---|---|
| `actions/checkout@v7` with `repository`, `ref`, `persist-credentials: false` | harness convention |
| Node **24** rather than 26 (the newest cell upstream tests) | 24 is the Active LTS, the version upstream builds every cell with and its macOS cell; it is in the tool cache of both images (22.x, 24.x) — Node 26 is in neither (a download on every leg) and Node ≥ 25 no longer bundles corepack |
| `pnpm/action-setup` → `corepack enable && corepack prepare pnpm@12.4.2 --activate` | third-party action replaced by a plain command, same pnpm version as `packageManager` |
| `actions/setup-node@v7` without `cache: pnpm` | cold by design: empty pnpm store, everything from registry.npmjs.org |
| Playwright cache and `pnpm playwright install chromium` dropped | not used by the unit tests; Chromium's system libraries are not on the ICR image |
| `pnpm run test-unit --reporter=default --reporter=junit --outputFile.junit=$RUNNER_TEMP/evidence/vitest-unit.xml` | JUnit evidence; the reporters only add output. No `--`: pnpm declares the script name as its "escape arg" (everything after it goes to the script verbatim, `pnpm run echo --test` → `--test` reaches the script — cli/parse-cli-args), and a literal `--` would be forwarded to vitest, which would then ignore the flags |
| `COREPACK_ENABLE_DOWNLOAD_PROMPT=0` | no interactive prompt from corepack |
| Evidence + `actions/upload-artifact@v7` (`if: always()`, 90 days) | Vitest JUnit XML, `node`/`pnpm` versions, `pnpm ls -w --json` |

Upstream's env (`NODE_OPTIONS`, `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD`, `VITEST_SEGFAULT_RETRY`) kept, at job level.

## Third-party actions needed
none

## Expected duration on GitHub's 4-vCPU runner
Upstream job "Build&Test: node-24, ubuntu-latest", 8 most recent successful main runs in which `test` ran (warm pnpm store
and Playwright cache): 35303596922 238 s · 35300843734 241 · 35300714240 258 · 35299841220 234 · 34917609468 191 ·
34470796579 257 · 34431734984 208 · 34189569956 241 → **min 191 · median 239–241 · max 258 s**.
Node-26 cell, same runs: min 201 · median 245 · max 288 s (34431734984).
Steps in 35303596922 (node-24): Install deps 12 s (cached store), Build 9 s, **Test unit 8 s**, Test serve 78 s,
Test serve (bundled dev) 55 s, Test build 62 s. The replicated subset is ≈ 30 s of that 240-s job; a cold `pnpm install`
(≈ 1,700 packages, native binaries for rolldown, esbuild, sharp, workerd) adds 40–90 s → expect **1.5–3 min** per job. It
is a short, install-heavy measurement: the full job with Chromium (≈ 4 min) is the alternative if Chromium's libraries are
ever added to the ICR image (then add `pnpm playwright install chromium` and the three suites back). `timeout-minutes: 30`.

## Memory / disk
`NODE_OPTIONS=--max-old-space-size=6144` (upstream) allows a 6 GiB heap — fine on icr-2c (8 GiB); actual use ≈ 2 GB
(build + Vitest threads). Disk ≈ 1.5 GB (pnpm store + `node_modules`).

## What may fail on the ICR image, and the fix
- Node 24 is in the ICR tool cache (GitHub toolset) → `setup-node` resolves locally; `corepack` is bundled with Node 24
  (and on PATH on ICR). `corepack enable` writes the `pnpm` shim next to the tool-cache `node` binary — writable by the
  runner on GitHub's images; the ICR tool cache must be writable too (`setup-node` needs it anyway). Fallback if not:
  `npm install -g pnpm@12.4.2` in the "Install pnpm" step.
- `preinstall: npx only-allow pnpm` fetches `only-allow` (tiny) from the registry; `postinstall: simple-git-hooks` needs
  `.git` (the checkout provides it).
- `pnpm install` is `--frozen-lockfile` in CI by default (`CI=true` on both runners); `minimumReleaseAgeStrict` /
  `trustPolicy` in `pnpm-workspace.yaml` concern lockfile-free resolution only.
- No Docker, no sudo, no browser.
