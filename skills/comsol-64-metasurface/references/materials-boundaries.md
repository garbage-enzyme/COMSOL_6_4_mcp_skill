# Materials and boundary formulations

## Contents

- Loss-sign validation
- Dispersive wavelength control
- Layered conductive boundaries
- Geometry partitioning
- PML and manual Floquet
- Material-parameter uncertainty

## Loss-sign validation

COMSOL phasor conventions and boundary formulations can require different signs.
Never copy a Drude or complex-index sign from another model without a passivity
gate.

For each lossy domain record:

- the exact complex material expression;
- requested and evaluated wavelength/frequency;
- integrated `Qh` with sign;
- raw R/T/A and physical flux closure.

A passive model must not produce material gain, negative dissipated power beyond
numerical noise, or R/T/A outside justified bounds. If reversing the imaginary
sign fixes all passivity evidence, document the formulation-specific convention;
do not generalize it to other physics interfaces or boundary types.

For example, one volumetric `ewfd` convention can be passive with
`1-wp^2/(omega*(omega-i*gamma))` and `(n-i*k)^2`, while a calibrated layered
boundary can require the opposite imaginary sign. Treat these as
formulation-specific examples and let `Qh` plus physical flux decide.

## Dispersive wavelength control

If a material expression references a global `wl`, a wavelength-only study can
leave that parameter frozen while the solver frequency changes. Prefer a staged
one-point solve or combine a wavelength step with an active parametric sweep:

```python
wl_step.set("plist", "wl")
wl_step.set("punit", "m")
sweep.set("pname", JArray(JString)(["wl"]))
sweep.set("punit", JArray(JString)(["m"]))
sweep.set("plistarr", JArray(JString)([wavelength_list]))
sweep.set("sweeptype", "sparse")
sweep.active(True)
```

Use `c_const/wl` inside dispersive expressions. Persist both evaluated `wl` and
`c_const/ewfd.freq` for every row. A systematic mismatch is a classification
failure, not rounding noise.
Direct use of a changing `ewfd.freq` inside a layered impedance expression can
also produce an impedance singularity in a multi-wavelength solve; prefer the
explicit `wl` control and staged one-point validation.

## Layered conductive boundaries

A periodic port requires a homogeneous adjacent domain. A thin metal volume near
the port can violate that assumption. Replace suitable thin conductors with a
layered transition/impedance boundary so the port-adjacent volume remains
homogeneous.

The reliable hierarchy is:

1. A global Common material containing the conductor properties.
2. A global LayeredMaterial defining layer name, thickness, and material link.
3. A component LayeredMaterialLink restricted to the intended boundary.
4. A `LayeredTransitionBoundaryCondition` on the same boundary.

`SingleLayerMaterial` can replace a multi-layer definition for one layer, but it
still requires the component link and explicit boundary conductivity/
permeability validation. A component `LayeredMaterialLink` normally provides its
`shell` property group automatically; do not assume creating an arbitrary Shell
property group produces the same clientapi object.

Common materials normally select domains. Use a layered/single-layer material
link for boundary selections rather than forcing a Common material onto a face.

```python
bulk = jm.material().create("metal_bulk", "Common")
bulk.propertyGroup("def").set("relpermittivity", drude_expression)
bulk.propertyGroup("def").set("sigmabnd", "0")
bulk.propertyGroup("def").set("murbnd", "1")

layers = jm.material().create("metal_layers", "LayeredMaterial")
layers.set("layername", "metal")
layers.set("thickness", thickness_expression)
layers.set("link", "metal_bulk")

link = comp.material().create("metal_link", "LayeredMaterialLink")
link.set("link", "metal_layers")
link.selection().set([boundary_id])

bc = ewfd.feature().create("metal_transition", "LayeredTransitionBoundaryCondition", 2)
bc.selection().set([boundary_id])
bc.set("DisplacementFieldModel", "RelativePermittivity")
bc.set("sigmabnd_mat", "userdef")
bc.set("sigmabnd", "0")
bc.set("murbnd_mat", "userdef")
bc.set("murbnd", "1")
bc.set("lth", thickness_expression)
```

Do not assume `from_mat` resolves boundary conductivity/permeability correctly;
verify a known thin-film baseline first.

## Geometry partitioning

A global layered material cannot generally use coordinate-dependent thickness.
An embedded WorkPlane face may be a construction object rather than a physical
boundary. For a patterned conductive patch:

- partition the real interface with solid geometry;
- preserve the intended interface through union/assembly imprinting;
- apply the layered boundary only to the patch footprint;
- classify the footprint by adjacent domains and coordinates;
- re-probe outer periodic faces so patch sides are not mistaken for cell sides.

## PML and manual Floquet

A PML is a component coordinate system, not an `ewfd` domain feature:

```python
pml = comp.coordSystem().create("pml1", "PML")
pml.selection().set(pml_domains)
pml.set("ScalingType", "Cartesian")
```

`PeriodicStructure` has locked child selections that can move to PML outer faces
or expand across PML side partitions. If this architecture fails, use a manual
scattered-field formulation with exact 6.4 feature/property names:

`BackgroundField` is a physics property group, not a child feature.

```python
ewfd.prop("BackgroundField").set("SolveFor", "scatteredField")
ewfd.prop("BackgroundField").set("Eb", ["0", "exp(j*ewfd.k0*z)", "0"])
scattering = ewfd.feature().create("sctr1", "Scattering", 2)
floquet = ewfd.feature().create("fpc1", "PeriodicCondition", 2)
floquet.set("PeriodicType", "Floquet")
floquet.set("kFloquet", ["0", "0", "0"])
cross_section = ewfd.feature().create("csc1", "CrossSectionCalculation", 3)
```

Mesh every periodic source segment and copy to every matching destination
segment before `FreeTet`.

The scattering feature type is `Scattering`, not
`ScatteringBoundaryCondition`. Floquet wavevector input is the vector
`kFloquet`, not separate `kFloquetx/y/z` properties.

Agreement between volume loss and cross-section absorption is only an internal
normalization check. Require independent top/bottom physical flux for emissivity
or energy-closure claims.

## Material uncertainty

Distinguish optical mobility from Hall mobility and damping uncertainty from
resonance-position uncertainty. A missing damping parameter can change linewidth
and peak height without explaining a large center shift. Before tuning material
parameters, audit geometry, polarization, wavelength synchronization, boundary
formulation, competing branches, and mesh convergence.
