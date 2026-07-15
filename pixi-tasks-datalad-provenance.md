# Pixi tasks, DataLad provenance, and reproducible replay

**Assessment date:** 2026-07-15  
**Target project:** `dl_morphometrics_biases/`  
**Question:** should result-producing commands be invoked through Pixi tasks or Make targets when DataLad `run` and `rerun` are the provenance authority?

## Executive conclusion

The concern is valid. DataLad records the command string it is given. If it is given:

```bash
pixi run --locked analyze
```

then the run record contains that task invocation, not the command currently stored under `analyze` in `pixi.toml`. The same limitation applies to:

```bash
make analyze
```

Both remain useful executable interfaces, but neither improves DataLad provenance. They add a level of indirection between the run record and the scientific command. That indirection has two distinct costs:

1. **Interpretability:** a reviewer must retrieve the historical task or Make recipe before knowing which program and arguments were invoked.
2. **Replay semantics:** `datalad rerun` re-executes the recorded string in the tree onto which it is replaying. If that tree contains a newer task definition, the same recorded alias can now mean a different command.

This does not make Pixi unsuitable. It means Pixi should have two deliberately separated roles:

- **Environment authority:** Pixi selects and realizes the locked software environment.
- **Developer convenience:** Pixi tasks may provide linting, formatting, tests, environment checks, container builds, and memorable interactive shortcuts.

For result-producing transformations, DataLad should record the explicit scientific command. Pixi can still activate the environment without resolving a task alias:

```bash
datalad run \
  -m "Extract FreeSurfer statistics" \
  -i envs/pixi.toml \
  -i envs/pixi.lock \
  -i scripts/fsstats_extraction.py \
  -i config/analysis.toml \
  -i 'inputs/freesurfer/**' \
  -o 'outputs/stats/**' \
  -- \
  pixi run --no-config --locked -e extract --executable \
    python scripts/fsstats_extraction.py \
      --config config/analysis.toml \
      --input inputs/freesurfer \
      --output outputs/stats
```

The run record now exposes the program, arguments, paths, Pixi environment, and locked-execution policy. It still relies on versioned code—as every script invocation does—but it no longer hides scientific parameters behind a mutable task name.

The BABS work by Austin Macdonald directly recognizes the same problem. [PennLINC/babs issue #328](https://github.com/PennLINC/babs/issues/328) proposes replacing a `datalad run` of a generated wrapper script with `datalad containers-run` of the explicit BIDS App command, followed by a separate provenance record for zipping. A working branch implements much of that design, but the change is not yet present in the fork's main branch or its current `working-branch-v4`, and its design document lists unresolved compatibility and pipeline questions. It should therefore be treated as promising work in progress, not as a provenance property of released BABS.

## 1. The four layers should not be conflated

| Layer | Primary question | Appropriate authority |
|---|---|---|
| Environment | Which executable and libraries are available? | `envs/pixi.toml`, `envs/pixi.lock`, container/SIF identity |
| Command provenance | What command and parameters produced this result? | DataLad `run`/`containers-run` record |
| Workflow/actionability | What steps exist, and how can a person invoke them conveniently? | Pixi tasks, Make, documentation, or BABS |
| Scalable orchestration | Which subjects run where, with which resources and retry/audit behavior? | BABS and the scheduler |

Pixi tasks and Make targets operate primarily at the workflow/actionability layer. DataLad operates at the command-provenance layer. A task runner can call DataLad, or DataLad can call a task runner, but the direction determines what the history reveals.

The preferred rule is:

> The provenance system should receive the most explicit stable command that remains meaningful to a scientific reviewer. Convenience aliases should sit outside that boundary.

## 2. What DataLad actually records

The [DataLad `run` documentation](https://docs.datalad.org/en/latest/generated/man/datalad-run.html) describes it as executing an arbitrary shell command and recording its impact on a dataset. Its structured record includes the command string, declared inputs and outputs, dataset identity, and relative working directory. `datalad rerun` later retrieves that recorded command and executes it again.

DataLad does not ask the shell, Make, or Pixi for a semantic expansion of the command. Consequently, these records have materially different transparency:

| Command passed to DataLad | Immediately visible in the run record | Additional historical resolution required |
|---|---|---|
| `python scripts/analyze.py --input A --output B --method C` | Program, script, paths, and scientific arguments | Script implementation and environment |
| `pixi run --locked -e analysis --executable python scripts/analyze.py ...` | Environment selector plus the explicit scientific command | Script implementation, manifest, and lock |
| `pixi run --locked analyze` | Environment wrapper and task name | Historical task definition, its dependency tasks, templating, and then the script implementation |
| `make analyze` | Make target | Historical Make recipe, prerequisites, Make's decision about what was stale, and then invoked scripts |
| `bash code/run-analysis.sh` | Wrapper path | Historical wrapper contents and then invoked scripts |

Some indirection is unavoidable. A command such as `python scripts/analyze.py` still points to source code. The useful boundary is not “put the complete implementation in the commit message.” It is “make the scientific entry point and its parameterization visible without requiring a reviewer to reconstruct an orchestration graph.”

### Historical interpretability is possible, but it is work

If run commit `R` records `pixi run analyze`, a reviewer can usually inspect the task definition from the state immediately before execution:

```bash
git show R^:envs/pixi.toml
```

The reviewer may then need to follow `depends-on`, feature-specific tasks, environment selection, MiniJinja arguments, scripts, and configuration files. Git makes this investigation possible; it does not make it immediate.

That distinction matters for STAMPED Tracking. “The information can be reconstructed by a capable investigator” is weaker than “the generating command is directly stated in machine-readable provenance.”

## 3. What `datalad rerun` guarantees—and what it does not

The [DataLad `rerun` documentation](https://docs.datalad.org/en/latest/generated/man/datalad-rerun.html) says that rerun re-executes the recorded command in the recorded path. By default, the selected command is executed onto the current `HEAD`; `--onto` and range replay can choose a historical starting state.

This creates two valid but different operations:

### 3.1 Historical verification

The goal is to reproduce the original computation with its historical code, manifest, lock, and task definitions. Replay should begin from the original historical base or reconstruct the relevant history in order. For example, after reviewing the report:

```bash
datalad rerun --report R
datalad rerun --onto=R^ --branch=verify-R R
```

For a multi-step history, range replay is usually safer because non-run commits that changed code or task definitions can be applied in sequence:

```bash
datalad rerun --report --since=BASE REVISION
datalad rerun --onto=BASE --branch=verify --since=BASE REVISION
```

Subdatasets require additional care: DataLad documents that `--onto` changes the current dataset's working tree but does not itself reset every subdataset worktree.

### 3.2 Forward recomputation

The goal is to apply an earlier command to current code, current parameters, or a current environment. Running:

```bash
datalad rerun R
```

at a newer `HEAD` intentionally replays the recorded string in the newer tree. If the record is `pixi run analyze`, it resolves the current `analyze` task. If the record contains the explicit Python command, the arguments remain visible and stable, but the referenced script can still have changed.

These are both useful workflows. They should not both be called “run the exact historical computation.” Historical verification requires a historical tree and historical environment identity; adaptive recomputation intentionally does not.

### 3.3 `rerun --script` cannot remove hidden indirection

`datalad rerun --script=- R` is valuable for inspection because it extracts the recorded commands without executing them. However, if the record contains `pixi run analyze`, the generated script contains the same alias. DataLad cannot recover an expanded command it was never given.

## 4. What Make and Pixi tasks add

The local [From Script to STAMPED Research Object](stamped-examples/content/examples/stamped-awk-evolution.md) example correctly demonstrates that a Makefile adds an executable dependency graph and a memorable reproduction entry point. Its claim that Git tracks the Makefile is also correct. However, a Makefile is a versioned prospective specification: it says what should happen when Make evaluates the current graph. It is not, by itself, a retrospective record of what actually ran.

The more extensive [stellar-distance walkthrough](stamped-examples/content/examples/stellar-distance-walkthrough.md) implicitly shows the division:

- Make supplies the convenient `make`/`make test` interface and dependency scheduling.
- Separate `datalad run` calls record the fetch and analysis commands.
- `datalad rerun` replays those recorded DataLad commands.

The phrase “Make dependencies are a lightweight form of provenance” should be interpreted narrowly: they document an intended dependency graph. They do not record execution-specific commands, resolved variables, environment identity, or whether a recipe was skipped as already up to date.

The local [`datalad run` provenance example](stamped-examples/content/examples/datalad-run-provenance.md) gives the stronger rule: keep the recorded command self-contained and declare inputs and outputs explicitly. A mutable task alias is not self-contained in that sense, even when its definition is recoverable from Git.

Pixi tasks add somewhat different value:

- selection of a named Pixi environment;
- a cross-platform task shell;
- typed arguments and templating;
- task dependencies;
- a convenient catalog of developer operations;
- optional input/output-based task skipping.

None of these is DataLad provenance. Pixi's cache makes the boundary especially important: when task `inputs` or `outputs` are declared, Pixi may skip the task if its fingerprints say the result is reusable. The [Pixi task documentation](https://pixi.prefix.dev/latest/workspace/advanced_tasks/) defines this as a performance cache. A provenance-producing DataLad call should not depend on an inner task runner deciding that no command needs to execute.

Task dependency graphs add another loss of resolution. A single record such as `pixi run reproduce` can trigger extraction, analysis, and figures through `depends-on`, yet DataLad sees one alias and may create only one aggregate result commit. Separate explicit DataLad activities preserve the command, inputs, outputs, and failure boundary of each scientific step.

### Comparative value

| Property | Make target | Pixi task | DataLad run record |
|---|---:|---:|---:|
| Memorable user entry point | Yes | Yes | Possible, but verbose |
| Dependency/task composition | Strong file graph | Lightweight task graph | Ordered provenance records, not a general scheduler |
| Environment realization | No | Yes | No |
| Records what actually ran | No | No | Records the command it was given |
| Directly declares run inputs/outputs | Make prerequisites/targets | Cache-oriented inputs/outputs | Yes, as provenance and data preparation |
| Re-executable historical record | Only by resolving historical Makefile | Only by resolving historical task | Yes, subject to tree/environment state |
| Subject/session HPC orchestration | No | No | No; BABS adds this layer |

## 5. Recommended policy for the morphometric-bias analysis

### 5.1 Keep result-producing commands out of Pixi tasks

Do not use these as the recorded provenance boundary:

```bash
datalad run -- pixi run --locked extract-stats
datalad run -- pixi run --locked make-figures
datalad run -- make figures
```

Instead, pass the explicit command through Pixi's environment without task resolution:

```bash
datalad run \
  -m "Create morphometric bias figures" \
  -i envs/pixi.toml \
  -i envs/pixi.lock \
  -i scripts/make_figures.py \
  -i config/analysis.toml \
  -i outputs/stats/statistics.parquet \
  -o 'outputs/figures/**' \
  -- \
  pixi run --no-config --locked -e analysis --executable \
    python scripts/make_figures.py \
      --config config/analysis.toml \
      --statistics outputs/stats/statistics.parquet \
      --output-dir outputs/figures
```

`--executable` tells Pixi to execute the supplied command instead of treating it as a task name. The DataLad record therefore remains explicit while Pixi supplies the selected locked environment.

### 5.2 Keep Pixi tasks for non-provenance conveniences

Good candidates include:

```toml
[tasks]
lint = { cmd = "ruff check .", default-environment = "dev" }
format-check = { cmd = "ruff format --check .", default-environment = "dev" }
unit-tests = { cmd = "pytest -q", default-environment = "dev" }
environment-report = { cmd = "python scripts/report_environment.py", default-environment = "dev" }
```

A task may also print or dry-run a recommended DataLad invocation for a user. It should not silently become the only place where scientific arguments are stored.

### 5.3 If a convenience task launches DataLad, put DataLad inside the task

For interactive convenience, this direction is better:

```toml
[tasks]
record-figures = "datalad run ... -- pixi run --locked -e analysis --executable python scripts/make_figures.py ..."
```

The user types `pixi run record-figures`, but the committed run record contains the explicit inner command supplied to DataLad. The task remains a convenience launcher, not the provenance record.

This pattern has maintenance duplication and quoting costs, so a small reviewed script that constructs the `datalad run` call may be easier to test. If such a launcher is used, its purpose is to generate an explicit DataLad record; the recorded result-producing command must still be inspected in tests.

### 5.4 Do not use Pixi task caching for scientific run tasks

Avoid `inputs`/`outputs` cache declarations on any Pixi task that can sit inside `datalad run` or `datalad rerun`. DataLad should prepare inputs/outputs and decide whether a provenance commit is created. An inner cache hit can otherwise turn a requested recomputation into a no-op.

### 5.5 Test the provenance record, not only the output

For each result-producing entry point, an integration test should:

1. execute the DataLad-recorded operation on a small fixture;
2. inspect the resulting run record;
3. assert that it contains the explicit program and critical arguments rather than only a Pixi task/Make target;
4. run `datalad rerun --report`;
5. replay from the historical parent on a verification branch;
6. compare output checksums;
7. confirm that the expected Pixi lock or container identity is reachable from the run commit.

## 6. Austin's BABS work

### 6.1 The current indirection

The fork's main branch currently generates a participant job that calls approximately:

```bash
datalad run \
  ... \
  "bash ./code/CONTAINER_zip.sh SUBID SESID"
```

The historical generated script is tracked and can be inspected, so the full invocation is recoverable. But the run record itself points to a wrapper that combines the BIDS App invocation, output handling, and zipping. This is the same interpretability concern as recording a Make target or Pixi task.

### 6.2 Issue #328 explicitly proposes removing that layer

Austin opened [BABS issue #328](https://github.com/PennLINC/babs/issues/328) on 2026-01-23. It initially motivates `datalad containers-run` through portable container discovery, but its “Going further: cleaner provenance” section states the stronger design:

1. use `datalad containers-run` for the explicit BIDS App invocation;
2. create a separate `datalad run` record for zipping;
3. remove the generated wrapper that currently performs the heavy lifting.

That proposal directly addresses both concerns in this report:

- the run record exposes BIDS App arguments rather than only a wrapper name;
- processing and packaging become separate provenance activities.

`datalad containers-run` is particularly appropriate because its [official documentation](https://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-run.html) describes it as a drop-in replacement for `run` that generates the container command and records the container image as an input dependency.

### 6.3 What the branch implements

The [`add-containers-run-v2` design document](https://github.com/asmacdo/babs/blob/add-containers-run-v2/design/containers-run.md) and [`participant_job.sh.jinja2`](https://github.com/asmacdo/babs/blob/add-containers-run-v2/babs/templates/participant_job.sh.jinja2) implement a three-stage job:

1. `datalad containers-run` records the BIDS App command, explicit inputs, output directory, container name, participant/session arguments, and configured BIDS App arguments.
2. `datalad run` records the separate zip script and zip outputs.
3. a separate Git commit removes raw outputs because the branch retained a workaround for DataLad deletion handling.

This is a meaningful provenance improvement. In particular, the BIDS App arguments are present directly after the `--` delimiter in the `containers-run` call instead of being hidden inside the generated zip/run script.

The key implementation commit is [`dd8b44b`](https://github.com/asmacdo/babs/commit/dd8b44b) (“Replace singularity run with datalad containers-run in job template”); the reviewed `add-containers-run-v2` branch head was `5f4103e5448304dd6748788635c0c6292d4be3b7` on 2026-07-15.

### 6.4 Status and limits as of this assessment

The implementation should not yet be assumed available in BABS:

- [Issue #328](https://github.com/PennLINC/babs/issues/328) remains open.
- The relevant commits are on `add-containers-run-v2`; that branch is not listed among the fork's currently active branches on 2026-07-15.
- `mechababs-working-branch` contains the work, but is also not in the current active list.
- The current `working-branch-v4` does not contain the `containers-run` implementation.
- The design document lists unresolved pipeline support and compatibility gaps involving TemplateFlow, FreeSurfer licensing, BIDS filter files, path handling, and `--explicit` behavior.

The accurate conclusion is therefore: **Austin has designed and implemented a credible remedy for the provenance indirection, but it remains unmerged work with identified functional gaps.** Before relying on it for the planned analysis, the exact BABS revision should be pinned and its generated run record should be tested on a representative participant/session.

## 7. Relationship to BABS and Pixi

The preferred production stack is:

```text
Pixi lock / OCI build
        │
        ▼
versioned BIDS App SIF
        │
        ▼
BABS job generation and Slurm orchestration
        │
        ▼
datalad containers-run with explicit BIDS App arguments
        │
        ▼
DataLad run commit + result branch
```

Pixi tasks are not needed in the participant-level execution chain. Pixi helps build and test the BIDS App and can manage the BABS CLI environment. BABS should generate the explicit `containers-run` provenance record. DataLad should remain the authority for what was executed and what result it produced.

Until the `containers-run` work is ready, there are three defensible choices:

| Choice | Provenance quality | Recommendation |
|---|---|---|
| Use current BABS wrapper record | Re-executable if historical scripts/container/data remain available, but requires resolving the wrapper | Accept only with an explicit documented limitation and a test of historical replay |
| Pin and validate Austin's branch | More explicit container and BIDS App provenance, but unreleased and incomplete | Suitable for a controlled pilot after reviewing the remaining gaps |
| Wait for upstream integration | Best maintenance position, timing uncertain | Prefer for the full analysis if the merge timeline is compatible |

## 8. Recommended decision rule

Pixi adoption need not be decided by whether Pixi tasks are suitable provenance records. They are not required for the main value of Pixi.

Adopt Pixi if its environment solving, lock, multi-environment workspace, and container-build integration are valuable. Apply this constraint:

> No scientific result should depend for its only command provenance on a Pixi task name, Make target, or generated wrapper name.

For local analysis stages, record explicit commands with `datalad run` and use `pixi run --executable` only as the locked environment launcher. For participant/session container processing, prefer a BABS implementation that records the explicit `datalad containers-run` command.

Under that model:

- Pixi improves environment reproducibility without competing with DataLad.
- DataLad retains its full provenance and `rerun` value.
- BABS supplies scalable orchestration without hiding the BIDS App call.
- Make/Pixi tasks remain optional convenience interfaces rather than claims about what historically happened.

## 9. Acceptance criteria

- [ ] Every result commit contains a DataLad run record.
- [ ] The recorded command exposes the scientific executable/script and critical arguments.
- [ ] A run record never contains only `pixi run TASK` or `make TARGET` for a result-producing operation.
- [ ] Pixi is invoked with `--locked` and an explicit environment.
- [ ] `envs/pixi.toml` and `envs/pixi.lock` are declared inputs or are otherwise unambiguously reachable at the run revision.
- [ ] Result-producing Pixi invocations use `--executable`, not task-name resolution.
- [ ] Pixi task input/output caching is not in the DataLad replay path.
- [ ] Historical verification starts from the historical tree/environment rather than silently resolving current task definitions.
- [ ] Forward recomputation is labeled as such and receives a new provenance record.
- [ ] BABS records the explicit BIDS App/container invocation or documents and tests any remaining wrapper indirection.
- [ ] The container/SIF identity, input dataset states, output paths, and scientific parameters are available from the run commit without consulting an unversioned external system.

## Sources consulted

Local STAMPED examples:

- [From Script to STAMPED Research Object](stamped-examples/content/examples/stamped-awk-evolution.md)
- [Recording Computational Provenance with `datalad run`](stamped-examples/content/examples/datalad-run-provenance.md)
- [Re-executing Computations with `datalad rerun`](stamped-examples/content/examples/datalad-rerun-actionability.md)
- [Stellar-distance walkthrough](stamped-examples/content/examples/stellar-distance-walkthrough.md)

Primary external material:

- [DataLad `run`](https://docs.datalad.org/en/latest/generated/man/datalad-run.html)
- [DataLad `rerun`](https://docs.datalad.org/en/latest/generated/man/datalad-rerun.html)
- [`datalad containers-run`](https://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-run.html)
- [Pixi tasks](https://pixi.prefix.dev/latest/workspace/advanced_tasks/)
- [Pixi `run`](https://pixi.prefix.dev/latest/reference/cli/pixi/run/)
- [BABS issue #328: use `datalad containers-run`](https://github.com/PennLINC/babs/issues/328)
- [Austin Macdonald's active BABS branches](https://github.com/asmacdo/babs/branches/active)
- [`add-containers-run-v2` design](https://github.com/asmacdo/babs/blob/add-containers-run-v2/design/containers-run.md)
- [`add-containers-run-v2` participant-job template](https://github.com/asmacdo/babs/blob/add-containers-run-v2/babs/templates/participant_job.sh.jinja2)
- [BABS reproducibility paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC10461987/)
