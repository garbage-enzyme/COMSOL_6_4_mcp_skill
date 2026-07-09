---
name: comsol-64-operations
description: COMSOL Multiphysics 6.4+ with MPh 1.3.1 standalone (clientapi) operations guide. Use when driving COMSOL 6.4+ via the comsol MCP server (mph.Client standalone) or writing/fixing code in the COMSOL_Multiphysics_MCP src/tools/ directory. Covers clientapi vs direct-Model API differences, geometry boundary probing (getUpDown/faceX/faceNormal), fin Form Assembly (imprint=True, createpairs=False), Electrostatics ChargeConservation trap, Heat Transfer transient pitfalls (TemperatureBoundary / Solid k,rho,Cp / Transient tlist), Wave Optics ewfd periodic metasurface setup (PeriodicStructure/Layered Impedance BC/Drude/LayeredMaterial+LML 4-level material hierarchy/wl parameter sweep trap), 1D hybrid metagrating Au/aSi MaGMR-SPP thermal emitters, Block boundary numbering, mesh/study pitfalls, MIM paper baseline verification recipe.
---

# COMSOL 6.4+ Operations Guide (MPh standalone / clientapi)

How to drive COMSOL 6.4+ via the comsol MCP server, and clientapi pitfalls you must know when writing/fixing code in `COMSOL_Multiphysics_MCP/src/tools/`. All findings are verified by testing (ParallelPlateCapacitor C=1.8593794420 pF = theoretical value, error 7e-10 pF; MIM baseline R matches theory across all wavelengths).

## Environment Quick Reference
- MCP repo (clientapi-calibrated fork): `github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_4_Calibrated`. Start from repo root with target Python env: `python -m src.server`.
- **COMSOL 6.4** ships with Java 21; MPh 1.3.1 + JPype 1.7.x work. Usually no `sitecustomize.py` JVM patch needed.
- COMSOL 6.4 adds **cuDSS** (CUDA Direct Sparse Solver). Switch: Stationary solver subnode `dDef.set('linsolver','cudss')` (default mumps). GPU acceleration typically needs large DOF and suitable hardware to be noticeable.
- Prerequisite: COMSOL must be started first. `comsol_start` first-call timeout is normal; use `comsol_status` to confirm.
- **After modifying `src/tools/` source, must restart the host agent/CLI** (MCP server is a subprocess, no hot-reload).

## Key: standalone uses clientapi, not direct Model
mph 1.3.1 `mph.Client(cores=...)` returns `model.java` as `com.comsol.clientapi.impl.ModelClient`:
- `mj = m.java` (property, **do not call**); `mj.physics().get('ewfd')` etc.
- `component().create(tag, True)` — no `(str,bool,int)` overload. Space dimension is determined by `geom().create(tag, sdim)`.
- `physics().create(tag, type, sdim)` — third parameter is **String** (e.g. `"3"`), not int! Get sdim via `comp.geom(tag).getSDim()` → `str()`.
- `feature().create(tag, type, edim)` — third parameter is **int** (boundary dimension, e.g. 2). Different from physics String sdim.
- `*.get(i)` int indexing not supported — use `tags()` to iterate: `for t in list.tags(): obj = list.get(t)`.
- `len(geom.feature())` not supported — use `geom.feature().size()`.
- `geom.getNBoundaries()`/`getNDomains()`/`getSDim()` (capital first letter); `mesh.getNumElem()`/`getNumVertex()`.
- study step type must use **full name**: `Stationary`/`Transient` (not TimeDependent!)/`FrequencyDomain`/`Eigenfrequency`/`Perturbation`. `study_create` tool does normalization mapping.
- `m.study().run()` not available — use `jm.study('std1').run()`. `m.evaluate('V')` works.
- Expressions: `1[V]^2` gives syntax error, must use `(1[V])^2`.
- `phys.tags()`/`feat.tags()` returns Java list, use `list(...)` to convert to Python list.
- `geom.feature().get(t).type()` — PhysicsFeatureClient has no `type()` attribute; use `.label()` to distinguish.

## Geometry Probing API (clientapi verified)
- **selection().entities()** returns int[] (use `list(...)` to convert to Python) → boundary/domain numbers. Warning: Common material has no selection container (reports "entity has no selection"); LayeredMaterialLink selects boundary.
- **geom.getUpDown()** takes **no arguments**, returns `int[2][n_bnd]`: row0 = up dom, row1 = down dom (in assembly mode up is all 0). `list(ud[1])` gets down dom per boundary.
- `geom.faceX(bn, JArray(JArray(JDouble))([[u, v]]))` returns nested `double[][]`, `list(arr)[0]` gets inner array, then index cx,cy,cz. `geom.faceNormal(bn, pp)` same pattern.
- `pp` recommended **(0.33, 0.33)** not (0.5, 0.5) (some patch side faces have parameter range outside [0,1]², reports "parameter out of range").
- `geom.faceParamRange(bn)` returns `[umin, umax, vmin, vmax]` → use midpoint as fallback.
- `geom.getBoundingBox()` returns `[xmin,xmax,ymin,ymax,zmin,zmax]`.
- Geometry feature client has no `output()`, `faceBB(int)` etc; use `faceX(bn, pp)` center coordinates + normal to determine face position and orientation.

## fin Node (FormUnion/FormAssembly)
- **Cannot create('fin', any)** — tag `fin` is reserved for "Form Union/Assembly", always auto-created by `geom.run()`.
- Modify fin type: `fin = geom.feature().get('fin')` + `fin.set('action', 'assembly')` + `fin.set('createpairs', False)` + `fin.set('imprint', True)` (preserves interior boundary, no pair splitting).
- Then `geom.run()` to take effect. With imprint=True, Al2O3/air interface merges into single bnd, patch side faces and air also have no pair splitting, side Floquet won't trigger pair conflicts.
- **After geometry rerun, all selections auto-update to new bnd numbers** (selectionStorage retains references) — ltr1/lml_au/pport1/pport2/fpc1/fpc2/material selections need no reset.
- **pport1/pport2 selection is not editable** ("Selection is not editable") — auto-detected by ps1, must rerun geometry to let it re-assign automatically.

## Electrostatics Modeling Notes (6.3+)

### 1. Default fsp1=FreeSpace uses vacuum, must manually add ChargeConservation
6.3+ Electrostatics default domain feature is `fsp1` (FreeSpace), **uses vacuum eps0, does not read material relpermittivity**. Must manually add ChargeConservation:
```python
ccn = p.feature().create('ccn1', 'ChargeConservation', 3)  # 3=sdim(int)
ccn.selection().set([1]); ccn.set('materialType', 'from_mat')
mat = comp.material().create('mat1', 'Common')
mat.propertyGroup('def').set('relpermittivity', '2.1'); mat.selection().set([1])
```
**MCP tool shortcut**: `physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` auto-creates everything.

### 2. Block boundary numbering is not fixed
Use `geometry_get_boundaries` (returns normal+center+bounding_box) to determine. Verified Block [0.01,0.01,0.001]: bnd3=z=0 (normal[0,0,-1]), bnd4=z=0.001 (normal[0,0,1]).

### 3. Terminal V0 unreliable, use ElectricPotential
Terminal `V0=1[V]` measured ΔV≈0.16V. Use **ElectricPotential** BC: `p.feature().create('ep1','ElectricPotential',2); ep.set('V0','1[V]')`.

### 4. Mesh: must manually create mesh sequence
COMSOL does not auto-create mesh. **MCP tool**: `mesh_sequence_create(mesh_name='mesh1', element_type='FreeTet', build=True)`.

### 5. Capacitance calculation
`C = m.evaluate('2*es.intWe/(1[V])^2', 'pF')`. Warning: `(1[V])^2` must have parentheses. `es.V` reports undefined variable — use bare `V`. `es.intWe`/`es.C11`/`es.normE`/`es.normD` work.

## ParallelPlateCapacitor Verification Recipe (C=1.8593794420 pF, error 7e-10 pF)
Theory: `C = eps0*eps_r*L²/d = 8.854e-12*2.1*0.01²/0.001 = 1.8593794407 pF`.

### MCP tool workflow
1. `model_create(name='ParallelPlateCapacitor')` → `model_create_component(3D)` → `geometry_create(3D)`
2. `geometry_add_block(size=[0.01,0.01,0.001])` → `geometry_build`
3. `physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` — auto-creates ChargeConservation + material
4. `physics_configure_boundary('Electrostatics', 'Ground', [3])` — z=0; `physics_configure_boundary('Electrostatics', 'ElectricPotential', [4], {V0:'1[V]'})` — z=0.001
5. `mesh_sequence_create(element_type='FreeTet', build=True)` — ~1651 elements
6. `study_create(study_type='Stationary')` → `study_solve`
7. `results_global_evaluate('2*es.intWe/(1[V])^2', 'pF')` — 1.8593794420

## Heat Transfer Transient Modeling Notes (Si block verified: dT=7.6923K, 6 significant digits match)

### 1. Fixed temperature BC type name is `TemperatureBoundary` (not `Temperature`)
`p.feature().create('temp1','TemperatureBoundary',2); bc.set('T0','293.15[K]')`.

### 2. Solid domain feature property names vs Material Basic property names
`Solid` feature properties: `k`/`rho`/`Cp`. Material node Basic(`def`) properties: `thermalconductivity`/`density`/`heatcapacity`. Keep feature `from_mat`, write to Material Basic.
**MCP**: `physics_set_material(physics_name='Heat Transfer in Solids', material_name='Silicon', domain_selection=[1], properties={...})`.

### 3. Transient study step type is `Transient` (not `TimeDependent`)
**MCP**: `study_create(study_type='Transient', time_list=[0,0.001,0.01,0.1], time_unit='s')` auto-sets tlist.

### 4. Transient results require inner index + explicit dataset
`results_evaluate(expression='T', dataset='Study 1//Solution 1', inner='last', unit='K')`. Warning: dataset=None may error, use `datasets_list` to find name.

## Wave Optics (ewfd) Periodic Metasurface Simulation

Reproducing Au-Al₂O₃-Au MIM metasurface thermal emitter (Chen et al. *Int. J. Thermal Sciences* 185 (2023) 108069). Paper geometry: **P=1.35µm, L=0.856µm, h=0.10µm (Au patch thickness), d=0.04µm (Al₂O₃ spacer), tsub=0.10µm (Au substrate thickness)**. Resonance targets: MP1@4.37µm (ε≈0.95), MP2@2.27µm (ε≈0.5). Mechanism: magnetic polariton (MP) in MIM trilayer — loop current between patch and substrate gives strong magnetic enhancement.

### PeriodicStructure API (verified)
- **Key**: must use `PeriodicStructure` (parent node), not `PeriodicCondition`!
- `comp.physics().create('ps1','PeriodicStructure',str(sdim))` auto-generates fpc1/fpc2(Floquet) + pport1/pport2(PeriodicPort) + rdir1.
- **Activate port excitation**: `ps1.selection("excitedPortSelection").set(top_bnd)` — critical step!
- S-parameter variables: `ewfd.Rtotal`/`ewfd.Ttotal`/`ewfd.Atotal` (not `ewfd.S11`). Study type: `Wavelength`.
- Subnode selectors auto-assigned and locked, do not manually set. `addDiffractionOrders`: `ps1.runCommand("addDiffractionOrders")`.
- Reference examples: `metasurface_beam_deflector.mph` (Rtotal=0.122 verified), `plasmonic_wire_grating.mph`, `hexagonal_grating.mph`.

### 3D PeriodicStructure rdir1 / axisy trap
- From-scratch **3D** `PeriodicStructure` may create `ps1/rdir1` with an empty selection. Symptom during solve: `comp1.ewfd.axisy` or `axisx/axisy` undefined on the periodic port boundary.
- Fix by selecting one edge on the excited port to define the first primitive vector: `ps.feature("rdir1").selection().set([edge_id])`. Pick an edge parallel to the intended lattice vector. For a 1D grating with period along `x`, use a top-port edge along `x`.
- 3D `edgeX` probing must use `edgeParamRange(edge)` first; the parameter range is not always `[0,1]`. Use midpoint/endpoint values from the returned range.
- The `PeriodicStructure` subnode has its own `wee1`. If materials are Common materials with `relpermittivity`, also set `ps.feature("wee1").set("DisplacementFieldModel", "RelativePermittivity")`, `mur_mat="userdef"`, `mur="1"`, `sigma_mat="userdef"`, `sigma="0"`.
- After setting `rdir1`, call `ps.runCommand("addDiffractionOrders")` when diffraction orders are needed.

### Long standalone sweep robustness
- Prefer staged one-point solves for fragile wavelength sweeps: set `jm.param().set("wl", value)`, run `jm.study("std1").run()`, evaluate with `EvalGlobal`, append CSV immediately. This avoids COMSOL/mph outer-sweep evaluation surprises such as only seeing the last point.
- Standalone `mph.Client` scripts may finish saving files but not exit because JVM/helper threads stay alive. After all outputs are flushed and saved, `os._exit(0)` is acceptable for long-running handoff scripts.

### Port failure root cause + solution (PDF docs p151, p179-181)
- **Root cause**: Periodic port assumes **the domain adjacent to the port is a homogeneous isotropic medium** (WaveOpticsModuleUsersGuide.pdf p151). Metal patch (Drude negative ε) in the domain violates this assumption; even if port is in air domain it fails. Minimal test: dielectric patch → R=1.08 normal; metal patch → R=0 fails.
- **Solution: Layered Impedance Boundary Condition** (p179-181):
  - Supports **Drude-Lorentz dispersion model**! Use BC to replace metal patch volume, domain only has dielectric material, port mode assumption satisfied.
  - = Layered Transition BC + Impedance BC, assumes wave propagates along normal in thin layer (suitable for good conductor).
  - Requires Layered Material (Shell property group) to define Au thin film (thickness + Drude properties).

### LayeredTransition BC working setup (4-level material hierarchy, all required)
```python
# 1. Global Common material mat_au (Drude eps + sigmabnd + murbnd in def group)
mat_au = jm.material().create('mat_au','Common')
mat_au.propertyGroup('def').set('relpermittivity', au_drude)
mat_au.propertyGroup('def').set('sigmabnd', '0')   # required
mat_au.propertyGroup('def').set('murbnd', '1')     # required

# 2. Global LayeredMaterial lm_au (layer definition)
lm = jm.material().create('lm_au','LayeredMaterial')
lm.set('layername','Au'); lm.set('thickness', str(t_au)); lm.set('link','mat_au')
lm.propertyGroup('def').set('relpermittivity', au_drude)
lm.propertyGroup('def').set('sigmabnd', '0'); lm.propertyGroup('def').set('murbnd', '1')

# 3. Component LayeredMaterialLink lml_au (restrict to boundary)
lml = comp.material().create('lml_au','LayeredMaterialLink')
lml.set('link','lm_au')
lml.selection().all(); lml.selection().clear(); lml.selection().add([bnd6])  # only on interface
sh = lml.propertyGroup('shell')  # auto-has shell group
sh.set('lth', str(t_au)); sh.set('relpermittivity', au_drude)
sh.set('sigmabnd', '0'); sh.set('murbnd', '1')

# 4. LayeredTransition BC (shelllist auto=lml_au, but sigmabnd/murbnd must be userdef)
ltr = p.feature().create('ltr1','LayeredTransitionBoundaryCondition',2)
ltr.selection().set([bnd6])
ltr.set('DisplacementFieldModel','RelativePermittivity')
ltr.set('sigmabnd_mat','userdef'); ltr.set('sigmabnd','0')   # from_mat doesn't work!
ltr.set('murbnd_mat','userdef'); ltr.set('murbnd','1')       # from_mat doesn't work!
ltr.set('lth', str(t_au))
```
- **Verification**: eps=2.1 dielectric thin film → Rtotal=0.0757 (thin film interference); Drude continuous film → R(1µm)=0.41, R(5µm)≈1.0.
- **SingleLayerMaterial** also works (`comp.material().create(tag,'SingleLayerMaterial')`), but similarly requires LML + sigmabnd/murbnd userdef.
- **Common material selection only supports domain**; LML/SingleLayerMaterial selection supports `all()+clear()+add([bnd])` to restrict to boundary.
- **propertyGroup create('shl','Shell')** returns type='def' (clientapi limitation); LML auto-has 'shell' group (contains lth/lrot/lne/relpermittivity/sigmabnd/murbnd).

### Drude wavelength sweep trap (verified)
- `ewfd.freq` in multi-wavelength Wavelength study causes impedance singularity (Jsupx cannot compute, Zs²-Zt²=0).
- **Solution**: `wl` parameter + `c_const/wl`:
  ```python
  jm.param().set('wl','5e-6[m]')
  au_drude = "1-(1.37e16)^2/((2*pi*c_const/wl)*((2*pi*c_const/wl)+i*4.1e13))"
  # Study: Wavelength step (plist='5e-6' dummy) + Parametric sweep step
  study.create('step1','Wavelength'); step.set('punit','m'); step.set('plist','5e-6')
  study.create('sweep1','Parametric'); sweep.set('pname','wl'); sweep.set('plist','1e-6 2e-6 ...')
  ```
- Continuous Au film Drude sweep: R(1µm)=0.41, R(2µm)=0.93, R(3µm)=0.997, R(4-10µm)≈1.007 (metal full-reflection baseline).

### Spatially varying lth not feasible (confirmed)
- LML Shell group lth set to `if(x>...)` expression → solve OK but R unchanged (LML reads fixed thickness from global LM).
- Global LM set to coordinate expression → error "invalid layer definition" (global material cannot use coordinates).
- **Conclusion**: patch spatial partitioning must use **geometry partition** (split bnd6 into patch + rest) so LTR only covers the interface above the patch.

### Patch geometry approaches (verified viable path)
- **WorkPlane + PartitionFaces**: bnd10 is a WorkPlane-embedded floating face in 3D, **does not participate in physics** — LayeredTransition/LayeredImpedance on bnd10 gives R=1 across all wavelengths, no effect. **Approach not viable**.
- **Block + Difference + FormAssembly (imprint=True)**: keeps patch as independent dom 3, interface merges via imprint into single interior bnd. Patch side faces and air also need no pair Continuity (auto-continuous under imprint mode). Floquet periodic pairs on outer cell sides are undisturbed. **This approach verified: geometry builds OK (3 dom, 24 bnd, no duplicate pairs)**, physics with pair Continuity pending solve verification.

### Key API notes
- `ewfd` default features: `wee1` (wave equation) / `pec1` (PEC) / `init1` (initial values) / `dcont1` (continuity).
- Material permittivity uses **single string** (e.g. `'-972+283.5*i'` or Drude expression), not `[real,imag]` array (reports "needs 3x3 matrix").
- Au Drude: `1-(1.37e16)^2/((2*pi*c_const/wl)*((2*pi*c_const/wl)+i*4.1e13))`, wp=1.37e16 rad/s, gamma=4.1e13 rad/s (*known trap*: use `c_const/wl` not `ewfd.freq`).
- Box selector fails under clientapi, must pass domain numbers directly.
- `m.evaluate([expr, 'wl'])` (list) gets multi-frequency sweep results, each row is [expr_value, wl_value]. Iterate inner indices with `inner=[i,...]` (0-indexed list).

### Complete MIM baseline recipe (verified working)
Theory: continuous Au film (no patch) → Drude full reflection, R(1µm)≈0.41, R(≥4µm)≈1.007 (numerical error).
1. **Template clone**: load from `MIM_Continuous_Drude.mph` as baseline (4-level material + Sweep mesh + Wavelength+parametric sweep all pre-built).
2. **Geometry resize**: `geom.feature().get('b_al2').set('size', jarr([P,P,d]))` + `b_air.set('size', jarr([P,P,H-d]))` + `b_air.set('pos', jarr([0,0,d]))` + `geom.run()` (fin auto-creates FormUnion, selections auto-update).
3. **Mesh rebuild**: MCP `mesh_create`.
4. **Solve**: MCP `study_solve` (MCP tool timeout is normal, use `comsol_status` + `results_inner_values`/`datasets_list` to check for inner indices to confirm completion).
5. **Evaluate**: `results_evaluate(['ewfd.Rtotal', 'wl'])` returns 10×2 array (wl 1-10µm, R column).
6. **Save**: `m.save()` or MCP `model_save`.

Output model can be saved to any local `.mph` file; when baseline is verified, R values should match continuous film.

## MIM patch MCP tools (src/tools/mim_patch.py, commit 1ba6a0a + 57801a5)
Three MIM patch workflow helper tools, end-to-end tested on MIM_paper_baseline_v1 → patch geometry + mesh auto-build success (3 dom, 17 bnd, 18837 elements).

### 1. geometry_probe_domains
- Enhanced `geometry_get_boundaries`: each bnd has up/down domain, interior flag, normal, center; additionally returns `interior_boundaries` (up!=0 and down!=0), `pairs` (comp.pair().tags()), `side_pairs` (auto-classified cell periodic edges).
- side_pairs classification: normal + **center coordinates** dual filter (using `geom.getBoundingBox()` as bbox), tol=1e-12. **Using normal only would misclassify patch side/top faces as cell boundaries** (commit 57801a5 fix) — CopyFace would then copy patch side mesh to cell boundary, breaking Floquet mesh compatibility.
- Example: 3 dom patch geometry, `x_src=[1,4]` (cell edge x=0), does not include patch side bnd10 (x=L/2).

### 2. mim_patch_build(patch_size=[L,L,h], patch_pos=[x0,y0,d])
- Builds patch geometry from 2-dom baseline (Al2O3+air): Block + Difference(keepsubtract=True) → FormUnion (imprint auto) + update LayeredTransition→patch footprint (interior bnd where up=patch_dom, down=al2_dom) + LayeredImpedance→bottom + air material→patch_dom + FreeTri+CopyFace+FreeTet mesh.
- **Parameters**: patch_size/patch_pos in meters. Paper values L=0.856µm, h=0.10µm, pos=[(P-L)/2,(P-L)/2,d=4e-8].
- **air_block_tag auto-detection**: finds Block feature with `size` z-component > 1e-7 (air block is much taller than Al2O3).
- **patch_dom = n_domains** (last added dom).
- **Known limitation**: if old `_identify_side_pairs` lacks coordinate filtering → PS port set multiple bnds reports "only single boundary allowed". Fixed in 57801a5.

### 3. mim_evaluate_spectral
- One-line evaluate `ewfd.Rtotal`/`Ttotal`/`Atotal` + `wl` parameter, returns `{wl_um, Rtotal, Ttotal, Atotal, emissivity}` list, emissivity=1-R.
- Verified patch_final: ε@4-5µm≈0.89 (paper MP1@4.37µm ε≈0.95), ε@2µm≈0.88 (paper MP2@2.27µm ε≈0.5).
- Underlying `model.evaluate(expr_list)`, wl column ×1e6 to µm. **Note: add 'wl' parameter before calling, otherwise KeyError**.

## Xu 2024 In:CdO MIM notes

### Wavelength-parametric sweep
- For `Parametric` study features use `plistarr`, not `plist`; `plist` is for the `Wavelength` step. Also call `sweep.active(True)`.
  ```python
  wl_step.set("plist", "wl")
  wl_step.set("punit", "m")
  sweep.set("pname", JArray(JString)(["wl"]))
  sweep.set("punit", JArray(JString)(["m"]))
  sweep.set("plistarr", JArray(JString)([plist_str]))
  sweep.set("sweeptype", "sparse")
  sweep.active(True)
  ```
- Read all outer sweep rows with `EvalGlobal.computeResult()[0]`. `mph.Model.evaluate(outer=...)` can fail because `outersolnum` scalar overload resolution is broken in COMSOL 6.4/mph 1.3.1.
- Wavelength-only sweeps can solve multiple points but do not update a global `wl` parameter. If Drude material expressions use `wl`, this freezes the dielectric function at the initial wavelength.

### Ref.66 / Drude checks
- Nolen et al. CdO supplemental tables distinguish Hall mobility from optical mobility. Optical mobility changes damping, linewidth, and peak height; it does not by itself guarantee a large resonance-position correction.
- SI-anchored `wp_fit + muOpt + m*` checks did not fully fix low-density In:CdO MIM peak shifts in FEM. Treat missing `muOpt` as a damping uncertainty, not an automatic peak-position root cause.

### PML trap with PeriodicStructure
- Bottom PML can be created with `coordSystem().create("pml1","PML")`, `pml.selection().set([pml_domain])`, `pml.set("d","z")`.
- Do not assume this works with `PeriodicStructure`: `ps1` creates locked periodic port subfeatures on external boundaries. The lowest port can move to the PML bottom, cannot be disabled through clientapi, and PML side faces can break Floquet copy meshes or leave port/PML variables undefined.
- Keep the finite substrate + periodic port workflow unless replacing the whole port setup.

### Mesh and mode selection
- High-density MIM spectra can be stable at moderate fine mesh, while lower densities may select long-wave side modes unless focused finer probes are run near the expected Fig.2 peak.
- In one verified In:CdO MIM case, `hmax=0.015*wl` probes moved `n=2e20` and `1.5e20` back to the expected main peaks, but `n=1e20` still favored a longer-wave peak. Do not claim full low-density agreement without field-profile or dielectric-function evidence.
- COMSOL memory sawtooth cycles during a sweep often correspond to one wavelength point: assembly/factorization raises memory, result write/free lowers it. Use as a progress hint only.

### Local mesh refinement (verified on Zhou 2025 QBIC 4-pillar supercell)
For 3D periodic structures with expensive narrow features (high Q resonances), use **selective local refinement** to keep element count low:
- Keep air domains (surrounding block) at coarse global size (e.g. `hmax=0.16*wl`)
- Apply a custom `Size` feature (`sz1`) only to metal + pillar domains
- Verified recipe (Zhou 2025 QBIC, 4-pillar supercell, 11 domains):
  ```python
  air_doms = [1, 3]        # air below + surroundings
  local_doms = [2,4,5,6,7,8,9,10,11]  # Au + SiO2 + aSi pillars

  size = mesh.feature("size")
  size.set("custom", "on"); size.set("hmax", "0.16*wl")
  size.set("hmin", "0.035*wl")

  sz1 = mesh.feature().create("sz1", "Size")
  sz1.set("custom", "on"); sz1.set("hmax", "0.06*wl")
  sz1.set("hmaxactive", "on"); sz1.set("hmin", "0.012*wl")
  sz1.set("hminactive", "on")
  sz1.selection().geom("geom1", 3)
  sz1.selection().set(jarr_i(local_doms))
  ```
- Result: 18k elements vs 64k global at same pillar resolution, ~3x faster
- **Key property difference**: default `size` node uses `hmax`/`hmin` directly (no `active` suffix); custom `Size` (`sz1`) uses `hmaxactive`/`hminactive` suffixes.
- For convergence: mesh shifts peak to longer λ (~0.75nm per step from 0.08→0.06→0.04*wl). Always re-find peak after mesh change.
- 0.04*wl on 220k elements + 392k DOF needs ~48+ GB RAM — avoid global 0.04*wl for production sweeps. Use 0.06*wl production + 0.04*wl single-point sanity check.

### Final-check addendum
- In a verified In:CdO MIM run, focused fine sweeps moved one mid-density point back to the paper peak and confirmed another point's higher FEM global peak within tolerance. Use focused fine sweeps before calling a side peak physical or spurious.
- Point-field profiles at competing wavelengths can show whether peaks belong to the same mode family. Similar field distributions imply a method/boundary-condition residual rather than a simple material-parameter error.

### Field-profile export (verified on Zhou2025 QBIC, 146k integration pts)
For 3D periodic structures, use `m.evaluate(exprs)` to get field data at all integration points:
```python
res = m.evaluate(["ewfd.normE", "abs(ewfd.Ex)", "abs(ewfd.Ey)", "abs(ewfd.Ez)", "x", "y", "z"])
# res is a list of arrays in the same order as exprs
normE = np.array(res[0]); Ex = np.array(res[1]); x = np.array(res[4])
```
Filter to 2D slices by coordinate tolerance, then `scipy.griddata` interpolate to regular grid + `matplotlib.pcolormesh`:
```python
tol = 5e-8
mask = np.abs(z - z_slice) < tol
xi = np.linspace(x[mask].min(), x[mask].max(), 200)
yi = np.linspace(y[mask].min(), y[mask].max(), 400)
XI, YI = np.meshgrid(xi, yi)
EI = griddata((x[mask], y[mask]), normE[mask], (XI, YI), method="linear")
plt.pcolormesh(XI, YI, EI, shading="auto")
```
**Findings** (Zhou2025 QBIC): Ex+Ez dominate over Ey in the TE QBIC mode (scattered field creates x-z components despite Ey-polarized excitation). Field max ~1.8e8 V/m localized at pillar region.

## Zhou 2024 1D hybrid metagrating (verified recipe)
Paper: Zhou et al., IEEE Sensors Journal 24(13), 2024. Structure: 1D Au grating + aSi cladding. TM excites leaky SPP on Au; TE excites MaGMR (guided-mode resonance) in aSi.

### Key results (all three structures verified, 3D thin-cell FEM)
| Structure | Mode | FEM | Paper | Q | Emissivity |
| --- | --- | --- | --- | --- | --- |
| I | TM SPP | 4.250 µm | 4.23 µm | 31 | 0.954 |
| I | TE MaGMR | 3.395 µm | 3.31 µm | 104 | 0.944 |
| II | TM SPP | 5.295 µm | 5.27 µm | 46 | 0.993 |
| II | TE MaGMR | 4.360 µm | 4.28 µm | 125 | 0.982 |
| III | TM SPP | 5.763 µm | 5.71 µm | 47 | 0.994 |
| III | TE MaGMR | 4.715 µm | 4.61 µm | 133 | 0.991 |

All 6 peaks match within 0.11 µm. Materials: Au Drude (wp=1.37e16, gamma=1e14), aSi lossless eps=11.9, Air eps=1. Mesh: 9k–14k elements, ~2s/solve per wl.

### Structure parameters
| Structure | P | W | H2 | H1 | TE peak | TM peak |
| --- | --- | --- | --- | --- | --- | --- |
| I | 1.40 µm | 0.70 µm | 0.10 µm | 0.44 µm | 3.31 µm | 4.23 µm |
| II | 1.75 µm | 0.80 µm | 0.10 µm | 0.61 µm | 4.28 µm | 5.27 µm |
| III | 2.00 µm | 0.90 µm | 0.10 µm | 0.61 µm | 4.61 µm | 5.71 µm |

### Geometry template reuse pattern (verified for all 3 structures)
Loading an existing .mph (built via MCP geometry tools) and modifying block `size`/`pos` is more reliable than building from scratch. The `FormUnion` ("fin") feature auto-exists; just set `action="union"` and `geom.run()`.

```python
geom = comp.geom("geom1")
for tag, (sx, sy, sz, px, py, pz) in new_dims.items():
    f = geom.feature(tag)          # e.g. "blk2".."blk7"
    f.set("size", JArray(JDouble)([sx, sy, sz]))
    f.set("pos", JArray(JDouble)([px, py, pz]))
geom.feature("fin").set("action", "union")
geom.run()
```

- Block tags are auto-generated (blk2..blk7 for 6 blocks), NOT named by the user.
- Read existing tags: `list(geom.feature().tags())`; read size/pos: `f.getString("size")` returns comma-separated string.
- **Domain mapping must be hardcoded** — auto-detection by boundary center probing fails for thin layers (100nm Au between 1µm air and 0.1µm ridge). Same blk order → same domain numbering: dom 1=air_below, 2=au_mirror, 3=au_ridge, 4=aSi_cover, 5=air_top, 6=aSi_groove.
- **rdir1 edge number is topology-invariant** — same block layout → same edge numbering. Hardcode after first probe (edge 17 for this 6-block layout).
- Y-dimension for thin cell: 0.4 µm works for λ 3–6 µm (1–2 mesh elements in y).

### Polarization mapping
- TE = E field parallel to ridge (y-direction). Dominant Ey, MaGMR mode in aSi layer.
- TM = E field in incidence plane (xz). Dominant Ex/Ez, SPP on Au surface.
- Always verify with field profiles before trusting polarization labels.

## 2D template trap for grating metagratings
- `plasmonic_wire_grating.mph` (2D x-z template) can only do `LinearPol='P'` (TM). Cannot represent TE/Ey MaGMR (out-of-plane E needs different physics interface).
- 2D TM response locks to **bulk SPP** at `λ ≈ P * sqrt(eff_eps)`, with effective index near the grating dielectric constant's sqrt. H1, Q, and mesh refinement cannot move the peak away from this asymptote.
- **Lesson**: 1D dual-polarization metagratings require a **3D thin-cell unit cell** with `PeriodicStructure` and `LinearPol = S/P` switching.

## 3D thin-cell PeriodicStructure for 1D gratings

### rdir1 edge selection (required for 3D)
- From-scratch 3D `PeriodicStructure` may leave `rdir1` selection empty. Symptom: `comp1.ewfd.axisy` undefined at solve.
- Fix: select one edge on the top (excited) port, parallel to the periodicity direction. Then set `wee1` to `RelativePermittivity` and call `addDiffractionOrders`.
```python
ps.feature("rdir1").selection().set([edge_on_top_port])
wee = ps.feature("wee1")
wee.set("DisplacementFieldModel", "RelativePermittivity")
wee.set("mur_mat", "userdef"); wee.set("mur", "1")
wee.set("sigma_mat", "userdef"); wee.set("sigma", "0")
ps.runCommand("addDiffractionOrders")
```

### Thin-cell geometry setup
- Build unit cell in 3D with y-extent thin enough for one mesh layer (e.g. 400 nm for λ~3-6µm, mesh 1-2 elements in y).
- Use `FreeTri` on source faces → `CopyFace` (src→dst) → `FreeTet` on volumes.
- `CopyFace` order: x_src→x_dst, then y_src→y_dst. The mesher needs aligned periodic face meshes for Floquet compatibility.

### Staged sweep for field-profile-resistant modes
- `Parametric` sweep + `Wavelength` step. Set `plist="wl"` on the wavelength step, `pname="wl"` on the parametric sweep, `plistarr` with comma-separated wavelengths, `punit="m"`.
- For dual-polarization: two complete solve-evaluate cycles. After solving P, re-set `ps.set("LinearPol","S")`, solve again.

## Staged sweep resilience pattern (COMSOL standalone scripts)
- **One wavelength per solve**: `jm.param().set("wl", value)` → `jm.study("std1").run()` → evaluate → append CSV row. Avoids losing all results to a long sweep timeout.
- **Resume**: script reads existing CSV first, skips already-computed wavelengths, runs only pending points.
- **Tee logging**: stdout/stderr simultaneously written to `.log` file for post-mortem after interruption.
- **Hard exit**: `os._exit(0)` at end to force JVM shutdown (standalone mph.Client leaves non-daemon threads).

## Field profile export workaround (client-server mode)
- `Image3D` export nodes in client-server COMSOL may not produce PNG files (the server has no local rendering context).
- **Workaround**: evaluate field expressions at mesh nodes, filter to slice plane, interpolate to regular grid, plot with matplotlib.
```python
data = model.evaluate(["ewfd.normE", "x", "y", "z"])
vals, x, y, z = [np.array(d) for d in data]
mask = np.abs(y) < 1e-10  # y≈0 slice
xs, zs, vs = x[mask], z[mask], vals[mask]
from scipy.interpolate import griddata
xi = np.linspace(xs.min(), xs.max(), NX)
zi = np.linspace(zs.min(), zs.max(), NZ)
XI, ZI = np.meshgrid(xi, zi)
grid = griddata((xs, zs), vs, (XI, ZI), method="linear")
# Then matplotlib pcolormesh + savefig
```
- Verification pattern for SPP vs MaGMR:
  - TM SPP: `|Ex|` and `|Ez|` dominate (max ~1e8), `|Ey|` near zero (max ~1e6).
  - TE MaGMR: `|Ey|` dominates (max ~1e8), `|Ex|` and `|Ez|` near zero.

## Parameter sweep workflow (geometry perturbation, verified on Zhou 2024 Fig.2)

### In-place geometry modification pattern
For parameter sweeps that only change block dimensions (not topology), modify the
existing geometry features in-place and re-run, then re-mesh. No need to reload
the model or recreate physics/study.
```python
geom = comp.geom("geom1")
for tag, (sx, sy, sz, px, py, pz) in new_dims.items():
    f = geom.feature(tag)
    f.set("size", JArray(JDouble)([sx, sy, sz]))
    f.set("pos", JArray(JDouble)([px, py, pz]))
geom.feature("fin").set("action", "union")
geom.run()
# Re-classify boundaries (top/bottom numbers are stable but re-probe to be safe)
bnd_info = classify_bnds(geom)
# Rebuild mesh on modified geometry
comp.mesh("mesh1").run()
```

### Coarse-then-fine peak finding per parameter point
For each parameter value:
1. **Coarse sweep**: wide wavelength range, 0.05 µm step, both polarizations.
2. **Find coarse peak**: `np.argmax(vals)` on the 1-R curve.
3. **Fine sweep**: ±0.10–0.15 µm around coarse peak, 0.005 µm step.
4. **Lorentzian fit** on fine data for Q = λ_peak / FWHM.
5. **Record**: (param, value, TM_peak, TM_Q, TM_emis, TE_peak, TE_Q, TE_emis, note).

### Scan range auto-scaling by period
For grating structures where resonance wavelength scales with period P:
```python
P_um = P * 1e6
wl_min = max(2.0, P_um * 1.5)
wl_max = P_um * 3.5
```
This avoids wasting time scanning irrelevant wavelengths for small/large P values.

### Mode switching at small period
For small P values (e.g. P=1.40–1.60 µm in Zhou 2024), the TE MaGMR peak may
**jump to a short-wavelength side mode** (~2.4–2.5 µm) instead of the expected
~3.3 µm peak. This is a real mode switching (different MaGMR order), not a
simulation error. The paper's Fig.2(e) shows the same behavior. Always check
field profiles if peak assignment is ambiguous.

### Timeout resilience for long multi-point sweeps
- 23 parameter points × (coarse ~80 + fine ~40) × 2 pol × ~2s ≈ 3 hours.
- Split into two scripts: main sweep + resume script that reads existing CSV
  and runs only missing points. Copy base-case results from existing rows.
- Each point writes to summary CSV immediately (append mode).
- **W sweep**: paper says W fine-tunes SPP via filling factor but TE/MaGMR
  is nearly independent of W. Confirmed: TM shifts 5.17→5.43 µm across
  W=0.6–1.0, TE stays at ~4.36 µm.
- **H2 sweep**: both TM and TE peaks nearly independent of H2 around 0.1 µm
  (robustness check only).

### PeriodicStructure incidence angle API (verified 2026-07-09)
For Wave Optics `PeriodicStructure`, do **not** look for `Theta0`/`Phi0` on `pport1`. Official COMSOL examples expose the incidence-angle fields as `alpha1_inc` and `alpha2_inc` on the Periodic Structure and mirrored Periodic Port subfeatures.

Verified references:
- `frequency_selective_surface_csrr.mph`: GUI `Periodic Structure > Port Mode Settings > alpha1 = theta`; clientapi shows `ps1.alpha1_inc = theta`, `pport1.alpha1_inc = theta`, `pport2.alpha1_inc = theta`.
- `scatterer_on_substrate.mph`: GUI `alpha1 = theta`, `alpha2 = phi`; clientapi shows `ps1.alpha1_inc = theta`, `ps1.alpha2_inc = phi`, mirrored on `pport1/pport2`.
- Zhou2025 QBIC smoke test: `theta = 0/20/40 deg`, `phi=0`, `wl=4.253 um`; `theta_eval_deg` matched requested values and emissivity changed `0.923969 -> 0.498725 -> 0.109854`, so the angle affected the incident wave.

Minimal staged setup:
```python
jm.param().set("theta", f"{theta_deg}[deg]")
jm.param().set("phi", f"{phi_deg}[deg]")
ps = comp.physics().get("ewfd").feature().get("ps1")
for obj in [ps, ps.feature().get("pport1"), ps.feature().get("pport2")]:
    obj.set("alpha1_inc", "theta")
    obj.set("alpha2_inc", "phi")
ps.set("Polarization", "LinearPol")
ps.set("LinearPol", "S")  # or P, depending on rdir1 convention
```

For staged angle sweeps, set `theta/phi` as model parameters before each one-wavelength solve. It is not necessary to use Wavelength-step auxiliary sweep for angle if the solve is staged. Keep `wl` as the wavelength parameter when material expressions use `wl`.

Runtime reference from Zhou2025 local_060: one `(theta, wl)` solve point took about 10.7-14.1 s; a 28-30 point scan for one theta is roughly 5-6 min plus setup/save overhead.
## Debugging tips
- Probe clientapi methods/overloads: `for mth in obj.getClass().getMethods(): if str(mth.getName())=='create': ...` (JPype reflection, note `str(p.getName())` to avoid Java String errors).
- Geometry bbox: `g.getBoundingBox()` returns `[xmin,xmax,ymin,ymax,zmin,zmax]`.
- `geometry_get_boundaries` returns each boundary's normal + center + bounding_box, use normal to determine face.
- **Distinguishing cell edges vs interior patch edges requires center coordinates** (normal alone is insufficient): cell edges at x=0/P, y=0/P, z=0/H; patch edges at x=L/2 ± L/2, z=d+h etc.
- Comparison test for material effectiveness: change eps_r and check if We changes proportionally (if not, fsp1 dominates, need to add ccn1).
- After standalone script disconnects, cannot start new mph.Client in same Python process ("Only one client can be instantiated per Python session") — must **restart host agent/CLI**. Recommend combining all probe/inspect operations into one script run.
- **After modifying MCP server `src/tools/`, must restart host agent/CLI** (MCP is subprocess, no hot-reload).
