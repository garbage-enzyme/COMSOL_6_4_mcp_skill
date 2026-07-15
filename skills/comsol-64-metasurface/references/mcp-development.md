# MCP development and release engineering

## Contents

- Capability profiles
- Public schema discipline
- Typed derived-model mutations
- Solver-free isolation
- Deployment identity
- Packaging and restart gates
- Regression design

## Capability profiles

Keep the default profile compact. Separate verified core tools from specialized
Wave Optics, documentation, generic mutation, and experimental tools. Report:

- active profile and available profiles;
- exact tool counts and names derived from the live catalog;
- maturity and known limitations;
- whether a tool can construct COMSOL, acquire a lease, spawn a worker, mutate a
  model, or load heavy optional dependencies;
- restart requirement after profile/source changes.

Treat live discovery as authoritative. Do not hardcode historical counts in a
reusable skill.
When supported, select the static profile with `COMSOL_MCP_PROFILE`; reject an
unknown name at startup and require a host restart for profile changes.

Documentation retrieval must remain isolated from solver ownership. Keep lexical
search available if an optional semantic worker is busy, crashes, times out, or
fails its benchmark. Do not advertise multilingual or semantic quality without
frozen evidence.

Instrument control-plane operations with bounded rolling samples, structured
outcome counts, and p50/p95/max latency. Test cancellation/status fairness under
concurrency; do not let retrieval or inventory work starve job control.

## Public schema discipline

For every public tool:

- use typed bounded inputs rather than arbitrary property setters;
- reject unknown fields and invalid combinations;
- normalize semantically equivalent inputs before hashing;
- cap response, history, queue, sample, and artifact sizes;
- report unavailable evidence explicitly;
- snapshot schemas and profile membership;
- avoid importing COMSOL or heavy optional libraries during discovery.

Keep raw evidence separate from caller policy. A tool should not reinterpret old
results when a new policy is introduced.

## Typed derived-model mutations

Never expose unrestricted mutation as a verified tool. Require:

1. immutable source path/hash;
2. provenance-tracked derived model ID and backing path;
3. exact current/pre-state hash;
4. preview of typed changes and likely topology effects;
5. serialized apply;
6. post-write readback;
7. best-effort rollback of every affected node;
8. dirty/unusable state when rollback cannot be proved.

Examples of safe typed surfaces include final union/assembly settings, complete
Block size/position vectors with units, and PeriodicStructure incidence settings
across the parent and locked port children.

Block edits should not implicitly rebuild geometry or mesh. Make transitions
explicit. A finalization edit that runs geometry must persist before/after
topology evidence.

## Solver-free isolation

Place validation, policy normalization, fingerprints, journal replay, process
identity checks, and artifact-manifest logic in pure modules. Test them without
constructing `mph.Client`.

Isolate optional manual/PDF/embedding/plot work in bounded subprocesses with
hard deadlines and exact identity. A failure must not block solver status,
cancellation, or job controls.

Before broad process testing on Windows, prove the launcher reports the actual
worker identity and that every child uses hidden-window creation flags.

## Deployment identity

Package version alone cannot prove new code is live, especially when installers
skip the same version. Expose a path-redacted deployment identity containing:

- source-tree versus installed-site-package classification;
- package version;
- dynamic tool-catalog hash;
- frozen public-schema hash;
- frozen profile-membership hash;
- build/manifest identity sufficient to distinguish source revisions.

All concurrent fresh processes from one installation should report one identical
identity. Discovery must not start COMSOL or import heavy semantic dependencies.

## Packaging and restart gates

After source changes:

1. require a clean reviewed tree;
2. compile and run the dependency-only suite;
3. build wheel and source distribution;
4. install the wheel non-editably into a fresh ASCII-path environment;
5. run `pip check` and installed discovery/schema/profile comparisons;
6. force reinstall into the target environment when content changed at the same
   package version;
7. restart the MCP/CLI host because stdio servers do not hot reload;
8. verify live deployment identity before any COMSOL action.

When sibling stdio hosts are present, restart only the process whose PID,
creation time, command, and parent identity match the intended deployment.
Never terminate processes by a broad executable-name or command-substring match.

Verify the installed restart through a fresh MCP `initialize`, tool discovery,
and capability receipt rather than an import alone. Record the active profile,
tool count, durable job types, deployment classification, and
`COMSOL connected=false` before licensed work. If the original transport cannot
reattach after its server exits, a bounded fresh-stdio receipt is valid restart
evidence.

Do not install/restart while another standalone solver can spawn child
processes. After restart, query ownership first.

Run licensed release gates serially. Record COMSOL PID sets and lease/collision
state before and after; preserve source hash and cleanup evidence.

## Regression design

Use three layers:

- deterministic pure/fake-clock tests for every state-machine branch;
- Windows process-only stress for scheduling, sharing violations, PID reuse,
  coordinator loss, and hidden-window behavior;
- minimal version-pinned real-COMSOL gates for clientapi and physical behavior.

Archive a failing durable job directory and phase-latency summary before retry.
Require repeated clean runs for timing-sensitive acceptance; do not "fix" a race
by only increasing a timeout.

Useful sanitized fixtures include:

- analytical electrostatics regression;
- valid and invalid periodic mesh recipes;
- reference-air polarization;
- passive physical flux closure;
- source immutability and typed rollback;
- cancellation/resume without duplicates;
- lexical documentation retrieval;
- resource warning/refuse/recovery transitions.

Keep fixtures relative and sanitized: no usernames, home directories, host
thresholds, credentials, private spectra, or internal project/phase labels.
