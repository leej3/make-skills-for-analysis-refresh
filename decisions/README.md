# Decision records

These documents preserve the rationale and responsibility boundaries used to develop the reusable STAMPED neuroimaging skill. The [skill](../skills/stamped-neuroimaging-analysis/SKILL.md) is the concise operational product; the [conversion plan](../conversion-plan.md) applies it to `STAMPED-dl_morphometrics_biases`.

Revise an affected decision before propagating a policy change through the skill or conversion plan. Analysis repositories do not need to copy these records unless a project-specific departure genuinely requires one; their repository-local skill and conversion plan are sufficient policy.

- [Why Pixi belongs in the analysis](pixi-role.md) — the complete Pixi responsibility boundary, lock/update policy, image-build relationship, and runtime limits.
- [Pixi tasks, DataLad provenance, and BABS operations](pixi-tasks-and-provenance.md) — static meta-graphs, provenance-producing leaves, prohibited caching, and BABS meta-provenance.
- [Candidate, qualified, and accepted SIF policy](runtime-image-strictness.md) — debugging freedom, runtime qualification and registration, and the strict boundary for retained results.
- [Disposable and isolated scientific execution](runtime-execution-isolation.md) — clean containment, explicit mounts and host interfaces, fresh transient state, and the BABS execution boundary.
- [Role of MechaBABS](mechababs-role.md) — patterns adopted now, reasons authoritative execution is deferred, and promotion gates for an experimental pilot.
