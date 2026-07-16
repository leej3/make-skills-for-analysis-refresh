# Repository and BIDS Study organization

## Decision

`stamped_dl_morphometrics_biases` will be a DataLad research superdataset, not a BIDS dataset. It will compose two independent BIDS 1.11.1 Study datasets—one for `ds007116` and one for ABCD—together with the authored analysis code, registered containers, BABS campaign state, and any cross-study results.

Each scientific Study will contain its raw BIDS dataset at `sourcedata/raw/` and its accepted scientific derivatives under `derivatives/`. A BABS project, campaign, attempt, RIA store, or scheduler workspace is operational infrastructure; none becomes a BIDS Study merely because it contains BIDS data or sits below a directory named `derivatives`.

For the initial implementation, each campaign will be an independent DataLad dataset containing stable, never-reused attempt identities whose histories are preserved. An attempt changes while BABS is running and is closed rather than overwritten when it finishes. The BABS analysis created within an attempt remains its own DataLad dataset. After BABS merges an attempt, an explicit provenance-captured finalization step will create or update an independently described BIDS derivative dataset, and the Study will install that exact DataLad dataset and commit under `derivatives/`. This keeps BABS administration out of the accepted derivative view while preserving BABS's run history and merged archive as upstream evidence.

BABS now has a plausible path to remove this finalization seam. Support for `analysis_path: "."` and hidden RIA paths was merged into BABS `main` in [commit `2cc536a`](https://github.com/PennLINC/babs/commit/2cc536a51282124f3811ffa971f82a7c34116af5) on 2026-07-16, and its [new configuration example](https://github.com/PennLINC/babs/blob/2cc536a51282124f3811ffa971f82a7c34116af5/notebooks/eg_freesurferpost-0-1-2_bids-study-layout.yaml#L14-L34) demonstrates the intended paths. The change is not in the 0.5.4 release. We may adopt a direct BABS-project-as-derivative layout after the qualification gate below passes. Until then, the separate operational-project and accepted-derivative boundaries are authoritative.

This document is standalone. Where it differs from an earlier proposed directory tree, use this document for repository, Study, campaign, and code placement.

## Evidence used for the decision

The organization is based on four sources, with different authority:

| Source | Role in this decision |
|---|---|
| [BIDS 1.11.1 Study dataset](https://github.com/bids-standard/bids-specification/blob/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec/src/common-principles.md#L432-L477) | Normative scientific dataset organization |
| [BABS](https://pennlinc-babs.readthedocs.io/en/stable/overview.html) | Execution engine for participant/session fan-out, Slurm, DataLad run provenance, and merge |
| [`babs_demo@7d46f37`](https://github.com/djarecka/babs_demo/tree/7d46f3763cbf8f76b17e5f345160a07d042446e8) | Executable prototype of BABS operating against a Study-like layout |
| [`mechababs@4a4deb8`](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09) | Design source for campaign composition, pinning, attempts, incremental state transitions, and inclusion accounting |

BIDS determines what a Study and derivative mean. BABS determines the operational shape required by the current execution engine. `babs_demo` demonstrates mechanics but is not a normative or complete BIDS example. Mechababs supplies valuable design patterns, but its own [output-structure document](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/docs/output_structure.md#L1-L49) calls the published shape a target and leaves final composition into scientific studies to the consuming project.

## What a BIDS 1.11.1 Study is

A Study is itself a BIDS dataset describing the organization of one scientific study. Its root `dataset_description.json` must set `DatasetType` to `"study"`. It contains no `sub-*` directories at its root. Instead, it composes source material, one nested raw BIDS dataset, and derivative datasets:

```text
<study>/
├── dataset_description.json       # DatasetType: "study"; BIDSVersion: "1.11.1"
├── README.md
├── code/                          # optional Study-local code/configuration
├── sourcedata/
│   ├── raw/                       # a distinct raw BIDS dataset root
│   │   ├── dataset_description.json
│   │   ├── README.md
│   │   └── sub-*/
│   └── <original-source>/         # optional non-BIDS source material
└── derivatives/
    └── <pipeline-name>-<variant>/ # a distinct derivative BIDS dataset root
        ├── dataset_description.json
        ├── README.md
        └── sub-*/                 # when participant-level output exists
```

The important boundaries are:

- Only `sourcedata/raw/` is the raw BIDS dataset in the Study convention. DICOMs or other original sources are siblings of `raw/`, not children of it.
- `sourcedata/raw/` has its own `dataset_description.json`, README, subject/session structure, BIDS validation result, DataLad dataset ID, and pinned commit.
- A child of `derivatives/` is not automatically a BIDS derivative. BIDS requires `GeneratedBy` for a derivative and recommends `SourceDatasets`; this project additionally requires an explicit `DatasetType: "derivative"`, independent validation, and a pinned DataLad identity before acceptance. BIDS explicitly permits non-compliant material below `derivatives/`, but this project will not label such material an accepted scientific derivative.
- BIDS's Study example recommends `../../sourcedata/raw/` for a derivative that exists only at its canonical Study path. This project may stage and install the same derivative dataset at more than one path, so its `SourceDatasets` instead uses the raw DataLad dataset's placement-independent retrieval URL and exact commit or snapshot in `Version`. Never record a host-local path. As an additional project provenance rule, a downstream derivative identifies its actual immediate derivative inputs as well, so the chain is not flattened to raw data alone.
- BIDS and DataLad dataset boundaries are related but not synonymous. In this project, every Study, raw dataset, and accepted derivative will also be an independent DataLad dataset. An ordinary directory does not become either kind of dataset merely because of its path.

A minimal Study description is:

```json
{
  "Name": "DL morphometrics biases — ds007116",
  "BIDSVersion": "1.11.1",
  "DatasetType": "study"
}
```

The BIDS 1.11.1 schema permits `code/`, `docs/`, `logs/`, `sourcedata/`, and `derivatives/` at a Study root. This is useful, but BIDS does not require code duplication or prescribe Git/DataLad repository boundaries. Its [Code section](https://github.com/bids-standard/bids-specification/blob/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec/src/modality-agnostic-files/code.md#L1-L10) only says that scripts used to prepare a dataset may be stored under `code/`.

## Research-object layout

```text
stamped_dl_morphometrics_biases/            # DataLad superdataset; NOT BIDS
├── .datalad/
├── pyproject.toml
├── src/dl_morphometrics_biases/            # canonical authored library code
├── apps/<app-name>/                        # canonical authored BIDS App entrypoints
├── tests/                                  # unit, schema, fixture, and replay tests
├── notebooks/                              # optional, non-authoritative views only
├── envs/
│   ├── pixi.toml
│   ├── pixi.lock
│   └── containers/
│       ├── repronim/                       # pinned source container dataset
│       ├── custom/                         # tracked custom definitions when required
│       └── accepted/                       # registered exact SIF dataset
├── config/
│   ├── datasets/                           # dataset identity, access, cohort policy
│   ├── pipelines/                          # app, SIF, scientific arguments, outputs
│   ├── clusters/                           # Slurm resources, binds, site preamble
│   └── campaigns/                          # authored prospective compositions/templates
├── studies/
│   ├── ds007116/                           # public BIDS Study/DataLad dataset
│   └── abcd/                               # protected BIDS Study/DataLad dataset
├── operations/
│   └── <dataset>__<pipeline>__<condition>/ # independent campaign DataLad dataset
├── results/
│   └── <analysis>-<variant>/               # standalone cross-study derivative dataset
├── docs/
└── LICENSES/
```

The research-object commit pins the authored code/configuration and the installed Study, container, campaign, and result dataset commits together. It is the composition layer. It is not made into an invented BIDS “super-study.”

Use one Study per source cohort because that is the most literal application of the single-raw-dataset convention in BIDS 1.11.1 and it keeps ABCD's access controls independent from the public pilot:

```text
studies/ds007116/
├── dataset_description.json               # DatasetType: study
├── README.md
├── code/README.md                          # explains code identities; no duplicate source
├── sourcedata/raw/                         # pinned OpenNeuro/DataLad dataset
└── derivatives/
    ├── freesurfer-7.4.1-native/
    ├── freesurfer-8.2.0-native/
    ├── recon-any-<version>-native/
    ├── recon-all-clinical-<version>-native/
    └── morphometrics-<version>/

studies/abcd/                               # same semantic shape, protected storage
├── dataset_description.json               # DatasetType: study
├── README.md
├── code/README.md
├── sourcedata/raw/                         # protected raw BIDS/DataLad dataset
└── derivatives/                            # protected derivative DataLad datasets
    └── <same qualified pipeline families and input variants>/
```

An input condition that changes the scientific input, such as native versus 1×1×5 mm resampling, receives a separate derivative identity. A retry that changes no scientific input is an operational attempt, not a new scientific variant. Only accepted outputs are installed in the canonical Study path. Failed and superseded attempts remain under `operations/`.

Before ABCD access is available, its URL, DUC-controlled retrieval procedure, and expected dataset identity belong under `config/datasets/`; a retrieval adapter is not mislabeled as a raw BIDS dataset. The `studies/abcd/sourcedata/raw/` slot is populated only by an actual protected raw BIDS dataset, which may remain uninstalled in checkouts lacking authorization.

Products derived entirely within one cohort belong in that Study's `derivatives/`. A result that actually combines exact inputs from both `ds007116` and ABCD belongs in a standalone derivative dataset under `results/`, with every source dataset and commit identified. It is not forced into either source Study.

## Where code belongs

The word “code” refers to several different artifacts in this stack. They must not be conflated:

| Location | Contents | Authority |
|---|---|---|
| `src/`, `apps/`, `tests/`, and root `pyproject.toml` | Human-authored library, BIDS Apps, tests, and package metadata | Canonical scientific implementation |
| `config/datasets/` | Dataset URL/ID/commit, access class, selection and cohort policy | Canonical dataset intent |
| `config/pipelines/` | Exact app/container identity, scientific options, input/output contract | Canonical pipeline intent |
| `config/clusters/` | Scheduler resources, scratch, binds, and site preamble | Canonical host policy; never scientific parameters |
| `studies/<study>/code/` | A small Study-local index or genuinely Study-specific preparation code/configuration | Optional BIDS content, not a duplicate implementation tree |
| `sourcedata/raw/code/` | Conversion, deidentification, or curation code owned by the raw dataset | Upstream raw-dataset authority |
| `derivatives/<name>/code/` | Realized scientific configuration, identity manifests, or a standalone release snapshot when needed | Evidence for that derivative; exact source still identified by commit |
| BABS analysis dataset `code/` | BABS-generated participant scripts, expanded config, inclusion tables, submission/check templates, and job-state files | Operational execution evidence, not authored application source |
| `operations/<campaign>/code/babs/` | Optional pinned BABS source checkout when an unreleased commit is qualified | Exact orchestration-tool source for that campaign |
| Registered SIF | Executable app and runtime dependencies | Runtime actually used; identified by DataLad commit, annex key, and checksum |

Do not edit BABS-generated `code/` to implement scientific logic. Change the canonical app/config, build and register a new SIF when required, and create a new campaign attempt.

DataLad run records live in the BABS analysis dataset's Git history rather than being authored files in `code/`. The generated scripts and configuration make those historical records interpretable.

The Study does not duplicate the research-object source tree by default. Its `code/README.md`, derivative `GeneratedBy.CodeURL`, container metadata, and parent DataLad commit identify the exact source and runtime. If a Study must be distributed independently of the research object, install an immutable standalone code dataset or export a source snapshot under its `code/`; do not use a filesystem symlink back to the research-object root.

## Campaign and attempt organization

Adopt mechababs's strongest organizational idea: a run is the explicit composition of independent dataset, pipeline, and cluster axes, as described in its [campaign model](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/README.md#L12-L67). Do not put Slurm policy in a pipeline file or scientific arguments in a cluster profile.

```text
operations/ds007116__freesurfer-7.4.1__native/
├── .gitignore                            # enforces RIA/scratch exclusions
├── campaign.yaml                         # immutable realized/resolved campaign snapshot
├── desired-state.tsv                     # intended cells and accepted transitions
├── observed-state.tsv                    # state reconstructed from BABS/RIA evidence
├── desired-inclusion.tsv                  # reviewed campaign processing universe
├── accepted-derivatives.tsv               # attempt -> derivative ID/commit -> Study path
├── code/
│   └── babs/                             # only when pinning an unreleased checkout
├── attempts/
│   ├── attempt-001/
│   │   ├── attempt.yaml                  # exact inputs, SIF, configs, host, start state
│   │   ├── requested-inclusion.csv       # exact --list-sub-file passed to BABS
│   │   ├── commands.jsonl                # expanded lifecycle commands and outcomes
│   │   └── babs-project/                 # BABS-owned project and RIA layout
│   └── attempt-002/
└── products/
    └── <derivative>__attempt-001/        # independent candidate derivative dataset
```

The campaign root is the DataLad dataset. `attempt-001/` and `attempt-002/` are tracked directories with stable identities, not additional DataLad datasets. Their state evolves during execution, but completed attempts are preserved and never repurposed. The nested BABS `analysis/` and every candidate under `products/` are independent DataLad subdatasets.

BABS RIA stores are operational storage owned by BABS, not campaign Git content. Prefer locating them outside the campaign worktree when the pinned BABS version supports configurable paths. With BABS 0.5.4's project-local stores, commit campaign-root ignore rules for `attempts/*/babs-project/input_ria/` and `output_ria/`, never recursively save those directories, and test after initialization that the campaign has no untracked RIA objects. Record their exact URLs and storage policy in `attempt.yaml`.

`config/campaigns/<name>.yaml` is the human-authored prospective composition or template. Campaign initialization resolves every dataset URL/commit, pipeline/config hash, SIF identity, cluster profile, tool version, and access class into `operations/<campaign>/campaign.yaml`. The realized file is immutable for that campaign. Changing an identity creates a new campaign or attempt according to whether the scientific identity changed; it does not rewrite the historical snapshot.

For released BABS 0.5.4, the BABS-owned project has its normal operational structure:

```text
babs-project/
├── analysis/                              # DataLad analysis dataset
│   ├── code/                              # generated by BABS
│   ├── containers/                        # cloned registered-container dataset
│   ├── inputs/data/<input-name>/          # cloned BIDS input dataset
│   ├── logs/
│   └── <participant-result>.zip
├── input_ria/
└── output_ria/
```

The exact BABS version, project code commit when applicable, Pixi lock, input dataset commits, registered SIF identity, BABS configuration, requested resources, and lifecycle commands belong in the campaign/attempt record. Preserve requested inclusion separately from BABS's realized `processing_inclusion.csv`. Their difference records initialization-time eligibility or required-input availability; runtime failures require separate BABS status, RIA, log, and output-completeness evidence.

Use explicit attempt identities and never overwrite a failed attempt. Advance campaign state through small, recorded transitions such as initialize, check, submit, inspect, retry, merge, finalize, validate, and accept. The ledger is a view of evidence, not a substitute for BABS/DataLad run records.

## From BABS output to an accepted Study derivative

The initial production path is:

```text
Study raw BIDS dataset + exact registered SIF + reviewed configuration
    -> BABS project in operations/<campaign>/attempts/<attempt>/
    -> participant DataLad run branches and annexed archives in output RIA
    -> babs merge
    -> merged BABS output dataset retained as upstream evidence
    -> candidate derivative dataset at operations/<campaign>/products/<name>__<attempt>/
    -> explicit datalad containers-run finalization/materialization in that dataset
    -> independently described and validated candidate derivative
    -> accepted-derivatives.tsv acceptance record
    -> exact derivative dataset/commit installed in Study derivatives/
```

BABS receives `studies/<study>/sourcedata/raw/`, not the Study root. Create the candidate as an independent DataLad dataset under the campaign's `products/` directory. Install the exact merged BABS output dataset and commit as an input subdataset at `sourcedata/babs-output/`, and install the exact registered-container dataset and commit at `code/containers/`. Use persistent, placement-independent clone URLs for both subdatasets rather than paths relative to the campaign checkout. Their annex payloads may remain absent until required. Run `datalad containers-run -d <candidate>` in the candidate—not in the merged BABS analysis dataset—and declare the input subdataset, tracked configuration, and registered materialization SIF. These candidate-local DataLad dependencies ensure the historical command does not rely on an untracked sibling or absolute host path. The operation extracts or transforms only the promised scientific payload, writes derivative metadata and completeness records, and excludes BABS RIA stores, scheduler state, incidental logs, and unrelated container checkouts from the accepted derivative.

Acceptance appends a row to `accepted-derivatives.tsv` containing the campaign and attempt, derivative DataLad dataset ID and commit, validation-report identity, intended Study path, and acceptance event commit. The Study then installs that same dataset and commit at the stable scientific path. Because the derivative metadata identifies source datasets by placement-independent URL and version rather than a path relative to the campaign checkout, the identical commit remains valid in both installations. No untracked copy or filesystem symlink connects the campaign and Study, and the campaign product and Study installation do not create competing scientific identities.

## Qualification gate for direct BABS Study-layout use

The current `babs_demo` demonstrates the intended mechanics: `sourcedata/raw/`, a BABS project under `derivatives/`, [`analysis_path: "."` with hidden `.babs/` RIA paths](https://github.com/djarecka/babs_demo/blob/7d46f3763cbf8f76b17e5f345160a07d042446e8/babs_walkthrough_bids_layout.sh#L34-L38), and [provenance-captured archive extraction](https://github.com/djarecka/babs_demo/blob/7d46f3763cbf8f76b17e5f345160a07d042446e8/babs_walkthrough_merge_bids_layout.sh#L38-L55). Its [preparation script](https://github.com/djarecka/babs_demo/blob/7d46f3763cbf8f76b17e5f345160a07d042446e8/babs_walkthrough_prepare_bids_layout.sh#L53-L85), however, does not create the required Study-root description or recommended README. Its issue reproducer also shows that using a nested analysis path rather than `.` changes DataLad parent-registration behavior; `analysis_path: "."` is therefore a structural requirement, not cosmetic configuration.

The BABS change that enables this layout is newly merged and unreleased. Moreover, an unmodified BABS analysis root contains a visible top-level `containers/` directory, while the [BIDS 1.11.1 derivative directory schema](https://github.com/bids-standard/bids-specification/blob/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec/src/schema/rules/directories.yaml#L106-L175) permits `code/`, `docs/`, `derivatives/`, `logs/`, `phenotype/`, `sourcedata/`, `stimuli/`, participant, and template directories—not `containers/`. A direct BABS project may be a useful provisional work dataset, but it must not be called an accepted BIDS derivative merely because it occupies a derivative path.

Allow a BABS project to become the derivative dataset directly only after a toy campaign demonstrates all of the following with the exact pinned BABS commit or a release containing the change:

1. `babs init` with `analysis_path: "."` and hidden project-local RIA paths;
2. parent DataLad subdataset registration followed by a clean recursive clone;
3. successful fresh clones from both input and output RIAs after parent registration;
4. `babs check-setup --job-test`, submit, status, retry, and merge;
5. provenance-captured archive extraction or optional unzipped output;
6. preservation of the exact participant command, registered SIF identity, and requested/realized inclusion;
7. removal, relocation, or reviewed handling of non-BIDS top-level BABS machinery in the accepted view;
8. independent BIDS 1.11.1 validation of the Study, raw dataset, and final derivative;
9. historical DataLad rerun/replay from a clean clone; and
10. no loss of campaign/attempt auditability when the derivative is installed at its stable scientific path.

If this gate passes, the direct layout is preferable because the BABS run history and final derivative can share one DataLad identity. It may replace the separate finalization dataset for later campaigns. The qualification decision must be recorded before changing the production boundary; a successful toy run is not applied retroactively to earlier attempts.

## What is adopted from mechababs

Austin's mechababs design gets several important things right, and the manual BABS organization deliberately preserves them:

- dataset, pipeline, and cluster are independent configuration axes composed into a campaign;
- each campaign is a versioned DataLad object with exact tool and container identities;
- attempts are explicit and never silently overwritten;
- a clean-pin guard prevents the wrong BABS/tool environment from receiving credit;
- desired and observed state are kept distinct and advanced through small, resumable transitions derived from BABS/RIA evidence;
- requested inclusion is retained separately from BABS's realized intersection; and
- development and production use the same paths, differing through tracked configuration and content.

The desired/observed formulation above is the pattern this project will implement, not a claim that mechababs can reconstruct an arbitrary blank or corrupt ledger from RIA state today. Its current reconciler queries BABS for active cells and advances partially recorded state; full recovery remains an implementation responsibility here.

Mechababs is not the execution dependency yet. At the reviewed `v0.0.1` snapshot, its published output shape is still a target, direct ReproNim container integration uses a manual shim, selection is specialized for its current OpenNeuro MRIQC/fMRIPrep work, and scientific Study composition is explicitly project-specific. Those are reasonable seams in an actively developing tool, but they are not yet the place to delegate this analysis's four reconstruction families, protected ABCD inputs, or final Study organization.

The organization intentionally leaves a future adoption seam: a mature mechababs can replace the manual campaign transition and ledger implementation while retaining the same dataset/pipeline/cluster configs, attempt identities, BABS projects, Study roots, and accepted derivative datasets. Adopting it should not require moving scientific data or redefining what a Study is.

## Organizational invariants

1. There is one BIDS Study per source cohort: `ds007116` and ABCD.
2. The research-object root and campaign datasets are not labeled BIDS Studies.
3. BABS always receives the raw BIDS child, never the outer Study root.
4. Canonical authored code is separate from BABS-generated `code/`.
5. Every accepted derivative is independently described, validated, and pinned by DataLad identity and commit.
6. A directory below `derivatives/` is not called BIDS-compliant until it passes the derivative acceptance checks.
7. Dataset, pipeline, cluster, campaign, and attempt identities remain separate.
8. Retries create new attempts; scientific input/runtime changes create new pipeline or variant identities.
9. Public `ds007116` and protected ABCD data, logs, derivatives, and storage siblings remain separable.
10. No untracked copies or filesystem symlinks substitute for DataLad composition.
11. The direct BABS Study-layout path remains conditional until its complete qualification gate passes.
12. A future mechababs adoption changes campaign automation, not the BIDS scientific organization.

## Pinned source snapshots

- BIDS 1.11.1 tag commit: [`f7a04bc`](https://github.com/bids-standard/bids-specification/tree/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec)
- BABS 0.5.4 released source: [`89c93b2`](https://github.com/PennLINC/babs/tree/89c93b2e53877f8c64b7716c05688748e7881260)
- BABS Study-layout merge: [`2cc536a`](https://github.com/PennLINC/babs/tree/2cc536a51282124f3811ffa971f82a7c34116af5)
- `babs_demo`: [`7d46f37`](https://github.com/djarecka/babs_demo/tree/7d46f3763cbf8f76b17e5f345160a07d042446e8)
- mechababs `v0.0.1`: [`4a4deb8`](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09)
