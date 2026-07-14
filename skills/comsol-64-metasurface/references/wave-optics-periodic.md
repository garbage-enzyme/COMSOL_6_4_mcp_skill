# Periodic Wave Optics

## Contents

- PeriodicStructure setup
- Reference direction and polarization
- Incidence angles
- Periodic mesh
- Oblique primitive cells
- Read-only and clone-only audits

## PeriodicStructure setup

Create the real Wave Optics interface and the compound periodic feature:

```python
ewfd = comp.physics().create(
    "ewfd", "ElectromagneticWavesFrequencyDomain", "3"
)
ps = ewfd.feature().create("ps1", "PeriodicStructure", 3)
ps.selection("excitedPortSelection").set([top_boundary])
```

`PeriodicStructure` creates Floquet-condition and periodic-port children plus a
reference-direction feature. Use `ewfd.Rtotal`, `ewfd.Ttotal`, and
`ewfd.Atotal`; do not assume an `S11` variable exists.

The base interface commonly contains wave-equation, PEC, initial-value, and
continuity features. Discover their live tags instead of assuming a fixed
sequence. Periodic-port selections created by the compound feature can be locked
and auto-managed; repair topology/parent settings rather than forcing a child
selection.

The domains adjacent to periodic ports must be homogeneous and isotropic. Put
ports in homogeneous media and move thin conductive detail to an appropriate
boundary formulation when necessary.

## Reference direction and polarization

A fresh 3D periodic feature can have an empty reference-direction selection.
Select a top-port edge parallel to the intended first lattice direction:

```python
rdir = ps.feature().get("rdir1")
rdir.selection().set([edge_id])
wee = ps.feature().get("wee1")
wee.set("DisplacementFieldModel", "RelativePermittivity")
wee.set("mur_mat", "userdef")
wee.set("mur", "1")
wee.set("sigma_mat", "userdef")
wee.set("sigma", "0")
ps.runCommand("addDiffractionOrders")
```

Probe edge parameter ranges before choosing endpoints or midpoints. The physical
meaning of `LinearPol=S/P` depends on `rdir1`, incidence plane, and angle. Record
the requested physical target separately; keep it `label_only` until field
evidence establishes the actual direction.

Treat `S`, `P`, `Mixed`, and circular handedness enums as COMSOL labels. Discover
the exact enum accepted by the installed build, and verify the physical field;
`rhcp/lhcp` names are not self-validating across observation conventions.

## Incidence angles

Periodic Structure uses `alpha1_inc` for elevation and `alpha2_inc` for azimuth.
They are angles, not lattice Floquet phases.
Do not search for legacy-looking `Theta0` or `Phi0` properties on the port.

```python
jm.param().set("theta", "10[deg]")
jm.param().set("phi", "0[deg]")
for node in (
    ps,
    ps.feature().get("pport1"),
    ps.feature().get("pport2"),
):
    node.set("alpha1_inc", "theta")
    node.set("alpha2_inc", "phi")
ps.set("Polarization", "LinearPol")
ps.set("LinearPol", "S")
```

When semantics are uncertain, inspect an official Application Library `.mph`
read-only. Modern `.mph` files are ZIP containers; property descriptions and
stored set actions can appear in `smodel.json` or `dmodel.xml`. Never modify or
unpack the source in place.

For every angle point, persist requested and independently evaluated parent and
port angles. Label script-derived `kx/ky` as derived. Report COMSOL wavevector
variables as unavailable unless they were independently evaluated.

An all-air model with negligible reflection proves port and periodic-mesh
consistency. It does not prove angle mapping. Show that a physical structure
responds to signed elevation and azimuth changes before accepting an angle
calibration.

## Periodic mesh

Corresponding periodic faces require translation-congruent geometry and matching
meshes. Build in this order:

1. `FreeTri` on each source-face group.
2. `CopyFace` from source to its destination group.
3. Repeat for the second lattice direction.
4. `FreeTet` on volumes after all copy operations.

```python
src_tri = mesh.feature().create("tri_a1", "FreeTri")
src_tri.selection().set(a1_source)
copy = mesh.feature().create("copy_a1", "CopyFace")
copy.selection("source").set(a1_source)
copy.selection("destination").set(a1_destination)
tet = mesh.feature().create("tet1", "FreeTet")
mesh.run()
```

- Reuse the default `size` node; creating another feature with tag `size` fails.
- `CopyFace` source/destination selections normally inherit the geometry context.
- Do not add `.geom(...)` to CopyFace selections unless the installed API
  requires it and the exact overload was verified.
- Equal group counts are necessary but not sufficient. Compare face partitioning,
  centers, normals, inferred translation, and the mesh feature order.
- A read-only recipe audit cannot prove node equality. Report
  `geometry_consistent`, `mesh_recipe_present`, `built_mesh_observed`, and
  `compatibility_unproven` separately.
- If needed, build only an ephemeral clone, record native mesh counts/error, then
  delete the clone and verify the source hash.
- FormUnion or FormAssembly does not make a FreeTet-only periodic mesh compatible;
  corresponding face partitions and copied source meshes are still required.

## Oblique primitive cells

Do not replace an oblique primitive cell with a rectangular box when material
partitions shift across a periodic pair. Floquet phase changes field phase, not
geometry mapping.

For lattice vectors `a1=(a,0)` and `a2=(delta,b)`, a centered parallelogram can
use vertices:

```python
cell_x = [-delta/2, a-delta/2, a+delta/2, delta/2]
cell_y = [-b/2, -b/2, b/2, b/2]
```

- Extrude full layers from the same footprint.
- Build internal shifted features so the lower partition translates exactly to
  the upper partition under `a2`.
- Fill the surrounding volume; do not leave unintended voids.
- Classify `a1` faces from the slanted boundary equations and `a2` faces from
  the top/bottom coordinate planes.
- Prove source/destination cardinality and translation before meshing.

## Typed incidence mutation

For MCP setters, require a provenance-tracked derived model and an exact
pre-state hash. Preflight exactly one `PeriodicStructure`, exactly two periodic
ports, and a nonempty reference direction. Snapshot parent and both children;
serialize mutation; write all required nodes; read them back exactly. On any
failure, roll back every node. If restoration cannot be proved, mark the derived
model dirty and refuse it for validated work.
