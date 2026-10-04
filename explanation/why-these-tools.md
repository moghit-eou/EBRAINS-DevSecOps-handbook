# Why These Tools

Every tool in these pipelines was tested locally before being wired into
CI. This page summarizes what was selected, what was tested and
explicitly rejected, and why. See the case studies for the fuller
narrative of each investigation.

## Selected for CI

| Tool | Role | Why |
|---|---|---|
| **Trivy** | SCA (dependencies + image), Container Scanning | Exposes a simple CLI flag for severity-based failing (`--severity CRITICAL`); scans both SBOMs and container images; also scans `pom.xml`/lockfiles directly as a secondary check. |
| **OSV-Scanner** | SCA (dependencies + image), Container Scanning | Backed by Google's OSV database with fast CVE updates; catches findings Trivy sometimes marks as severity "unknown"; scans SBOMs. |
| **OpenGrep** | SAST, Container Scanning (Dockerfile) | Same CLI and rule format as Semgrep, but without Semgrep's paid-tier feature gating. See [why-opengrep-not-semgrep.md](why-opengrep-not-semgrep.md). |
| **Hadolint** | Container Scanning (Dockerfile) | Dockerfile-specific linter, used alongside OpenGrep's dockerfile ruleset for Dockerfile SAST. |

## Tested and not selected

| Tool | Why it was rejected |
|---|---|
| **npm-audit** | Locked to the npm ecosystem only, doesn't generalize across a multi-ecosystem project, so it wasn't chosen as the standard SCA tool. |
| **OWASP Dependency-Check** | Pulls CVE data from NVD, and NVD's own rate limiting made it unreliable to pull data from consistently. It also does not support scanning an existing SBOM file directly, it scans source code, project manifests, and lockfiles directly instead, which conflicts with this project's SBOM-first approach (see [why-sboms.md](why-sboms.md)). |
| **Dependency-Track** | Useful for local analysis and visualization (uploading a generated SBOM and browsing results in its dashboard), but not wired into CI, it's a standalone platform rather than a CLI step suited to a pipeline gate. |
| **Snyk (Snyk Code)** | Considered as a secondary layer for app-logic vulnerabilities. Free tier caps out at 100 tests/month, which doesn't fit a project with private repos and frequent PR-triggered scans. Requires a `SNYK_TOKEN` and a live network call, so it can't run in a fully offline or local dry-run the way OpenGrep can. |
| **SpotBugs / FindSecBugs** | Java-specific SAST tools, but noisier than OpenGrep, making it harder to separate real findings from false positives. Considered more of a legacy tool relative to the alternatives tested. |

## Why not just pick the "best" single tool per category

No single tool tested covered everything needed:

- Trivy and OSV-Scanner disagree on some findings, and one sometimes has a
  severity score where the other reports "unknown," see
  [why-two-sca-tools.md](why-two-sca-tools.md) for the specific example
  and reasoning that led to running both rather than one.
- Dependency-Check's NVD dependency made it unreliable in exactly the
  place a CI gate needs to be reliable, on every PR.
- Snyk's cost and offline limitations ruled it out as a primary tool for a
  pipeline that needs to run on every PR without a paid budget or a
  network dependency at gate time.

The result is a small, deliberately chosen set of tools per category
rather than a single "best" tool, because none of the alternatives tested
covered the same ground with the same reliability.

## Why local testing came first

Every tool above was run against real local findings on `platform-backend`
and `platform-ui` before it was wired into a GitHub Actions workflow. This
mattered in practice: the false-positive story in
[why-sboms.md](why-sboms.md) was discovered during exactly this local
testing phase, before any CI pipeline existed. Wiring an unvalidated tool
straight into CI would have meant discovering a 300+ finding false-positive
problem inside a blocking pull request check, instead of during controlled
local iteration.

## Related

- [why-two-sca-tools.md](why-two-sca-tools.md)
- [why-opengrep-not-semgrep.md](why-opengrep-not-semgrep.md)
- [why-sboms.md](why-sboms.md)
- [case-studies/platform-backend.md](../case-studies/platform-backend.md)
- [case-studies/platform-ui.md](../case-studies/platform-ui.md)
