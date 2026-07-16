# Architecture and provenance for the converted analysis

## Outcome

The target is one top-level DataLad research object named `stamped_dl_morphometrics_biases`. It composes importable analysis code, environment definitions, registered SIF images, two BIDS Study datasets, independent scientific derivatives, campaign operations, and cross-study result datasets without placing controlled ABCD content in the public code history.

This document records responsibility boundaries and identities. The [conversion plan](conversion-plan.md) records sequencing.

The study layout follows the released [BIDS 1.11.1 Common Principles: Study dataset](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) convention. Each study records its BIDS version so that a future specification change does not silently change the repository structure.

## Responsibility map

| Layer | What it controls | What it does not control |
|---|---|---|
| `pyproject.toml` | Python package metadata, build configuration, and importable project code | Conda system dependencies, command provenance, or production runtime identity |
| Pixi workspace | Named local environments, locked conda-forge/PyPI resolution used during SIF construction, and actionable task interfaces | Scientific provenance, image construction, or runtime identity |
| Apptainer/Singularity definition and build | Direct construction of a Linux SIF from tracked and checksummed inputs | Registration, data provenance, or proof that a result used the image |
| SIF image | Immutable runtime executed on Slurm | Raw-data or result provenance by itself |
| DataLad/git-annex and DataLad Containers | Component identity and availability, SIF registration/retrieval, direct container execution, run records, and historical state | General-purpose image building or subject/session scheduling |
| BIDS Study layout | Domain-valid composition of source, raw BIDS, and derivative dataset roots | Version control by itself |
| BABS | Participant/session expansion, Slurm job lifecycle, auditing, merge | Transparent provenance for its own lifecycle commands |
| Campaign operations dataset and ledger | BABS/mechababs state, exact lifecycle invocations, attempts, and campaign transitions | Scientific dataset identity or participant execution, which remains in DataLad/BABS records |

## Target layout

```text
stamped_dl_morphometrics_biases/
├── .datalad/
├── pyproject.toml
├── pixi.toml -> envs/pixi.toml
├── pixi.lock -> envs/pixi.lock
├── .pixi -> envs/.pixi
├── apps/
│   └── <app-name>/run.py         # container entrypoint based on bids-apps/example
├── src/dl_morphometrics_biases/
│   ├── cohort.py
│   ├── resample.py
│   ├── extract.py
│   ├── assemble.py
│   ├── model.py
│   ├── figures.py
│   └── validate.py
├── config/
│   ├── scientific/
│   ├── babs/
│   └── hosts/
├── envs/
│   ├── pixi.toml
│   ├── pixi.lock
│   ├── .pixi/.gitignore
│   ├── images.lock.yaml
│   ├── containers/
│   │   ├── repronim/                # pinned ReproNim DataLad subdataset
│   │   ├── custom/<runtime>/        # definitions only when needed
│   │   └── accepted/                # project DataLad dataset of registered SIFs
├── studies/
│   ├── ds007116/                 # DataLad study subdataset
│   │   ├── dataset_description.json  # DatasetType: study; BIDSVersion: 1.11.1
│   │   ├── README
│   │   ├── sourcedata/raw/            # independent raw BIDS/DataLad root
│   │   └── derivatives/<pipeline>-<variant>/  # scientific DataLad root
│   └── abcd/                     # access-controlled DataLad study subdataset
│       ├── dataset_description.json  # DatasetType: study; BIDSVersion: 1.11.1
│       ├── README
│       ├── sourcedata/raw/            # independent protected raw BIDS/DataLad root
│       └── derivatives/<pipeline>-<variant>/  # protected scientific root
├── operations/<campaign>/       # independent operational DataLad dataset
│   ├── campaign.yaml
│   ├── runbook.md
│   ├── commands/                 # reviewed literal BABS lifecycle commands
│   ├── commands.jsonl
│   ├── attempts/
│   ├── babs-project/             # when required by the pinned BABS layout
│   ├── generated/
│   └── audit/
├── results/<analysis>-<variant>/   # independent cross-study derivative/DataLad root
│   ├── dataset_description.json
│   ├── models/
│   ├── figures/
│   └── result-manifest.tsv
├── notebooks/                    # optional, non-authoritative
├── tests/
├── docs/
├── skills/stamped-neuroimaging-analysis/
├── LICENSES/
└── REUSE.toml
```

The research-object root is not itself a BIDS dataset. Each `studies/<study>/` child is a BIDS Study dataset and a distinct DataLad dataset; its raw BIDS child and every scientific derivative are also distinct DataLad dataset roots with their own identities and `dataset_description.json`. The outer description sets `DatasetType` to `"study"`. Each derivative records `GeneratedBy` and a `SourceDatasets` reference to its actual immediate input or inputs, such as `../../sourcedata/raw/`. Validate the installed raw child, Study composition, and every claimed BIDS derivative independently.

The study hierarchy is the scientific and publication view. `operations/<campaign>/` is a separate DataLad boundary for BABS or mechababs administration, including attempts, lifecycle state, generated configuration, and any outer BABS project required by the pinned tool version. It is not a BIDS derivative. BABS receives the installed raw BIDS child rather than the outer Study dataset.

The BABS merged archive dataset is a scientific upstream artifact retained inside the campaign operations boundary when its payload is zipped or otherwise not yet a valid claimed derivative. Materialize that payload with an explicit `datalad containers-run` operation into a separate, independently described scientific derivative dataset. Install the exact finalized derivative at `studies/<study>/derivatives/<pipeline>-<variant>/`, pinned by dataset ID and commit. If a future BABS version emits an already valid derivative dataset, that exact dataset may be installed directly after validation. If the campaign also refers to the finalized derivative, both locations identify the same dataset and commit through DataLad; never connect them with a filesystem symlink or an untracked file copy. The campaign or storage RIA is a retrieval source, not a second scientific identity. Per-study downstream products follow the same derivative rule. A result combining multiple studies belongs in an independent `results/<analysis>-<variant>/` derivative dataset whose `SourceDatasets` identifies every upstream dataset and version.

## Environment layout

The real Pixi files live under `envs/`, with tracked root symlinks for conventional discovery:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi
```

Root `pyproject.toml` contains no Pixi environment tables. This is a project policy, not a limitation of Pixi’s `pyproject.toml` support: `pixi.toml` keeps non-Python dependencies and multiple operational environments from overwhelming Python package metadata.

Track `envs/pixi.toml` and `envs/pixi.lock`. The lock supports local realization and, for a custom Pixi-based image, supplies the reviewed Linux package solution installed during SIF construction; it does not identify the completed image. Ignore `.venv/` and the realized contents of `envs/.pixi/`. Define independently solved environments for at least:

- BABS/DataLad orchestration;
- development and tests;
- analysis, including `pandas<2` while the selected `freesurfer-stats` release requires it;
- named image environments where processing runtimes have different dependency graphs.

The `pandas<2` compatibility constraint belongs to the analysis itself, not to a separate “legacy extraction” path. When `freesurfer-stats` supports `pandas>=2` and the analysis passes its regression tests, update the same `analysis` environment and lock. Do not use solve groups to erase a real incompatibility between analysis and orchestration or development tooling.

## Runtime construction and identity

```text
ReproNim container dataset             tracked custom definition
  + pinned commit/image key       or    + pixi manifest/lock when used
  + verified SIF checksum               + checksummed build inputs
                 │                       + recorded builder/architecture
                 │                                  │
                 │                                  ▼
                 │                         direct SIF construction
                 └──────────────────────┬───────────┘
                                        ▼
                                  exact SIF bytes
                                        ├── candidate/debug execution
                                        │       └── quarantined outputs; never promoted
                                        │
                                        │ register exact SIF bytes
                                        ▼
       SIF SHA-256 + git-annex key + DataLad commit
                              │
                              ▼
       DataLad/BABS run record + derivative commit
```

The image ledger under `envs/images.lock.yaml` should record, for every runtime:

- logical image name and scientific tool version;
- source type and, for ReproNim, its container dataset commit, image name, annex key, and upstream checksum;
- for a custom image, the Apptainer definition path and Git commit, manifest and lock hashes when used, checksums for every build input, and the exact build command;
- Pixi and Apptainer/Singularity versions when they participate in construction;
- builder environment, including Lima configuration when used, and target architecture;
- SBOM identity;
- SIF signing identity, verification policy, and bundle or native signature reference;
- SIF SHA-256, size, annex key, DataLad dataset ID/commit, and persistent retrieval locations;
- required bind mounts, distributed or externally supplied license files, environment variables, and smoke tests;
- source and completed OCI digests only when an OCI artifact is intentionally consumed or published.

Use an exact SIF from [ReproNim/containers](https://github.com/ReproNim/containers) when it provides the required software version and runtime behavior. Pin the ReproNim container dataset commit and the selected image’s annex key and checksum in the image ledger, then register or reference those exact bytes from the project container dataset as required by the pinned BABS layout. Do not infer image availability from a desired software version. ReproNim currently registers a [FreeSurfer 7.4.1 BIDS App](https://github.com/ReproNim/containers/blob/0284fc8ad8b7fa9a76c3c9f03cfb2919708ba2b2/.datalad/config#L37-L39) and a [FreeSurfer 8.2.0 NeuroDesk image](https://github.com/ReproNim/containers/blob/0284fc8ad8b7fa9a76c3c9f03cfb2919708ba2b2/.datalad/config#L303-L305); the latter is not itself the required BIDS App interface.

Create a custom definition only when ReproNim lacks the required version or its image is unsuitable for the declared interface. Reuse the relevant ReproNim registration conventions and reviewed BIDS-Apps or NeuroDesk build/interface code where possible, recording every divergence. ReproNim/containers primarily imports and registers application images; it is not necessarily the upstream application build system. For a custom image, build the SIF directly from the tracked definition. Apptainer/Singularity is the builder; Pixi resolves and installs any selected locked environment during the build; DataLad Containers registers, retrieves, and executes the completed image. A Pixi task may wrap those explicit operations but does not become the builder or image identity.

On macOS, run the build inside Linux with Lima; on Apple Silicon, use an x86_64 guest when the Slurm target is x86_64. A separate OCI application image is optional. If the definition consumes an OCI base, pin it by digest. Build and publish a completed OCI application image only for a documented consumer, and identify it separately from the authoritative SIF. The exact SIF may also be distributed through an ORAS-capable registry, but its git-annex key and DataLad commit remain its research-object identity.

Sign and verify the SIF with a tracked Sigstore blob bundle and/or Apptainer’s native SIF signing. Sign an OCI artifact separately when one exists. See [Sigstore verification](https://docs.sigstore.dev/cosign/verifying/verify/) and [Apptainer SIF verification](https://apptainer.org/docs/user/main/signNverify.html).

### Candidate and authoritative images

Unregistered candidate SIFs may be used locally or on Slurm to debug installation, architecture, bind mounts, licenses, scheduler integration, or resource requests. Write their outputs only to ignored scratch or another quarantined location; never merge, retain, compare scientifically, or cite them. If a debug observation influences a parameter, inclusion decision, or model choice, record that decision and confirm it with an authoritative run.

Before retaining any output, hash and smoke-test the exact SIF, register it with `datalad containers-add`, publish its annex content to declared storage, point BABS or `datalad containers-run` to the registered image, and regenerate the output. Registration after a debug run cannot retroactively make that earlier output authoritative. The full gate is recorded in the [candidate versus authoritative SIF decision](decisions/runtime-image-strictness.md).

## Project-authored BIDS Apps

Treat project-authored cohort construction, resampling, extraction, assembly, modelling, figures, and validation as scientific applications with the same runtime and provenance expectations as reconstruction. Implement every operation whose primary input is a BIDS dataset or BIDS derivative as a BIDS App based on [`bids-apps/example`](https://github.com/bids-apps/example), whether or not it will be published. Each app has a container entrypoint accepting `bids_dir`, `output_dir`, and an implemented `analysis_level`, with participant selection, validation, help, version, and unknown-argument behavior defined as applicable. The entrypoint calls tested importable project code.

These are candidate project operations, not a generic BIDS App vocabulary. Decide which operations are separate apps and which are coherent modes of one app before importing their code:

| Operation | Allowed analysis level | Scientific product |
|---|---|---|
| `cohort` | `group` | pinned inclusion/exclusion table and cohort metadata |
| `resample` | `participant` | resampled-input derivative for selected participants/sessions |
| `extract` | `participant` | subject/session morphometric tables from reconstruction derivatives |
| `assemble` | `group` | cohort-level analysis table assembled from participant outputs |
| `model` | `group` | statistical estimates, diagnostics, and machine-readable tables |
| `figures` | `group` | declared figures and presentation tables |
| `validate` | `participant` or `group` | machine-readable validation and acceptance reports |

Freeze each app’s required input dataset types, supported operations and analysis levels, optional session/participant filters, configuration schema, output files, and exit behavior before relying on it. An app may consume a raw BIDS dataset, one or more BIDS derivatives, or both; record all immediate sources in its output metadata. Capture every result-changing operation independently with BABS or `datalad containers-run`. Do not hide a chain of operations inside one run record or use Pixi task dependencies as that chain.

Pixi tasks may launch these apps for actionability, but they do not define the app interface. The SIF runscript invokes the tracked `apps/<app-name>/run.py` entrypoint, and the DataLad record exposes the standard BIDS App arguments plus any explicit project operation and configuration. `pyproject.toml` describes the importable package used by the entrypoint.

## Campaign matrix

Use one campaign identity per dataset × pipeline/runtime × input condition. Do not overload a directory with outputs from different tool versions or acquisition transformations.

| Dataset | Campaign family | Target runtime | Input condition | Phase |
|---|---|---|---|---|
| ds007116 | `reconall-fs7` | FreeSurfer 7.4.1 | available native structural inputs | pilot first |
| ds007116 | `reconall-fs8` | FreeSurfer 8.2.0 | matching native inputs | pilot |
| ds007116 | `recon-any` | version to pin | native T1w; then 1×1×5 mm where valid | pilot |
| ds007116 | `recon-all-clinical` | version to pin | native T1w; then 1×1×5 mm where valid | pilot |
| ABCD | same four families | exact versions above | poster-declared native/resampled conditions | protected scale-up |

The table declares intended scientific comparisons, not the availability of prebuilt BIDS Apps. Resolve each runtime first against ReproNim/containers; for a missing or unsuitable interface, review the actual upstream application build and ReproNim registration path before creating a tracked custom SIF definition. For FreeSurfer 8.2.0, qualify the registered generic NeuroDesk image as an installation/runtime base or build the required BIDS App wrapper explicitly.

`ds007116` is the public Penn LEAD BIDS dataset and has a DataLad-backed OpenNeuro repository. Pin a released snapshot or commit rather than its moving default branch. OpenNeuro exposes snapshot Git hashes and DataLad download instructions in its [user guide](https://docs.openneuro.org/user_guide.html), and the dataset is available in the [OpenNeuro DataLad collection](https://github.com/OpenNeuroDatasets/ds007116).

Use stable campaign names such as:

```text
ds007116__reconall-fs7.4.1__native-t1t2__v1
abcd__recon-any-<version>__t1w-1x1x5__v1
```

The campaign record must contain the exact dataset commit, container key, BABS config hash, processing level, input filters, resources, exclusion policy, and expected output pattern.

## Three types of provenance

### Scientific execution provenance

This is authoritative. Use BABS-generated DataLad run records when a BIDS App target needs participant/session expansion and Slurm scheduling; use explicit `datalad containers-run` records for direct participant or group execution. Reconstruction and project-authored targets have the same provenance bar. The choice between BABS and direct execution reflects scheduling topology, not a difference in scientific importance. Each record must resolve to exact inputs, code/configuration, outputs, and registered SIF content.

A parameterized Pixi task may launch either command path. For a project-authored scientific step, put DataLad inside the task and pass the explicit scientific program and critical arguments after `--`, so the durable record contains the resolved command rather than a task name:

```text
allowed:    pixi task -> datalad containers-run -> explicit scientific command
prohibited: datalad containers-run -> pixi task -> hidden scientific command
prohibited: pixi task -> scientific command without DataLad provenance
```

Do not use Pixi task dependencies, input/output fingerprints, or skip decisions as the scientific workflow. DataLad and BABS own scientific provenance and replay. See the [Pixi tasks and provenance decision](decisions/pixi-tasks-and-provenance.md).

### Lifecycle or meta-provenance

`babs init`, setup checks, `submit`, `status`, retry, and `merge`, and equivalent mechababs transitions affect orchestration but are not all captured as scientific DataLad run records. Expose their exact parameterized forms as Pixi tasks or tracked command templates, document repository and campaign setup in the README and `runbook.md`, and do not hide substantive arguments in user shell variables.

Record each command that was actually executed verbatim in `operations/<campaign>/commands.jsonl`, including:

```json
{
  "timestamp": "RFC3339 timestamp",
  "actor": "identity",
  "cwd": "path relative to research-object root",
  "argv": ["babs", "submit", "--count", "1"],
  "tool_versions": {"babs": "...", "datalad": "..."},
  "campaign_config_sha256": "...",
  "before_commit": "...",
  "after_commit": "...",
  "scheduler_job_ids": ["..."],
  "exit_code": 0
}
```

The Pixi tasks or command templates provide prospective actionability; the JSONL ledger records what actually happened. Store generated BABS/mechababs scripts, configuration snapshots, and attempt identities beside both. A Pixi task may execute a state-changing lifecycle command, but the expanded argv and resulting state transition must be entered in the ledger and committed with the change it explains. Neither the task nor the ledger replaces participant-level BABS/DataLad run records or the exact finalized derivative commit installed under `studies/`.

### Development history

Git commits, pull requests, tests, notebooks, and Pixi task definitions explain how code was developed and make operations discoverable. A task definition is not evidence that it ran, and these records do not substitute for scientific provenance or the BABS operations ledger.

## BABS demo and mechababs as design inputs

BABS is the chosen Slurm layer. Its current release uses DataLad and the FAIRly big pattern, creates participant/session result branches, stores zipped results, and merges successful outputs. The [BABS documentation](https://pennlinc-babs.readthedocs.io/en/) describes DataLad datasets for BIDS inputs and containers as required inputs.

The local [`babs_demo` snapshot](https://github.com/djarecka/babs_demo/tree/7d46f3763cbf8f76b17e5f345160a07d042446e8) demonstrates useful mechanics:

- `sourcedata/raw` is a DataLad subdataset;
- a SIF is registered with `datalad containers-add` in a container dataset;
- the BABS project is placed under `derivatives/`;
- merged zip extraction is itself recorded with `datalad run`.

Use those as tested inputs, not as the repository organization. The demo omits the outer BIDS Study description and its project-root-as-analysis pattern depends on open [BABS PR #369](https://github.com/PennLINC/babs/pull/369), so it must be verified against the exact pinned BABS version. Our architecture instead separates campaign state from the Study composition and installs the finalized output dataset into the derivative location by DataLad identity and commit. Replace the demo’s mutable image tags, handwritten temporary YAML, host `.env` dependence, disabled setup check, and unpinned tool installation before using any of its mechanics scientifically.

[`mechababs@4a4deb8`](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09) contributes promising operational patterns: an independent campaign DataLad dataset, separate dataset/pipeline/cluster configuration axes, pinned BABS and container inputs, an attempt ledger, clean-pin guards, and reconciled one-step state transitions. Its Study composition, output layout, container integration, and provenance closure remain under development. If it is adopted, place its campaign state under `operations/<campaign>/` and compose its finalized derivative datasets into `studies/` or `results/`; do not make its current hierarchy the project architecture. The organizational decisions in this document are ours and are established before scientific code migration. Its current role and promotion gates are recorded in the [MechaBABS decision](decisions/mechababs-role.md).

DataLad Containers is not the general-purpose image builder. When construction is required, use Apptainer/Singularity; use `datalad containers-add` to register an exact completed SIF and `datalad containers-run` for every direct project-authored scientific execution so the image is an explicit run input. BABS is the current execution exception: its released job template may record a generated wrapper or direct Singularity command instead. Austin Macdonald’s unmerged BABS work would make those records more explicit, but it is not required for this conversion.

The released BABS wrapper and zipped outputs introduce undesirable indirection. Accept that as a BABS-specific limitation, retain the state needed to interpret and test a representative pilot, and do not repeat the wrapper/archive pattern in project-authored steps.

## Protected ABCD boundary

- Put the ABCD study and every ABCD derivative on access-controlled DataLad siblings.
- Track DUC/DUA metadata and a non-sensitive retrieval adapter in the public superdataset only if permitted.
- Keep participant lists, scheduler logs containing identifiers, extracted subject-level morphometrics, QC images, and failure tables private until the DUC is reviewed for each artifact class.
- Publish aggregate statistics or figures only after disclosure review.
- Test public publication from a credential-free environment and verify that no annex URL grants indirect access to protected content.

Uncertainty about the old NIH derivatives is not a blocker. Do not retrieve them until the DUC question is resolved; rebuild the derivatives from declared inputs and tracked images instead.

## Licensing boundary

Use REUSE-style file-level licensing with a root `REUSE.toml` and license texts under `LICENSES/`. The [REUSE specification](https://reuse.software/spec/) provides machine-readable file-to-license association.

Maintain separate records for:

- analysis code and documentation;
- imported code or configuration;
- raw datasets and DUCs;
- derivative datasets;
- models/weights used by `recon-any` or `recon-all-clinical`;
- SIF redistribution terms and, when one exists, separate OCI artifact terms;
- software license texts and operational license files such as the FreeSurfer key;
- credentials and secrets, which are not license material.

Distribute required license texts and operational license files with the repository or image when their terms and privacy properties permit it. When redistribution is prohibited or inappropriate, document acquisition and verification and provide the operational file through a declared runtime bind. Never embed or publish credentials or secrets.

License compatibility and data-access compatibility are release gates, not README footnotes.
