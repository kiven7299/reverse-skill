# IDA Pro MCP tool cheat sheet

> ida-pro-mcp 2.x tools by function, with common params and typical usage.
> Server name: `idapro`, tool prefix: `idapro_*`, HTTP mode. Tool count varies by version (~66, including `py_eval`).

---

## Start and session

### Server start

```powershell
# start MCP HTTP server (background silent; healthy → OK:<n>:reuse)
powershell -File "scripts/start.ps1"
# OK:<tool_count> means ready (~66, including py_eval)

# open target (bypass schema check)
powershell -File "scripts/open.ps1" -Path "C:\target.exe"
# output OK:filename:session_id

# large files/GUI: add timeout
powershell -File "scripts/open.ps1" -Path "C:\big.exe" -TimeoutSeconds 600

# skip auto-analysis (fast open)
powershell -File "scripts/open.ps1" -Path "C:\huge.sys" -NoAutoAnalysis
```

### Session tools

| Tool | Use | Example |
|------|------|------|
| `idapro_idb_list()` / HTTP `idb_list` | list all sessions | — |
| `idapro_idb_open()` / HTTP `idb_open` | open database (prefer `open.ps1`) | large files via script |
| `idapro_idb_save(path)` / HTTP `idb_save` | save database | save analysis progress |
| `idapro_idb_current()` | currently bound session (if version provides) | — |
| `idapro_idb_switch(session_id)` | switch session | multi-file compare |
| `idapro_idb_close(session_id)` | close session | free resources |
| `idapro_server_health()` | server health | — |
| `idapro_server_warmup()` | warm subsystems | before first use |

---

## Step 1: global survey

### survey_binary — fast overview

```
idapro_survey_binary(detail_level="minimal")
```

Returns:
- architecture (x86/x64/ARM/MIPS)
- entry point
- function count
- string stats
- segment info
- import classes (crypto/net/file IO/registry)
- high-xref hot functions

**detail_level options**:
- `"minimal"` — fast overview (recommended first)
- `"standard"` — more detail
- `"full"` — complete

### Function list

```
# list all functions (paged)
idapro_list_funcs(queries=[{"offset": 0, "limit": 50}])

# filter by name
idapro_list_funcs(queries=[{"filter": "crypt", "offset": 0, "limit": 20}])
idapro_list_funcs(queries=[{"filter": "main", "offset": 0, "limit": 10}])
```

### Unified query

```
# query imports
idapro_entity_query(kind="imports", filter="Create")

# query strings
idapro_entity_query(kind="strings", filter="http")

# query all named symbols
idapro_entity_query(kind="names", filter="")
```

---

## Decompile and disasm

### Decompile (pseudocode)

```
# by function name
idapro_decompile(addr="main")
idapro_decompile(addr="sub_140001000")

# by address
idapro_decompile(addr="0x140001000")
```

### Disasm

```
# default insn count
idapro_disasm(addr="main")

# specified insn count
idapro_disasm(addr="0x401000", max_instructions=100)
```

### Combined analysis (recommended)

```
# one-shot: pseudocode + strings + constants + callers + callees + basic blocks
idapro_analyze_function(addr="main", include_asm=false)

# include assembly
idapro_analyze_function(addr="sub_401000", include_asm=true)
```

### Function profile

```
# batch function metrics (size, block count, xref count)
idapro_func_profile(queries=["main", "sub_401000", "sub_402000"])
```

---

## Xrefs and call graph

### Who refs the target

```
# who calls a function
idapro_xrefs_to(addrs=["sub_401000"])

# who refs a string/data
idapro_xrefs_to(addrs=["0x404000"])

# batch
idapro_xrefs_to(addrs=["CreateFileW", "ReadFile", "WriteFile"])
```

### Advanced xref query

```
# direction and type
idapro_xref_query(addr="0x401000", direction="to")    # who refs me
idapro_xref_query(addr="0x401000", direction="from")  # whom I ref
```

### Callee list

```
idapro_callees(addrs=["main"])
```

### Call graph

```
# from main, depth 3
idapro_callgraph(roots=["main"], max_depth=3)

# multiple roots
idapro_callgraph(roots=["sub_401000", "sub_402000"], max_depth=2)
```

### Data-flow trace

```
# backward: where did this value come from
idapro_trace_data_flow(addr="0x401050", direction="backward", max_depth=5)

# forward: where does this value go
idapro_trace_data_flow(addr="0x401050", direction="forward", max_depth=5)
```

---

## Search

### String search (regex)

```
# search URL
idapro_find_regex(pattern="https?://", limit=20)

# search file paths
idapro_find_regex(pattern="C:\\\\", limit=20)

# search error messages
idapro_find_regex(pattern="error|fail|invalid", limit=30)

# search key/password related
idapro_find_regex(pattern="key|password|secret|token", limit=20)
```

### Disasm text search

```
# search in disassembly listing
idapro_search_text(pattern="call    sub_")
idapro_search_text(pattern="xor     eax, eax")
```

### Byte-pattern search

```
# exact bytes
idapro_find_bytes(patterns=["48 8B 05"], limit=10)

# with wildcards
idapro_find_bytes(patterns=["48 89 ?? 24 ??"], limit=10)

# multiple patterns
idapro_find_bytes(patterns=["CC CC CC CC", "90 90 90 90"], limit=5)
```

### Advanced search

```
# search immediates
idapro_find(type="immediate", targets=["0xDEADBEEF"])

# search string refs
idapro_find(type="string", targets=["password"])
```

---

## Memory and data reads

### Raw bytes

```
idapro_get_bytes(addrs=[{"addr": "0x401000", "size": 64}])
```

### Strings

```
idapro_get_string(addrs=["0x404000", "0x404100"])
```

### Integers

```
idapro_get_int(queries=[{"addr": "0x405000", "size": 4}])
```

### Globals

```
idapro_get_global_value(queries=["g_flag", "g_key_size"])
```

### Structs

```
idapro_read_struct(queries=[{"addr": "0x405000", "type": "HEADER"}])
```

### Search structs

```
idapro_search_structs(filter="FILE")
```

---

## Mutations

### Comments

```
# single comment
idapro_set_comments(items=[{"addr": "0x401000", "comment": "decrypt function entry"}])

# batch comments
idapro_set_comments(items=[
    {"addr": "0x401000", "comment": "XOR decrypt loop"},
    {"addr": "0x401050", "comment": "key init"},
    {"addr": "0x4010A0", "comment": "result check"}
])

# append (do not overwrite existing)
idapro_append_comments(items=[{"addr": "0x401000", "comment": "note: key length 16"}])
```

### Rename

```
# rename functions
idapro_rename(batch={"func": [
    {"addr": "sub_401000", "name": "decrypt_payload"},
    {"addr": "sub_402000", "name": "verify_license"}
]})

# rename globals
idapro_rename(batch={"global": [
    {"addr": "0x405000", "name": "g_encryption_key"}
]})

# rename locals
idapro_rename(batch={"local": [
    {"func": "decrypt_payload", "old": "v1", "name": "plaintext_buf"}
]})
```

### Patch asm

```
# NOP detection code
idapro_patch_asm(items=[{"addr": "0x401050", "asm": "nop"}])

# change jump
idapro_patch_asm(items=[{"addr": "0x401060", "asm": "jmp 0x401080"}])

# force return true
idapro_patch_asm(items=[
    {"addr": "0x401000", "asm": "mov eax, 1"},
    {"addr": "0x401005", "asm": "ret"}
])
```

### Patch bytes

```
# write bytes directly
idapro_patch(patches=[{"addr": "0x401050", "bytes": "9090909090"}])
```

---

## Type system

### Declare struct

```
idapro_declare_type(decls=[{
    "name": "PacketHeader",
    "decl": "struct PacketHeader { uint32_t magic; uint16_t type; uint16_t length; uint8_t data[0]; };"
}])
```

### Apply type

```
# set function prototype
idapro_set_type(edits=[{
    "addr": "sub_401000",
    "type": "int __fastcall decrypt(void *buf, int size, const char *key)"
}])

# set global type
idapro_set_type(edits=[{
    "addr": "0x405000",
    "type": "PacketHeader"
}])
```

### Infer types

```
idapro_infer_types(addrs=["sub_401000", "sub_402000"])
```

### Query/inspect types

```
idapro_type_query(queries=["Packet"])
idapro_type_inspect(queries=["PacketHeader"])
```

---

## Stack frame

```
# view function stack frames
idapro_stack_frame(addrs=["main", "sub_401000"])

# declare stack var
idapro_declare_stack(items=[{
    "func": "sub_401000",
    "offset": -0x20,
    "name": "local_buf",
    "type": "char [32]"
}])
```

---

## Signature generation

```
# unique byte signature for an address
idapro_make_signature(addrs=["0x401000"])

# signature for whole function
idapro_make_signature_for_function(addrs=["decrypt_payload"])

# signatures for code that refs an address
idapro_find_xref_signatures(addrs=["0x405000"])
```

---

## Base conversion

```
# hex → decimal
idapro_int_convert(inputs=["0x401000"])

# decimal → hex
idapro_int_convert(inputs=["4198400"])

# batch
idapro_int_convert(inputs=["0xDEAD", "0xBEEF", "12345"])
```

> ⚠️ **Always use this tool for base conversion; do not compute by hand.**

---

## Export and scripts

### Export functions

```
# JSON
idapro_export_funcs(addrs=["main", "sub_401000"], format="json")

# C header
idapro_export_funcs(addrs=["main", "sub_401000"], format="c_header")

# prototypes
idapro_export_funcs(addrs=["main", "sub_401000"], format="prototypes")
```

### Run Python

```
# Python in IDA context
idapro_py_eval(code="import idautils; print(list(idautils.Functions())[:10])")

# segment info
idapro_py_eval(code="import idc; print(idc.get_segm_name(0x401000))")

# batch ops
idapro_py_eval(code="import ida_funcs; f=ida_funcs.get_func(0x401000); print(f.size())")
```

---

## Typical analysis flows

### Malware analysis

```text
1. survey_binary → inspect imports (net API? crypto? registry?)
2. find_regex("http|socket|connect") → find net-related strings
3. xrefs_to(net string addr) → find referencing functions
4. decompile(referencing fn) → inspect comms logic
5. trace_data_flow(crypto param, "backward") → trace key origin
6. set_comments + rename → annotate findings
```

### License-check crack

```text
1. find_regex("serial|license|register|valid") → find verify-related strings
2. xrefs_to(verify strings) → locate verify function
3. analyze_function(verify fn) → understand logic
4. callgraph(verify fn, 2) → see call chain
5. patch_asm(conditional-jump addr, "jmp always_pass") → patch
```

### CTF RE

```text
1. survey_binary → confirm arch and entry
2. decompile("main") → inspect main logic
3. find_regex("flag|correct|wrong") → find decision points
4. trace_data_flow(decision point, "backward") → trace input transform
5. Python helper compute/decrypt → get flag
```

### Vulnerability analysis

```text
1. entity_query(kind="imports", filter="strcpy|sprintf|gets") → find dangerous fns
2. xrefs_to(dangerous fn) → find call sites
3. analyze_function(caller) → inspect context
4. stack_frame(fn) → confirm buffer size
5. trace_data_flow(dangerous arg, "backward") → confirm user-controlled
```

---

## Common errors and fixes

| Error | Cause | Fix |
|------|------|------|
| "No database bound" | no file open | run `open.ps1` |
| "Failed to open database" | old DB locked | `open.ps1` auto-falls back to Temp |
| schema check failed | MCP client BUG | use `open.ps1` instead of `idb_open` |
| tool timeout | large file still analyzing | add `-TimeoutSeconds 600` |
| "ERR:timeout" (start.ps1) | server start failed | check Python/idalib-mcp install |
| base-conversion error | hand math wrong | use `idapro_int_convert` |
| function name not found | name imprecise | `list_funcs` + filter first |
