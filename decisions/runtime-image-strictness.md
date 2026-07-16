# Candidate, qualified, and accepted SIF policy

- **Status:** accepted with an explicit debugging exception
- **Decision date:** 2026-07-16
- **Last reviewed:** 2026-07-16

## Question

Must every local or HPC invocation use a SIF that has already been qualified, registered, and made persistently retrievable, or would that make image development and cluster debugging unnecessarily expensive?

## Normative status

This is an implementation decision supporting STAMPED identity, provenance, actionability, portability, and distribution requirements. Candidate execution is permitted, not required.

Signing, signature verification, SBOMs, build attestations, and separately signing an intentionally distributed OCI artifact are hardening choices. Consider and record each relevant decision; none is a blanket condition for STAMPED conformance or SIF acceptance.

## Decision

Unqualified or unregistered candidate SIFs may be used for local and cluster debugging. Their scientific payload is disposable, must remain outside accepted derivative datasets and `result-manifest.tsv`, and cannot support a scientific conclusion.

Before an output is retained, merged, used for a scientific comparison or decision, or released, the exact SIF bytes must:

1. pass the declared qualification checks;
2. be registered under `envs/containers/accepted/`;
3. be persistently retrievable from declared storage; and
4. be identified by the accepted container dataset commit, annex key or equivalent content identity, and checksum.

The intended output must then be generated anew through BABS or `datalad containers-run`. Registering an image after an execution never promotes that execution or its output.

An accepted SIF makes a runtime eligible for result-producing execution. It is not sufficient by itself: an authoritative execution and result must also identify exact inputs, tracked configuration, the resolved command, the DataLad/BABS run record, and the resulting dataset state, and must satisfy the execution-isolation decision.

## States

| State | Meaning | Permitted use |
|---|---|---|
| Candidate | Exact bytes have not completed qualification and acceptance | Disposable testing and debugging only |
| Qualified | Exact bytes pass application/version, interface, architecture, terms, fixture, and target-host checks | Eligible for registration; not yet eligible to produce retained results |
| Accepted/registered | Qualified bytes are registered and persistently retrievable with exact identities | Eligible for result-producing execution |
| Authoritative execution/result | Generated after acceptance with exact inputs, configuration, command, isolated execution, and DataLad/BABS provenance | Retention, comparison, figures, tables, and release |

“Used for a scientific comparison or decision” includes using a candidate output to choose parameters, include or exclude participants, select a model, or make a claim. Candidate evidence may suggest a question, but it cannot be the sole basis for a retained choice. Document the choice and re-establish it using an accepted-runtime execution before it affects retained analysis.

## Qualification and acceptance

1. Acquire an exact SIF from a pinned source container dataset or build a candidate from pinned or checksummed inputs.
2. Qualify its application and version, command-line or BIDS App interface, architecture, licence or terms, project fixtures, and representative target host.
3. Record the source dataset commit and annex identity for an acquired image, or the definition, lock, non-Pixi inputs, build command, builder versions, and platform for a custom image. In both cases, record tests, SIF SHA-256, and annex key.
4. Register the exact qualified bytes under `envs/containers/accepted/` with `datalad containers-add` and publish the annex content to persistent storage.
5. Point BABS or the direct DataLad Containers operation to the accepted image and generate the intended output anew.
6. Record the exact SIF, input, configuration, command, run, and output dataset identities, and map every retained result in `result-manifest.tsv`.
7. Delete or quarantine candidate scientific payload. Retain diagnostic logs only when they are clearly marked non-authoritative and excluded from accepted derivatives and the result manifest.

An optional `envs/images.lock.yaml` may index these identities, but it is not the authority. Its entries must resolve to the accepted container dataset and exact SIF content.

## Debugging safeguards

- Put candidate scientific payload in ignored scratch, disposable Slurm workspaces, or explicitly disposable branches that are never accepted or merged into a derivative.
- Mark candidate jobs and image filenames clearly and never reuse an accepted campaign identity.
- Make accepted output paths writable only through the project's BABS/DataLad entry points where practical.
- Do not require persistent registration, signing, or an SBOM for each failed candidate build.
- Never use a mutable tag, definition file, Pixi lock, local cache, or build recipe as a substitute for exact accepted SIF content.
- Apply the separate [disposable and isolated scientific execution](runtime-execution-isolation.md) controls to every result-producing run and as many candidate tests as practical.

## Why registration precedes retained execution

A debug execution may lack a DataLad run record, exact input declarations, stable image retrieval, tracked configuration, or isolation evidence. Registration performed afterward cannot add those facts retroactively. Re-execution after acceptance creates one auditable path and prevents accidental promotion of an opaque artifact.

## Acceptance criteria

- Every `result-manifest.tsv` entry resolves through a DataLad/BABS record to an accepted exact SIF, exact inputs, tracked configuration, and executable command.
- Every accepted SIF has qualification evidence, content identity, accepted-container dataset commit, and persistent retrieval evidence.
- No candidate scientific payload occurs in an accepted derivative or result manifest.
- Retained candidate diagnostic logs are clearly non-authoritative and cannot be mistaken for scientific results.
- Any scientific choice prompted by candidate output is documented and re-established using an accepted-runtime execution.
- Clean verification retrieves the exact SIF without relying on a developer workstation, mutable tag, or cluster cache.
