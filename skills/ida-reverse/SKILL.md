---
name: ida-reverse
description: |
  IDA Pro reverse-analysis assistant. Use when the user mentions reverse engineering, decompilation, analyzing binaries/PE/ELF/APK/DLL/SO, cracking, finding passwords, vulnerability analysis, malware analysis, firmware analysis, or needs to analyze exe/dll/so/elf/macho/sys files.

  Ensure to use this skill when the user wants to analyze any binary file, regardless of whether they explicitly mention "IDA" or "reverse engineering". This includes requests like "look at this exe", "analyze this dll", "help me crack this", "find the password", "how does this software register", etc.

  Use the bundled scripts (scripts/start.ps1, scripts/open.ps1) for deterministic server management and file opening — do NOT write ad-hoc PowerShell commands for these operations.
---

# IDA Pro reverse analysis skill

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

## Known issues and lessons (required reading)

### Pitfalls already hit

1. **Do not rely on some AI-client MCP calls for `idb_open` (old name `idalib_open`)**
   - Some code-AI MCP clients have a BUG validating open-class tool output schemas
   - Error: `Structured content does not match the tool's output schema`
   - **Fix**: use `scripts/open.ps1` to call the HTTP API directly, bypassing the MCP validation layer
   - Current ida-pro-mcp 2.x tool names are `idb_open` / `idb_list` / `idb_save` (no longer `idalib_*`)
   - Opening a file returns `session_id` (database); later tool calls MUST carry that session

2. **Cannot open files under `C:\Windows\System32\`**
   - idalib cannot read files in System32 directly
   - **Fix**: `open.ps1` auto-detects and copies to a temp dir before opening

3. **Start-server command blocks the conversation**
   - After start, `idalib-mcp` keeps writing INFO logs to the console
   - **Fix**: use `scripts/start.ps1` (`-WindowStyle Hidden` silent background start)
   - The script waits until the service is ready, then exits; it does not block the conversation

4. **MCP server names cannot use hyphens**
   - Using `ida-pro-mcp` as the server name could break tool registration
   - **Current config**: server name `idapro`, tool prefix `idapro_*`

5. **Remote HTTP vs Local Stdio**
   - `type:"local"` (stdio): `idalib_open` has the same schema validation problem
   - `type:"remote"` (HTTP): open the file with the script first, then use MCP tools
   - **Current approach**: Remote HTTP mode

6. **PR #389 fixed part of the schema issue**
   - Author mrexodia merged a fix in PR #389 after issue #388
   - HTTP-mode structuredContent schema was fixed, but some code-AI client-side validation still fails
   - Latest `main` branch is installed

7. **idalib timeout leaves orphan worker process lock files**
   - After the first `open.ps1` timeout, idalib's python worker child may become an orphan and hold `.id0`/`.id1`/`.nam`
   - Later tools or dragging into IDA GUI then report "access denied"
   - **MUST NOT** `taskkill /F /T` the process tree — `/T` also kills GUI `ida.exe` children
   - **Fix**: `start.ps1` replaces the managed supervisor only when nothing listens on the port, or `tools/list` returns quickly but lacks `py_eval` (old supervisor); RPC timeout with 13337 still listening is busy, do not kill. While opening, `open.ps1` writes `opening.lock`; watchdog MUST NOT `-Force`
   - **Deadlock exception**: only if `tools/list` **fails continuously for more than 3 minutes** (by last-healthy timestamp, not process create time), and there is no in-flight `opening.lock`, and GUI does not own the port, then `-Force` replace supervisor; still do not kill `ida.exe`
   - **Fallback**: `open.ps1` detects a locked old DB, copies to Temp with a GUID prefix

8. **Open with auto-analysis looks hung**
   - `idalib_open(run_auto_analysis=true)` may not reply for a long time while the backend keeps opening and analyzing
   - Users previously saw "PowerShell has no output" and misread it as a hung script
   - **Current fix**: `open.ps1` added `-TimeoutSeconds`, background request + foreground poll + periodic progress
   - When the session is ready it returns early `OK:filename:session_id`; timeout returns `ERR:open_timeout_xxs`

9. **HTTP MCP exits silently after login**
   - Cursor/Claude `type: http` does not spawn the process; old scheduled tasks ran once at login
   - `pythonw` has no console; Application log is empty on crash
   - **Fix**: `start.ps1` reuses when healthy; `watchdog.ps1` inspects every minute; logs at `%LOCALAPPDATA%\reverse-skill\ida-mcp\`
   - Install: `scripts/install-autostart.ps1`. If the HTTP client starts before the port is up, still refresh once in the MCP panel

10. **Streamable HTTP GET `/mcp` stalls a single-thread supervisor**
    - Some HTTP MCP clients send a long-lived GET (SSE) to `/mcp`. Stock `idalib_supervisor` uses `HTTPServer` with `background=False` and handles one request at a time
    - Result: `tools/list` times out; the client marks `idapro` as error
    - **Fix**: `run-supervisor.py` switches HTTP to `ThreadingHTTPServer` and accepts GET `/mcp`; if the patch fails, skip it and still start the supervisor. When stuck, use `scripts/recover.ps1` (immediate `-Force`)

### Workflow principles

| Step | What | With |
|------|--------|--------|
| 1 | Ensure the HTTP server is running | `scripts/start.ps1` (no args) |
| 2 | Open the target binary | `scripts/open.ps1 -Path "xxx.exe"` |
| 3 | Use MCP analysis tools | Call `idapro_*` / HTTP tools directly (~65, version-dependent) |
| 4 | After analysis | Tools stay available |

## Script resources

### start.ps1 — start MCP HTTP server

Path: `scripts/start.ps1`

- Auto-resolve `IDADIR` (env / portable desktop path / common install paths)
- Prefer IDA-bundled `Python314\python.exe -m ida_pro_mcp.idalib_supervisor`
- Default: probe `http://127.0.0.1:13337/mcp`; if healthy, print `OK:<n>:reuse` and exit
- 13337 listening but `tools/list` timeout → `WARN:busy` / `OK:busy:reuse`, **do not kill** (open or GUI occupancy cannot reply)
- `tools/list` **fails continuously more than 3 minutes** (last-healthy timestamp) and no `opening.lock` → deadlock, print `INFO:deadlock` and `-Force` replace supervisor. In-flight `idb_open` and GUI do not take this path
- Replace managed supervisor only when the port is unused, `py_eval` is missing, or the deadlock above; **never kill `ida.exe`, never `taskkill /T`**
- GUI owns 13337 → print `WARN:gui_busy` and exit; do not start another supervisor
- Success prints `OK:<tool-count>` (currently ~66); failure prints `ERR:timeout`
- Supervisor log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\supervisor.log`
- Server runs in the background and does not block the conversation

**Invocation**:
```
powershell -File "<skill-root>\ida-reverse\scripts\start.ps1"
```

### watchdog.ps1 / recover.ps1 / install-autostart.ps1 — keep-alive

- `watchdog.ps1`: probe 13337; healthy reuse (and refresh last-healthy); GUI / `open.ps1` open lock / busy with last-healthy under 3 minutes → reuse; only if `tools/list` fails continuously more than 3 minutes, `start.ps1 -Force`
- `recover.ps1`: immediate `start.ps1 -Force` (does not kill `ida.exe`). Use when the HTTP client marks `idapro` as error
- `install-autostart.ps1`: register scheduled task `reverse-skill-ida-mcp` (logon + every minute)
- Log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\watchdog.log`

### open.ps1 — open a binary

Path: `scripts/open.ps1`

- Call `idb_open` via HTTP API, bypass MCP schema validation
- Auto-detect System32 paths and copy to a temp dir
- Auto-clean same-name old DB files (`.id0`/`.id1`/`.nam`/`.til`/`.i64`)
- If the old DB is locked, degrade automatically: copy to Temp with a GUID prefix and open; do not error
- Run the open request in the background so long sync waits do not freeze the script
- Supports `-TimeoutSeconds`; on timeout returns `ERR:open_timeout_xxs` instead of hanging forever
- Every 10 seconds prints `INFO:opening:elapsed/timeout-seconds` so analysis-in-progress is visible
- Success prints `OK:filename:session_id`; degrade adds `(temp copy)`
- On failure, auto-retry via the Temp copy

**Invocation**:
```
powershell -File "<skill-root>\ida-reverse\scripts\open.ps1" -Path "C:\path\to\file.exe"
```

**Optional args**:
```
# Specify SessionId
powershell -File "scripts\open.ps1" -Path "file.exe" -SessionId "my_session"

# Skip auto-analysis (recommended for large files)
powershell -File "scripts\open.ps1" -Path "large.exe" -NoAutoAnalysis

# Set timeout so auto-analysis does not stall with no return
powershell -File "scripts\open.ps1" -Path "file.exe" -TimeoutSeconds 600
```

**Output contract**:
```
# Analysis in progress (every 10 seconds)
INFO:opening:11/600s

# Opened successfully
OK:sample.exe:abcd1234

# Opened successfully, degraded to Temp copy because of lock files
OK:1234abcd-sample.exe:abcd1234 (temp copy)

# Hit timeout cap
ERR:open_timeout_600s
```

**Measured notes**:
- `Snipaste.exe` with auto-analysis took about `324s` to succeed; that is "analysis is slow", not "script deadlock"
- For GUI programs or complex samples, prefer explicit `-TimeoutSeconds 600`

## Core tool list

### Overview (step 1)
- `idapro_survey_binary(detail_level="minimal")` — fast overview: function count, strings, segments, entry, import classes (crypto/network/file IO)
- `idapro_list_funcs(queries)` — list functions (paged, filter by name)
- `idapro_list_globals(queries)` — list globals
- `idapro_entity_query(kind, filter)` — unified query: functions/globals/imports/strings/names

### Decompile and disassembly
- `idapro_decompile(addr)` — decompile to pseudocode
- `idapro_disasm(addr, max_instructions=N)` — disassemble
- `idapro_analyze_function(addr, include_asm=false)` — combined analysis (pseudocode+strings+constants+callers+callees+blocks)
- `idapro_func_profile(queries)` — function summary metrics

### Cross-refs and data flow
- `idapro_xrefs_to(addrs)` — who references the target address
- `idapro_xref_query(addr, direction)` — advanced xref query (direction/type filter)
- `idapro_callees(addrs)` — callee list
- `idapro_callgraph(roots, max_depth)` — call graph
- `idapro_trace_data_flow(addr, direction, max_depth)` — data-flow trace (forward/backward)

### Search
- `idapro_find_regex(pattern, limit)` — regex search strings
- `idapro_search_text(pattern)` — search text in the disassembly listing
- `idapro_find_bytes(patterns, limit)` — byte-pattern search (supports ?? wildcards)
- `idapro_find(type, targets)` — advanced search (immediates/strings/refs)

### Memory and data
- `idapro_get_bytes(addrs)` — read raw bytes
- `idapro_get_string(addrs)` — read strings
- `idapro_get_int(queries)` — read integer values
- `idapro_get_global_value(queries)` — read global values
- `idapro_read_struct(queries)` — read struct field values
- `idapro_search_structs(filter)` — search structs

### Mutations
- `idapro_set_comments(items)` — add comments (disasm+decompile two-way sync)
- `idapro_append_comments(items)` — append comments
- `idapro_rename(batch)` — batch rename (functions/globals/locals/stack vars)
- `idapro_patch_asm(items)` — Patch assembly
- `idapro_patch(patches)` — Patch bytes
- `idapro_define_func(items)` — define functions
- `idapro_undefine(items)` — undefine
- `idapro_define_code(items)` — turn bytes into code

### Type system
- `idapro_declare_type(decls)` — declare C structs/enums/unions
- `idapro_set_type(edits)` — apply types to functions/globals/locals
- `idapro_infer_types(addrs)` — infer types
- `idapro_type_query(queries)` — query declared types
- `idapro_type_inspect(queries)` — inspect type details

### Stack frame
- `idapro_stack_frame(addrs)` — view stack-frame vars
- `idapro_declare_stack(items)` — declare stack vars
- `idapro_delete_stack(items)` — delete stack vars

### Signatures
- `idapro_make_signature(addrs)` — unique byte signature for an address
- `idapro_make_signature_for_function(addrs)` — signature for a function
- `idapro_find_xref_signatures(addrs)` — signatures for code that references an address

### Debugger (needs ?ext=dbg)
- `idapro_open_file(file_path)` — open a file in a GUI IDA instance
- Debugger tools are hidden by default; enable with URL param `?ext=dbg`

### Session management (ida-pro-mcp 2.x)
- `idapro_idb_open` / HTTP `idb_open` — ⚠️ prefer `open.ps1`
- `idapro_idb_list` / HTTP `idb_list` — list all sessions
- `idapro_idb_save` / HTTP `idb_save` — save the database
- Most analysis tools need `database=<session_id>` (the session from open.ps1)

### Other
- `idapro_int_convert(inputs)` — radix convert (**MUST use this; do not convert radices yourself!**)
- `idapro_export_funcs(addrs, format)` — export functions (json/c_header/prototypes)
- `idapro_py_eval(code)` — run Python in IDA context
- `idapro_server_health()` — server health check
- `idapro_server_warmup()` — warm subsystems (string cache, Hex-Rays, etc.)

## Full reverse-analysis workflow

### Step 1: Start the server

**Path A — Headless idalib (needs a valid license)**
```
powershell -File "scripts/start.ps1"
```
`OK:<tool-count>` (currently ~65) means ready.

**Path B — GUI + plugin (idalib license failed or interactive analysis needed)**
```
powershell -File "scripts/start-gui.ps1" -Path "C:\target.exe"
```
Or double-click portable `Launch-IDA-Pro.cmd` and open the sample in IDA.

After the Output window shows `[MCP] ... port=13337`, MCP tools are usable.

Generic attach steps: `LOCAL-SETUP.md`.

### Step 2: Open the file

Headless:
```
powershell -File "scripts/open.ps1" -Path "C:\target.exe" -TimeoutSeconds 600
```
`OK:filename:session_id` means success (trailing `(temp copy)` means auto-degrade to a temp copy).

If `ERR:idalib_license:...` appears, switch to path B (GUI mode); do not keep retrying open.ps1.

GUI mode: Open the sample in IDA; open.ps1 is not needed.

### Step 3: Global overview (includes import-table hard gate)
```
idapro_survey_binary(detail_level="minimal")
```
Watch:
- Architecture (x86/x64/ARM)
- Entry (main/WinMain/DllMain)
- Interesting strings (URL, paths, error messages)
- **Import classes (MUST)**: crypto / network API / file ops / process injection / registry — MUST land as Evidence (suggested id: `E-imports`); use `idapro_entity_query(kind="imports")` or the imports section of survey output
- **DLL/SYS**: export table alongside import table (Evidence `E-exports`)
- **.NET**: no classic IAT; write module/metadata/managed-ref summary as the equivalent anchor into the E-imports semantic slot
- **Clean import table**: note dynamic-load suspicion; push dynamic API breakpoint verification
- Hot functions (high xref count is often key logic)

**Hard gate**: MUST NOT enter Step 4 deep conclusions, and MUST NOT claim survey complete, until the imports view/class summary (or a legal equivalent anchor) is written as Evidence. If the import table is empty or the query fails, still MUST record the failure. If packed IAT repair fails, MUST record `E-iat-repair-fail` and switch to dynamic debug to catch APIs; MUST NOT grind statically. If the user asks to redo import-table/IAT checks, MUST redo the named step (on block: feasibility latch — explain + confirm; if forced, mark quality=unreadable); MUST NOT swap in unrelated steps.

### Step 4: Deep-dive key functions
```
idapro_analyze_function(addr="key_function_name")
```
or:
```
idapro_decompile(addr="function_name")
idapro_disasm(addr="function_name", max_instructions=50)
```

### Step 5: Data flow and cross-refs
```
idapro_xrefs_to(addrs="key_address/string")
idapro_callgraph(roots=["key_function"], max_depth=3)
idapro_trace_data_flow(addr="key_address", direction="backward", max_depth=5)
```

### Step 6: Record and refine
```
idapro_set_comments(items=[{"addr": "0x140001000", "comment": "your understanding"}])
idapro_rename(batch={"func": [{"addr": "function_address", "name": "meaningful_name"}]})
```

### Step 7: Write the report
After analysis, generate `report.md` with findings and steps.

## Prompt engineering rules

1. **Do not convert radices by hand** — whenever a number must be converted, use `idapro_int_convert`
2. **Survey before deep dive** — overview first, then targeted analysis
3. **Keep adding comments and renames** — update function and variable names during analysis to improve later accuracy
4. **Follow cross-refs** — on interesting data/strings, use `xrefs_to` to see who references them
5. **On obfuscated code** — preprocess first: string decrypt, import-hash strip, control-flow flattening removal
6. **C++ STL code** — identify library functions with FLIRT/Lumina, then analyze business logic
7. **Do not brute-force** — derive the solution from disassembly; use simple Python only as a helper
8. **On "No database bound"** — no binary is open yet; run `open.ps1` first
9. **On "Failed to open database"** — old DB files may be locked; `open.ps1` auto-degrades to a Temp copy (output includes `(temp copy)`)
10. **Opening GUI/complex samples with auto-analysis** — default add `-TimeoutSeconds 600`; do not treat long `INFO:opening:...` as a hung script

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Upstream fallback**: `radare2/` (if IDA should not be opened, r2 for fast recon first)
**Downstream exits**:
- Need Frida dynamic verify → `reverse-engineering/tools-dynamic.md`
- Need symbolic execution/angr → `reverse-engineering/tools-dynamic.md`
- Need general RE methodology → `reverse-engineering/SKILL.md`

**Peer modules**: `radare2/` (fallback when IDA is unavailable)

---

## On-Demand Bootstrap

This skill's entry scripts are wired to the unified bootstrap system.

### Automation bounds

| Tool | Auto-install | Method | Notes |
|------|-----------|---------|------|
| idalib-mcp | ✓ | pip install (from GitHub) | Auto-install when missing in `start.ps1` |
| IDA Pro itself | ✗ | Commercial; install manually | Set `IDADIR` to the install dir |

### Install steps (verified)

```cmd
# 1. Set IDA path (replace with your actual IDA install dir)
setx IDADIR "<your-IDA-install-dir>"

# 2. Install ida-pro-mcp from GitHub (PyPI ida-mcp is a different project; do not install the wrong one!)
pip install git+https://github.com/mrexodia/ida-pro-mcp.git

# 3. Install the IDA plugin (choose Streamable HTTP + Global + all clients)
ida-pro-mcp --install

# 4. Restart IDA Pro, open the target file
# Plugin listens on 127.0.0.1:13337 automatically

# 5. Verify
ida-pro-mcp --config
```

> ⚠️ **Note**: The PyPI `ida-mcp` package (author jtsylve) is a different project.
> MUST install `mrexodia/ida-pro-mcp` from GitHub.

### Bootstrap triggers

- `scripts/start.ps1`: missing `idalib-mcp` → auto-call `bootstrap-reverse.ps1`
- MCP registration: bootstrap auto-writes `idapro` into Claude MCP config

### Prerequisites

- IDA Pro installed and `IDADIR` set (or the script default path is correct)
- Prefer `ida-pro-mcp` in IDA-bundled Python314 (portable build already includes it)
- Common local config:
  - User env `IDADIR` → IDA install dir (contains `ida.exe`)
  - Optional `~\Tools\bin\idalib-mcp.cmd` / `ida-pro-mcp.cmd` wrappers
  - Client MCP server name only `idapro` → `http://127.0.0.1:13337/mcp`


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Are survey/imports written as Evidence (E-imports or equivalent)? DLL/SYS includes E-exports? IAT failure recorded as E-iat-repair-fail?
- [ ] If the user asked to redo the import table/IAT, was the same step redone?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES checklist items?
