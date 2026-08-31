# How to Integrate the Security Pipeline Into a New Repository

This is the single, ecosystem-agnostic guide for wiring Container Scanning,
SCA, and SAST into any repository, regardless of build tool or language.

Maven, npm, and Go are the ecosystems the pipelines actually run in
production and in CI. Anything else plugs into the same pattern, see
[Adding a new ecosystem](#adding-a-new-ecosystem).

## What actually changes per ecosystem

Only two things in the whole pipeline are ecosystem-specific. Everything
else (`sca_scan.py`, `sast_scan.py`, `container_scan.py`, `parse_sarif.py`,
the gate models, the suppression files) is already language-agnostic and
requires **no changes**:

| What | Ecosystem-specific? | Where it lives |
|---|---|---|
| Resolving dependencies, then generating the CycloneDX SBOM | Yes | The `--sbom-ecosystem` branch in `setup-tools.sh` |
| Caching the dependency store between runs | Yes | The `actions/cache` step in `sca.yml` |
| Scanning the SBOM (Trivy, OSV-Scanner) | **No** | `sca_scan.py` reads `SBOM_PATH`; it has no idea what produced the file |
| SAST of source code (OpenGrep) | Partially | Same tool everywhere; only the ruleset/exclude list differs, see [pipeline-sast.md](../reference/pipeline-sast.md) |
| Container Scanning | **No** | Scans the built image and the `Dockerfile`; independent of language |

> **Resolution currently happens in `setup-tools.sh`.** Each
> `--sbom-ecosystem` branch resolves dependencies and then generates the
> SBOM: `mvn dependency:resolve`, `npm ci`, `go mod download`. It could
> equally be a step in `sca.yml`. Keeping it in the script means a local
> run resolves the same way CI does.

## Ecosystem matrix

Cache paths and lockfiles for the `actions/cache` step, canonical shape in
[pipeline-sca.md](../reference/pipeline-sca.md#caching):

| Ecosystem | Cache `path` | Lockfile hashed in the `key` |
|---|---|---|
| Maven | `~/.m2/repository` | `**/pom.xml` |
| npm | `~/.npm` | `**/package-lock.json` |
| Go | `~/go/pkg/mod`, `~/.cache/go-build` | `**/go.sum` |
| Gradle | `~/.gradle/caches`, `~/.gradle/wrapper` | `**/*.gradle*` |

<details>
<summary><strong>Maven</strong></summary>

```bash
maven)
  echo "Generating SBOM for Maven project ... this may take a while"
  mvn -B -ntp dependency:resolve -q
  mvn -B -ntp org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q
  ;;
```

**SBOM output:** `target/bom.json`

**Local run:**
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py
```

`dependency:resolve` must run first: the CycloneDX plugin builds the SBOM
from Maven's **resolved** dependency tree, not from `pom.xml` declarations
directly. See [why-sboms.md](../explanation/why-sboms.md).

</details>

<details>
<summary><strong>npm</strong></summary>

```bash
npm)
  echo "Generating SBOM for NPM project ... this may take a while"
  npm ci
  npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json
  ;;
```

**SBOM output:** `target/bom.json`

**Local run:**
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

`npm ci`, never `npm install`: it installs strictly from
`package-lock.json`, wipes `node_modules` first, and fails on a
lockfile mismatch instead of rewriting the lockfile. `cyclonedx-npm` walks
the installed `node_modules/` tree, so an `npm install` can make the SBOM
reflect a different dependency set than what is locked. A committed
`package-lock.json` is required. See
[troubleshooting.md](troubleshooting.md#npm-elsproblems--peer-dependency-errors-during-sbom-generation).

</details>

<details>
<summary><strong>Go</strong></summary>

```bash
golang|go)
  echo "Generating SBOM for Go project ... this may take a while"
  mkdir -p target
  go mod download
  go install "github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@${CYCLONEDX_GOMOD_VERSION}"
  "$(go env GOPATH)/bin/cyclonedx-gomod" mod -json -output target/bom.json
  ;;
```

**SBOM output:** `target/bom.json`

**Local run:**
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem golang
python ci/sca_scan.py
```

`cyclonedx-gomod` reads `go.sum`, the resolved and checksummed module set,
not `go.mod`'s version constraints, so it has the same resolved-versus-
declared property as the Maven and npm paths.

</details>

<details>
<summary><strong>Gradle</strong> (example, not yet run in production)</summary>

Add the plugin to `build.gradle`:

```groovy
plugins {
    id 'org.cyclonedx.bom' version '2.3.1'
}
```

Then add the branch:

```bash
gradle)
  echo "Generating SBOM for Gradle project"
  mkdir -p target
  ./gradlew cyclonedxBom -q
  cp build/reports/bom.json target/bom.json
  ;;
```

**SBOM output:** `build/reports/bom.json`, copied to `target/bom.json` so
`SBOM_PATH` does not need to vary per ecosystem.

**Local run:**
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem gradle
python ci/sca_scan.py
```

Like Maven, the SBOM comes from Gradle's own dependency resolution rather
than from reading `build.gradle` declarations. Check the
[plugin's documentation](https://github.com/CycloneDX/cyclonedx-gradle-plugin)
for the current version and output path before using this.

</details>

## The generic fallback: Trivy filesystem scan

`--sbom-ecosystem generic` skips the language toolchain entirely and lets
Trivy build the SBOM by scanning the filesystem:

```bash
generic|auto)
  mkdir -p target
  trivy fs --format cyclonedx --output target/bom.json .
  ;;
```

Trivy finds dependencies by recognising lockfiles it already supports
(`Cargo.lock`, `requirements.txt`, `Gemfile.lock`, `composer.lock`, and
others), so coverage depends on whether your lockfile format is one of
them. It is the quickest way to cover an ecosystem, not the most accurate:
there is no resolution step, so the result reflects what the lockfile
declares rather than what the build resolves. See
[why-sboms.md](../explanation/why-sboms.md).

> **Availability.** The `generic` branch exists in the blueprint
> repository only. Neither `platform-backend` nor `platform-ui` needs it,
> since both have a native generator, so it was never added to their
> vendored copy of `setup-tools.sh`. Copy the branch across if you want it.
> See [reference/reusable-blueprint.md](../reference/reusable-blueprint.md).

## The generic `sca.yml` template

```yaml
name: Software Composition Analysis (SCA)

on:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'

jobs:
  sca:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      SBOM_PATH: target/bom.json
      TRIVY_IGNOREFILE: ci/suppress_trivy.yaml
      OSV_IGNOREFILE: ci/suppress_osv_scanner.toml
      TRIVY_SARIF_OUTPUT: trivy-<service>.sarif
      OSV_SARIF_OUTPUT: osv-scanner-<service>.sarif
      SCA_MERGED_SARIF_OUTPUT: SCA-<service>-merged.sarif
      # CVSS gate: fail at >= FAIL, warn in between. Override per project.
      GATE_FAIL_THRESHOLD: "8.0"
      GATE_WARN_THRESHOLD: "5.0"

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      # --- ecosystem-specific: cache step only ---------------------------
      # No resolve step here, setup-tools.sh runs it in the
      # --sbom-ecosystem branch.
      # - name: Cache dependencies
      #   uses: actions/cache@v6
      #   with: { path: <cache path>, key: ..., restore-keys: ... }
      # -------------------------------------------------------------------

      - name: Setup tools and generate SBOM
        run: bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem <ecosystem>

      - name: Run SCA tools
        run: python ci/sca_scan.py

      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: ${{ env.TRIVY_SARIF_OUTPUT }}
          category: trivy-app

      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: ${{ env.OSV_SARIF_OUTPUT }}
          category: osv-scanner-app

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: sca-scan-sarif-report
          path: ${{ env.SCA_MERGED_SARIF_OUTPUT }}
          retention-days: 30
```

> **Check what your runner already has.** `ubuntu-latest` ships Python,
> a JDK, Maven, Node, and Go, so most ecosystems work with no setup step.
> A different runner, a self-hosted one, or a slimmer image may not. If a
> tool is missing, add the matching step (`actions/setup-java`,
> `actions/setup-node`, `actions/setup-go`, `actions/setup-python`) before
> `setup-tools.sh`. Pinning the version with a setup step is worth doing
> anyway, since runner images change over time.

`sast.yml` and `container-scan.yml` need no ecosystem-specific block beyond
the `SEMGREP_CONFIG_RULESETS` and `OPENGREP_EXCLUDE` values, see
[pipeline-sast.md](../reference/pipeline-sast.md#ruleset-configuration-by-repository).

## Adding a new ecosystem

1. Try `--sbom-ecosystem generic` first. If Trivy already parses your
   lockfile, you may need no new code at all.
2. If not, find a CycloneDX generator in the
   [tool center](https://cyclonedx.org/tool-center/). Confirm it reads the
   **resolved** dependency state, not just manifest declarations, see
   [why-sboms.md](../explanation/why-sboms.md).
3. Add one `case` branch to `setup-tools.sh` that resolves dependencies and
   then writes a CycloneDX file to `SBOM_PATH`.
4. Add a cache step to `sca.yml` keyed on that ecosystem's lockfile.
5. Nothing else changes. `sca_scan.py`, the gate model, and suppression are
   already ecosystem-agnostic.

PHP (Composer), Ruby (Bundler), Rust (Cargo), and .NET (NuGet) all have
CycloneDX generators and would be added exactly this way.

This is the same pattern used for
[adding a new scanner](add-new-scanner.md), applied to ecosystems instead
of tools.