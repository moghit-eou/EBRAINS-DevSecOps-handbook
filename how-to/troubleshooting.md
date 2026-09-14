# How to Troubleshoot the Pipelines

This page covers the failure modes actually seen in practice, running
these pipelines against `platform-backend` and `platform-ui`: package-
registry rate limiting during dependency resolution, and SBOM generation
errors per ecosystem.

## Package-registry rate limiting

Applies to any ecosystem where a scanner resolves artifact metadata live
against a public package registry (Maven Central, npmjs.org, PyPI, the Go
module proxy) instead of reading only what's already local. Under
repeated CI runs from the same IP range (a shared GitHub Actions runner
pool), registries can rate-limit that traffic.

### How this shows up

**Maven Central**, observed directly while testing Trivy against
artifacts not yet present in the local `~/.m2` cache: repeated requests
from a CI-runner IP were interpreted as abuse, and Maven Central began
returning HTTP 429 (Too Many Requests). The scan step failed outright,
not just returned incomplete results.

The same class of failure applies to any ecosystem where the scanning
tool (not just the build tool) makes live registry calls: npm's registry,
PyPI, and the Go module proxy all apply similar rate limits under
sustained automated traffic.

### Why scanning the SBOM avoids most of this

This pipeline's SCA design, scan the generated SBOM rather than resolving
packages live during the scan, sidesteps this class of failure for the
scan step itself: `trivy sbom` and `osv-scanner scan source --lockfile`
both read a local file, they don't need network access to the registry
to evaluate what's already in the SBOM. See
[why-sboms.md](../explanation/why-sboms.md) for the full reasoning behind
scanning the SBOM instead of the raw dependency cache.

Rate limiting can still occur **before** the SBOM exists, during the
dependency resolve/install step itself (`mvn dependency:resolve`,
`npm ci`, `pip install`, `go mod download`), since that step does talk to
the registry directly. This is why dependency resolution runs as its own
explicit CI step ahead of SBOM generation,see
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md), rather than
letting a scanner resolve artifacts on demand mid-scan.

### Mitigations, by ecosystem

| Ecosystem | Mitigation |
|---|---|
| Maven | Ensure the dependency cache (`~/.m2/repository`) is restored from `actions/cache` before `mvn dependency:resolve` runs, so only new/changed dependencies hit the network. If Trivy itself is resolving artifacts live (rather than scanning the SBOM), pass `--offline-scan` so it only uses what's already cached locally. |
| npm | Ensure `~/.npm` is restored from cache before `npm ci`. `npm ci` only re-downloads what changed since the lockfile hash last matched. |
| Python (pip) | Ensure `~/.cache/pip` is restored from cache before `pip install`. Consider a private package index/mirror for high-frequency CI if rate limiting persists. |
| Go | Ensure `~/go/pkg/mod` is restored from cache before `go mod download`. `GOPROXY` can be pointed at an internal module proxy to reduce direct calls to the public proxy. |

In every case, the underlying fix is the same: make the dependency cache
step effective (correct path, correct lockfile-derived key, restored
*before* the install/resolve step), so the resolve step itself makes as
few live registry calls as possible. See the
[ecosystem matrix](integrate-a-new-ecosystem.md#ecosystem-matrix) for the
exact cache path and key per ecosystem.

### If the rate limit is hit anyway

Treat it as a transient CI infrastructure failure, not a code or
dependency problem:

1. Re-run the job. Rate limit windows are typically short (minutes, not
   hours).
2. Confirm the cache step actually restored (check the workflow log for a
   cache hit vs. a cold cache) before assuming the mitigation above is
   working as intended.
3. If it recurs frequently on the same repository, consider a scheduled
   "warm the cache" job outside of PR-triggered runs, so PR builds are
   more likely to hit a warm cache rather than resolving cold.

## SBOM generation errors

SBOM generation is handled by `setup-tools.sh --sbom-ecosystem <value>`.
Exact commands per ecosystem are in
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md#ecosystem-matrix).

### Maven: SBOM is empty or missing expected dependencies

The CycloneDX Maven plugin builds the SBOM from Maven's **resolved**
dependency tree. If `mvn dependency:resolve` hasn't run first (or failed
silently), there's nothing resolved for the plugin to read. Confirm the
resolve step succeeded and populated `~/.m2/repository` before generating
the SBOM, this is why `sca.yml` runs `mvn dependency:resolve -q` as its own
step, before `setup-tools.sh --sbom-ecosystem maven`, see
[reference/pipeline-sca.md](../reference/pipeline-sca.md).

If the resulting SBOM contains a version you didn't expect, that's
expected behavior, not a bug: the SBOM reflects whatever Maven's own
conflict resolution decided to use (nearest-definition, first-come-wins,
BOM order), not necessarily the version declared by a specific transitive
dependency. See
[explanation/why-sboms.md](../explanation/why-sboms.md) for the full
mechanics and a worked example. The same class of problem applies to Go, where `go.sum` must be up to date
before `cyclonedx-gomod` runs.

### npm: `ELSPROBLEMS` / peer dependency errors during SBOM generation

`cyclonedx-npm` walks the actual installed `node_modules/` tree, which
only exists after `npm ci` (or `npm install`) has run against
`package-lock.json`. Errors here usually mean the install itself is
broken, not the SBOM tool.

Typical symptom:

```
npm error code ELSPROBLEMS
npm error invalid: @angular/common@X.Y.Z /path/to/node_modules/@angular/common
```

This means `package.json` and `package-lock.json` (or the installed tree)
disagree about what should be there. `npm ci` is intentionally strict
about this and fails rather than silently reconciling it, treat the
failure as a signal the lockfile needs to be regenerated:

```bash
npm install
git add package-lock.json
git commit -m "chore: sync package-lock.json"
git push
```

Do not swap `npm ci` for `npm install` inside the pipeline to work around
this. `npm install` can quietly upgrade dependencies to whatever's newest
on the registry at run time, which would make what's scanned in CI
different from what's actually shipped, and would let the lockfile drift
without anyone noticing. See
[explanation/why-sboms.md](../explanation/why-sboms.md) for why the SBOM
needs to reflect exactly what's locked and shipped, not what's merely
declared as a version range in `package.json`.

If you need a temporary lockfile just to unblock local SBOM generation
without a full install, `npm install --package-lock-only --legacy-peer-deps`
regenerates only the lockfile without touching `node_modules`. Don't rely
on this in CI, it's a local debugging step, not a substitute for `npm ci`.

### `generic`: the SBOM is thinner than you expected

`ECOSYSTEM: generic` runs `trivy fs --format cyclonedx`, which has no
resolve step and no package manager behind it. It finds dependencies only
by recognising lockfiles it already supports (`poetry.lock`,
`requirements.txt`, `Cargo.lock`, `Gemfile.lock`, `composer.lock`, and
others), so coverage depends on whether your lockfile is one of them.

Two consequences worth knowing before trusting the output:

- **Dev dependencies may be excluded.** Trivy prints
  `Suppressing dependencies for development and testing` and leaves them
  out. A native generator such as `cyclonedx-py poetry` includes them.
  Neither is wrong, but the choice decides whether a CVE in a test-only
  package can block a release.
- **No resolution means declared versions, not resolved ones.** This is the
  problem [why-sboms.md](../explanation/why-sboms.md) exists to describe,
  reintroduced in a smaller form.

If the gaps matter, add a native branch for the ecosystem, see
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md#adding-a-new-ecosystem-not-in-the-matrix).
A worked comparison of `generic` against a native Poetry generator is in
[case-studies/datacatalog.md](../case-studies/datacatalog.md#why-python-used-generic).

### Both ecosystems: SBOM step succeeds but scanners still report old data

If Trivy or OSV-Scanner appear to be scanning a stale dependency set,
confirm you're pointing them at the freshly generated `target/bom.json`
(`SBOM_PATH` in `sca_scan.py`) rather than a cached copy from a previous
run. The dependency cache step in `sca.yml` (for example, `~/.npm`, keyed
on the hash of `package-lock.json`) caches the package-manager's own
download cache, not the SBOM itself, so a stale SBOM points to a stale
generation step, not a stale dependency cache.
