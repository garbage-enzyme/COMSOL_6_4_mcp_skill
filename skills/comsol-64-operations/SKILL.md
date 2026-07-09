---
name: comsol-64-operations
description: COMSOL Multiphysics 6.4+ with MPh 1.3.1 standalone (clientapi) 操作指南。Use when driving COMSOL 6.4+ via the comsol MCP server (mph.Client standalone) or writing/fixing code in the COMSOL_Multiphysics_MCP src/tools/ directory. Covers clientapi vs direct-Model API differences, geometry boundary probing (getUpDown/faceX/faceNormal), fin Form Assembly (imprint=True, createpairs=False), Electrostatics ChargeConservation trap, Heat Transfer transient pitfalls (TemperatureBoundary / Solid k,rho,Cp / Transient tlist), Wave Optics ewfd periodic metasurface setup (PeriodicStructure/Layered Impedance BC/Drude/LayeredMaterial+LML 4级材料层次/wl参数扫描陷阱), 1D hybrid metagrating Au/aSi MaGMR-SPP thermal emitters, Block boundary numbering, mesh/study pitfalls, MIM paper baseline verification recipe.
---

# COMSOL 6.4+ 操作指南（MPh standalone / clientapi）

教怎么用 comsol MCP 服务器驱动 COMSOL 6.4+，以及在 `COMSOL_Multiphysics_MCP/src/tools/` 写/改代码时必须知道的 clientapi 陷阱。所有结论均经实测（ParallelPlateCapacitor C=1.8593794420 pF = 理论值，误差 7e-10 pF；MIM baseline R 全波长与理论一致）。

## 环境速查
- MCP 仓库（clientapi 校准 fork）：`github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_4_Calibrated`。在该仓库根目录用目标 Python 环境启动：`python -m src.server`。
- **COMSOL 6.4** 自带 Java 21；MPh 1.3.1 + JPype 1.7.x 可用。通常不需要 `sitecustomize.py` JVM patch。
- COMSOL 6.4 新增 **cuDSS**（CUDA Direct Sparse Solver）。切换方法：Stationary solver 子节点 `dDef.set('linsolver','cudss')`（默认 mumps）。GPU 加速通常需要大规模 DOF 和合适硬件才明显。
- 前提：COMSOL 必须先启动。`comsol_start` 首次超时正常，用 `comsol_status` 确认。
- **改 `src/tools/` 源码后必须重启宿主 agent/CLI**（MCP server 是子进程，不热加载）。

## 关键：standalone 用 clientapi，不是直接 Model
mph 1.3.1 `mph.Client(cores=...)` 下 `model.java` 返回 `com.comsol.clientapi.impl.ModelClient`：
- `mj = m.java`（属性，**不 call**）；`mj.physics().get('ewfd')` 等。
- `component().create(tag, True)` — 无 `(str,bool,int)` 重载。空间维度由 `geom().create(tag, sdim)` 决定。
- `physics().create(tag, type, sdim)` — 第三参数 **String**（如 `"3"`），不是 int！取 sdim 用 `comp.geom(tag).getSDim()` → `str()`。
- `feature().create(tag, type, edim)` — 第三参数 **int**（边界维度，如 2）。和 physics 的 String sdim 不同。
- `*.get(i)` int 索引不支持 — 用 `tags()` 遍历：`for t in list.tags(): obj = list.get(t)`。
- `len(geom.feature())` 不支持 — 用 `geom.feature().size()`。
- `geom.getNBoundaries()`/`getNDomains()`/`getSDim()`（首字母大写）；`mesh.getNumElem()`/`getNumVertex()`。
- study step type 必须用**完整名**：`Stationary`/`Transient`（不是 TimeDependent!）/`FrequencyDomain`/`Eigenfrequency`/`Perturbation`。`study_create` 工具已做归一化映射。
- `m.study().run()` 不可用 — 用 `jm.study('std1').run()`。`m.evaluate('V')` 可用。
- 表达式：`1[V]^2` 报语法错误，必须 `(1[V])^2`。
- `phys.tags()`/`feat.tags()` 返回 Java list，用 `list(...)` 转 Python list。
- `geom.feature().get(t).type()` — PhysicsFeatureClient 无 `type()` 属性；用 `.label()` 区分。

## 几何探查 API（clientapi 已验证）
- **selection().entities()** 返回 int[]（用 `list(...)` 转 Python）→ 边界/域编号。⚠️ Common material 无 selection 容器（报"实体没有选择"）；LayeredMaterialLink 选 boundary。
- **geom.getUpDown()** **无参**，返回 `int[2][n_bnd]`：row0 = up dom, row1 = down dom（assembly 模式下 up 全 0）。`list(ud[1])` 得每 bnd down dom。
- `geom.faceX(bn, JArray(JArray(JDouble))([[u, v]]))` 返回 `double[][]`（嵌套），`list(arr)[0]` 取内层，再索引 cx,cy,cz。`geom.faceNormal(bn, pp)` 同。
- `pp` 推荐用 **(0.33, 0.33)** 而非 (0.5, 0.5)（某些 patch 侧面参数范围非 [0,1]² 会报"参数超出范围")。
- `geom.faceParamRange(bn)` 返回 `[umin, umax, vmin, vmax]` → 用中点 fallback。
- `geom.getBoundingBox()` 返回 `[xmin,xmax,ymin,ymax,zmin,zmax]`。
- 几何特征客户端无 `output()`、`faceBB(int)` 等；用 `faceX(bn, pp)` 中心坐标 + 法向判断面位置和朝向。

## fin 节点（FormUnion/FormAssembly）
- **不能 create('fin', any)** — tag `fin` 为"形成联合体/装配"保留，都得 `geom.run()` **自动创建**。
- 修改 fin 类型：`fin = geom.feature().get('fin')` + `fin.set('action', 'assembly')` + `fin.set('createpairs', False)` + `fin.set('imprint', True)`（保留 interior boundary，不分裂为 pair）。
- 然后 `geom.run()` 重新生效。imprint=True 后 Al2O3/air interface 合并为单一 bnd，patch 侧面与 air 也无 pair 分裂，侧面 Floquet 不会触发 pair 冲突。
- **geometry rerun 后所有 selection 自动 update 到新 bnd 编号**（selectionStorage 保留指认）— ltr1/lml_au/pport1/pport2/fpc1/fpc2/材料 selection 无需重设。
- **pport1/pport2 selection 不可编辑**（"Selection is not editable"）— 由 ps1 自动检测，必须 rerun geometry 让它自动重新指认。

## Electrostatics 建模要点（6.3+）

### 1. 默认 fsp1=FreeSpace 用真空，必须手动加 ChargeConservation
6.3+ Electrostatics 默认 domain feature 是 `fsp1`(FreeSpace)，**用真空 eps0，不读材料 relpermittivity**。必须手动加 ChargeConservation：
```python
ccn = p.feature().create('ccn1', 'ChargeConservation', 3)  # 3=sdim(int)
ccn.selection().set([1]); ccn.set('materialType', 'from_mat')
mat = comp.material().create('mat1', 'Common')
mat.propertyGroup('def').set('relpermittivity', '2.1'); mat.selection().set([1])
```
**MCP 工具捷径**：`physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` 自动建。

### 2. Block 边界编号不固定
用 `geometry_get_boundaries`（返回 normal+center+bounding_box）判断。实测 Block [0.01,0.01,0.001]：bnd3=z=0(normal[0,0,-1])，bnd4=z=0.001(normal[0,0,1])。

### 3. Terminal V0 不可靠，用 ElectricPotential
Terminal `V0=1[V]` 实测 ΔV≈0.16V。用 **ElectricPotential** BC：`p.feature().create('ep1','ElectricPotential',2); ep.set('V0','1[V]')`。

### 4. 网格：必须手动创建 mesh 序列
COMSOL 不自动建 mesh。**MCP 工具**：`mesh_sequence_create(mesh_name='mesh1', element_type='FreeTet', build=True)`。

### 5. 电容计算
`C = m.evaluate('2*es.intWe/(1[V])^2', 'pF')`。⚠️ `(1[V])^2` 必须加括号。`es.V` 报未定义变量——用裸 `V`。`es.intWe`/`es.C11`/`es.normE`/`es.normD` 可用。

## ParallelPlateCapacitor 验证 recipe（C=1.8593794420 pF，误差 7e-10 pF）
理论：`C = eps0*eps_r*L²/d = 8.854e-12*2.1*0.01²/0.001 = 1.8593794407 pF`。

### MCP 工具流程
1. `model_create(name='ParallelPlateCapacitor')` → `model_create_component(3D)` → `geometry_create(3D)`
2. `geometry_add_block(size=[0.01,0.01,0.001])` → `geometry_build`
3. `physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` — 自动建 ChargeConservation + 材料
4. `physics_configure_boundary('静电', 'Ground', [3])` — z=0；`physics_configure_boundary('静电', 'ElectricPotential', [4], {V0:'1[V]'})` — z=0.001
5. `mesh_sequence_create(element_type='FreeTet', build=True)` — ≈1651 单元
6. `study_create(study_type='Stationary')` → `study_solve`
7. `results_global_evaluate('2*es.intWe/(1[V])^2', 'pF')` — 1.8593794420

## Heat Transfer 瞬态建模要点（Si 块验证：dT=7.6923K，6 位有效数字匹配）

### 1. 固定温度 BC 类型名是 `TemperatureBoundary`（不是 `Temperature`）
`p.feature().create('temp1','TemperatureBoundary',2); bc.set('T0','293.15[K]')`。

### 2. Solid 域 feature 属性名 vs Material Basic 属性名
`Solid` feature 属性：`k`/`rho`/`Cp`。Material 节点 Basic(`def`) 属性：`thermalconductivity`/`density`/`heatcapacity`。保持 feature `from_mat`，写 Material Basic。
**MCP**：`physics_set_material(physics_name='固体传热', material_name='Silicon', domain_selection=[1], properties={...})`。

### 3. 瞬态 study step type 是 `Transient`（不是 `TimeDependent`）
**MCP**：`study_create(study_type='Transient', time_list=[0,0.001,0.01,0.1], time_unit='s')` 自动设 tlist。

### 4. 瞬态结果取场需指定 inner index + 显式 dataset
`results_evaluate(expression='T', dataset='研究 1//解 1', inner='last', unit='K')`。⚠️ dataset=None 可能报错，用 `datasets_list` 查名。

## Wave Optics (ewfd) 周期超表面仿真

复现 Au-Al₂O₃-Au MIM 超表面热发射器（Chen et al. *Int. J. Thermal Sciences* 185 (2023) 108069）。论文几何：**P=1.35µm, L=0.856µm, h=0.10µm (Au patch 厚), d=0.04µm (Al₂O₃ 间距), tsub=0.10µm (Au substrate 厚)**。共振目标：MP1@4.37µm (ε≈0.95), MP2@2.27µm (ε≈0.5)。机制：MIM trilayer 中 magnetic polariton (MP) — patch 与 substrate 间 loop current 强磁增强。

### PeriodicStructure API（已验证）
- **关键**：必须用 `PeriodicStructure`（父节点），不是 `PeriodicCondition`！
- `comp.physics().create('ps1','PeriodicStructure',str(sdim))` 自动生成 fpc1/fpc2(Floquet) + pport1/pport2(PeriodicPort) + rdir1。
- **激活端口激励**：`ps1.selection("excitedPortSelection").set(top_bnd)` — 关键步骤！
- S 参数变量：`ewfd.Rtotal`/`ewfd.Ttotal`/`ewfd.Atotal`（不是 `ewfd.S11`）。Study type：`Wavelength`。
- 子节点选择器自动分配且锁定，不要手动 set。`addDiffractionOrders`：`ps1.runCommand("addDiffractionOrders")`。
- 参考示例：`metasurface_beam_deflector.mph`（Rtotal=0.122 验证通过）、`plasmonic_wire_grating.mph`、`hexagonal_grating.mph`。

### 3D PeriodicStructure rdir1 / axisy trap
- From-scratch **3D** `PeriodicStructure` may create `ps1/rdir1` with an empty selection. Symptom during solve: `comp1.ewfd.axisy` or `axisx/axisy` undefined on the periodic port boundary.
- Fix by selecting one edge on the excited port to define the first primitive vector: `ps.feature("rdir1").selection().set([edge_id])`. Pick an edge parallel to the intended lattice vector. For a 1D grating with period along `x`, use a top-port edge along `x`.
- 3D `edgeX` probing must use `edgeParamRange(edge)` first; the parameter range is not always `[0,1]`. Use midpoint/endpoint values from the returned range.
- The `PeriodicStructure` subnode has its own `wee1`. If materials are Common materials with `relpermittivity`, also set `ps.feature("wee1").set("DisplacementFieldModel", "RelativePermittivity")`, `mur_mat="userdef"`, `mur="1"`, `sigma_mat="userdef"`, `sigma="0"`.
- After setting `rdir1`, call `ps.runCommand("addDiffractionOrders")` when diffraction orders are needed.

### Long standalone sweep robustness
- Prefer staged one-point solves for fragile wavelength sweeps: set `jm.param().set("wl", value)`, run `jm.study("std1").run()`, evaluate with `EvalGlobal`, append CSV immediately. This avoids COMSOL/mph outer-sweep evaluation surprises such as only seeing the last point.
- Standalone `mph.Client` scripts may finish saving files but not exit because JVM/helper threads stay alive. After all outputs are flushed and saved, `os._exit(0)` is acceptable for long-running handoff scripts.

### ★ 端口失效根因 + 解决方案（PDF 文档 p151, p179-181）
- **根因**：Periodic port 假设**端口相邻域是均匀各向同性介质**（WaveOpticsModuleUsersGuide.pdf p151）。金属 patch（Drude 负 ε）在域中违反此假设，即使端口在空气域也失效。极简测试：介电 patch→R=1.08正常；金属 patch→R=0失效。
- **解决方案：Layered Impedance Boundary Condition**（p179-181）：
  - 支持 **Drude-Lorentz 色散模型**！用 BC 替代金属 patch 体积，域中只留介电材料，端口模式假设满足。
  - = Layered Transition BC + Impedance BC，假设薄层中波沿法向传播（适合 good conductor）。
  - 需 Layered Material（Shell property group）定义 Au 薄膜（厚度 + Drude 属性）。

### ★★ LayeredTransition BC 生效方案（4 级材料层次，缺一不可）
```python
# 1. 全局 Common material mat_au（Drude eps + sigmabnd + murbnd 在 def group）
mat_au = jm.material().create('mat_au','Common')
mat_au.propertyGroup('def').set('relpermittivity', au_drude)
mat_au.propertyGroup('def').set('sigmabnd', '0')   # ★ 必需
mat_au.propertyGroup('def').set('murbnd', '1')     # ★ 必需

# 2. 全局 LayeredMaterial lm_au（层定义）
lm = jm.material().create('lm_au','LayeredMaterial')
lm.set('layername','Au'); lm.set('thickness', str(t_au)); lm.set('link','mat_au')
lm.propertyGroup('def').set('relpermittivity', au_drude)
lm.propertyGroup('def').set('sigmabnd', '0'); lm.propertyGroup('def').set('murbnd', '1')

# 3. component LayeredMaterialLink lml_au（限制到 boundary）
lml = comp.material().create('lml_au','LayeredMaterialLink')
lml.set('link','lm_au')
lml.selection().all(); lml.selection().clear(); lml.selection().add([bnd6])  # ★ 只在 interface
sh = lml.propertyGroup('shell')  # ★ 自动有 shell group
sh.set('lth', str(t_au)); sh.set('relpermittivity', au_drude)
sh.set('sigmabnd', '0'); sh.set('murbnd', '1')

# 4. LayeredTransition BC（shelllist 自动=lml_au，但 sigmabnd/murbnd 必须 userdef）
ltr = p.feature().create('ltr1','LayeredTransitionBoundaryCondition',2)
ltr.selection().set([bnd6])
ltr.set('DisplacementFieldModel','RelativePermittivity')
ltr.set('sigmabnd_mat','userdef'); ltr.set('sigmabnd','0')   # ★ from_mat 不生效！
ltr.set('murbnd_mat','userdef'); ltr.set('murbnd','1')       # ★ from_mat 不生效！
ltr.set('lth', str(t_au))
```
- **验证**：eps=2.1 介电薄膜 → Rtotal=0.0757（薄膜干涉）；Drude 连续膜 → R(1µm)=0.41, R(5µm)≈1.0。
- **SingleLayerMaterial** 也可用（`comp.material().create(tag,'SingleLayerMaterial')`），但同样需 LML + sigmabnd/murbnd userdef。
- **Common material selection 只支持 domain**；LML/SingleLayerMaterial selection 支持 `all()+clear()+add([bnd])` 限制到 boundary。
- **propertyGroup create('shl','Shell')** 返回 type='def'（clientapi 限制）；LML 自动有 'shell' group（含 lth/lrot/lne/relpermittivity/sigmabnd/murbnd）。

### ★ Drude 波长扫描陷阱（已验证）
- `ewfd.freq` 在多波长 Wavelength study 中导致阻抗奇异（Jsupx 无法计算，Zs²-Zt²=0）。
- **解决方案**：`wl` 参数 + `c_const/wl`：
  ```python
  jm.param().set('wl','5e-6[m]')
  au_drude = "1-(1.37e16)^2/((2*pi*c_const/wl)*((2*pi*c_const/wl)+i*4.1e13))"
  # Study: Wavelength step (plist='5e-6' dummy) + Parametric sweep step
  study.create('step1','Wavelength'); step.set('punit','m'); step.set('plist','5e-6')
  study.create('sweep1','Parametric'); sweep.set('pname','wl'); sweep.set('plist','1e-6 2e-6 ...')
  ```
- 连续 Au 膜 Drude 扫描：R(1µm)=0.41, R(2µm)=0.93, R(3µm)=0.997, R(4-10µm)≈1.007（金属全反射基线）。

### ★ 空间变化 lth 不可行（已确认）
- LML Shell group lth 设 `if(x>...)` 表达式 → solve OK 但 R 不变（LML 从全局 LM 读固定 thickness）。
- 全局 LM 设坐标表达式 → 报错"层定义无效"（全局材料不能用坐标）。
- **结论**：patch 空间分块必须用 **geometry partition**（bnd6 分成 patch + rest）才能让 LTR 只覆盖 patch 上方的 interface。

### Patch 几何方案（已验证可行路径）
- **WorkPlane + PartitionFaces**：bnd10 是 WorkPlane 嵌入 3D 的浮动面，**不参与物理场**——LayeredTransition/LayeredImpedance 设在 bnd10 上 R=1 全波长无效。**方案不可行**。
- **Block + Difference + FormAssembly (imprint=True)**：保留 patch 为独立 dom 3，interface 经 imprint 合并为单一 interior bnd。patch 侧面与 air 也无需 pair Continuity（imprint 模式下自动连续）。Floquet 周期 pair 在单元外侧不受干扰。**此方案已 build 验证几何 OK（3 dom, 24 bnd，无重复 pair）**，物理场设 pair Continuity 后待 solve 验证。

### 关键 API 笔记
- `ewfd` 默认特征：`wee1`(波动方程)/`pec1`(PEC)/`init1`(初值)/`dcont1`(连续性)。
- 材料介电常数用**单字符串**（如 `'-972+283.5*i'` 或 Drude 表达式），不要用 `[real,imag]` 数组（报"需要 3x3 矩阵"）。
- Au Drude：`1-(1.37e16)^2/((2*pi*c_const/wl)*((2*pi*c_const/wl)+i*4.1e13))`，wp=1.37e16 rad/s, gamma=4.1e13 rad/s（*已知陷阱*：用 `c_const/wl` 不用 `ewfd.freq`）。
- Box 选择器在 clientapi 下失效，必须用域编号直传。
- `m.evaluate([expr, 'wl'])`（list）取多频扫描结果，每一行是 [expr_value, wl_value]。遍历 inner indices 也可用 `inner=[i,...]`（0-indexed list）。

### 完整 MIM baseline recipe（已跑通）
理论：连续 Au 膜（无 patch）→ Drude 全反射，R(1µm)≈0.41, R(≥4µm)≈1.007（数值误差）。
1. **模板克隆**：从 `MIM_Continuous_Drude.mph` 加载作 baseline（4 级材料 + Sweep mesh + Wavelength+parametric sweep 已全建好）。
2. **几何改尺寸**：`geom.feature().get('b_al2').set('size', jarr([P,P,d]))` + `b_air.set('size', jarr([P,P,H-d]))` + `b_air.set('pos', jarr([0,0,d]))` + `geom.run()`（fin 自动建 FormUnion，selection 自动更新）。
3. **mesh 重建**：MCP `mesh_create`。
4. **solve**：MCP `study_solve`（MCP 工具 timeout 是常态，用 `comsol_status` + `results_inner_values`/`datasets_list` 确认是否有 inner indices 判断完成）。
5. **evaluate**：`results_evaluate(['ewfd.Rtotal', 'wl'])` 返回 10×2 数组（wl 1-10µm, R 列）。
6. **保存**：`m.save()` 或 MCP `model_save`。

输出模型可保存为任意本地 `.mph` 文件；baseline 已跑通时 R 值应与连续膜一致。

## MIM patch MCP 工具（src/tools/mim_patch.py，commit 1ba6a0a + 57801a5）
三个 MIM patch 工作流辅助工具，端到端测试于 MIM_paper_baseline_v1 → patch 几何 + mesh 自动构建成功（3 dom, 17 bnd, 18837 elements）。

### 1. geometry_probe_domains
- 增强版 `geometry_get_boundaries`：每 bnd 带 up/down domain、interior flag、normal、center；额外返回 `interior_boundaries`（up!=0 and down!=0）、`pairs`（comp.pair().tags()）、`side_pairs`（自动分类 cell 周期单元边）。
- side_pairs 分类依据：法向 + **中心点坐标**双重过滤（用 `geom.getBoundingBox()` 作 bbox），tol=1e-12。**只用法向会把 patch 侧面/顶面误判成 cell 边界**（commit 57801a5 修复）——CopyFace 会因此把 patch 侧面网格复制到 cell 边界，break Floquet mesh 兼容。
- 例：3 dom patch 几何，`x_src=[1,4]`（cell 边 x=0），不含 patch 侧面 bnd10（x=L/2）。

### 2. mim_patch_build(patch_size=[L,L,h], patch_pos=[x0,y0,d])
- 从 2-dom baseline（Al2O3+air）构建 patch 几何：Block + Difference(keepsubtract=True) → FormUnion（imprint 自动）+ 更新 LayeredTransition→patch footprint (interior bnd where up=patch_dom, down=al2_dom) + LayeredImpedance→bottom + air material→patch_dom + FreeTri+CopyFace+FreeTet mesh。
- **参数**：patch_size/patch_pos 单位米。论文值 L=0.856µm, h=0.10µm, pos=[(P-L)/2,(P-L)/2,d=4e-8]。
- **air_block_tag 自动检测**：找 `size` z 分量 > 1e-7 的 Block feature（air 块比 Al2O3 高得多）。
- **patch_dom = n_domains**（最后添加的 dom）。
- **已知 limitation**：若旧 `_identify_side_pairs` 缺坐标过滤 → PS port set 多个 bnd 报"只允许单个边界"。57801a5 修复后正常。

### 3. mim_evaluate_spectral
- 一行 evaluate `ewfd.Rtotal`/`Ttotal`/`Atotal` + `wl` 参数，返回 `{wl_um, Rtotal, Ttotal, Atotal, emissivity}` 列表，emissivity=1-R。
- 实测 patch_final：ε@4-5µm≈0.89（论文 MP1@4.37µm ε≈0.95），ε@2µm≈0.88（论文 MP2@2.27µm ε≈0.5）。
- 底层 `model.evaluate(expr_list)`，wl 列 ×1e6 转 µm。**注意添加 'wl' 参数后再调，否则 KeyError**。

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

### Final-check addendum
- In a verified In:CdO MIM run, focused fine sweeps moved one mid-density point back to the paper peak and confirmed another point's higher FEM global peak within tolerance. Use focused fine sweeps before calling a side peak physical or spurious.
- Point-field profiles at competing wavelengths can show whether peaks belong to the same mode family. Similar field distributions imply a method/boundary-condition residual rather than a simple material-parameter error.

## Zhou 2024 1D hybrid metagrating (verified recipe)
Paper: Zhou et al., IEEE Sensors Journal 24(13), 2024. Structure: 1D Au grating + aSi cladding. TM excites leaky SPP on Au; TE excites MaGMR (guided-mode resonance) in aSi.

### Key results (Structure II, 3D thin-cell FEM)
| Mode | FEM | Paper | Q | Emissivity |
| --- | --- | --- | --- | --- |
| TM SPP | 5.295 µm | 5.27 µm | 46 | 0.993 |
| TE MaGMR | 4.360 µm | 4.28 µm | 125 | 0.982 |
Geometry: P=1.75µm, W=0.80µm, H2=0.10µm (groove depth), H1=0.61µm (aSi on ridge), Au mirror 0.10µm. Materials: Au Drude (wp=1.37e16, gamma=1e14), aSi lossless eps=11.9, Air eps=1. Mesh: ~12k elements, ~2s/solve per wl.

### Targets (all three structures)
| Structure | P | W | H2 | H1 | TE peak | TM peak |
| --- | --- | --- | --- | --- | --- | --- |
| I | 1.40 µm | 0.70 µm | 0.10 µm | 0.44 µm | 3.31 µm | 4.23 µm |
| II | 1.75 µm | 0.80 µm | 0.10 µm | 0.61 µm | 4.28 µm | 5.27 µm |
| III | 2.00 µm | 0.90 µm | 0.10 µm | 0.61 µm | 4.61 µm | 5.71 µm |

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

## 调试技巧
- 探测 clientapi 方法/重载：`for mth in obj.getClass().getMethods(): if str(mth.getName())=='create': ...`（JPype 反射，注意 `str(p.getName())` 避免 Java String 报错）。
- 几何 bbox：`g.getBoundingBox()` 返回 `[xmin,xmax,ymin,ymax,zmin,zmax]`。
- `geometry_get_boundaries` 返回每边界的 normal + center + bounding_box，直接用法向判断面。
- **辨别 cell 边 vs interior patch 边必须看中心坐标**（法向不够）：cell 边在 x=0/P, y=0/P, z=0/H；patch 边在 x=L/2 ± L/2, z=d+h 等。
- 对比实验判断材料是否生效：改 eps_r 看 We 是否按比例变（不变则 fsp1 主导，需加 ccn1）。
- standalone 脚本断开后无法在同一 Python 进程再起新 mph.Client（"Only one client can be instantiated per Python session"）— 必须**重启宿主 agent/CLI**。crawl_mph 脚本障碍 → 建议各 probe/inspect 操作合并到一次脚本里跑完。
- **改 MCP server `src/tools/` 后必须重启宿主 agent/CLI**（MCP 是子进程，不热加载）。
