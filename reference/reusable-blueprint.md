# Reference: The Reusable Blueprint Repository

The pipelines documented in this handbook exist in two forms:

| Form | Where | Who uses it |
|---|---|---|
| **Vendored** | `ci/` copied into a consuming repository, driven by that repository's own workflow files | `platform-backend`, `platform-ui`, and any project adopting the pipelines |
| **Blueprint** | [`DevSecOps-CI-pipelines`](https://github.com/moghit-eou/DevSecOps-CI-pipelines), the standalone artifact submitted for publication | Reference implementation, test bed, and the thing you copy `ci/` from |

Both run the same `ci/` scripts with the same pinned tool versions. The
blueprint adds a local Docker path so the pipelines can be run without
installing any scanner on the host, which is what makes the artifact
reproducible on a reviewer's machine. See
[ABOUT-JOSS-PUBLICATION.md](../ABOUT-JOSS-PUBLICATION.md).

## What the blueprint adds

```
DevSecOps-CI-pipelines/
├── Makefile          # make sast | sca | container-scan
├── toolbox.sh        # builds and runs the Docker toolbox images
├── ci/               # identical to the vendored ci/ folder
│   └── docker/
│       ├── Dockerfile      # three toolbox images from one shared installer stage
│       ├── entrypoints/
│       └── env/            # sast.env | sca.env | container-scan.env
├── .github/workflows/      # the three pipelines, scanning the test targets
└── test-*/                 # sample projects in several ecosystems
```

`Makefile` and `toolbox.sh` are blueprint-only. Nothing in `ci/` depends
on them, which is why a consuming repository can vendor `ci/` and ignore
the Docker path entirely.

## The `make` interface

```bash
make sast PROJECT=<dir>
make sca  PROJECT=<dir> ECOSYSTEM=<maven|npm|golang|generic|none>

docker build -t myapp:local ./my-service
make container-scan IMAGE=myapp:local SCAN_TYPE=sast PROJECT=./my-service
make container-scan IMAGE=myapp:local SCAN_TYPE=sca
```

`toolbox.sh` builds the matching stage from `ci/docker/Dockerfile` and
runs it with `PROJECT` bind mounted at `/workspace`, which is the image
`WORKDIR`. Runtime configuration comes from `ci/docker/env/*.env`, so
thresholds, output filenames, and suppression paths can be changed
without rebuilding the image.

**Container Scanning needs the image built beforehand.** `SCAN_TYPE=sca`
inspects an image that already exists in the local Docker daemon, so
`toolbox.sh` mounts the Docker socket
(`-v /var/run/docker.sock:/var/run/docker.sock`) into the toolbox
container. `SCAN_TYPE=sast` does not mount it, because it only reads the
Dockerfile.

## The `PROJECT` variable and why paths differ

The vendored pipelines in `platform-backend` and `platform-ui` always scan
the repository root, so every path in their workflows is relative and
everything resolves against the checkout directory.

The blueprint has to scan a chosen subdirectory, because its test targets
live in `test-maven/`, `test-npm/`, and so on. Its workflows therefore set
`working-directory: ${{ env.PROJECT }}` on the scan steps, which is the
native-runner equivalent of the `/workspace` mount in the Docker path.
That has one consequence worth knowing:

| Path | Form in the blueprint workflows | Reason |
|---|---|---|
| `ci/*.py`, `ci/setup-tools.sh` | absolute, `$GITHUB_WORKSPACE/ci/...` | lives at the repository root, not under `PROJECT` |
| Suppression files | absolute | same reason |
| Semgrep rulesets | absolute | cloned outside the checkout, see below |
| `OPENGREP_EXCLUDE` patterns | relative | matched against `PROJECT` |
| SARIF outputs | relative | written into `PROJECT` |

A repository that scans its own root, like both MIP components, needs none
of this. Relative paths are correct there and the workflows stay shorter.

## Ruleset clone location

`setup-tools.sh --install-tool semgrep-rules` clones the ruleset into
`./semgrep-rules`, relative to the working directory. The semgrep-rules
repository ships thousands of deliberately vulnerable test fixtures, so
where that clone lands matters:

| Path | Consequence |
|---|---|
| Inside the directory being scanned | OpenGrep walks the fixtures and reports them as findings |
| Outside the directory being scanned | Correct |

In the Docker path this is handled by construction: rules are baked into
`/app/semgrep-rules` at image build time, outside the `/workspace` mount.
In the blueprint's native workflows the clone step runs from
`$GITHUB_WORKSPACE/../tools`, outside the checkout, and
`SEMGREP_CONFIG_RULESETS` points at it by absolute path.

For a repository that scans its own root and clones into that root, the
fixtures fall inside the scanned tree. Both MIP components avoid the
problem through their `OPENGREP_EXCLUDE` lists, but cloning outside the
checkout is the more robust arrangement, because it cannot be undone by
someone editing an exclude list later.

## Gate thresholds

`GATE_FAIL_THRESHOLD` and `GATE_WARN_THRESHOLD` are read from the
environment by `parse_sarif.py`, defaulting to `8.0` and `5.0`. Set them
in a workflow `env:` block, or in `ci/docker/env/*.env` for local runs.
See [adjust-severity-gate.md](../how-to/adjust-severity-gate.md).

## Where dependency resolution lives

In the blueprint, `mvn dependency:resolve`, `npm ci`, and `go mod
download` run inside `setup-tools.sh`, in the same `case` branch that
generates the SBOM. The vendored MIP workflows run them as their own
explicit CI step instead.

Both are correct. The blueprint puts them in the script because it also
has a `make` path, and a local run never sees the workflow file: if
resolution lived only in YAML, `make sca` would build an SBOM from
unresolved dependencies and local results would diverge from CI. A
repository with no local path does not have that problem, and keeping the
resolve step visible in the workflow is arguably clearer for whoever
maintains it.
