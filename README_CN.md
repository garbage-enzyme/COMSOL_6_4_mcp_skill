# COMSOL MCP Skill — 6.3 ClientAPI 操作技能

[English](README.md) | 中文

一个 [opencode](https://opencode.ai) agent skill，教 AI 助手如何通过 [comsol MCP server](https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_3_Calibrated)（MPh 1.3.1 standalone / `clientapi`）驱动 **COMSOL Multiphysics 6.3**，以及在 `src/tools/` 里写/改代码时如何应对 API 不匹配。

> ⚠️ 仓库名是 `6_2`，但 skill 内容是为 **COMSOL 6.3 + MPh 1.3.1 standalone**（`clientapi` 包装层）校准的。6.2 下若用 `mph.Client(standalone)` 同样适用；5.x 的直接 `Model` API 不同。

## 里面有什么

一个 opencode skill：

```
skills/comsol-63-operations/SKILL.md
```

当 agent 的任务匹配其 `description`（通过 comsol MCP server 驱动 COMSOL 6.3，或编辑 MCP 项目的 `src/tools/`）时，opencode 会自动加载。没有代码、没有运行时依赖 —— 就是一份聚焦的指令文档。

## Skill 覆盖内容

- **clientapi vs 直接 Model 的差异** —— `model.java` 返回 `com.comsol.clientapi.impl.ModelClient`（而非 `com.comsol.model.Model`）时的常见坑：`tags()` 遍历 vs int 索引、`feature().size()` vs `len()`、`getNBoundaries()` 首字母大写、`physics().create(tag, type, sdim_String)` 三参数、study step type 用完整名、`mesh.getNumElem()`、用 `jm.study().run()` 而非 `m.study().run()`。
- **6.3 Electrostatics 陷阱** —— 默认 `fsp1` (FreeSpace) domain feature 用真空 ε₀，忽略材料 `relpermittivity`，必须加 `ChargeConservation` feature + 材料节点。Block 边界编号不是 1–6 ↔ −x/+x/−y/+y/−z/+z（bnd 3 = z=0 面，bnd 4 = z=最大值面）。`Terminal` 的 `V0` 不能正确约束电压 —— 用 `ElectricPotential`。表达式语法：`1[V]^2` 会报语法错误，必须 `(1[V])^2`。
- **网格与求解陷阱** —— COMSOL 不自动创建 mesh 序列；study step type 必须用完整名（`Stationary`，不是 `stat`）。
- **电容验证 recipe** —— 纯 Python（mph Client）和 MCP 工具流程两版平行板电容器端到端 recipe。验证结果：**C = 1.8593794414 pF**，理论值 1.8593794407 pF（误差 4 × 10⁻¹⁰ pF）。
- **调试技巧** —— JPype 反射探测 clientapi 重载、用 `Box` selection 按坐标识别边界编号、`m.evaluate('V')` 取场数组。

## 安装

### 方式 A —— 装进 opencode skills 目录

opencode 会自动从 `~/.config/opencode/skills/`（Windows 下 `%USERPROFILE%\.config\opencode\skills\`）加载 skill。把 skill 文件夹拷过去：

```bash
git clone https://github.com/garbage-enzyme/COMSOL_6_2_mcp_skill.git
mkdir -p ~/.config/opencode/skills
cp -r COMSOL_6_2_mcp_skill/skills/comsol-63-operations ~/.config/opencode/skills/
```

Windows PowerShell：

```powershell
git clone https://github.com/garbage-enzyme/COMSOL_6_2_mcp_skill.git
$dest = "$env:USERPROFILE\.config\opencode\skills"
New-Item -ItemType Directory -Path $dest -Force | Out-Null
Copy-Item -Recurse "COMSOL_6_2_mcp_skill\skills\comsol-63-operations" $dest
```

重启 opencode。skill 的 `description` frontmatter 会自动匹配 COMSOL 相关任务 —— 无需显式调用。

### 方式 B —— 直接看

skill 就是单个 markdown 文件。打开 `skills/comsol-63-operations/SKILL.md`，当写 COMSOL MCP 代码或手动驱动 server 时当参考用。

## 前提条件

本 skill 假设你已有：

- 已安装并运行的 COMSOL Multiphysics 6.3
- 在 opencode（或其他 MCP 客户端）配置好的 [comsol MCP server fork](https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_3_Calibrated)
- MCP server 的 Python 环境里有 MPh 1.3.1 + JPype

skill 本身无依赖 —— 它只是给 agent 的指令。

## 来源

本 skill 提炼自一次真实调试会话：AI 助手（opencode + glm-5.2）在人工指导下为 6.3 clientapi 校准 MCP server 的 `src/tools/`，随后通过 MCP 工具接口端到端验证。配套的 fork 仓库有代码改动和完整 `git log`。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。
