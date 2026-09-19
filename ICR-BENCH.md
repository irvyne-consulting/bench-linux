# ICR benchmark — Linux kernel build

This repository holds one workflow and no kernel source. The workflow downloads a pinned stable kernel from
[kernel.org](https://www.kernel.org/) (`linux-<version>.tar.xz`, default 7.2.6 on 2026-09-19), verifies its SHA-256
against kernel.org's `sha256sums.asc` (both fetched over TLS; the PGP signature on that list is not checked here),
configures it (`x86_64_defconfig` by default, `allmodconfig` on request — on every leg, never on one side only) and builds
`bzImage` and the modules with `make -j$(nproc)`: no ccache, no cache, fixed build metadata (no claim of reproducible
binaries is made). What it measures is **clean-build throughput of a large C code base**, not an application's CI.

Two modes. **Delivered**: each provider's shipped image and toolchain, dependency installation included and timed — what a
customer gets on `ubuntu-latest` (Ubuntu 24.04, gcc 13 at the time), `ubuntu-26.04` (GitHub's newer image) and ICR
(Ubuntu 26.04). **Controlled**: every leg builds inside the same Docker Hub `gcc` image pinned by digest, so compiler,
binutils and libc are identical and the rows compare the same work; the image pull is a timed step on every leg.
Legs: GitHub-hosted `ubuntu-latest` and `ubuntu-26.04` (4 vCPU / 16 GB on a public repository), and
[ICR](https://www.irvyne.eu/icr/) (Irvyne Consulting Runners) `icr-2c`, `icr-4c`, `icr-8c` (2 / 4 / 8 vCPU with
8 / 16 / 32 GiB, one fresh virtual machine per job, destroyed afterwards).

Every step is timed by GitHub itself and read from its API by the publisher (the steps are written out in each job:
a composite action would collapse them into one); each job uploads its evidence (`config`, `versions.txt`,
`packages.txt`, `build.log`, GNU `time -v` output with elapsed time and peak memory) as a run artifact. Several runs
per leg, on a quiet host and under a declared concurrent load; medians and observed maxima are published with the sample
size, every number linking to its run.

**Who can run it.** Only Irvyne: a manual dispatch by a writer, then an approval on the `icr-bench` environment. Pull
requests are switched off, issues and discussions too, interactions are limited to collaborators, and the runners are
visible to this repository alone within the organisation's runner group.

The Linux kernel is the work of Linus Torvalds and thousands of contributors, licensed GPL-2.0; nothing of it is
redistributed here.
