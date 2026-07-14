# Clientapi core

## Contents

- Standalone object model
- Java collection and overload traps
- Geometry probing and finalization
- Electrostatics
- Heat transfer
- Study, mesh, results, and saving

## Standalone object model

MPh standalone returns `model.java` as a
`com.comsol.clientapi.impl.ModelClient`, not a direct
`com.comsol.model.Model`. Use clientapi-compatible overloads:

```python
jm = model.java                 # property, not model.java()
comp = jm.component().create("comp1", True)
geom = comp.geom().create("geom1", 3)
physics = comp.physics().create(
    "ewfd", "ElectromagneticWavesFrequencyDomain", "3"
)
feature = physics.feature().create("bc1", "FeatureType", 2)
```

- Component creation accepts `(tag, bool)`, not a direct-Model overload with an
  integer dimension.
- Physics creation uses a string spatial dimension such as `"3"`.
- Physics-feature creation uses an integer entity dimension such as `2`.
- Use the exact Wave Optics interface name. An `emw` namespace, `TE/TM` enums,
  or missing `ewfd.Eampl*` variables usually means the wrong interface was
  created.
- Run a study with `jm.study("std1").run()`. Do not expect `model.study()`.

## Java collection and overload traps

- Convert Java tag arrays explicitly: `list(obj.tags())`.
- Iterate tags and call `.get(tag)`; integer `.get(i)` is commonly unsupported.
- Use `feature().size()` instead of `len(feature())`.
- A physics feature may lack `.type()`; use known tags, labels, or exported model
  metadata rather than guessing from unavailable methods.
- Do not expect geometry-feature helpers such as `output()` or `faceBB()` on
  clientapi objects; use geometry-level probing.
- Coordinate `Box` selectors are not reliable across all clientapi objects. Probe
  geometry and pass explicit entity IDs when a selector cannot be proved.
- Use exact capitalization: `getSDim()`, `getNBoundaries()`, `getNDomains()`,
  `getNumElem()`, and `getNumVertex()`.
- Inspect overloads with Java reflection when uncertain:

```python
for method in obj.getClass().getMethods():
    if str(method.getName()) == "create":
        print(method)
```

- COMSOL expressions require valid grouping. For example, use `(1[V])^2`, not
  `1[V]^2`.

## Geometry probing

Never assume entity numbers from construction order alone when selections affect
physics or validation.

- `selection().entities()` returns an integer array. Convert it to a Python list.
- `geom.getUpDown()` takes no argument and returns two boundary-indexed rows:
  upper and lower adjacent domain IDs. Assembly configurations can report zeros;
  retain that ambiguity instead of inventing adjacency.
- Probe a face with `faceParamRange(boundary)` and use its parameter-range
  midpoint. A fixed `(0.5, 0.5)` may lie outside a patch face's range.
- `faceX(boundary, points)` and `faceNormal(boundary, points)` return nested
  arrays. Classify with both center and normal.
- `getBoundingBox()` returns
  `[xmin, xmax, ymin, ymax, zmin, zmax]`.
- Do not classify cell-periodic faces using normals alone: interior patch faces
  can share the same normal. Require proximity to the cell bounding planes and
  translation-congruent geometry.

## Final union and assembly feature

The finalization tag `fin` is reserved and appears automatically after geometry
construction. Do not create it manually.

```python
fin = geom.feature().get("fin")
fin.set("action", "assembly")
fin.set("createpairs", False)
fin.set("imprint", True)
geom.run()
```

- `imprint=True` can preserve intended interfaces without creating unnecessary
  pair splits.
- After a topology-changing rebuild, re-probe all domains, boundaries, pairs,
  and auto-managed selections. Do not rely on old numeric IDs even if COMSOL
  selection storage appears to update.
- Compound-feature child selections such as periodic ports may be locked. Repair
  the parent/topology and rebuild rather than forcing a locked child selection.

## Electrostatics

Recent COMSOL versions may create a default `FreeSpace` domain feature that uses
vacuum and does not consume material relative permittivity. Add an explicit
charge-conservation domain feature and a material:

```python
es = comp.physics().create("es", "Electrostatics", "3")
cc = es.feature().create("cc1", "ChargeConservation", 3)
cc.selection().set([domain_id])
cc.set("materialType", "from_mat")

mat = comp.material().create("mat1", "Common")
mat.propertyGroup("def").set("relpermittivity", "2.1")
mat.selection().set([domain_id])
```

- Probe top and bottom boundaries from centers/normals.
- Prefer `ElectricPotential` with `V0` when a `Terminal` produces an unexpected
  voltage normalization.
- COMSOL does not guarantee a mesh sequence exists. Create and build one
  explicitly before solving.
- A capacitor sanity expression can use
  `2*es.intWe/(1[V])^2`; compare against the analytical capacitance before using
  the model as a regression fixture.
- Useful electrostatics quantities can include `es.intWe`, `es.C11`, `es.normE`,
  and `es.normD`; a bare potential variable may work where `es.V` is undefined.

A portable analytical regression follows this sequence: create a 3D component
and block, build geometry, add electrostatics plus ChargeConservation/material,
probe and assign Ground/ElectricPotential to opposite faces, create/build a
FreeTet mesh, solve a Stationary study, and compare the evaluated capacitance
against `eps0*eps_r*area/separation`.

## Heat transfer

- The fixed-temperature boundary feature is commonly `TemperatureBoundary`.
- Domain feature properties (`k`, `rho`, `Cp`) differ from Common material keys
  (`thermalconductivity`, `density`, `heatcapacity`). Prefer material assignment
  with domain features reading `from_mat`.
- The time-dependent study step type is `Transient`, not `TimeDependent`.
- Evaluate transient results against an explicit dataset and inner solution,
  such as the last time index. Do not assume the default dataset is the intended
  solution.

## Study, mesh, results, and saving

- Use full study type names: `Stationary`, `Transient`, `FrequencyDomain`,
  `Eigenfrequency`, `Perturbation`, and for Wave Optics wavelength studies,
  `Wavelength`.
- `Wavelength` step properties use `plist` and `punit`. A `Parametric` feature
  uses `pname`, `plistarr`, and `punit`; activate it explicitly.
- A multi-point outer sweep can be difficult to index reliably through MPh.
  Prefer staged one-point solves or `EvalGlobal.computeResult()` when all outer
  solutions are required.
- When `model.evaluate()` accepts an expression list, preserve column order and
  use explicit inner indices; do not assume a broken scalar outer-solution
  overload will select the requested point.
- Save Unicode destinations through the Java clientapi with an absolute path:
  `jm.save(str(path.resolve()))`.
- When a feature/property name is uncertain, export a trusted model as Java
  through the available model-save tool and copy the generated API names.
- A loaded `.mph` can be file-locked. Use unique derived/checkpoint paths and
  never overwrite the immutable source.
- One JVM generally permits one standalone client. After a session-reset trap or
  a completed standalone client, start a fresh Python/MCP host rather than
  constructing another client in the same interpreter.
- Clear owned models before process exit. A standalone client's remote port being
  `None` does not imply that `disconnect()` is appropriate.
- COMSOL 6.4 can expose cuDSS through a solver property such as
  `linsolver="cudss"`; benchmark it for the exact model and GPU rather than
  assuming acceleration.
