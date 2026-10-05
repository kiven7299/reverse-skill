---
name: dotnet-reverse
description: .NET / C# binary reverse. Use when the target is a .NET assembly (PE header has CLR, managed .exe/.dll), a C# compile product (including NativeAOT), red-team Sharp* tools (Rubeus / SharpHound / etc.), .NET obfuscators (ConfuserEx / SmartAssembly / Babel / Eazfuscator), or a .NET loader / info-stealer / packed malware. Prefer dnSpyEx + de4dot; when AI must drive the UI, pair with dnSpy MCP. Not for pure native binaries (use reverse-engineering / ida-reverse).
license: MIT
compatibility: Requires a filesystem-based code agent or CLI with shell access, Windows host preferred (dnSpyEx is a Windows GUI); Linux/macOS can use ILSpy/de4dot CLI + mono/dotnet runtime.
allowed-tools: Bash Read Write Edit Glob Grep Task WebFetch WebSearch
metadata:
  user-invocable: "false"
---

# .NET / C# reverse work spec

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: use DIE/`file`/CLR header to confirm the target is managed .NET (else SWITCH to `ida-reverse/` / `reverse-engineering/`)
2. `NOW`: if likely packed/obfuscated → `de4dot` first, emit `*-clean.exe`, keep the original sample
3. `NEXT`: dnSpyEx (or dnSpy MCP / `ilspycmd`) static: browse C# + **IL view** for key predicates
4. `ACT`: dynamic-debug when plaintext/C2 is needed; when logic must change, **IL patch** beats C# recompile
5. at stage end, give the user a 3–6 item next-step menu (include export report)

## Scope

Prefer this skill when the task is:

- Identify and reverse .NET / C# compile products (managed PE / .exe / .dll)
- Analyze red-team Sharp* toolchains (Rubeus, SharpHound, SharpShell, etc.)
- Deobfuscate ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor and similar packers
- Reverse .NET loader / info-stealer / RAT decrypt and C2 logic
- Patch a C# program (change predicates, constants, keygen)
- Analyze the Mono/Unity managed layer before IL2CPP (note: after IL2CPP the result is native; use `reverse-engineering/` + seed-014)

If the target is a pure native binary (C/C++/Go/Rust, no CLR), use `reverse-engineering/`, `ida-reverse/`, or `radare2/` instead.

## Core principles

- **Identify before acting**: confirm it is managed .NET (PE header CLR + `#~` / `#Strings` streams + mscoree `_CorExeMain`) before choosing dnSpy over IDA
- **IL over C#**: dnSpyEx's C# decompiler loses/distorts information (compiler-generated state machines, async/await, yield). Key predicates and patches MUST switch to the **IL editor**; C# view is for fast browsing only
- **de4dot first**: on obfuscators run `de4dot` once before static analysis, or strings/control flow stay garbled
- **MCP pairing**: if dnSpy MCP (`dnspy_*` tools) is registered, prefer the MCP surface for decompile / IL inspection instead of flipping the GUI
- **Evidence output**: deobfuscated artifacts, extracted config/C2/key, and patch diffs MUST land on disk

## Toolchain mapping

| Capability | Preferred | Notes |
|------|------|------|
| Decompile + debug + patch | **dnSpyEx** | Primary; the only GUI with an IL editor; old dnSpy is unmaintained, use the Ex fork |
| Light CLI / headless decompile | **ILSpy** (`ilspycmd`) | Batch, scripted, Linux/macOS |
| Deobfuscate | **de4dot** | Default unpack for ConfuserEx family, SmartAssembly, and other mainstream packers |
| Obfuscator ID | **Detect It Easy (DIE)** / **file** | Identify packer type before choosing de4dot args |
| Programmatic IL | **dnlib** | C# scripts for batch metadata edits / string decryptors |
| AI-driven ops | **dnSpy MCP** | `dnspy_decompile` / `dnspy_inspect_il` and similar |

> Prerequisite: on Windows install dnSpyEx + de4dot (choco or release); on Linux/macOS use `ilspycmd` + `dotnet runtime`. See the install matrix in `references/sharp-tools.md`.

## Six-phase workflow

### 1. Identify (.NET)

Confirm the target is managed. Do not analyze a native PE as .NET:

```powershell
# Windows
file target.exe                       # "PE32 executable ... for MS Windows" is not enough
# Key: is there a CLR
powershell -c "[System.Reflection.AssemblyName]::GetAssemblyName('target.exe')"
# or
dnSpyEx drag-and-drop — if it opens, it is managed

# Generic
strings target.exe | grep -iE "mscoree|_CorExeMain|mscorlib|System\\."
```

**.NET identification marks:**
- PE header `Data Directory[14]` (CLR Runtime Header) nonzero
- `mscoree.dll` import / `_CorExeMain` entry
- `#~`, `#Strings`, `#US`, `#GUID`, `#Blob` metadata streams
- `mscorlib` / `System.Private.CoreLib` strings

**NativeAOT exception:** compiled to native, no CLR header, but has `System.Private.CoreLib` strings and reconstructed type metadata — use `reverse-engineering/` (IDA/r2). This skill only flags identification.

### 2. Detect (obfuscator)

```powershell
# DIE fast ID
diec target.exe                        # Detect It Easy CLI
# or drag into dnSpyEx and look for mass garbled class names / control-flow morph
```

Common obfuscators → unpack strategy (see `references/obfuscators.md`):

| Obfuscator | Traits | de4dot handling |
|--------|------|------------|
| ConfuserEx (1.0.0 / 2.x) | `<module>` anti-tamper, control-flow morph, string encrypt | `de4dot target.exe` usually auto-detects |
| SmartAssembly | `circular`/`string encoding`, resource compress | `de4dot target.exe` |
| Babel.NET | method-body encrypt, control flow | `de4dot target.exe` |
| Eazfuscator.NET | string/resource encrypt | `de4dot`; some versions need manual work |
| .NET Reactor | anti-tamper + necrobit | `de4dot`; new versions MAY fail and need manual work |

### 3. Deobfuscate

```powershell
# de4dot default auto-detects most packers
de4dot target.exe -o target-clean.exe

# Specify type (when auto-detect fails)
de4dot --type cfze target.exe          # ConfuserEx
de4dot --type sa target.exe            # SmartAssembly

# Multi-layer obfuscation / de4dot reports unknown
de4dot --detect target.exe             # see what it identified
# MAY need to patch anti-tamper first, then de4dot (see references/obfuscators.md)
```

Output: `target-clean.exe` for later analysis. **Keep the original sample** for comparison.

### 4. Static Analyze

Load the unpacked sample in dnSpyEx:

- **C# view**: fast browse of class structure, method signatures, strings (for location)
- **IL view**: key predicates, crypto logic, state machines MUST be read in IL (right-click → Edit IL or IL view)
- Find entry: `Main` / `Startup` / module initializer (`Module .cctor`)
- Find key logic: search `flag`, `password`, `verify`, `check`, `encrypt`, `http`, `Config`

```text
Locate string → reverse xref → find the method that uses it → IL view for predicate logic
```

### 5. Dynamic (debug)

dnSpyEx debugger: attach / start debug, breakpoint key methods, observe at runtime:
- Plaintext strings after decrypt (many obfuscators decrypt strings only at runtime)
- C2 addresses, config decrypt results
- Exception-driven control flow (anti-debug often hides the real path in `try/catch`)

> .NET dynamic debug is much friendlier than native — object values and string contents are visible. Prefer dynamic over grinding static.

### 6. Patch (as needed)

```text
dnSpyEx → right-click method → Edit Method (C#) or Edit IL
  - Change predicate: ldc.i4.0 → ldc.i4.1 (false→true)
  - Change constant: edit string/number directly
  - Drop check: nop the whole block
File → Save Module → replace the original file
```

**IL patch reliability > C# patch**: C# recompile MAY fail (missing refs, bad syntax); IL edits almost never distort. See `references/common-workflow.md`.

## Trigger-scene routing

Enter this skill when the user says:
- ".NET / C# binary reverse" / "decompile a C# program"
- "dnSpy analysis" / "dnSpyEx patch"
- "ConfuserEx / SmartAssembly / Babel deobfuscate / unpack"
- "Sharp* tool analysis" (Rubeus / SharpHound / SharpShell)
- ".NET malware / loader / info-stealer reverse"
- "C# program patch / keygen / change a predicate"

## When to switch out

- IL2CPP-compiled Unity game → `reverse-engineering/` + `seed-014_unity-il2cpp-reverse.md` (IL2CPP is native; not dnSpy)
- NativeAOT product → `reverse-engineering/` (same, native)
- Pure native PE (no CLR) → `reverse-engineering/` / `ida-reverse/`
- Need symbol/function bulk migrate to another version → `binary-diff/`
- Need attack-path / call-chain diagrams → `diagram-generator/`

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Downstream exits**:
- IL2CPP / NativeAOT (native) → `reverse-engineering/`
- Deep native .so/.dll segment analysis → `ida-reverse/` / `radare2/`
- Need AI to drive dnSpy directly → register and pair dnSpy MCP (see `references/sharp-tools.md`)

**Peer modules**:
- `reverse-engineering/languages-compiled.md` (.NET intro points here)
- `apk-reverse/` (Xamarin/MAUI Android reverse MAY switch back here for the C# layer)

## References

- [references/obfuscators.md](references/obfuscators.md) — ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor deobfuscation detail + anti-tamper bypass
- [references/common-workflow.md](references/common-workflow.md) — full workflow, IL patch reliability, string-decryptor extract, state-machine ID
- [references/sharp-tools.md](references/sharp-tools.md) — red-team Sharp* tool analysis, install matrix, dnSpy MCP integration, community index

## Task-complete self-check

- [ ] Was CLR / managed identity confirmed (or SWITCH out of this skill)?
- [ ] Was an obfuscated sample de4dot / equivalent unpacked before deep analysis?
- [ ] Was key logic verified in IL view (not C# pseudocode only)?
- [ ] Did artifacts (clean sample / config / patch diff) land on disk and reproduce?
- [ ] Was a next-step menu or report exit provided?
