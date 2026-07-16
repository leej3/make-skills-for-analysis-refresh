# Conversion plan for `stamped_dl_morphometrics_biases`

## Objective

Create a new DataLad research object, `stamped_dl_morphometrics_biases`, that reconstructs the analyses summarized in the [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf). Begin with all eligible scans in the open `ds007116` BIDS dataset, then scale the validated pattern to controlled ABCD data.

The new repository will not try to sanitize the existing `dl_morphometrics_biases` history in place. It will import reviewed scientific content with explicit provenance, leaving the old repository as historical evidence. The repository-local [STAMPED neuroimaging skill](skills/stamped-neuroimaging-analysis/SKILL.md) must be copied into the new repository at its first commit and treated as project policy.

The implementation is governed by the accepted decisions on the [role of Pixi](decisions/pixi-role.md), [Pixi tasks and provenance](decisions/pixi-tasks-and-provenance.md), and [candidate versus authoritative SIFs](decisions/runtime-image-strictness.md). Revise those records before changing the corresponding policy in this plan.

## Fixed decisions

- Use a BIDS study/superdataset pattern adapted from the local `babs_demo`.
- Pin [BIDS 1.11.1 Study datasets](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) and represent each `ds007116` and ABCD study accordingly: the outer dataset has `DatasetType: "study"`, while the raw BIDS input and each BIDS derivative are independently described and validated datasets composed as DataLad subdatasets.
- Use DataLad/git-annex for data and SIF identity, availability, provenance, and replay.
- Use BABS for participant/session expansion, Slurm submission, audit/retry, and merge.
- Target generic Slurm rather than NIH Swarm or one cluster. Keep host profiles separate.
- Use Pixi for named local development, BABS/tooling, test, and analysis environments and to resolve/install locked Linux dependencies during SIF construction. Pixi is not the image builder.
- Store the real Pixi manifest and lock under `envs/`; use root symlinks for discovery.
- Use exact, registered SIF files for every retained, merged, scientifically compared, or released HPC result. Candidate SIFs may be used for quarantined local or Slurm debugging, but their outputs are disposable and must be regenerated after image registration.
- Build SIFs directly from tracked Apptainer/Singularity definitions by default. A separate OCI application image is conditional on a concrete Docker/OCI consumer; consuming a digest-pinned OCI base does not itself create that requirement.
- Do not use environment packing as an image or HPC strategy.
- Permit Pixi tasks to expose BABS lifecycle commands, explicit image operations, and parameterized scientific entry points that invoke `datalad containers-run`. Never place a Pixi task inside `datalad run`/`containers-run`, never let a task run a scientific program without DataLad provenance, and never use Pixi task dependencies or caching as the scientific workflow.
- Accept current BABS wrapper and zip indirection, document it, and test replay. Do not depend on Austin Macdonald’s unreleased BABS branch.
- Treat `ds007116` as an engineering/scientific pilot and ABCD as the later claim-bearing analysis.
- Ignore the old `balanced_scans.csv` until ABCD access exists; then implement a tracked cohort builder from the permitted release/query.
- Do not retrieve old NIH cluster derivatives until their DUC status is resolved. They are not needed to begin.

## Migration strategy

Use gated phases. A phase is complete only when its evidence is committed to the new DataLad dataset. Do not start a large campaign while a smaller equivalent gate is failing.

## Phase 0 — preserve evidence and declare the target

### Work

1. Create a read-only inventory of the old repository at its current commit.
2. Record the source commit for each imported module/script, even though it is not the result-generating historical commit.
3. Copy the poster into a documentation/reference dataset or register a persistent reference if redistribution is not allowed.
4. Copy [poster-derived result targets](result-targets.md) into the new repository and assign stable result IDs.
5. Record known versions:
   - FreeSurfer `recon-all` 7.4.1;
   - FreeSurfer `recon-all` 8.2.1;
   - exact `recon-any` distribution/build: unresolved gate;
   - exact `recon-all-clinical` distribution/build: unresolved gate.
6. Record the intended native and 1×1×5 mm input conditions without copying notebook-generated command files.

### Evidence

- `docs/source-inventory.tsv`
- `docs/result-targets.md`
- `docs/decisions/`, including the copied accepted decision records
- initial result-manifest schema

### Exit gate

Every intended panel in the poster has an ID, expected inputs, expected outputs, and a declared status of pilot-only or claim-bearing.

## Phase 1 — create the new STAMPED/DataLad skeleton

### Work

1. Create the top-level DataLad dataset with a text-friendly configuration.
2. Add the repository-local agent skill and a one-line `AGENTS.md` pointer requiring it for analysis work.
3. Establish directories from [architecture and provenance](architecture-and-provenance.md) and record BIDS 1.11.1 as the pinned specification.
4. Give each `studies/<dataset>/` outer dataset a `dataset_description.json` with `DatasetType: "study"` and a study README. Install `sourcedata/raw/` as an independent raw BIDS/DataLad dataset, and create each `derivatives/<pipeline>-<variant>/` as an independent derivative BIDS/DataLad dataset with its own README and `dataset_description.json`.
5. Add `GeneratedBy` and immediate `SourceDatasets` metadata to every claimed BIDS derivative. Treat metadata inheritance and BIDS validation as local to each dataset root; do not label BABS administrative state as a BIDS derivative.
6. Add Git attributes so large binary outputs and SIFs are annexed rather than committed to Git.
7. Add public/private sibling policy before any ABCD content is installed.
8. Add root `REUSE.toml` and the required SPDX license texts. Assign initial file groups:
   - code;
   - documentation;
   - configurations;
   - generated metadata;
   - externally governed data and models.
9. Add a machine-readable component inventory containing dataset IDs, URLs, licenses/DUCs, BIDS dataset descriptions, and distribution status.

### Evidence

- clean recursive clone of the empty skeleton;
- `datalad subdatasets` and `git annex info` output saved as a release check;
- BIDS validation or reviewed expected exceptions for the empty Study dataset skeleton and every installed independent BIDS child;
- `reuse lint` passing for project-owned files;
- a credential-free publication dry run that cannot reach future protected siblings.

### Exit gate

The empty research object can be cloned and its Study/raw/derivative boundaries, BIDS metadata responsibilities, licenses, and access policy are understandable without local paths.

## Phase 2 — refactor the Python package and remove notebook authority

### Work

Retain the useful concepts from `dl_morphometrics_helpers` and `fsstats_extraction.py`, but do not copy notebook execution state as the new pipeline.

Create small modules and console entry points for:

1. `cohort` — enumerate eligible BIDS scans and later construct the ABCD cohort;
2. `resample` — create the declared 1×1×5 mm T1w condition with explicit interpolation and metadata;
3. `extract` — read FreeSurfer statistics into stable BIDS-derivative tables;
4. `assemble` — combine pipeline tables using explicit subject/session keys and completeness rules;
5. `model` — fit global and regional age-bias models from tracked tables;
6. `figures` — generate only declared figure IDs from declared model outputs;
7. `validate` — check schemas, counts, numeric ranges, container/dataset identities, and result manifests.

Specific cleanup:

- Replace hard-coded `/data/ABCD_*` and `/home/.../.cache/templateflow` paths with CLI/config inputs and tracked TemplateFlow retrieval.
- Remove inline `uv` script metadata and UV installation instructions.
- Replace pickle caches whose key depends only on path strings. Any performance cache must include content identities and remain outside provenance; scientific tables remain declared outputs.
- Replace `OUT_SUBDIR = "leej3/fs_stats_multi"` and path-parent inference with explicit output arguments.
- Remove shell-driven `ls`, interactive `freeview`, and scheduler submission from authoritative code.
- Make missing/incomplete participant policy explicit and emit a structured exclusions table.
- Make atlases, metrics, hemispheres, global measures, transformations, statistical models, and seeds configuration fields.
- Pin TemplateFlow resources by content and record their licenses.
- Add tests using a tiny synthetic/redistributable FreeSurfer-like fixture.

Notebooks may remain under `notebooks/` only if they:

- consume tracked, already-generated tables;
- do not submit jobs or write authoritative derivatives;
- contain no unique analysis logic;
- can be deleted without preventing reconstruction;
- are stripped and tested, or are rendered explanatory documents whose source is tracked.

### Evidence

- unit tests for path handling, extraction, joins, exclusions, model formulas, and figure metadata;
- integration test from fixture FreeSurfer stats to one figure;
- `rg` check showing no institutional absolute paths in code/config;
- console `--help` output for every scientific boundary;
- notebook inventory marking each as remove, reduce, or retain.

### Exit gate

Every authoritative result has a non-interactive CLI path and no result depends on notebook cell order or user shell state.

## Phase 3 — establish Pixi environments

### Work

1. Keep root `pyproject.toml` for the installable package and console entry points.
2. Create `envs/pixi.toml`, `envs/pixi.lock`, and tracked root symlinks.
3. Declare `linux-64` for runtime-dependency environments and the relevant macOS platform(s) for developer environments.
4. Use distinct named environments:
   - `dev` for lint, tests, documentation, and local notebooks;
   - `analysis` for current extraction/model/figure development;
   - `extract-legacy` only while `freesurfer-stats` requires `pandas<2`;
   - `babs` for pinned BABS, DataLad, datalad-container, and supporting tools;
   - runtime-specific Linux environments whose locked dependencies are installed during SIF construction;
   - an image-authoring environment for Apptainer/Lima, signing, and SBOM tools only when useful.
5. Pin the Pixi version range supported by the workspace and record the exact Pixi, Apptainer/Singularity, and Lima versions/configuration used for release image construction.
6. Remove `.python-version` if it misleadingly implies one Python for all environments; otherwise document it as an editor hint only.
7. Ignore `.venv/` and realized `envs/.pixi/`. Do not bind either into production containers or Slurm jobs.
8. Add Pixi tasks for lint, unit tests, documentation, `reuse lint`, environment reports, explicit image build/register/verify operations, and parameterized BABS lifecycle commands such as `init`, `check-setup`, `submit`, status/retry, and merge. Their definitions make operations actionable; they are not evidence that an invocation occurred.
9. Permit a result-changing task only when its body invokes `datalad containers-run` with declared inputs, outputs, the registered SIF, and the explicit scientific executable and arguments after `--`. Prohibit the reverse direction (`datalad ... -- pixi run <task>`) because it leaves only task indirection in the run record.
10. Do not use Pixi task dependencies, task caching, or skip decisions to compose or replay scientific stages. Record every actual BABS state-changing invocation in `operations/<campaign>/commands.jsonl` with expanded argv, versions, status, scheduler identifiers, and before/after commits.

### Dependency update cycle

1. Create a branch dedicated to the environment change.
2. Edit `envs/pixi.toml` directly.
3. Run `pixi lock` intentionally and inspect `git diff -- envs/pixi.toml envs/pixi.lock`.
4. Realize and test with `--locked` on macOS where relevant and install the reviewed `linux-64` solution in a fresh target-architecture Linux environment.
5. If the change can affect a runtime, build a new SIF directly from its tracked definition; never mutate an existing image identity. Build a separate OCI application image only when its documented consumer requires one.
6. Commit manifest, lock, tests, and image-ledger changes together with the reason for the change.
7. Leave a stable analysis lock unchanged unless a dependency change is required. CI should reject manifest/lock mismatch through locked realization; it should not update dependencies for freshness.

### Exit gate

A fresh macOS checkout can realize developer environments, a Linux image build can install the reviewed lock with `--locked`, and neither route silently edits `pixi.lock`.

## Phase 4 — construct, sign, and register SIF runtimes

### Image families

Create independent image definitions for:

- FreeSurfer 7.4.1 `recon-all`;
- FreeSurfer 8.2.1 `recon-all`;
- a pinned `recon-any` release/build and weights;
- a pinned `recon-all-clinical` release/build and weights;
- downstream extraction/statistics/figure analysis.

An image may cover multiple commands only when its license, dependencies, update cadence, and scientific role make that boundary defensible. Do not combine all tools merely to reduce image count.

### Build path

1. Use tracked Apptainer/Singularity definitions under `envs/containers/<image>/`.
2. Pin or checksum the base and every non-Pixi build input. A definition may bootstrap from a digest-pinned OCI base, a pinned SIF, or another supported source; this does not require building or publishing an OCI application image.
3. Copy `envs/pixi.toml` and `envs/pixi.lock` into the definition and install the selected Linux environment at its final in-image path with a pinned Pixi executable and `pixi install --locked`.
4. Record activation explicitly and do not copy a developer `.venv` or realized `envs/.pixi/` prefix into the image. Pixi is the dependency resolver/installer within this step; Apptainer/Singularity is the image builder.
5. Build the immutable SIF directly for `linux/amd64` unless a target Slurm system requires another architecture. On macOS, use Apptainer inside Lima; on Apple Silicon, use an x86_64 Linux guest when the Slurm target is x86_64. Record the Lima template/configuration, guest architecture, Pixi version, Apptainer version, build command, and definition/lock hashes.
6. Run smoke tests that execute the actual tool entry point and a tiny fixture, not only `--version`, then test the completed SIF on a representative Slurm host before declaring that runtime supported.
7. Generate an SBOM and build attestation. Sign and verify the final SIF with an appropriate SIF signature and/or a detached Sigstore bundle; compute the authoritative checksum after any embedded signature changes the SIF bytes.
8. Add the completed local SIF to the DataLad container dataset with `datalad containers-add`, publish its annex content to persistent storage, and record its SHA-256, annex key, DataLad commit, verification policy, and retrieval locations in `envs/images.lock.yaml`.
9. Use `datalad containers-run` to execute direct scientific steps. DataLad Containers registers, retrieves, and executes completed images; it does not replace the tracked Apptainer definition build. A Pixi task may wrap the explicit build, signing, registration, or verification command for convenience.
10. Optionally publish the same SIF through an ORAS-capable registry as a secondary location. Produce and sign a separate OCI application image only if an identified Docker/OCI consumer, layer-cache workflow, or OCI-native release requires it, and record that artifact independently from the SIF.

### Candidate and authoritative images

Candidate SIFs may be built and used locally or on Slurm to debug installation, architecture, bind mounts, licenses, resources, and scheduler integration. Put their outputs in ignored scratch space or a disposable campaign that cannot be merged, scientifically compared, or enter a result manifest.

When a candidate is ready, stop the debug campaign, smoke-test and hash its exact bytes, register it with `datalad containers-add`, publish the annex content, and point a BABS or `datalad containers-run` execution at the registered image. Regenerate every output intended for retention. Registration after a debug run does not retroactively make that output authoritative. Follow the full [candidate versus authoritative SIF decision](decisions/runtime-image-strictness.md).

### Licensing

- Determine and record the redistribution terms for each required license file, model weight, installer, and tool before publishing an image.
- Distribute required license files with the repository or SIF when their terms permit it and the files contain no credentials or inappropriate personal information. Do not assume that all license files must remain external.
- When redistribution is prohibited, document acquisition and verification, declare the expected in-container path, and bind the user-provided file read-only at runtime.
- If persistent storage cannot lawfully host an image, publish the definition, permitted components, and a tracked institutional retrieval/build adapter; do not imply public distributability.

### Exit gate

Each authoritative image has a valid signature/verification policy, SBOM, SIF checksum, annex key, persistent retrieval test, required-license test, and successful target-Slurm smoke test. Any optional OCI application artifact has its own documented consumer and identity rather than standing in for the SIF.

## Phase 5 — adapt `babs_demo` into a reusable campaign template

### What to retain

- a BIDS study root;
- raw data under `sourcedata/raw/` as an independently described and validated raw BIDS/DataLad dataset;
- a DataLad dataset of registered SIF containers;
- one independently described BIDS derivative/DataLad dataset per dataset × pipeline/runtime × input-condition campaign under `derivatives/`;
- BABS administrative state kept distinct from claimed BIDS derivatives;
- merged zip extraction as a separate `datalad containers-run` record using a registered analysis SIF.

### What to replace

- Replace `uv` and `requirements.txt` with the pinned Pixi `babs` environment.
- Replace mutable `docker://...:tag` inputs with exact registered SIF content.
- Replace generated YAML written to a temporary directory with reviewed files under `config/babs/`.
- Replace cluster `.env` files with tracked, non-secret host profiles plus external credentials.
- Enable and fix `babs check-setup --job-test`; do not retain the demo’s commented-out check.
- Replace hard-coded container paths and names with image-ledger lookups or validated campaign configuration.
- Store exact `babs init`, submit, retry, and merge commands in the operations record, and store the explicit DataLad Containers invocation used for extraction in its run record.
- Pin BABS rather than installing its moving default branch.

### Campaign specification

Create a schema-validated `operations/<campaign>/campaign.yaml` containing:

- campaign ID and purpose;
- BIDS input DataLad URL, dataset ID, and commit;
- selection/filter file and hash;
- processing level;
- registered container name, dataset commit, annex key, and SIF SHA-256;
- BABS and DataLad versions;
- BABS container config path/hash;
- input condition and resampling identity;
- Slurm host profile and resources;
- expected zip/output names;
- retry and exclusion policy;
- protected/public classification.

Store reviewed literal BABS commands under `operations/<campaign>/commands/` and explain their order and recovery conditions in `runbook.md`. Append each actual execution and its before/after state to `commands.jsonl`. This keeps the campaign actionable while distinguishing the intended command from evidence that it ran.

A Pixi task may validate the campaign, call the documented parameterized `babs init`, setup check, submit, status/retry, or merge command, or expose another lifecycle operation. The task definition is prospective actionability; the README/runbook explains setup, and the operations ledger plus repository history records what actually ran and changed. This is the explicit BABS meta-provenance exception.

For project-authored result-changing steps outside BABS, a Pixi task may call `datalad containers-run` with the registered SIF and explicit scientific program. Never call `pixi run <task>` from inside DataLad, because the historical run record would preserve only the task indirection. Follow the full [task/provenance decision](decisions/pixi-tasks-and-provenance.md).

### Exit gate

The toy/simulated BABS campaign works from a clean clone, every lifecycle command is recorded, the generated participant command names the tracked SIF path, and merged/unzipped outputs have DataLad provenance.

## Phase 6 — run the `ds007116` pilot

### Order

1. Install a pinned `ds007116` snapshot as `studies/ds007116/sourcedata/raw`.
2. Validate the BIDS dataset and record reviewed warnings.
3. Generate a cohort manifest containing all eligible scans; do not use the old ABCD balance file.
4. Start with FreeSurfer 7.4.1 on one participant/session.
5. Inspect the complete BABS run record, SIF identity, bind mounts, outputs, zip, logs, merge, and extraction.
6. Replay a representative run from historical state or perform the strongest replay current BABS supports and document remaining wrapper indirection.
7. Scale FreeSurfer 7.4.1 to the complete pilot only after the one-case gate passes.
8. Repeat the one-case gate then full pilot for FreeSurfer 8.2.1, `recon-any`, and `recon-all-clinical`.
9. Add native/resampled variants only where the required input is available and the transformation has passed unit/integration tests.
10. Run extraction, completeness accounting, analysis, and figures as separate explicit `datalad containers-run` operations. They may be launched by parameterized Pixi tasks only when DataLad receives and records the explicit program and arguments.

### Expected pilot result

The pilot should generate the same kinds of tables and figures as the poster but is not expected to reproduce ABCD effect sizes. It validates the research object, identifies unmodeled input heterogeneity, and estimates storage/Slurm requirements.

### Exit gate

- all eligible scans are accounted for;
- failures and retries have structured reasons;
- every derivative resolves to input, code/config, and SIF identities;
- downstream steps replay with `datalad rerun` or `datalad containers-run`;
- no authoritative output depends on notebook state or on interpreting a historical Pixi task definition; any task-launched scientific step resolves to an explicit DataLad command and registered SIF;
- a second Slurm site can adapt only the host profile, or any remaining site coupling is documented.

## Phase 7 — prepare controlled ABCD data

Do not begin this phase until data access and derivative handling are reviewed against the current ABCD DUC.

### Work

1. Record the allowed ABCD release, access endpoint, DUC version, project approval, and permitted outputs.
2. Create a private DataLad input dataset or tracked retrieval adapter without placing credentials in Git.
3. Write `cohort build-abcd` to regenerate the intended selection from release metadata.
4. Initially allow an all-eligible scan manifest for systems testing if policy and compute permit; add the poster-like unrelated/four-wave selection as a distinct, versioned cohort.
5. Unit-test relationship exclusion, wave balancing, modality pairing, duplicate handling, age derivation, missingness, and stable ordering.
6. Put the cohort manifest itself in protected storage if it contains subject identifiers.
7. Establish private input, derivative, logs, extracted morphometrics, and QC siblings. Establish a separate disclosure-reviewed aggregate-output sibling.
8. Run a credential-free public push test before submitting jobs.

### Exit gate

The approved release and exact cohort can be regenerated, protected content cannot leak through any public sibling, and every intended output class has a distribution decision.

## Phase 8 — run ABCD campaigns and reconstruct the poster

Use the campaign pattern proven on `ds007116`:

1. FreeSurfer 7.4.1 reference, with high-resolution T1w/T2w as declared.
2. FreeSurfer 8.2.1 comparison on matching inputs.
3. Pinned `recon-all-clinical` on native T1w.
4. Pinned `recon-all-clinical` on 1×1×5 mm T1w.
5. Pinned `recon-any` on native T1w.
6. Pinned `recon-any` on 1×1×5 mm T1w.

For each campaign:

- run one participant/session and inspect;
- submit in bounded batches;
- record status and retries;
- merge only audited successes;
- extract zips through a recorded `datalad containers-run` step using the registered analysis SIF;
- freeze the campaign commit before downstream extraction;
- emit completeness and exclusion tables.

Then execute separate registered analysis-image steps for:

- morphometric extraction;
- cross-pipeline assembly;
- cohort/age joins;
- global agreement and age-bias models;
- region-wise models and milestone join;
- figure/table generation;
- result-manifest generation;
- numeric and disclosure validation.

### Exit gate

Each poster target has a result-manifest row and regenerates from exact permitted data, derivative commits, code/config, and SIFs. Deviations from poster reference observations have a written analysis rather than silent parameter changes.

## Phase 9 — verification and release

### Verification matrix

- Fresh recursive DataLad clone with only public credentials.
- Fresh private clone with ABCD credentials.
- SIF retrieval from every promised persistent location.
- SIF signature/checksum verification, plus verification of any optional OCI application artifact that was actually produced.
- One clean Slurm participant replay per image family.
- One downstream `datalad rerun` per scientific boundary.
- Independent BIDS validation, or reviewed expected exceptions, for each outer Study dataset, raw input, and claimed derivative dataset.
- Schema and checksum validation for every authoritative table/figure.
- Cross-host pilot on a second Slurm system when available.
- macOS/Lima direct-build and smoke-test path where used, including an x86_64 guest on Apple Silicon for x86_64 targets, followed by acceptance on a representative Slurm host.
- Public-release scan for subject identifiers, credentials, absolute paths, annex URLs, and prohibited derivative content.
- REUSE compliance and license/DUC compatibility review.

### Release objects

- top-level DataLad dataset tag and signed release;
- pinned subdataset commits;
- public code/documentation archive with persistent identifier;
- private ABCD dataset state identifier where permitted;
- SIF checksums, signature/Sigstore bundles, attestations, and SBOMs, plus OCI digest references only for optional OCI artifacts actually produced;
- exact SIF artifacts in the DataLad container dataset and persistent annex storage, plus the same bytes in a SIF/ORAS registry when that secondary channel is used;
- result manifest and validation report;
- known limitations, including current BABS wrapper indirection and any non-redistributable runtime.

## Risks and controls

| Risk | Control |
|---|---|
| `recon-any` or clinical version remains ambiguous | Block that image family at Phase 4; proceed with FreeSurfer pilots |
| Pixi silently rewrites a lock during ordinary work | Require `--locked`; permit lock changes only in dependency-authoring branches |
| Scientific command hidden behind task name | Require task → `datalad containers-run` → explicit program; prohibit DataLad → Pixi task and inspect run records in integration tests |
| Current BABS record contains a wrapper | Pin BABS; track generated scripts/config; test representative replay; record limitation |
| BABS lifecycle command missing from provenance | Append exact argv/state transition to tracked operations ledger |
| A direct SIF rebuild differs despite the same definition/lock | Treat the registered SIF bytes as authoritative; record all build inputs, hash, annex, and publish the exact artifact |
| Candidate debug output enters the research object | Quarantine debug campaigns; register the image first and regenerate every retained output |
| A license is unlawfully withheld or redistributed | Record terms; distribute when permitted, otherwise use a documented read-only bind and scan for secrets/personal data |
| Study/raw/derivative metadata boundaries blur | Pin BIDS 1.11.1; describe and validate the outer Study dataset, raw child, and each derivative independently |
| ABCD subject data or derivatives become public | Separate datasets/siblings, protected logs/manifests, credential-free publication test |
| Pilot cohort cannot support poster inference | Treat pilot only as workflow validation; reserve conclusions for ABCD |
| Notebooks regain authoritative logic | Test CLIs and declare notebooks non-authoritative; fail review on unique notebook logic |
| External TemplateFlow/milestone data drift | Register exact content as DataLad input with license and citation |

## Definition of done

The conversion is done when a collaborator can install the top-level DataLad dataset, obtain every component they are authorized to access, verify the exact SIF images, inspect the command and data lineage of each result, replay representative steps, and regenerate the declared poster-like result set without depending on the original notebooks, NIH path layout, shell history, an historical Pixi task definition, or an untracked environment.
