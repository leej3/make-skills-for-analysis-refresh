---
name: stamped-neuroimaging-analysis
description: Coordinate or audit a reproducible neuroimaging research object using Pixi, Apptainer/Singularity, DataLad Containers, BIDS Study datasets, and BABS on Slurm. Use when designing, migrating, executing, reviewing, debugging, or releasing a containerized BIDS analysis under the STAMPED principles, including analyses with access-controlled inputs or derivatives.
---

# Coordinate a STAMPED neuroimaging analysis

Use the local [STAMPED principles paper](../../../Macdonald%20et%20al.%20-%20STAMPED%20principles%20for%20reproducible%20research%20objects.pdf), or its [canonical source](https://github.com/stamped-principles/stamped-paper), as the normative basis. Do not restate the principles. Apply them through the responsibilities and gates below.

Follow the released [BIDS 1.11.1 Common Principles: Study dataset](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) convention. Record the BIDS specification version rather than silently following the changing development documentation.

## Assign one authority to each concern

| Concern | Authority |
|---|---|
| Python package metadata and entry points | Root `pyproject.toml` |
| Local environments and locked package resolution used during image construction | `envs/pixi.toml` and `envs/pixi.lock` |
| SIF construction | Tracked Apptainer/Singularity definition plus recorded build inputs |
| Result-producing runtime identity | Exact SIF content registered in a DataLad container dataset |
| Code, configuration, data identity, and command provenance | Git, git-annex, DataLad, and DataLad Containers |
| Subject/session fan-out and Slurm operations | BABS |
| Raw/derivative composition and domain metadata | Version-pinned BIDS Study datasets |

Treat a Pixi lock as both a local environment specification and an input to reconstructing an image. Never claim that it alone identifies the runtime used for a result; the registered SIF does.

## Compose the research object as a BIDS Study dataset

For BIDS 1.11.1, use this structure for each study:

```text
<study>/
├── dataset_description.json       # DatasetType: study
├── README
├── sourcedata/
│   ├── raw/                       # independent raw BIDS/DataLad dataset
│   └── <non-BIDS-source>/         # only when retained and permitted
└── derivatives/
    └── <pipeline>-<variant>/      # independent derivative/DataLad dataset
```

- Pin the BIDS version before adopting the layout; newer released versions may replace `sourcedata/raw/` with another path.
- Give the outer study, raw child, and each BIDS derivative its own `dataset_description.json` and required README. Set the outer `DatasetType` to `study`.
- Give each derivative `GeneratedBy` metadata and a `SourceDatasets` reference to its immediate source where applicable. Follow the recommended `<pipeline-name>-<variant>` directory naming rule.
- Treat metadata inheritance as local to each BIDS dataset root. Validate the installed raw child and every claimed BIDS derivative independently.
- Keep original source material, raw BIDS data, scientific derivatives, and BABS administrative state distinct. Do not label operational artifacts as BIDS derivatives unless they satisfy the derivative requirements.
- Compose the outer study, raw input, and each processing campaign as separate DataLad datasets or subdatasets. Point BABS to the installed raw BIDS child, not to the outer study root.

## Establish boundaries and isolate controlled data

- Inventory code, configurations, input datasets, derivatives, containers, models, licenses, and retrieval locations before execution.
- Install public inputs as pinned DataLad datasets or snapshots. For controlled inputs, track only permitted metadata, stable access locations, and exact selection/retrieval procedures in the public object.
- Keep public and controlled studies, derivatives, logs, and storage siblings separate. Test public publication without credentials.
- Treat derivative redistribution as a separate policy decision. Do not infer it from permission to access raw data.
- Record dataset descriptions, license or DUC versions, dataset commits, retrieval methods, and cohort-generation commands before processing.
- Stop and request a policy decision when an input, log, derivative, or participant-level record may violate an agreement.

## Make scientific boundaries explicit

- Move authoritative processing, extraction, cohort construction, modeling, and figure generation into importable modules with non-interactive CLIs.
- Give every command explicit inputs, outputs, configuration, seeds, and failure behavior.
- Keep notebooks exploratory or explanatory. Do not make notebook state the sole implementation or provenance of a result.
- Separate participant processing, extraction, cohort assembly, modeling, figure generation, and validation into independently recorded scientific steps.
- Put scientific parameters in tracked configuration and scheduler/site settings in separate host profiles.

## Use Pixi for environments and actionable commands

- Keep all Pixi environments in `envs/pixi.toml` and commit `envs/pixi.lock`. Track root symlinks for Pixi discovery; ignore realized `envs/.pixi/` contents and `.venv/`.
- Define separate environments for real dependency conflicts. Change the manifest and lock only during an explicit dependency-authoring session; use `--locked` for ordinary realization and testing.
- Do not use `pixi pack` or `pixi-pack` for the scientific runtime.
- Use Pixi tasks for developer convenience, validation, and actionable entry points. A task definition documents an available operation; it is not evidence that the operation ran.
- A task may contain a parameterized BABS command. Document the repository/campaign setup in the README and record each actual BABS lifecycle invocation, arguments, tool versions, and before/after state in `operations/<campaign>/`.
- A task may contain a parameterized `datalad containers-run` command. For a scientific step, make the task body invoke DataLad and put the explicit scientific executable and all critical arguments after `--`, so the resulting DataLad run record contains the command that produced the output.
- Never run `datalad run -- pixi run <task>` or `datalad containers-run -- pixi run <task>` for a scientific step. That records only task indirection.
- Never let a result-changing Pixi task invoke the scientific program directly without DataLad provenance. Do not compose scientific stages with Pixi task dependencies or rely on Pixi task caching for replay.

Prefer this direction:

```text
pixi run --locked <parameterized-task>
    -> datalad containers-run <declared inputs/outputs/SIF> -- <explicit program and arguments>
    -> DataLad run record
```

## Build, register, and distribute scientific runtimes

For each runtime:

1. Track an Apptainer/Singularity definition under `envs/containers/<runtime>/`. Pin or checksum the base and every non-Pixi build input.
2. Copy the reviewed Pixi manifest and lock into the definition and realize the selected Linux environment with `pixi install --locked` at its final in-image path.
3. Build the SIF directly with a recorded Apptainer/Singularity version and target architecture. On macOS, build inside Linux with Lima; on Apple Silicon, use an x86_64 guest when the Slurm target is x86_64.
4. Record the definition and lock hashes, Pixi and builder versions, target platform, build command, smoke tests, SIF SHA-256, and annex key. Do not assume that a lock makes every external build input deterministic or that rebuilding yields identical bytes.
5. Use `datalad containers-add` to register the completed SIF in a DataLad container dataset and publish its annex content to persistent storage. Use `datalad containers-run` for direct scientific execution.
6. Sign and verify the SIF and attach an SBOM or build attestation where supported. When an OCI artifact is intentionally produced, identify and sign it separately; do not require or store an intermediate Docker/OCI application image without a concrete consumer.
7. Distribute required license files with the repository or image when their terms permit it. When redistribution is prohibited, record acquisition and verification and provide the file through a declared runtime bind. Never embed credentials or secrets.

Apptainer builds the image; DataLad Containers registers, retrieves, and executes it. A Pixi task may wrap explicit build, registration, signing, or verification commands for convenience, but the task is not the container identity.

Allow candidate SIFs for local or Slurm debugging only when their outputs are quarantined as disposable and non-authoritative. Before retaining, merging, comparing scientifically, or releasing an output, register the exact SIF and regenerate the output through BABS or `datalad containers-run`. A recipe, lock, mutable registry tag, or local conversion cache is not enough for an authoritative run.

## Execute through DataLad and BABS

- Use `datalad containers-run` for direct scientific container execution. The exception is execution owned by BABS or another provenance-aware orchestrator that cannot use it directly.
- Give BABS pinned BIDS input dataset commits, the DataLad container dataset containing the exact SIF, and reviewed container/campaign configuration.
- Create one independent derivative dataset per dataset × pipeline/runtime × input-condition campaign.
- Pilot one participant/session and inspect the run record, SIF identity, input commits, outputs, and resource request before scaling.
- Record `babs init`, setup checks, submission, status-driven retries, and merge as operations/meta-provenance. Pixi tasks may make these commands actionable, while the operations record states what actually ran.
- Acknowledge that the current BABS wrapper and zip layer adds undesirable provenance indirection. Accept it as a BABS-specific limitation, test representative replay, and do not repeat that pattern in project-authored steps.

## Validate before retaining or releasing results

Require all of the following:

- A clean recursive clone can retrieve every permitted component or provides exact controlled-access instructions.
- Every retained result-changing step has an intelligible DataLad/BABS run record resolving to code, configuration, input commits, and exact SIF content.
- `datalad rerun --report` is intelligible, and representative steps replay from historical state.
- Clean SIF executions reproduce small-fixture outputs and pass domain checks.
- The outer study, raw input, and each claimed derivative validate against the pinned BIDS version or carry reviewed exceptions.
- SIF checksums, signatures, SBOM/attestations, and retrieval locations verify.
- Public release tests cannot retrieve controlled annex content.
- Code, documentation, data, models, containers, and included license files have compatible license/DUC records.
- A result manifest maps each authoritative figure, table, and statistic to its generating commits and files.

Report failed gates with evidence. Do not scale a campaign to compensate for an unresolved pilot failure.
