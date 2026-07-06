# COMSOL MCP Skill — 6.3 ClientAPI Operations

English | [中文](README_CN.md)

An [opencode](https://opencode.ai) agent skill that teaches AI assistants how to drive **COMSOL Multiphysics 6.3** through the [comsol MCP server](https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_3_Calibrated) (MPh 1.3.1 standalone / `clientapi`), and how to write or fix code in `src/tools/` when the API mismatches.

> ⚠️ The repository name says `6_2`, but the skill content is calibrated for **COMSOL 6.3 + MPh 1.3.1 standalone** (the `clientapi` wrapper layer). On 6.2 the same patterns apply if you use `mph.Client(standalone)`; the direct-`Model` API on 5.x is different.

## What's inside

A single opencode skill:

```
skills/comsol-63-operations/SKILL.md
```

It is loaded automatically by opencode when the agent's task matches its `description` (driving COMSOL 6.3 via the comsol MCP server, or editing `src/tools/` in the MCP project). No code, no runtime dependency — just a focused instruction document.

## What the skill covers

- **clientapi vs direct-Model differences** — the recurring gotchas when `model.java` returns `com.comsol.clientapi.impl.ModelClient` instead of `com.comsol.model.Model`: `tags()` iteration vs int index, `feature().size()` vs `len()`, `getNBoundaries()` capitalization, `physics().create(tag, type, sdim_String)` three-arg signature, study step type full names, `mesh.getNumElem()`, `jm.study().run()` instead of `m.study().run()`.
- **6.3 Electrostatics traps** — the default `fsp1` (FreeSpace) domain feature ignores material `relpermittivity`; you must add a `ChargeConservation` feature + material node. Block boundary numbering is NOT 1–6 ↔ −x/+x/−y/+y/−z/+z (bnd 3 = z=0, bnd 4 = z=max). `Terminal` `V0` doesn't pin voltage — use `ElectricPotential`. Expression syntax: `1[V]^2` is a parse error, must be `(1[V])^2`.
- **Mesh & study pitfalls** — COMSOL doesn't auto-create a mesh sequence; study step type must use full names (`Stationary`, not `stat`).
- **Capacitance verification recipe** — both pure-Python (mph Client) and MCP-tool-flavored end-to-end recipes for a parallel-plate capacitor. Verified result: **C = 1.8593794414 pF** vs theory 1.8593794407 pF (err 4 × 10⁻¹⁰ pF).
- **Debugging tips** — JPype reflection to probe clientapi overloads, `Box` selection to identify boundary numbers by coordinate, `m.evaluate('V')` field arrays.

## Install

### Option A — into your opencode skills directory

opencode auto-loads skills from `~/.config/opencode/skills/` (or `%USERPROFILE%\.config\opencode\skills\` on Windows). Copy the skill folder there:

```bash
git clone https://github.com/garbage-enzyme/COMSOL_6_2_mcp_skill.git
mkdir -p ~/.config/opencode/skills
cp -r COMSOL_6_2_mcp_skill/skills/comsol-63-operations ~/.config/opencode/skills/
```

On Windows PowerShell:

```powershell
git clone https://github.com/garbage-enzyme/COMSOL_6_2_mcp_skill.git
$dest = "$env:USERPROFILE\.config\opencode\skills"
New-Item -ItemType Directory -Path $dest -Force | Out-Null
Copy-Item -Recurse "COMSOL_6_2_mcp_skill\skills\comsol-63-operations" $dest
```

Restart opencode. The skill's `description` frontmatter will match COMSOL-related tasks automatically — no explicit invocation needed.

### Option B — read it directly

The skill is a single markdown file. Open `skills/comsol-63-operations/SKILL.md` and use it as a reference when writing COMSOL MCP code or driving the server by hand.

## Prerequisites

The skill assumes you already have:

- COMSOL Multiphysics 6.3 installed and running
- The [comsol MCP server fork](https://github.com/garbage-enzyme/COMSOL_Multiphysics_MCP_6_3_Calibrated) configured in opencode (or another MCP client)
- MPh 1.3.1 + JPype in the MCP server's Python environment

The skill itself has no dependencies — it's just instructions for the agent.

## Provenance

The skill was distilled from a real debugging session where an AI assistant (opencode + glm-5.2) calibrated the MCP server's `src/tools/` for 6.3 clientapi under human direction, then verified end-to-end through the MCP tool interface. The companion fork repo has the code changes and full `git log`.

## License

MIT — see [LICENSE](LICENSE).
