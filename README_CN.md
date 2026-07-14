# COMSOL 6.4+ Agent Skill

[English](README.md)

面向 COMSOL Multiphysics 6.4+、MPh 1.3.1 standalone/clientapi、周期 Wave
Optics、耐久求解工作流、物理验证和 COMSOL MCP 开发的渐进式技能。

## CLI 兼容性

同一个技能目录兼容 Claude Code、Codex CLI 和 opencode。

| CLI | 入口 |
| --- | --- |
| Claude Code | 从 `CLAUDE.md` 导入 `skills/comsol-64-metasurface/SKILL.md`。 |
| Codex CLI | 将完整目录安装到 Codex skills 目录，或从 `AGENTS.md` 指向它。 |
| opencode | 将完整目录复制到 `~/.config/opencode/skills/`。 |

`SKILL.md` 是短入口。其路由表使用相对链接指向 `references/` 下的任务模块，
因此 agent 只加载当前任务所需内容，不依赖任何 CLI 专属工具协议。

这种结构也是实际的兼容性选择：部分 agent runtime 的系统指令会在每次匹配任务时
强制完整读取 `SKILL.md`。保持必读入口精简，可以减少重复等待和上下文占用；完整
操作经验仍保留在路由模块中，并在任务涉及该领域时完整读取。

## 覆盖范围

- standalone `ModelClient` 重载差异与几何探测；
- 周期端口、入射角、偏振、CopyFace 网格和斜晶格；
- Drude/损耗符号、层状边界、色散扫描、PML/手动 Floquet；
- durable jobs、取消、Windows 共享/进程身份稳定性和资源门禁；
- R/T/A、物理通量闭合、波长同步、溯源、收敛和场证据；
- 受限派生模型修改、有界 schema、部署身份和发布门禁；
- MIM、光栅、纳米柱、参数扫描和场导出配方。

## 安装

克隆仓库后复制完整技能目录，不能只复制 `SKILL.md`：

```text
skills/comsol-64-metasurface/
```

opencode 示例：

```bash
mkdir -p ~/.config/opencode/skills
cp -r skills/comsol-64-metasurface ~/.config/opencode/skills/
```

Codex 个人安装示例：

```bash
mkdir -p ~/.codex/skills
cp -r skills/comsol-64-metasurface ~/.codex/skills/
```

Claude Code 项目级使用时保留 `CLAUDE.md` 和 `skills/` 的相对位置；个人配置可
从相应 `CLAUDE.md` 导入已安装技能的 `SKILL.md`。

## 目录结构

```text
skills/comsol-64-metasurface/
├── SKILL.md
└── references/
    ├── clientapi-core.md
    ├── durable-runtime.md
    ├── materials-boundaries.md
    ├── mcp-development.md
    ├── troubleshooting.md
    ├── validation-evidence.md
    ├── wave-optics-periodic.md
    └── workflow-recipes.md
```

公开内容不包含用户名、主目录、本机阈值、凭据、私有光谱或内部项目阶段代号。

配套 server：
https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_4_Calibrated

许可证：MIT。
