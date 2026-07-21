# Durable runtime and resource safety

## Contents

- Ownership and preflight
- MCP transport serialization
- Shared Desktop and attached Server
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

## MCP transport serialization

Issue only one request at a time to a given COMSOL MCP stdio server. This rule
also covers `capabilities`, session status, ownership status, job status, and
polling. Do not infer that read-only or control-plane tools are safe to batch;
server-side concurrency classes do not guarantee transport-level parallelism.

After a host-side wait is interrupted, assume the original request may still be
running. Do not issue a compensating start, disconnect, reset, or ownership
mutation until the original call reaches a known terminal state or the MCP host
is deliberately restarted. A long parallel-call sample is transport evidence,
not proof that COMSOL, the JVM, or a solver is running.

## Shared Desktop and attached Server

Use a protected shared profile only when the user explicitly requests
collaboration with a user-owned local COMSOL Server and Desktop. Keep it
default-off. A safe first-release topology is one local user, one manually
started Server, one connected Desktop window, and one exact server-held model.
Do not treat a direct `mph.Client(port=...)` connection as equivalent to this
lifecycle.

For COMSOL 6.4 on Windows, the user can start **COMSOL Multiphysics Server 6.4**
from COMSOL Launchers. Persistent repeated-client operation can use:

```text
comsolmphserver -multi on -port 2036
```

Record the actual listening port from the Server console and leave the console
running. The usual default is 2036, but an occupied or configured port can
differ. The protected MCP path must never start or terminate this external
Server.

In Desktop, use **File > COMSOL Multiphysics Server > Connect to Server**, select
`localhost`, and enter the exact port. On one accepted Windows installation,
the dialog automatically populated locally stored username/password fields;
this is a UX observation, not a portable guarantee. Never put credentials or
login files in prompts, logs, screenshots, or receipts. The lower-left
`localhost:<port>` indicator is useful user evidence; if it disappears, Desktop
is disconnected.

Before constructing an MPh client:

1. discover the default-off shared profile and static feature gate;
2. restart the MCP host after changing that profile or gate;
3. perform two bounded process/listener probes;
4. require the user to confirm that Desktop shows the declared endpoint;
5. attach non-owningly and enumerate bounded server model metadata;
6. adopt one exact tag plus available label/path/unsaved expectations;
7. establish and retain exact server, model, lock, and revision identities.

Accept only the `6.4.0.*` release line for a surface calibrated to that family.
A final build change, such as another `6.4.0` automatic-update build, can retain
the release-line conclusion while preserving an exact-build warning. A third
numeric component change such as `6.4.1.*`, an older release, a mixed
Desktop/Server family, or unreadable version must fail closed.

State handling must remain explicit:

- no Desktop/Server or a still-starting process is retryable, not attachable;
- a connected Desktop with no server model needs the user to create, transfer,
  or open one model while connected;
- an unsaved blank model may be adopted by exact tag for bounded interactive
  work, but it has no immutable source-file identity;
- a standalone-only Desktop model is invisible until explicitly transferred to
  the Server;
- multiple Desktop windows, candidate Servers, or ambiguous models require the
  user to reduce or identify the topology; never choose “the first” item;
- preserve wildcard listener evidence. Connecting through `localhost` does not
  turn a `0.0.0.0` or `::` listener into loopback-only exposure.

Use explicit turns. Unlock before a Desktop edit, then re-inventory/relock and
read back a new revision. During a longer agent mutation or solve, COMSOL may
lock Desktop editing and show an occupied-model/busy warning. Short writes or
read-only calls may finish without that warning. The native dialog is a Server-
busy effect, not proof that MCP identity, revision, evidence, or cleanup guards
passed.

Keep three file roles separate:

- **immutable source**: caller-owned, readable under an allowed root, exact
  SHA-256, never overwritten within the formal identity;
- **open working model**: the mutable in-memory Server model visible in Desktop;
- **Save Copy snapshot/checkpoint**: a new collision-free file under the owned
  ASCII artifact root, with size/hash/manifest evidence, that does not change
  the working model's main path.

COMSOL/Windows may lock an open `.mph`. Save As commonly switches the working
model to the new file, so it does not automatically create a distinct immutable
source. Use a true Save Copy or separately preserved source. For an unsaved
model, create and hash a distinct source before formal durable work.

Long or multi-point attached work must use the existing durable job controls,
not a foreground loop. Require an `automation_exclusive` lock, immutable source,
expected revision, and explicit user confirmation. The worker checks revisions
between points. Cancellation stops only the owned attached worker/client and is
terminal only after owned cleanup plus preservation of the external Server,
Desktop, listener, and model.

Normal unlock/detach leaves the user resources running; the user normally does
not restart the Server between collaboration steps. Only the user closes or
restarts those resources after evidence is saved. This release is local-only,
does not support simultaneous editing, and does not turn visible GUI output into
scientific validation.

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

Order nested sweeps by the parameters that invalidate geometry, diffraction
orders, or mesh. Put a mesh-dependent parameter on the outside and reuse one
verified mesh only across inner parameters that cannot alter it. Persist the
observed mesh identity under that same dependency key so resume rejects a
changed mesh without imposing false equality across unrelated mesh states.

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
If a launcher is moved into an archive such as a project `ps1` subdirectory,
recompute every relative root from the launcher's new location and run an exact
Windows PowerShell 5.1 preflight there. A launcher that validated before the
move may otherwise resolve a nonexistent driver.

Estimate unattended runtime from a smoke point that matches the solver backend,
mesh level, physics, and memory mode of the planned run. Do not reuse timings
from a different solver version, factorization path, mesh, or in-memory versus
process-isolated configuration. Record per-level timings, update a bounded
rolling estimate from completed points, and admit the next point only when that
estimate plus margin fits the remaining wall budget.

For a geometry sweep whose cell size or feature scale changes materially, smoke
at least the expected smallest and largest mesh/resource cases. Verify topology,
periodic selections, mesh identity, passive evidence, and cleanup at both ends;
one representative geometry does not bound runtime or memory. Use the endpoint
timings only as an initial bounded estimate and replace them with rolling
completed-point telemetry.

When an operator requires an absolute wall limit, use two clocks. Set a shorter
worker limit that stops only between flushed point rows, then let the foreground
launcher enforce the later absolute deadline against the exact owned process
tree. The margin must exceed the expected longest point plus cleanup time. If
the absolute guard fires, preserve prior durable rows, verify owned-process and
lock absence, and write an atomic termination receipt; never label the
interrupted point complete.

Some standalone clients leave JVM/helper threads alive after all outputs are
saved. After flushing every artifact, saving through the Java clientapi, and
releasing owned resources, `os._exit(0)` is acceptable for a dedicated worker.
Do not call `disconnect()` merely because a standalone client's remote port is
`None`; clear owned models/resources and let the dedicated process exit.
