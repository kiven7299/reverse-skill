# IDA ↔ reverse-skill wiring (portable)

Generic steps. No machine-absolute paths. Local readiness report lives at repo-root `LOCAL-READINESS.md` (gitignored).

## Target shape

| Item | Convention |
|----|------|
| IDA install dir | env `IDADIR` (dir contains `ida.exe` or `ida.dll`) |
| HTTP MCP | `http://127.0.0.1:13337/mcp` |
| Client server name | keep only **`idapro`** (do not also register `ida-pro-mcp`) |
| Start (this machine) | `scripts/start-gui.ps1 -Path "<binary>"` — no headless idalib license; `-A` skips the load dialog |
| Start (headless) | `scripts/start.ps1` only after an idalib license is confirmed |
| Open DB | GUI: file is opened by `start-gui.ps1`; headless large files use `scripts/open.ps1` |

Two MCP names pointing at the same 13337 register tools twice and race the idalib worker for the port.

## Install

```powershell
setx IDADIR "<your IDA install directory>"

# MUST use mrexodia/ida-pro-mcp; do not install PyPI ida-mcp
python -m pip install "git+https://github.com/mrexodia/ida-pro-mcp.git"

# Activate idalib (adjust path for this machine's IDA)
python "<IDADIR>\idalib\python\py-activate-idalib.py" -d "<IDADIR>"

# Install plugin + client config
python -m ida_pro_mcp --install --transport streamable-http --scope global
```

## Start and keep-alive

An MCP entry with `type: http` does not spawn the process. If 13337 is not listening, every client reports error.

| Script | Role |
|------|------|
| `scripts/start.ps1` | healthy → `OK:<n>:reuse` and refresh last-healthy; port listening but RPC timeout = busy, do not kill; replace managed supervisor only when nobody is listening, `py_eval` is missing, or tools/list has failed continuously for over 3 minutes (and no `opening.lock`); never kill `ida.exe` |
| `scripts/watchdog.ps1` | inspect every minute; healthy reuse; GUI / `open.ps1` open-lock / last-healthy under 3 minutes → reuse; **`-Force` only after tools/list failed continuously for over 3 minutes** |
| `scripts/recover.ps1` | immediate `-Force` restart of supervisor (does not kill `ida.exe`). Use when an HTTP client marks `idapro` as error |
| `scripts/install-autostart.ps1` | register scheduled task `reverse-skill-ida-mcp` (logon + every minute) |
| `scripts/start-gui.ps1` | open GUI plugin when idalib license fails |
| `scripts/open.ps1` | HTTP call `idb_open` directly; bypass some client schema checks |

Logs: `%LOCALAPPDATA%\reverse-skill\ida-mcp\supervisor.log` and `watchdog.log`.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\start.ps1"
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\open.ps1" -Path "C:\path\to\target.exe" -TimeoutSeconds 600
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\install-autostart.ps1"
```

When GUI holds 13337 but has not answered yet, `start.ps1` prints `WARN:gui_busy` and exits so it does not kill an IDA that is still analyzing.

## Clients

All point at Streamable HTTP: `http://127.0.0.1:13337/mcp`, server name `idapro`.

After config changes, open a new session. If Cursor starts while the port is down, bringing the service up later **will not auto-reconnect**; refresh manually in the MCP panel.

## Known caveats

1. System32 files: `open.ps1` copies to a temp path (output includes `(temp copy)`)
2. Do not call `idb_open` via some client MCP paths
3. `start.ps1` prefers `python -m ida_pro_mcp.idalib_supervisor`; more stable than the `.cmd` wrapper
4. When a full install and a desktop portable pack coexist, `IDADIR` wins
5. Do not add `?ext=dbg` (debugger tools are not exposed by default)
