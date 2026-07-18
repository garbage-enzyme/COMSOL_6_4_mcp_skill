---
name: comsol-64-metasurface
description: COMSOL Multiphysics 6.4+ and MPh 1.3.1 operations through a COMSOL MCP server or standalone/clientapi, shared Desktop/attached-Server collaboration, periodic Wave Optics and metasurface FEM, durable staged solves and bounded validation matrices, evidence validation, and safe runtime practice. Use when driving COMSOL through an MCP server or mph.Client, collaborating with a user-owned local Server/Desktop model, debugging clientapi/periodic-mesh/port/material/study failures, running resumable sweeps or small durable evidence matrices, or auditing polarization, passivity, power closure, wavelength synchronization, mesh convergence, provenance, resource admission, and solver ownership.
---

# COMSOL 6.4+ operations

Use this file as the short entry point. Read only the reference modules required
for the current task. All paths below are relative so the same folder works in
Claude Code, Codex CLI, and opencode.

## Mandatory operating rules

1. Inspect live capabilities and solver ownership before any COMSOL action.
2. Keep one solver owner. If a lease or external COMSOL/MPh process exists, do
   not construct another client.
3. Treat source models as immutable. Apply mutations only to provenance-tracked
   derived copies and verify the source SHA-256 afterward.
4. Use one point per solve for fragile or long sweeps. Persist each validated row
   with flush and `fsync`; resume only exact configuration identities.
5. Never infer physical polarization from `S/P`, energy closure from shared
   internal normalizations, convergence from a fixed-wavelength amplitude, or
   solver progress from CPU/disk activity alone.
6. Require explicit caller policy for scientific classification and resource
   thresholds. Do not invent host defaults.
7. Stop before a visual mode claim unless an image-capable reviewer has received
   the exact bounded artifacts and returned a review receipt.
8. Keep responses, journals, queues, retries, subprocesses, and artifact sizes
   bounded. Fail closed when process identity, cleanup, telemetry, or rollback
   evidence is uncertain.
9. Keep research scope, publication standards, and project priorities
   caller-owned. Describe capabilities and evidence contracts rather than a
   default project strategy.
10. In a shared Desktop/Server session, require explicit local endpoint and
    model adoption, take turns rather than editing simultaneously, preserve the
    user's Server/Desktop/model on detach, and treat GUI visibility as distinct
    from verified scientific evidence.
11. Use one shared project-root `settings.json` for every agent. Do not create
    agent-specific copies of profile, runtime, path, Java, shared-server, or
    evidence settings; pass only `COMSOL_MCP_SETTINGS_PATH` when the host cannot
    preserve the project path.

## Shared settings contract

The COMSOL MCP project groups startup settings by function in its root
`settings.json`, with English `_comment`/`_comment_*` fields because standard
JSON has no comment syntax. Missing entries use safe defaults. An illegal value
falls back only that entry and is reported through `capabilities` or
`evidence_integrity_status` as a bounded `settings_errors` item; malformed JSON
falls back to the complete safe default document and reports the error. Check
`project_settings.configuration_state` before relying on a profile or path.

For shared Desktop/Server work, set `profile.name` to `desktop_shared` and
`shared_server.enabled` to `true` in that same file, then restart the MCP host.
Evidence-integrity checks remain default-on in
`evidence_integrity.checks`; only explicit JSON `false` is an exploration opt-out
and it must propagate `strictly_verified: false`. The old individual environment
variables are compatibility overrides, not the normal multi-agent configuration.

## Reference router

Read each selected file completely before acting.

| Task | Read |
| --- | --- |
| `ModelClient` overloads, components, geometry probing, electrostatics, heat transfer, study/result basics | [clientapi-core.md](references/clientapi-core.md) |
| `PeriodicStructure`, `rdir1`, incidence angles, polarization, periodic mesh, oblique cells | [wave-optics-periodic.md](references/wave-optics-periodic.md) |
| Drude/loss signs, layered boundaries, dispersive sweeps, PML, manual Floquet | [materials-boundaries.md](references/materials-boundaries.md) |
| Solver ownership, shared Desktop/attached Server, durable jobs/validation matrices, cancellation, Windows load stability, resource telemetry/admission | [durable-runtime.md](references/durable-runtime.md) |
| Default-on evidence integrity, R/T/A, flux closure, polarization evidence, wavelength sync, provenance, convergence, fields | [validation-evidence.md](references/validation-evidence.md) |
| MIM, gratings, nanopillars, parameter scans, field export, common modeling recipes | [workflow-recipes.md](references/workflow-recipes.md) |
| Error signatures and the smallest safe diagnostic | [troubleshooting.md](references/troubleshooting.md) |

For a task spanning several areas, read only their union. Examples:

- Periodic angle sweep: periodic + durable runtime + validation evidence.
- New dispersive metasurface model: clientapi + periodic + materials + evidence.

## Default execution sequence

1. Discover the active tool/profile surface; treat live discovery as authority.
2. Query ownership/status without starting COMSOL.
3. Hash the source and normalize the exact requested configuration.
4. Run solver-free preflight first: topology, selections, expressions, policy,
   artifact paths, resource availability, and immutable identity.
5. Acquire ownership once; then create/load the client and derived model.
6. Run the smallest diagnostic or one-point gate before a sweep.
7. Persist raw values and assumptions before interpretation.
8. Release model/client/worker/descendants/port/lease and verify absence.
9. Re-hash the source and archive a path-redacted evidence manifest.

## Acceptance language

Use `verified`, `measured`, `derived_from_declared_convention`, `label_only`,
`unknown`, `not_requested`, and `not_applicable` precisely. A successful native
call is not by itself physical validation. A requested cancellation is not a
terminal cancellation. A read-only mesh audit is not node-equality proof. An
all-air result proves port/mesh consistency, not angle mapping or target
polarization in the physical structure.
