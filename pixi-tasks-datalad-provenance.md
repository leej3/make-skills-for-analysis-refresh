# Pixi tasks, DataLad provenance, and current BABS

**Decision date:** 2026-07-15

## Decision

Pixi task names will never be the command provenance for this analysis. Pixi tasks are limited to developer checks and read-only/meta-level aids. DataLad run records remain authoritative for scientific execution, and an operations ledger records BABS lifecycle commands that BABS does not itself preserve as run provenance.

The current released BABS wrapper and zipped-result pattern is acceptable. It adds indirection, so the project will pin BABS, retain generated scripts and configuration, track the exact SIF and input commits, and test representative replay. Austin Macdonald’s proposed `containers-run` work would improve legibility but is not a prerequisite.

## Why the concern is valid

A commit containing:

```text
pixi run make-figures
```

does not tell a reader which Python module, configuration, or arguments were executed. To interpret it, the reader must check out the historical `pixi.toml`, locate the task, resolve any dependencies, and understand whether Pixi skipped work through its task cache. A later task edit does not corrupt the old Git tree, but it raises the cost of interpreting the run and makes careless forward replay easier.

The same problem applies to a Make target or generated BABS wrapper. Indirection can be historically resolvable while still being inconvenient and opaque.

DataLad improves this only to the extent that it receives an explicit command. A `datalad run` record stores the exact string it was given, declared inputs and outputs, working directory, exit status, and dataset identity. It does not expand a Pixi task name into its historical command and does not automatically recreate a software environment.

## Separate three records

### 1. Scientific execution provenance

This answers: “Which code, arguments, inputs, container, and outputs produced this scientific artifact?”

Use:

- BABS-generated DataLad run records for participant/session processing; and
- explicit `datalad containers-run` records for extraction, cohort construction, statistics, and figures.

The `datalad-container` documentation describes `containers-run` as a drop-in replacement for `datalad run` that adds the selected container image as an input dependency of the run record. That is the desired direct-step behavior: [official command reference](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-run.html).

### 2. BABS lifecycle or meta-provenance

This answers: “Who initialized, checked, submitted, retried, audited, merged, or materialized the campaign, with which configuration and tool versions?”

BABS does not make all of those operator commands scientific DataLad run records. Keep reviewed literal commands and a tracked runbook under `operations/<campaign>/`, then preserve each actual invocation in `commands.jsonl` and commit the record after each state transition. Include exact argv, timestamp, actor, working directory, tool versions, campaign/config hash, before/after commits, scheduler job IDs, and exit status.

The operations record is not a substitute for participant execution provenance. It explains how the BABS campaign was operated.

### 3. Development actionability

This answers: “How does a developer run lint, tests, documentation checks, environment reports, and read-only validation?”

Pixi tasks are useful here because they activate the correct development environment and give short entry points. Git history is sufficient for these non-scientific conveniences.

## Policy for Pixi tasks

Allowed examples:

```toml
[tasks]
lint = "ruff check ."
unit-tests = "pytest -q"
reuse-check = "reuse lint"
environment-report = "python tools/report_environment.py"
campaign-validate = "python tools/validate_campaign.py operations/pilot/campaign.yaml"
babs-status = "babs status"  # read-only invocation; never pass retry/resubmit flags
show-submit-command = "python tools/show_recorded_command.py operations/pilot submit"
```

Prohibited examples:

```toml
[tasks]
extract-stats = "python -m dl_morphometrics_biases.extract ..."
make-figures = "python -m dl_morphometrics_biases.figures ..."
submit = "babs submit"
merge = "babs merge"
reproduce = { depends-on = ["extract-stats", "make-figures"] }
```

Rules:

- No Pixi task may alter raw inputs, derivatives, cohorts, statistics, authoritative figures, or BABS campaign state.
- No Pixi task dependencies are used anywhere in the project.
- No scientific Pixi `inputs`/`outputs` cache is used.
- A task may print a command from a tracked runbook but must not become the only place where its arguments exist.
- Local pre-container scientific work, when unavoidable, is recorded as an explicit `datalad run` command using `pixi run --executable`; the production path uses the tracked SIF.

This deliberately leaves DataLad in charge of scientific replay and means Pixi’s known task-cache behavior is irrelevant to the analysis.

## What `datalad rerun` means here

### Historical verification

To verify an old result, begin from the old result’s parent/tree and its pinned subdatasets and SIF. Re-execute the recorded command there, then compare content. This answers whether the historical state reproduces its output.

### Forward recomputation

Applying an old run record to newer code/data is a new computation. Record a new run commit and label it as an update. Do not describe it as verification of the old result.

### Limits

`datalad rerun` can re-execute only what its run record contains. If that command is a wrapper, task, or Make target, it must still resolve that historical layer. Exporting records with `datalad rerun --script` does not expand those layers.

For each scientific boundary, integration tests should inspect the stored run record, run `datalad rerun --report`, replay a tiny fixture from historical state, and compare output checksums or defined numeric tolerances.

## DataLad containers is not synonymous with `containers-run`

`datalad-container` is the extension. It provides:

- `datalad containers-add` to register an image in a DataLad dataset;
- listing and removal commands; and
- `datalad containers-run` to wrap a command and record the image as an input.

Using a DataLad container dataset therefore does not prove that a job was launched with `containers-run`. Current BABS can consume a DataLad dataset whose SIF was registered with `containers-add` while its generated participant script invokes Singularity through BABS’s existing wrapper.

The project will use the extension in both ways:

- BABS receives a DataLad container dataset containing the exact SIF, whether or not the released BABS job template uses `containers-run` internally.
- Direct downstream scientific steps use `datalad containers-run` so the SIF is an explicit run input.

This does not require Austin’s branch.

## Current BABS provenance position

The official [BABS documentation](https://pennlinc-babs.readthedocs.io/en/) describes BABS 0.5.4 as a DataLad/FAIRly-big workflow for BIDS Apps and Slurm. BABS takes BIDS DataLad datasets, a container DataLad dataset, and configuration; generates subject/session jobs; records job results on branches; and merges successful zipped results.

Current advantages retained:

- exact input dataset states are composed into the BABS project;
- the container is tracked in a DataLad dataset;
- participant/session results and run records are committed independently;
- large campaigns can be audited and retried;
- merged outputs retain history.

Current interpretability cost accepted:

- the run record can expose a generated wrapper rather than the final BIDS App invocation;
- the reader may need the historical generated script and BABS config to understand the exact inner command;
- results are zipped and require a separate materialization step.

Controls:

1. Pin BABS, DataLad, datalad-container, and Apptainer/Singularity versions.
2. Commit the reviewed BABS YAML and campaign specification before `babs init`.
3. Retain generated participant scripts and their hashes.
4. Ensure the BABS project references the exact SIF annex key/checksum, not a mutable registry tag.
5. Run one participant and inspect the stored run record before scaling.
6. Test replay or the strongest supported historical reconstruction on the pilot.
7. Record zip extraction/materialization as its own explicit `datalad run` step, as the local `babs_demo` already demonstrates.
8. State the wrapper indirection in release limitations.

## Austin Macdonald’s proposed remedy

[BABS issue #328](https://github.com/PennLINC/babs/issues/328) proposes generating `datalad containers-run` participant records so container identity and BIDS App arguments are more direct. Austin’s [`add-containers-run-v2` design](https://github.com/asmacdo/babs/blob/add-containers-run-v2/design/containers-run.md) and template branch demonstrate a credible direction.

The previous review found it unmerged and carrying unresolved compatibility items, including pipeline-specific arguments, FreeSurfer licensing, path handling, TemplateFlow, filters, and `--explicit` behavior. The policy is therefore:

- do not make this conversion depend on the branch;
- reassess when upstream publishes a supported implementation;
- migrate only after a representative campaign proves equivalent outputs and improved run records;
- keep current DataLad/BABS provenance rather than delaying the project for a more legible future record.

## Adapting the local `babs_demo`

Retain:

- BIDS study root;
- `sourcedata/raw` DataLad subdataset;
- registered SIF container dataset;
- BABS project under `derivatives/`;
- post-merge `datalad update` and provenance-recorded archive extraction.

Replace before scientific use:

- mutable image tags with exact SIF content;
- UV/unpinned installs with the locked Pixi `babs` environment;
- temporary generated YAML with tracked reviewed configuration;
- local `.env` assumptions with tracked non-secret host profiles;
- the disabled `babs check-setup --job-test` with a passing setup gate;
- implied terminal history with the operations ledger.

## Acceptance criteria

- Every authoritative output has a DataLad/BABS run commit.
- A direct scientific step’s run record contains the program, critical arguments, inputs, outputs, and SIF identity.
- A BABS wrapper record resolves to tracked historical scripts, configuration, input dataset commits, and exact SIF content.
- No result record contains only `pixi run <task>` or `make <target>`.
- No Pixi task can change a campaign or scientific artifact.
- Every BABS lifecycle command is present in the operations ledger.
- Zip files and materialized contents have a recorded relationship.
- A pilot replay demonstrates the remaining BABS indirection is operationally reproducible.
- Release notes state that Austin’s more explicit `containers-run` path is not yet required or relied upon.
