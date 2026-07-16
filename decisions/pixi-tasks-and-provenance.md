# Pixi tasks, DataLad provenance, and BABS operations

- **Status:** accepted
- **Decision date:** 2026-07-16
- **Last reviewed:** 2026-07-16
- **Supersedes:** the earlier prohibition on result-changing or BABS-changing Pixi tasks

## Decision

Use typed, described Pixi tasks as actionable project launchers. They may expose explicit DataLad Containers commands, BABS lifecycle operations, validation, development, retrieval, image operations, and static reproduction meta-graphs. They do not define a BIDS App interface; the tracked container entrypoint does.

The durable record determines the evidence:

- A direct project-authored scientific leaf invokes `datalad containers-run` with the resolved executable or standard BIDS App arguments, critical options, declared inputs and outputs, and accepted SIF.
- A BABS lifecycle leaf invokes a declared operation. BABS/DataLad records participant execution; the campaign operations ledger records the expanded lifecycle command and state transition.
- A static Pixi `depends-on` graph may compose those leaves as the discoverable executable specification for a result set. It neither records nor proves that a child ran.

Every result-changing leaf must produce explicit DataLad/BABS evidence. Never use Pixi `inputs`/`outputs` caching to skip such a leaf.

## Call direction

Prefer:

```text
pixi run --locked <scientific-leaf> <parameters>
    -> datalad containers-run ... -- <resolved scientific command>
    -> run record contains the command, inputs, outputs, and SIF

pixi run --locked <babs-operation> <parameters>
    -> declared BABS lifecycle command
    -> BABS/DataLad execution evidence plus operations-ledger entry
```

Prohibit:

```text
datalad run -- pixi run <task>
datalad containers-run -- pixi run <task>
pixi task -> result-changing scientific program without DataLad/BABS evidence
```

Putting Pixi inside a DataLad record preserves only the task indirection. Putting the explicit DataLad Containers command inside the task lets Pixi substitute parameters before DataLad records the resolved operation. Editing the task later cannot change that historical run record.

## Direct scientific leaf tasks

A task may construct an explicit DataLad Containers command from typed arguments:

```toml
[tasks.extract-stats]
description = "Extract participant morphometrics with the accepted app SIF"
args = ["input", "output"]
cmd = """
datalad containers-run \
  --container-name morphometrics \
  --input "{{ input }}" \
  --input config/extraction.yaml \
  --output "{{ output }}" \
  --message "Extract morphometrics from {{ input }}" \
  -- "{{ input }}" "{{ output }}" participant \
       --operation extract \
       --config config/extraction.yaml \
       --upstream-pipeline freesurfer
"""
```

Register a BIDS App container with a call format that invokes its SIF runscript, so the command after `--` is the standard BIDS App argument vector. Test quoting and repeated arguments with paths containing spaces. Prefer validated path or configuration identifiers over arbitrary shell fragments. Keep scientific parameters in tracked configuration where practical and declare that configuration as an input.

Each leaf produces one understandable provenance record. Use a closed, validated project operation vocabulary; reject arbitrary operations. Keep independently meaningful extraction, assembly, modeling, figure, and validation boundaries as separate recorded leaves.

## Static reproduction meta-graphs

A meta-task such as `reproduce-<result-set>` may join independently executable leaves with `depends-on`. This provides one discoverable command and an executable specification of intended ordering.

The meta-graph is prospective actionability, not execution evidence:

- every result-changing leaf produces its own DataLad/BABS record;
- no result-changing task uses Pixi input/output fingerprints or cache-based skip decisions;
- `result-manifest.tsv` maps each retained result to the actual run record, exact input and SIF identities, and its reproduction task; and
- validation checks the manifest and run records rather than inferring success from the graph.

For dynamic participant fan-out, conditional recovery, retries, or durable scheduler state, use BABS or a tracked orchestrator that invokes and verifies the same provenance-producing leaves. Do not encode opaque control flow in task shell text.

## BABS lifecycle and meta-provenance

BABS owns participant/session fan-out, Slurm submission, branch collection, audit, and merge. Its lifecycle commands are normally not scientific `datalad run` operations. A Pixi task may expose them, while an operations ledger records what was actually invoked and what state changed.

For each campaign:

1. keep exact parameterized BABS lifecycle commands as typed, described Pixi tasks or tracked templates;
2. explain initialization and the operation sequence in the root README and `operations/<campaign>/runbook.md`;
3. append each actual invocation to `operations/<campaign>/commands.jsonl`, including expanded argv, timestamp, actor, working directory, relevant tool versions, configuration hash, exit status, scheduler IDs, and before/after dataset commits; and
4. commit the operation record with the state change it explains.

Pin and qualify a BABS revision with direct Study-layout support using `analysis_path: "."`; the reviewed minimum is [`2cc536a`](https://github.com/PennLINC/babs/commit/2cc536a51282124f3811ffa971f82a7c34116af5). Keep the BABS project, DataLad analysis dataset, and provisional derivative as one dataset identity. Record initialization, setup checks, submission, status inspection, retry, merge, finalization, validation, and acceptance. Accept an exact commit only after merge, provenance-captured finalization, metadata completion, and independent validation.

BABS's generated wrapper and zip/merge layer add provenance indirection. Preserve and inspect the generated execution material and test representative historical replay. This accepted BABS-specific boundary does not authorize an authored scientific program to bypass DataLad or reproduce the indirection elsewhere.

The task definition is prospective actionability. The operations ledger and repository history are evidence of the actual lifecycle operation. Neither replaces BABS/DataLad participant-execution records.

## Other useful task classes

Tasks may also expose:

- `validate-stamped` and `validate-stamped-ideal`;
- lint, unit tests, documentation, schema checks, and environment reports;
- container registration, listing, retrieval, and smoke tests;
- signature verification when that hardening measure was adopted;
- explicit Apptainer/Lima build commands; and
- read-only BABS status and audit queries.

A task may call `datalad run` for a non-container administrative artifact when that is the correct authority. Result-producing scientific steps use an accepted SIF and `datalad containers-run`, except where BABS performs container execution.

## What the records mean

`datalad containers-run` adds the selected image as an input dependency and records the command it receives, declared inputs and outputs, working directory, and dataset state. It does not expand a surrounding Pixi task or infer a runtime from the lock file.

For direct steps, `datalad rerun` can retrieve the registered SIF and replay the explicit historical command. Start verification from the recorded historical state. Applying an old command to newer code or data is a new computation and requires a new run record.

The root result manifest is the result-level index. A task definition says how to attempt reproduction; the manifest and provenance records say which execution produced the retained result.

## Relationship to image construction

Apptainer/Singularity builds a SIF. DataLad Containers registers and directly executes it. Pixi resolves and installs a locked environment during an authored build and may wrap explicit build, qualification, and registration commands; it is not the image builder or runtime identity.

## Acceptance criteria

- Every task is typed where it accepts parameters and has a useful description.
- No scientific DataLad record contains only `pixi run <task>` or another task/Make target.
- Every direct project-authored result-changing leaf records the explicit executable or BIDS App arguments, critical options, inputs, outputs, and accepted SIF through `datalad containers-run`.
- Static Pixi meta-graphs may express dependencies, but every result-changing leaf produces explicit evidence, no result-changing task uses Pixi input/output caching, and no graph or cache state is treated as execution evidence.
- Dynamic fan-out, recovery, retry, and durable scheduler state are delegated to BABS or a tracked orchestrator.
- Every BABS lifecycle task is documented, and every actual state-changing invocation has a corresponding operations-ledger entry.
- BABS participant results resolve to pinned inputs, configuration, BABS revision, and exact SIF content; the wrapper/zip indirection is preserved and tested as a BABS-specific limitation.
- Every retained result in `result-manifest.tsv` resolves to its actual run record, exact inputs and accepted SIF, and executable reproduction task.
- `validate-stamped` and `validate-stamped-ideal` check the assessment rather than relying on task existence as evidence.

## Sources

- [Pixi advanced tasks and typed arguments](https://pixi.prefix.dev/latest/workspace/advanced_tasks/)
- [DataLad Containers extension](https://docs.datalad.org/projects/container/en/stable/)
- [DataLad `containers-run`](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-run.html)
- [BABS documentation](https://pennlinc-babs.readthedocs.io/en/)
