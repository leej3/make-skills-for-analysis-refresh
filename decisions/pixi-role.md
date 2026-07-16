# Why Pixi belongs in this STAMPED analysis

- **Status:** accepted
- **Decision date:** 2026-07-16
- **Last reviewed:** 2026-07-16

## Decision

Adopt Pixi as the project implementation for locked local multi-environment resolution, exact tool bootstrap, reviewed dependency input to custom SIF construction, typed and described project tasks, STAMPED validation entry points, and static reproduction meta-tasks.

A Pixi `depends-on` graph may be a prospective executable specification. It is not evidence that a child ran, must not hide a scientific command, and must not determine scientific replay through task caching.

Pixi does not identify a result-producing runtime, build a SIF, record scientific provenance, provide dynamic fan-out, recovery, or durable scheduler state, or serve as a distribution artifact. Those responsibilities belong to the accepted exact SIF, Apptainer/Singularity, DataLad and DataLad Containers, BABS, the operations ledger, and `result-manifest.tsv`.

## Responsibility boundary

| Concern | Pixi role | Authority |
|---|---|---|
| Local environments | Resolve and realize named locked environments | `envs/pixi.toml` and `envs/pixi.lock` |
| Pixi and user-space tool bootstrap | Provide an exact, retrievable Pixi release and locked tools | Recorded bootstrap artifact, checksum, and procedure |
| Custom SIF dependencies | Install the reviewed Linux solution with `--locked` | Definition and build record, then the accepted exact SIF |
| Project commands | Provide typed, described launchers | Expanded command and its resulting record |
| Static reproduction graph | Compose provenance-producing leaf tasks | Run records and `result-manifest.tsv` |
| Campaign operations | Launch declared BABS operations | BABS state and the operations ledger |
| Dynamic scheduling and recovery | No ownership | BABS or a tracked orchestrator |
| Runtime identity | No ownership | Exact SIF content registered in the accepted container dataset |
| Scientific caching | No ownership; Pixi task caching is prohibited for result-changing tasks | Re-execution and provenance records |

## Environment and lock roles

The project needs distinct environments for genuinely distinct responsibilities:

- `dev` for tests, documentation, linting, and optional notebooks;
- `analysis` for extraction, modeling, figures, and validation, constrained to `pandas<2` while the selected `freesurfer-stats` release requires it;
- `babs` for BABS, DataLad, git-annex, and orchestration tools; and
- named image environments where processing runtimes have different dependency graphs.

The stable environment name `analysis` may be retained when its dependencies change. Its historical realization is not updated in place: a science-relevant change creates a reviewed lock state, a new SIF identity, and reruns of affected results.

The lock has two roles:

1. realize fast local environments for testing, notebooks, debugging, and project tooling; and
2. provide the reviewed Linux solution installed with `--locked` when constructing a project-authored SIF.

The lock does not identify the base operating system, arbitrary downloads, licensed installers, model weights, the image filesystem, or the runtime used for a result. The exact accepted SIF does.

## Repository layout

Keep environment churn in one reviewable directory:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi

envs/
├── pixi.toml
├── pixi.lock
├── .pixi/.gitignore
├── images.lock.yaml              # optional derived convenience index
└── containers/
    ├── repronim/                 # pinned ReproNim DataLad subdataset
    ├── custom/<runtime>/         # definitions for required custom images
    └── accepted/                 # DataLad dataset of registered exact SIFs
```

Keep root `pyproject.toml` focused on Python package metadata. Edit `envs/pixi.toml` directly; track the manifest, lock, and root discovery symlinks; ignore `.venv/` and the realized contents of `envs/.pixi/`.

`envs/images.lock.yaml`, when used, is a derived index. Each entry must resolve to the accepted container dataset commit and exact SIF annex key or checksum. The accepted container dataset, run record, and result manifest prevail if the index conflicts with them.

## Control dependency changes

Pixi may update a lock and synchronize a prefix when a command sees a manifest/lock mismatch. Permit that only during explicit dependency authoring.

| Mode | Allowed mutation | Rule |
|---|---|---|
| Dependency authoring | Manifest, lock, prefix, then affected SIFs | Edit intentionally, run `pixi lock`, and review the complete diff |
| Routine development | Local prefix only | Use `--locked`; failure signals that authoring is required |
| SIF construction | Fresh in-image prefix and a new SIF | Install the named Linux environment with `--locked` |
| Historical replay | No dependency refresh | Retrieve the historical accepted SIF |

CI verifies that the manifest and lock agree and that selected environments realize; it must not rewrite or upgrade the lock. There is no freshness requirement for an otherwise working lock.

Record and make retrievable the supported Pixi release, its bootstrap source or artifact, checksum, and clean-host installation procedure. Record the Pixi version used when a lock is created and when it is installed during image construction.

Do not use `pixi pack` or `pixi-pack`. A packed prefix would create a competing runtime artifact without replacing the exact-SIF identity and qualification requirements.

## Relationship to scientific images

Search a pinned ReproNim/containers dataset first. Qualify an existing exact SIF against the required application and version, BIDS App interface, architecture, licence or terms, project fixtures, and target host. Reuse it when it passes; it need not be rebuilt merely to incorporate Pixi.

When no external image qualifies:

1. update `envs/pixi.toml` and `envs/pixi.lock` intentionally;
2. update the tracked Apptainer definition and pin or checksum every non-Pixi input;
3. build the target-architecture SIF with recorded Pixi and Apptainer/Singularity versions, definition and lock hashes, and build command;
4. qualify the exact SIF bytes with interface, licence, fixture, and target-host tests;
5. register the qualified SIF with `datalad containers-add` under `envs/containers/accepted/` and publish its annex content to declared persistent storage;
6. update `envs/images.lock.yaml` if the project uses that convenience index; and
7. use the accepted SIF in a pilot DataLad/BABS execution before scaling.

Apptainer/Singularity is the image builder. Pixi resolves and installs selected dependencies during a custom build. DataLad Containers identifies, retrieves, and directly executes the completed SIF.

On macOS, run Apptainer inside a Linux Lima VM. On Apple Silicon, use an x86_64 guest under QEMU when the target Slurm systems are x86_64. Record the Lima base/configuration, guest architecture, and builder versions, then test the completed SIF on a representative Slurm host.

Signing, signature verification, an SBOM, and a build attestation are hardening choices. Consider each, record the decision and evidence, and do not treat adoption as a universal STAMPED or Pixi requirement.

## Tasks, provenance, and reproduction

Permit these paths:

```text
Pixi static meta-task
    -> independently executable provenance-producing leaf tasks

Pixi scientific leaf
    -> datalad containers-run
    -> explicit scientific executable or BIDS App arguments

Pixi campaign leaf
    -> declared BABS lifecycle operation
    -> BABS/DataLad evidence plus an operations-ledger entry
```

Prohibit these paths:

```text
datalad run or containers-run -> pixi run <task> -> hidden scientific command
pixi task -> result-changing scientific command without DataLad/BABS evidence
```

Static `depends-on` meta-graphs may provide the one-command reproduction entry point. Every result-changing leaf remains independently executable and produces an intelligible DataLad/BABS record. Never enable Pixi `inputs`/`outputs` caching for a result-changing task. For dynamic fan-out, conditional recovery, or durable scheduler state, use BABS or a tracked orchestrator.

See [Pixi tasks, DataLad provenance, and BABS operations](pixi-tasks-and-provenance.md) for the complete evidence boundary.

## Runtime isolation boundary

Local Pixi package caches and realized prefixes are disposable accelerators. Deleting them and realizing from the lock must remain a tested path. Never bind `.venv`, `envs/.pixi`, a developer home, or a mutable package cache into a result-producing execution.

Stage authenticated retrieval before scientific execution. Do not pass retrieval credentials into the scientific process. Permit only the accepted SIF, exact declared inputs, tracked configuration, a new output location, fresh scratch, a declared application licence file when required, and explicit host interfaces.

The full execution controls are defined in [Disposable and isolated scientific execution](runtime-execution-isolation.md).

## Alternatives and rationale

| Alternative | Useful when | Why Pixi is selected here |
|---|---|---|
| `uv` | The project is Python-only | It does not solve the Conda/system or multi-environment requirements |
| Conda/Mamba with `conda-lock` | A site standard requires it | Pixi provides one workspace for Conda, PyPI, platforms, and actionable tasks |
| Nix or Guix | The team already operates that ecosystem | Integration cost is higher for the selected licensed tools and HPC sites |
| Containers alone | Only result-producing execution matters | Local development and multiple tooling environments remain cumbersome |
| Snakemake or Nextflow | A separate dynamic workflow engine is required | BABS already owns BIDS/Slurm fan-out; Pixi covers the static project interface |

Pixi is the selected implementation for these objectives. STAMPED assessment credit comes from the resulting identities, records, retrievability, and replay—not merely from using Pixi.

## Acceptance criteria

- All Pixi environments and solutions live under `envs/` with tracked root discovery symlinks.
- The exact Pixi release and bootstrap procedure are recorded and retrievable.
- Routine use cannot update the lock; dependency updates are intentional and reviewed.
- A science-relevant dependency change produces a reviewed lock state and new SIF identity.
- Every retained result names an exact accepted SIF; `images.lock.yaml`, if present, resolves to that identity and is never the authority by itself.
- Tasks are typed and described; `validate-stamped` and `validate-stamped-ideal` enforce the assessment structure.
- A static reproduction graph may compose leaf tasks, but every result-changing leaf produces explicit DataLad/BABS evidence and uses no Pixi input/output caching.
- Every result in `result-manifest.tsv` resolves to its actual run record, inputs, accepted SIF, and reproduction task.
- Dynamic orchestration is delegated to BABS or a tracked orchestrator.
- Realized prefixes and caches can be deleted without losing data, provenance, or reconstruction ability.
- Result-producing executions receive no host home, mutable environment prefix, persistent cache, or retrieval credential.

## Sources

- [Pixi lock files](https://pixi.prefix.dev/latest/workspace/lock_file/)
- [Pixi multi-platform workspaces](https://pixi.prefix.dev/latest/workspace/multi_platform_configuration/)
- [Pixi advanced tasks](https://pixi.prefix.dev/latest/workspace/advanced_tasks/)
- [Pixi container deployment](https://pixi.prefix.dev/latest/deployment/container/)
- [Apptainer definition files](https://apptainer.org/docs/user/main/definition_files.html)
- [Apptainer builds](https://apptainer.org/docs/user/latest/build_a_container.html)
- [Apptainer on macOS through Lima](https://apptainer.org/docs/admin/main/installation.html#installation-on-windows-or-mac)
- [DataLad `containers-add`](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-add.html)
- [ReproNim/containers](https://github.com/ReproNim/containers)
