# Candidate versus authoritative SIF policy

- **Status:** accepted with an explicit debugging exception
- **Decision date:** 2026-07-16

## Question

Must every local or HPC invocation use a SIF that was already registered and persistently retrievable, or would that make image development and cluster debugging unnecessarily expensive?

## Decision

Unregistered candidate SIFs may be used for debugging locally or on Slurm. Their outputs are disposable and cannot enter the retained research object or support a scientific conclusion.

Every retained, merged, scientifically compared, or released output must be regenerated through BABS or `datalad containers-run` using the exact SIF already registered in the DataLad container dataset and available from declared storage.

This separates fast failure discovery from authoritative execution without weakening the lineage of the research object.

## Image and output states

| State | Image requirement | Permitted output use |
|---|---|---|
| Local development | Candidate SIF or local Pixi environment | Tests, interactive inspection, and disposable debug files only |
| Slurm debugging | Candidate SIF allowed from project/scratch storage | Scheduler, bind, resource, and smoke-test debugging only; never merge or retain |
| Authoritative pilot | Exact SIF registered before execution; run through BABS/DataLad | May be retained if all provenance and validation gates pass |
| Scaled/release run | Same strictness plus durable retrieval, verification, and release checks | Scientific comparison, figures, tables, and release |

“Scientifically compared” includes using a candidate output to choose parameters, include/exclude participants, select a model, or make a claim. If a debug observation influences a scientific decision, record the decision and confirm it with a strict run.

## Promotion procedure

When a candidate is ready:

1. Stop writing to its debug output location.
2. Hash and smoke-test the exact SIF bytes.
3. Register that SIF with `datalad containers-add` and publish the annex content to a declared durable location.
4. Record the definition/lock hashes, non-Pixi inputs, build command, tool versions, architecture, SIF SHA-256, annex key, and DataLad commit.
5. Point the BABS campaign or `datalad containers-run` step at the registered image.
6. Regenerate the intended output. Do not promote an earlier unrecorded debug output, even when the SIF bytes are identical.
7. Delete or quarantine debug outputs so they cannot be mistaken for authoritative data.

The rerun is necessary because registration after execution cannot retroactively make the original command, inputs, outputs, and image dependency part of a DataLad/BABS provenance record.

## Practical safeguards

- Put debug outputs in an ignored `scratch/` path, disposable Slurm workspace, or explicitly disposable branch that is never merged into a derivative dataset.
- Mark candidate jobs and image filenames clearly; never reuse an authoritative campaign identifier.
- Make authoritative output paths writable only through the project’s BABS/DataLad entry points where practical.
- Do not require registry publication, signing, or an SBOM before each failed candidate build. Require exact registration and durable retrieval before an authoritative run, and complete signature/SBOM/release gates before release.
- A mutable tag, definition file, Pixi lock, or build cache is not a substitute for the exact SIF once an output is retained.

## Why not require registration for every debug attempt?

Image development often fails on package installation, bind paths, licenses, architecture, memory, or scheduler integration. Annexing and publishing every failed candidate adds persistent noise without improving the provenance of a result. The strict boundary belongs at retention and scientific use.

## Why not retain a successful debug output?

The research object must explain its own outputs. A successful debug job may lack a DataLad run record, exact input declarations, stable image retrieval, or the reviewed campaign configuration. Re-execution after registration creates one auditable path and prevents accidental promotion of an opaque artifact.

## Acceptance criteria

- No debug output is present in an authoritative derivative dataset or result manifest.
- Every retained output was produced after its SIF was registered and is linked to that SIF by a DataLad/BABS record.
- Candidate and authoritative output locations and campaign names cannot be confused.
- A candidate that affects a scientific choice is followed by a documented decision and strict confirmation run.
- Release verification retrieves the exact SIF without relying on the developer’s workstation or cluster cache.
