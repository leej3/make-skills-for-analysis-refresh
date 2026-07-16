# Documentation map

This directory now separates current design decisions from the evidence that led to them.

## Canonical guidance

- [Conversion plan](conversion-plan.md) — staged migration to `stamped_dl_morphometrics_biases`.
- [Architecture and provenance](architecture-and-provenance.md) — responsibility boundaries, repository layout, campaign matrix, container identity, and operations records.
- [Poster-derived result targets](result-targets.md) — the result set that defines “good enough” scientific reconstruction.
- [Decision records](decisions/README.md) — the authoritative index for choices that govern the converted analysis.
- [Why Pixi belongs in the analysis](decisions/pixi-role.md) — environment roles, intentional lock updates, direct SIF construction, alternatives, and limitations.
- [Pixi tasks, DataLad provenance, and BABS operations](decisions/pixi-tasks-and-provenance.md) — safe task direction, provenance anti-patterns, and the BABS meta-provenance exception.
- [Candidate versus authoritative SIF policy](decisions/runtime-image-strictness.md) — disposable image debugging and the strict gate for retained outputs.
- [Repository-local agent skill](skills/stamped-neuroimaging-analysis/SKILL.md) — concise operating rules for agents working on the converted analysis.

## Baseline evidence

- [Original STAMPED audit](STAMPED_REVIEW_DL_MORPHOMETRICS_BIASES.md) — detailed review of the pre-conversion notebooks and scripts. It describes the starting point; the conversion plan supersedes its proposed layout and unresolved-question list.
- [STAMPED principles paper](../Macdonald%20et%20al.%20-%20STAMPED%20principles%20for%20reproducible%20research%20objects.pdf) — normative source.
- [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf) — scientific reconstruction target.

## Decision summary

- Pixi defines named local environments and provides reviewed lock inputs that are also used to reconstruct locked environments inside SIF images.
- Apptainer/Singularity builds the SIF directly from a tracked definition. It may bootstrap from a digest-pinned OCI base, but a separately built or published OCI application image is not required.
- DataLad Containers registers, retrieves, and executes the completed SIF. The exact registered SIF—not `pixi.lock`, a recipe, or an OCI tag—is the runtime identity attached to an authoritative scientific run.
- DataLad owns data identity, command provenance, and replay.
- BABS owns BIDS App expansion, Slurm execution, auditing, and merge.
- Pixi tasks may expose parameterized scientific commands when the task invokes `datalad containers-run` with the explicit executable, arguments, declared inputs and outputs, and registered SIF. The durable DataLad record must not contain only `pixi run <task>`.
- A Pixi task must not invoke a result-changing scientific command without DataLad provenance. Pixi task dependencies, caching, and skip decisions do not define the scientific workflow.
- BABS lifecycle tasks such as `init`, `submit`, retry, and merge are the explicit exception: the task and runbook provide prospective actionability, while an operations ledger records each expanded command and resulting state change.
- The current BABS wrapper and zipped result pattern is accepted as a documented, BABS-specific indirection and is not repeated in project-authored steps. Unreleased BABS improvements are not required.
- Candidate SIFs may be used locally or on Slurm for disposable debugging. No candidate output may be retained, merged, scientifically compared, or released; authoritative outputs are regenerated only after the exact SIF is registered and durably retrievable.
- The study layout follows the released [BIDS 1.11.1 Study dataset convention](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset): the outer DataLad dataset has `DatasetType: study`, the raw BIDS subdataset is at `sourcedata/raw/`, and each derivative is independently described and versioned. Pin the BIDS version rather than silently adopting the development-only `rawbids/` layout.
- `ds007116` is the engineering pilot. ABCD is the later controlled-data campaign and the basis for the poster-level age analyses.
