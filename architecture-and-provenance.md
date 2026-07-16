# Architecture and provenance for the converted analysis

## Outcome

The target is one top-level DataLad research object named `stamped_dl_morphometrics_biases`. It composes code, environment definitions, registered SIF images, two BIDS Study datasets, multiple independent derivatives, downstream analysis, and result manifests without placing controlled ABCD content in the public code history.

This document records responsibility boundaries and identities. The [conversion plan](conversion-plan.md) records sequencing.

The study layout follows the released [BIDS 1.11.1 Common Principles: Study dataset](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) convention. Each study records its BIDS version so that a future specification change does not silently change the repository structure.

## Responsibility map

| Layer | What it controls | What it does not control |
|---|---|---|
| `pyproject.toml` | Python package metadata and console entry points | Conda system dependencies or production runtime identity |
| Pixi workspace | Named local environments, locked conda-forge/PyPI resolution used during SIF construction, and actionable task entry points | Scientific provenance, image construction, or runtime identity |
| Apptainer/Singularity definition and build | Direct construction of a Linux SIF from tracked and checksummed inputs | Registration, data provenance, or proof that a result used the image |
| SIF image | Immutable runtime executed on Slurm | Raw-data or result provenance by itself |
| DataLad/git-annex and DataLad Containers | Component identity and availability, SIF registration/retrieval, direct container execution, run records, and historical state | General-purpose image building or subject/session scheduling |
| BIDS Study layout | Domain-valid composition of source, raw BIDS, and derivative dataset roots | Version control by itself |
| BABS | Participant/session expansion, Slurm job lifecycle, auditing, merge | Transparent provenance for its own lifecycle commands |
| Operations ledger | Exact BABS lifecycle invocations and campaign state transitions | Scientific participant execution, which remains in BABS/DataLad run records |

## Target layout

```text
stamped_dl_morphometrics_biases/
├── .datalad/
├── pyproject.toml
├── pixi.toml -> envs/pixi.toml
├── pixi.lock -> envs/pixi.lock
├── .pixi -> envs/.pixi
├── src/dl_morphometrics_biases/
│   ├── cohort.py
│   ├── extract.py
│   ├── model.py
│   ├── figures.py
│   ├── validate.py
│   └── cli.py
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
│   │   ├── freesurfer-7.4.1/
│   │   ├── freesurfer-8.2.1/
│   │   ├── recon-any-<version>/
│   │   ├── recon-all-clinical-<version>/
│   │   └── analysis-<version>/
│   └── container-dataset/        # DataLad subdataset with annexed SIFs
├── studies/
│   ├── ds007116/                 # DataLad study subdataset
│   │   ├── dataset_description.json  # DatasetType: study; BIDSVersion: 1.11.1
│   │   ├── README
│   │   ├── sourcedata/raw/            # independent raw BIDS/DataLad root
│   │   └── derivatives/<campaign>/    # independent derivative/DataLad root
│   └── abcd/                     # access-controlled DataLad study subdataset
│       ├── dataset_description.json  # DatasetType: study; BIDSVersion: 1.11.1
│       ├── README
│       ├── sourcedata/raw/            # independent protected raw BIDS/DataLad root
│       └── derivatives/<campaign>/    # independent protected derivative/DataLad root
├── operations/<campaign>/
│   ├── campaign.yaml
│   ├── runbook.md
│   ├── commands/                 # reviewed literal BABS lifecycle commands
│   ├── commands.jsonl
│   ├── generated/
│   └── audit/
├── outputs/                       # independent downstream result DataLad dataset
│   ├── extracted/
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

Each outer study, its raw BIDS child, and every derivative is a distinct DataLad dataset root with its own dataset identity and `dataset_description.json`. The outer description sets `DatasetType` to `"study"`. Each derivative records `GeneratedBy` and, where applicable, a `SourceDatasets` reference to its immediate source, such as `../../sourcedata/raw/`. Validate the raw child and each claimed BIDS derivative independently.

The precise superdataset/subdataset installation mechanics should be tested against BABS before they are frozen. The invariants are that raw inputs never own their derivatives, every campaign derivative remains independently versioned, and BABS receives the installed raw BIDS child rather than the outer Study dataset. Keep BABS administrative state distinguishable from scientific derivative content even when the released BABS layout places both under `derivatives/`.

## Environment layout

The real Pixi files live under `envs/`, with tracked root symlinks for conventional discovery:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi
```

Root `pyproject.toml` contains no Pixi environment tables. This is a project policy, not a limitation of Pixi’s `pyproject.toml` support: `pixi.toml` keeps non-Python dependencies and multiple operational environments from overwhelming Python package metadata.

Track `envs/pixi.toml` and `envs/pixi.lock`. The lock supports local realization and supplies the reviewed Linux package solution installed during SIF construction; it does not identify the completed image. Ignore `.venv/` and the realized contents of `envs/.pixi/`. Define independently solved environments for at least:

- BABS/DataLad orchestration;
- development and tests;
- legacy extraction if `pandas<2` remains necessary;
- current downstream analysis;
- named image environments where processing runtimes have different dependency graphs.

Do not use solve groups to erase a real incompatibility. The pandas conflict in the current repository is a reason for distinct environments.

## Runtime construction and identity

```text
tracked Apptainer definition + pixi manifest/lock
    + checksummed non-Pixi inputs + recorded builder/architecture
                              │
                              ▼
                    direct SIF construction
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
- Apptainer definition path and Git commit;
- manifest and lock hashes;
- Pixi and Apptainer/Singularity versions;
- builder environment, including Lima configuration when used, and target architecture;
- checksums or immutable references for the base and every non-Pixi build input;
- the exact build command;
- SBOM identity;
- SIF signing identity, verification policy, and bundle or native signature reference;
- SIF SHA-256, size, annex key, DataLad dataset ID/commit, and persistent retrieval locations;
- required bind mounts, distributed or externally supplied license files, environment variables, and smoke tests;
- source and completed OCI digests only when an OCI artifact is intentionally consumed or published.

Build the SIF directly from the tracked definition. Apptainer/Singularity is the builder; Pixi resolves and installs the selected locked environment during the build; DataLad Containers registers, retrieves, and executes the completed image. A Pixi task may wrap those explicit operations but does not become the builder or image identity.

On macOS, run the build inside Linux with Lima; on Apple Silicon, use an x86_64 guest when the Slurm target is x86_64. A separate OCI application image is optional. If the definition consumes an OCI base, pin it by digest. Build and publish a completed OCI application image only for a documented consumer, and identify it separately from the authoritative SIF. The exact SIF may also be distributed through an ORAS-capable registry, but its git-annex key and DataLad commit remain its research-object identity.

Sign and verify the SIF with a tracked Sigstore blob bundle and/or Apptainer’s native SIF signing. Sign an OCI artifact separately when one exists. See [Sigstore verification](https://docs.sigstore.dev/cosign/verifying/verify/) and [Apptainer SIF verification](https://apptainer.org/docs/user/main/signNverify.html).

### Candidate and authoritative images

Unregistered candidate SIFs may be used locally or on Slurm to debug installation, architecture, bind mounts, licenses, scheduler integration, or resource requests. Write their outputs only to ignored scratch or another quarantined location; never merge, retain, compare scientifically, or cite them. If a debug observation influences a parameter, inclusion decision, or model choice, record that decision and confirm it with an authoritative run.

Before retaining any output, hash and smoke-test the exact SIF, register it with `datalad containers-add`, publish its annex content to declared storage, point BABS or `datalad containers-run` to the registered image, and regenerate the output. Registration after a debug run cannot retroactively make that earlier output authoritative. The full gate is recorded in the [candidate versus authoritative SIF decision](decisions/runtime-image-strictness.md).

## Campaign matrix

Use one campaign identity per dataset × pipeline/runtime × input condition. Do not overload a directory with outputs from different tool versions or acquisition transformations.

| Dataset | Campaign family | Initial runtime | Input condition | Phase |
|---|---|---|---|---|
| ds007116 | `reconall-fs7` | FreeSurfer 7.4.1 | available native structural inputs | pilot first |
| ds007116 | `reconall-fs8` | FreeSurfer 8.2.1 | matching native inputs | pilot |
| ds007116 | `recon-any` | version to pin | native T1w; then 1×1×5 mm where valid | pilot |
| ds007116 | `recon-all-clinical` | version to pin | native T1w; then 1×1×5 mm where valid | pilot |
| ABCD | same four families | exact versions above | poster-declared native/resampled conditions | protected scale-up |

`ds007116` is the public Penn LEAD BIDS dataset and has a DataLad-backed OpenNeuro repository. Pin a released snapshot or commit rather than its moving default branch. OpenNeuro exposes snapshot Git hashes and DataLad download instructions in its [user guide](https://docs.openneuro.org/user_guide.html), and the dataset is available in the [OpenNeuro DataLad collection](https://github.com/OpenNeuroDatasets/ds007116).

Use stable campaign names such as:

```text
ds007116__reconall-fs7.4.1__native-t1t2__v1
abcd__recon-any-<version>__t1w-1x1x5__v1
```

The campaign record must contain the exact dataset commit, container key, BABS config hash, processing level, input filters, resources, exclusion policy, and expected output pattern.

## Three types of provenance

### Scientific execution provenance

This is authoritative. Use BABS-generated DataLad run records for participant/session processing and explicit `datalad containers-run` records for project-authored extraction, cohort construction, statistics, and figures. Each record must resolve to exact inputs, code/config, outputs, and registered SIF content.

A parameterized Pixi task may launch either command path. For a project-authored scientific step, put DataLad inside the task and pass the explicit scientific program and critical arguments after `--`, so the durable record contains the resolved command rather than a task name:

```text
allowed:    pixi task -> datalad containers-run -> explicit scientific command
prohibited: datalad containers-run -> pixi task -> hidden scientific command
prohibited: pixi task -> scientific command without DataLad provenance
```

Do not use Pixi task dependencies, input/output fingerprints, or skip decisions as the scientific workflow. DataLad and BABS own scientific provenance and replay. See the [Pixi tasks and provenance decision](decisions/pixi-tasks-and-provenance.md).

### Lifecycle or meta-provenance

`babs init`, setup checks, `submit`, `status`, retry, and `merge` affect orchestration but are not all captured as scientific DataLad run records. Expose their exact parameterized forms as Pixi tasks or tracked command templates, document repository and campaign setup in the README and `runbook.md`, and do not hide substantive arguments in user shell variables.

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

The Pixi tasks or command templates provide prospective actionability; the JSONL ledger records what actually happened. Store generated BABS scripts and configuration snapshots beside both. A Pixi task may execute a state-changing lifecycle command, but the expanded argv and resulting state transition must be entered in the ledger and committed with the change it explains. Neither the task nor the ledger replaces participant-level BABS/DataLad run records.

### Development history

Git commits, pull requests, tests, notebooks, and Pixi task definitions explain how code was developed and make operations discoverable. A task definition is not evidence that it ran, and these records do not substitute for scientific provenance or the BABS operations ledger.

## Current BABS compatibility position

BABS is the chosen Slurm layer. Its current release uses DataLad and the FAIRly big pattern, creates participant/session result branches, stores zipped results, and merges successful outputs. The [BABS documentation](https://pennlinc-babs.readthedocs.io/en/) describes DataLad datasets for BIDS inputs and containers as required inputs.

The local `babs_demo` proves a useful BIDS study pattern:

- `sourcedata/raw` is a DataLad subdataset;
- a SIF is registered with `datalad containers-add` in a container dataset;
- the BABS project is placed under `derivatives/`;
- merged zip extraction is itself recorded with `datalad run`.

Retain that pattern, but add the BIDS 1.11.1 Study `dataset_description.json`, independent DataLad roots and derivative metadata described above. Replace the demo’s mutable image tags, handwritten temporary YAML, host `.env` dependence, disabled setup check, and unpinned tool installation before using it scientifically.

DataLad Containers is not the general-purpose image builder. Use Apptainer/Singularity to construct the SIF, `datalad containers-add` to register it, and `datalad containers-run` for every direct project-authored scientific execution so the image is an explicit run input. BABS is the current execution exception: its released job template may record a generated wrapper or direct Singularity command instead. Austin Macdonald’s unmerged BABS work would make those records more explicit, but it is not required for this conversion.

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
