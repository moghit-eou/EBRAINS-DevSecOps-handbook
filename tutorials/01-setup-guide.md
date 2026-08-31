# Tutorial: Set Up and Run the Security Pipelines Locally

This tutorial walks through installing the scanners and running all three
security pipelines (Container Scanning, SCA, SAST) against a repository on
your own machine, start to finish. It uses a generic placeholder
(`<your-service>`) instead of any specific repository name, and covers
every supported ecosystem, not just Maven and npm, so it applies to any
repository set up the same way.

> Maven and npm are shown first below only because they're the two
> ecosystems the reference implementations (`platform-backend`,
> `platform-ui`) actually run, so they're the two commands verified against
> a live repository. Gradle, Python, Go, and Rust are documented right
> alongside them, and if your project uses something else entirely, the
> pipeline isn't limited to any fixed list, see
> [integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md#adding-a-new-ecosystem-not-in-the-matrix)
> for how to add it.

**Platform requirement:** this tutorial covers the native install path,
verified on Linux x86_64 (Ubuntu), matching the `ubuntu-latest` GitHub
Actions runner. On Windows, use WSL2. On macOS, use the dockerized path in
the blueprint repository instead, since `setup-tools.sh` downloads Linux
amd64 binaries. See
[tool-installation-flags.md](../reference/tool-installation-flags.md) for
the full platform table and
[reference/reusable-blueprint.md](../reference/reusable-blueprint.md) for
the `make` interface.

## What you'll need

* A Linux x86_64 environment (native Ubuntu or WSL2).
* `bash`, `curl`, `sudo`, `git`.
* Docker (for the Container Scanning pipeline).
* Your project's own build tool: Maven, Gradle, npm, pip, Go, Cargo, or
  whatever your ecosystem actually uses.
* The `ci/` scripts from your repository: `setup-tools.sh`,
  `container_scan.py`, `sca_scan.py`, `sast_scan.py`, `parse_sarif.py`.

## Step 1: check out the repository

```bash
git clone <your-repo-url>
cd <your-service>
```

## Step 2: Run the SAST pipeline

SAST scans your source code only. It doesn't need Docker or your
dependency manager, and is identical for every ecosystem.

```bash
bash ci/setup-tools.sh --install-tool opengrep,semgrep-rules
python ci/sast_scan.py
```

This installs OpenGrep and clones the pinned `semgrep-rules` community
ruleset, then runs OpenGrep against your source tree twice: once to write
a full SARIF report, once as the actual pass/fail gate. See
[pipeline-sast.md](../reference/pipeline-sast.md) for what each step does,
and for how to choose `SEMGREP_CONFIG_RULESETS` for a language not shown
in the existing examples.

Expect a `sast-opengrep-app.sarif` file (or your project's configured
output name) in the working directory when this finishes.

## Step 3: Run the SCA pipeline

SCA scans your application's declared dependencies, via a generated SBOM,
not your source code and not a container image. The install/resolve step
and the `--sbom-ecosystem` value are the only two things that change
between ecosystems, everything after that is identical.

**Maven:**
```bash
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py
```

**Gradle:**
```bash
./gradlew dependencies --write-locks -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem gradle
python ci/sca_scan.py
```

**npm:**
```bash
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

**Python:**
```bash
pip install -r requirements.txt
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem python
python ci/sca_scan.py
```

**Go:**
```bash
go mod download
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem go
python ci/sca_scan.py
```

**Rust:**
```bash
cargo fetch --locked
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem rust
python ci/sca_scan.py
```

**Anything else** (PHP, Ruby, .NET, or a language not listed above): the
same three-line shape applies, install/resolve dependencies, run
`setup-tools.sh` with your ecosystem name, run `sca_scan.py`. See
[integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md#adding-a-new-ecosystem-not-in-the-matrix)
to add the one missing piece: a `setup-tools.sh` branch that produces a
CycloneDX SBOM for your ecosystem.

For npm specifically, use `npm ci`, not `npm install`, before generating
the SBOM: `npm ci` installs strictly from `package-lock.json`, wipes
`node_modules` first for a clean install, and fails immediately if
`package.json` and `package-lock.json` are out of sync, instead of
silently rewriting the lockfile. If it fails, regenerate the lockfile
locally with `npm install` and commit the result, don't work around the
failure inside the pipeline.

This installs Trivy and OSV-Scanner, generates a CycloneDX SBOM at
`target/bom.json`, then scans that SBOM (not your local dependency cache)
with both tools. See [why-sboms.md](../explanation/why-sboms.md) for why
the SBOM, and not the raw dependency cache, is what gets scanned, and
[integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md) if
your project uses a build tool not listed above (raw JavaScript with no
package manager, for example).

## Step 4: Run the Container Scanning pipeline

Container Scanning covers two different things in one pipeline: the
Dockerfile itself (SAST-style linting) and the built image (SCA-style CVE
scanning). This is identical for every ecosystem, it operates on the image
and the Dockerfile, not on source code or dependency manifests.

```bash
docker build -t <your-image>:local .
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
python ci/container_scan.py --scan-type sast
python ci/container_scan.py --scan-type sca --image <your-image>:local
```

The `--scan-type sast` run scans the `Dockerfile` with Hadolint and
OpenGrep. The `--scan-type sca` run scans the built image with Trivy and
OSV-Scanner. Both use the same `container_scan.py` CLI, just with a
different `--scan-type`.

### Orchestrator CLI reference

View all available arguments for the container orchestrator at any time
with `--help`. This exposes advanced pipeline functionality, such as
merging multiple SARIF reports into a single consolidated artifact.

```bash
python ci/container_scan.py --help
```

**Output:**

```text
usage: sec-orchestrator [-h] [-s {sast,sca}] [-i IMAGE] [--merge-sarif SARIF_FILE [SARIF_FILE ...]] [--merge-output MERGE_OUTPUT]

Agnostic DevSecOps Container scanning Pipeline Orchestrator

options:
  -h, --help            show this help message and exit
  -s, --scan-type {sast,sca}
                        Specify the security methodology to execute (e.g., sast, sca)
  -i, --image IMAGE     Target Docker image reference
  --merge-sarif SARIF_FILE [SARIF_FILE ...]
                        List of SARIF files to merge into one report
  --merge-output MERGE_OUTPUT
                        Output path for the merged SARIF file
```

## Step 5: Read the results

Each tool writes its own SARIF file. All three orchestrator scripts print
a summary block at the end of their run, for example:

```text
========== SCA PIPELINE SUMMARY ==========
[trivy]: PASSED
[osv-scanner]: WARNING (findings between 5.0 and 8.0)
============================================
```

If you don't have the GitHub Security tab available (a local run, or a
SARIF file downloaded from workflow artifacts), open it in a SARIF viewer
such as
[Microsoft's SARIF Web Component](https://microsoft.github.io/sarif-web-component/)
instead of reading the raw JSON.

To understand what PASSED, WARNING, FAILED, and ERROR mean for each tool,
see [gate-status-cvss.md](../reference/gate-status-cvss.md) and
[gate-status-rule-severity.md](../reference/gate-status-rule-severity.md).

## Next steps

* Setting up a repository whose ecosystem isn't Maven, Gradle, npm,
  Python, Go, or Rust: see
  [integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md).
* If a finding is a false positive, see
  [suppress-a-finding.md](../how-to/suppress-a-finding.md).
* If you hit a package-registry rate limit while resolving dependencies,
  or SBOM generation itself fails, see
  [troubleshooting.md](../how-to/troubleshooting.md).