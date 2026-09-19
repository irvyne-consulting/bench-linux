> This workflow targets **seed4j/seed4j** (`main` @ `2adf56c6aacc1c57a1f1ad48f036c9f773161d60`, 2026-09-19, Temurin 25 + Node 24), the maintained successor of jhipster-lite, which is archived since 2025-08. The notes below were written against jhipster-lite at its final commit and describe the same `tests-linux` job; the five retargeted fields are listed in them.

# jhipster-lite — facts

- Upstream: `jhipster/jhipster-lite`, default branch `main`, pinned `88cba4a609adc47b47aeec914e7ec4723768d854`
  (2025-08-02, "add forked information" — the repository's final commit).
- **The repository is archived** (GitHub `archived: true`; README: "This project will no longer be maintained, as it has
  been forked and will now be maintained at seed4j/seed4j"). Its Actions history is gone (0 runs retained). The successor
  `seed4j/seed4j` (`main` @ `2adf56c6aacc1c57a1f1ad48f036c9f773161d60`, 2026-09-19) keeps the same layout and the same
  `tests-linux` job, now on Java 25 / Node 24 / Spring Boot 4.0.6. Retargeting this workflow = `UPSTREAM`, `UPSTREAM_REF`,
  `repository:`, `ref:` and `java-version: '25'` (Temurin 25 is in both tool caches).
- Build: **Maven, not Gradle** — `pom.xml`, `./mvnw` (wrapper → Apache Maven 3.9.11), Spring Boot 3.5.4, `java.version` 21;
  front end Vue 3.5 / Vite 7 / Vitest 3.2 / Cypress 14.5.3, built *inside* the Maven build by `frontend-maven-plugin` 1.15.1
  (installs Node v22.17.1 + npm 11.5.1 under `target/`). `package.json` `engines.node >= 20`; `.npmrc` `save-exact=true`.

## Upstream job replicated
`.github/workflows/github-actions.yml` "build", job `tests-linux` (`ubuntu-latest`, `timeout-minutes: 20`) — the Linux
build+test job (the others: `tests-windows`, `cypress` e2e on the built jar, the 27-cell `generation` matrix generating and
building sample apps, `status-checks`, `dependabot-auto-merge`):
1. `actions/checkout@v4` (`fetch-depth: 0`)
2. `./.github/actions/setup` (composite): `actions/setup-node@v4` `lts/*` + `check-latest`, `npm install -g npm`,
   `actions/setup-java@v4` `temurin` `21` + `check-latest`, `sed` adding `<interactiveMode>false</interactiveMode>` to
   `~/.m2/settings.xml`, `node -v && npm -v && java -version`
3. `actions/cache@v4` on `~/.m2/repository`
4. `npm ci`
5. `npm run lint:ci` (= `eslint . && prettier --check .`)
6. `docker compose -f src/main/docker/sonar.yml up -d` (SonarQube 25.7 community + a locally built token container)
7. `chmod +x mvnw` · `./mvnw clean verify -Dsonar.qualitygate.wait=true`
8. `./mvnw initialize sonar:sonar -Dsonar.token=…` · `./tests-ci/sonar.sh` (twice) · SonarCloud analysis (main only) ·
   `actions/upload-artifact` of the jar for the `cypress` and `generation` jobs

Replicated, same order, same commands: **`npm ci` · `npm run lint:ci` · `./mvnw -B clean verify`**.

What `./mvnw clean verify` runs (pom): enforcer (Maven ≥ 3.6.3, dependency convergence) → Checkstyle (`validate`) →
compile with Error Prone (`.mvn/jvm.config` add-exports) → `frontend-maven-plugin`: install Node v22.17.1/npm 11.5.1,
`npm install`, `npm run build` (tikui + `vue-tsc` + `vite build`) → Surefire: 261 `*Test` classes (JUnit 5, Cucumber,
ArchUnit; `runOrder` alphabetical) → `npm run test:unit:coverage` (Vitest, 35 specs, jsdom, istanbul, 2 threads) →
Failsafe: 7 `*IT` Spring Boot tests incl. `MongoDBStatisticsRepositoryIT` (Testcontainers `mongo:5.0.11`, Linux only) →
`npm run test:component:headless` (Vite dev server on :9000 + Cypress 14.5.3 component tests in headless Electron) →
`npm run test:coverage:check` (nyc merge/report) → JaCoCo merge/report and `check` (profile `jacoco-check`, active on
unix — upstream's own 100 % gate) → modernizer → javadoc and sources jars → Spring Boot repackage.

## Changes versus upstream
| change | why |
|---|---|
| `actions/checkout@v7`, `repository`/`ref` pinned, `persist-credentials: false`, default depth instead of `fetch-depth: 0` | harness convention; the full history served Sonar's blame only |
| SonarQube containers, `-Dsonar.qualitygate.wait=true`, `sonar:sonar`, `tests-ci/sonar.sh`, SonarCloud, jar upload dropped | analysis and publishing, not build or test; the property is read by the sonar goal only, `verify` ignores it |
| `./mvnw -B clean verify` | `-B` replaces the composite action's `sed -i` on `~/.m2/settings.xml` (the file exists on GitHub's image, not on ICR — `sed -i` would fail there) |
| `actions/setup-java@v5` temurin `21`, `actions/setup-node@v7` `24`; no `check-latest`, no `npm install -g npm`, no `actions/cache` | Temurin 21.0.12 and Node 24.x are in both tool caches (no download); `lts/*` resolves to Node 24 today; `check-latest` and `npm install -g npm` fetch whatever is newest on the day → drift between campaigns; cold by design |
| `chmod +x mvnw` dropped | `mvnw` is mode 100755 in git |
| "Toolchain" step | prints JDK / Node / npm actually used |
| "Cypress system libraries when the image lacks them" step | see below — a no-op on GitHub-hosted legs |
| Evidence + `actions/upload-artifact@v7` (`if: always()`, 90 days) | Surefire/Failsafe XML, JaCoCo XML, Vitest report (sonar XML format), tool versions, image digests |

## Third-party actions needed
none (`pnpm/action-setup` in upstream's composite is only reached for pnpm-based generated apps, not by `tests-linux`)

## Expected duration on GitHub's 4-vCPU runner
No run of `jhipster/jhipster-lite` is retained (archived 2025-08). **Proxy: `seed4j/seed4j` `tests-linux`** — same steps
plus Sonar, warm `~/.m2` — 4 most recent successful main runs:
35412821252 **604 s** (`npm ci` 23 s, `lint:ci` 31 s, `./mvnw clean verify` 270 s, Sonar start 17 s + analyses 85 s + 135 s) ·
35412572050 **516 s** (18 / 23 / 224 s; Sonar 27 + 64 + 119 s) · 35412331104 **509 s** · 35311310874 **636 s** →
min 509 · median 560 · max 636 s, of which ≈ 4 min is Sonar (absent here).
This workflow, cold: `npm ci` 40–90 s (≈ 1,500 packages + the Cypress 14.5.3 binary ≈ 250 MB from download.cypress.io),
lint 25–40 s, `./mvnw clean verify` 5–8 min (Maven Central downloads ≈ 1–2 min, Node v22 tarball, `mongo:5.0.11` pull) →
expect **7–11 min** on 4 vCPU, perhaps 15–20 min on icr-2c. `timeout-minutes: 45`.

## Memory / disk
Peak ≈ 4–5 GB: Maven JVM (~1 GB) + Surefire fork with Spring contexts (~1.5 GB) + Vite dev server and Cypress/Electron
(~1.5 GB) + MongoDB container (256 MB WiredTiger cache) — fits icr-2c (8 GiB) but it is the tightest of the three
projects. Disk ≈ 3 GB: `node_modules` ≈ 1 GB, `~/.cache/Cypress` ≈ 0.7 GB, `~/.m2` ≈ 0.7 GB, `target/` ≈ 0.5 GB,
`mongo:5.0.11` ≈ 0.25 GB.

## What may fail on the ICR image, and the fix
- **Cypress** (inside `mvnw verify`): headless Electron needs `Xvfb` and the GTK 3 / NSS / ALSA shared libraries.
  GitHub's 24.04 and 26.04 images have them (Chrome 152, `xvfb` in the apt toolset); the ICR image lists no browser or X →
  fix applied: a guarded step runs `sudo apt-get install -y --no-install-recommends xvfb libgtk-3-0t64 libgbm1 libnss3
  libasound2t64 libxss1 libnotify4 libxtst6 xauth` only when `Xvfb`, `libgtk-3.so.0`, `libnss3.so` or `libasound.so.2`
  is missing (no-op on GitHub). Needs passwordless sudo on ICR (the laravel benchmark's PHP install already relied on it);
  the install time is that step's own duration, outside the workload steps. Package names are the 24.04 `t64` names,
  unchanged in 26.04. If a name were wrong on 26.04 the step fails fast, before any workload.
- JDK 21 / Node 24: both in the ICR toolset → no download. The Maven build downloads its own Node v22.17.1 (nodejs.org)
  on every leg — upstream's design.
- `~/.m2/settings.xml` absent on ICR → `-B` instead of upstream's `sed` (see above).
- Docker: `MongoDBStatisticsRepositoryIT` pulls `mongo:5.0.11`
  (`sha256:e7a7fdda54cdfdb217d82af7f63cb6e0506bb010208aae7e5a99331ced7de9ed`) and `testcontainers/ryuk` — 2 anonymous
  Docker Hub pulls per job (limit 10/hour/IPv4; the three ICR legs share an egress IP).
- Archived snapshot: Maven and npm artifacts are immutable, but the Cypress binary (cypress.io) and Node v22.17.1
  (nodejs.org) are fetched live — an availability risk, not a content one.
- Upstream's own gates run unchanged (JaCoCo 100 %, ArchUnit, Checkstyle, Error Prone): a failure of one of them fails the
  job, the step timings and the evidence stay usable.
