---
name: comsol-63-operations
description: COMSOL Multiphysics 6.4 + MPh 1.3.1 standalone (clientapi) 操作指南。Use when driving COMSOL 6.4 via the comsol MCP server (mph.Client standalone) or writing/fixing code in the COMSOL_Multiphysics_MCP src/tools/ directory. Covers clientapi vs direct-Model API differences, Electrostatics ChargeConservation trap, Heat Transfer transient pitfalls (TemperatureBoundary / Solid k,rho,Cp / Transient tlist), Wave Optics ewfd periodic metasurface setup (PeriodicStructure/Layered Impedance BC/Drude), Block boundary numbering, mesh/study pitfalls, capacitance + transient thermal verification recipes.
---

# COMSOL 6.4 操作指南（MPh standalone / clientapi）

本指南教你怎么用 comsol MCP 服务器驱动 COMSOL 6.4，以及在 COMSOL_Multiphysics_MCP 项目的 `src/tools/` 里写/改代码时必须知道的 clientapi 陷阱。所有结论均经实测（ParallelPlateCapacitor C=1.8593794420 pF = 理论值，误差 7e-10 pF）。

## 环境速查
- MCP 仓库（clientapi 校准 fork）：`github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_3_Calibrated`。
- Python 环境：`D:\condaenvs\comsol-mcp`（Python 3.11）。依赖：mcp、mph 1.3.1、jpype 1.7.1、pydantic、pymupdf、chromadb 0.5.23、sentence-transformers 3.4.1。
- 启动：`python -m src.server`（cwd=项目目录）。
- **COMSOL 6.4**：`D:\COMSOL64\Multiphysics`。Java 21（6.4 自带 Temurin 21.0.7，`D:\COMSOL64\Multiphysics\java\win64\jre`）。JPype 1.7.1 支持。**不需要** sitecustomize.py JVM patch（已改名 .bak）。
- CUDA Toolkit 12.4 已装，RTX 3060 12GB。6.4 新增 cuDSS（CUDA Direct Sparse Solver），支持所有物理场直接求解器 GPU 加速。
- 前提：COMSOL 必须先启动。`comsol_start` 首次超时正常，用 `comsol_status` 确认。
- **改 src/tools/ 源码后必须重启 opencode**（MCP server 是子进程，不热加载）。

## 关键：standalone 用 clientapi，不是直接 Model
mph 1.3.1 `mph.Client(cores=...)` 下 `model.java` 返回 `com.comsol.clientapi.impl.ModelClient`，方法重载和直接 Model 不同：

- `component().create(tag, True)` — 无 `(str,bool,int)` 重载。空间维度由 `geom().create(tag, sdim)` 决定。
- `physics().create(tag, type, sdim)` — 第三参数 **String**（如 `"3"`），不是 int！取 sdim 用 `comp.geom(tag).getSDim()` → `str()`。
- `feature().create(tag, type, edim)` — 第三参数 **int**（边界维度，如 2）。和 physics 的 String sdim 不同。
- `*.get(i)` int 索引不支持 — 用 `tags()` 遍历：`for t in list.tags(): obj = list.get(t)`。
- `len(geom.feature())` 不支持 — 用 `geom.feature().size()`。
- `geom.getNBoundaries()`/`getNDomains()`/`getSDim()`（首字母大写）；`mesh.getNumElem()`/`getNumVertex()`。
- study step type 必须用**完整名**：`Stationary`/`Transient`（不是 TimeDependent!）/`FrequencyDomain`（不是 Frequency!）/`Eigenfrequency`/`Perturbation`。`study_create` 工具已做归一化映射。
- `m.study().run()` 不可用 — 用 `jm.study('std1').run()`。`m.evaluate('V')` 可用。
- 表达式：`1[V]^2` 报语法错误，必须 `(1[V])^2`。

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

### MCP 工具流程（已验证通过）
1. `model_create(name='ParallelPlateCapacitor')`
2. `model_create_component(space_dimension=3)` → `geometry_create(space_dimension=3)`
3. `geometry_add_block(size=[0.01,0.01,0.001])` → `geometry_build`
4. `physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` — 自动建 ChargeConservation + 材料
5. `physics_configure_boundary('静电', 'Ground', [3])` — z=0
6. `physics_configure_boundary('静电', 'ElectricPotential', [4], {V0:'1[V]'})` — z=0.001
7. `mesh_sequence_create(element_type='FreeTet', build=True)` — ≈1651 单元
8. `study_create(study_type='Stationary')` → `study_solve`
9. `results_global_evaluate('2*es.intWe/(1[V])^2', 'pF')` — 返回 1.8593794420

### 纯 Python（mph Client）recipe
```python
import mph, jpype
client = mph.Client(version='6.4')  # 6.4 自动发现
m = client.create('PPC'); jm = m.java
comp = jm.component().create('comp1', True)
g = comp.geom().create('geom1', 3)
blk = g.feature().create('blk1', 'Block')
blk.set('size', jpype.JArray(jpype.JDouble)([0.01,0.01,0.001])); g.run()
p = comp.physics().create('es', 'Electrostatics', str(g.getSDim()))
mat = comp.material().create('mat1','Common'); mat.propertyGroup('def').set('relpermittivity','2.1'); mat.selection().set([1])
ccn = p.feature().create('ccn1','ChargeConservation',3); ccn.selection().set([1]); ccn.set('materialType','from_mat')
p.feature().create('gnd1','Ground',2).selection().set([3])
ep = p.feature().create('ep1','ElectricPotential',2); ep.selection().set([4]); ep.set('V0','1[V]')
mesh = comp.mesh().create('mesh1'); mesh.feature().create('ftr1','FreeTet'); mesh.run()
study = jm.study().create('std1'); study.create('step1','Stationary'); jm.study('std1').run()
C = m.evaluate('2*es.intWe/(1[V])^2','pF')  # 1.8593794420
```

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

### Si 块瞬态热传 MCP recipe（已验证）
理论：`dT=q*L/k=1e6*0.001/130=7.6923K`，T_hot=300.8423K。
1. `model_create` → `model_create_component(3D)` → `geometry_create(3D)` → `geometry_add_block([0.01,0.01,0.001])` → `geometry_build`
2. `physics_add_heat_transfer()` → `physics_set_material('固体传热','Silicon',[1],{thermalconductivity:'130[W/(m*K)]',density:'2329[kg/m^3]',heatcapacity:'700[J/(kg*K)]'})`
3. `physics_setup_heat_boundaries('固体传热', heat_flux_boundaries=[3], heat_flux_value='1e6[W/m^2]', temperature_boundaries=[4], temperature_value='293.15[K]')`
4. `mesh_sequence_create(build=True)` → `study_create(Transient, time_list=[0,0.001,0.01,0.1], time_unit='s')` → `study_solve`
5. `datasets_list` → `results_evaluate('T', dataset='研究 1//解 1', inner='last', unit='K')` → max≈300.8423K

## Wave Optics (ewfd) 周期超表面仿真

复现 Au-Al₂O₃-Au MIM 超表面热发射器（Chen et al. *Int. J. Thermal Sciences* 185 (2023) 108069），ε=1-|S11|²。

### PeriodicStructure API（已验证）
- **关键**：必须用 `PeriodicStructure`（父节点），不是 `PeriodicCondition`！
- `comp.physics().create('ps1','PeriodicStructure',str(sdim))` 自动生成 fpc1/fpc2(Floquet) + pport1/pport2(PeriodicPort) + rdir1。
- **激活端口激励**：`ps1.selection("excitedPortSelection").set(top_bnd)` — 关键步骤！
- S 参数变量：`ewfd.Rtotal`/`ewfd.Ttotal`/`ewfd.Atotal`（不是 `ewfd.S11`）。Study type：`Wavelength`。
- 子节点选择器自动分配且锁定，不要手动 set。`addDiffractionOrders`：`ps1.runCommand("addDiffractionOrders")`。
- 参考示例：`metasurface_beam_deflector.mph`（Rtotal=0.122 验证通过）、`plasmonic_wire_grating.mph`、`hexagonal_grating.mph`。

### ★ 端口失效根因 + 解决方案（PDF 文档 p151, p179-181）
- **根因**：Periodic port 假设**端口相邻域是均匀各向同性介质**（WaveOpticsModuleUsersGuide.pdf p151）。金属 patch（Drude 负 ε）在域中违反此假设，即使端口在空气域也失效。极简测试：介电 patch→R=1.08正常；金属 patch→R=0失效。
- **解决方案：Layered Impedance Boundary Condition**（p179-181）：
  - 支持 **Drude-Lorentz 色散模型**！用 BC 替代金属 patch 体积，域中只留介电材料，端口模式假设满足。
  - = Layered Transition BC + Impedance BC，假设薄层中波沿法向传播（适合 good conductor）。
  - 需 Layered Material（Shell property group）定义 Au 薄膜（厚度 + Drude 属性）。
  - Electric displacement field model 选项：Relative permittivity / Refractive index / Loss tangent / **Drude-Lorentz** / Debye / Sellmeier。
  - 文档示例：`enhanced_mems_mirror_coating`（需优化模块许可，本机装不了）。

### 关键 API 笔记
- `ewfd` 默认特征：`wee1`(波动方程)/`pec1`(PEC)/`init1`(初值)/`dcont1`(连续性)。
- 材料介电常数用**单字符串**（如 `'-972+283.5*i'` 或 Drude 表达式），不要用 `[real,imag]` 数组（报"需要 3x3 矩阵"）。
- Au Drude：`1-(1.37e16)^2/((2*pi*ewfd.freq)*((2*pi*ewfd.freq)+i*4.1e13))`，wp=1.37e16 rad/s, gamma=4.1e13 rad/s。
- Box 选择器在 clientapi 下失效，必须用域编号直传。
- `model.evaluate('ewfd.S11', inner=[i])` 用 0-indexed list 取多频扫描第 i 个。

## 调试技巧
- 探测 clientapi 方法/重载：`for mth in obj.getClass().getMethods(): if str(mth.getName())=='create': ...`（JPype 反射，注意 `str(p.getName())` 避免 Java String 报错）。
- 几何 bbox：`g.getBoundingBox()` 返回 `[xmin,xmax,ymin,ymax,zmin,zmax]`。
- `geometry_get_boundaries` 返回每边界的 normal + center + bounding_box，直接用法向判断面。
- 对比实验判断材料是否生效：改 eps_r 看 We 是否按比例变（不变则 fsp1 主导，需加 ccn1）。
