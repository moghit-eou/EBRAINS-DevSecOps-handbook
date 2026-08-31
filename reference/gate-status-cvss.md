# Reference: CVSS-Score Gate

Applies to: **Trivy, OSV-Scanner**, in both the SCA pipeline and the SCA
half of the Container Scanning pipeline. Ecosystem-independent: the same
gate applies whether the SBOM came from Maven, Gradle, npm, Python, or Go,
see [integrate-a-new-ecosystem.md](../how-to/integrate-a-new-ecosystem.md).

## How the score is computed

`parse_sarif.evaluate()` reads every result in the given SARIF file(s),
looks up each result's `security-severity` property (declared on the
matching rule), and takes the **single highest score across all
results**:

```python
severities = {r["id"]: r.get("properties", {}).get("security-severity") for r in rules}

for result in run.get("results", []):
    score = severities.get(result["ruleId"])
    if score is not None:
        max_score = max(max_score, float(score))
```

One number, `max_score`, decides the outcome for the entire file. There is
no "average" or "count of findings above X" logic, a single CRITICAL
finding among a hundred LOW findings is enough to fail the gate.

## Thresholds

| Status | Condition | Blocks the pipeline? |
|---|---|---|
| `PASSED` | `max_score < GATE_WARN_THRESHOLD` | No |
| `WARNING` | `GATE_WARN_THRESHOLD <= max_score < GATE_FAIL_THRESHOLD` | No (logged only) |
| `FAILED` | `max_score >= GATE_FAIL_THRESHOLD` | **Yes** |
| `ERROR` | Tool crashed, or SARIF file missing/unparseable | **Yes** |

`GATE_FAIL_THRESHOLD` defaults to `8.0` and `GATE_WARN_THRESHOLD` to
`5.0`. Both are read from the environment, so a repository can tune them
without editing `parse_sarif.py`, see
[adjust-severity-gate.md](../how-to/adjust-severity-gate.md).

## Reference: standard CVSS severity bands

| CVSS score | Severity |
|---|---|
| 0.1 - 3.9 | LOW |
| 4.0 - 6.9 | MEDIUM |
| 7.0 - 8.9 | HIGH |
| 9.0 - 10.0 | CRITICAL |

Note this pipeline's own thresholds (5.0 / 8.0) don't line up exactly on
these standard band boundaries, 8.0 sits inside the standard HIGH band,
not at the CRITICAL boundary. See
[why-cvss-based-gating.md](../explanation/why-cvss-based-gating.md) for
why 8.0 was chosen as the fail line rather than 9.0.

## Where the threshold can be changed

The 5.0 / 8.0 values are hardcoded in `parse_sarif.py`, not exposed as a
configuration option. See
[adjust-severity-gate.md](../how-to/adjust-severity-gate.md) to change
them.

