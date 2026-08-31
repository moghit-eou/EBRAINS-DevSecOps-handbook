# Case Study: `platform-ui`

[`platform-ui`](https://github.com/Medical-Informatics-Platform/platform-ui)
(Medical Informatics Platform org on GitHub) is the second reference
implementation this handbook is built against, an Angular, npm service,
with the same small amount of Python used for the CI orchestrators as
`platform-backend` (`ci/*.py`, see
[reference/python-orchestrators.md](../reference/python-orchestrators.md)).
Where the [`platform-backend` case study](platform-backend.md) covers the
Maven-side false-positive investigation, this one covers the npm-side
finding that shaped the case for running multiple SCA tools rather than
one: a malware false positive caused by a PURL casing collision.

## The malware false positive

While testing SCA tooling locally against `platform-ui`, Dependency-Track
flagged a malware finding (`MAL-2022-4051`) against `platform-ui`'s
legitimate dependency `jQuery-QueryBuilder`.

**The cause was a PURL collision.** `cyclonedx-npm` lowercases all package
names when generating the SBOM, producing
`pkg:npm/jquery-querybuilder@3.0.0`. A real, unrelated malicious package
is registered on npm under that same lowercase name,
`jquery-querybuilder`. Because npm's own registry is case-insensitive,
both the legitimate `jQuery-QueryBuilder` and the malicious
`jquery-querybuilder` collapse into the same PURL once lowercased, and
Dependency-Track (which relies on the PURL as its lookup key against OSV)
couldn't tell them apart.

Notably, this same malware flag was **not** caught by `npm audit`, only by
Dependency-Track via OSV, concrete evidence for why relying on a single
tool and a single database leaves gaps, see
[explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).

## Why this matters beyond the one false positive

The PURL-casing mechanism is a structural property of how CycloneDX PURLs
are generated for npm packages, not a one-off bug in this specific
dependency. Any npm package whose real name differs from another
package's name only by case is exposed to the same collision. This is
worth flagging during triage of any future npm-ecosystem malware or
typosquat finding, verify the actual package identity (registry page,
maintainer, publish history) before assuming a PURL match is correct,
rather than trusting the PURL string alone.

## What this shaped in the final pipeline

- SCA scans the generated SBOM for `platform-ui` the same way it does for
  `platform-backend`, via `cyclonedx-npm` against the installed
  `node_modules/` tree (which requires `npm ci`, not `npm install`, see
  [explanation/why-sboms.md](../explanation/why-sboms.md)).
- Both Trivy and OSV-Scanner run against the SBOM, rather than a single
  tool, precisely because a single tool and a single database (as with
  `npm audit` here) can miss findings the other catches, see
  [explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).
- The `~/.npm` cache is restored between CI runs, keyed on
  `hashFiles('**/package-lock.json')`, never on `package.json`, see
  [reference/pipeline-sca.md](../reference/pipeline-sca.md#caching).

See [reference/pipeline-sca.md](../reference/pipeline-sca.md) for the
resulting workflow, and
[case-studies/platform-backend.md](platform-backend.md) for the companion
Maven-side investigation that shaped the SBOM-first design more broadly.

## Code snapshot: `platform-ui`'s actual `sca.yml`

This is a trimmed version of the real, currently-running workflow (Action
`uses:` pins shortened for readability; the live file pins full commit
SHAs, see [tool-installation-flags.md](../reference/tool-installation-flags.md)
for why). It's the concrete instance of the
[generic SCA template](../reference/pipeline-sca.md#generic-github-actions-template),
with the npm block filled in:

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
      TRIVY_SARIF_OUTPUT: trivy-platform-ui.sarif
      OSV_SARIF_OUTPUT: osv-scanner-platform-ui.sarif
      SCA_MERGED_SARIF_OUTPUT: SCA-platform-ui-merged.sarif

    steps:
      - name: Check out repository
        uses: actions/checkout@v7

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      # --- the npm-specific block; this is the only part that differs from platform-backend's Maven version ---
      - name: Cache npm packages
        uses: actions/cache@v6
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-v1-${{ hashFiles('**/package-lock.json') }}
          restore-keys: ${{ runner.os }}-npm-v1-
      # ----------------------------------------------------------------------------------------------------------

      - name: Setup tools
        run: bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm

      - name: Run SCA tools
        run: python ci/sca_scan.py

      - name: Upload Trivy SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: ${{ env.TRIVY_SARIF_OUTPUT }}
          category: trivy-app

      - name: Upload OSV Scanner SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v2
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