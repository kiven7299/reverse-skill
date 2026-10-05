---
name: binary-diff
description: |
  Cross-version symbol migration and binary diff. Use when old-version symbols/RE results must be moved to a new version.
  Scenarios: kernel missing PDB inferred from old symbols; bulk function-name migration after an update; locating new offsets after an app update.
  Core method: LLM structured comparison with programmatic I/O; very low cost (~200 functions ~1 CNY).
  Trigger keywords: symbol migration, bindiff, cross-version, missing PDB, function offset migration, binary diff, version compare.
---

# Cross-version symbol migration (Binary Diff)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

## Scope

Use this skill when the task is one of:

1. **Kernel/driver missing PDB** — old ntoskrnl.exe symbols exist; new PDB was pulled by Microsoft; infer non-exported addresses in the new build
2. **Post-update symbol migration** — prior RE of a program; the program updated; migrate results in bulk instead of reversing again
3. **Protection-mechanism update** — complete old RE; quickly locate the same function's new offset
4. **Any binary-compare case with old symbols + new unsigned binary**

### Split with other skills

| Scenario | Use |
|------|--------|
| Reverse a binary from scratch | `ida-reverse/` or `radare2/` |
| Old results, migrate to new version | **this skill** |
| Two unrelated binaries | BinDiff / Diaphora (traditional tools) |

### Core advantage

Versus traditional options:

| Approach | Cost for 200 functions | Time | Accuracy |
|------|--------------|------|--------|
| Manual two-IDA comparison | free, expensive in labor | hours | high |
| BinDiff auto-match | free | fast | medium (fails on large structural change) |
| Full Agent (CC/Codex) | 50-100 CNY | slow | high |
| **this skill (LLM batch compare)** | **~1 CNY** | **~10 s/function** | **high** |

## Core principle

```text
Old function (named)               Same function in new build (unnamed)
    ↓                              ↓
Export disasm + decompile          Export disasm + decompile
    ↓                              ↓
    └──────── LLM structured compare ────────┘
                    ↓
         Output YAML (symbol map)
                    ↓
         Parse programmatically → apply in bulk to new IDB
```

Key points:
- prompt is a fixed template, filled programmatically
- I/O formats are fixed, parsed programmatically
- LLM only does "look at two code snippets, find correspondences"
- time and token cost stay very low

## Prompt template

### Standard compare Prompt

```text
I have disassembly outputs and procedure code of the same function.

This is the function for reference:

**Disassembly for Reference**
```c
{disasm_for_reference}
```

**Procedure code for Reference**
```c
{procedure_for_reference}
```

This is the function you need to reverse-engineering:

**Disassembly to reverse-engineering**
```c
{disasm_code}
```

**Procedure code to reverse-engineering**
```c
{procedure}
```

What you need to do is to collect all references to "{symbol_name_list}" in the function you need to reverse-engineering and output those references as YAML.

Example:
```yaml
found_vcall: # This is for indirect call to virtual function or virtual function pointer fetching.
  - insn_va: '0x180777700' # Always be the instruction with displacement offset
    insn_disasm: call [rax+68h] # Always be the instruction with displacement offset
    vfunc_offset: '0x68'
    func_name: ILoopMode_OnLoopActivate
  - insn_va: '0x180777778' # Always be the instruction with displacement offset
    insn_disasm: mov rax, [rax+80h] # Always be the instruction with displacement offset
    vfunc_offset: '0x80'
    func_name: INetworkMessages_GetNetworkGroupCount

found_call: # This is for direct call to non-virtual regular function.
  - insn_va: '0x180888800'
    insn_disasm: call sub_180999900
    func_name: CLoopMode_RegisterEventMapInternal
  - insn_va: '0x180888880'
    insn_disasm: call sub_180555500
    func_name: CLoopMode_SetSystemState

found_funcptr: # This is for non-virtual regular function pointer.
  - insn_va: '0x180666600' # Must load/reference the function pointer target address
    insn_disasm: lea rdx, sub_15BC910 # Must load/reference the function pointer target address
    funcptr_name: CLoopMode_OnClientPollNetworking

found_gv: # This is for reference to global variable.
  - insn_va: '0x180444400'
    insn_disasm: mov rcx, cs:qword_180666600 # Must load/reference the global variable
    gv_name: g_pNetworkMessages
  - insn_va: '0x180333300'
    insn_disasm: lea rax, unk_180222200 # Must load/reference the global variable
    gv_name: s_EventManager

found_struct_offset: # This is for reference to struct offset. NOTE THAT virtual function pointer should not be here! virtual function pointer should ALWAYS be in found_vcall !
  - insn_va: '0x1801BA12A' # Always be the instruction with displacement offset
    insn_disasm: mov rcx, [r14+58h] # Always be the instruction with displacement offset
    offset: '0x58'
    size: 8
    struct_name: CResourceService
    member_name: m_pEntitySystem
```

If nothing found, output an empty YAML. DO NOT output anything other than the desired YAML. DO NOT collect unrelated symbols.
```

### Variable notes

| Variable | Source | Notes |
|------|------|------|
| `{disasm_for_reference}` | old IDA export | named disassembly |
| `{procedure_for_reference}` | old IDA export | named decompile |
| `{disasm_code}` | new IDA export | unnamed disassembly |
| `{procedure}` | new IDA export | unnamed decompile |
| `{symbol_name_list}` | extracted from old | symbols to locate in the new build |

## Workflow

### Full flow

```text
Step 1: Prepare data
  - Load old binary in IDA (PDB/symbols present)
  - Load new binary in IDA (no symbols)
  - Find shared anchor functions (exports, string xrefs, etc.)

Step 2: Batch export
  - From old: disasm + decompile of anchors (with names)
  - From new: disasm + decompile of the same anchors (unnamed)

Step 3: LLM compare
  - Fill the prompt template
  - Call LLM API (prefer: deepseek for volume/cost; huge functions switch to gpt)
  - Parse returned YAML

Step 4: Apply results
  - Apply YAML symbol map to the new IDB in bulk
  - Bulk rename via idapro_rename or IDAPython

Step 5: Iterate
  - Round-1 migrated functions become new anchors
  - Enter those functions, compare inner calls
  - Repeat until all target functions are covered
```

### Anchor selection

| Anchor type | Reliability | Notes |
|---------|--------|------|
| Exported function | highest | name stable, address may change |
| String xref | high | string content stable, xref site may change |
| Constant/magic | medium | signature value stable |
| Code pattern | medium | similar structure, all addresses changed |

### Batch-processing tips

- Compare 1 function at a time (avoid context blow-up)
- Medium functions (<200 lines) use deepseek
- Huge functions (>500 lines) switch to gpt-4o or claude
- Concurrent calls for speed (10-20 concurrent)
- Cache results; avoid repeat calls

## Output format

### Five YAML symbol types

| Type | Meaning | Key fields |
|------|------|---------|
| `found_vcall` | virtual call (indirect call) | `vfunc_offset`, `func_name` |
| `found_call` | direct call | `insn_va`, `func_name` |
| `found_funcptr` | function-pointer xref | `insn_va`, `funcptr_name` |
| `found_gv` | global-variable xref | `insn_va`, `gv_name` |
| `found_struct_offset` | struct-offset xref | `offset`, `struct_name`, `member_name` |

### Apply actions after parse

```text
found_call → idapro_rename(addr=call_target, name=func_name)
found_vcall → idapro_set_comments(addr=insn_va, comment="vcall: {func_name} @ +{offset}")
found_funcptr → idapro_rename(addr=funcptr_target, name=funcptr_name)
found_gv → idapro_rename(addr=gv_addr, name=gv_name)
found_struct_offset → idapro_set_comments(addr=insn_va, comment="{struct_name}.{member_name}")
```

## Typical scenarios

### Scenario 1: ntoskrnl.exe missing PDB

```text
Have: ntoskrnl.exe 10.0.26100.2000 + full PDB
Target: ntoskrnl.exe 10.0.26100.2605 (PDB pulled)
Need: new address of PspSetCreateProcessNotifyRoutine

Steps:
1. Load both versions in IDA
2. Find export PsSetCreateProcessNotifyRoutine (both versions)
3. Old build calls PspSetCreateProcessNotifyRoutine (named)
4. New build calls sub_140822108 (unnamed)
5. LLM sees: sub_140822108 = PspSetCreateProcessNotifyRoutine
6. Apply in bulk
```

### Scenario 2: migrate after app update

```text
Have: full RE of target.exe v1.0 (200+ named functions)
Target: target.exe v1.1 (all symbols gone)
Need: bulk-migrate 200 function names

Steps:
1. Export disasm+decompile of all named functions from old
2. Find matching anchors in new via exports/strings
3. Batch LLM compare
4. Parse YAML, bulk rename
5. Iterate deeper
```

## LLM choice

| Model | Fit | Cost | Speed |
|------|---------|------|------|
| DeepSeek V3 | small/medium functions (<200 lines), batch | very low | fast |
| GPT-4o | huge functions, complex CFG | medium | fast |
| Claude Sonnet | medium/large functions, needs reasoning | medium | fast |
| Claude Opus | extremely complex functions, deep understanding | high | slow |

Recommended: default DeepSeek; auto-upgrade on context overflow or bad results.

## Notes

- **Do not dump the whole binary to the LLM** — one function per compare
- **Anchors MUST be reliable** — a wrong anchor wastes everything after
- **Spot-check results** — LLM is not 100% accurate; verify critical symbols
- **Cache intermediates** — avoid wasting tokens on repeats
- **Watch context limits** — huge functions (>1000 lines of disasm) need split or a large-context model

---

## On-Demand Bootstrap

### Tool dependencies

| Tool | Purpose | Auto-install |
|------|------|-----------|
| IDA Pro | export disasm/decompile | ✗ (commercial) |
| Python | scripts, API calls | ✓ |
| PyYAML | parse LLM YAML | ✓ (`pip install pyyaml`) |
| LLM API | run compares | needs API key |

### Notes

This skill's core does not need heavy installs. It mainly needs:
- IDA Pro already present (managed by `ida-reverse/` skill)
- Python + requests/httpx (API calls)
- an LLM API endpoint

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Trigger**: old symbols/RE results exist; need migration to a new version
**Downstream exit**:
- need to open the binary first → `ida-reverse/`
- need fast recon of version delta → `radare2/`

**Peer modules**: `ida-reverse/` (export and apply symbols both go through IDA)

## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES Checklist items?
