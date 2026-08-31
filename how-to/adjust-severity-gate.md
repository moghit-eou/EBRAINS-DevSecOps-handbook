# How to Adjust the Severity Gate

This covers the two different gate models used by these pipelines. Confirm
which one applies to the tool you're adjusting before you start, see
[explanation/why-two-gate-models.md](../explanation/why-two-gate-models.md)
if you're not sure why there are two.

## CVSS-score gate (Trivy, OSV-Scanner)

The thresholds are read from the environment by `ci/parse_sarif.py`, with
`8.0` and `5.0` as defaults:

```python
GATE_FAIL_THRESHOLD = float(os.getenv("GATE_FAIL_THRESHOLD", "8.0"))
GATE_WARN_THRESHOLD = float(os.getenv("GATE_WARN_THRESHOLD", "5.0"))
```

Override them per repository, or per job, without editing any Python. In a
GitHub Actions workflow, add them to the job's `env:` block:

```yaml
    env:
      GATE_FAIL_THRESHOLD: "7.0"
      GATE_WARN_THRESHOLD: "4.0"
```

For a local dockerized run, set them in `ci/docker/env/sca.env` or
`ci/docker/env/container-scan.env`. Do not quote the values there, because
`docker run --env-file` passes quotation marks through literally and
`float()` will reject them:

```dotenv
GATE_FAIL_THRESHOLD=7.0
GATE_WARN_THRESHOLD=4.0
```

`parse_sarif.py` is shared by `sca_scan.py` and `container_scan.py` (the
SCA half), but because the values now come from the environment, the two
pipelines can be tuned independently: set different values in each
workflow, or in each `.env` file.

After changing them, re-run the affected pipeline and confirm the new
threshold is reflected in the printed summary, which interpolates the
current values rather than hardcoding them:

```bash
python ci/sca_scan.py
```

See [reference/gate-status-cvss.md](../reference/gate-status-cvss.md) for
what the default thresholds mean before changing them.

> **Set thresholds deliberately, not to make CI green.** Raising
> `GATE_FAIL_THRESHOLD` because a build is blocked converts a finding into
> a silent acceptance with no record of the decision. For a specific
> vulnerability that cannot be fixed, use a suppression entry with an
> expiry date and a stated reason instead, see
> [suppress-a-finding.md](suppress-a-finding.md).

## Rule-severity gate (OpenGrep, Hadolint)

These tools don't go through `parse_sarif.py` for the pass/fail decision,
each tool's own severity flag decides directly. Adjust the gate by
changing the flag passed to the tool, not by editing a shared file.

**OpenGrep**, in `sast_scan.py` or `container_scan.py`:

```python
gate_cmd = (base_cmd + ["--severity=ERROR", "--error"])
```

Change `--severity=ERROR` to a different OpenGrep severity level
(`WARN`, `INFO`, see
[gate-status-rule-severity.md](../reference/gate-status-rule-severity.md#rule-severity-levels))
to change what counts as a blocking finding.

**Hadolint**, in `container_scan.py`:

```python
cmd = [
    "hadolint", "Dockerfile",
    "--failure-threshold", "error",
    "--format", "sarif",
]
```

Change `--failure-threshold error` to a different Hadolint threshold
(for example `warning`) to change what blocks the pipeline.

See
[reference/gate-status-rule-severity.md](../reference/gate-status-rule-severity.md)
for what the current defaults mean.
