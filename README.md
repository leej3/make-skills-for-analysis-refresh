# Documentation map

This repository preserves the reasoning, evidence, and implementation guidance used to develop an ideal-oriented STAMPED neuroimaging pattern. The reusable skill is the end product; the longer documents make its choices auditable without burdening every analysis repository with the complete design history.

## Operational guidance

- [STAMPED neuroimaging skill](skills/stamped-neuroimaging-analysis/SKILL.md) — concise operating policy for creating, running, assessing, and releasing a STAMPED neuroimaging research object.
- [BIDS App builder skill](skills/bids-app-builder/SKILL.md) — operating policy for project-authored BIDS Apps.
- [Conversion plan](conversion-plan.md) — project-specific sequence for converting `dl_morphometrics_biases`.

Copy the conversion plan and complete skill bundles into the analysis repository. The supporting decision and rationale documents below need not be copied unless a project-specific departure requires its own record.

## Supporting design and rationale

- [Decision records](decisions/README.md) — rationale and responsibility boundaries that inform the skills.
- [Repository and Study organization](repository-and-study-organization.md) — extended discussion of research-object, BIDS Study, DataLad, derivative, and campaign boundaries.
- [Architecture and provenance](architecture-and-provenance.md) — broader architectural reasoning and provenance concepts.
- [Poster-derived result targets](result-targets.md) — the result set that defines the intended scientific reconstruction.

The skill and conversion plan govern implementation if a supporting document is more detailed or reflects an earlier design stage. Reconcile material conflicts here before changing those operational artifacts.

## Baseline evidence

- [Original STAMPED audit](STAMPED_REVIEW_DL_MORPHOMETRICS_BIASES.md) — review of the pre-conversion notebooks and scripts. Its evidence remains useful, but its proposed implementation is superseded by the skill and conversion plan.
- [STAMPED paper LaTeX source (`main.tex`)](https://github.com/stamped-principles/stamped-paper/blob/main/main.tex) — normative STAMPED source; clone the repository when local inspection or an exact revision is required.
- [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf) — scientific reconstruction target.

## Current boundary summary

- Pixi provides locked local environments, exact bootstrap and user-space tooling, custom-image dependency input, typed tasks, validation entry points, and static reproduction meta-graphs. It is not the runtime identity, provenance authority, image builder, distribution artifact, or durable scheduler state.
- A static Pixi graph may compose reproduction leaves. Each result-changing leaf must produce explicit DataLad/BABS evidence, and Pixi input/output caching must never skip it.
- Prefer and qualify an existing exact SIF from a pinned ReproNim/containers dataset. Build a custom SIF only when no existing image meets the required version, interface, architecture, terms, fixtures, and target-host behavior.
- Candidate SIF payload is disposable. A result can be retained only after the exact qualified SIF is registered and persistently retrievable, and the result is regenerated through BABS or DataLad Containers.
- An accepted SIF does not provide execution isolation by itself. Result-producing execution uses fresh transient state, no host home or persistent cache, no retrieval credentials, explicit read-only inputs/configuration, declared writable output/scratch, and recorded host interfaces.
- BABS owns participant/session fan-out and Slurm execution. Pin and qualify the direct Study-layout revision, preserve one attempt dataset identity through finalization and acceptance, and record lifecycle meta-provenance separately.
- `result-manifest.tsv` links each retained result to its exact inputs, accepted SIF, run record, dataset state, and executable reproduction entry point.
