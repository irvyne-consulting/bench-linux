# fastify — facts (drafted 2026-09-19)

## Upstream
- Repository: `fastify/fastify`, default branch `main`.
- Pinned: `630acd0b6cf8a91322ff05c3d95feb991091866d` = head of `main` on 2026-09-19; commit dated 2026-09-16T10:20:30Z,
  message "Bumped v6.0.0-alpha.4". No `packageManager`/`engines` fields, no lockfile (`.npmrc`: `package-lock=false`,
  `ignore-scripts=true`, `min-release-age=7`). No submodules.

## Replicated: `.github/workflows/ci.yml`, job `test-unit`, cell `node-version: 24, os: ubuntu-latest`
Upstream matrix: `node-version: [24, 26]` × `os: [macos-latest, ubuntu-latest, windows-latest]`. Node 24 is the newest LTS
(26 is "Current" until October 2026), as asked. Steps: `actions/checkout` (persist-credentials false) → `actions/setup-node`
(`node-version: 24`, `cache: npm`, `cache-dependency-path: package.json`, `check-latest: true`) → `npm install --ignore-scripts`
→ `npm run unit` = `borp` (borp 1.0.0; `.borp.yaml` files `test/**/*.test.js`, `test/**/*.test.mjs`; Node's built-in test
runner; concurrency default `availableParallelism() - 1`, i.e. 1 / 3 / 7 processes on 2 / 4 / 8 vCPU; timeout 30 s per test;
the `gh` annotations reporter is added automatically under GitHub Actions). The job's upstream prerequisites (`quality-check`
reusable workflow from fastify/workflows, `coverage-nix`, `coverage-win`) are separate jobs and not part of the measurement.

## Changes versus upstream, and why
1. `cache: 'npm'` and `cache-dependency-path` removed from setup-node (cold benchmark); `package-manager-cache: false` added
   (setup-node v6+/v7 auto-cache — only triggers when package.json's `packageManager` names npm, which fastify has not; explicit
   anyway). `check-latest: true` kept as upstream: the newest 24.x is resolved from the Node dist index and downloaded when the
   tool cache is behind — identical behaviour on every leg, a few seconds of variance possible.
2. `actions/setup-node@v7` by tag (upstream pins v7.0.0 by SHA); `actions/checkout@v7` idem.
3. `npm run unit` → `npm run unit -- --reporter=spec --reporter="junit:$RUNNER_TEMP/evidence/junit.xml"`: borp's `--reporter`
   is repeatable and `name:destination` writes to a file (borp.js: `input.split(':')`, `createWriteStream(dest)`); passing any
   `--reporter` drops the default `spec`, hence it is re-added. Same tests, same console output, plus a JUnit file.
4. `Runner facts` and `Evidence` steps added (JUnit, `npm ls --all --json` as the resolved dependency tree — there is no
   lockfile upstream, so this records what `min-release-age=7` resolved on the day — and `versions.txt`).

## Third-party actions needed
None.

## Expected duration on GitHub's 4-vCPU runner (upstream `test-unit (24, ubuntu-latest)`, 12 successful `main` runs, 2026-05-30 → 2026-09-11)
- min 44 s, median 59.5 s, max 68 s. Per run (run id: seconds): 34581002223: 68 · 31245012521: 44 · 31244355412: 68 ·
  31084905149: 68 · 28598794688: 55 · 28384702401: 62 · 28316323401: 57 · 28315918448: 59 · 28315529993: 67 ·
  28313741807: 60 · 28085989072: 55 · 26682991964: 56.
- Upstream restores the npm cache; cold `npm install` of ~700 packages adds an estimated 10–25 s here. Expected ≈ 60–90 s per
  4-vCPU leg; `icr-2c` runs borp with concurrency 1, so its test step is roughly 2–3× longer.

## Memory / disk risks
Small: node_modules ≈ 300 MB, borp workers ≈ 100–200 MB each (1–7 processes). Some tests open many sockets / HTTP/2 sessions;
default `ulimit -n` on Ubuntu images (1024 soft) has been fine upstream.

## What may fail on the ICR image, and the fix applied
- `min-release-age=7` needs npm ≥ 11.x, which Node 24 bundles — setup-node provides the job's Node/npm on every leg, so the
  image's Node version does not matter (printed in `Runner facts` for the record).
- Nothing else known: the job is pure Node, no native compilation (`--ignore-scripts`), no service containers, no third-party
  actions. A leg without outbound access to registry.npmjs.org and nodejs.org would fail at install / setup-node.
