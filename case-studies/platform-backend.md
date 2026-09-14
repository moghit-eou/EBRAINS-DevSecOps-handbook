# Case Study: `platform-backend`

[`platform-backend`](https://github.com/Medical-Informatics-Platform/platform-backend)
(Medical Informatics Platform org on GitHub) is one of the two reference
implementations this handbook is built against, a Maven, Spring Boot
service with a small amount of Python mixed in for the CI orchestrators
themselves (`ci/*.py`, see
[reference/python-orchestrators.md](../reference/python-orchestrators.md)).
This case study walks through the concrete, named investigation that
shaped the SCA pipeline's design, in particular why it scans an SBOM
instead of the raw Maven cache.

For the companion npm-side investigation (a malware false positive caused
by a PURL casing collision), see
[case-studies/platform-ui.md](platform-ui.md).

## The starting problem

Early local testing pointed OSV Scanner at a Maven cache instead of an
SBOM. Two scopes were tried, and neither was correct.

| Scope | Critical findings | Why it is wrong |
|---|---|---|
| Cache populated by this project only | 46 | A directory listing, not the resolved dependency graph |
| Full `~/.m2/repository` cache | 91 | Also holds artifacts from every other project on the machine |

A local repository is a download cache, not a dependency list.

Pointed at a directory, the scanner reads every `pom.xml` it finds,
including the POMs of transitive dependencies Maven downloaded while
working out the graph. Each of those declares its own dependencies, so the
scanner walks trees the build never resolved.

It also sees versions the build threw away. Two of your dependencies can
each pull in the same library at a different version. Maven keeps only
one, through dependency mediation: the version declared closest to your
project wins, and ties are broken by declaration order. Both versions stay
on disk, but only the winner is compiled and packaged. You can decide
which one wins ahead of time with `dependencyManagement` or by importing a
BOM, though neither changes what sits in `~/.m2`. See the
[official Maven documentation on the dependency mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html).

The SBOM is generated after Maven has resolved the graph, so it records
the outcome: one entry per library, the version that actually ends up
compiled into the project. Discarded versions and unrelated cache
artifacts are absent by construction. That is why these pipelines scan the
SBOM instead of a directory. See
[explanation/why-sboms.md](../explanation/why-sboms.md).

## The worked example

The specific case that made the mechanism concrete: `platform-backend`
depends on `micrometer-core 1.16.5`. Micrometer's own POM file, bundled in
the Maven cache, declares its own dependency on `tomcat-embed-core
8.5.100`. But `platform-backend`'s own `pom.xml` pins `tomcat.version` to
`11.0.22`. Maven's own resolution (nearest-definition, first-come-wins,
BOM order, see [explanation/why-sboms.md](../explanation/why-sboms.md) for
the mechanics) picks `11.0.22` as the version actually used. The generated
SBOM reflects that resolution: it contains `11.0.22`, and
`tomcat-embed-core 8.5.100` never appears in it. Scanning the SBOM with
OSV-Scanner against `11.0.22` came back clean, correctly, because that's
what actually ships.

## The Maven Central rate limit

While testing Trivy against artifacts not yet in the local cache, Maven
Central began returning HTTP 429 (Too Many Requests), it interpreted the
volume of requests from a single CI-runner IP as abuse. This is
documented, with Trivy's own recommended mitigation
(`--offline-scan`), in
[how-to/troubleshooting.md](../how-to/troubleshooting.md).
This is also part of why `mvn dependency:resolve` runs as its own explicit
CI step ahead of SBOM generation, see
[reference/pipeline-sca.md](../reference/pipeline-sca.md), rather than
letting Trivy resolve artifacts on demand during the scan itself.

## What this shaped in the final pipeline

- SCA scans the generated SBOM (`target/bom.json`), not the raw
  dependency cache, for every ecosystem, not just Maven.
- `mvn dependency:resolve -q` runs as an explicit, separate CI step before
  SBOM generation.
- Both Trivy and OSV-Scanner run against the SBOM, gated through the
  shared `parse_sarif.evaluate()` CVSS-score model.
- The `~/.m2/repository` cache (not the whole `~/.m2` directory, which
  risks cache corruption) is cached between CI runs, keyed on
  `hashFiles('**/pom.xml')`.

See [reference/pipeline-sca.md](../reference/pipeline-sca.md) for the
resulting workflow, and
[explanation/why-sboms.md](../explanation/why-sboms.md) for the general
explanation this case study is the evidence for.

## Code snapshot: `platform-backend`'s actual `sca.yml`

This is a trimmed version of the real, currently-running workflow (Action
`uses:` pins shortened for readability; the live file pins full commit
SHAs, see [tool-installation-flags.md](../reference/tool-installation-flags.md)
for why). It's the concrete instance of the
[generic SCA template](../reference/pipeline-sca.md#generic-github-actions-template),
with the Maven block filled in:

```yaml
name: Software Composition Analysis (SCA)

on:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1' # weekly SCA scan, every Monday 02:00 UTC

jobs:
  sca:
    name: Software Composition Analysis
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      SBOM_PATH: target/bom.json
      TRIVY_IGNOREFILE: ci/suppress_trivy.yaml
      OSV_IGNOREFILE: ci/suppress_osv_scanner.toml
      TRIVY_SARIF_OUTPUT: trivy-platform-backend.sarif
      OSV_SARIF_OUTPUT: osv-scanner-platform-backend.sarif
      SCA_MERGED_SARIF_OUTPUT: SCA-platform-backend-merged.sarif

    steps:
      - name: Check out repository
        uses: actions/checkout@v7

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      # --- the Maven-specific block; this is the only part that differs from platform-ui's npm version ---
      - name: Cache Maven packages
        uses: actions/cache@v6
        with:
          path: ~/.m2/repository # /.m2 alone is too broad and can cause cache corruption issues
          key: ${{ runner.os }}-m2-v1-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2-v1-

      # No separate resolve step: setup-tools.sh runs `mvn dependency:resolve`
      # itself as part of SBOM generation.

      # -------------------------------------------------------------------------------------------------------

      - name: Setup tools
        run: bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven

      - name: Run SCA tools
        run: python ci/sca_scan.py

      - name: Upload Trivy SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: ${{ env.TRIVY_SARIF_OUTPUT }}
          category: trivy-app

      - name: Upload OSV Scanner SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: ${{ env.OSV_SARIF_OUTPUT }}
          category: osv-scanner-app

      - name: Upload SARIF artifacts
        if: always()
        uses: actions/upload-artifact@v7
        with:
          name: sca-scan-sarif-report
          path: ${{ env.SCA_MERGED_SARIF_OUTPUT }}
          retention-days: 30
```

Compare this to
[`platform-ui`'s npm version](platform-ui.md#code-snapshot-platform-uis-actual-scayml):
every step outside the marked Maven block is byte-for-byte identical. This
is the concrete proof, not just the claim, behind
[integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md)'s
statement that only the cache step and the install/resolve command change
between ecosystems.

