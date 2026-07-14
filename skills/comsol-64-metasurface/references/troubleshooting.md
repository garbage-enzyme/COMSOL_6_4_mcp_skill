# Troubleshooting index

## Contents

- Startup and ownership
- Clientapi errors
- Geometry and mesh
- Ports and physics
- Materials and studies
- Results and validation
- Runtime and deployment

## Startup and ownership

| Symptom | Smallest safe diagnostic |
| --- | --- |
| First start call times out | Poll status; do not issue a second start while `starting=true`. |
| External collision reported | Inspect exact process identity and lease; do not attach/start another client. |
| Only one client can be instantiated | Restart the Python/MCP host; do not construct another client in the same JVM. |
| Job stuck cancelling | Inspect attempt-bound phases, worker/descendants, port, lease, and coordinator identity; remain nonterminal if uncertain. |
| Visible terminal flashes | Stop broad process tests; verify the launcher/actual child PID handshake and hidden-window flags. |

## Clientapi errors

| Symptom | Likely cause/fix |
| --- | --- |
| `No matching overloads` | Direct-Model overload used on `ModelClient`; inspect Java reflection and use clientapi signatures. |
| Collection is not subscriptable | Convert tags/entities explicitly and call `.get(tag)`. |
| Feature has no `.type()` | Use tag/label/exported metadata; do not assume the direct API. |
| Operation cannot be created | Wrong study/feature type or wrong physics interface; use exact full names and dimension types. |

## Geometry and mesh

| Symptom | Likely cause/fix |
| --- | --- |
| Cannot create `fin` | It is reserved and auto-created; configure the existing finalization feature. |
| Face parameter out of range | Query `faceParamRange()` and use its midpoint. |
| Periodic source/destination incompatible | Prove translation-congruent partitions and use `FreeTri -> CopyFace -> FreeTet`. |
| CopyFace copies no target | Wrong boundary mapping/partition or missing congruence; re-probe centers, normals, adjacency, and translation. |
| Periodic port selects several faces | Cell-side classification used normals only and included internal faces; add bounding-plane coordinate tests. |

## Ports and physics

| Symptom | Likely cause/fix |
| --- | --- |
| `axisx/axisy` undefined | Empty/invalid `rdir1`; select a top-port edge along the intended lattice direction. |
| Missing `ewfd.Eampl*` or `TE/TM` enums appear | Wrong physics interface/namespace; create `ElectromagneticWavesFrequencyDomain`. |
| Periodic port fails near metal | Adjacent domain is not homogeneous/isotropic; use an appropriate layered boundary or redesign the port region. |
| Angle appears ineffective | Verify `alpha1_inc/alpha2_inc` on parent and both ports, evaluate them independently, and test a physical structure across signed directions. |
| `S/P` disagrees with expected field | Calibrate in a homogeneous all-air reference region; `rdir1` and incidence plane control the mapping. |

## Materials and studies

| Symptom | Likely cause/fix |
| --- | --- |
| R>1, A<0, negative lossy-domain Qh | Wrong loss sign or normalization; audit formulation-specific phasor convention and physical flux. |
| Material does not change electrostatics | Default FreeSpace feature ignores material permittivity; add ChargeConservation reading `from_mat`. |
| Dispersion frozen in sweep | Study changes frequency but not global `wl`; stage points or use an active parametric sweep. |
| Wavelength study cannot be created | Use `Wavelength` for Wave Optics, not a generic frequency-domain step. |
| Parametric list ignored | Use `plistarr`, `pname`, `punit`, and activate the sweep. |

## Results and validation

| Symptom | Likely cause/fix |
| --- | --- |
| Only last outer point is visible | Use staged one-point solves or `EvalGlobal.computeResult()`. |
| Internal absorption agrees but exceeds one | Shared normalization is internally consistent but not physical closure; evaluate declared flux planes. |
| Fine mesh changes amplitude at old peak | Re-find and bracket each mesh's own peak before comparing. |
| High Q depends on scan window | Baseline/linewidth definition is inconsistent or peak is unbracketed; persist crossings and fit residuals. |
| Headless image export creates no file | Evaluate arrays, interpolate a declared slice, and render outside COMSOL. |

## Runtime and deployment

| Symptom | Likely cause/fix |
| --- | --- |
| Sharing violation on state/lease | Use bounded retry with exact-byte revalidation; preserve foreign locks and fail closed. |
| Resume creates duplicate points | Deduplicate exact full configuration identities and load durable rows into discovery state. |
| Resource metrics are missing | Refuse when the explicit policy requires them; never fabricate telemetry. |
| Source edits do not appear live | Force non-editable reinstall, restart the MCP/CLI host, and compare deployment hashes. |
| Same version appears unchanged | Version is insufficient; compare catalog/schema/profile/build identity. |
