# Poster-derived result targets

## Purpose

The [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf) is the authoritative description of the result set to reconstruct. Historic branches, transient cluster logs, and the original `balanced_scans.csv` are not required. The goal is to produce scientifically comparable figures and statistics through a fully tracked pipeline, not to recover bit-identical poster artwork.

The poster was visually inspected as a rendered page, not inferred only from PDF text extraction.

## Scientific question

Test whether deep-learning FreeSurfer tools intended for clinical scans show age-dependent bias in cortical morphometry, and whether the FreeSurfer 8 default `recon-all` agrees with the FreeSurfer 7 baseline in adolescents.

## Reference design represented by the poster

- Cohort: 1,000 unrelated ABCD participants, evenly sampled from four waves.
- Reference: FreeSurfer 7 `recon-all` using high-resolution T1w and T2w inputs.
- Compared tools: `recon-all-clinical` and `recon-any`.
- Compared input conditions: original high-resolution T1w and T1w resampled to 1×1×5 mm.
- Primary global summaries: cortical gray-matter volume and cerebellar white-matter volume.
- Developmental analysis: associate age with percent difference from the reference, then relate region-wise age-bias magnitude to published regional age at peak gray-matter volume.
- Version comparison: FreeSurfer 7 versus FreeSurfer 8 `recon-all` on adolescent data.

The original `balanced_scans.csv` is intentionally out of scope for the first implementation. The ABCD phase must eventually replace it with a tracked, executable cohort-building command whose inputs identify the allowed ABCD release/query.

## Authoritative output families

### 1. Tool agreement and age-residual panels

For each of `recon-all-clinical` and `recon-any`, and for native and 1×1×5 mm T1w inputs, produce:

- scatterplots against the FreeSurfer 7 reference for cortical gray-matter volume;
- scatterplots against the reference for cerebellar white-matter volume;
- an agreement statistic such as R² for each measure;
- percent difference from the reference plotted against participant age;
- a tracked table containing the values shown in every panel.

The poster’s approximate R² values are reference observations, not pass/fail thresholds:

| Comparison | Cortical GMV | Cerebellar WMV |
|---|---:|---:|
| `recon-all-clinical`, native T1w | 0.95 | 0.98 |
| `recon-all-clinical`, 1×1×5 mm T1w | 0.93 | 0.97 |
| `recon-any`, native T1w | 0.93 | 0.97 |
| `recon-any`, 1×1×5 mm T1w | 0.87 | 0.95 |

Acceptance is based on reproducing the comparisons, measures, directions, and visible age-related trend under the newly declared cohort and exact runtimes. Material disagreement must be reported, not tuned away.

### 2. Regional developmental-bias analysis

Produce:

- a cortical surface map of published regional age at peak gray-matter volume;
- a cortical surface map of the fitted regional age-bias coefficient;
- a scatterplot relating those quantities across regions;
- a table with atlas, hemisphere, region, peak-age value, model coefficient, uncertainty, inclusion status, and source citation.

The poster reports an anti-correlation of approximately `r = -0.39`, `p = 0.02`. Treat this as a diagnostic reference. The reconstructed result must use a predeclared atlas, model, exclusion rule, multiple-testing policy where relevant, and pinned external milestone table.

### 3. FreeSurfer 7 versus 8 agreement

Produce scatterplots and age-residual plots comparing:

- FreeSurfer 7.4.1 `recon-all`; and
- FreeSurfer 8.2.1 `recon-all`.

The poster shows near-perfect agreement for the two global measures (`R² ≈ 1.00`) and no comparable age trend. Reproduce the analysis rather than hard-coding that conclusion.

### 4. Cohort and completeness accounting

Produce machine-readable tables for:

- candidate, included, excluded, failed, retried, and complete participants/sessions;
- exclusion and failure reasons by campaign;
- input acquisition type and resolution;
- pipeline/runtime identity;
- metric extraction completeness;
- the exact rows entering each statistic and figure.

These tables replace notebook-only counts and make differences between the original poster cohort and the reconstruction interpretable.

## Pilot versus claim-bearing analysis

`ds007116` (Penn LEAD) is the initial open BIDS pilot. Process all eligible scans rather than reconstructing `balanced_scans.csv`. Use it to prove dataset retrieval, BABS campaign generation, container execution, extraction, modeling interfaces, figure generation, and replay.

The pilot does not need to reproduce the ABCD effect sizes and must not be used to assert the poster’s population-level claims. ABCD remains the claim-bearing phase because its age sampling and scale match the poster design.

## Result manifest

Every release candidate must contain a result manifest with one row per authoritative figure, panel, table, or statistic. At minimum record:

```text
result_id
study_dataset_commit
cohort_manifest_key
derivative_dataset_commit(s)
container_name
sif_annex_key
sif_sha256
code_commit
config_path_and_hash
datalad_run_commit
output_path
validation_status
```

The manifest, not the poster PDF, becomes the authoritative map from claims to reconstructable artifacts.

## Completion criterion

The reconstruction is scientifically complete when a clean installation can retrieve all permitted inputs, replay the declared steps with the tracked SIF images, and regenerate panels that address the four output families above. Exact historical numerical agreement is not required; all deviations must be attributable to explicit differences in cohort, runtime, input availability, or model specification.
