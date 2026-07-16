# Documentation map

This directory now separates current design decisions from the evidence that led to them.

## Canonical guidance

- [Conversion plan](conversion-plan.md) — staged migration to `stamped_dl_morphometrics_biases`.
- [Architecture and provenance](architecture-and-provenance.md) — responsibility boundaries, repository layout, campaign matrix, container identity, and operations records.
- [Poster-derived result targets](result-targets.md) — the result set that defines “good enough” scientific reconstruction.
- [Why Pixi](pixi-in-stamped.md) — the positive case for Pixi, alternatives, and the limits imposed by STAMPED.
- [Pixi tasks and DataLad provenance](pixi-tasks-datalad-provenance.md) — the decision record for task indirection, current BABS behavior, and Austin Macdonald’s proposed BABS work.
- [Repository-local agent skill](skills/stamped-neuroimaging-analysis/SKILL.md) — concise operating rules for agents working on the converted analysis.

## Baseline evidence

- [Original STAMPED audit](STAMPED_REVIEW_DL_MORPHOMETRICS_BIASES.md) — detailed review of the pre-conversion notebooks and scripts. It describes the starting point; the conversion plan supersedes its proposed layout and unresolved-question list.
- [STAMPED principles paper](../Macdonald%20et%20al.%20-%20STAMPED%20principles%20for%20reproducible%20research%20objects.pdf) — normative source.
- [OHBM 2025 poster](../recon_all_recon_any_poster_ohbm2025.pdf) — scientific reconstruction target.

## Decision summary

- Pixi defines local named environments and provides reviewed lock inputs for image builds.
- A tracked SIF, not `pixi.lock`, is the runtime identity attached to a scientific run.
- DataLad owns data identity, command provenance, and replay.
- BABS owns BIDS App expansion, Slurm execution, auditing, and merge.
- Pixi tasks are limited to development and read-only/meta-level aids. They never create or modify scientific results and never use task dependencies.
- The current BABS wrapper and zipped result pattern is accepted with documented indirection. Unreleased BABS improvements are not required.
- No Slurm scientific job runs without an exact SIF already registered in a DataLad container dataset and available from persistent storage.
- `ds007116` is the engineering pilot. ABCD is the later controlled-data campaign and the basis for the poster-level age analyses.
