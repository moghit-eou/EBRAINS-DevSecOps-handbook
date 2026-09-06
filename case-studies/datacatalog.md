# Case Study: `datacatalog`

**This was a test of the workbooks, not of the scanners.**

`platform-backend` and `platform-ui` were used to *build* the pipelines.
[`datacatalog`](https://github.com/Medical-Informatics-Platform/datacatalog)
was picked afterwards to answer one question: can somebody take
[`reusable-ci-pipelines`](https://github.com/moghit-eou/reusable-ci-pipelines),
follow this handbook, and wire the three pipelines into a repository the
handbook has never seen?

Result: [pull request #1](https://github.com/moghit-eou/datacatalog/pull/1),
10 files added, nothing removed, no existing workflow touched.

## The repository

Three services in one repo, three package managers, three Dockerfiles.

| Unit | Path | Stack | SCA ecosystem |
|---|---|---|---|
| backend | `backend/` | Java 21, Maven, Spring Boot | `maven` |
| frontend | `frontend/` | Angular, npm, Nginx | `npm` |
| data quality tool | `data_quality_tool/` | Python, Poetry, Flask | `generic` |

Every earlier example scans a repository root. This one cannot.

## How it was adopted

**1. Copied `ci/` in, unchanged.** All 7 files: `setup-tools.sh`,
`sast_scan.py`, `sca_scan.py`, `container_scan.py`, `parse_sarif.py`, and
the two suppression files.

> **`ci/docker/` was left out.** That folder exists only for the local
> `make` path. `datacatalog` runs in CI only, so it would have been dead
> weight. Nothing in `ci/*.py` or `setup-tools.sh` depends on it, which is
> the separation
> [reusable-blueprint.md](../reference/reusable-blueprint.md) describes.

**2. Copied the three workflows, changed only `env:` values.** No step was
added, removed, or reordered.

| Changed | Left alone |
|---|---|
| `PROJECT`, `ECOSYSTEM`, `IMAGE_NAME` | Every `steps:` entry |
| `SARIF_CATEGORY`, `ARTIFACT_NAME` | Every `uses:` pin |
| `SEMGREP_CONFIG_RULESETS`, `OPENGREP_EXCLUDE` | Gate logic, SARIF filenames |

## A note on `matrix`

Three services means each pipeline runs three times. There are two ways to
write that:

- **Three `jobs:` blocks**, one per service.
- **One `jobs:` block with a matrix**, which produces the same three runs.

The matrix is **optional**. It is not a feature of the pipelines, it is a
GitHub Actions convenience that replaces three near-identical job blocks
with one. It was used for SAST and container scanning, where the steps are
identical and only values differ. It was skipped for SCA, where the steps
themselves differ per ecosystem and three explicit blocks read better.

> `fail-fast: false` is required when using a matrix here. The default
> cancels the other legs as soon as one fails, and these gates are designed
> to fail.

## SAST

Matrix over three legs. Only two values differ per service, and both are
plain strings that fit in the matrix.

```yaml
jobs:
  sast:
    name: SAST (${{ matrix.slug }}, ${{ matrix.language }})
    strategy:
      fail-fast: false
      matrix:
        include:
          - slug: backend
            language: java
            project: ./backend
            rulesets: >-
              .../semgrep-rules/java ... p/default auto
            exclude_paths: "*.sarif target/ src/test/ Dockerfile*"
          - slug: frontend
            language: typescript
            project: ./frontend
            rulesets: >-
              .../semgrep-rules/typescript .../javascript .../html ... p/default auto
            exclude_paths: "*.sarif node_modules/ dist/ *.spec.ts Dockerfile*"
          - slug: dqt
            language: python
            project: ./data_quality_tool
            rulesets: >-
              .../semgrep-rules/python ... p/default auto
            exclude_paths: "*.sarif tests/ target/ poetry.lock Dockerfile*"

    env:
      PROJECT: ${{ matrix.project }}
      SEMGREP_CONFIG_RULESETS: ${{ matrix.rulesets }}
      OPENGREP_EXCLUDE: ${{ matrix.exclude_paths }}
      OPENGREP_SARIF_OUTPUT: sast-opengrep.sarif
      SARIF_CATEGORY: sast-${{ matrix.slug }}
      ARTIFACT_NAME: sast-report-${{ matrix.slug }}

    steps:
      # unchanged from the blueprint
      - checkout
      - setup-python
      - setup-tools.sh --install-tool opengrep,semgrep-rules   # outside the checkout
      - sast_scan.py                                           # working-directory: PROJECT
      - upload-sarif + upload-artifact
```

All three keep the shared packs (`generic`, `problem-based-packs`,
`package_managers`, `p/default auto`).

> The `tests/` exclusion on the data quality tool matters: that folder
> holds deliberately malformed Excel and JSON fixtures for the validator.
> Same category of noise as the semgrep-rules fixtures, but inside the
> project being scanned.

## Container scanning

Matrix over three legs. Only the build context and the image tag differ.
The Dockerfile ruleset is the same for all three.

```yaml
jobs:
  container-scan:
    name: Container scan (${{ matrix.slug }})
    strategy:
      fail-fast: false
      matrix:
        include:
          - slug: backend
            project: ./backend
          - slug: frontend
            project: ./frontend
          - slug: data_quality_tool
            project: ./data_quality_tool

    env:
      PROJECT: ${{ matrix.project }}
      IMAGE_NAME: datacatalog-${{ matrix.slug }}:scan
      DOCKERFILE_PATH: Dockerfile
      SEMGREP_CONFIG_RULESETS: .../semgrep-rules/dockerfile auto
      SARIF_CATEGORY: container-${{ matrix.slug }}
      ARTIFACT_NAME: container-report-${{ matrix.slug }}

    steps:
      # unchanged from the blueprint
      - checkout
      - setup-python
      - docker build -t "$IMAGE_NAME" .                        # working-directory: PROJECT
      - setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
      - container_scan.py --scan-type sast                     # Dockerfile
      - container_scan.py --scan-type sca --image "$IMAGE_NAME" # image layers
      - container_scan.py --merge-sarif ...
      - upload-sarif + upload-artifact
```

## SCA

Three separate jobs, no matrix. The steps genuinely differ: each ecosystem
caches a different path, and npm needs a Node setup step the other two do
not.

```yaml
jobs:
  # ===== backend: Maven ====================================================
  sca-backend:
    env:
      PROJECT: ./backend
      ECOSYSTEM: maven
      SARIF_CATEGORY: sca-backend
      ARTIFACT_NAME: sca-report-backend
    steps:
      - checkout
      - setup-python
      - cache: ~/.m2/repository, key on backend/**/pom.xml
      - setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
      - sca_scan.py
      - upload-sarif + upload-artifact

  # ===== frontend: npm =====================================================
  sca-frontend:
    env:
      PROJECT: ./frontend
      ECOSYSTEM: npm
      SARIF_CATEGORY: sca-frontend
      ARTIFACT_NAME: sca-report-frontend
    steps:
      - checkout
      - setup-python
      - setup-node 20                      # <-- only this job needs it
      - cache: ~/.npm, key on frontend/package-lock.json
      - setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
      - sca_scan.py
      - upload-sarif + upload-artifact

  # ===== data quality tool: Poetry, via generic ============================
  sca-dqt:
    env:
      PROJECT: ./data_quality_tool
      ECOSYSTEM: generic
      SARIF_CATEGORY: sca-dqt
      ARTIFACT_NAME: sca-report-dqt
    steps:
      - checkout
      - setup-python
      # no cache, no Poetry install: generic resolves nothing
      - setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem generic
      - sca_scan.py
      - upload-sarif + upload-artifact
```

Cache keys are scoped to `PROJECT`:

```yaml
key: ${{ runner.os }}-m2-v1-${{ hashFiles(format('{0}/**/pom.xml', env.PROJECT)) }}
```

> **Monorepos need the `format()` wrapper.** A bare `hashFiles('**/pom.xml')`
> hashes every manifest in the repository, so a frontend-only commit
> invalidates the Maven cache. Single-service repos never hit this.

## Why Python used `generic`

`setup-tools.sh` has no Poetry branch. Both routes were tried locally:

| Route | Result |
|---|---|
| `cyclonedx-py poetry` | Valid SBOM, correct PURLs |
| `trivy fs --format cyclonedx` (`generic`) | Valid SBOM, read `poetry.lock` and `requirements.txt` |

`generic` was enough, so no branch was added. Two things came out of the
comparison that are worth recording.

**The two generators describe different projects.** `trivy fs` prints
`Suppressing dependencies for development and testing` and excludes dev
groups. `cyclonedx-py poetry` includes them. The image runs
`poetry install --without dev`, so the `generic` view happens to match
what ships. Neither is wrong, but the choice decides whether a CVE in a
test-only package can block a release.

**Trivy and OSV-Scanner disagreed.**

| Tool | Input | Result |
|---|---|---|
| Trivy | `sbom.json` | 0 vulnerabilities |
| OSV-Scanner | `poetry.lock` | 1 High: `click 8.1.7`, PYSEC-2026-2132, CVSS 7.2, fixed in 8.3.3 |

The PURL was correct (`pkg:pypi/click@8.1.7`), so this is not malformed
input. **PYSEC** advisories come from the PyPA database and do not always
have a GHSA counterpart, and Trivy's Python matching leans on GitHub
Advisories. `click` is not a dev dependency, Flask pulls it in, so it
ships.

This is the Python counterpart to the npm PURL-casing finding in
[platform-ui.md](platform-ui.md), arguing the same point from the opposite
direction: there the second tool removed a false positive, here it was the
only tool that reported a real one. See
[why-two-sca-tools.md](../explanation/why-two-sca-tools.md).

## Naming: what has to be unique

| Value | Scope | Collision effect |
|---|---|---|
| `SARIF_CATEGORY` | Repository-wide | A later upload silently replaces an earlier one |
| `ARTIFACT_NAME` | Repository-wide | Upload fails or overwrites |
| SARIF filenames | Inside `PROJECT` | None, all three can be `sca-merged.sarif` |

The category collision is the dangerous one: nothing fails, you just get a
green run and a Security tab showing one service's findings under another
service's name.

Convention used: `<pipeline>-<slug>`.

> **Known inconsistency in the PR.** SAST and SCA use the slug `dqt`,
> container scanning uses `data_quality_tool`. Both work, categories are
> still unique, but they should match before this is the reference example.

## Gate thresholds

All nine jobs start at `GATE_FAIL_THRESHOLD: "9.5"` / `GATE_WARN_THRESHOLD: "5.0"`,
looser than the blueprint default of 8.0. `datacatalog` had never been
scanned, and a pipeline that blocks every pull request in week one gets
switched off in week two. Tighten once the backlog is triaged. Findings on
a first scan are the expected output of this exercise, not a failure of
it. See [adjust-severity-gate.md](../how-to/adjust-severity-gate.md).

## Left out

| Left out | Reason |
|---|---|
| `ci/docker/` and `make` | CI-only adoption |
| A `poetry` branch | `generic` covered it |
| pre-commit | Separate piece of work |
| IaC scanning | Helm chart and Compose file make this an obvious future test bed, but that pipeline is still roadmap work |
| Path filters | Every leg should prove itself on a full run first |
| Fixing `data_quality_tool.yml` | It sets up Python 3.9 for a `^3.12` project. Real, unrelated, needs its own issue |

## What the exercise proved

| Claim | Verdict |
|---|---|
| Vendoring `ci/` is a copy, no edits | Held, 7 files unchanged |
| Only `env:` values change per repository | Held |
| Only the cache step and install command differ per ecosystem | Held |
| `ci/docker/` is optional | Held |
| `generic` covers an ecosystem with no native branch | Held, on Poetry |
| Two SCA tools catch more than one | Held, in a new ecosystem |
| The docs cover a multi-service repository | **Did not hold.** Nothing explained per-unit categories, scoped cache keys, or when a matrix helps. This case study is that gap |

## Related

- [reusable-blueprint.md](../reference/reusable-blueprint.md)
- [integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md)
- [why-two-sca-tools.md](../explanation/why-two-sca-tools.md)
- [platform-backend.md](platform-backend.md), [platform-ui.md](platform-ui.md)
