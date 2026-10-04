# Reference: Exit Code Handling

Facts only. For the reasoning behind treating a missing SARIF file as an
`ERROR` rather than a pass, see
[why-cvss-based-gating.md](../explanation/why-cvss-based-gating.md).

## Orchestrator script exit codes

| Script | Exit 0 | Exit 1 |
|---|---|---|
| `sast_scan.py` | OpenGrep gate passed | OpenGrep gate failed, or its SARIF output is missing (`ERROR`, treated as a failure) |
| `sca_scan.py` | Every SCA tool passed or only warned | Any SCA tool's gate failed, its SARIF is missing, or it exited non-zero while still writing a SARIF file |
| `container_scan.py --scan-type sast` | Both Hadolint and OpenGrep passed | Either tool's gate failed or errored |
| `container_scan.py --scan-type sca` | Every SCA tool passed or only warned | Same failure conditions as `sca_scan.py` |

## Raw tool exit code handling

| Tool | Raw exit code meaning | How the orchestrator normalizes it |
|---|---|---|
| Trivy (`fs`, `sbom`, `image`) | Reflects Trivy's own run status | Passed through as-is to the orchestrator's exit-code check |
| OSV-Scanner | `1` = vulnerabilities found, not a crash | Normalized to `0` so the pipeline continues to `parse_sarif.evaluate()` for the actual gate decision |
| OpenGrep (gate run) | `0` = no error-severity findings, `1` = error-severity findings present | Used directly as the gate result |
| Hadolint | `0` = passed `--failure-threshold`, `1` = failed it | Used directly as the gate result |

## Why OSV-Scanner's exit code is special-cased

OSV-Scanner returns exit code `1` whenever it finds **any** vulnerability,
regardless of severity. If the orchestrator treated that as a normal
process failure, every scan with any finding at all (even a single LOW
severity item) would short-circuit before the SARIF file could be
evaluated for the actual CVSS-based gate decision. `run_osv_scanner()`
explicitly remaps exit code `1` to `0` so the pipeline always proceeds to
`parse_sarif.evaluate()`, which is what actually decides pass/warn/fail
based on score, not OSV-Scanner's own binary exit code.

## Orchestrator-level status values

All three orchestrators (`sca_scan.py`, `sast_scan.py`,
`container_scan.py`) report one of four statuses per tool:

| Status | Meaning |
|---|---|
| `PASSED` | Tool ran successfully, no gate-blocking findings |
| `WARNING` | (CVSS-gate tools only) Findings present, but below the fail threshold |
| `FAILED` | Gate-blocking findings present |
| `ERROR` | Tool did not run correctly, or its expected SARIF output is missing |

## Process exit codes

| Orchestrator condition | `sys.exit(...)` |
|---|---|
| Any tool status is `ERROR` | `sys.exit(1)` |
| Any tool status is `FAILED` | `sys.exit(1)` |
| All tools `PASSED` or `WARNING` | Implicit `0` |

A `WARNING` status alone never fails the job. Only `FAILED` and `ERROR`
do.

## Missing SARIF output

If the expected SARIF output file does not exist after a tool runs, the
orchestrator does not treat it as "no findings." It's logged as:

```
[!] <tool> SARIF missing: <path>, tool failed to run (not a vulnerability)
```

and the status is forced to `ERROR`, which fails the gate. This
distinction matters: an absent report means the tool crashed or was
misconfigured, not that the code is clean.

## Reports from an earlier run

A report left in the folder by an earlier run could otherwise pass that
existence check and be gated and uploaded as if it were the current one,
for example when a scanner crashes before writing its output during a
local run. To rule this out, each scanner's previous report is removed
before the tool starts (`remove_stale_report()` in `parse_sarif.py`,
Hadolint's report file is emptied when it is opened for writing), so a
report that exists after the run was written by this run. In Container
Scanning, a missing OpenGrep report now gives `ERROR` too, the same as in
the SAST pipeline.

This is in the blueprint. Vendored copies get it when they are refreshed
from it.
