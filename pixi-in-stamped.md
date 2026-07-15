# Pixi in a STAMPED analysis

**Assessment date:** 2026-07-14  
**Target project:** `dl_morphometrics_biases/`  
**Purpose:** evaluate Pixi as the environment and lightweight task layer for the STAMPED conversion, replacing direct use of `uv`

## Executive conclusion

Pixi is a good fit for the Python and Conda-manageable portion of this analysis. It can replace the project's direct `uv` workflow, commit an exact multi-environment `envs/pixi.lock`, realize disposable environments, and expose the first end-to-end commands as tasks. In a neuroimaging project, its main advantage over `uv` is not just another Python lock file: Pixi can resolve Conda packages and PyPI packages across several named environments in one workspace and one lock.

Pixi does not by itself make the analysis STAMPED. In particular:

- `envs/pixi.lock` identifies package artifacts, but not the host OS, kernel, drivers, licensed FreeSurfer installation, HPC module system, input data, or the provenance of a result.
- Pixi tasks can make local commands and container entry points actionable, but they do not replace DataLad run records. BABS should own subject/session fan-out, Slurm submission, auditing, and DataLad provenance for the large-scale BIDS App stages.
- A lock file remains dependent on external package repositories. A digest-pinned container, an archived Apptainer SIF, or in narrower cases a `pixi-pack` archive is needed for stronger distributability.
- Pixi's automatic lock update is convenient during dependency authoring but is inappropriate as an unnoticed side effect of analysis execution.

The recommended operating rule is therefore:

> The Pixi manifest states dependency intent; `envs/pixi.lock` records the reviewed resolution; realized `envs/.pixi/` environments are disposable; a container digest records an OS-level build; and DataLad/BABS run history links those identities to data, commands, and results.

Ordinary development, CI, container builds, and HPC jobs should use `pixi ... --locked`. Only an explicit, researcher-initiated dependency-update session should change `envs/pixi.toml` and `envs/pixi.lock`; a stable analysis may keep the same reviewed lock indefinitely. Historic reproduction should check out the matching manifest and lock from Git and still use `--locked`; `--frozen` should not be used to ignore a mismatch.

## 1. Recommended decision

| Question | Recommendation |
|---|---|
| Adopt Pixi? | **Yes**, for Python, Conda packages, environments, and lightweight project tasks. |
| Replace direct `uv` commands? | **Yes.** Remove `uv`-specific environment instructions and the inline `uv` dependency block after its requirements are represented in Pixi. Pixi currently uses `uv` internally for PyPI resolution/building, so this replaces the user-facing workflow, not necessarily every internal implementation component. |
| `pyproject.toml` or `pixi.toml`? | Always use `envs/pixi.toml` for Pixi environments, non-Python/Conda dependencies, operational pins, and tasks. Keep root `pyproject.toml` as Python package metadata. Expose the Pixi manifest through a tracked root `pixi.toml` symlink. |
| Where should the lock live? | Track the content as `envs/pixi.lock` and expose it through a tracked root `pixi.lock` symlink, which satisfies Pixi's workspace-root convention. |
| Where should realized environments live? | Point a tracked root `.pixi` symlink at `envs/.pixi/`; ignore its realized contents. Ignore `.venv/` outright. |
| Can Pixi tasks replace Make/Snakemake? | They can replace Make for local entry points and checks. BABS—not Snakemake—will own subject/session fan-out, Slurm resources, job auditing, and DataLad-backed execution provenance. |
| Is a lock enough? | No. Use a lock-built environment for rapid development and clean-room tests; use a digest-pinned Docker/OCI image and an archived Apptainer/Singularity SIF for the FreeSurfer/HPC execution boundary. |

This recommendation extends the findings in the [initial STAMPED review](STAMPED_REVIEW_DL_MORPHOMETRICS_BIASES.md). It does not resolve the review's data identity, path configuration, notebook state, provenance, licensing, or result-distribution findings.

## 2. Why Pixi helps each STAMPED principle

| Principle | Contribution from Pixi | Residual limitation |
|---|---|---|
| **S — Self-containment** | A tracked `envs/` module containing the Pixi manifest, lock, containers, BABS configuration, and tasks makes the software setup reachable from the research-object root. | The package bytes usually remain in Conda/PyPI repositories. Pixi does not bring ABCD data, FreeSurfer licenses, TemplateFlow data, or HPC services into the object. |
| **T — Tracking** | `envs/pixi.lock` records exact artifacts for all declared environments/platforms and includes artifact checksums. Git can review the manifest, lock, and tasks together. | `envs/.pixi/` is mutable realized state, not a tracked identity. Source/path dependencies do not receive the same locked artifact checksum protection. A lock is environment provenance, not result provenance. |
| **A — Actionability** | `pixi run` exposes named commands; `depends-on`, inputs, outputs, arguments, and failure propagation can express local preparation and verification. BABS supplies the scalable BIDS App/Slurm execution layer. | Pixi task metadata is not automatically DataLad provenance. Stateful notebooks must first become parameterized, non-interactive commands or BIDS App entry points. |
| **M — Modularity** | Features and named environments separate analysis, development, and production concerns while a solve group can keep shared packages compatible. | Features are package groupings, not independent data/code/environment modules with separate persistent identities and licenses. |
| **P — Portability** | The lock can cover multiple declared platforms, and the manifest can constrain platform properties. Pixi can combine compiled Conda software with Python packages. | A solution for `linux-64` is not a complete description of glibc, drivers, CPU features, host tools, or licensed neuroimaging applications. A container or documented HPC module remains necessary for those boundaries. |
| **E — Ephemerality** | A fresh clone can realize an environment from `envs/pixi.lock`; `envs/.pixi/` can be deleted and rebuilt; containers and node-local scratch can be discarded. | Reusing one persistent environment is not evidence that reconstruction works. Clean realization should be tested at migration/release milestones and periodically thereafter. |
| **D — Distributability** | The small lock is easy to publish; Pixi environments can be packed, and a lock can drive OCI image builds. | Package URLs can disappear, credentials may be required, and `pixi-pack` has platform and PyPI-wheel limitations. Archive an image/SIF or package mirror for long-term distribution. |

Pixi's [lock-file documentation](https://pixi.prefix.dev/latest/workspace/lock_file/) explicitly distinguishes the direct requirements in the manifest from the exact resolved artifacts in `pixi.lock`, and recommends committing the lock for research and data-analysis projects. Its [security guidance](https://pixi.prefix.dev/latest/security/) adds an important qualification: Conda and registry PyPI artifacts are verified against locked checksums, while Git, direct-URL, and local source dependencies are not covered in the same way.

## 3. Current project issues to address during migration

The present `dl_morphometrics_biases/pyproject.toml` is useful Python package metadata, but the operational environments should be modeled explicitly in a Pixi workspace:

1. Python has only a lower bound (`>=3.9`) and packages have broad lower bounds. These express compatibility intent, not the result-producing environment.
2. Analysis dependencies are under `[project.optional-dependencies]`, while `nbstripout` also appears in a second `[dependency-groups].dev`. Root package metadata should describe the installable Python project; operational groupings should be expressed as Pixi features/environments in `envs/pixi.toml`.
3. `scripts/fsstats_extraction.py` requires `freesurfer-stats>=1.2.0` and `pandas<2`, while the broader analysis permits newer pandas. This is not a constraint that must be forced into one solution. It is a reason to define a legacy/extraction environment separately from the current analysis environment. The two environments can coexist in one workspace and lock without sharing a solve group.
4. `.gitignore` currently ignores `pixi.lock`. Remove that rule: both the root `pixi.lock` symlink and its `envs/pixi.lock` target must be tracked. Only the realized contents of `envs/.pixi/` and `.venv/` should be ignored.
5. `.python-version` may remain as an editor/bootstrap hint, but individual Pixi environments may intentionally use different Python versions. For example, the legacy extraction environment can remain on Python 3.9 while the BABS orchestration environment uses a supported newer Python.
6. FreeSurfer, recon-any, AFNI, FSL, TemplateFlow assets, and NIH HPC behavior are not fully represented by the current Python metadata. Each must either become a declared Pixi/Conda dependency or a separately tracked environment/data module. Do not imply that a Python/Conda lock captures a host module merely because a task can call it.

The Pixi executable is also part of the environment-building provenance. The workstation inspected for this report currently has Pixi 0.66.0, while the current official container guide illustrates Pixi 0.72.1. Select and test an adoption version before migration. Put a compatible range in `requires-pixi`, pin the exact executable or builder image for release builds, and record `pixi --version` in run provenance. Pixi documents that newer Pixi versions can read older lock formats, but older Pixi versions cannot necessarily read newer formats.

## 4. Manifest and repository layout

### 4.1 Put the Pixi workspace in `envs/`

Pixi's current [manifest reference](https://pixi.prefix.dev/latest/reference/pixi_manifest/) says that `pyproject.toml` can use the same Pixi tables as `pixi.toml`, including Conda/non-Python dependencies under `[tool.pixi.dependencies]`; it is therefore not technically restricted to Python dependencies. Nevertheless, `pixi.toml` is the better project policy here: it keeps Python package metadata separate from operational environments and avoids an increasingly cumbersome set of `[tool.pixi.*]` task and feature tables.

Store the real files under `envs/` and track root symlinks for Pixi's conventional discovery and workspace-root lock location:

```text
pixi.toml -> envs/pixi.toml
pixi.lock -> envs/pixi.lock
.pixi     -> envs/.pixi
```

Run Pixi from the research-object root and do not select `envs/pixi.toml` with `-m`: the root `pixi.toml` symlink deliberately makes the research-object root the Pixi workspace, so relative paths and task working directories remain intuitive. Edit `envs/pixi.toml` directly; Pixi 0.66.0 preserved the lock symlink during testing, but `pixi add` replaced a symlinked manifest, so manifest-mutating Pixi subcommands are not part of this project's supported authoring cycle.

Root `pyproject.toml` should contain no `[tool.pixi.*]` tables. It remains package metadata consumed by an editable dependency from `envs/pixi.toml`.

An illustrative `envs/pixi.toml` is below. Version bounds remain examples until checked against the preserved analysis.

```toml
[workspace]
name = "dl-morphometrics-biases"
channels = ["conda-forge"]
platforms = ["linux-64"]
requires-pixi = ">=0.72,<0.73"  # example tested range

# Install the root Python project into analysis environments.
[pypi-dependencies]
dl_morphometrics_biases = { path = ".", editable = true }

[feature.extract.dependencies]
python = "3.9.*"
pandas = ">=1.5,<2"

[feature.extract.pypi-dependencies]
freesurfer-stats = ">=1.2"

[feature.analysis.dependencies]
python = "3.9.*"
# Add a pandas 2 constraint here only if the analysis actually requires it.
pandas = ">=1.5"

[feature.dev.dependencies]
ruff = ">=0.7"
pre-commit = ">=4.1"

# BABS has its own orchestration environment and does not inherit the
# project's legacy Python/pandas dependencies.
[feature.babs.dependencies]
python = ">=3.10"

[feature.babs.pypi-dependencies]
babs = "==0.5.4"

[environments]
extract = { features = ["extract"] }
analysis = { features = ["analysis"], solve-group = "analysis" }
dev = { features = ["analysis", "dev"], solve-group = "analysis" }
babs = { features = ["babs"], no-default-feature = true }

[tasks]
lint = { cmd = "ruff check .", default-environment = "dev" }
format-check = { cmd = "ruff format --check .", default-environment = "dev" }

# Target interfaces after the scripts accept tracked parameters.
extract-stats = { cmd = "python scripts/fsstats_extraction.py --config config/analysis.toml", default-environment = "extract" }
make-figures = { cmd = "python scripts/make_figures.py --config config/analysis.toml", depends-on = ["extract-stats"], default-environment = "analysis" }
verify-results = { cmd = "python scripts/verify_results.py", depends-on = ["make-figures"], default-environment = "analysis" }
reproduce = { depends-on = ["verify-results"] }
```

The important workspace point is that `extract` and `analysis` are separately solved environments. Do not put them in one solve group merely to make the versions agree. The `analysis` and `dev` environments can share a solve group because `dev` is intentionally the analysis environment plus development tools. `babs` uses `no-default-feature = true` so it does not inherit the root project or its legacy dependencies.

Because the root manifest symlink is Pixi's entry point, the task working directory defaults to the research-object root; no compensating `cwd = ".."` is needed.

### 4.2 Repository layout and Git filtering

Use this layout:

```text
dl_morphometrics_biases/
├── pyproject.toml            # Python package metadata; no [tool.pixi] tables
├── pixi.toml -> envs/pixi.toml
├── pixi.lock -> envs/pixi.lock
├── .pixi -> envs/.pixi
├── envs/
│   ├── pixi.toml             # sole Pixi workspace manifest
│   ├── pixi.lock             # all named environments; tracked
│   ├── README.md             # supported targets and update instructions
│   ├── Dockerfile            # OCI build recipe
│   ├── apptainer.def         # only if maintained as a separate build path
│   ├── images.lock.yml       # OCI digest, SIF checksum, build metadata
│   ├── babs/
│   │   └── config.yml        # BIDS App, mounts, resources, arguments
│   └── .pixi/
│       └── .gitignore        # tracked; ignores realized contents
├── config/                   # tracked scientific and path-independent config
├── scripts/                  # non-interactive commands
└── provenance/               # run manifests in addition to DataLad/BABS history
```

This makes environment-only review straightforward:

```bash
git diff -- envs/
git log -p -- envs/
```

It also makes it easy to omit generated lock/container noise while reviewing scientific code:

```bash
git diff -- . ':(exclude)envs/**'
git log -p -- . ':(exclude)envs/**'
```

The lock should remain reviewable when the environment is intentionally changed; putting it under `envs/` makes that review opt-in by path rather than hiding it with a Git diff attribute.

Track all three root symlinks. Keep the `envs/.pixi/` target directory present in a fresh clone with this tracked `envs/.pixi/.gitignore`:

```gitignore
*
!.gitignore
```

The root `.gitignore` should ignore `.venv/`, but must not ignore the root `pixi.toml`, `pixi.lock`, or `.pixi` symlinks or either manifest/lock target.

Avoid reproducibility-relevant settings in ignored `envs/.pixi/config.toml`. `--no-config`/`PIXI_NO_CONFIG=1` skips system and user configuration, but Pixi documents that workspace-local `.pixi/config.toml` is still merged. Put stable choices in `envs/pixi.toml` or a tracked, explicitly selected configuration, and keep credentials outside Git.

### 4.3 Why parameterized tasks help DataLad—and what is not automatic

The requirement that result-producing commands be non-interactive and parameterized is primarily about Actionability and provenance capture:

- the same command can be executed by a person, CI, a container entry point, `datalad run`, or BABS;
- inputs, outputs, parameters, and failure status can be stated before execution;
- rerunning does not depend on notebook memory, prompts, or a user's shell history.

Pixi's `inputs` and `outputs` task fields do **not** automatically become DataLad provenance. They drive Pixi's own task-skip cache by fingerprinting the declared files, command, and environment. DataLad separately records the command and repository changes; its `-i/--input` and `-o/--output` options prepare/retrieve inputs and outputs and store those declarations with the run record. The two systems can describe the same paths, but no current automatic bridge should be assumed.

For a local, non-BABS stage, the composition can look like:

```bash
datalad run \
  -m "extract FreeSurfer statistics" \
  -i 'inputs/**' \
  -o 'outputs/stats/**' \
  -- pixi run --locked -e extract extract-stats
```

For participant-level processing, the stronger plan is to expose the computation as a BIDS App container and let BABS generate and record the exact Singularity invocation, input datasets, app, and results. Pixi task declarations remain useful for building, smoke-testing, and locally exercising that entry point; BABS/DataLad remains the provenance authority.

## 5. Controlling automatic lock and environment updates

Pixi's default convenience behavior needs a project policy. According to the [lock documentation](https://pixi.prefix.dev/latest/workspace/lock_file/#lock-file-changes), `install`, `run`, `shell`, `shell-hook`, `tree`, `list`, `add`, and `remove` may create or update `pixi.lock` when it is missing or inconsistent. `pixi run` can also realize or update the local environment before executing the task. This does **not** mean that every run upgrades everything to the newest release; explicit upgrades are performed with `pixi update`. It does mean an analysis command can mutate environmental metadata if the manifest and lock have drifted.

That risk is not unique to Pixi. `uv run` also locks and syncs by default and uses `--locked` to reject an outdated lock, as described in the [uv locking documentation](https://docs.astral.sh/uv/concepts/projects/sync/#automatic-lock-and-sync). The STAMPED problem in the current repository is policy: the lock was ignored, so no reviewed environment identity existed.

Use four modes:

| Mode | Permitted behavior | Command policy |
|---|---|---|
| **Dependency authoring** | The researcher deliberately changes `envs/pixi.toml` and regenerates `envs/pixi.lock`; changes are reviewed as a unit. | Edit `envs/pixi.toml` directly, then run `pixi lock` from the repository root. Use an explicit `pixi update` only for an intentional upgrade. |
| **Routine development** | The realized environment may be synchronized to the committed lock; the lock must not change. | `pixi run --locked -e dev <task>` or export `PIXI_LOCKED=true` in the project wrapper. |
| **CI, container, release, HPC** | Manifest/lock mismatch is fatal; no dependency upgrade occurs; outputs are linked to identities. | Run from the repository root and use `--locked`. A separate `pixi lock --check` is unnecessary when the tested install/run already uses `--locked`. |
| **Historical replay** | Use the historical manifest and historical lock together. | Check out both files from the target Git commit and use `--locked`. Do not use `--frozen` to make a newer manifest accept an older lock. |

The distinction between `--locked` and `--frozen` is essential:

- `--locked` verifies that the manifest and lock agree and aborts when they do not. This is the normal STAMPED setting.
- `--frozen` installs the lock without updating it even when the manifest is inconsistent. This can be useful for package-manager debugging, but it conceals a contradiction between two tracked specifications and should not be the reproduction policy.
- `--no-install` allows lock authoring without mutating the realized environment.
- `pixi lock --check` also detects manifest/lock drift, but it adds little when CI immediately performs a `--locked` install or run, which performs the relevant consistency check itself.

Nothing in this policy asks CI to update dependencies. A lock file can remain unchanged for the lifetime of a stable analysis. `pixi update` should run only when a researcher chooses to evaluate a dependency/security/compatibility change. The value of CI is to use the already selected lock without silently replacing it.

A “fresh locked realization” also does not mean an update: it deletes or bypasses the realized environment and installs the exact artifacts already named in `envs/pixi.lock`. It tests Ephemerality and detects missing upstream artifacts, but it can be expensive. A balanced policy is:

- routine pull-request CI may reuse an environment/package cache keyed by the lock and run tasks with `--locked`;
- release, publication, or major milestone CI performs a clean locked realization;
- an optional scheduled clean-room job provides earlier warning that locked artifacts are no longer retrievable.

The clean-room job is therefore a release gate or periodic assurance test, not a requirement on every commit.

For protection against accidental unlocked invocation, provide a small tracked wrapper or CI environment setting rather than relying on memory. For example, the documented project entry point can be:

```bash
PIXI_LOCKED=true PIXI_NO_CONFIG=1 pixi run -e analysis reproduce
```

The wrapper is a guard, not provenance. The run record should additionally capture at least:

```text
Git commit
SHA-256 of pyproject.toml, envs/pixi.toml, and envs/pixi.lock
pixi --version
selected Pixi environment and platform
OCI image digest or SIF SHA-256, when used
input dataset IDs/checksums
task name, arguments, and relevant resource configuration
output manifest/checksums
```

## 6. Exact dependency development cycle

### 6.1 First migration without a scientific upgrade

1. Preserve a baseline from the currently working environment: package versions, tool/module/container versions, input identities, commands, and selected result checksums. The migration should first reproduce the baseline, not combine environment conversion with a broad scientific software upgrade.
2. Select the Pixi release to adopt. Upgrade the local 0.66.0 installation if the chosen workflow depends on newer documented features, then set `requires-pixi` and pin the exact Pixi builder image/binary used in automated builds.
3. Keep root `pyproject.toml` focused on package metadata. Create `envs/pixi.toml`, migrate the inline `uv` dependencies into an `extract` feature, and define separate `extract`, `analysis`, `dev`, and `babs` environments.
4. Add only the supported execution platforms. Start with `linux-64`, because the real production target is Linux/HPC. Add macOS only if the analysis is actually tested there; a solvable lock entry is not evidence of equivalent numerical behavior.
5. Decide package source deliberately. Keep general installable-package metadata under root `[project]`; put operational Conda/PyPI choices in the appropriate feature in `envs/pixi.toml`. Declare channels explicitly and keep channel priority stable.
6. Create the root `pixi.toml`, `pixi.lock`, and `.pixi` symlinks, generate `envs/pixi.lock` with `pixi lock` from the repository root, inspect it, and remove the broad `pixi.lock` ignore rule. Do not hand-edit the lock.
7. Realize `extract` and `analysis` with `--locked` and reproduce the relevant preserved smoke results in each environment.
8. Before accepting the migration baseline, use `pixi clean -e extract` and `pixi clean -e analysis` or test from a fresh clone, then repeat the locked installs. This is the initial Ephemerality check; it need not run on every later commit.
9. Commit the three root symlinks, `envs/pixi.toml`, `envs/pixi.lock`, `envs/.pixi/.gitignore`, tasks, and documentation together. Do not commit realized `envs/.pixi/` contents or `.venv/`.

### 6.2 Adding or changing a dependency

Use this repeatable cycle:

```bash
# 1. Edit envs/pixi.toml directly, then resolve without relying on an
# analysis run. Do not use `pixi add`: it may replace the root manifest
# symlink with a regular file.
pixi lock

# 2. Inspect the complete resolution change.
git diff -- envs/

# 3. Realize exactly that reviewed resolution and test it. `--locked`
# also rejects a manifest/lock mismatch.
pixi install --locked -e analysis
pixi run --locked -e dev lint
pixi run --locked -e analysis smoke

# 4. Perform the clean-room check before accepting a release change.
pixi clean -e analysis
PIXI_NO_CONFIG=1 pixi install --locked -e analysis
PIXI_NO_CONFIG=1 pixi run --locked -e analysis smoke
```

The clean-room step is for a release/milestone or an environment migration, not necessarily every dependency edit. `pixi clean -e analysis` removes the realized named environment without deleting the tracked symlink target; automated release tests should normally use a fresh checkout. Review intentional lock diffs for unexpected channel changes, source changes, large transitive updates, and platform-specific divergence. Pixi's security guide recommends lock review and provides `pixi diff` as a human-readable aid.

For an intentional upgrade, preview it first:

```bash
pixi update --dry-run PACKAGE
pixi update --no-install PACKAGE
git diff -- envs/pixi.lock
```

Then repeat locked tests and the clean-room check. Batch routine updates into dedicated pull requests so scientific code changes and environment changes can be evaluated separately.

### 6.3 When to rebuild container artifacts

| Change | Edit manifest/lock? | Edit container recipe? | Rebuild environment image? | Rebuild final release image? |
|---|---:|---:|---:|---:|
| Python/Conda/PyPI requirement | Yes | Usually no | Yes | Yes |
| Lock-only dependency refresh | Lock | No | Yes | Yes |
| Pixi bootstrap version | Usually `requires-pixi` and builder pin | Yes | Yes | Yes |
| Base OS, system library, locale, FreeSurfer/container component | Maybe | Yes | Yes | Yes |
| Analysis source only | No | No | No for a dependency-only dev image | Yes when producing a release image containing code |
| Task definition only | Manifest only; no dependency refresh is needed | No | No | Yes if the task/entry point is embedded in the release image |
| Input data or scientific parameters | No | No | No | Not necessarily; record a new run and data/parameter identities |

Do not duplicate Python package versions in a Dockerfile or Apptainer definition. Operational package changes belong in `envs/pixi.toml` and `envs/pixi.lock`; the container recipe should consume the lock with `--locked`. Edit the recipe only when the container layer itself changes.

## 7. Docker/OCI development and release cycle

Pixi's [official container guide](https://pixi.prefix.dev/latest/deployment/container/) demonstrates a multi-stage build with a versioned Pixi image, `pixi install --locked`, `pixi shell-hook`, and copying the realized environment to the same absolute prefix in a smaller runtime image. For STAMPED use, strengthen that example as follows:

1. Pin both the Pixi builder and runtime base images by immutable digest, not `latest` or a tag alone.
2. Build only from a clean, reviewed `envs/pixi.toml` plus `envs/pixi.lock`; fail on mismatch with `--locked`. Root `pyproject.toml` and source are also copied because the Pixi workspace installs the project as an editable path dependency.
3. Keep the environment prefix at the same absolute path between build and runtime stages. Conda prefixes and activation metadata may contain absolute paths.
4. Label the output image with the Git revision, lock SHA-256, source repository, build date, and Pixi version. Labels help inspection; the registry digest remains the image identity.
5. Run a smoke test during the build or immediately after it.
6. Push and record the resulting `image@sha256:...` in `envs/images.lock.yml`. Never record only a mutable tag.
7. Convert/pull that exact OCI digest to SIF for HPC and record the SIF's own SHA-256. OCI and SIF digests identify different byte streams and should not be conflated.

Use two container roles:

- **Development provider:** contains the pinned OS/system tool layer and Pixi executable. Source is bind-mounted; a cache or container-owned Pixi prefix accelerates iteration.
- **Release/reproduction image:** contains the exact realized environment and code. It is immutable and identified by digest; only data, output, scratch, and license/configuration files are mounted at runtime.

The development image can be rebuilt less often. The release image must be rebuilt whenever code, the lock, or its container recipe changes.

### 7.1 What “consume the lock and pin base images by digest” means

These are two separate layers of identity:

1. The Dockerfile **consumes the Pixi lock** when it copies the tracked root symlinks and their `envs/pixi.toml` and `envs/pixi.lock` targets, then runs `pixi install --locked -e ENVIRONMENT` from the repository root. `--locked` means Pixi installs the exact already selected package artifacts and fails if the manifest no longer agrees. It does not search for newer versions. A package change normally edits the Pixi manifest/lock and rebuilds the image; it does not duplicate package versions in the Dockerfile.
2. A Docker `FROM` line **pins an image by digest** when it uses `image:readable-tag@sha256:...`. The tag is useful to humans but can be moved by a registry owner; the digest identifies the exact image content. The builder image supplies the Pixi executable used to assemble the environment. The runtime image supplies the final Ubuntu/system-library layer. Pinning both prevents either foundation from changing under an otherwise unchanged lock.

The lock controls the packages installed above the base image; it does not identify the base image itself. Conversely, a pinned base image does not identify the Python/Conda environment. The final OCI digest identifies the composition that was actually built and tested.

A correctness-first release recipe can follow this shape. Replace every placeholder with a reviewed version/digest; keep `.git`, realized `envs/.pixi` contents, `.venv`, data, and outputs out of the Docker build context, while retaining the tracked root symlinks and `envs/.pixi/.gitignore` target marker.

```dockerfile
FROM ghcr.io/prefix-dev/pixi:PINNED_VERSION@sha256:PINNED_BUILDER_DIGEST AS build

WORKDIR /opt/dlmb
COPY . .
ARG PIXI_ENV=analysis
RUN pixi install --locked -e "${PIXI_ENV}"
RUN pixi shell-hook -e "${PIXI_ENV}" -s bash > /pixi-shell-hook
RUN printf '#!/bin/bash\n' > /entrypoint.sh \
    && cat /pixi-shell-hook >> /entrypoint.sh \
    && printf 'exec "$@"\n' >> /entrypoint.sh \
    && chmod 0755 /entrypoint.sh

FROM ubuntu:PINNED_RELEASE@sha256:PINNED_RUNTIME_DIGEST AS runtime

WORKDIR /opt/dlmb
# Preserve the absolute environment prefix used in the build stage.
COPY --from=build /opt/dlmb /opt/dlmb
COPY --from=build /entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python", "scripts/verify_results.py"]
```

This copies the project before environment installation because `envs/pixi.toml` installs the root project as an editable path dependency and therefore needs its source. The generated environment lives under `/opt/dlmb/envs/.pixi/`, and copying `/opt/dlmb` to the identical runtime path preserves its prefix. Optimize layers only after preserving that semantic. Build separate images from the relevant named environment when the BIDS App processing and downstream analysis have materially different dependencies; the `babs` orchestration environment normally runs outside the BIDS App image.

Even with a pinned base and `envs/pixi.lock`, do not assume two container rebuilds will be byte-for-byte identical: build metadata and source builds may vary. The pushed OCI digest identifies the artifact that was actually tested. Archiving that digest/artifact is stronger evidence than merely asserting that the Dockerfile can rebuild it.

## 8. Bind mounts, `.venv`, and fast iteration without a STAMPED cost

### 8.1 Do not carry the uv `.venv` pattern over unchanged

Pixi normally realizes named environments under the workspace's `.pixi/envs/<environment>`. The tracked root `.pixi -> envs/.pixi` symlink places them at `envs/.pixi/envs/<environment>`, not `.venv`. Maintaining a second `.venv` would split authority between Pixi and Python tooling, make it unclear which packages were actually imported, and weaken Tracking. Once Pixi replaces the direct `uv` workflow, the default should be:

- no project `.venv`;
- ignored `envs/.pixi/` as realized state;
- tracked `envs/pixi.toml` and `envs/pixi.lock` as environment authority.

### 8.2 Three different things called “cache”

The earlier wording conflated three types of state:

| State | What it contains | Reuse policy | Provenance status |
|---|---|---|---|
| **Package-download cache** (`PIXI_CACHE_DIR`) | Downloaded/extracted Conda packages, PyPI wheels, and repository metadata. | Reuse across installs. Pixi verifies locked registry artifacts against checksums; routine lock changes do not require clearing this cache. Delete it only for corruption, storage policy, or troubleshooting. | Never authoritative; it merely avoids downloads. |
| **Realized environment prefix** (`envs/.pixi/envs/<name>`) | The installed, executable environment for a named Pixi environment. | Pixi can reconcile it to `envs/pixi.lock` on a `--locked` install/run. Reuse it for development; discard it for a clean-room test, after incompatible image/OS/platform changes, or after ad hoc mutation/corruption. | Never cite the directory as the environment identity; cite the manifest/lock and container. |
| **Pixi task cache** | Fingerprints used to decide whether a task with declared `inputs`/`outputs` can be skipped. | Let Pixi invalidate it when the command, environment, inputs, or outputs change. | An efficiency mechanism, not DataLad provenance or proof that a result is current. |

Thus, “development cache/prefix volumes are invalidated by lock/image/platform identity” was too compressed. The corrected rule is:

- the package-download cache normally needs no manual invalidation;
- a persisted realized prefix must always be reconciled through `pixi ... --locked`, and should be discarded when its underlying container ABI/platform changes or when performing a clean-room test;
- the task cache is never accepted as provenance.

### 8.3 Safest acceleration order

Use these optimizations in descending order of safety:

1. **Bind-mount source and data, not the environment.** An immutable image supplies the environment; edited source appears at a stable path. This has essentially no reproducibility cost if the image digest, source commit/diff, data identities, and command are captured.
2. **Persist package-download caches.** A `PIXI_CACHE_DIR` volume saves downloads while Pixi still constructs the environment from the lock. Cache contents are not treated as authoritative and may be deleted at any time.
3. **Use a container-owned named volume for `/workspace/envs/.pixi`.** On startup, always run `pixi install --locked` or `pixi run --locked` from `/workspace`; the root symlinks select the tracked manifest and lock. Pixi can update the realized prefix to the committed lock, so a new volume per lock is optional rather than required. Delete/recreate the volume after a base-image/platform/ABI change or for a clean-room test.
4. **Bind a host `envs/.pixi` prefix only in a tightly controlled case.** The host and container must have compatible OS/architecture/ABI expectations and the same absolute in-container prefix. This is fragile across machines and is not recommended as the documented path.

An illustrative Docker development invocation is:

```bash
docker run --rm -it \
  --mount type=bind,src="$PWD",dst=/workspace \
  --mount type=volume,src=dlmb-pixi-linux64,dst=/workspace/envs/.pixi \
  --mount type=volume,src=dlmb-pixi-cache,dst=/pixi-cache \
  -e PIXI_CACHE_DIR=/pixi-cache \
  -w /workspace \
  "registry.example/dlmb-dev@sha256:IMAGE_DIGEST" \
  pixi run --locked -e dev lint
```

The exact cache path and entry point depend on the adopted image. The important properties are that the lock cannot change, `envs/.pixi` is a replaceable optimization, and neither Docker volume is cited as environment provenance.

This persistence has no material STAMPED cost only if all of the following are true:

- the manifest and lock are tracked and reviewed;
- every invocation checks `--locked`;
- the persisted prefix is never committed, archived as the canonical environment, or used as evidence of reproducibility;
- release/milestone clean-room tests reconstruct from an empty prefix;
- results record the lock and image identity, not the volume name;
- any ad hoc package installation into the prefix is prohibited or erased before analysis.

If the environment itself is already baked into the immutable image, do not bind `envs/.pixi` over it. Bind source at a different path or mount only the fast-changing source subdirectories so the baked prefix remains visible.

### 8.4 HPC cache and prefix policy

Do not place one writable `envs/.pixi/envs/analysis` on shared storage and let many Slurm jobs update it concurrently. Prefer, in order:

1. an immutable SIF containing the environment;
2. a per-job/per-node locked realization on `$SLURM_TMPDIR` or `/lscratch/$SLURM_JOB_ID`, backed by a shared read-mostly package cache;
3. a `pixi-pack` archive unpacked to node-local scratch when containers are unavailable.

Current Pixi [cache configuration](https://pixi.prefix.dev/latest/reference/pixi_configuration/#cache) distinguishes shared-friendly caches such as Conda packages and PyPI wheels from environment caches that prefer node-local storage, and can redirect the latter to Slurm/PBS scratch. This behavior is newer than the installed Pixi 0.66.0 and must be tested with the adopted pinned version before relying on it. Pixi also supports detached environments, but its documentation warns that they disconnect the workspace from its environment and require manual cleanup. Use them only as an explicit HPC optimization, never as the research-object identity.

## 9. Apptainer/Singularity on NIH HPC

An Apptainer/Singularity artifact is the better production boundary for the FreeSurfer stages because it captures more OS-level state and is supported on Biowulf. It can be produced in either of two ways:

### Preferred: one OCI recipe, then exact conversion

1. Build and test the digest-pinned OCI image in CI or on a permitted build host.
2. Push it to a registry and record the OCI digest.
3. On the HPC side, pull/convert that exact digest to SIF once.
4. Compute and record `sha256sum image.sif`.
5. Run the immutable SIF for all jobs; do not rebuild it inside each job.

A minimal conversion definition can retain an inspectable, tracked pointer to the exact OCI artifact:

```text
Bootstrap: docker
From: registry.example/dlmb@sha256:PINNED_OCI_DIGEST

%test
    /entrypoint.sh python scripts/verify_results.py --smoke
```

On a host that permits the build, the production operation is then conceptually `apptainer build dlmb.sif envs/apptainer.def`; on Biowulf the executable may be named `singularity`. The resulting `dlmb.sif` must receive its own SHA-256 and smoke test before use.

Apptainer supports Docker/OCI sources and ORAS-hosted SIF artifacts, although its [OCI compatibility documentation](https://apptainer.org/docs/user/latest/docker_and_oci.html) notes differences caused by its security model and close host integration. Test FreeSurfer, paths, locale, and any GPU behavior on the actual cluster.

### Alternative: tracked `apptainer.def`

Maintain `envs/apptainer.def` only when a separate recipe is genuinely required. Pin its `From:` image by digest, install the selected Pixi version explicitly, copy the root Pixi symlinks and their `envs/` targets, and run `pixi install --locked -e analysis` from the repository root in `%post`. Add a `%test` smoke check. Track changes to the definition and lock together and archive the resulting SIF checksum.

The [Apptainer build command](https://apptainer.org/docs/user/latest/cli/apptainer_build.html) supports immutable SIF output, Docker/OCI sources, and unprivileged/fakeroot builds subject to host configuration. NIH's current [Biowulf Singularity guidance](https://hpc.nih.gov/apps/singularity.html) describes several build paths, including compatible `proot` builds and remote/fakeroot alternatives. Therefore, an image **can sometimes be built on Biowulf**, but this is definition- and policy-dependent. The robust workflow is to build once in CI/off-cluster or use the supported cluster method, test on Biowulf, archive the exact SIF, and execute that artifact rather than rebuilding during analysis.

For runtime mounts:

- mount input data read-only where possible;
- mount a dedicated writable output directory;
- map job-local `/lscratch/$SLURM_JOB_ID` to container scratch/`/tmp`;
- bind the FreeSurfer license at runtime and do not publish it inside a public image;
- capture the final bind map in the run record;
- avoid binding a host `.venv` into the SIF.

## 10. BABS as the scalable execution and provenance layer

BABS replaces the proposed Snakemake escalation path. Its [official overview](https://pennlinc-babs.readthedocs.io/en/stable/overview.html) describes three required inputs: DataLad BIDS dataset(s), a DataLad dataset containing the containerized BIDS App, and a YAML execution configuration. It submits and audits subject/session jobs on Slurm and records provenance including the exact Singularity command, input datasets, BIDS App, and results.

The division of responsibility should be:

| Layer | Responsibility |
|---|---|
| `envs/pixi.toml` and `envs/pixi.lock` | Developer/orchestration environments, local tasks, tests, and exact package solutions. |
| OCI/Apptainer recipe | Build the actual processing environment and BIDS App entry point from a selected Pixi environment plus system software such as FreeSurfer. |
| DataLad container dataset | Version and distribute the BIDS App image/SIF used by BABS. |
| `envs/babs/config.yml` | Declare BABS input datasets, BIDS App arguments, Singularity arguments, imported files, compute space, and Slurm resources. |
| BABS | Subject/session selection, Slurm fan-out, job status/audit/resubmission, result merging, and DataLad-backed provenance. |

This imposes a useful interface on the current notebooks: the FreeSurfer/recon-any processing stage should become a non-interactive BIDS App-compatible command with explicit input dataset, output directory, processing level, participant/session selection, and named arguments. Pixi tasks can build and smoke-test this command locally. BABS then executes the immutable SIF at scale rather than asking Pixi tasks to submit or monitor jobs.

The recommended cycle is:

1. Build the BIDS App OCI image from `envs/pixi.toml` and `envs/pixi.lock` with `--locked`.
2. Convert/test the exact OCI digest as a SIF and record its SHA-256.
3. Add that SIF to the DataLad container dataset used by BABS.
4. Update and review `envs/babs/config.yml`, including input DataLad dataset locations/versions, app arguments, FreeSurfer license binding, scratch behavior, and Slurm resources. BABS's [configuration documentation](https://pennlinc-babs.readthedocs.io/en/stable/preparation_config_yaml_file.html) defines these fields and the generated Singularity invocation.
5. Use the separate Pixi `babs` environment to run BABS setup checks, submission/status operations, and merge. Pin the BABS version in `envs/pixi.lock` like every other orchestration dependency.
6. Treat the BABS/DataLad history as the participant-level execution provenance; do not duplicate that function in Pixi's task cache.

The `babs` Pixi environment is intentionally separate from `extract` and `analysis`. BABS currently requires a newer Python than the legacy extraction environment and has a different lifecycle. This is another concrete benefit of a multi-environment Pixi workspace.

## 11. `pixi-pack` as a narrower HPC/distribution option

[`pixi-pack`](https://pixi.prefix.dev/latest/deployment/pixi_pack/) can download the locked Conda packages into a transferable archive and recreate an environment without requiring Conda or micromamba on the target. It is valuable for offline nodes and for archiving a package environment more strongly than a URL-only lock.

It is not equivalent to a container:

- a pack can only be unpacked on the matching target platform;
- it does not capture the base OS or unrelated host tools;
- PyPI support is limited to wheels, not source distributions;
- compatibility of manually injected wheels is not verified;
- FreeSurfer licensing and system-level components still require separate handling.

Use it for a Python/Conda-only analysis environment or as a fallback when containers are prohibited. For the complete FreeSurfer pipeline, prefer an immutable SIF.

## 12. Comparison with mainstream alternatives

| Alternative | Relative advantage | Relative issue | Recommendation here |
|---|---|---|---|
| **uv** | Excellent Python speed and a simple `.venv` workflow; native Python project locking. | Python-focused; it does not by itself solve the compiled Conda/neuroimaging environment. It also auto-locks on `uv run`, so it does not avoid the mutation-policy question. | Replace the direct workflow with Pixi. Note transparently that Pixi currently uses uv internally for PyPI resolution/building. |
| **Conda/mamba + `environment.yml`** | Familiar on HPC and broadly understood. | `environment.yml` normally expresses constraints and causes a new solve; it is not an exact package identity. | Pixi is more integrated. If institutional adoption is a concern, `conda-lock` is a credible alternative: its [official documentation](https://conda.github.io/conda-lock/) provides multi-platform, solveless locks with pip support, but tasks remain separate. |
| **Nix** | Stronger declarative system construction and reproducible environment reuse, as described by the [official Nix documentation](https://nix.dev/). | Higher packaging and operational burden; less likely to fit NIH HPC policy and licensed neuroimaging tools without substantial work. | Not the first conversion step. Pixi plus a container is more incremental; reconsider Nix only if system-level reproducible builds become the dominant requirement. |
| **Docker/OCI** | Captures the user-space OS and environment as an immutable digest and is good for CI/distribution. | Heavy artifacts; Docker is not the Biowulf runtime; data and host drivers still cross the boundary. | Use as the canonical build path and convert/pull by digest to SIF. |
| **Apptainer/Singularity** | HPC-compatible immutable SIF, appropriate for licensed neuroimaging execution. | Build permissions and OCI behavior differ by cluster; bind mounts can reintroduce host state. | Use for production HPC execution, with Pixi inside the build rather than as a per-job resolver. |
| **Make** | Ubiquitous, transparent, and mature file-based dependency semantics. | Does not manage environments; shell portability and timestamp semantics require care. | Pixi tasks are sufficient initially and keep environment plus commands together. Make remains a valid small alternative if task behavior outgrows Pixi. |
| **BABS** | Purpose-built BIDS App execution over DataLad datasets with Slurm submission, status/audit/resubmission, result merging, and provenance. | Requires the processing stage to expose a BIDS App-compatible container interface and has its own configuration/project lifecycle. | Use for all subject/session-scale HPC processing. Manage the BABS CLI itself in a distinct Pixi environment; do not recreate its scheduler/provenance behavior with Pixi tasks. |
| **DataLad run/rerun** | Links commands, inputs, outputs, and dataset history, which addresses result provenance better than a task runner. | It does not solve packages or replace workflow scheduling. | Complementary, not competing. A Pixi task can be invoked within a provenance-capturing DataLad run. |

Pixi's notable additional risks compared with mature alternatives are:

1. **Bootstrap/version coupling.** Lock formats can move forward faster than an institution's installed Pixi. Pin and archive the Pixi bootstrap or use a container/pack that does not require Pixi at runtime.
2. **Hidden configuration.** User/system Pixi configuration, mirrors, TLS roots, or an ignored project config can affect resolution and retrieval. Use explicit channels and config isolation in automated runs.
3. **Mixed ecosystem complexity.** Conda/PyPI name mapping and precedence are powerful but can surprise reviewers. State which packages deliberately come from Conda and review lock diffs.
4. **Supply-chain scope.** A checksum proves retrieved bytes match the lock, not that a package is safe. Pixi does not provide a complete vulnerability scanner, and source dependencies require additional trust.
5. **Task-runner overreach.** Inputs/outputs and caching may make Pixi look like a full research workflow system. Use BABS/DataLad—not Pixi's task cache—for subject/session-scale scheduling, audit, and provenance.
6. **Repository availability.** A lock points to artifacts; it does not ensure they remain downloadable indefinitely. Preserve package archives or immutable images for releases.

None of these is a reason to reject Pixi. They define the controls needed to use it honestly within STAMPED.

## 13. Proposed conversion phases

### Phase 1 — tracked environment

- select/pin a Pixi version;
- create `envs/pixi.toml`, add the tracked root symlinks, and migrate inline `uv` requirements into an `extract` feature;
- define distinct `extract`, `analysis`, `dev`, and `babs` environments;
- track `envs/pixi.lock` and `envs/.pixi/.gitignore`; ignore realized `envs/.pixi/` contents plus `.venv/`;
- reproduce preserved extraction and analysis baselines with `--locked` from an empty `envs/.pixi/`.

### Phase 2 — actionable local analysis

- replace hardcoded paths with tracked configuration and logical input/output roots;
- refactor result-producing notebook cells into parameterized scripts;
- add `smoke`, `extract-stats`, `make-figures`, `verify-results`, and `reproduce` tasks;
- make tasks non-interactive and parameterized so DataLad or BABS can invoke them;
- add Pixi task inputs/outputs only where their skip cache is useful, while declaring provenance inputs/outputs separately in DataLad/BABS;
- let routine CI reuse caches but run with `--locked`; reserve a clean locked realization for releases/milestones or a scheduled assurance job.

### Phase 3 — immutable system environment

- add a digest-pinned OCI recipe that installs with `--locked`;
- capture FreeSurfer and other system-level components and their license/access handling;
- publish the image by digest and verify it;
- convert/pull it to SIF, record the SIF SHA-256, and test it on Biowulf.

### Phase 4 — BABS provenance and distribution

- make the processing container a BIDS App and add its SIF to the DataLad container dataset;
- track `envs/babs/config.yml` and use BABS for subject/session Slurm execution, audit, retry, and merge;
- link local/non-BABS stages with `datalad run` where appropriate;
- ensure every run resolves to Git commit, Pixi lock hash, image/SIF identity, DataLad input state, parameters, job records, and outputs;
- archive releases, images or packs, and license/access metadata;
- run periodic clean-room reproduction rather than relying on a long-lived developer prefix.

## 14. Acceptance criteria for Pixi's part of the STAMPED conversion

Pixi should be considered successfully integrated only when all of these are true:

- [ ] Root `pyproject.toml` contains package metadata but no competing `[tool.pixi.*]` workspace.
- [ ] `envs/pixi.toml` is the sole Pixi manifest; tracked root `pixi.toml`, `pixi.lock`, and `.pixi` symlinks point into `envs/`.
- [ ] Pixi is invoked from the repository root without `-m envs/pixi.toml`, preserving one workspace-root interpretation.
- [ ] `envs/pixi.lock` is tracked and can be reviewed or filtered together with other `envs/` changes.
- [ ] `envs/.pixi/.gitignore` preserves the symlink target directory while realized contents and `.venv/` remain ignored and disposable.
- [ ] environment authors edit `envs/pixi.toml` directly; automated checks verify that manifest-mutating commands have not replaced the root symlinks.
- [ ] legacy extraction, current analysis/development, and BABS orchestration use distinct named environments where their requirements differ.
- [ ] the adopted Pixi version is constrained, and the exact build version is recorded.
- [ ] routine commands and all automated builds use `--locked`.
- [ ] CI never updates the lock implicitly; routine CI may reuse a cache, while release/milestone testing performs a clean locked realization.
- [ ] `--frozen` is not used to bypass a manifest/lock contradiction.
- [ ] package source/channel choices are explicit.
- [ ] no inline `uv` requirements or untracked environment definition competes with the manifest.
- [ ] result-producing tasks/entry points are non-interactive and parameterized; DataLad/BABS explicitly declares or records their inputs and outputs.
- [ ] Pixi `inputs`/`outputs` and task caching are treated only as optimizations, never as DataLad provenance.
- [ ] the OCI recipe copies the tracked root symlinks plus their `envs/` targets, installs with `--locked`, and pins builder/runtime `FROM` images by digest.
- [ ] the production OCI digest and SIF SHA-256 are recorded.
- [ ] package-download cache, realized Pixi prefix, and task cache are documented as three different disposable optimizations.
- [ ] a persisted realized prefix is reconciled with `--locked` and discarded for incompatible image/platform changes or clean-room testing.
- [ ] the BIDS App SIF is tracked in the DataLad container dataset and `envs/babs/config.yml` is versioned.
- [ ] BABS/DataLad provenance records code, container, data, command/parameters, and result identities for subject/session processing.

Meeting this checklist would substantially improve Tracking, Actionability, Portability, and Ephemerality. It would still not by itself satisfy Self-containment or Distributability until the data, neuroimaging binaries/licenses, results, and persistent retrieval references identified in the original review are brought inside the research-object boundary.

## Sources consulted

Primary documentation used for the technical assessment:

- [Pixi lock files](https://pixi.prefix.dev/latest/workspace/lock_file/)
- [Pixi workspace and lock location](https://pixi.prefix.dev/latest/first_workspace/)
- [Pixi `run` command and update-control flags](https://pixi.prefix.dev/latest/reference/cli/pixi/run/)
- [Pixi `lock` command](https://pixi.prefix.dev/latest/reference/cli/pixi/lock/)
- [Pixi `update` command](https://pixi.prefix.dev/latest/reference/cli/pixi/update/)
- [Pixi manifests and discovery priority](https://pixi.prefix.dev/latest/reference/pixi_manifest/)
- [Pixi with `pyproject.toml`](https://pixi.prefix.dev/latest/python/pyproject_toml/)
- [Pixi tasks](https://pixi.prefix.dev/latest/workspace/advanced_tasks/)
- [Pixi container deployment](https://pixi.prefix.dev/latest/deployment/container/)
- [Pixi configuration and HPC-oriented caches](https://pixi.prefix.dev/latest/reference/pixi_configuration/)
- [Pixi supply-chain security guidance](https://pixi.prefix.dev/latest/security/)
- [`pixi-pack`](https://pixi.prefix.dev/latest/deployment/pixi_pack/)
- [uv locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/)
- [`conda-lock`](https://conda.github.io/conda-lock/)
- [Apptainer Docker/OCI support](https://apptainer.org/docs/user/latest/docker_and_oci.html)
- [Apptainer build command](https://apptainer.org/docs/user/latest/cli/apptainer_build.html)
- [NIH Biowulf Singularity/Apptainer guidance](https://hpc.nih.gov/apps/singularity.html)
- [BABS overview](https://pennlinc-babs.readthedocs.io/en/stable/overview.html)
- [BABS execution configuration](https://pennlinc-babs.readthedocs.io/en/stable/preparation_config_yaml_file.html)
- [BABS package metadata](https://pypi.org/project/babs/)
- [DataLad `run`](https://docs.datalad.org/en/latest/generated/man/datalad-run.html)
- [Nix documentation](https://nix.dev/)
