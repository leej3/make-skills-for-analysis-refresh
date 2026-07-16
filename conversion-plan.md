# Conversion plan for `stamped_dl_morphometrics_biases`

## Objective

Create a new DataLad research object, `stamped_dl_morphometrics_biases`, that reconstructs the analyses summarized in the [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf). Begin with all eligible scans in the open `ds007116` BIDS dataset, then scale the proven pattern to controlled ABCD data.

This is an opinionated implementation of a STAMPED-compliant neuroimaging analysis. Repository organization, data boundaries, environment policy, image identity, provenance capture, access controls, and campaign operations must exist and pass a toy proof before historical scientific code is imported. The old `dl_morphometrics_biases` repository remains read-only evidence; it is not cleaned up in place.

Copy the repository-local [STAMPED neuroimaging skill](skills/stamped-neuroimaging-analysis/SKILL.md) into the new repository in its first commit and treat it as project policy. The implementation is also governed by the accepted decisions on the [role of Pixi](decisions/pixi-role.md), [Pixi tasks and provenance](decisions/pixi-tasks-and-provenance.md), [candidate versus authoritative SIFs](decisions/runtime-image-strictness.md), and [role of MechaBABS](decisions/mechababs-role.md). Revise the applicable decision record before changing those policies.

## Fixed decisions

- Make the repository's organizational and provenance decisions at the start. Draw useful patterns from `babs_demo` and MechaBABS without allowing either project to determine the repository layout after scientific code has been imported.
- Pin [BIDS 1.11.1 Study datasets](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset). Each study root has `DatasetType: "study"`; its raw BIDS input and every claimed derivative are independently described and validated DataLad datasets composed into that study.
- Use DataLad/git-annex for data and SIF identity, availability, provenance, and replay.
- Use BABS for BIDS participant/session expansion, Slurm submission, audit/retry, and merge. Target generic Slurm and keep host-specific policy in profiles.
- Treat MechaBABS as a pinned design reference and optional pilot only. Reuse suitable declarative configuration, command-rendering, and testing patterns now, but do not make authoritative execution depend on MechaBABS until the [promotion gates](decisions/mechababs-role.md#promotion-gates) are met.
- Use Pixi for named development, analysis, BABS/tooling, test, and image-authoring environments, and for locked Linux dependency installation during SIF construction. Pixi is not the image builder, workflow engine, provenance authority, or result-runtime identity.
- Store the real Pixi manifest and lock under `envs/`, with tracked root symlinks for discovery. Do not use `pixi pack` or another packed-prefix artifact.
- Use one `analysis` environment with `pandas<2` while the selected `freesurfer-stats` release requires it. This is the analysis environment, not a legacy extraction environment. When a qualified release supports `pandas>=2`, review the dependency update, produce a new lock and analysis SIF, and rerun affected results.
- Let Pixi tasks provide local and actionable interfaces. A task may wrap a BABS lifecycle operation, an image operation, or a complete `datalad containers-run` command. Never record only `pixi run <task>` inside DataLad, never let a scientific task bypass DataLad, and never use Pixi task dependencies or caching as the scientific workflow.
- Build project-authored BIDS Apps from [`bids-apps/example`](https://github.com/bids-apps/example). Their SIF runscripts invoke tracked app entrypoints accepting the standard BIDS App arguments; Pixi tasks may launch them but do not define their interfaces. DataLad records must expose the app arguments and any project-specific operation.
- Use exact registered SIFs for every retained, merged, compared, or released result. Candidate images may run in quarantined debugging campaigns, but their outputs are disposable and must be regenerated after image registration.
- Prefer an exact SIF already distributed through ReproNim/containers when its tool version, BIDS App interface, architecture, licensing, and behavior qualify. Pin the ReproNim DataLad commit and image annex key, compute a project SHA-256, and validate the image; a name or upstream OCI tag is not sufficient identity.
- Build a SIF directly from a tracked Apptainer/Singularity definition when no existing image qualifies. A separate OCI application image requires a documented OCI consumer.
- Accept the current BABS wrapper and zip indirection, document it, and test representative replay. Do not reproduce that indirection in project-authored targets or depend on an unreleased BABS branch.
- Treat `ds007116` as an engineering and scientific pilot. ABCD is the later claim-bearing analysis.
- Ignore the old `balanced_scans.csv` until ABCD access exists, then replace it with a tracked cohort target using the permitted release and query.
- Do not retrieve old NIH cluster derivatives until their DUC status is resolved. They are not required to begin.

## Migration strategy

Use gated phases. A phase is complete only when its evidence is committed to the new DataLad research object. Do not import historical scientific code until the empty repository, environments, image registry, and toy BABS campaign have proved the intended organization and guardrails.

## Phase 0 — preserve evidence and declare targets

### Work

1. Create a read-only inventory of the old repository at its current commit.
2. Record the source commit for every module or script that may later be imported, even though it is not the historical result-generating commit.
3. Copy the poster into a documentation/reference dataset or register a persistent reference if redistribution is not permitted.
4. Copy [poster-derived result targets](result-targets.md) into the new repository and assign stable result IDs.
5. Record runtime knowledge:
   - FreeSurfer `recon-all` 7.4.1 is an intended comparison;
   - FreeSurfer `recon-all` 8.2.0 is the second intended comparison; the [ReproNim NeuroDesk 8.2.0 image](https://github.com/ReproNim/containers/blob/0284fc8ad8b7fa9a76c3c9f03cfb2919708ba2b2/.datalad/config#L303-L305) is an installation/runtime candidate that still needs the required BIDS App interface and project qualification;
   - exact `recon-any` and `recon-all-clinical` distributions, builds, and weights remain qualification gates.
6. Record the intended native and 1×1×5 mm input conditions without copying notebook-generated command files.
7. Define the initial result-manifest schema and distinguish pilot-only from claim-bearing outputs.

### Evidence

- `docs/source-inventory.tsv`;
- `docs/result-targets.md`;
- accepted records under `docs/decisions/`;
- initial result and runtime-candidate manifests.

### Exit gate

Every intended poster panel has an ID, expected inputs and outputs, and a declared evidence class. Every uncertain runtime is explicitly unresolved rather than assigned a guessed version.

## Phase 1 — establish the repository and BIDS Study guardrails

### Work

1. Create the top-level DataLad dataset with a text-friendly configuration.
2. Add the repository-local skill and an `AGENTS.md` pointer requiring it for analysis work.
3. Establish the complete directory pattern from [architecture and provenance](architecture-and-provenance.md), including `envs/`, `config/`, `operations/`, `studies/`, `results/`, the registered-container dataset, tests, documentation, and access-control metadata.
4. Record BIDS 1.11.1 as the pinned specification. Create an empty toy Study dataset immediately so the organization is executable rather than aspirational.
5. Give every outer `studies/<study>/` dataset a `dataset_description.json` with `DatasetType: "study"` and a README. Install `sourcedata/raw/` as an independent raw BIDS/DataLad dataset. Create every `derivatives/<pipeline>-<variant>/` as an independent derivative BIDS/DataLad dataset with its own README and `dataset_description.json`.
6. Require each claimed derivative to record `GeneratedBy` and its immediate `SourceDatasets`. Validate metadata at each dataset root; do not label BABS administrative state as a BIDS derivative.
7. Add Git attributes so SIFs and large outputs are annexed rather than committed to Git.
8. Define the public/private sibling matrix before any ABCD content exists. Include a protected-data simulation that proves a public push cannot reach protected inputs, identifiers, logs, or derivatives.
9. Add `REUSE.toml`, required SPDX license texts, and file-group policy for code, documentation, configuration, generated metadata, externally governed data, models, installers, and license files.
10. Add a machine-readable component inventory containing DataLad dataset IDs, URLs, versions/commits, licenses or DUCs, BIDS descriptions, and distribution status.
11. Define the campaign record and append-only operations-ledger schemas now. Require every `operations/<campaign>/` to be an independent DataLad dataset with its own identity and commit history. A campaign must identify inputs, image, configuration, host profile, resources, outputs, retry policy, and access class before it can be initialized.
12. Review `babs_demo` and MechaBABS only against these established invariants. Adopt useful patterns; reject any pattern that would blur Study/raw/derivative boundaries, hide commands, or make the repository depend on unqualified tooling.

### Evidence

- a clean recursive clone of the empty skeleton;
- saved `datalad subdatasets` and `git annex info` reports;
- BIDS validation or reviewed expected exceptions for the toy Study root, raw child, and empty derivative roots;
- `reuse lint` passing for project-owned files;
- a credential-free publication dry run that cannot reach protected siblings;
- schema tests for campaign records, image records, result manifests, and operations events.

### Exit gate

A collaborator can understand and validate the research object's Study/raw/derivative composition, operations model, licensing, and access boundaries before any historical analysis code is present.

## Phase 2 — establish Pixi and image-source foundations

### Pixi workspace

1. Keep `pyproject.toml` focused on package metadata and importable code where packaging is useful. Keep BIDS App entrypoints under `apps/` and project launchers in Pixi tasks rather than treating either as package metadata.
2. Create `envs/pixi.toml`, `envs/pixi.lock`, and tracked root discovery symlinks.
3. Declare `linux-64` for runtime-dependency environments and the relevant macOS platforms for development.
4. Define at least:
   - `dev` for tests, documentation, linting, and optional notebooks;
   - `analysis` for extraction, assembly, models, figures, and validation, initially including the required `pandas<2` constraint;
   - `babs` for pinned BABS, DataLad, datalad-container, git-annex, and orchestration tools;
   - named Linux image environments where dependency graphs genuinely differ;
   - an image-authoring environment for Apptainer/Lima, signing, SBOM, and verification tools when useful.
5. Pin the supported Pixi range and record exact Pixi, Apptainer/Singularity, and Lima versions/configuration used for release builds.
6. Ignore `.venv/` and realized `envs/.pixi/`; never bind them into authoritative runs.
7. Add Pixi tasks for validation, tests, documentation, `reuse lint`, environment reports, explicit image operations, parameterized BABS operations, and complete DataLad-container invocations. A task definition supplies actionability, not execution evidence.
8. Run routine work and image installation with `--locked`. Only dependency-authoring changes may update the lock.

### Image-source registry

1. Create the DataLad container dataset and `envs/images.lock.yaml` before selecting production images.
2. Define image states: discovered, candidate, qualified, registered-authoritative, superseded, and rejected.
3. For external images, record the source DataLad dataset ID and commit, annex key, original source reference, local SHA-256, architecture, licenses, expected BIDS App interface, and retrieval locations.
4. For project-built images, reserve tracked definition paths and record the manifest/lock, base, non-Pixi inputs, builder versions, architecture, build command, SBOM, signature, SIF hash, annex key, and tests.
5. Add retrieval and verification tests using a small redistributable toy SIF. Do not wait for the scientific runtimes to test the registry.

### Dependency update cycle

1. Make a focused dependency/image change.
2. Edit `envs/pixi.toml`, run `pixi lock` intentionally, and inspect the manifest and lock diff.
3. Test relevant macOS environments with `--locked` and realize the reviewed Linux solution on the target architecture.
4. If the change affects a runtime, create a new SIF identity; never mutate an authoritative image in place.
5. Commit the manifest, lock, tests, image-ledger change, and rationale together. There is no freshness requirement for an otherwise working lock.

### Exit gate

A clean macOS checkout can realize development environments, Linux can realize selected image environments without rewriting the lock, and an exact toy SIF can be registered, removed, retrieved, and verified from persistent storage.

## Phase 3 — prove the organization with a toy BABS campaign

### Work

1. Install a small redistributable raw BIDS/DataLad dataset at `studies/toy/sourcedata/raw/`.
2. Register a small exact SIF with a BIDS App interface in the container dataset. Record its commit, annex key, SHA-256, and retrieval location.
3. Create `operations/toy-<campaign>/` as an independent DataLad dataset, then add a schema-valid `campaign.yaml`, reviewed BABS configuration, literal command renderings, runbook, and append-only `commands.jsonl`.
4. Use a Pixi task to make each BABS lifecycle operation actionable, but record the actual expanded `babs init`, setup check, submit, audit/retry, and merge argv plus before/after repository state in the operations ledger.
5. Run the released BABS path from a clean clone, including its wrapper and merged-zip behavior.
6. Preserve the merged BABS archive dataset as a scientific upstream artifact inside the operations dataset. Keep BABS administrative state and logs outside any claimed derivative boundary.
7. When BABS emits zipped or otherwise non-derivative payloads, materialize them into a separate scientific derivative dataset with an explicit `datalad containers-run` record, registered SIF, declared archive inputs and derivative outputs, and visible command. Do not hide finalization behind a historical Pixi task name.
8. Give the finalized derivative its own README and `dataset_description.json`, including `GeneratedBy` and immediate `SourceDatasets`; validate it independently, then install that exact DataLad dataset and commit at `studies/toy/derivatives/<pipeline>-<variant>/`. A future already-valid BABS derivative may skip materialization but not independent validation.
9. Test representative replay, SIF retrieval, failed-job retry, merge audit, operations-ledger recovery, and a clean recursive clone of the composed Study dataset.
10. Compare the implementation with useful `babs_demo` and MechaBABS patterns. Record adopted patterns in project documentation. MechaBABS may run a non-authoritative comparison pilot inside its operations dataset, but its outputs cannot be composed into a scientific Study derivative or result until the separate promotion decision is satisfied.

### Exit gate

The toy campaign proves, before scientific code import, that a BABS archive can be provenance-captured, materialized, and installed as an independent Study derivative; exact images are retrievable; lifecycle commands are recorded; protected and public state remain separated; and merged/extracted outputs carry DataLad provenance.

## Phase 4 — import and refactor scientific code as tested BIDS Apps

### Target design

Import one scientific boundary at a time. Each import records its old-repository source commit, removes notebook and host assumptions, receives tests, and passes a toy containerized invocation before the next target is imported.

Implement a closed, documented target interface such as:

| Target | Allowed analysis level | Accepted inputs | Declared outputs |
|---|---|---|---|
| `cohort` | `group` | raw BIDS metadata and tracked selection configuration | ordered cohort and exclusion tables |
| `resample` | `participant` | declared BIDS structural images | BIDS derivative images plus transformation metadata |
| `extract` | `participant` | a qualified reconstruction derivative | stable morphometric tables and completeness records |
| `assemble` | `group` | declared extraction derivatives | keyed cross-pipeline analysis tables |
| `model` | `group` | declared analysis tables and model configuration | model estimates, diagnostics, and statistical tables |
| `figures` | `group` | declared model/statistical outputs and result IDs | only the requested figures and table renderings |
| `validate` | `participant` or `group` | a declared dataset, derivative, or result set | machine-readable validation report |

Before importing each operation, decide whether it is its own BIDS App or a coherent mode of another app. Default to a separate app for an independently versioned scientific transformation; combine operations only when they share one meaningful input/output contract and lifecycle. Each app follows [`bids-apps/example`](https://github.com/bids-apps/example): its container entrypoint accepts `bids_dir`, `output_dir`, and an implemented `analysis_level`, provides participant selection where meaningful, documents validation behavior, exposes help/version information, and rejects unknown arguments. Any additional operation selector must be closed and explicit. Reconstruction and downstream apps follow the same image, testing, DataLad, and metadata rules.

The SIF runscript invokes the tracked `apps/<app-name>/run.py` entrypoint. Pixi tasks may expose local invocations or wrap a complete `datalad containers-run` command, but they do not define the BIDS App interface. Do not let one operation silently invoke a hidden operation DAG. Record each result-changing operation as its own explicit DataLad operation.

### Refactoring requirements

- Replace hard-coded `/data/ABCD_*`, NIH paths, and user cache paths with declared inputs and tracked configuration.
- Remove inline `uv` metadata and installation instructions.
- Keep the current `freesurfer-stats` compatibility constraint in the `analysis` environment. Do not label the work `extract-legacy`; qualify a newer dependency and image when available.
- Replace path-only pickle caches. Disposable caches must include content identities and remain outside provenance; scientific tables are declared outputs.
- Replace implicit output directories and path-parent inference with explicit target arguments and derivative metadata.
- Remove shell-driven `ls`, interactive `freeview`, scheduler submission, and unique notebook logic from authoritative paths.
- Emit structured exclusions and completeness tables. Make atlases, metrics, hemispheres, transformations, statistical models, seeds, and figure IDs reviewed configuration.
- Pin TemplateFlow and other reference resources by content and record their licenses.
- Use tiny synthetic or redistributable FreeSurfer-like fixtures for unit and integration tests.

Notebooks may remain only when they consume tracked outputs, contain no unique scientific logic, do not submit or write authoritative results, and can be deleted without preventing reconstruction.

### Iteration gate for each target

1. Import the minimum reviewed code and record its source commit.
2. Add schema and unit tests.
3. Run the target locally through the locked Pixi environment for development.
4. Add it to the selected `apps/<app-name>/run.py` entrypoint, expose that entrypoint through the BIDS App SIF runscript, and build a candidate SIF.
5. Run a toy `datalad containers-run` invocation with explicit inputs, outputs, target, and arguments.
6. Review the resulting BIDS derivative metadata and DataLad record.
7. Only then import the next target.

### Phase exit gate

The fixture path from a reconstruction-like derivative through extraction, assembly, modeling, one declared figure, and validation works through explicit BIDS App executions. No result depends on notebook order, shell history, an institutional path, or interpretation of an old Pixi task definition.

## Phase 5 — qualify, reuse, or build scientific images

### Candidate hierarchy

1. Prefer an exact ReproNim SIF when it implements the required BIDS App behavior.
2. Otherwise qualify another exact redistributable SIF and register it in the project container dataset.
3. Otherwise build a project SIF from a tracked definition and locked Pixi environment.
4. Never substitute a similarly named version or reconstruct authoritative results from a mutable tag when exact SIF bytes are available.

### FreeSurfer candidates

- FreeSurfer 7.4.1: begin with ReproNim/containers' registered [`bids-freesurfer--7.4.1-unstable.sif`](https://github.com/ReproNim/containers/blob/0284fc8ad8b7fa9a76c3c9f03cfb2919708ba2b2/images/bids/Singularity.bids-freesurfer--7.4.1-unstable). The `unstable` source tag is not the runtime identity. Pin the ReproNim DataLad commit and annex key, compute SHA-256, inspect licensing, and validate the exact SIF on a fixture and full pilot subject before promotion.
- FreeSurfer 8.2.0: evaluate ReproNim's registered NeuroDesk image as an installation/runtime candidate. It exposes FreeSurfer tools rather than the required BIDS App interface, so either qualify a tracked wrapper or build a project BIDS App.
- For a project FS8 BIDS App, reuse audited ideas from [BIDS-Apps/freesurfer](https://github.com/BIDS-Apps/freesurfer) and the [NeuroDesk FreeSurfer recipe](https://github.com/NeuroDesk/neurocontainers/blob/aa4b9af982ce409c91258281b551a06bf7028fb4/recipes/freesurfer/build.yaml). Update the wrapper for FS8 semantics, use the project's Pixi lock, checksum every fetched input, and add a real BIDS participant integration test. Do not merely change the old generator's version string.

Qualify exact `recon-any` and `recon-all-clinical` releases, weights, licenses, architecture, and BIDS App interfaces separately. Build the downstream analysis SIF from the targets proven in Phase 4.

### Qualification gate

For every candidate:

- retrieve and verify the exact SIF from a clean clone;
- record tool, wrapper, and embedded dependency versions;
- inspect the runscript, environment sanitation, binds, architecture, and license behavior;
- run interface, version, fixture, invalid-target, and expected-output tests;
- run one complete representative subject when the application is participant-level;
- validate derivative metadata and stable output naming;
- confirm that BABS-generated execution identifies the registered SIF path;
- compute SHA-256, record the annex key/DataLad commit, generate an SBOM where feasible, and apply the project's signature/verification policy;
- reject or wrap the image explicitly if its interface does not meet the target contract.

### Project SIF construction

1. Store definitions under `envs/containers/<image>/`.
2. Pin/checksum the base and every non-Pixi input.
3. Install the selected Linux environment from `envs/pixi.toml` and `envs/pixi.lock` with a pinned Pixi and `--locked` at its final in-image path.
4. Build directly for the target Slurm architecture. On macOS, use Apptainer inside Lima; use an x86-64 guest on Apple Silicon for x86-64 Slurm targets.
5. Record the builder, architecture, definition and lock hashes, command, tests, and all inputs.
6. Smoke-test and sign/verify the final SIF, then compute its authoritative checksum.
7. Register it with `datalad containers-add`, publish annex content to persistent storage, and update `envs/images.lock.yaml`.
8. Publish a separate OCI application image only for an identified OCI consumer; identify it independently from the SIF.

Candidate SIFs may be used to debug in quarantined local or Slurm space. Retained outputs must be regenerated through the promoted registered SIF according to the [runtime-image decision](decisions/runtime-image-strictness.md).

### Licensing

- Record redistribution terms for software, installers, weights, and required license files.
- Distribute license files with the repository or SIF when permitted and when they contain no secret or inappropriate personal information.
- When redistribution is prohibited, document acquisition and verification and bind the supplied file read-only.
- If the exact image cannot lawfully be hosted, publish only permitted definitions/components and a tracked retrieval/build adapter; do not imply public availability.

### Exit gate

Every image admitted to a scientific campaign has a qualified interface, exact SIF identity, persistent retrieval test, license decision, target-architecture test, and registered DataLad identity. Unresolved image families remain blocked without delaying already qualified pilots.

## Phase 6 — run the `ds007116` pilot

### Work

1. Install a pinned `ds007116` snapshot at `studies/ds007116/sourcedata/raw/` and validate it.
2. Generate an all-eligible cohort manifest with the tested `cohort` target; do not reuse the old ABCD balance file.
3. Start with the qualified FreeSurfer 7.4.1 ReproNim BIDS App candidate on one participant/session.
4. Inspect the complete BABS operation record, exact SIF, binds, outputs, logs, merge, and zip behavior.
5. Materialize a zipped BABS result through an explicit `datalad containers-run` operation with a registered SIF, then validate and install the resulting derivative dataset at the Study derivative path.
6. Run `extract`, `assemble`, `model`, `figures`, and `validate` as separate explicit DataLad-container operations for the one-case or smallest meaningful fixture.
7. Test representative historical replay and document the remaining BABS wrapper/zip indirection.
8. Scale FreeSurfer 7.4.1 only after the one-case gate passes.
9. Repeat the one-case and scale gates for FreeSurfer 8.2.0 and for qualified `recon-any` and `recon-all-clinical` images.
10. Add native/resampled variants only when the required input exists and the `resample` target has passed unit and integration tests. Freeze each operations dataset commit before downstream comparisons.

### Expected result

The pilot should generate the same kinds of tables and figures as the poster but is not expected to reproduce ABCD effect sizes. It validates the research object, target interfaces, completeness policy, storage needs, and Slurm resource estimates.

### Exit gate

- all eligible scans are accounted for;
- failures and retries have structured reasons;
- every derivative resolves to input, code/config, operations, and exact SIF identities;
- downstream targets replay through DataLad without notebook or Pixi-task indirection;
- the Study/raw/derivative composition survives a clean recursive clone;
- a second Slurm site can adapt only the host profile, or remaining coupling is documented.

## Phase 7 — prepare controlled ABCD data

Do not begin until access and derivative handling have been reviewed against the current ABCD DUC.

### Work

1. Record the allowed release, access endpoint, DUC version, project approval, and permitted outputs.
2. Create a private DataLad input dataset or tracked retrieval adapter without credentials in Git.
3. Use the tested `cohort` target to regenerate the intended selection from release metadata.
4. Permit an all-eligible systems-test cohort only if policy and compute allow; create the poster-like unrelated/four-wave selection as a separate versioned cohort.
5. Test relationship exclusion, wave balancing, modality pairing, duplicates, age derivation, missingness, and stable ordering.
6. Store protected cohort manifests, logs, derivatives, extracted morphometrics, and QC only on declared private siblings.
7. Establish a separate disclosure-reviewed aggregate-output sibling.
8. Run a credential-free public push test before submitting jobs.

### Exit gate

The approved release and cohort can be regenerated, protected content cannot leak through any public sibling, and every input/output class has a documented distribution decision.

## Phase 8 — run ABCD campaigns and reconstruct the poster

Use only campaign and image patterns that passed the toy and `ds007116` gates. The intended comparison families are:

1. qualified FreeSurfer 7.4.1 reference on declared T1w/T2w inputs;
2. qualified FreeSurfer 8.2.0;
3. qualified `recon-all-clinical` on native T1w;
4. qualified `recon-all-clinical` on 1×1×5 mm T1w;
5. qualified `recon-any` on native T1w;
6. qualified `recon-any` on 1×1×5 mm T1w.

For each campaign:

- run one participant/session and inspect;
- submit in bounded batches;
- record lifecycle commands, status, retries, exclusions, and commits;
- merge only audited successes;
- preserve merged BABS zips as protected scientific upstream artifacts inside the operations dataset;
- extract each BABS payload through a separate explicit `datalad containers-run` step using the registered analysis SIF, then compose and validate that independently described protected dataset as the Study derivative;
- emit completeness and exclusion tables;
- freeze the campaign commit before cross-pipeline analysis.

Then run the closed downstream targets separately for extraction, cross-pipeline assembly, cohort/age joins, global and regional models, milestone joins, figure/table generation, result-manifest generation, numeric validation, and disclosure review.

### Exit gate

Every poster target has a result-manifest row and regenerates from exact permitted data, derivative commits, code/configuration, recorded operations, and registered SIFs. Deviations from poster reference observations receive written analysis rather than silent parameter changes.

## Phase 9 — verify and release

### Verification matrix

- Fresh recursive public DataLad clone.
- Fresh private clone with authorized ABCD credentials.
- SIF retrieval and verification from every promised location.
- One clean Slurm participant replay per image family.
- One downstream DataLad replay per scientific target boundary.
- Independent BIDS validation, or reviewed exceptions, for every Study root, raw input, and claimed derivative.
- Schema and checksum validation for every authoritative table and figure.
- Cross-host pilot on a second Slurm system when available.
- macOS/Lima build and smoke-test path where used, followed by acceptance on representative Slurm.
- Public-release scan for subject identifiers, credentials, absolute paths, annex URLs, and prohibited derivatives.
- REUSE compliance and license/DUC review.

### Release objects

- tagged and signed top-level DataLad dataset state;
- pinned subdataset commits;
- public code/documentation archive with persistent identifier;
- permitted private ABCD dataset state identifier;
- exact SIFs in DataLad/annex storage with checksums, verification material, attestations, and SBOMs;
- OCI references only for optional OCI artifacts actually produced;
- result manifest and validation report;
- known limitations, including BABS wrapper/zip indirection and unavailable or non-redistributable runtimes.

## Risks and controls

| Risk | Control |
|---|---|
| The generic FreeSurfer 8.2.0 tool image is mistaken for the required BIDS App | Build or qualify the tracked BIDS App entrypoint and test the exact registered SIF |
| A ReproNim image name is mistaken for sufficient identity | Pin DataLad commit and annex key, compute SHA-256, retrieve, inspect, and validate exact bytes |
| `recon-any` or clinical version remains ambiguous | Block that image family while proceeding with qualified FreeSurfer pilots |
| MechaBABS maturity changes during conversion | Reuse design patterns only; require the separate promotion decision before authoritative dependency |
| Pixi rewrites a lock during ordinary work | Require `--locked`; permit lock changes only during reviewed dependency authoring |
| Scientific command is hidden behind a Pixi task | Require task → explicit `datalad containers-run`; prohibit DataLad → Pixi task and inspect run records |
| A BIDS App operation hides an internal scientific DAG | Use closed operations and one explicit DataLad record per result-changing boundary |
| Current BABS record contains wrapper/zip indirection | Pin BABS; record lifecycle operations; test representative replay; do not copy the pattern elsewhere |
| A BABS lifecycle command is missing | Append expanded argv and before/after state to the operations ledger |
| A direct SIF rebuild differs | Treat registered bytes as authoritative and preserve all build inputs and the exact artifact |
| Candidate output enters the research object | Quarantine candidate campaigns; register the image and regenerate every retained output |
| A license is unlawfully withheld or redistributed | Record terms; distribute when permitted; otherwise use a documented read-only bind |
| Study/raw/derivative boundaries blur | Pin BIDS 1.11.1 and independently describe, compose, and validate every dataset root |
| ABCD content becomes public | Separate datasets/siblings and run credential-free publication tests |
| Pilot data cannot support poster inference | Use the pilot for workflow/scientific validation and reserve claims for ABCD |
| Notebooks regain authority | Require tested BIDS App targets and reject unique notebook logic |
| External reference data drift | Register exact content with DataLad, license, citation, and checksum |

## Definition of done

The conversion is complete when a collaborator can install the top-level DataLad dataset, obtain every component they are authorized to access, verify every exact SIF, inspect the explicit operation and data lineage of each result, replay representative operations, and regenerate the poster-like result set without relying on the old notebooks, NIH paths, shell history, a historical Pixi task definition, an untracked environment, or a guessed software version.
