# Conversion plan for `stamped_dl_morphometrics_biases`

## Objective

Create a new DataLad research object, `stamped_dl_morphometrics_biases`, that reconstructs the analyses summarized in the [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf). Begin with all eligible scans in the open `ds007116` BIDS dataset, then scale the validated pattern to controlled ABCD data.

The new repository will not try to sanitize the existing `dl_morphometrics_biases` history in place. It will import reviewed scientific content with explicit provenance, leaving the old repository as historical evidence. The repository-local [STAMPED neuroimaging skill](skills/stamped-neuroimaging-analysis/SKILL.md) must be copied into the new repository at its first commit and treated as project policy.

## Fixed decisions

- Use a BIDS study/superdataset pattern adapted from the local `babs_demo`.
- Use DataLad/git-annex for data and SIF identity, availability, provenance, and replay.
- Use BABS for participant/session expansion, Slurm submission, audit/retry, and merge.
- Target generic Slurm rather than NIH Swarm or one cluster. Keep host profiles separate.
- Use Pixi for named local development, build, BABS, test, and analysis environments.
- Store the real Pixi manifest and lock under `envs/`; use root symlinks for discovery.
- Use exact, tracked SIF files for every scientific HPC run. A Pixi lock is a container-build input, not the production runtime identity.
- Build OCI images from reviewed locks, sign/attest the OCI digest, convert the exact digest to SIF once in a pinned Linux builder, and track the SIF as a distinct artifact.
- Do not use environment packing as an image or HPC strategy.
- Do not use Pixi task dependencies. Do not use Pixi tasks for any result-changing or campaign-changing action.
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
- `docs/decisions.md`
- initial result-manifest schema

### Exit gate

Every intended panel in the poster has an ID, expected inputs, expected outputs, and a declared status of pilot-only or claim-bearing.

## Phase 1 — create the new STAMPED/DataLad skeleton

### Work

1. Create the top-level DataLad dataset with a text-friendly configuration.
2. Add the repository-local agent skill and a one-line `AGENTS.md` pointer requiring it for analysis work.
3. Establish directories from [architecture and provenance](architecture-and-provenance.md).
4. Add Git attributes so large binary outputs and SIFs are annexed rather than committed to Git.
5. Add public/private sibling policy before any ABCD content is installed.
6. Add root `REUSE.toml` and the required SPDX license texts. Assign initial file groups:
   - code;
   - documentation;
   - configurations;
   - generated metadata;
   - externally governed data and models.
7. Add a machine-readable component inventory containing dataset IDs, URLs, licenses/DUCs, and distribution status.

### Evidence

- clean recursive clone of the empty skeleton;
- `datalad subdatasets` and `git annex info` output saved as a release check;
- `reuse lint` passing for project-owned files;
- a credential-free publication dry run that cannot reach future protected siblings.

### Exit gate

The empty research object can be cloned and its module boundaries, licenses, and access policy are understandable without local paths.

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
3. Declare `linux-64` for runtime/build environments and the relevant macOS platform(s) for developer environments.
4. Use distinct named environments:
   - `dev` for lint, tests, documentation, and local notebooks;
   - `analysis` for current extraction/model/figure development;
   - `extract-legacy` only while `freesurfer-stats` requires `pandas<2`;
   - `babs` for pinned BABS, DataLad, datalad-container, and supporting tools;
   - container-specific build environments only when needed.
5. Pin the Pixi version range supported by the workspace and record the exact Pixi binary/builder image for releases.
6. Remove `.python-version` if it misleadingly implies one Python for all environments; otherwise document it as an editor hint only.
7. Ignore `.venv/` and realized `envs/.pixi/`. Do not bind either into production containers or Slurm jobs.
8. Add only non-scientific Pixi tasks, for example lint, unit tests, docs checks, `reuse lint`, environment reports, BABS config validation, `babs status`, and printing a tracked command.

### Dependency update cycle

1. Create a branch dedicated to the environment change.
2. Edit `envs/pixi.toml` directly.
3. Run `pixi lock` intentionally and inspect `git diff -- envs/pixi.toml envs/pixi.lock`.
4. Realize and test with `--locked` on macOS where relevant and `linux-64` in a clean builder.
5. If the change can affect a runtime, rebuild a new OCI/SIF; never mutate an existing image identity.
6. Commit manifest, lock, tests, and image-ledger changes together with the reason for the change.
7. Leave a stable analysis lock unchanged unless a dependency change is required. CI should reject manifest/lock mismatch through locked realization; it should not update dependencies for freshness.

### Exit gate

A fresh macOS checkout can realize developer environments, a clean Linux job can realize the reviewed lock, and neither route silently edits `pixi.lock`.

## Phase 4 — build, sign, and register runtime images

### Image families

Create independent image definitions for:

- FreeSurfer 7.4.1 `recon-all`;
- FreeSurfer 8.2.1 `recon-all`;
- a pinned `recon-any` release/build and weights;
- a pinned `recon-all-clinical` release/build and weights;
- downstream extraction/statistics/figure analysis.

An image may cover multiple commands only when its license, dependencies, update cadence, and scientific role make that boundary defensible. Do not combine all tools merely to reduce image count.

### Build path

1. Use tracked OCI recipes under `envs/containers/<image>/`.
2. Pin `FROM` images by digest.
3. In the builder, copy `envs/pixi.toml` and `envs/pixi.lock`, then realize the selected environment with `pixi install --locked`.
4. Copy the realized prefix and application code into a minimal final stage at the same absolute prefix, or retain Pixi in the final image only with a documented reason.
5. Build for `linux/amd64` unless the target Slurm systems require an additional architecture.
6. Run smoke tests that execute the actual tool entry point and a tiny fixture, not only `--version`.
7. Push OCI by digest to the registry.
8. Generate an SBOM and build-provenance attestation; sign the OCI digest with Sigstore.
9. Verify the signature using a tracked certificate-identity and issuer policy.
10. Convert the exact OCI digest to SIF in a pinned Linux conversion environment.
11. Hash and optionally sign the SIF, then push the exact SIF to a registry that preserves SIF bytes and to a persistent annex remote.
12. Add the SIF as content in the DataLad container dataset with `datalad containers-add` and record its annex key and DataLad commit in `envs/images.lock.yaml`.

Do not build a SIF opportunistically on a compute node. Apptainer notes that repeated OCI-to-SIF conversion can change SIF bytes; therefore the built SIF is a release artifact with its own identity, not an automatic cache of the OCI digest.

### Licensing

- Never copy `license.txt` into a FreeSurfer image or Git history.
- Declare the expected in-container path and bind the user’s licensed file read-only at runtime.
- Record model-weight and tool redistribution terms before publishing either OCI or SIF artifacts.
- If a registry cannot lawfully host an image, publish the recipe, permitted components, and a tracked institutional retrieval/build adapter; do not imply public distributability.

### Exit gate

Each image has an immutable OCI digest, valid signature policy, SBOM, SIF checksum, annex key, persistent retrieval test, required-license test, and successful clean-node smoke test.

## Phase 5 — adapt `babs_demo` into a reusable campaign template

### What to retain

- a BIDS study root;
- raw data under `sourcedata/` as an independent DataLad subdataset;
- a DataLad dataset of registered SIF containers;
- BABS projects/derivatives under `derivatives/`;
- merged zip extraction as a separate `datalad run` record.

### What to replace

- Replace `uv` and `requirements.txt` with the pinned Pixi `babs` environment.
- Replace mutable `docker://...:tag` inputs with exact registered SIF content.
- Replace generated YAML written to a temporary directory with reviewed files under `config/babs/`.
- Replace cluster `.env` files with tracked, non-secret host profiles plus external credentials.
- Enable and fix `babs check-setup --job-test`; do not retain the demo’s commented-out check.
- Replace hard-coded container paths and names with image-ledger lookups or validated campaign configuration.
- Store exact `babs init`, submit, retry, merge, and extraction commands in an operations record.
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

A Pixi task may validate the campaign file, show current status, or print one of those exact commands; it must not call `babs init`, `submit`, retry, or `merge`.

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
10. Run extraction, completeness accounting, analysis, and figures as separate explicit containerized DataLad operations.

### Expected pilot result

The pilot should generate the same kinds of tables and figures as the poster but is not expected to reproduce ABCD effect sizes. It validates the research object, identifies unmodeled input heterogeneity, and estimates storage/Slurm requirements.

### Exit gate

- all eligible scans are accounted for;
- failures and retries have structured reasons;
- every derivative resolves to input, code/config, and SIF identities;
- downstream steps replay with `datalad rerun` or `datalad containers-run`;
- no authoritative output depends on a notebook or Pixi task;
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
- extract zips through a recorded DataLad step;
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
- OCI and SIF signature/checksum verification.
- One clean Slurm participant replay per image family.
- One downstream `datalad rerun` per scientific boundary.
- BIDS validation for every input and derivative dataset.
- Schema and checksum validation for every authoritative table/figure.
- Cross-host pilot on a second Slurm system when available.
- macOS/Lima smoke test of the same Linux image path, without treating it as the HPC acceptance test.
- Public-release scan for subject identifiers, credentials, absolute paths, annex URLs, and prohibited derivative content.
- REUSE compliance and license/DUC compatibility review.

### Release objects

- top-level DataLad dataset tag and signed release;
- pinned subdataset commits;
- public code/documentation archive with persistent identifier;
- private ABCD dataset state identifier where permitted;
- OCI digest references, Sigstore bundles/attestations, and SBOMs;
- exact SIF artifacts in the registry and DataLad container dataset;
- result manifest and validation report;
- known limitations, including current BABS wrapper indirection and any non-redistributable runtime.

## Risks and controls

| Risk | Control |
|---|---|
| `recon-any` or clinical version remains ambiguous | Block that image family at Phase 4; proceed with FreeSurfer pilots |
| Pixi silently rewrites a lock during ordinary work | Require `--locked`; permit lock changes only in dependency-authoring branches |
| Scientific command hidden behind task name | Prohibit result-changing Pixi tasks; inspect DataLad/BABS run records in integration tests |
| Current BABS record contains a wrapper | Pin BABS; track generated scripts/config; test representative replay; record limitation |
| BABS lifecycle command missing from provenance | Append exact argv/state transition to tracked operations ledger |
| SIF rebuilt differently from the same OCI | Convert once, hash separately, annex and publish exact SIF |
| FreeSurfer license leaked into image/repository | Runtime read-only bind plus automated image/repository scan |
| ABCD subject data or derivatives become public | Separate datasets/siblings, protected logs/manifests, credential-free publication test |
| Pilot cohort cannot support poster inference | Treat pilot only as workflow validation; reserve conclusions for ABCD |
| Notebooks regain authoritative logic | Test CLIs and declare notebooks non-authoritative; fail review on unique notebook logic |
| External TemplateFlow/milestone data drift | Register exact content as DataLad input with license and citation |

## Definition of done

The conversion is done when a collaborator can install the top-level DataLad dataset, obtain every component they are authorized to access, verify the exact SIF images, inspect the command and data lineage of each result, replay representative steps, and regenerate the declared poster-like result set without depending on the original notebooks, NIH path layout, shell history, Pixi tasks, or an untracked environment.
