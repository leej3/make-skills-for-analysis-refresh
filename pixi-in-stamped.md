# Why Pixi belongs in this STAMPED analysis

## Decision

Adopt Pixi for local development, testing, BABS tooling, and reviewed dependency resolution for container builds. Do not make Pixi the scientific workflow engine, provenance authority, distribution format, or production runtime.

That limited role is useful enough to justify the additional tool:

- the project needs several incompatible environments rather than one Python virtual environment;
- most Python and compiled dependencies are available from conda-forge;
- developers need fast macOS/Linux environments for notebooks, tests, and debugging;
- OCI builds need a reviewed, exact dependency resolution;
- BABS/DataLad tooling should be isolated from legacy extraction and current statistics dependencies.

The scientific runtime associated with a result is an exact tracked Singularity/Apptainer SIF. The Pixi lock is one of the inputs used to build that runtime.

## What Pixi contributes

### One workspace for genuinely distinct environments

The current repository mixes a general analysis environment with a `freesurfer-stats` script that requires `pandas<2`. That is not a dependency puzzle to force into one solve. It is evidence for independently solved environments in one workspace.

At minimum, define:

- `dev` — lint, tests, documentation, and optional Jupyter;
- `analysis` — current extraction, modeling, and figures;
- `extract-legacy` — only while the old parser needs its pandas constraint;
- `babs` — BABS, DataLad, datalad-container, and orchestration utilities.

Pixi can lock Conda and PyPI packages for all declared environments and platforms in one `pixi.lock`. The official [lock-file documentation](https://pixi.prefix.dev/latest/workspace/lockfile/) describes the lock as the exact resolution of the manifest and recommends committing it for research/data-analysis projects.

### Better coverage than a Python-only project tool

`uv` is excellent for Python packaging and virtual environments, but this project crosses Python, compiled neuroimaging libraries, container builders, DataLad/git-annex tooling, and BABS. Pixi’s Conda-plus-PyPI model reduces the amount of bootstrap software that must be inherited from the workstation.

This does not make every dependency a Pixi dependency. FreeSurfer distributions, model weights, the FreeSurfer license, container engines, Linux kernel behavior, and Slurm remain separate boundaries. Pixi is preferable because it covers more of the local/build stack coherently, not because it covers everything.

### A practical local mirror of container dependencies

Researchers can use the `analysis` environment on macOS or Linux to debug CLIs and inspect data without rebuilding a SIF for every edit. The release image is nevertheless rebuilt and tested from the reviewed lock. Differences between a developer platform and `linux-64` remain visible in the lock and must pass container tests before results are accepted.

### A clean input to OCI construction

The tracked OCI recipe copies `envs/pixi.toml` and `envs/pixi.lock` into a pinned Linux builder and realizes the selected environment with `--locked`. The final runtime contains the realized packages and application, not a promise that Pixi will resolve them later.

Pinning a builder or base image “by digest” means using an immutable OCI reference such as:

```Dockerfile
FROM ghcr.io/prefix-dev/pixi@sha256:<digest> AS build
```

rather than only `:latest` or another mutable tag. The completed image is also named in provenance by its OCI digest. It is then converted once to SIF, and the SIF receives its own SHA-256 and git-annex identity because OCI-to-SIF conversion may add varying metadata.

## Repository policy

Keep environment churn in one reviewable directory:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi

envs/
├── pixi.toml
├── pixi.lock
├── .pixi/.gitignore
├── images.lock.yaml
├── containers/<runtime>/Containerfile
└── container-dataset/           # DataLad subdataset containing SIFs
```

Root `pyproject.toml` remains ordinary Python package metadata. Pixi can technically be configured through `[tool.pixi.*]` tables in `pyproject.toml`, including Conda dependencies, but `envs/pixi.toml` is the project rule because multi-environment and non-Python definitions quickly obscure packaging metadata.

Edit `envs/pixi.toml` directly. Track both manifest and lock. Ignore `.venv/` and the realized contents of `envs/.pixi/`. Keep root symlinks tracked so routine `pixi` commands operate from the research-object root while environment-related diffs remain filterable:

```bash
git diff -- envs/
git log -p -- envs/
git diff -- . ':(exclude)envs/**'
```

## Controlling on-the-fly updates

Pixi’s default behavior is helpful during environment authoring and unsafe as an unnoticed side effect of analysis. Its documentation states that `pixi run` may update the lock and install/synchronize the environment when needed; `--locked` instead aborts if the lock is inconsistent with the manifest. See the official [`pixi run` reference](https://pixi.prefix.dev/latest/reference/cli/pixi/run/).

Use four modes:

| Mode | Allowed mutation | Rule |
|---|---|---|
| Dependency authoring | Manifest, lock, local prefix, then new images | Dedicated branch; edit manifest and run `pixi lock` intentionally |
| Routine development | Local realized prefix only | Use `pixi run --locked -e <env> ...` |
| Container/release/HPC preparation | Fresh local/build prefix and new content-addressed artifacts | Locked realization; manifest/lock mismatch is fatal |
| Historical replay | Temporary realization of historical state | Check out matching historic manifest/lock or retrieve the historic SIF; never refresh dependencies |

There is no requirement to update a working lock for freshness. CI should prove that the committed manifest and lock can be realized and that relevant tests pass; it should not rewrite the lock or upgrade dependencies. A dependency update earns a new lock and, when it can affect science, a new OCI digest and SIF identity.

Exact update cycle:

1. Create an environment-change branch.
2. Edit `envs/pixi.toml` directly.
3. Run `pixi lock` explicitly.
4. Review the complete manifest/lock diff.
5. Realize with `--locked` and run local plus clean-Linux tests.
6. If a scientific runtime is affected, build, sign, convert, hash, register, and smoke-test a new OCI/SIF pair.
7. Update `envs/images.lock.yaml` and commit the reason, manifest, lock, tests, recipe, and artifact identities together.

## Pixi tasks: useful only outside the scientific path

Pixi tasks are acceptable for developer convenience and read-only actionability:

```toml
[tasks]
lint = "ruff check ."
unit-tests = "pytest -q"
docs-check = "python -m linkcheck docs"
environment-report = "python tools/report_environment.py"
babs-config-check = "python tools/validate_campaign.py"
babs-status = "babs status"
```

They must not:

- preprocess or resample images;
- select a cohort;
- extract morphometrics;
- fit models or create authoritative figures;
- call `babs init`, submit, retry, or merge;
- unzip/materialize BABS outputs;
- use `depends-on` to create a scientific dependency graph;
- declare scientific `inputs`/`outputs` for Pixi’s task cache.

This is not a judgment that tasks are unreliable. It preserves interpretability: a DataLad commit should expose the actual scientific command without requiring a reader to resolve the historical task definition. It also prevents Pixi’s task-skip cache from deciding whether a requested scientific replay executes.

For an occasional local scientific command before the SIF boundary, DataLad may invoke an explicit executable through the environment:

```bash
datalad run \
  -i envs/pixi.toml \
  -i envs/pixi.lock \
  -i config/example.yaml \
  -o outputs/example.tsv \
  -- pixi run --locked -e analysis --executable \
     python -m dl_morphometrics_biases.extract \
       --config config/example.yaml \
       --output outputs/example.tsv
```

The run record then contains the program and arguments, not only a task name. Claim-bearing/release operations use the registered analysis SIF instead.

## Caches and bind mounts

Three different mechanisms are often called a cache:

1. the package-download cache, which stores fetched artifacts;
2. the realized Pixi prefix under `envs/.pixi/`;
3. Pixi task input/output fingerprints.

Only the first two are useful here. Both are disposable accelerators and neither identifies a result. The third is excluded from scientific execution.

For local Docker/Lima iteration, a bind-mounted `envs/.pixi/` prefix may reduce installation time only when all of these are true:

- it is mounted at the same absolute prefix expected by the environment;
- it was realized from the current lock for the same OS/architecture;
- it is used for development/tests, not accepted scientific runs;
- deleting it and performing a clean realization remains a tested path.

Do not bind `.venv`, `envs/.pixi`, a developer home, or package caches into BABS/Slurm scientific jobs. The tracked SIF must supply the runtime. Bind only declared data, scratch/output locations, and required licenses/credentials.

## Alternatives

| Alternative | Strength | Why Pixi is preferred here | What remains useful |
|---|---|---|---|
| `uv` | Fast, strong Python project workflow | Python-first; does not solve the Conda/system side or one multi-environment Conda workspace | Python packaging concepts and Pixi’s PyPI resolver implementation |
| Conda/Mamba + `conda-lock` | Established HPC ecosystem and explicit locks | More separate tools/conventions for named environments, PyPI integration, and task-free local commands | A fully viable fallback if Pixi cannot solve a required stack |
| Nix/Guix | Strong declarative and source-oriented reproducibility | Higher adoption/integration cost for licensed FreeSurfer tools, BABS, and target HPC sites; still needs a portable HPC artifact | Useful for a future builder or infrastructure layer |
| Containers only | Strong runtime isolation and distribution | Slow/awkward for notebook iteration and does not itself organize several local tool environments | Remains mandatory for result-producing runtimes |
| ReproNim/containers | Reusable DataLad collection of neuroimaging images | May not contain the exact new clinical/recon-any combinations needed | Prefer an existing verified image when its exact content and licenses fit |
| Snakemake/Nextflow | Rich scientific workflow graphs | The chosen execution problem is BIDS participant fan-out on Slurm with DataLad provenance, which BABS addresses | Reconsider only for a distinct non-BABS workflow; Pixi tasks are not the substitute |

Pixi is therefore not the uniquely STAMPED choice. Conda/Mamba with a reviewed lock and the same OCI/SIF/DataLad pattern could satisfy the same responsibilities. Pixi wins on practical workspace ergonomics while remaining deliberately subordinate to the stronger provenance and runtime layers.

## Tensions with STAMPED and controls

| Pixi behavior or limitation | Risk | Control |
|---|---|---|
| `run`, `install`, or `shell` can update an inconsistent lock | Execution mutates environmental metadata | Use `--locked` outside dependency authoring |
| A realized prefix is incrementally synchronized | Hidden local state may make development appear cleaner than it is | Periodic deletion/clean realization; release builds are fresh |
| A lock points to remote artifacts | Old packages may become unavailable | Archive OCI and exact SIF artifacts; preserve multiple DataLad remotes |
| Platform solutions differ | macOS success does not imply Linux/HPC success | Lock declared platforms and test clean `linux-64` image builds |
| Task names add command indirection | Historic runs are harder to interpret | No result-changing Pixi tasks; explicit DataLad/container commands |
| Task caching may skip work | Replay semantics become ambiguous | No scientific task inputs/outputs or dependency graph |
| Root symlinks can surprise manifest-mutating commands | `pixi add` may replace a symlink or move environmental diffs | Edit `envs/pixi.toml` directly and run `pixi lock` |
| Pixi version affects lock compatibility | An older client may not read a newer lock format | Constrain supported Pixi and pin the builder version |
| Environment packing creates another artifact class | Runtime identity competes with OCI/SIF and weakens the single path | Do not use `pixi pack` or `pixi-pack` |

## Acceptance criteria for Pixi’s role

- All operational environments are declared in `envs/pixi.toml` and resolved in the tracked `envs/pixi.lock`.
- Distinct dependency conflicts are distinct environments.
- Routine and historical use cannot modify the lock.
- macOS is supported for development; clean Linux builds remain the release authority.
- Every scientific result resolves to an exact SIF, not merely to a lock.
- No authoritative result or BABS state transition is implemented as a Pixi task.
- No Pixi task dependency graph exists.
- Realized prefixes and caches can be removed without losing data, provenance, or the ability to rebuild.
- A viable migration path to Mamba/conda-lock remains possible because commands, data provenance, and images do not depend on Pixi task semantics.
