# About the JOSS Publication

## Software artifact

The dockerized, reusable version of the pipelines documented in this
handbook is packaged as a standalone repository and submitted as a short
software paper (JOSS or a similar venue). Everything runs inside a
container built from `ci/docker/Dockerfile`, driven through a single
Makefile, no local tool installation required. The submission date is not
yet fixed.

| Property | Value |
|---|---|
| Repository | `reusable-ci-pipelines` |
| Suggested citable name | `reusable-devsecops-pipelines` (working suggestion) |
| Target venue | JOSS (Journal of Open Source Software) |
| Core design goal | Reusability: any project drops in the pipelines via `make sast` / `make sca` / `make container-scan`, no local setup |

## How it relates to this handbook

```mermaid
flowchart LR
    A["This handbook"] -->|documents design and rationale| B["reusable-ci-pipelines repo"]
    B -->|submitted as| C["JOSS paper"]
```

The `reusable-ci-pipelines` repo is the runnable, dockerized artifact
itself, the thing being cited, and it is self-contained: its own README and
`docs/` cover running, configuring and adopting the pipelines without
reference to this handbook. That matters, because a reviewer arriving from
JOSS or OWASP has no EBRAINS context and should not need any.

This handbook is the EBRAINS-facing companion. It carries the longer
narrative of one production rollout, the case studies behind each decision,
and the MIP-specific detail that does not belong in a general-purpose
repository.

## What the artifact contains

| Component | Purpose |
|---|---|
| `Makefile`, `toolbox.sh` | Local entrypoint, runs each pipeline in its own Docker toolbox image |
| `ci/` | The scanners' orchestrators, install script, and suppression files, identical to the vendored copy in a consuming repository |
| `ci/docker/` | Three toolbox images built from one shared installer stage, so scanners and the Trivy database are downloaded once |
| `docs/` | Operational guides for an adopter: CI, local runs, configuration, suppressions, adding an ecosystem, tool choices |
| `.github/workflows/` | The three pipelines as plain workflow steps, also serving as the repository's own test suite |
| `test-*/` | Sample projects in several ecosystems, exercised on every pull request |

See [reference/reusable-blueprint.md](reference/reusable-blueprint.md) for
the full interface.
