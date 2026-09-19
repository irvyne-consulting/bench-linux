# petclinic — facts

- Upstream: `spring-projects/spring-petclinic`, default branch `main`, pinned `818c4136ea971c21674525f9053de0d9c7ad8cfe`
  (2026-08-26, "docs: Fix formatting in Docker container instructions").
- Build: Maven — `./mvnw` (wrapper 3.3.4, `only-script`, downloads Apache Maven 3.9.16 from repo.maven.apache.org),
  Spring Boot parent 4.1.0, `java.version` 17. A Gradle build (`build.gradle`, `gradlew`) exists as well.

## Upstream job replicated
`.github/workflows/maven-build.yml` "Java CI with Maven", job `build`, its only matrix cell `java: '17'`, `ubuntu-latest`:
1. `actions/checkout@v4`
2. `actions/setup-java@v4` — `java-version: 17`, `distribution: 'adopt'`, `cache: maven`
3. `./mvnw -B verify`

What `verify` runs (pom): enforcer (Java ≥ 17), Spring Java Format `validate`, Checkstyle with nohttp on the whole tree,
compile, Surefire — 20 test classes: MockMvc/controller tests, `PetClinicIntegrationTests`, `PetClinicConcurrencyTests`,
`MySqlIntegrationTests` (Testcontainers `mysql:9.7`, `@Testcontainers(disabledWithoutDocker = true)`),
`PostgresIntegrationTests` (spring-boot-docker-compose starts the `postgres` service of `docker-compose.yml`,
`postgres:18.4`; `assumeTrue(Docker available)`) — JaCoCo report (`prepare-package`), `build-info`, git-commit-id,
CycloneDX SBOM, jar.

Not replicated: `gradle-build.yml` (`./gradlew build`, needs third-party `gradle/actions/setup-gradle@v4`),
`deploy-and-test-cluster.yml` (kind cluster from `k8s/`, runs only on `k8s/**` changes — not a build/test job).

## Changes versus upstream
| change | why |
|---|---|
| `actions/checkout@v7` with `repository`, `ref`, `persist-credentials: false` | harness convention, pinned commit |
| `actions/setup-java@v5`, `distribution: temurin` instead of `adopt` | `adopt` is the deprecated AdoptOpenJDK alias and is fetched from its API; Temurin 17.0.20+1 sits in the tool cache of both images — no download, same JDK on every leg |
| `cache: maven` removed | cold by design: `~/.m2` empty, every artifact from Maven Central |
| "Toolchain" step (`java -version`) | prints the JDK actually used (the image default is 17 on 24.04 but 25 on 26.04) |
| Evidence step + `actions/upload-artifact@v7` (`if: always()`, 90 days) | Surefire XML, JaCoCo XML, `java -version` + `mvnw -v`, image digests |

Command unchanged: `./mvnw -B verify`.

## Third-party actions needed
none

## Expected duration on GitHub's 4-vCPU runner
Upstream job `build (17)`, 12 most recent successful runs (all with a warm `~/.m2` restored by `actions/cache`):
32960914180 **88 s** (the pinned commit itself; step "Build with Maven Wrapper" 78 s) · 32960779817 104 · 30537547257 150 ·
30537514744 109 · 30537469135 173 · 29521922824 99 · 29515647146 90 · 28808898314 107 · 27825705695 83 · 27825695505 105 ·
27822461468 92 · 27811618841 121.
**min 83 s** (27825705695) · **median 104–105 s** · **max 173 s** (30537469135).
Cold, as this workflow runs it: add the Maven distribution (~10 MB), ~300–400 MB of dependencies and plugins, and the
`mysql:9.7`, `postgres:18.4`, `testcontainers/ryuk` image pulls → expect **3–5 min** on GitHub, the first minutes dominated
by downloads. `timeout-minutes: 30`.

## Memory / disk
Maven JVM + Spring Boot test contexts ≈ 1.5–2 GB, MySQL + Postgres containers ≈ 0.5–1 GB → comfortable on icr-2c (8 GiB).
Disk ≈ 1.5 GB: `~/.m2` ≈ 400 MB, images ≈ 0.8 GB, `target/` ≈ 100 MB.

## What may fail on the ICR image, and the fix
- JDK: Temurin 17 is in the ICR toolset (GitHub toolset versions) → `setup-java` resolves from the tool cache. If it were
  absent, `setup-java` downloads it (≈ 190 MB) — visible as that step's duration, not in the build step.
- Docker: Testcontainers and spring-boot-docker-compose need the Docker socket and `docker compose` v2 — both images have
  them. If Docker were unusable the two DB tests are *skipped*, not failed (`disabledWithoutDocker`, `assumeTrue`): check
  `images.txt` and the Surefire XML (skipped count) in the evidence when comparing legs.
- Docker Hub anonymous pull limit (10 pulls/hour/IPv4): 3 pulls per job; the three ICR legs behind one egress IP make 9 per
  campaign — near the limit if laravel's service containers run in the same hour. Mitigation is campaign spacing or a Docker
  Hub login (not in this workflow).
- Image tags used by upstream's tests are not digest-pinned and were left as upstream wrote them (test code and
  `docker-compose.yml`): today `mysql:9.7` = `sha256:29abb0a179982e4a8928138bfc7f918af9eda64e7eeb1b1d084c1720a20159e6`,
  `postgres:18.4` = `sha256:a02db8cac496f15b094798a38254f14d6e00741f709360e5e00bb6668ea31636`; the evidence records what was
  actually pulled on each leg.
- No sudo, no apt, no browser needed.
