<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/0/08/GSoC_logo.svg/500px-GSoC_logo.svg.png" height="100">
  <img src="https://mip.ebrains.eu/img/mip-logo-short-compact.png" height="80">
  <img src="https://www.flagera.eu/wp-content/uploads/2021/06/ebrains_logo_black.png__400x148_q85_crop_subsampling-2-300x111.png" height="80">
</p>

# EBRAINS DevSecOps Handbook

This handbook documents a working reference implementation of an OWASP
DevSecOps-aligned CI/CD security pipeline, built as a Google Summer of Code
2026 project for the EBRAINS community, using the **Medical Informatics
Platform (MIP)** as the proof of concept.

The pipeline design itself is **vendor-neutral and language-agnostic**: it
runs on GitHub Actions today, but nothing in `setup-tools.sh`,
`sca_scan.py`, `sast_scan.py`, `container_scan.py`, or `parse_sarif.py`
depends on a specific CI provider or a specific build ecosystem. See
[how-to/integrate-a-new-ecosystem.md](how-to/integrate-a-new-ecosystem.md).

## Platform and maintenance notes

> **Status: active, in progress.** This handbook documents a Google Summer
> of Code 2026 project that is still under active development. Content
> reflects the pipelines as they exist today, not a finished, frozen
> product. Commands, thresholds, and file layouts may still change before
> the project ends.

- Everything documented here has been built and tested on **Linux x86_64 /
  Ubuntu**, matching the `ubuntu-latest` GitHub Actions runner image. The
  install script (`setup-tools.sh`) is not portable as-is to macOS or
  native Windows for a native (non-dockerized) setup. WSL2 works for
  Windows because it provides a real Linux userspace. A dockerized
  version of the toolchain now closes this gap, see
  [ABOUT-JOSS-PUBLICATION.md](ABOUT-JOSS-PUBLICATION.md); see
  [tool-installation-flags.md](reference/tool-installation-flags.md) for
  the native install-script path.

## Table of Contents

- [README.md](README.md) (this file)
- [ABOUT-OWASP-CONTRIBUTION.md](ABOUT-OWASP-CONTRIBUTION.md)
- [ABOUT-JOSS-PUBLICATION.md](ABOUT-JOSS-PUBLICATION.md)
- **tutorials/**
  - [01-setup-guide.md](tutorials/01-setup-guide.md)
- **how-to/**
  - [integrate-a-new-ecosystem.md](how-to/integrate-a-new-ecosystem.md) — the universal, per-ecosystem integration matrix (Maven, Gradle, npm, raw JavaScript, Python, Go, Rust, and how to add any ecosystem not yet listed)
  - [suppress-a-finding.md](how-to/suppress-a-finding.md)
  - [adjust-severity-gate.md](how-to/adjust-severity-gate.md)
  - [add-new-scanner.md](how-to/add-new-scanner.md)
  - [troubleshooting.md](how-to/troubleshooting.md)
- **reference/**
  - [pipeline-container-scanning.md](reference/pipeline-container-scanning.md)
  - [pipeline-sca.md](reference/pipeline-sca.md)
  - [pipeline-sast.md](reference/pipeline-sast.md)
  - [python-orchestrators.md](reference/python-orchestrators.md)
  - [tool-installation-flags.md](reference/tool-installation-flags.md)
  - [exit-codes.md](reference/exit-codes.md)
  - [gate-status-cvss.md](reference/gate-status-cvss.md)
  - [gate-status-rule-severity.md](reference/gate-status-rule-severity.md)
- **explanation/**
  - [why-three-independent-pipelines.md](explanation/why-three-independent-pipelines.md)
  - [why-cvss-based-gating.md](explanation/why-cvss-based-gating.md)
  - [why-two-gate-models.md](explanation/why-two-gate-models.md)
  - [why-these-tools.md](explanation/why-these-tools.md)
  - [why-two-sca-tools.md](explanation/why-two-sca-tools.md)
  - [why-sboms.md](explanation/why-sboms.md)
  - [why-opengrep-not-semgrep.md](explanation/why-opengrep-not-semgrep.md)
- **case-studies/**
  - [platform-backend.md](case-studies/platform-backend.md)
  - [platform-ui.md](case-studies/platform-ui.md)

## Why this exists

Security practices across EBRAINS and the wider Neuroinformatics community
currently vary from project to project, and in many cases only partially
meet the requirements of the Cyber Resilience Act (CRA) and NIS2. This
project starts from a maturity snapshot of MIP's pipelines, assessed against
the [OWASP DevSecOps Maturity Model (DSOMM)](https://dsomm.owasp.org/), and
turns that snapshot into an actionable, implemented set of controls, using
the [OWASP DevSecOps Guideline](https://github.com/OWASP/DevSecOpsGuideline)
as the implementation reference.

The outcome is a working reference pipeline that produces security
artifacts automatically (scan reports, SBOMs, gate decisions), proven
against two MIP components with different stacks (`platform-backend`,
Maven/Java; `platform-ui`, npm/Angular) plus their container images, and
packaged here as a reusable, ecosystem-agnostic secure-pipeline blueprint.

## What is actually implemented

Three independent GitHub Actions pipelines, each following the OWASP
DevSecOps model, each with its own workflow file, orchestrator script, and
gate:

| Pipeline | Scans | Tools |
|---|---|---|
| **Container Scanning** | The built Docker image + the Dockerfile itself | Trivy, OSV-Scanner (image CVEs); Hadolint, OpenGrep (Dockerfile SAST) |
| **SCA** (Software Composition Analysis) | Application dependencies, via a generated SBOM (CycloneDX) | Trivy, OSV-Scanner |
| **SAST** (Static Application Security Testing) | Application source code | OpenGrep |

See [reference/pipeline-container-scanning.md](reference/pipeline-container-scanning.md),
[reference/pipeline-sca.md](reference/pipeline-sca.md), and
[reference/pipeline-sast.md](reference/pipeline-sast.md) for the exact
commands, flags, and files involved in each. The full architecture diagram
(all three pipelines, their triggers, and where they converge on the
GitHub Security tab) lives in
[explanation/why-three-independent-pipelines.md](explanation/why-three-independent-pipelines.md#independent-triggers-and-parallel-execution)
rather than being repeated here.

All three workflows trigger independently and run in parallel; each
uploads its own SARIF category to the Security tab, see
[reference/exit-codes.md](reference/exit-codes.md) for the full category
list.

## How this handbook is organized

This handbook follows the [Diátaxis](https://diataxis.fr/) framework, the
same structure used by the official
[EBRAINS Handbook](https://handbook.ebrains.eu), so that migrating this
content there later, if that path is chosen, needs minimal rework:

- **tutorials/** - learning-oriented, step by step, works start to finish.
- **how-to/** - task-oriented, one job per file, assumes a working pipeline.
- **reference/** - information-oriented, tables and facts, no narrative.
- **explanation/** - understanding-oriented, the reasoning and tradeoffs
  behind the design.
- **case-studies/** - the one place real names, repo links, and
  project-specific detail are expected.

## Where to start

- New to this and want to set up the tools locally: start with
  [tutorials/01-setup-guide.md](tutorials/01-setup-guide.md).
- Onboarding a repository that isn't Maven or npm (Gradle, raw JavaScript,
  Python, Go, Rust, or an ecosystem not documented yet at all): go straight
  to
  [how-to/integrate-a-new-ecosystem.md](how-to/integrate-a-new-ecosystem.md).
  Maven and npm are the two ecosystems the reference implementations
  actually run; they are not a limitation of the pipeline itself.
- Already running the pipelines and need to do one specific thing (suppress
  a finding, add a scanner, adjust a gate): go to [how-to/](how-to/).
- Want exact commands, flags, exit codes, or gate thresholds: go to
  [reference/](reference/).
- Want to understand *why* the pipelines are built this way: go to
  [explanation/](explanation/).
- Want the concrete, named story of how this was built and validated
  against `platform-backend` (Maven) and `platform-ui` (npm): read
  [case-studies/](case-studies/).

## Relationship to OWASP

This project has an open pull request against the upstream OWASP
DevSecOps Guideline. See
[ABOUT-OWASP-CONTRIBUTION.md](ABOUT-OWASP-CONTRIBUTION.md) for the full
history and the still-open question of whether this handbook stays tied to
the OWASP guide or becomes a standalone EBRAINS publication.
