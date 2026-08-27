# About This Handbook and the OWASP Contribution

## Relationship to the OWASP DevSecOps Guideline

This handbook is a companion to, not a replacement for, the
[OWASP DevSecOps Guideline](https://github.com/OWASP/DevSecOpsGuideline).
The guideline provides the general model, this handbook documents one
concrete, working implementation of that model, including the tool
choices, rejected alternatives, and trade-offs that a general guideline
cannot cover. The implementation itself is vendor-neutral: it runs the
same way on GitHub Actions or any other CI runner capable of executing
bash and Python, see
[how-to/integrate-a-new-ecosystem.md](how-to/integrate-a-new-ecosystem.md).

During early research it became clear that two OWASP repositories exist
for this content, and they are not in sync:

| Repository | Status |
|---|---|
| `OWASP/DevSecOpsGuideline` | The active repository; not yet reflected on the public OWASP guideline website at the time of writing. |
| `OWASP/www-project-devsecops-guideline` | The repository currently in sync with the public website; effectively the older version. |

### Upstream pull request

A pull request improving the OWASP DevSecOps Guideline has already been
opened, based directly on lessons learned while building these pipelines:

- **Title:** docs: add SARIF normalization and exit code handling to
  Section 2-3-5 (Security Gates)
- **PR:** [OWASP/DevSecOpsGuideline#107](https://github.com/OWASP/DevSecOpsGuideline/pull/107)
- **Scope:** two improvements: combining multiple scanners for container
  scanning, and normalizing gate checks across scanners so that tools
  reporting CVSS and tools reporting rule severity are handled
  consistently.

## A second upstream contribution: `jwilder/dockerize` CVE report

The OWASP PR above is a documentation contribution. Separately, while
running the container scanning pipeline against `platform-backend`,
container scanning flagged several high-severity vulnerabilities inside
the compiled `dockerize` binary used in `platform-backend`'s container
image build. These aren't flaws in the `dockerize` codebase, they come
from the binary being compiled against an outdated Go standard library
(`v1.25.12`).

| Property | Value |
|---|---|
| Discovered by | Container scan against `platform-backend` |
| Upstream project | `jwilder/dockerize` |
| Root cause | Binary compiled with outdated Go stdlib (`v1.25.12`) |
| Upstream issue | [jwilder/dockerize#339](https://github.com/jwilder/dockerize/issues/339) |
| Requested fix | Rebuild and release compiled with Go 1.25.13 or higher |
| Status | Open |

### `jwilder/dockerize` and this project are unrelated

`jwilder/dockerize` is a separate, third-party project. The report itself
is a standalone contribution to the `jwilder/dockerize` upstream
repository.