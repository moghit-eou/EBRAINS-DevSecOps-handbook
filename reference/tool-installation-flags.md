# Reference: `setup-tools.sh` Flags and Installed Tools

```bash
bash ci/setup-tools.sh --install-tool <tool1,tool2,...|all> [--sbom-ecosystem <ecosystem>|none]
```

## Platform support

The script downloads **Linux x86_64** release assets, matching the
`ubuntu-latest` GitHub Actions runner image. The architecture is not
detected at runtime, so the asset URLs are fixed.

The dockerized path in the blueprint repository closes most of this gap,
because the container is Linux regardless of the host. See
[reusable-blueprint.md](reusable-blueprint.md) for the `make` interface and
the full platform table.

## Flags

| Flag | Values | Purpose |
|---|---|---|
| `--install-tool` | comma-separated list, or `all` | Which scanner binaries to install |
| `--sbom-ecosystem` | `maven`, `gradle`, `npm`, `raw-js`, `python`, `go`, `rust`, `none`, or a value you add yourself | Which SBOM generator to run after tool installation, see [integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md) |

## Installable tools

| Tool | Installed from | Verification | Used by |
|---|---|---|---|
| `trivy` | Official release tarball | SHA256-pinned | Container Scanning (SCA half), SCA |
| `osv-scanner` | GitHub release binary | SHA256-pinned | Container Scanning (SCA half), SCA |
| `opengrep` | GitHub release binary | SHA256-pinned | Container Scanning (SAST half), SAST |
| `hadolint` | GitHub release binary | SHA256-pinned | Container Scanning (SAST half) |
| `semgrep-rules` | Cloned from `semgrep/semgrep-rules` at a pinned commit | Git ref pin | Container Scanning (SAST half), SAST |

`should_install "<name>"` returns true if `--install-tool all` was passed,
or if `<name>` appears in the comma-separated `--install-tool` list.

## Version and checksum pinning

Every downloadable tool is pinned to an exact version and SHA256 checksum
at the top of the script:

```bash
# renovate: datasource=github-release-attachments depName=aquasecurity/trivy
TRIVY_VERSION="${TRIVY_VERSION:-v0.71.1}"
TRIVY_SHA256="${TRIVY_SHA256:-3cbae37cd440cd8676e5ce9207fe460b5641c7579a17e9d00f8894928c41a88d}"
```

The `# renovate:` comment lets Renovate bump the version and its matching
checksum together in one automated update. If you override a `*_VERSION`
via environment variable, you must override the matching `*_SHA256` too,
or `download_and_verify` will refuse to install it:

```bash
download_and_verify() {
  local url="$1" dest="$2" sha256="$3"
  curl -fsSL --retry 3 "${url}" -o "${dest}"
  echo "${sha256}  ${dest}" | sha256sum -c -
}
```

## Pinning `uses:` actions to a commit SHA

`setup-tools.sh`'s SHA256 checksum pinning above and the GitHub Actions
`uses:` pins referenced throughout this handbook are the same underlying
practice applied to two different kinds of downloaded artifact: a binary
release tarball in one case, a third-party Action's source in the other.
Both exist to give the pipeline an **immutable** reference to what it
runs, rather than a mutable one that could resolve to different content
tomorrow than it does today.

A tag or branch name (`@v6`, `@main`) is a mutable pointer: the maintainer
of that Action can move the tag to point at different, possibly malicious,
code without changing the version string in your workflow file at all.
The correct technical term for the fix is **commit SHA pinning**: pinning
a `uses:` reference to the full 40-character Git commit SHA of the exact
commit you intend to run, instead of a tag. A commit SHA is content-
addressed, it cannot be reassigned to point at different code, so it
gives the same immutability guarantee for an Action's source that a
SHA256 checksum gives for a downloaded binary.

```yaml
# Mutable: this tag can be moved to point at different code later
- uses: actions/setup-python@v6

# Commit SHA-pinned (illustrative SHA below): this reference cannot be
# reassigned; the trailing comment records the human-readable version for
# maintainers, the same role the "# renovate:" comment plays for
# setup-tools.sh's checksums
- uses: actions/setup-python@3bf3c327b16a3c5c1e0e2d1d9d5d9d5e5f5c5b5a # v6.2.0
```

This is why every code snapshot in this handbook (see
[platform-backend.md](../case-studies/platform-backend.md#code-snapshot-platform-backends-actual-scayml)
and
[platform-ui.md](../case-studies/platform-ui.md#code-snapshot-platform-uis-actual-scayml))
pins every `uses:` to a full commit SHA in the real, currently-running
workflow, even though the generic templates elsewhere in this handbook
show floating `@v6`/`@v7`-style tags for readability. Treat those floating
tags as placeholders to resolve to a commit SHA before using a template
in a real workflow, the same way the ecosystem placeholders (`<ecosystem
cache path>`, `<ecosystem install command>`) need resolving before the
template runs.

Renovate (or an equivalent dependency-update bot) can keep commit-SHA
pins current automatically, the same way the `# renovate:` markers keep
`setup-tools.sh`'s version/checksum pairs current, so pinning to a SHA
does not mean manually chasing upstream releases by hand.

## Script safety flags

```bash
set -e          # stop the pipeline if any command fails
set -o pipefail # prevents silent pipeline successes if the curl download drops
set -u          # treat unset variables as an error
trap 'echo "[setup-tools] ERROR: command failed (exit $?) at line $LINENO: $BASH_COMMAND" >&2' ERR
```

On any error, the script stops and prints the failing line number and
command, rather than continuing silently or failing with an unrelated
downstream error.

## SBOM generation (`--sbom-ecosystem`)

| Value | Command run |
|---|---|
| `maven` | `mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q` |
| `gradle` | `./gradlew cyclonedxBom -q` |
| `npm` | `npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json` |
| `raw-js` | `npx --yes @cyclonedx/cdxgen -t js -o target/bom.json .` |
| `python` | `cyclonedx-py requirements requirements.txt -o target/bom.json` |
| `go` | `cyclonedx-gomod mod -json -output target/bom.json` |
| `rust` | `cargo cyclonedx --format json --override-filename bom` (then normalized to `target/bom.json`) |
| `none` | No SBOM generated |

This table is not a hard ceiling on what the pipeline supports, it's the
set of `case` branches already written into `setup-tools.sh`. Adding a
value not listed here (PHP, Ruby, .NET, or anything else with a CycloneDX
generator) is a one-branch addition, not a change to this flag's design,
see
[Adding a new ecosystem](../how-to/integrate-a-new-ecosystem.md#adding-a-new-ecosystem-not-in-the-matrix).

`container-scan.yml` scans the built image directly and needs no SBOM, so
it always uses `--sbom-ecosystem none` (or omits the flag). Full
command-by-command breakdown, including dependency install/resolve steps
and cache configuration per ecosystem, is in
[integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md#ecosystem-matrix).
