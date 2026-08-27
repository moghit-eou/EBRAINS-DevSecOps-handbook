# About the JOSS Publication

## Software artifact

The dockerized, reusable version of the pipelines documented in this
handbook is being packaged as a standalone repository and submitted as a
short software paper (JOSS or a similar venue). Everything runs inside a
container built from `ci/docker/Dockerfile`, driven through a single
Makefile, no local tool installation required.

| Property | Value |
|---|---|
| Repository | `DevSecOps-CI-pipelines` |
| Suggested citable name | `reusable-devsecops-pipelines` (working suggestion) |
| Target venue | JOSS (Journal of Open Source Software) |
| Core design goal | Reusability: any project drops in the pipelines via `make sast` / `make sca` / `make container-scan`, no local setup |

## How it relates to this handbook

```mermaid
flowchart LR
    A["This handbook"] -->|documents design and rationale| B["DevSecOps-CI-pipelines repo"]
    B -->|submitted as| C["JOSS paper"]
```

The handbook explains why the pipelines are built this way (tool choices,
gate model, ecosystem matrix). The `DevSecOps-CI-pipelines` repo is the
runnable, dockerized artifact itself, the thing being cited.

## Roadmap

| Item | Status |
|---|---|
| GitHub composite actions wrapping the Makefile targets | Planned, not yet implemented |

The composite actions would let a consuming repository add a pipeline with
a single `uses:` step instead of copying workflow YAML. This is future
work, not part of the current release.
