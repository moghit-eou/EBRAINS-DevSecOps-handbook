# Reference: Secret Scanning Pipeline

| Property | Value |
|---|---|
| Workflow file | `secrets-scan.yml` |
| Orchestrator | None. The tool is called directly |
| Triggers | `pull_request`, `push` to `master`, `workflow_dispatch` |
| Scans | The checked-out working tree |
| Tool | Gitleaks |
| Gate model | Gitleaks' own exit code |
| Where it runs | `platform-backend` and `platform-ui` only |

## Scope, stated plainly

**This pipeline scans the working tree, not git history.** It runs
`gitleaks dir .`, which walks the files present in the checkout. It does not
run `gitleaks git`, which would walk commits.

The practical consequence: a secret that was committed and later removed is
still in history, still retrievable by anyone who can clone the repository,
and this pipeline will not report it. What it catches is a secret about to
be introduced, which is the case a pull request check can actually prevent.

Treat it as a guard on new commits, not as an audit of the repository's
past. A history sweep is separate work, see
[Known limitations](../README.md#known-limitations).

## Why it is not in the blueprint

`reusable-ci-pipelines` does not carry this pipeline. It is deliberately
minimal, it exists to cover an OWASP DSOMM control on the two MIP
components, and it does not have the ecosystem parameterisation, the shared
gate, or the SARIF normalisation that make the other three reusable. A
project wanting secret scanning is better served by adding Gitleaks
directly, or by GitHub's own push protection, than by copying this.

The three pipelines in the blueprint are reusable by design. This one is
not, and is documented here rather than promoted.

## What it runs

```yaml
      - name: Install gitleaks
        run: bash ci/setup-tools.sh --install-tool gitleaks

      - name: Install and run gitleaks
        run: >
          gitleaks dir .
          --config ci/suppress_gitleaks.toml
          --redact
          --report-format sarif
          --report-path "$GITLEAKS_SARIF_OUTPUT"
```

`--redact` keeps the matched secret value out of the SARIF report. Without
it the report itself becomes a copy of the thing you are trying to protect,
and that report is uploaded to the Security tab.

Gitleaks installs through the same `setup-tools.sh` path as every other
scanner, SHA256-pinned with a Renovate marker:

```bash
GITLEAKS_VERSION="${GITLEAKS_VERSION:-v8.30.1}"
GITLEAKS_SHA256="${GITLEAKS_SHA256:-551f6fc...}"
```

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `GITLEAKS_SARIF_OUTPUT` | `gitleaks.sarif` | Report path |

## Suppression

`ci/suppress_gitleaks.toml` holds an allowlist of paths:

```toml
[allowlist]
paths = [
  '''AGENTS\.md''',
]
```

Paths are regular expressions. Prefer allowlisting a specific path over
widening the rule set, and prefer removing the file's contents over
allowlisting it when the match is real.

## Gate

There is no shared gate here. Gitleaks exits non-zero when it finds a leak
and the job fails on that exit code directly. It does not go through
`parse_sarif.evaluate()`, because a leak has no CVSS score, in the same way
that OpenGrep and Hadolint findings do not, see
[why-two-gate-models.md](../explanation/why-two-gate-models.md).

## Differences from the other three pipelines

| | Secret Scanning | The other three |
|---|---|---|
| In the blueprint | No | Yes |
| Python orchestrator | No | Yes |
| Shared gate logic | No | Yes for SCA and Container Scanning |
| Weekly schedule | No | Yes |
| Workflow artifact | No, SARIF goes to the Security tab only | Yes |
| Ecosystem-aware | No | SCA only |

Because it uploads no artifact, the optional artifact encryption in
[how-to/encrypt-sarif-artifacts.md](../how-to/encrypt-sarif-artifacts.md)
does not apply to it.

## Known gap

The `upload-sarif` step in `secrets-scan.yml` is still pinned to a v2 commit
of `github/codeql-action`, where the other three pipelines run v4. It works,
but it should be brought in line.
