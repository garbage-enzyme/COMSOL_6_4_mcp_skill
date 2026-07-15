# Durable runtime and resource safety

## Contents

- Ownership and preflight
- Durable job artifacts
- Cancellation and recovery
- Windows load hardening
- Staged sweep persistence
- Bounded validation matrices
- Resource admission and telemetry
- Unattended launchers

## Ownership and preflight

Treat same-host coordination as optional. On every host:

1. Discover available capabilities.
2. Inspect the shared runtime and solver lease.
3. Inventory external MPh/COMSOL processes with exact identity evidence.
4. Refuse a new client when another owner or an uncertain collision exists.

Use a local ASCII runtime root. Different machines or runtime roots do not share
ownership. Network-share locking is not a distributed lease.

When supported by the companion server, select the shared root with
`COMSOL_MCP_RUNTIME_DIR`. A compatibility `COMSOL_MCP_JOBS_DIR` override must
resolve exactly to the runtime root's `jobs` directory; refuse conflicting roots.

Mutation paths require a fresh, complete process inventory. Read-only status may
use a recent labeled cache with a strict deadline, but it must report cache age
and incompleteness. Never allow an incomplete/expired inventory to acquire a
lease, start a worker, refresh ownership, or recover an orphan.

## Durable job artifacts

Store each job under a stable job directory containing:

- immutable normalized `spec.json` and source/configuration hashes;
- atomic `state.json`;
- append-only event/resource/result journals;
- attempt-bound control requests;
- checkpoint/model artifacts;
- bounded worker logs and summaries.

Write JSON through temporary-file replacement, flush and `fsync` the file, then
sync the directory when required. Never replace an atomic artifact with an
in-place write.

Resume only rows whose full configuration identity matches. Deduplicate exact
valid point identities, not filenames or scalar wavelengths. Retry errors and
partial rows.

## Cancellation and recovery

Bind every cancellation request to an exact attempt and worker identity: PID,
creation time, command signature, and captured descendant evidence.

Use explicit phases such as request, native/cooperative grace, terminate,
force-kill, cleanup verification, and terminal commit. Persist first-entry times,
budgets, observed latencies, process identities, and blockers.

Use a native cancellation API such as `ProgressContext.cancel()` only for the
exact COMSOL/MPh build on which it was probed. On any other build, report the
profile mismatch and use exact-identity owned-process fallback; never assume a
native API is portable across versions.

Commit `cancelled` only after proving:

- worker and owned descendants are absent;
- recorded server port is clean;
- the owned solver lease is absent;
- the request targets the active attempt.

A request, native cancel return, missing PID, or coordinator loss is not enough.
Uncertain identity remains nonterminal and fail-closed. Ignore stale-attempt
requests after resume.

On coordinator restart, reconcile from durable evidence. Remove a stale lease or
lock only when exact contents and owner identity prove it is stale and owned by
the job.

## Windows load hardening

Windows readers can temporarily deny replace/unlink access. Apply bounded retries
to every read, atomic replace, heartbeat, release, recovery unlink, and lock
cleanup path. After each retry:

- re-read and validate exact bytes/identity;
- refuse a competing write;
- preserve foreign locks;
- fail closed after the deadline;
- clean only owned temporary files.

Stress durable state with concurrent readers/writers, injected sharing failures,
and native exclusive handles. Require valid terminal JSON and zero owned temp or
lock residue.

Do not assume a virtual-environment launcher PID is the worker PID. Some Windows
launchers create child interpreters and can escape hidden-window flags. Record a
startup identity handshake from the actual worker before accepting ownership.
Use hidden-window creation flags for noninteractive helpers and prove the entire
launcher/child tree stays hidden before broad process tests.

## Staged sweep persistence

Solve one point at a time:

```text
set parameter -> apply point settings -> pre-solve admission -> run study
-> evaluate raw evidence -> validate -> append row -> flush+fsync
-> post-solve telemetry -> checkpoint -> next point
```

Skip completed exact identities before setting parameters. Admit only whole
stages that fit the remaining wall budget plus margin. Keep control polling and
durable-row callbacks bounded so cancellation cannot starve.

Never edit a driver or child script that an active parent process may import or
spawn later. Stop the owner at a durable boundary, defer the change, or create a
new versioned path with new outputs and hashes.

For disk-light point solves, allow successful point models to be removed only
after the child evidence and matching aggregate row have both been flushed and
`fsync`ed. Retain failed-point models and preserve the deletion policy in the
run provenance.

## Bounded validation matrices

Use a matrix job only for a small, explicit set of evidence points. Its
immutable spec must bind source/configuration hashes, exact point settings,
collectors, one artifact identifier per collector, and caller-declared point,
wall-time, and resource caps. Reject unknown, nonfinite, duplicate, or oversized
inputs before acquiring ownership or creating a client.

Serialize exact duplicate submissions under a bounded runtime-root lock and
return the existing job. Reuse the established manager, lease, cancellation,
resource journal, worker, and client paths; a matrix is orchestration, not a
second solver runtime. Matrix-owned settings cannot be overridden by collector
configuration.

Keep attempts isolated. Each collector writes bounded evidence inside its
attempt subtree and a small wrapper manifest that binds the inner artifact's
hash and size. Append hash-chained, flushed, and `fsync`ed result rows. Resume
may skip only complete, policy-evaluated rows with verified artifacts. Malformed,
tampered, partial, or integrity-blocked rows must fail before client creation.

A pre-solve resource refusal is a durable policy outcome, not a solve error, and
must not create a false failed-solve row. Commit the durable result row before
post-solve admission/telemetry. Replaying a verified row as `skip_completed` is
normal recovery behavior.

Real acceptance needs both off-target and target evidence plus a coordinator
restart. For passive scattering, verify bounded `R`, `T`, and `A`, power closure,
wavelength synchronization, source hash, unique row identities, complete
artifact inventory, and absence of lease/collision residue.

A fast read-only process inventory timeout does not prove cleanup. Retry with
the mutation-grade fresh inventory deadline and validate existing durable
evidence without rerunning physics.

## Resource admission and telemetry

Require an explicit caller policy. Do not create machine-specific defaults in a
public skill or server.

Useful policy inputs include:

- minimum available-memory and remaining-commit fractions;
- warning and refusal thresholds;
- runtime-volume free-space thresholds;
- maximum mesh elements/DOF when available;
- total wall budget and minimum time for the next point.

Sample at `pre_mesh`, `post_mesh`, `pre_solve`, `post_solve`, and `recovery`:

- physical available/total memory;
- remaining/limit commit;
- runtime free space;
- worker private bytes and working set;
- CPU-time progress proxy;
- disk and pagefile counters;
- mesh elements/DOF;
- elapsed wall time and latest durable-result timestamp.

Label unavailable metrics explicitly. If a required metric is unavailable, fail
closed. Disk activity alone is not paging or progress evidence; correlate commit,
memory, working set, pagefile I/O, CPU time, and durable timestamps.

Use green/warning/red decisions:

- green: allow;
- warning: require a separate caller confirmation bound to the exact decision;
- red: checkpoint/no-start; do not begin another factorization.

Append telemetry and decisions as separately hashed, monotonic per-attempt
records. Flush and `fsync` each record. Reject sequence gaps, nonmonotonic attempts,
mismatched telemetry/decision hashes, stale confirmations, and oversized
journals. A recovery sample may authorize the same point later while preserving
refusal history. Completed valid points always replay as `skip_completed`.

Build calibration reports only against a caller-declared known-safe baseline.
Compare relative ratios/deltas, preserve missing comparisons, emit no automatic
portable policy, and scope recommendations to that project/host/solver/model.

Do not scavenge temporary artifacts until naming, directory scope, age/PID,
active-writer, and reference proofs are all explicit.

## Unattended launchers

Do not attach long work to an agent process whose lifetime may end. Validate the
driver with compile and zero-work dry-run checks, then create a foreground
operator launcher that:

- validates executable, driver, model, output, and runtime paths;
- refuses duplicate starts and solver collisions;
- declares cores, memory/commit, element/DOF, free-space, and wall gates;
- prints durable log/artifact paths;
- keeps its console open on exit.

On legacy Windows PowerShell, avoid non-ASCII literals in UTF-8 scripts without
a BOM. Derive sibling paths from the script location or emit a verified BOM.

Some standalone clients leave JVM/helper threads alive after all outputs are
saved. After flushing every artifact, saving through the Java clientapi, and
releasing owned resources, `os._exit(0)` is acceptable for a dedicated worker.
Do not call `disconnect()` merely because a standalone client's remote port is
`None`; clear owned models/resources and let the dedicated process exit.
