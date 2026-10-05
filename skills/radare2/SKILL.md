---
name: radare2
description: |
  Use this skill whenever the user wants to analyze binaries with radare2/r2 from the command line, including reverse engineering, disassembly, function analysis, strings/import inspection, patching, binary diffing, hex inspection, or r2 scripting. Also use it when the user mentions PE/ELF/Mach-O/DEX/WASM files together with CLI analysis, `rabin2`, `rasm2`, `radiff2`, `r2pipe`, or asks for radare2 command help on Windows/Linux/macOS.
---

# radare2

Binary analysis skill for the `radare2` CLI. Focus is recon, analysis, location, export, and light edits from the command line, without a GUI.

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

## Scope

Prefer this skill when the user intends to:

- Analyze `exe`, `dll`, `so`, `elf`, `apk`, `dex`, `wasm` with `r2` / `radare2`
- Ask how to use `rabin2`, `rasm2`, `radiff2`, `rahash2`, `rax2`
- Need CLI disassembly, functions, strings, imports/exports, xrefs, or patch
- Need `radare2` batch commands, `-c` automation, or `r2pipe` scripts

If the user explicitly wants GUI reverse, Hex-Rays-style pseudocode, or an IDA workflow, prefer `ida-reverse`. For web JS reverse, prefer `reverse-engineering`.

## Confirm the environment first

Do not assume `r2` is available. Check first:

```powershell
r2 -v
rabin2 -v
```

If not installed, check common install locations or prompt to install.

Common Windows executables:

- `radare2.exe`
- `rabin2.exe`
- `rasm2.exe`
- `radiff2.exe`
- `rahash2.exe`
- `rax2.exe`
- `r2pm.exe`

## Bundled resources

This skill ships two resources. Reuse them; do not rebuild a duplicate command set each time.

### `scripts/recon.ps1`

Standard recon script for the first-pass overview. Emits:

- Basic info
- Sections
- Imports
- Exports
- Strings
- Optional `r2 -A` auto-analysis summary

Invocation:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe"
```

If `r2` auto-analysis is also needed:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe" -RunAnalysis
```

### `references/cheatsheet.md`

When more command detail, scenario templates, or syntax recall is needed, read this cheatsheet instead of guessing from memory.

## Known behavior

### Occasional missing `.sdb` warning on Windows

Some PE files may warn during `rabin2` recon like:

```text
ERROR: Cannot find ...\share\format\dll\*.sdb
```

If the main output still returns normally, this usually does not affect basic recon conclusions. Continue analysis. Do not treat this side warning as analysis failure.

## Core principles

### 1. Recon first, then deep dive

Do not start with full auto-analysis. First confirm type, arch, entry, strings, and import table with light commands, then decide whether to run `aaa`, `aaaa`, or targeted analysis.

### 2. Prefer the smallest sufficient command

`radare2` has many commands. Users usually need the shortest path:

- File info: `rabin2 -I`
- Strings: `rabin2 -z`
- Imports/exports: `rabin2 -i` / `rabin2 -E`
- Interactive analysis: `r2 <file>` then local commands

### 3. Be careful before mutating

If the user wants to patch a binary:

- Default open read-only: `r2 <file>`
- Use write mode only when mutation is explicit: `r2 -w <file>` or `oo+` in session
- State the risk before editing so the original file is not overwritten by accident

## Common workflows

## Workflow 1: Fast recon

Use when a binary was just received.

### Hard gate (MUST — MUST NOT enter workflow 2+ until met)

For PE/ELF/Mach-O and other binaries with an import table, **MUST** finish the import-table check and land Evidence before function-level analysis or dynamic steps:

1. Run `rabin2 -i <sample>` (or the imports section of `recon.ps1` output); DLL/SYS also MUST `rabin2 -E` and record `E-exports`
2. Write the full/classified import-table result as Evidence (suggested id: `E-imports` or `E-triage-imports`), at least:
   - Repro command (`repro_command`)
   - Key import class summary: network / file / crypto / process injection / registry / other suspicious APIs
   - If the import table is empty, parse fails, or the tool errors: still MUST record the failure and raw output as Evidence; **do not skip silently**
   - Import table "too clean" (only base DLLs): MUST note dynamic-load suspicion; SHOULD switch to dynamic API capture
3. .NET and others with no classic IAT: MUST use an equivalent anchor (dnSpy/IL/metadata summary) in the same Evidence semantic slot; MUST NOT skip empty
4. Packed-sample IAT repair: x86 use ImportREC (or equivalent), x64 use Scylla (or equivalent). Repair failure MUST record `E-iat-repair-fail` then switch to dynamic API breakpoints; **MUST NOT** grind forever on static IAT (see `reverse-engineering/references/re-agent-workflow.md` §1.2)
5. When the user explicitly asks to "redo import-table check / recheck imports / redo IAT": MUST redo the named step itself (on block, take the feasibility latch first: state prerequisites + ask confirm; if forced, mark quality=unreadable); **MUST NOT swap unrelated steps and claim done**

Until import-table (or a legal equivalent anchor / IAT-fail bypass) Evidence is recorded: MUST NOT claim "basic recon complete", MUST NOT enter workflow 2+ deep conclusions.

Prefer running the bundled script directly:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "sample.exe"
```

If only the manual minimum is needed:

```powershell
rabin2 -I sample.exe
rabin2 -z sample.exe
rabin2 -i sample.exe
rabin2 -E sample.exe
```

Watch:

- File format, bitness, arch, platform
- Entry address
- Suspicious strings: URL, paths, errors, registry, command-line args
- Imported functions: network, file, crypto, process injection, registry ops (**MUST land Evidence; see hard gate above**)

## Workflow 2: Interactive function analysis

```powershell
r2 sample.exe
```

Common after enter:

```text
aaa          # normal auto-analysis
afl          # list functions
iz           # list strings
iS           # list sections
is           # list symbols
s entry0     # jump to entry
pdf          # disassemble current function
VV           # visual mode (if the terminal fits)
q            # quit
```

Notes:

- Prefer `aaa` by default; do not start with heavier `aaaa`
- If the sample is large or analysis is slow, analyze near the entry only, then expand by hand

## Workflow 3: Locate main / key logic

```text
afl~main
afl~sym.
iz~http
iz~error
axt <addr>
```

Approach:

- Start from `main`, entry, and string refs
- Use `axt` to see who references a string or address
- After finding the xref, `s <addr>`, then `pdf`

## Workflow 4: Hex and memory view

```text
px 64        # 64 bytes hex from current address
pd 20        # disassemble 20 instructions
psz          # read string at current address
pxa          # friendlier hex view
```

## Workflow 5: Binary patch

Use only when the user explicitly asks to modify the file:

```powershell
r2 -w sample.exe
```

Then for example:

```text
s 0x401000
wa nop
wa jmp 0x401050
wq
```

Common write ops:

- `wa <asm>`: write assembly
- `wx <hex>`: write raw bytes
- `wq`: write and quit

Back up the original file before edits. If the user did not mention backup, remind at least once.

## Workflow 6: Non-interactive automation

For one-shot output:

```powershell
r2 -A -q -c "afl;iz;ii;q" sample.exe
```

Common flags:

- `-A`: auto-analyze on start
- `-q`: quiet
- `-c`: command string

If there are many commands, order them for readability; do not pack an unmaintainable mega-string.

Prefer the bundled recon script as a base, then decide whether to add custom commands.

## Common subtools

### `rabin2`

Static info extract:

```powershell
rabin2 -I sample.exe   # basic info
rabin2 -S sample.exe   # sections
rabin2 -s sample.exe   # symbols
rabin2 -i sample.exe   # imports
rabin2 -E sample.exe   # exports
rabin2 -z sample.exe   # strings
rabin2 -zz sample.exe  # more detailed strings
```

### `rasm2`

Fast assemble/disassemble:

```powershell
rasm2 -d "9090"
rasm2 -a x86 -b 64 "xor eax, eax"
```

### `radiff2`

Compare two binaries:

```powershell
radiff2 old.exe new.exe
radiff2 -C old.exe new.exe
```

### `rahash2`

Hash:

```powershell
rahash2 -a md5 sample.exe
rahash2 -a sha256 sample.exe
```

### `rax2`

Radix and encoding convert:

```powershell
rax2 0x401000
rax2 4198400
rax2 -s hello
```

## Recommended analysis order

On an unknown sample, do this order:

1. `rabin2 -I` for format, arch, entry
2. `rabin2 -z` for strings
3. `rabin2 -i` for imports — **MUST + Evidence (hard gate; see workflow 1)**
4. Enter `r2` only if interactive analysis is needed (only after step 3 Evidence is on disk)
5. `aaa` first, then `afl` / `iz` / `pdf`
6. Locate key functions via string refs, import calls, and entry flow

This order is low-noise and builds direction quickly. Step 3 is not an optional optimization; it is the hard gate before deep dive.

## Windows notes

- Paths with spaces MUST be quoted correctly
- If the current terminal cannot find `r2`, PATH may have just updated; open a new terminal and retry
- Some samples need admin to read; do not elevate by default unless the user explicitly needs it
- Before dynamic debug of a suspicious sample, confirm user intent to avoid mishandling

## Output style

When the user wants actual analysis, not only commands:

- Give a recon summary first
- Then list key evidence: strings, imports, functions, addresses
- End with next-step advice or continued deep analysis

Do not list commands without explaining why.

## Typical request examples

### Example 1: Analyze an exe

User: `help me see what this exe does, radare2 is fine`

Handling:

1. Start with `rabin2 -I/-z/-i`
2. Decide whether to enter `r2`
3. Deep-dive entry and key string refs with `aaa`, `afl`, `pdf`

### Example 2: Find where a string is called

User: `which function triggers this error string`

Handling:

1. Find the string address with `iz~keyword`
2. Find refs with `axt <addr>`
3. Jump to the xref `s <addr>` then `pdf`

### Example 3: Change a jump

User: `change this jne to je`

Handling:

1. Confirm the target address first
2. Explicitly state write mode will be used
3. Use `wa je <target>` or `wx` directly
4. Disassemble again after the edit to verify

## Practices to avoid

- Do not treat `radare2` as a one-command `aaa` tool
- Do not open the user's file in write mode without stating the risk
- Do not conclude before basic recon
- **MUST NOT skip the import-table check** (`rabin2 -i` / recon imports): MUST NOT proceed until Evidence is written; if the user asks to redo imports, MUST NOT switch to other steps
- **MUST NOT grind statically after IAT repair fails**: record `E-iat-repair-fail` then go dynamic; MUST NOT use only ImportREC on 64-bit samples
- Do not route web JS reverse here; that is `reverse-engineering` scope

## References

- Command cheatsheet: `references/cheatsheet.md`
- Standard recon script: `scripts/recon.ps1`

## radare2-skills ecosystem

The radare2-skills project (radareorg/radare2-skills) provides a fuller ecosystem of tools and workflows:

- **r2xsql**: SQL query of binary imports / strings / functions
- **r2mcp / r2http**: MCP tools and HTTP stateful command channel
- **radius2**: symbolic execution, symbolic dynamic analysis
- **r2pm**: plugin management, extensions
- **decompiler plugins**: radare2 plugin mechanism

**Usage strategy**:
- When the user mentions `r2xsql`, `r2mcp`, `r2http`, `radius2`, `r2pm`, `rabin2`, `rasm2`, `radiff2`, `rahash2`, `rax2`, prefer routing to this skill (radare2/SKILL.md)
- These tools are ecosystem accelerators only; they **cannot bypass**: auth gate, `tool-index` check, Evidence import, write-mode confirm
- Give minimal reproducible command examples:
  - `r2xsql -s <file> -q "SELECT ..."`
  - `curl.exe -sS --data-binary 'aaa' http://127.0.0.1:9393/cmd`
  - `radius2 -p <binary> ...`
  - `r2pm -ci <plugin>`

This skill keeps the original hard gates and evidence-chain integrity. Do not skip any auth or Evidence step.

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Upstream fallback**: `ida-reverse/` (upgrade to IDA when decompile/pseudocode is needed)
**Downstream exits**:
- Need dynamic analysis → `reverse-engineering/tools-dynamic.md` (Frida/GDB)
- Need deep decompile → `ida-reverse/`
- After PAT finds interesting strings and xrefs are needed → `ida-reverse/` (IDA xrefs are stronger)

**Peer modules**: `ida-reverse/` (complement: r2 recon is fast, IDA decompile is deep)

## On-Demand Bootstrap

This skill's entry scripts are wired to the unified bootstrap system. Missing radare2 does not fail immediately; install is attempted automatically.

### Automation bounds

| Tool | Auto-install | Method | Notes |
|------|-----------|---------|------|
| r2 | ✓ | GitHub Release ZIP (w64) | Download and extract to `%USERPROFILE%\Tools\radare2\` |
| rabin2 | ✓ | Same (in the radare2 release) | — |
| rasm2 | ✓ | Same | — |
| radiff2 | ✓ | Same | — |
| rahash2 | ✓ | Same | — |
| rax2 | ✓ | Same | — |

### Bootstrap triggers

- `scripts/recon.ps1`: missing `rabin2` or `r2` → auto-call `bootstrap-reverse.ps1`

### On bootstrap failure

If auto-install fails (no network, GitHub API rate limit, etc.), the script throws a clear error plus a manual install link.

Manual install: download `radare2-*-w64.zip` from https://github.com/radareorg/radare2/releases, extract to `%USERPROFILE%\Tools\radare2\`, and ensure the `bin\` dir is on PATH.


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Was the import-table check run and written as Evidence (E-imports / E-triage-imports or .NET equivalent)? DLL/SYS includes E-exports?
- [ ] If IAT repair failed, was E-iat-repair-fail recorded and dynamic used? Did redo requests return to the same step?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES checklist items?
