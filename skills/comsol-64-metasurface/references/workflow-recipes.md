# Metasurface workflow recipes

## Contents

- Continuous-film and patterned MIM
- Local refinement
- Thin-cell gratings and polarization
- Parameter sweeps
- Nanopillar/supercell geometry
- Field export
- Independent RCWA cautions

## Continuous-film and patterned MIM

Establish a continuous-film baseline before adding a patch:

1. Load an immutable template with verified materials, boundary formulation,
   mesh, and wavelength controls.
2. Resize existing geometry features only when topology remains unchanged.
3. Rebuild geometry, re-probe domains/boundaries, and rebuild mesh.
4. Solve staged wavelengths and compare reflection against an analytical or
   trusted thin-film baseline.
5. Add a patterned patch with a real geometry partition; update the layered
   boundary only on the footprint.
6. Re-probe periodic faces using centers plus normals; patch sides must not enter
   cell-side groups.

Use an MIM helper only when it exposes the exact source identity, geometry
parameters, selected domains/boundaries, mesh recipe, wavelength controls, raw
R/T/A, and cleanup evidence. Treat emissivity as `1-R` only when transmission is
physically zero or independently measured.

On servers that expose the legacy helpers, `geometry_probe_domains`,
`mim_patch_build`, and `mim_evaluate_spectral` can accelerate this workflow.
Treat live discovery and returned provenance as authoritative; do not depend on
them on a server/profile where they are absent.

## Local refinement

Refine resonant, conductive, or high-index domains first and keep surrounding
air coarse:

```python
global_size = mesh.feature().get("size")
global_size.set("custom", "on")
global_size.set("hmax", "0.16*wl")

local = mesh.feature().create("local_size", "Size")
local.set("custom", "on")
local.set("hmax", "0.06*wl")
local.set("hmaxactive", "on")
local.set("hmin", "0.012*wl")
local.set("hminactive", "on")
local.selection().geom("geom1", 3)
local.selection().set(resonant_domains)
```

The default size node and a custom `Size` feature use different activation
properties. Run a mesh-only resource gate before a finer level. Re-find each
mesh's own resonance before comparing.

## Thin-cell gratings and polarization

A 2D x-z grating cannot generally represent an out-of-plane electric
polarization with the same physics interface. Use a 3D thin cell for dual polarization:

- keep the invariant dimension thin enough for a small number of mesh layers;
- select `rdir1` along the intended grating/lattice direction;
- use `FreeTri -> CopyFace -> FreeTet` on all periodic pairs;
- solve both `S` and `P` and verify their physical field directions in reference
  air.

For mode diagnostics, TM-like surface modes usually emphasize in-plane/normal
electric components, while TE-like guided modes emphasize the ridge-parallel
component. Treat this as a hypothesis until field evidence confirms it.

## Parameter sweeps

For topology-preserving block changes:

```python
for tag, dimensions in updates.items():
    feature = geom.feature().get(tag)
    feature.set("size", dimensions["size"])
    feature.set("pos", dimensions["pos"])
geom.run()
mesh.run()
```

Re-probe selections after every geometry run. For each parameter:

Read existing feature tags and properties from the live model (for example with
`feature.tags()` and `getString("size")`) rather than relying on user-facing
labels or construction-order guesses.

1. coarse scan a broad wavelength range;
2. find all interior local maxima;
3. fine scan around candidates;
4. fit only fully bracketed peaks;
5. record every branch and validation state.

Scale scan ranges from physical expectations, but keep enough width to reveal
side-mode switching. A jump to another branch can be physical; compare field
profiles before calling it numerical.

For long scans, use resumable per-point rows and separate discovery/refinement
stages. Never restart a driver whose resume logic skips durable rows without
loading them into the peak-search state.

## Nanopillar and supercell geometry

For equivalent rectangular supercells of nonrectangular lattices:

- derive pillar centers from the primitive vectors, not a drawing alone;
- distinguish same-column spacing from half-row offsets;
- ensure translated edge partitions match across both periodic directions;
- calibrate `S/P` after choosing the reference direction;
- compare a fresh primitive/equivalent cell with matched per-domain mesh density,
  not merely equal total element count.

A common rectangular representation of a hexagonal row pattern uses
`Px=a`, `Py=a*sqrt(3)`, supercell height `2*Py`, half-row offset `Py/2`, and a
quarter-row origin shift. Same-column separation is `Py`, not `Py/2`; derive the
actual centers and translations explicitly before building geometry.

Do not attribute angular disagreement to the cell representation before checking
Bloch phase, azimuth, band folding, branch identity, and actual polarization.

## Field export

When headless COMSOL image export is unreliable, evaluate arrays directly:

```python
values, ex, ey, ez, x, y, z = model.evaluate([
    "ewfd.normE", "abs(ewfd.Ex)", "abs(ewfd.Ey)", "abs(ewfd.Ez)",
    "x", "y", "z",
])
```

Filter a declared slice by coordinate tolerance, interpolate to a regular grid,
record coverage, and render with a normal plotting library. Use shared grids and
limits for on/off comparisons.
`scipy.griddata` plus `matplotlib.pcolormesh` is a practical headless path.

## Independent RCWA cautions

RCWA layer profiles run from the incident medium toward the substrate. Verify
the order explicitly. Some Fourier implementations fail for highly lossy metals
at oblique incidence and can return trivial `R=0, T=0, A=1`. Treat such output as
numerical failure, not perfect absorption. Use a solver with appropriate
factorization or a validated COMSOL port model for the angular comparison.
