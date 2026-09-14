# Reference: The Reusable Blueprint Repository

The pipelines documented in this handbook exist in two forms:

| Form | Where | Who uses it |
|---|---|---|
| **Vendored** | `ci/` copied into a consuming repository, driven by that repository's own workflow files | `platform-backend`, `platform-ui`, `datacatalog`, and any project adopting the pipelines |
| **Blueprint** | [`reusable-ci-pipelines`](https://github.com/moghit-eou/reusable-ci-pipelines), the standalone artifact submitted for publication | Reference implementation, test bed, and the thing you copy `ci/` from |

Both run the same `ci/` scripts with the same pinned tool versions. The
blueprint adds a local Docker path so the pipelines can run without
installing any scanner on the host.

## What the blueprint adds

```
reusable-ci-pipelines/
├── Makefile          # make sast | sca | container-scan
├── toolbox.sh        # builds and runs the Docker toolbox images
├── ci/               # identical to the vendored ci/ folder
│   └── docker/
│       ├── Dockerfile      # three toolbox images from one shared installer stage
│       ├── entrypoints/
│       └── env/            # sast.env | sca.env | container-scan.env
├── docs/                   # operational guides: CI, local runs, configuration,
│                           # suppressions, adding an ecosystem, tool choices
├── .github/workflows/      # the three pipelines, scanning the test targets
└── test-*/                 # sample projects in several ecosystems
```

The blueprint's own `README.md` and `docs/` are written for an adopter who
has never heard of EBRAINS or MIP, so that repository stands on its own.
This handbook is the EBRAINS-facing companion, not a prerequisite for using
it.

`Makefile` and `toolbox.sh` are blueprint-only. Nothing in `ci/` depends on
them, which is why a consuming repository can vendor `ci/` and ignore the
Docker path entirely.

## The `make` interface

```bash
make sast PROJECT=<dir>
make sca  PROJECT=<dir> ECOSYSTEM=<maven|npm|golang|generic|none>

docker build -t myapp:local ./my-service
make container-scan IMAGE=myapp:local SCAN_TYPE=sast PROJECT=./my-service
make container-scan IMAGE=myapp:local SCAN_TYPE=sca
```

`toolbox.sh` builds the matching stage from `ci/docker/Dockerfile` and runs
it with `PROJECT` bind mounted at `/workspace`, which is the image
`WORKDIR`. Runtime configuration comes from `ci/docker/env/*.env`, so
thresholds, output filenames, and suppression paths can be changed without
rebuilding the image.

## The `PROJECT` variable

`PROJECT` is the folder to scan. Everything happens inside it: the scanners
read that folder, and the SARIF reports are written there too. Point it at
your repository root with `.`, or at a subdirectory such as
`./services/api`.

Locally it becomes the `/workspace` bind mount. In the blueprint's
workflows it becomes `working-directory: ${{ env.PROJECT }}` on the scan
steps, which does the same job on a native runner.

Because the scan runs inside `PROJECT`, anything living at the repository
root has to be referenced by absolute path:

| Path | Form | Reason |
|---|---|---|
| `ci/*.py`, `ci/setup-tools.sh` | absolute | lives at the repository root, not under `PROJECT` |
| Suppression files | absolute | same reason |
| Semgrep rulesets | absolute | cloned outside the checkout |
| `OPENGREP_EXCLUDE` patterns | relative | matched against `PROJECT` |
| SARIF outputs | relative | written into `PROJECT` |

A repository that scans its own root, like both MIP components, needs none
of this. Relative paths are correct there and the workflows stay shorter.

## Container Scanning needs Docker access

`SCAN_TYPE=sca` inspects an image that already exists in your local Docker
daemon, so two things are required:

- **Build the image first.** The toolbox inspects it, it does not build it.
- **The Docker socket must be reachable.** `toolbox.sh` resolves it from
  the active Docker context. On Docker Desktop the default socket has to be enabled under
  Settings, Advanced, "Allow the default Docker socket to be used". If no
  socket is found, the script stops with a message rather than failing
  later inside the container.

`SCAN_TYPE=sast` does not use the socket, because it only reads the
Dockerfile.

## Platform support

**Everything here was developed and tested on Linux, x86_64.** That is the
reference platform and the one CI runs on.

| Platform | Status |
|---|---|
| Linux, x86_64 | Tested |
| Windows with WSL2 | Tested, follow the Linux instructions inside the distribution |
| macOS | Not tested. Docker Desktop should work, on Apple Silicon under emulation |

The scanners in `ci/setup-tools.sh` are x86-64 Linux binaries, so
`toolbox.sh` builds and runs every image with `--platform linux/amd64`. On
a machine with a different CPU architecture, Docker emulates it. This also
keeps local results the same as CI, which runs on x86-64 too.

This covers the toolbox images only, not the image you scan. If you build
your own image on a different architecture, it stays that architecture.