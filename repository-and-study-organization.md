# Generic repository and BIDS Study organization

## Purpose

Use this organization for an ideal-oriented STAMPED neuroimaging research object that executes BIDS Apps through BABS. It defines the boundaries among the research object, BIDS Study datasets, raw datasets, derivative attempts, authored code, BABS-generated code, containers, and campaign state.

This is an opinionated implementation for a reusable skill. Replace placeholders such as `<study>`, `<pipeline>`, and `<variant>` with names meaningful to the analysis at hand. Do not inherit operation names, dependency names, cohorts, pipelines, or versions from another project.

## Decision

- Make the research-object root a DataLad superdataset, not a BIDS dataset.
- Create one BIDS 1.11.1 Study dataset for each independently governed source study or cohort.
- Put the raw BIDS/DataLad dataset at `<study>/sourcedata/raw/`.
- Put each BABS project directly under the corresponding Study's `derivatives/` directory.
- Use BABS Study-layout support with `analysis_path: "."` so the BABS project root is also its DataLad analysis dataset. Put BABS RIA stores and initialization state under `.babs/`.
- Pin the exact bleeding-edge BABS commit that contains the Study-layout fix. The reviewed minimum is [`2cc536a`](https://github.com/PennLINC/babs/commit/2cc536a51282124f3811ffa971f82a7c34116af5), which merged [BABS #369](https://github.com/PennLINC/babs/pull/369). A later commit or release may be used only after recording and qualifying its exact identity.
- Do not recreate the old BABS `analysis/` layout and then build a project-local promotion or copying system around it. BABS 0.5.4 does not provide the required direct layout and is not the implementation path prescribed here.
- Treat a BABS project as a provisional derivative while it is running. Call it an accepted BIDS derivative only after merge, provenance-captured finalization, metadata completion, and independent validation.

The direct layout is important because the BABS run history and the final derivative retain one DataLad identity. If the pinned upstream implementation cannot satisfy the boundary or validation rules, stop and resolve the requirement in BABS or in this skill. Do not silently introduce a second derivative dataset, an untracked copy, or a filesystem-symlink workaround.

## Evidence and authority

| Source | Authority in this organization |
|---|---|
| [BIDS 1.11.1 Study dataset](https://github.com/bids-standard/bids-specification/blob/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec/src/common-principles.md#L432-L477) | Defines the scientific Study, raw, and derivative structure |
| [BABS](https://pennlinc-babs.readthedocs.io/en/stable/overview.html) | Executes participant/session BIDS App jobs, records DataLad provenance, and merges results |
| [`babs_demo@7d46f37`](https://github.com/djarecka/babs_demo/tree/7d46f3763cbf8f76b17e5f345160a07d042446e8) | Demonstrates `analysis_path: "."`, hidden RIA paths, Study composition, and provenance-captured extraction |
| [`mechababs@4a4deb8`](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09) | Supplies campaign-axis, pinning, attempt, state, and inclusion-accounting patterns |

BIDS determines what the scientific datasets mean. BABS determines how participant/session execution is performed. `babs_demo` is an implementation prototype rather than a complete normative Study. Mechababs informs campaign organization but does not define the BIDS boundary.

## Identify BIDS and DataLad boundaries precisely

A BIDS Study is itself a BIDS dataset. Its root `dataset_description.json` sets `DatasetType` to `"study"`. It has no `sub-*` directories at its root. It composes source material, a nested raw BIDS dataset, and derivatives:

```text
<study>/
├── dataset_description.json       # DatasetType: "study"; BIDSVersion: "1.11.1"
├── README.md
├── code/                          # optional Study-local index/configuration
├── sourcedata/
│   ├── raw/                       # independent raw BIDS/DataLad dataset
│   │   ├── dataset_description.json
│   │   ├── README.md
│   │   └── sub-*/
│   └── <original-source>/         # optional non-BIDS source material
└── derivatives/
    └── <pipeline>-<variant>-attempt-<N>/
```

Apply these distinctions:

- Only `sourcedata/raw/` is the raw BIDS dataset in this Study convention. Original DICOMs or other pre-BIDS sources are siblings of `raw/`, not children of it.
- Give the Study, raw dataset, and each accepted derivative their own `dataset_description.json`, README, DataLad dataset ID, commit, retrieval location, and validation result.
- BIDS requires `GeneratedBy` for a derivative and recommends `SourceDatasets`. This implementation additionally requires explicit `DatasetType: "derivative"`, exact source dataset versions, and independent validation before acceptance.
- Use placement-independent source dataset URLs and exact versions or commits in derivative metadata. Never record a host-local path.
- A DataLad dataset and a BIDS dataset are different concepts. This implementation deliberately makes every Study, raw dataset, BABS derivative attempt, and accepted derivative both, but an ordinary directory is neither merely because of its path.
- BIDS permits a heterogeneous mixture below `derivatives/`. A running, failed, or incomplete BABS attempt may remain there without being represented as an accepted derivative.

The outer Study is the only object labeled `DatasetType: "study"`. A campaign, BABS project, derivative attempt, RIA store, cluster workspace, or multi-study research-object root is not another BIDS Study.

## Organize the research object

```text
<research-object>/                         # DataLad superdataset; NOT BIDS
├── .datalad/
├── pyproject.toml
├── src/<package>/                         # canonical authored library code
├── apps/<app-name>/                       # canonical authored BIDS App entrypoints
├── tests/
├── notebooks/                             # optional and non-authoritative
├── envs/
│   ├── pixi.toml
│   ├── pixi.lock
│   └── containers/
│       ├── repronim/                      # pinned source container dataset
│       ├── custom/                        # custom definitions only when required
│       └── accepted/                      # registered exact SIF dataset
├── config/
│   ├── datasets/                          # dataset identity, access, cohort policy
│   ├── pipelines/                         # app, SIF, scientific arguments, outputs
│   ├── clusters/                          # scheduler resources, binds, site preamble
│   └── campaigns/                         # prospective axis compositions
├── studies/
│   └── <study>/                           # independent BIDS Study/DataLad dataset
├── operations/
│   └── <campaign>/                        # campaign state and meta-provenance dataset
├── results/
│   └── <analysis>-<variant>/              # standalone multi-study derivative dataset
├── docs/
└── LICENSES/
```

The research-object commit composes exact code, configuration, Study, campaign, container, and result dataset commits. Do not label it a BIDS “super-study.”

Use separate Study datasets when raw inputs have different governance, access rules, participant universes, release identities, or publication boundaries. A result that truly combines more than one Study belongs in a standalone derivative dataset under `results/`, with every exact source identified.

## Put each kind of code in one unambiguous place

| Location | Contents | Authority |
|---|---|---|
| Root `src/`, `apps/`, `tests/`, and `pyproject.toml` | Human-authored libraries, BIDS Apps, tests, and package metadata | Canonical scientific implementation |
| `config/datasets/` | Dataset URL/ID/commit, access class, cohort and selection policy | Canonical dataset intent |
| `config/pipelines/` | Exact app/SIF identity, scientific options, input/output contract | Canonical pipeline intent |
| `config/clusters/` | Scheduler resources, scratch, binds, and site preamble | Canonical host policy; never scientific parameters |
| Study `code/` | Small Study-local index or genuinely Study-specific preparation configuration | Optional BIDS content, not a duplicated source tree |
| Raw dataset `code/` | Conversion, deidentification, or curation code owned by that raw dataset | Raw-dataset authority |
| BABS project `code/` | Generated participant scripts, expanded config, inclusion tables, submission/check templates, and job-state files | Operational execution evidence |
| Operations dataset `code/babs/` | Exact BABS source checkout when qualifying an unreleased commit | Orchestration-tool identity, not analysis code |
| Registered SIF | Executable application and runtime dependencies | Runtime identity used by BABS/DataLad |

Do not edit BABS-generated `code/` to implement science. Change the canonical app or configuration, build and register a new SIF if required, and create a new attempt.

DataLad run records live in the BABS project's Git history. The generated scripts and expanded configuration make those historical records interpretable, but they do not replace the authored application source.

Do not duplicate the canonical source tree inside every Study. Use exact `GeneratedBy.CodeURL`, container metadata, and parent-superdataset composition. When distributing a Study independently, install an immutable code dataset or export a source snapshot under Study `code/`; never symlink to another checkout.

## Compose campaigns without redefining Studies

Adopt the mechababs pattern that a campaign composes three independent axes:

- **dataset**: exact raw dataset identity, version, access class, and inclusion policy;
- **pipeline**: exact registered SIF, BIDS App arguments, scientific configuration, and expected outputs; and
- **cluster**: scheduler resources, scratch, binds, and host preamble.

Keep prospective templates under root `config/`. Resolve their exact commits, hashes, SIF identity, tool versions, and access class into an immutable campaign snapshot under `operations/<campaign>/`.

```text
operations/<campaign>/
├── campaign.yaml                    # resolved dataset + pipeline + cluster snapshot
├── desired-state.tsv
├── observed-state.tsv
├── desired-inclusion.tsv
├── accepted-attempts.tsv            # scientific identity -> exact attempt dataset/commit
├── code/
│   └── babs/                        # pinned bleeding-edge BABS checkout when used
└── attempts/
    ├── attempt-001.yaml
    ├── attempt-001-commands.jsonl
    └── attempt-001-requested-inclusion.csv
```

The operations dataset records BABS lifecycle/meta-provenance: initialization, setup checks, submission, status inspection, retry decisions, merge, finalization, validation, and acceptance. It points to the BABS project under the Study; it does not contain or replace that derivative dataset.

Preserve the exact CSV passed through `babs init --list-sub-file` separately from BABS's realized `code/processing_inclusion.csv`. Their difference records initialization-time eligibility or required-input availability. Record runtime failures separately from BABS status, RIA, log, and output-completeness evidence.

Give every attempt a stable identity and never overwrite or repurpose it. Attempt state evolves while a run is active; close and preserve its history when it finishes. A retry creates a new attempt. A change to a scientific input, SIF, or pipeline configuration creates a new scientific variant or campaign rather than masquerading as a retry.

Use desired and observed state separately. Advance campaigns through small, recorded transitions derived from BABS/RIA evidence. This is a pattern credited to mechababs, not a claim that its current implementation can reconstruct every missing or corrupt ledger.

## Use the direct BABS derivative layout

Initialize each attempt directly at:

```text
studies/<study>/derivatives/<pipeline>-<variant>-attempt-<N>/
```

Use the upstream BABS Study-layout configuration pattern:

```yaml
analysis_path: "."
input_ria_path: ".babs/input_ria"
output_ria_path: ".babs/output_ria"

input_datasets:
  BIDS:
    origin_url: <placement-independent-raw-dataset-url>
    path_in_babs: sourcedata/raw
```

The resulting boundary is:

```text
<pipeline>-<variant>-attempt-<N>/      # BABS project + DataLad analysis dataset
├── .babs/                             # BABS init state and RIA stores; operational
├── code/                              # BABS-generated execution material
├── containers/                        # BABS-installed registered container dataset
├── logs/                              # operational logs
├── sourcedata/raw/                    # exact raw BIDS/DataLad input
├── dataset_description.json           # completed before derivative acceptance
└── sub-*/ or other declared outputs   # final scientific payload
```

The direct root is initially a provisional derivative. Run BABS initialization, setup checks, participant/session execution, status handling, and merge against that same dataset identity. Record archive extraction or other finalization with DataLad in that dataset; do not copy the payload into a second dataset.

An unmodified BABS root currently includes a visible top-level `containers/` directory, while the [BIDS 1.11.1 derivative directory schema](https://github.com/bids-standard/bids-specification/blob/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec/src/schema/rules/directories.yaml#L106-L175) does not list it. Treat validation of the exact upstream layout as a hard acceptance gate. If upstream BABS must change how this content is represented, address that upstream or revise this skill explicitly. Do not hide the mismatch with an undocumented ignore, copying layer, or alternate authoritative dataset.

Accept an attempt only when:

1. the exact BABS commit containing Study-layout support is recorded and clean-pin checks prove it ran;
2. parent DataLad registration and a clean recursive clone preserve the BABS project;
3. input and output RIAs remain retrievable after parent registration;
4. `babs check-setup --job-test`, submit, status, retry, and merge succeed as applicable;
5. finalization is provenance-captured in the same DataLad dataset;
6. the exact registered SIF, raw input commit, scientific configuration, requested inclusion, and realized inclusion are recoverable;
7. BABS administrative paths are distinguished from the declared scientific payload;
8. the derivative has complete `dataset_description.json`, `GeneratedBy`, and `SourceDatasets` metadata;
9. the derivative and its containing Study validate independently against BIDS 1.11.1 or carry an explicitly reviewed upstream exception; and
10. representative DataLad historical replay succeeds from a clean clone.

Record acceptance in `operations/<campaign>/accepted-attempts.tsv`. Scientific consumers use only the exact accepted attempt dataset and commit. Failed or superseded attempts remain preserved but are not authoritative inputs.

## Use `babs_demo` and mechababs appropriately

`babs_demo` supports the direct-layout decision by demonstrating [`analysis_path: "."` and hidden `.babs/` paths](https://github.com/djarecka/babs_demo/blob/7d46f3763cbf8f76b17e5f345160a07d042446e8/babs_walkthrough_bids_layout.sh#L34-L38) plus [provenance-captured archive extraction](https://github.com/djarecka/babs_demo/blob/7d46f3763cbf8f76b17e5f345160a07d042446e8/babs_walkthrough_merge_bids_layout.sh#L38-L55). Supply the Study metadata and validation policy that its demonstration script does not provide.

Use mechababs's dataset/pipeline/cluster separation, pinned code, attempt identities, clean-pin guard, incremental transitions, and requested-versus-realized inclusion accounting. Do not install or invoke mechababs on the authority of this document. When it is mature for the required datasets and pipelines, it may replace manual campaign control without changing the BIDS Study, code, or derivative boundaries.

## Invariants

1. The research-object root is a DataLad superdataset, not a BIDS Study.
2. Each independently governed raw study or cohort has its own BIDS Study dataset.
3. The Study root is the only object labeled `DatasetType: "study"`.
4. BABS receives the raw BIDS child, never the outer Study root.
5. Every BABS attempt has its own direct derivative path and DataLad identity.
6. The BABS project, DataLad analysis dataset, and provisional derivative are the same root when using `analysis_path: "."`.
7. Canonical authored code is separate from BABS-generated `code/`.
8. Dataset, pipeline, cluster, campaign, and attempt identities remain separate.
9. Retries never overwrite attempts; scientific changes never masquerade as retries.
10. Only accepted attempt commits may feed later scientific steps or releases.
11. No copying, symlink, or secondary derivative dataset substitutes for the direct BABS layout.
12. A future mechababs adoption changes campaign automation, not scientific dataset boundaries.

## Pinned references

- BIDS 1.11.1: [`f7a04bc`](https://github.com/bids-standard/bids-specification/tree/f7a04bc28cbd0b4f0f0bd9f9409f4762f2459cec)
- BABS Study-layout implementation: [`2cc536a`](https://github.com/PennLINC/babs/tree/2cc536a51282124f3811ffa971f82a7c34116af5)
- `babs_demo`: [`7d46f37`](https://github.com/djarecka/babs_demo/tree/7d46f3763cbf8f76b17e5f345160a07d042446e8)
- mechababs `v0.0.1`: [`4a4deb8`](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09)
