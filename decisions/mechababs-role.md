# Role of MechaBABS in this analysis

- **Status:** accepted as a pinned design reference and optional experimental pilot; deferred for authoritative execution
- **Decision date:** 2026-07-16
- **Reviewed snapshot:** [`asmacdo/mechababs@4a4deb8` (`v0.0.1`)](https://github.com/asmacdo/mechababs/tree/4a4deb8f01c8837a5481d497140a3bb41c450f09)

## Decision

Include MechaBABS in the project’s design references and evaluate it through a disposable, one-participant `ds007116` pilot. Do not initially make it a runtime dependency of any campaign whose outputs may be retained, merged into an authoritative derivative, compared scientifically, or used in a result.

Record its repository URL and exact reviewed commit in the component inventory. If the optional pilot proceeds, install that commit as a DataLad subdataset under the pilot operations dataset rather than copying its code or resolving a moving branch. Keep the pilot on a disposable branch and in a distinct attempt whose outputs cannot be composed into a scientific Study derivative.

Adopt the useful campaign-management patterns listed below in the repository design from the outset. Keep Pixi, tracked SIFs, DataLad/DataLad Containers, BABS, and the project operations ledger authoritative. Promotion of MechaBABS to the execution stack requires all promotion gates in this record to pass and a revision of this decision.

This is not a rejection of MechaBABS. It reflects that its architecture is promising while its current implementation, upstream dependencies, environment model, and target-pipeline coverage are not yet stable enough to mediate authoritative execution.

## What MechaBABS contributes

MechaBABS is automation around BABS, not a replacement for BABS or DataLad. It composes three independent axes—dataset, pipeline, and cluster—into the configuration consumed by `babs init` ([README](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/README.md#L12-L25)). Its unit of work is a campaign DataLad dataset containing configuration, a state ledger, pinned BABS/MechaBABS code, container datasets, inputs, and BABS projects ([campaign description](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/README.md#L27-L61)).

The reconciler advances each dataset × pipeline cell by at most one scaffold, submit/wait, or merge transition per tick ([state model](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/README.md#L122-L144)). Scientific participant execution and its run record remain owned by the selected BABS version.

## Patterns adopted now

- **Dataset × pipeline × cluster separation.** Use independent, reviewed descriptions for input datasets, BIDS Apps/runtime images, and Slurm hosts. This maps naturally onto `ds007116` and ABCD; the reconstruction variants and input conditions; and multiple Slurm systems.
- **Explicit attempts.** Allocate a new attempt identity when configuration or runtime identity changes instead of overwriting an earlier campaign. MechaBABS implements this with `attempt-N` ([implementation](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/mechababs/iterate.py#L109-L118)); this project will also encode pipeline/runtime and input-condition identity in campaign metadata.
- **Pinned causal inputs.** Track BABS, project code, application definitions, and container datasets at exact commits. Refuse authoritative execution when a working checkout differs from the recorded pin, following MechaBABS’s dirty-pin guard ([guard.py](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/mechababs/guard.py#L1-L44)).
- **Requested and realized inclusion.** Preserve both the intended cohort and BABS’s realized intersection with available participants/sessions ([inclusion record](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/mechababs/iterate.py#L202-L220)). Do not conflate eligibility, data availability, DUC restrictions, execution failure, and final exclusion.
- **Small state transitions.** Prefer inspectable and resumable lifecycle transitions over a monolithic campaign script. Treat an operational ledger as a cache/view over BABS, scheduler, RIA, and DataLad state rather than as scientific provenance.
- **Development exercises production paths.** Use small inclusions and inexpensive fixtures through the same interfaces used for full campaigns. Adapt the SimBIDS local-Slurm and real-cluster test-rung pattern ([end-to-end test](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/tests/e2e/test_full_run.py#L53-L113)).

## Patterns not adopted

- Do not use MechaBABS’s `bootstrap.sh`, UV-created campaign `.venv`, editable installation, or unbounded dependency resolution. It constructs a second environment authority and enforces the campaign venv at runtime ([bootstrap](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/bootstrap.sh#L82-L114)). This project uses the reviewed Pixi manifest/lock for orchestration and locked environments inside tracked SIF builds.
- Do not copy its temporary ReproNim container shim. Current MechaBABS re-registers images at BABS’s expected path pending [BABS #383](https://github.com/PennLINC/babs/issues/383), as tracked in [MechaBABS #62](https://github.com/asmacdo/mechababs/issues/62). Use ReproNim SIFs directly where available and register every accepted upstream or project-built SIF in the project’s DataLad container dataset.
- Do not copy its current selection implementation. Eligibility is hard-coded to MRIQC and fMRIPrep ([select.py](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/mechababs/select.py#L72-L115)); it does not define recon-all, recon-any, recon-all-clinical, protected ABCD retrieval, or downstream-analysis targets.
- Do not require study configuration to live inside vendored MechaBABS code. Project-owned pipeline, image, cohort, and host configuration must remain first-class tracked inputs in this repository.
- Do not treat `mechababs iterate`, its state ledger, task definitions, or Git effects as proof that a particular lifecycle command ran.
- Do not make MechaBABS’s outer campaign layout the BIDS authority. Operational aggregation and BIDS Study organization have different responsibilities.

## Provenance interaction

MechaBABS does not remove the distinction between scientific provenance and BABS lifecycle meta-provenance:

- BABS/DataLad own participant-level scientific run records and the exact application command associated with each result.
- MechaBABS invokes `babs init`, `submit`, and `merge` from Python; Git generally captures their effects rather than the expanded lifecycle invocation. Its explicit `datalad run` currently records the inclusion copy, not the complete campaign lifecycle.
- The project must therefore retain its parameterized Pixi tasks and `operations/<campaign>/commands.jsonl` policy for BABS/MechaBABS operations. Record expanded argv, versions and pins, working directory, configuration hashes, status, scheduler identifiers, and before/after commits.
- A direct `datalad run -- mechababs iterate ...` is not the same anti-pattern as `datalad run -- pixi run <task>`: it records the actual orchestrator and arguments and may be useful meta-provenance when its state inputs and outputs can be declared. It still does not record the inner participant command and cannot replace the BABS/DataLad scientific record or operations ledger. Do not put a Pixi task name between DataLad and MechaBABS.
- A Pixi task may invoke a MechaBABS lifecycle operation during a pilot, but the task provides actionability only. The operations ledger records the invocation; BABS/DataLad records the scientific work.

These are recognized gaps upstream: campaign construction and environment provenance remain open in [#57](https://github.com/asmacdo/mechababs/issues/57), produced BABS projects are not yet saved into the campaign superdataset in [#52](https://github.com/asmacdo/mechababs/issues/52), portable run records remain open in [#29](https://github.com/asmacdo/mechababs/issues/29), and the overall provenance readiness decision remains open in [#71](https://github.com/asmacdo/mechababs/issues/71).

## BIDS Study and campaign boundaries

MechaBABS proposes a useful inner “BABS study” target: one raw BIDS dataset under `sourcedata/raw/` and its generated derivative under `derivatives/`. If fully realized and described, that would follow the pinned [BIDS 1.11.1 Study dataset](https://bids-specification.readthedocs.io/en/v1.11.1/common-principles.html#study-dataset) pattern closely. It is not current compliance: study composition and tracked derivative integration remain open in [#4](https://github.com/asmacdo/mechababs/issues/4) and [#24](https://github.com/asmacdo/mechababs/issues/24).

The proposed outer MechaBABS campaign is different. It labels the campaign a BIDS “super-study” and places child BABS-study datasets beneath `campaign/derivatives/` ([target structure](https://github.com/asmacdo/mechababs/blob/4a4deb8f01c8837a5481d497140a3bb41c450f09/docs/output_structure.md#L16-L48)). This is an operational hierarchy specific to MechaBABS, not the canonical BIDS Study example, and it is still described as a target with current gaps tracked in [#4](https://github.com/asmacdo/mechababs/issues/4).

For this project:

- the top-level research object is a DataLad superdataset and is not automatically a BIDS Study dataset;
- `ds007116` and ABCD each receive a separately described Study dataset;
- each Study dataset composes its raw BIDS dataset and independently described derivative datasets;
- BABS attempts, RIA stores, scheduler state, and lifecycle records remain operational objects unless they independently satisfy a declared BIDS role;
- accepted outputs from processing attempts are composed into the appropriate study derivative rather than making the campaign ledger or project directory the BIDS authority.

## Benefits and costs

| Benefits | Costs and risks |
|---|---|
| Provides a concrete campaign matrix for many datasets, pipelines, attempts, and clusters | Adds another orchestration/state layer whose invocation also needs meta-provenance |
| Encodes restartable BABS lifecycle transitions and explicit attempts | Current ledger routing is not a complete reconstruction of state from DataLad/RIA ground truth |
| Pins code/container datasets and guards causal consistency | Current UV/`.venv` design conflicts with the project’s Pixi environment authority |
| Separates pipeline requirements from host configuration | Existing configurations contain project-specific assumptions and portability gaps |
| Provides a real SimBIDS submit-to-merge test pattern | No repository CI workflow currently runs the tests; core CLI coverage remains open in [#50](https://github.com/asmacdo/mechababs/issues/50) |
| Engages directly with BIDS Study and ReproNim container organization | Its BIDS-study end-to-end path currently relies on open [BABS PR #369](https://github.com/PennLINC/babs/pull/369), and direct ReproNim layout support is unresolved |
| Could eventually reduce hand-managed repetition across the four reconstruction families | Target reconstruction BIDS Apps, configurable eligibility, allowable targets, and protected inputs are not implemented; generic chained inputs also remain open in [#72](https://github.com/asmacdo/mechababs/issues/72) but are outside the initial promoted role |

At the reviewed snapshot, MechaBABS has a `v0.0.1` tag but no GitHub release or published package, no GitHub Actions workflow, one SimBIDS happy-path end-to-end test with manual prerequisites, and limited unit coverage. The existing 15 unit tests for status decisions and pin guarding passed during this review, but [#50](https://github.com/asmacdo/mechababs/issues/50) confirms that the campaign CLI core remains largely uncovered. The supplied fMRIPrep configurations cannot currently be configured because required `short_name` fields are absent ([#69](https://github.com/asmacdo/mechababs/issues/69)).

## Promotion gates

The possible promoted role is deliberately narrow: coordinate BABS reconstruction campaigns across dataset × pipeline × cluster combinations. Project-authored group analysis BIDS Apps remain direct DataLad-container operations and are not a MechaBABS requirement.

Promote MechaBABS to that authoritative reconstruction-orchestration role only when all of the following are demonstrated at a new pinned snapshot:

1. The exact required BABS behavior is present in a pinned release or explicitly accepted commit; authoritative execution does not silently depend on a moving BABS or MechaBABS branch.
2. The selected campaign/Study hierarchy matches this repository’s organization and validates against pinned BIDS 1.11.1 with reviewed metadata responsibilities.
3. ReproNim and project-built SIFs are consumed as exact registered DataLad container content without the temporary sibling shim.
4. MechaBABS can run from the project’s reviewed Pixi orchestration environment without constructing an independent UV environment or weakening its causal pin guard.
5. Pipeline configuration supports recon-all 7.4.1, recon-all 8.2.0, recon-any, and recon-all-clinical, including versioned identities and allowable participant operations.
6. Cohort/eligibility policy is configurable and tested for both `ds007116` and controlled ABCD inputs; requested, realized, failed, and excluded sets remain distinguishable.
7. Every state-changing MechaBABS/BABS lifecycle invocation is entered in the project operations ledger, while every scientific participant result retains an intelligible BABS/DataLad record linked to the exact registered SIF.
8. Produced BABS projects and accepted derivatives are saved as intended DataLad subdatasets; no retained result depends only on a cluster-local RIA store or ledger flag.
9. Re-execution records contain no undisclosed host-specific absolute paths, ambient environments, mutable image tags, or untracked credentials/licenses.
10. Unit tests cover configuration, selection, state, construction, iteration, failure, retry, and merge behavior; automated end-to-end tests cover the target pipeline interface and at least one supported Slurm profile.
11. Failure, partial-merge, retry, licensing, BIDS validation, and schema/version migration policies are resolved for the adopted feature set.
12. A disposable one-participant `ds007116` pilot completes init → setup check → submit → status/retry → merge and representative replay using the exact registered SIF, without producing any authoritative output during the evaluation.

Until then, changes in MechaBABS should be monitored and useful patterns may be contributed upstream, but the authoritative execution stack remains Pixi + tracked SIFs + DataLad/DataLad Containers + BABS.
