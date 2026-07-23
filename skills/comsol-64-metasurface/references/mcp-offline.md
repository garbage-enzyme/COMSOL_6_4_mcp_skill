# COMSOL MCP unavailable fallback

Read this reference only when COMSOL MCP is unavailable. Prefer the MCP server
whenever it can accept serialized calls; it supplies ownership, containment,
durable-state, and evidence controls that direct MPh does not recreate.

## Gate

Use direct MPh only after one of these terminal conditions:

- no COMSOL MCP client or server is configured on the host;
- the server executable/import is unavailable; or
- a serialized startup attempt completed with a terminal failure.

Do not use this fallback to bypass a profile, containment rule, input bound, or
MCP error. Do not use it while `comsol_start` is in flight, after a timeout whose
original request may still run, or while a healthy MCP session is connected.

Before creating a direct client, perform a fresh process inventory and refuse if
any COMSOL, Java, MPh, solver-worker, or uncertain lease owner exists. Do not
start a second client to recover a failed direct-client process; terminate that
Python process and begin again only after a fresh inventory.

## Install MPh

COMSOL Multiphysics must already be installed and licensed locally. Create a
non-editable, ASCII-path environment; never install a fallback into the project
source tree:

```powershell
py -3.14 -m venv D:\condaenvs\comsol-standalone
D:\condaenvs\comsol-standalone\Scripts\python.exe -m pip install --upgrade pip
D:\condaenvs\comsol-standalone\Scripts\python.exe -m pip install "mph==1.3.1" "jpype1==1.7.1"
```

When discovery cannot find the local COMSOL runtime, set process-local Java
paths before importing `mph`, using the Java runtime shipped with the installed
COMSOL version:

```powershell
$env:JAVA_HOME = 'D:\COMSOL64\Multiphysics\java\win64\jre'
$env:JDK_HOME = $env:JAVA_HOME
```

These paths are examples, not portable defaults. Verify the installed COMSOL
root and requested version first. Direct MPh is not a license bypass.

## Minimal isolated run

Use one source model, one declared point, and one unique output path. Hash the
source before and after; save through the Java ClientAPI so Unicode destinations
remain reliable.

```python
from hashlib import sha256
from pathlib import Path

import mph


def file_hash(path: Path) -> str:
    digest = sha256()
    with path.open("rb") as handle:
        for block in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(block)
    return digest.hexdigest()


source = Path(r"D:\approved_models\source.mph").resolve()
output = Path(r"D:\owned_artifacts\derived_point.mph").resolve()
source_hash = file_hash(source)
client = mph.Client(version="6.4")
model = client.load(str(source))
try:
    jm = model.java
    # Apply only a declared, bounded derived mutation and one study point.
    jm.save(str(output))
finally:
    client.remove(model)

if file_hash(source) != source_hash:
    raise RuntimeError("immutable source changed")
```

- Use `client.remove(model)`, not `model.remove()`, to release a loaded model.
- Do not call `client.disconnect()` for a standalone client that never connected
  to a COMSOL Server.
- Use `str(tag)` before applying Python string operations to Java tag values.
- Probe study feature names on a derived copy or exported trusted model. Do not
  assume every baseline accepts the same study type.

## Manual controls and evidence

Record the preflight process inventory, source/output absolute paths, source and
output SHA-256 values, output size, exact COMSOL/MPh/JPype/Python versions,
declared inputs, raw evaluated values, and post-run process absence. Persist
each point before starting another.

Direct MPh lacks MCP's solver lease, path containment, durable job/cancellation,
profile discovery, and evidence-integrity controls. Apply those checks manually
or label the result as a limited standalone smoke. A successful native call or
finite number is not physical validation; use declared material, geometry,
mesh, boundary, convergence, and independent-reference evidence for the claim.
