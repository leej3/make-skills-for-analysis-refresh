---
name: stamped-neuroimaging-analysis
description: Perform a STAMPED-compliant neuroimaging analysis using an opinionated stack of BIDS Study datasets, DataLad and DataLad Containers, Pixi, Apptainer or Singularity, BABS, and explicit campaign-management patterns. Use when creating, migrating, executing, debugging, auditing, or releasing a scientific neuroimaging analysis, including controlled-data and Slurm workflows.
---

# Perform a STAMPED-compliant neuroimaging analysis

This skill is an opinionated neuroimaging implementation of the [STAMPED principles paper](https://github.com/stamped-principles/stamped-paper). Use the paper as the normative basis; do not substitute this tooling prescription for the principles or restate them in project documentation.

Follow the released [BIDS 1.11.1 Common Principles: Study dataset](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) convention. Record the BIDS specification version rather than silently following the changing development documentation.

## Assign one authority to each concern

| Concern | Authority |
|---|---|
| Python package metadata and importable code | Root `pyproject.toml` and `src/` |
| Local environments and locked package resolution used during image construction | `envs/pixi.toml` and `envs/pixi.lock` |
| Existing scientific runtimes | Exact, validated SIF content from a pinned ReproNim/containers dataset |
| Custom SIF construction | Tracked Apptainer/Singularity definition plus recorded build inputs |
| Result-producing runtime identity | Exact SIF content registered in a DataLad container dataset |
| Code, configuration, data identity, and command provenance | Git, git-annex, DataLad, and DataLad Containers |
| Subject/session fan-out and Slurm operations | BABS |
| Raw/derivative composition and domain metadata | Version-pinned BIDS Study datasets |

Treat a Pixi lock as both a local environment specification and an input to reconstructing an image. Never claim that it alone identifies the runtime used for a result; the registered SIF does.

## Compose each scientific study as a BIDS Study dataset

The top-level research object is not itself a BIDS dataset. For BIDS 1.11.1, use this structure for each scientific study:

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
- Keep original source material, raw BIDS data, scientific derivatives, and BABS administrative state distinct. Put campaigns and attempts in independent `operations/<campaign>/` DataLad datasets, not in the Study as if they were derivatives.
- Point BABS to the installed raw BIDS child, not to the outer Study root. Keep a merged zip dataset in the operations boundary; materialize its payload through explicit `datalad containers-run` provenance into an independent scientific derivative. Install the exact finalized derivative at `studies/<study>/derivatives/<pipeline>-<variant>/` by DataLad dataset identity and commit. Do not copy or filesystem-symlink it from the operations tree.
- Put analyses that combine multiple studies in independent derivative datasets under `results/`, with `SourceDatasets` identifying each exact upstream derivative.

## Establish boundaries and isolate controlled data

- Inventory code, configurations, input datasets, derivatives, containers, models, licenses, and retrieval locations before execution.
- Install public inputs as pinned DataLad datasets or snapshots. For controlled inputs, track only permitted metadata, stable access locations, and exact selection/retrieval procedures in the public object.
- Keep public and controlled studies, derivatives, logs, and storage siblings separate. Test public publication without credentials.
- Treat derivative redistribution as a separate policy decision. Do not infer it from permission to access raw data.
- Record dataset descriptions, license or DUC versions, dataset commits, retrieval methods, and cohort-generation commands before processing.
- Stop and request a policy decision when an input, log, derivative, or participant-level record may violate an agreement.

## Implement BIDS-facing programs as BIDS Apps

- Build every project-authored program whose primary input is a BIDS dataset from the [`bids-apps/example`](https://github.com/bids-apps/example) template, whether or not the app will be published.
- Preserve the standard container-entrypoint contract: positional `bids_dir`, `output_dir`, and `analysis_level`; support only the analysis levels the program actually implements; add `--participant_label` when participant selection is meaningful; expose help and version information; define the BIDS-validation policy; and reject unknown arguments.
- Define app-specific operations only when the science requires them. Give each operation explicit accepted input dataset types, outputs, configuration, seeds, and failure behavior. Do not imply that operation names or participant/group assignments from another analysis form a generic neuroimaging vocabulary.
- Do not let one operation silently execute an internal scientific workflow graph. Invoke each independently meaningful result-changing boundary as its own provenance-captured BIDS App execution.
- Apply the same image, provenance, metadata, and testing requirements to project-authored BIDS Apps as to externally maintained apps.
- Keep notebooks exploratory or explanatory. Do not make notebook state the sole implementation or provenance of a result.
- Put scientific parameters in tracked configuration and scheduler/site settings in separate host profiles.

## Use Pixi for environments and actionable commands

- Keep all Pixi environments in `envs/pixi.toml` and commit `envs/pixi.lock`. Track root symlinks for Pixi discovery; ignore realized `envs/.pixi/` contents and `.venv/`.
- Name environments for responsibilities and genuine dependency boundaries in the analysis at hand; do not prescribe names inherited from another project. Keep incompatible dependency sets in separate environments. Change the manifest and lock only during an explicit dependency-authoring session; use `--locked` for ordinary realization and testing.
- Do not use `pixi pack` or `pixi-pack` for the scientific runtime.
- Use Pixi tasks for developer convenience, validation, and actionable project operations. A Pixi task may invoke a BIDS App, but it does not replace or define the app's container entrypoint. A task definition documents an available operation, not evidence that it ran.
- A task may contain a parameterized BABS command. Document the repository/campaign setup in the README and record each actual BABS lifecycle invocation, arguments, tool versions, and before/after state in `operations/<campaign>/`.
- A task may contain a parameterized `datalad containers-run` command. For a BIDS App step, make the task body invoke DataLad and pass the resolved standard app arguments plus every critical app-specific option after `--`; register the container so this vector reaches its SIF runscript. For a non-BIDS scientific program, record its explicit executable and arguments. The resulting DataLad record must expose the operation that produced the output.
- Never run `datalad run -- pixi run <task>` or `datalad containers-run -- pixi run <task>` for a scientific step. That records only task indirection.
- Never let a result-changing Pixi task invoke the scientific program directly without DataLad provenance. Do not compose scientific stages with Pixi task dependencies or rely on Pixi task caching for replay.

Prefer this direction:

```text
pixi run --locked <parameterized-task>
    -> datalad containers-run <declared inputs/outputs/SIF> -- <explicit app/program arguments>
    -> DataLad run record
```

## Acquire or build, register, and distribute scientific runtimes

For each runtime:

1. Search the pinned ReproNim/containers dataset first. When an existing image has the required application, version, architecture, interface, and redistribution terms, pin its dataset commit and annex key, retrieve the exact SIF, and qualify it with project fixtures.
2. Build a custom SIF only when no existing image is suitable. Track its Apptainer/Singularity definition under `envs/containers/custom/<runtime>/`; adapt reviewed ReproNim, BIDS-Apps, or NeuroDesk patterns where useful, and pin or checksum every build input.
3. For a project-authored image, copy the reviewed Pixi manifest and lock into the definition and realize the selected Linux environment with `pixi install --locked` at its final in-image path.
4. Build the SIF directly with a recorded Apptainer/Singularity version and target architecture. On macOS, build inside Linux with Lima; on Apple Silicon, use an x86_64 guest when the Slurm target is x86_64.
5. Record source dataset/annex identities or definition/lock hashes, tool versions, target platform, build or retrieval command, smoke tests, SIF SHA-256, and annex key. Do not assume that a lock makes every external build input deterministic or that rebuilding yields identical bytes.
6. Register the qualified SIF in the project container dataset with `datalad containers-add` and publish its annex content to persistent storage.
7. Sign and verify the SIF and attach an SBOM or build attestation where supported. When an OCI artifact is intentionally produced, identify and sign it separately; do not require or store an intermediate Docker/OCI application image without a concrete consumer.
8. Distribute required license files with the repository or image when their terms permit it. When redistribution is prohibited, record acquisition and verification and provide the file through a declared runtime bind. Never embed credentials or secrets.

Apptainer builds the image; DataLad Containers registers, retrieves, and executes it. A Pixi task may wrap explicit build, registration, signing, or verification commands for convenience, but the task is not the container identity.

Allow candidate SIFs for local or Slurm debugging only when their outputs are quarantined as disposable and non-authoritative. Before retaining, merging, comparing scientifically, or releasing an output, register the exact SIF and regenerate the output through BABS or `datalad containers-run`. A recipe, lock, mutable registry tag, or local conversion cache is not enough for an authoritative run.

## Execute through DataLad and BABS

- Use `datalad containers-run` for every direct scientific container execution. BABS owns the container execution for BABS participant/session jobs.
- Give BABS pinned BIDS input dataset commits, the DataLad container dataset containing the exact SIF, and reviewed container/campaign configuration.
- Create one independent operational dataset per dataset × pipeline/runtime × input-condition campaign and one independent scientific derivative dataset for each finalized output identity.
- Pilot one participant/session and inspect the run record, SIF identity, input commits, outputs, and resource request before scaling.
- Record `babs init`, setup checks, submission, status-driven retries, and merge as operations/meta-provenance. Pixi tasks may make these commands actionable, while the operations record states what actually ran.
- Acknowledge that the current BABS wrapper and zip layer adds undesirable provenance indirection. Accept it as a BABS-specific limitation, test representative replay, and do not repeat that pattern in project-authored steps.
- Use these campaign patterns credited to [mechababs](https://github.com/asmacdo/mechababs): separate dataset × pipeline × cluster configuration, clean-pin guards, explicit attempt identities, a desired/observed-state ledger, small resumable transitions, and separate requested-versus-realized inclusion records. These patterns do not make mechababs itself part of the execution stack. Do not install or invoke mechababs on the authority of this skill; revise the skill explicitly if the tool is later adopted.

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
