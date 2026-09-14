# Why Vendored Scripts, Not Composite Actions

What lands in a consuming repository is a `ci/` folder of plain scripts and
a workflow file of ordinary steps. No custom action, no `uses:` pointing at
this project, no indirection into another repository.

Composite actions were prototyped for all three pipelines and then removed.
This page says why.

## What a composite action would have looked like

A composite action would have collapsed each pipeline into one step:

```yaml
      - uses: moghit-eou/reusable-ci-pipelines/actions/sca@v1
        with:
          ecosystem: maven
          gate-fail-threshold: "8.0"
```

Shorter to write, and updates arrive by bumping a version. Both are real
benefits, and neither outweighed what follows.

## The reason it was dropped: a maintainer has to be able to read it

The people who maintain these pipelines day to day are not the people who
built them. They are the maintainers of the repository being scanned, and
security tooling is not their main job. When a gate blocks a pull request at
five in the evening, the question is always the same: *what exactly did it
run, and why did it fail?*

With vendored scripts, the answer is in the repository they already have
open. The workflow file lists every command. `ci/sca_scan.py` is under a
hundred lines and shows exactly which flags each scanner received. Nothing
is hidden behind a version tag.

With a composite action, that same question means leaving the repository,
finding the action's source, working out which version the tag currently
resolves to, and reading the steps there. A maintainer who did not build the
pipeline should be able to read a workflow file top to bottom and see every
command it runs. That property is worth more here than the lines it saves.

## The supporting reasons

**Debugging matches CI exactly.** The scripts in `ci/` are the same ones the
local `make` path runs. A maintainer can run `python ci/sca_scan.py` on their
own machine and get what CI got. A composite action adds a layer that only
exists inside GitHub Actions, so the local reproduction is no longer the same
code path.

**No upstream dependency at scan time.** A vendored `ci/` keeps working if
this repository is renamed, moved, or archived. An action reference is a live
dependency on another repository resolving correctly on every run, for a
pipeline whose entire job is to check the supply chain.

**Pinning stays honest.** This project pins scanner binaries by SHA256 and
actions by commit SHA precisely because a mutable tag can be moved under you.
Adding our own action would have introduced exactly the kind of reference the
rest of the design argues against, see
[reference/tool-installation-flags.md](../reference/tool-installation-flags.md).

**Modification is expected, not exceptional.** Repositories genuinely differ:
rulesets, exclude lists, thresholds, suppression paths. Some of that fits in
`with:` inputs, but not all of it, and the moment a project needs a change the
action does not expose, it is stuck. With vendored scripts it edits the file.

**It is not CI-provider-specific.** A composite action only works on GitHub
Actions. Plain scripts run on any runner that has bash and Python, which is
what lets this design claim to be vendor-neutral.

## What it costs

Being honest about the trade-off: vendoring means copies, and copies drift.
An improvement to `sca_scan.py` here does not reach a consuming repository
until someone copies it across. There is no automatic update.

That cost is accepted deliberately. Drift in a file you can read is easier to
diagnose than a silent change arriving through a tag you did not move. And in
practice the consuming repositories treat `ci/` as settled: the
[datacatalog adoption](../case-studies/datacatalog.md) copied all seven files
unchanged and altered nothing but `env:` values.

## Related

- [reference/reusable-blueprint.md](../reference/reusable-blueprint.md)
- [case-studies/datacatalog.md](../case-studies/datacatalog.md)
