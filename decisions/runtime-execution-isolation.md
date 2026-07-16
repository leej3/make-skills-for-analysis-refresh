# Disposable and isolated scientific execution

- **Status:** accepted
- **Decision date:** 2026-07-16
- **Last reviewed:** 2026-07-16

## Question

Does an immutable accepted SIF provide sufficient portability, or must each result-producing execution also exclude undeclared host, home, cache, credential, mount, network, and prior-output state?

## Normative status

This implementation adopts the STAMPED E.1 disposable-environment SHOULD and uses the controls below to enforce the explicitness required by P.1–P.3 and A.1. These are project implementation rules, not optional hardening.

When a platform makes a control impossible, record the unavoidable state, test its effect, and assess the limitation explicitly. Unknown or unrecorded host state fails acceptance.

## Decision

Every result-producing execution starts with fresh transient state and receives only:

- the accepted exact SIF;
- declared exact input datasets;
- tracked configuration;
- a new output location;
- fresh work and scratch space;
- a declared application licence file when required; and
- explicit host resource interfaces.

Authenticated retrieval happens before scientific execution. Retrieval credentials, the host home, undeclared working directories, persistent caches, implicit project binds, prior outputs, and unnecessary network access must not be available to the scientific process.

Image identity and execution isolation are independent. An accepted immutable SIF does not make inherited host state reproducible, and clean containment does not identify an unregistered image.

## Required controls

### Stage retrieval

- Retrieve the SIF and controlled inputs before starting computation.
- Keep credentials in the retrieval context; never pass them to the scientific process or write them to configuration, commands, logs, images, or provenance.
- Verify the staged SIF and input identities before execution.

### Start with fresh state

- Create new work, scratch, cache, temporary-home, and output locations for each execution.
- Do not reuse a prior output directory for a result-changing run. Treat rerun and resume behavior as an explicit application or campaign decision with provenance.
- Retain only declared outputs, required logs, and provenance. Destroy transient work, scratch, cache, and temporary-home state after the run.

### Contain the process

- Use Apptainer/Singularity `--cleanenv` with `--containall`, or a demonstrably equivalent combination.
- If full containment is unavailable, use controls such as `--no-home` or `--no-mount home,cwd` and provide an empty temporary home or intentional read-only home from the image.
- Never mount the host home or inherit an undeclared host working directory.

### Control caches

- Disable persistent container and application caches.
- If an application requires a writable cache, direct it to fresh scratch and destroy it after execution.
- Never bind `.venv`, `envs/.pixi`, a package cache, or another mutable environment prefix into a result-producing run.

### Declare mounts

- Disable configured host bind paths where the platform permits.
- Declare every project bind. Mount inputs and configuration read-only; make only the required output, work, and scratch paths writable.
- Inspect and record the effective mount table during the pilot.
- Record unavoidable system binds and demonstrate that no unknown mount supplies execution-essential state.

### Control network and host interfaces

- Disable network access when computation does not require it.
- When network access is required, record the endpoint, purpose, and information obtained as an explicit input or host dependency.
- Record unavoidable kernel, accelerator, driver, scheduler, filesystem, security-policy, locale, and other host coupling in the cluster profile and STAMPED assessment.

## BABS integration

Apply equivalent controls in BABS configuration and generated job templates. During the pilot:

1. inspect the expanded participant command and script;
2. record the environment-cleaning, containment, bind, cache, network, work, scratch, and cleanup settings;
3. inspect the effective mount table from a representative job;
4. verify the exact SIF, inputs, configuration, and host resource interfaces; and
5. execute from a clean clone with an empty home/cache and no prior output.

Do not assume that a BABS template inherits these controls merely because it invokes Singularity or Apptainer. Validate the generated command actually submitted to the scheduler.

## Declared exceptions

A required application licence file may be mounted read-only when it cannot lawfully or appropriately be distributed. It must be treated as an identified input without exposing secret content in public records.

A required network endpoint or unavoidable system mount may be accepted only when it is declared, tested, and represented in the cluster profile and relevant P.1 assessment. An exception does not become an undocumented default and must be reconsidered for a different host.

## Acceptance criteria

- A pilot records the expanded command, environment policy, effective mounts, network policy, exact SIF, inputs, configuration, and host interfaces.
- No host home, undeclared working directory, persistent cache, prior output, or retrieval credential is exposed to the scientific process.
- Inputs and configuration are read-only, and only declared output, work, and scratch paths are writable.
- Every unavoidable host dependency is recorded and assessed; no unknown host state is required.
- A representative clean-clone execution succeeds with fresh transient state.
- Cleanup leaves only declared outputs, required logs, and provenance.
- BABS-generated executions demonstrate the same controls rather than relying on an uninspected template assumption.
