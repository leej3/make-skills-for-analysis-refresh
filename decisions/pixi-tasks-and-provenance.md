# Pixi tasks, DataLad provenance, and BABS operations

- **Status:** accepted
- **Decision date:** 2026-07-16
- **Supersedes:** the earlier prohibition on result-changing or BABS-changing Pixi tasks

## Decision

Pixi tasks may expose parameterized DataLad or BABS commands. The important boundary is not whether a Pixi task starts the operation; it is what the durable record says happened.

For project-authored scientific steps, the task must invoke `datalad containers-run` and pass the explicit scientific executable and arguments to it. DataLad therefore records the resolved command, declared inputs and outputs, and registered SIF—not `pixi run <task>`.

For BABS lifecycle operations such as `init`, setup checks, submission, retry, and merge, a Pixi task may expose the command even though the invocation is not a DataLad scientific run. The task and README make the repository actionable; an operations ledger records what was actually invoked and what state changed.

The two provenance anti-patterns are:

1. `datalad run -- pixi run <task>` or `datalad containers-run -- pixi run <task>`, because the run record preserves only the task indirection.
2. A result-changing Pixi task that invokes the scientific program directly, because no DataLad record captures the resulting scientific change.

Pixi task dependencies and task caching must not define the scientific workflow. DataLad and BABS own execution provenance and replay.

## Why direction matters

These two call graphs look similar at the terminal but create different records:

```text
BAD
datalad containers-run -- pixi run extract
    -> run record: "pixi run extract"
    -> reader must recover and interpret the historical task definition

GOOD
pixi run --locked extract <parameters>
    -> task invokes: datalad containers-run ... -- python -m ... <resolved parameters>
    -> run record: explicit Python module, resolved arguments, inputs, outputs, and SIF
```

In the good direction, Pixi is only a convenient command launcher. Its parameter substitution happens before DataLad receives the command, so the DataLad record is the provenance authority. Editing the Pixi task later does not change the historical run record.

## A parameterized scientific task

Pixi supports typed task arguments. A task may use them to construct an explicit DataLad Containers command:

```toml
[tasks.extract-stats]
args = ["input", "output"]
cmd = """
datalad containers-run \
  --container-name analysis \
  --input "{{ input }}" \
  --input config/extraction.yaml \
  --output "{{ output }}" \
  --message "Extract morphometrics from {{ input }}" \
  -- python -m dl_morphometrics_biases.extract \
       --input "{{ input }}" \
       --config config/extraction.yaml \
       --output "{{ output }}"
"""
```

The precise quoting and repeated-argument behavior must be tested for paths containing spaces. Prefer validated path/config identifiers over arbitrary shell fragments. Keep scientific parameters in tracked configuration where possible, and include that configuration as a declared DataLad input.

Each such task should produce one understandable DataLad run record. Do not join extraction, modeling, and figures with `depends-on`; expose independent DataLad-recorded steps instead.

## BABS as the explicit exception

BABS owns participant/session fan-out, Slurm submission, branch collection, audit, and merge. Commands such as `babs init` are normally executed outside `datalad run`; Git records their effects, not the command line that caused them. This is meta-provenance rather than the scientific run provenance of each participant job.

For each campaign:

1. Keep the exact, parameterized BABS lifecycle commands as Pixi tasks or tracked command templates.
2. Explain repository initialization and the operation sequence in the root README and `operations/<campaign>/runbook.md`.
3. Record each actual invocation in `operations/<campaign>/commands.jsonl`, including the expanded argv, timestamp, actor, working directory, relevant tool versions, configuration hash, exit status, scheduler IDs, and before/after dataset commits.
4. Commit the operation record with the state change it explains.

The task definition is prospective actionability. The ledger plus repository history is evidence of an actual operation. Neither replaces the BABS/DataLad records for participant execution.

This exception also applies to BABS setup commands that create repositories and to administrative queries whose output is worth preserving. It does not authorize a project-authored scientific program to bypass DataLad.

## Other useful Pixi tasks

Tasks may also wrap:

- lint, unit tests, documentation, schema checks, and environment reports;
- `datalad containers-add`, container listing, retrieval, signature verification, and smoke tests;
- an explicit Apptainer/Lima build command;
- read-only `babs status` queries;
- the DataLad-recorded scientific pattern shown above.

A task may call `datalad run` for a non-container administrative artifact where that is the correct authority. Result-producing scientific steps use a registered SIF and `datalad containers-run`, except where BABS performs container execution.

## What DataLad records—and what it does not

`datalad containers-run` is a DataLad run wrapper that adds the selected image as an input dependency. It records the command it is given, declared inputs and outputs, working directory, and dataset state. It does not expand a Pixi task name that surrounds it or infer an environment from a lock file. See the official [`containers-run` reference](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-run.html).

For direct steps, `datalad rerun` can therefore retrieve the registered SIF and replay the explicit recorded command. Historical verification must start from the recorded historical state. Applying an old command to newer code or data is a new computation and should create a new run commit.

## Current BABS limitation

The released BABS path may record a generated wrapper rather than the final inner BIDS App command, and its merged outputs are zipped. This is the same kind of indirection that the project avoids elsewhere. It is accepted as a BABS-specific limitation because BABS contributes the broader DataLad/Slurm campaign model; it must not become a pattern for project-authored steps.

Pin BABS and its configuration, retain the generated state that is needed to interpret a pilot, and test representative replay. Do not make the conversion depend on Austin Macdonald’s unmerged `containers-run` work. Reassess a supported upstream implementation when available.

## Relationship to image construction

Apptainer/Singularity builds a SIF. DataLad Containers registers it with `containers-add` and executes it with `containers-run`. A Pixi task may wrap those explicit operations for convenience. Calling that “Pixi building the image” is inaccurate: Pixi resolves and installs the locked environment inside the image build, while Apptainer is the builder.

## Acceptance criteria

- No scientific DataLad record contains only `pixi run <task>` or another task/Make target.
- Every project-authored result-changing task creates an explicit `datalad containers-run` record naming the registered SIF, inputs, outputs, program, and critical arguments.
- No scientific stage graph depends on Pixi `depends-on`, input/output caching, or skip decisions.
- Every BABS lifecycle task is documented, and every actual state-changing invocation has a corresponding operations-ledger entry.
- BABS participant results still resolve to pinned inputs, configuration, BABS version, and exact SIF content.
- The BABS wrapper/zip indirection is acknowledged as a unique limitation rather than repeated elsewhere.

## Sources

- [Pixi advanced tasks and typed arguments](https://pixi.prefix.dev/latest/workspace/advanced_tasks/)
- [DataLad Containers extension](https://docs.datalad.org/projects/container/en/stable/)
- [DataLad `containers-run`](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-run.html)
- [BABS documentation](https://pennlinc-babs.readthedocs.io/en/)
