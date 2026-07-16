# Why Pixi belongs in this STAMPED analysis

- **Status:** accepted
- **Decision date:** 2026-07-16

## Decision

Adopt Pixi for named local environments, locked Conda/PyPI resolution, and actionable project commands. Use the reviewed Linux solution as an input when constructing project-authored Apptainer/Singularity images. A qualified external application SIF, such as an exact image from ReproNim/containers, keeps its own environment identity and need not be rebuilt merely to incorporate Pixi.

Pixi is not the scientific provenance authority, image builder, runtime identity, workflow engine, or distribution format. Those responsibilities belong to DataLad/DataLad Containers, Apptainer/Singularity, the exact registered SIF, and BABS.

## Why it is useful here

The project needs several genuinely distinct environments:

- `dev` for tests, documentation, linting, and optional notebooks;
- `analysis` for extraction, modeling, figures, and validation, constrained to `pandas<2` while `freesurfer-stats` requires it and updated in place when that constraint is removed;
- `babs` for BABS, DataLad, git-annex, and orchestration tools;
- named image environments when processing runtimes have different dependency graphs.

The pandas conflict is a reason to solve separate workspace environments, not to force one compromise solution. Pixi covers conda-forge packages, compiled dependencies, and PyPI dependencies in one workspace while supporting both macOS development and a `linux-64` image solution.

The lock has two useful roles:

1. it realizes fast local environments for testing, notebooks, and debugging; and
2. for project-authored images, it is copied into a SIF definition so the selected Linux environment can be installed with `--locked` during image construction.

The lock does not cover the base operating system, arbitrary downloads, licensed installers, model weights, or the image filesystem. The exact registered SIF remains the runtime identity associated with a result.

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
├── images.lock.yaml
└── containers/
    ├── repronim/               # pinned ReproNim DataLad subdataset
    ├── custom/<runtime>/<runtime>.def
    └── accepted/               # DataLad dataset with registered SIFs
```

Keep root `pyproject.toml` focused on Python package metadata. Pixi can read Conda dependencies from Pixi tables in `pyproject.toml`, but this project always uses `envs/pixi.toml`: multiple environments, platforms, tasks, and non-Python dependencies are clearer there.

Edit `envs/pixi.toml` directly. Track the manifest and lock; ignore `.venv/` and the realized contents of `envs/.pixi/`. Root symlinks provide conventional Pixi discovery while keeping environmental changes easy to filter in Git.

## Controlling automatic updates

Pixi may update a lock and synchronize a prefix when an ordinary command sees a manifest/lock mismatch. That is convenient during dependency authoring and inappropriate as an unnoticed side effect of routine work.

| Mode | Allowed mutation | Rule |
|---|---|---|
| Dependency authoring | Manifest, lock, prefix, then affected SIFs | Edit intentionally; run `pixi lock`; review the complete diff |
| Routine development | Local prefix only | Use `--locked`; failure signals that authoring is required |
| SIF construction | Fresh in-image prefix and new SIF | Run `pixi install --locked` for the named Linux environment |
| Historical replay | No dependency refresh | Retrieve the historical registered SIF |

There is no freshness requirement for a working lock. CI should verify that the committed manifest and lock agree and that selected environments realize; it should not rewrite or upgrade the lock. A science-relevant dependency change creates a reviewed lock and a new SIF identity.

## Exact project-image update cycle

1. Create a focused dependency/image change.
2. Edit `envs/pixi.toml` and run `pixi lock` intentionally.
3. Review the manifest and lock diff and test local environments with `--locked`.
4. Update the tracked Apptainer definition when the base, system packages, non-Pixi files, activation, labels, container runscript, or target dispatch changes.
5. Build a SIF directly from the definition. Pin/checksum all non-Pixi inputs and record the Pixi and Apptainer versions, target architecture, definition/lock hashes, and build command.
6. Smoke-test the SIF, hash it, register it with `datalad containers-add`, and publish its annex content to durable storage.
7. Sign/verify the SIF and update `envs/images.lock.yaml` with the SIF checksum, annex key, DataLad commit, build inputs, supported architecture, tests, and retrieval locations.
8. Use the new registered SIF in a pilot DataLad/BABS run before scaling.

Apptainer is the image builder. Pixi resolves and installs the selected environment during that build. DataLad Containers registers, retrieves, and executes the completed image. Pixi tasks may wrap any of these explicit commands for convenience.

When an existing ReproNim/containers SIF appears suitable, retrieve and pin the exact DataLad dataset commit and annex key, then qualify that image with the same fixture, interface, license, and target-host tests. Reuse it if it passes. Create a project image only when the required version or interface is absent or unsuitable; use ReproNim, BIDS-Apps, or NeuroDesk definitions as reviewed precedents rather than treating Pixi as the author of the scientific application.

## Direct SIF construction and macOS

A separate Docker/OCI application image is not required. An Apptainer definition may bootstrap from a digest-pinned OCI base and produce the final SIF directly; consuming an OCI base does not require publishing another OCI application artifact.

On macOS, run Apptainer inside a Linux Lima VM. On Apple Silicon, use an x86_64 guest under QEMU when the target Slurm systems are x86_64, because `%post` commands and installed binaries need to execute for the target architecture. Record the Lima configuration/base, guest architecture, and builder versions. Test the completed SIF on a representative Slurm host before treating it as a supported runtime.

Store the exact SIF in git-annex/DataLad-backed storage. Optionally publish the same SIF through an ORAS-capable registry as a secondary distribution channel. Build and store a separate OCI application image only if Docker-native users, OCI layer caching, or OCI-native publication becomes an actual requirement.

## Pixi tasks and provenance

Pixi tasks may expose development commands, BABS lifecycle operations, image operations, and parameterized DataLad commands. The governing rule is call direction:

```text
allowed:    pixi task -> datalad containers-run -> explicit scientific command
prohibited: datalad containers-run -> pixi task -> hidden scientific command
prohibited: pixi task -> scientific command without DataLad provenance
```

Do not use task dependencies or task caching as a scientific DAG. See [Pixi tasks, DataLad provenance, and BABS operations](pixi-tasks-and-provenance.md) for the full decision.

## Caches and bind mounts

Keep these distinct:

1. Pixi’s downloaded-package cache;
2. the realized prefix under `envs/.pixi/`; and
3. Pixi task input/output fingerprints.

The first two are disposable local accelerators. The third must not control scientific replay.

A bind-mounted `envs/.pixi/` can speed local Linux/Lima development only when it was realized from the current lock for the same OS, architecture, and absolute prefix. Deleting it and realizing cleanly must remain a tested path. Never bind `.venv`, `envs/.pixi`, a developer home, or a mutable package cache into an authoritative BABS/DataLad run; the registered SIF supplies that environment.

Bind data, declared output/scratch locations, and any license or credential that cannot lawfully be distributed. Distribute required license files with the repository or SIF when their terms permit it; do not assume that every license file must remain external. Never distribute secrets.

## Comparison with alternatives

| Alternative | Strength | Why Pixi is preferred here | Still useful when |
|---|---|---|---|
| `uv` | Fast Python packaging and environments | It does not solve the Conda/system side or multi-environment Conda workspace | A project is Python-only |
| Conda/Mamba + `conda-lock` | Established HPC ecosystem and explicit locks | More separate conventions for multiple environments, PyPI, and actionable commands | Pixi cannot solve a required stack or a site standard requires it |
| Nix/Guix | Strong declarative/source-oriented model | Greater integration cost for licensed neuroimaging tools, BABS, and target sites | A team already operates that ecosystem |
| Containers only | Strong runtime isolation | Slower notebook/test iteration and no local multi-environment workspace | Authoritative execution, where the SIF remains mandatory |
| Snakemake/Nextflow | Rich workflow DAGs | BABS already addresses BIDS participant fan-out on Slurm | A distinct non-BABS workflow needs a DAG engine |

Pixi is therefore a practical choice, not a STAMPED requirement. Replacing it with another reviewed resolver would not change the DataLad/SIF/BABS provenance architecture.

## Tensions and controls

| Pixi behavior | Risk | Control |
|---|---|---|
| Commands can update an inconsistent lock | Execution mutates reviewed metadata | Use `--locked` outside dependency authoring |
| Prefixes are synchronized incrementally | Hidden local state masks omissions | Delete/recreate periodically; build fresh inside SIF |
| Lock artifacts can disappear upstream | Environment cannot be reconstructed forever | Preserve exact SIFs on multiple DataLad remotes |
| Platform solutions differ | macOS success is mistaken for Linux evidence | Lock/test Linux and validate the SIF on target architecture |
| Task names introduce indirection | Historical commands become costly to interpret | Put DataLad inside the task and record the explicit command |
| Task caching can skip work | Replay semantics become ambiguous | Do not use it for scientific execution |
| Root symlinks can be replaced by mutating commands | Environmental changes escape `envs/` | Edit the real manifest directly and inspect root links |
| Pixi lock formats evolve | A builder cannot read the lock | Pin/record the Pixi version used for image construction |
| Environment packing creates a competing artifact | Runtime identity becomes ambiguous | Do not use `pixi pack` or `pixi-pack` |

## Acceptance criteria

- All Pixi environments and their solutions live under `envs/` with tracked root symlinks.
- Routine use cannot update the lock; dependency updates are intentional and reviewed.
- The lock supports local work and locked SIF construction, but every retained result names an exact registered SIF.
- Qualify and reuse an exact ReproNim SIF when it fits; otherwise construct a custom SIF directly. An OCI application image exists only for a documented consumer.
- macOS/Lima and target-architecture assumptions are recorded and smoke-tested.
- Pixi tasks follow the DataLad call-direction decision and do not define scientific dependencies or caching.
- Realized prefixes and caches can be deleted without losing data, provenance, or reconstruction ability.

## Sources

- [Pixi lock files](https://pixi.prefix.dev/latest/workspace/lock_file/)
- [Pixi multi-platform workspaces](https://pixi.prefix.dev/latest/workspace/multi_platform_configuration/)
- [Pixi container deployment](https://pixi.prefix.dev/latest/deployment/container/)
- [Apptainer definition files](https://apptainer.org/docs/user/main/definition_files.html)
- [Apptainer builds](https://apptainer.org/docs/user/latest/build_a_container.html)
- [Apptainer on macOS through Lima](https://apptainer.org/docs/admin/main/installation.html#installation-on-windows-or-mac)
- [DataLad `containers-add`](https://docs.datalad.org/projects/container/en/stable/generated/man/datalad-containers-add.html)
- [ReproNim/containers](https://github.com/ReproNim/containers)
