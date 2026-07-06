---
name: comsol-63-operations
description: COMSOL Multiphysics 6.3 + MPh 1.3.1 standalone (clientapi) 操作指南。Use when driving COMSOL 6.3 via the comsol MCP server (mph.Client standalone) or writing/fixing code in C:\Users\陆星\COMSOL_Multiphysics_MCP\src\tools\. Covers clientapi vs direct-Model API differences, Electrostatics ChargeConservation trap, Block boundary numbering, mesh/study pitfalls, capacitance verification recipe.
---

# COMSOL 6.3 操作指南（MPh standalone / clientapi）

本指南教你怎么用 comsol MCP 服务器驱动 COMSOL 6.3，以及在 `C:\Users\陆星\COMSOL_Multiphysics_MCP\src\tools\` 里写/改代码时必须知道的 clientapi 陷阱。所有结论均经实测（2026-07-06，ParallelPlateCapacitor 端到端 C=1.8593794414 pF = 理论值，误差 4e-10 pF）。

## 环境速查
- COMSOL 6.3 安装路径：`D:\COMSOL63\Multiphysics`
- MCP 项目：`C:\Users\陆星\COMSOL_Multiphysics_MCP`（github.com/wjc9011/COMSOL_Multiphysics_MCP）
- Python 环境：`D:\condaenvs\comsol-mcp`（conda, Python 3.11, 清华镜像源）
- 启动 MCP server：`D:\condaenvs\comsol-mcp\python.exe -m src.server`（cwd=项目目录）
- 依赖：mcp、mph 1.3.1、pydantic、pymupdf、chromadb、sentence-transformers
- Java：6.3 自带 Temurin **Java 11**（`D:\COMSOL63\Multiphysics\java\win64\jre`），JPype 直接可用。**不需要** Java 17 补丁（那是 5.6 Java 8 才需要）。
- 前提：COMSOL Multiphysics 必须先启动好（MCP 通过 MPh/JPype 桥接 COMSOL Java API）。用 MCP 工具时 `comsol_start` 启动 standalone client（首次启动慢，可能超时一次，用 `comsol_status` 确认连上即可）。
- **改了 src/tools/ 源码后必须重启 opencode**，因为 MCP server 是子进程，不热加载。

## 关键：6.3 standalone 用 clientapi，不是直接 Model
mph 1.3.1 的 `mph.Client(cores=...)`（standalone 模式）下，`model.java` 返回的是 **`com.comsol.clientapi.impl.ModelClient`**（clientapi 包装），不是 `com.comsol.model.Model`。所有 `model.java.component()` / `.physics()` / `.geom()` 返回的都是 `*Client` 类，方法重载和直接 Model **不同**。写/改 `src/tools/` 代码时务必遵守：

- `component().create(tag, True)` — 只有 `(str)`/`(str,bool)`/`(str,str)` 重载，**没有 `(str,bool,int)`**。组件空间维度不由 create 设，由后续 `geom().create(tag, sdim)` 决定（COMSOL 官方文档确认）。
- `physics().create(tag, type, sdim)` — 三参数第三参数是 **String**（如 `"3"`），不是 int！传 int 报 "No matching overloads"。两参数 create 报"物理场接口不支持空间维度: 0维"（必须传 sdim）。取 sdim 用 `comp.geom(tag).getSDim()` 返回 int，再 `str(...)` 传给 create。
- `feature().create(tag, type, edim)` — boundary/domain feature 的第三参数是 **int**（边界维度，如 2）。和 physics interface 的 String sdim 不同，别混。
- `*.get(i)` 用 int 索引 — clientapi 的 `ModelEntityListClient.get` 只接受 String tag。遍历用 `tags()`：`for t in list.tags(): obj = list.get(t)`。
- `len(geom.feature())` 不支持 — 用 `geom.feature().size()`。
- `geom.getNboundary()` 不存在 — 用 `geom.getNBoundaries()`（首字母大写）；`getNDomains()`、`getSDim()` 同理。
- `mesh.getNumElem()` / `getNumVertex()`（返回 int）— 不是 `getElement().size()`。
- `study.create(tag)` 后 `study.create('step1','Stationary')` — **study step type 必须用完整名**（`Stationary`/`TimeDependent`/`Eigenfrequency`/`Frequency`/`Perturbation`），不是短名 `stat`/`time`/`eig`！clientapi 下传短名报 `Operation_cannot_be_created_in_this_context`。
- `m.study().run()` 不可用 — mph 1.3.1 的 Model 无 `.study()`，用 `jm.study('std1').run()`（java 层）。
- 高层 `m.evaluate('V')` 可用（返回场变量 numpy 数组）；但 `es.V`、`max(es.V)` 报"未定义变量"——用裸 `V`。`es.intWe`、`es.C11`、`es.normE`、`es.normD` 可用。

## 6.3 Electrostatics 建模要点（5.6 没有这些坑）

### 1. 默认 fsp1=FreeSpace 用真空，必须手动加 ChargeConservation
6.3 的 Electrostatics 创建时默认 domain feature 是 `fsp1` (FreeSpace)，**用真空 eps0，不读材料 relpermittivity**。要给介电层设 eps_r，必须手动创建 ChargeConservation feature：
```python
ccn = p.feature().create('ccn1', 'ChargeConservation', 3)  # 3=sdim(int, 边界维度)
ccn.selection().set([1])            # 赋给域 1
ccn.set('materialType', 'from_mat')  # 从材料取
# 配合材料：
mat = comp.material().create('mat1', 'Common')
mat.propertyGroup('def').set('relpermittivity', '2.1')   # 属性名是 relpermittivity，在 propertyGroup('def') 下
mat.selection().set([1])
```
fsp1 不能 remove（内部 feature），但加了 ccn1 并 set selection 后 ccn1 覆盖 fsp1 生效。
不创建 ccn1 的后果：We 精确等于 eps_r=1 的理论值（材料完全被忽略）。

**MCP 工具捷径**：`physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` 会自动建材料节点 + ChargeConservation feature。不传 relpermittivity 则只建裸 es 接口（真空）。

### 2. Block 边界编号不是 1-6 对应 -x/+x/-y/+y/-z/+z
实测 Block size [0.01,0.01,0.001] pos [0,0,0]（x∈[0,0.01], y∈[0,0.01], z∈[0,0.001]）：
- **边界 3 = z=0 面，边界 4 = z=0.001 面**
- 边界 1,2,5,6 是侧面（跨越 z 全程）

设 ground/terminal 时**不要假设 5=-z/6=+z**。用 Box selection 按坐标确认：
```python
sel = comp.selection().create('box_z0', 'Box')
sel.set('entitydim', '2'); sel.set('condition', 'inside')   # inside 精确，默认 intersects 会返回所有相交面
sel.set('xmin', '-0.0001'); sel.set('xmax', '0.0101')
sel.set('ymin', '-0.0001'); sel.set('ymax', '0.0101')
sel.set('zmin', '-0.00005'); sel.set('zmax', '0.00005')
z0_boundaries = list(sel.entities())   # [3]
```
注意 `set` 的值传 **String**（如 `'2'`），传 int 会触发 `set(str,int)` 歧义重载报错。

### 3. Terminal feature 的 V0 没正确约束电压
Terminal feature `TerminalType=Voltage` + `V0=1[V]` 实测 ΔV≈0.16V（原因未深究）。平行板电容验证用 **ElectricPotential** 边界条件更直接可靠：
```python
gnd = p.feature().create('gnd1', 'Ground', 2); gnd.selection().set([3])      # z=0
ep  = p.feature().create('ep1',  'ElectricPotential', 2); ep.selection().set([4])  # z=0.001
ep.set('V0', '1[V]')
```

### 4. 几何 size 用 JArray JDouble
`blk.set('size', [...])` 建议传 `jpype.JArray(jpype.JDouble)([0.01,0.01,0.001])` 数值数组，而非字符串数组（更稳，bbox 验证尺寸正确）。

### 5. 网格：必须手动创建 mesh 序列（COMSOL 不自动建）
COMSOL 不会自动建 mesh 序列，必须显式 create：
```python
mesh = comp.mesh().create('mesh1')          # 单参数即可
mesh.feature().create('ftr1', 'FreeTet')    # 自由四面体
# 可选加 Size 控制粗细：
# sz = mesh.feature().create('size1', 'Size'); sz.set('hmax', '0.0002')
mesh.run()
nelem = mesh.getNumElem()    # clientapi 用 getNumElem/getNumVertex（返回 int）
```
**MCP 工具**：`mesh_sequence_create(mesh_name='mesh1', element_type='FreeTet', build=True)` 一步到位（默认 build=True 立即生成并返回元素数）。`mesh_create` 只 run 已有序列，无法创建——新建网格用 `mesh_sequence_create`。

### 6. 电容计算
```python
# 方式 A：直接在 evaluate 里算（注意 [V]^2 必须加括号成 (1[V])^2，否则报"表达式语法错误 位置16 ^2"）
C = m.evaluate('2*es.intWe/(1[V])^2', 'pF')   # 返回 pF 数值
# 方式 B：分步
We = float(m.evaluate('es.intWe'))   # 电场能量 (J)
C = 2 * We / V**2                    # V 是电压差
# Terminal 电容 es.C11 需有 Terminal，但 Terminal V0 不可靠（见上），推荐上面公式
```
⚠️ `1[V]^2` 在 clientapi 下报语法错误，必须 `(1[V])^2`。

## ParallelPlateCapacitor 端到端验证 recipe（C=1.8593794414 pF = 理论值，误差 4e-10 pF）

理论值：`C = eps0 * eps_r * L^2 / d = 8.8541878128e-12 * 2.1 * 0.01^2 / 0.001 = 1.8593794407 pF`。

### 纯 Python（mph Client）recipe
```python
import mph, jpype
client = mph.Client(version='6.3')
m = client.create('ParallelPlateCap'); jm = m.java
for n,v in (('L','0.01[m]'),('d','0.001[m]'),('epsr','2.1'),('V0','1[V]')):
    jm.param().set(n,v)
comp = jm.component().create('comp1', True)
g = comp.geom().create('geom1', 3)
blk = g.feature().create('blk1', 'Block')
blk.set('size', jpype.JArray(jpype.JDouble)([0.01,0.01,0.001]))   # 10mm x 10mm x 1mm
blk.set('pos',  jpype.JArray(jpype.JDouble)([0,0,0]))
g.run()                                          # ndom=1, nbnd=6
p = comp.physics().create('es', 'Electrostatics', str(g.getSDim()))
mat = comp.material().create('mat1','Common'); mat.label('dielectric')
mat.propertyGroup('def').set('relpermittivity','2.1'); mat.selection().set([1])
ccn = p.feature().create('ccn1','ChargeConservation',3); ccn.selection().set([1])
ccn.set('materialType','from_mat')
gnd = p.feature().create('gnd1','Ground',2); gnd.selection().set([3])        # z=0
ep  = p.feature().create('ep1','ElectricPotential',2); ep.selection().set([4]); ep.set('V0','V0')  # z=0.001
mesh = comp.mesh().create('mesh1')
mesh.feature().create('ftr1','FreeTet'); mesh.run()   # nelem≈1663
study = jm.study().create('std1')
study.create('step1','Stationary')      # ← clientapi 必须完整名，不是 'stat'！
jm.study('std1').run()                  # ← mph 1.3.1 Model 无 .study()，用 jm.study().run()
C = m.evaluate('2*es.intWe/(1[V])^2','pF')   # ← [V]^2 必须加括号
# C = 1.8593794414 pF
```

### MCP 工具等价流程（已验证通过）
按顺序调用：
1. `model_create(name='ParallelPlateCapacitor')`
2. `model_create_component(space_dimension=3)`
3. `geometry_create(space_dimension=3)`
4. `geometry_add_block(size=[0.01,0.01,0.001], position=[0,0,0])`
5. `geometry_build`
6. `physics_add_electrostatics(relpermittivity=2.1, domain_numbers=[1])` — 自动建 ChargeConservation + 材料
7. `physics_configure_boundary('静电', 'Ground', [3])` — z=0 面
8. `physics_configure_boundary('静电', 'ElectricPotential', [4], {V0:'1[V]'})` — z=0.001 面
9. `mesh_sequence_create(mesh_name='mesh1', element_type='FreeTet', build=True)` — 返回 num_elements≈1663
10. `study_create(study_type='Stationary')`
11. `study_solve`
12. `results_global_evaluate('2*es.intWe/(1[V])^2', 'pF')` — 返回 1.8593794414

## 调试技巧
- 探测 clientapi 对象的方法/重载：`for mth in obj.getClass().getMethods(): if str(mth.getName())=='create': ...`（JPype 反射，注意 `str(p.getName())` 避免 Java String 报错）
- 几何 bbox：`g.getBoundingBox()` 返回 `[xmin,xmax,ymin,ymax,zmin,zmax]`
- 边界坐标确认：用 Box selection（condition='inside'）按 z 坐标筛边界编号
- 场变量评估：`m.evaluate('V')` 返回 numpy 数组，配合 `m.evaluate('z')` 查指定面电势
- 对比实验判断材料是否生效：改 eps_r 看 We 是否按比例变（不变则 fsp1 主导，需加 ccn1）

## 已知缺口
- `geometry_get_boundaries` 工具返回 nBoundaries/nDomains 但**不返回每个边界的坐标/法向**，且当前报 `'ComponentGeomListClient' object is not subscriptable`（`_get_geometry_node` 用 `jm.component(name)` 取 comp 后 `comp.geom()` 下标访问的 clientapi 兼容问题未修）。判断具体边界编号请用上面的 Box selection 方法绕过。
