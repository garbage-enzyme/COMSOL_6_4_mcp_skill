# AGENTS.md — cross-tool entry point

This repository ships the `comsol-64-metasurface` agent skill at:

```text
skills/comsol-64-metasurface/SKILL.md
```

Read that short routing entry before driving COMSOL 6.4+, using MPh standalone
or a COMSOL MCP server, changing MCP tools, or auditing a periodic metasurface
model. Then read each reference selected by its routing table completely. Do not
preload every reference module.

The progressive-disclosure layout is intentionally portable across Claude Code,
Codex CLI, and opencode:

- `SKILL.md` contains standard YAML `name` and `description` frontmatter plus the
  short core workflow.
- `references/*.md` contains task-specific detail linked with relative paths.
- No platform-specific tool protocol is required to follow the skill.

The modules cover clientapi overloads, periodic Wave Optics, material/boundary
formulations, durable jobs, Windows load stability, physical evidence, resource
admission, typed derived-model mutation, deployment identity, release gates, and
troubleshooting.

Do not publish usernames, home directories, host-specific thresholds,
credentials, private result values, or internal project/phase labels in this
repository.

Companion MCP server:
https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_4_Calibrated
