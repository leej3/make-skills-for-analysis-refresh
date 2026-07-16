# Architecture and provenance for the converted analysis

## Outcome

The target is one top-level DataLad research object named `stamped_dl_morphometrics_biases`. It composes code, environment definitions, container images, two BIDS studies, multiple independent derivatives, downstream analysis, and result manifests without placing controlled ABCD content in the public code history.

This document records responsibility boundaries and identities. The [conversion plan](conversion-plan.md) records sequencing.

## Responsibility map

| Layer | What it controls | What it does not control |
|---|---|---|
| `pyproject.toml` | Python package metadata and console entry points | Conda system dependencies or production runtime identity |
| Pixi workspace | Named local environments, conda-forge/PyPI resolution, BABS/dev/test tooling | Scientific workflow provenance, caching, or HPC runtime identity |
| OCI build | Auditable construction from a pinned base and reviewed lock | The exact bytes of a later SIF conversion |
| SIF image | Immutable runtime executed on Slurm | Raw-data or result provenance by itself |
| DataLad/git-annex | Component identity, availability, run records, historical state | Subject/session scheduling |
| BIDS study layout | Composition of source and derivative datasets | Version control by itself |
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
│   │   ├── sourcedata/raw/
│   │   └── derivatives/<campaign>/
│   └── abcd/                     # access-controlled DataLad study subdataset
│       ├── sourcedata/raw/
│       └── derivatives/<campaign>/
├── operations/<campaign>/
│   ├── campaign.yaml
│   ├── runbook.md
│   ├── commands/                 # reviewed literal BABS lifecycle commands
│   ├── commands.jsonl
│   ├── generated/
│   └── audit/
├── outputs/
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

The precise superdataset/subdataset arrangement should be tested against BABS before it is frozen. The invariant is that source datasets and every derivative remain independently versioned and are composed from above; raw inputs never own their derivatives.

## Environment layout

The real Pixi files live under `envs/`, with tracked root symlinks for conventional discovery:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi
```

Root `pyproject.toml` contains no Pixi environment tables. This is a project policy, not a limitation of Pixi’s `pyproject.toml` support: `pixi.toml` keeps non-Python dependencies and multiple operational environments from overwhelming Python package metadata.

Track `envs/pixi.toml` and `envs/pixi.lock`. Ignore `.venv/` and the realized contents of `envs/.pixi/`. Define independently solved environments for at least:

- BABS/DataLad orchestration;
- development and tests;
- legacy extraction if `pandas<2` remains necessary;
- current downstream analysis;
- each container build only where its dependency graph differs.

Do not use solve groups to erase a real incompatibility. The pandas conflict in the current repository is a reason for distinct environments.

## Runtime identity chain

```text
tracked recipe + pixi manifest + pixi lock + pinned builder
                         │
                         ▼
                 OCI manifest digest
                         │  signed / attested
                         ▼
        pinned OCI-to-SIF conversion operation
                         │
                         ▼
      SIF SHA-256 + git-annex key + DataLad commit
                         │
                         ▼
      DataLad/BABS run record + derivative commit
```

The image ledger under `envs/images.lock.yaml` should record, for every runtime:

- logical image name and scientific tool version;
- source recipe path and Git commit;
- manifest and lock hashes;
- Pixi and builder versions;
- target platform;
- base-image and completed OCI digests;
- registry repository and immutable digest reference;
- Sigstore signing identity, issuer, verification policy, and bundle/attestation references;
- SBOM identity;
- OCI-to-SIF converter and exact command;
- SIF SHA-256, size, annex key, DataLad dataset ID/commit, and persistent retrieval locations;
- required bind mounts, license files, environment variables, and smoke test.

Pin OCI references by digest. Tags may remain as navigation labels, but neither a tag nor an OCI digest is the SIF identity. Apptainer documents that converting the same Docker/OCI image twice can produce non-identical SIF files because conversion metadata varies; pulling an already stored SIF through `oras://` preserves the exact file. See the official [OCI support](https://apptainer.org/docs/user/latest/docker_and_oci.html) and [registry](https://apptainer.org/docs/user/main/registry.html) guidance.

Sign the OCI image by digest with Sigstore and verify an explicit certificate identity and issuer; Sigstore signatures bind the image digest by default. Sign the SIF as a blob with a tracked Sigstore bundle and/or use Apptainer’s native SIF signing. See [Sigstore verification](https://docs.sigstore.dev/cosign/verifying/verify/) and [Apptainer SIF verification](https://apptainer.org/docs/user/main/signNverify.html).

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

This is authoritative. Use BABS-generated DataLad run records for participant/session processing and explicit `datalad containers-run` records for extraction, cohort construction, statistics, and figures. Each record must resolve to exact inputs, code/config, outputs, and SIF content.

### Lifecycle or meta-provenance

`babs init`, `submit`, `status`, retry, and `merge` affect orchestration but are not all captured as scientific DataLad run records. Store the intended, reviewable commands under `operations/<campaign>/commands/` and explain their sequence in `runbook.md`. Keep commands literal and parameterized only by tracked campaign configuration; do not hide their substantive arguments in user shell variables.

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

The command files provide prospective actionability; the JSONL ledger records what actually happened. Store generated BABS scripts and configuration snapshots beside both. A read-only Pixi task may display status or print a command already stored here; it must not execute lifecycle commands that can change a campaign.

### Development history

Git commits, pull requests, tests, notebooks, and Pixi tasks explain how code was developed. They do not substitute for either scientific provenance or the operations ledger.

## Current BABS compatibility position

BABS is the chosen Slurm layer. Its current release uses DataLad and the FAIRly big pattern, creates participant/session result branches, stores zipped results, and merges successful outputs. The [BABS documentation](https://pennlinc-babs.readthedocs.io/en/) describes DataLad datasets for BIDS inputs and containers as required inputs.

The local `babs_demo` proves a useful BIDS study pattern:

- `sourcedata/raw` is a DataLad subdataset;
- a SIF is registered with `datalad containers-add` in a container dataset;
- the BABS project is placed under `derivatives/`;
- merged zip extraction is itself recorded with `datalad run`.

Retain that pattern, but replace the demo’s mutable image tags, handwritten temporary YAML, host `.env` dependence, disabled setup check, and unpinned tool installation before using it scientifically.

The `datalad-container` extension and `datalad containers-run` are related but not synonymous. Registering and tracking SIFs uses the extension even if BABS’s current job template records a wrapper/direct Singularity command. For direct downstream steps, prefer `containers-run`, which records the image as an input dependency. Austin Macdonald’s unmerged BABS work would make BABS records more explicit, but it is not required for this conversion.

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
- OCI/SIF redistribution terms;
- the FreeSurfer license file, which must not be committed or embedded and is supplied by runtime bind mount.

License compatibility and data-access compatibility are release gates, not README footnotes.
