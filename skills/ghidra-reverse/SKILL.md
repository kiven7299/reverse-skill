---
name: ghidra-reverse
description: Use for free/open reverse engineering with Ghidra (headless or GUI), including decompile, cross-refs, and optional Ghidra MCP workflows when IDA is unavailable.
---

# Ghidra Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md`
2. `NOW`: confirm Ghidra is required (no IDA / prefer OSS / batch headless)
3. `NEXT`: read `../tool-index.md` look up ghidra / ghidra-mcp paths
4. `NEXT`: missing tool → bootstrap `ghidra-mcp` (if the manifest supports it) or install Ghidra via the manual steps
5. `ACT`: import the sample → auto-analyze → export key-function decompilation

## When to use

- Primary RE entry when there is no IDA license
- Batch headless analysis / decompile in CI
- Ghidra script (Java/Python Jython/PyGhidra) automation
- Pair with ghidriff from `binary-diff` / `patch-diff-exploit`

## Split vs IDA

| Need | Prefer |
|------|------|
| Already have IDA MCP for deep dive | `ida-reverse/` |
| Open source / batch / teaching | **this skill** |
| CLI fast recon only | `radare2/` |

## Workflow

### 1. Project and auto-analysis

```text
□ New Project → Import file → Analyze (default analyzers)
□ Record language/compiler ID result and base address
□ Mark entry, export table, string xrefs
```

### 2. Key functions

```text
□ Reverse-lookup from strings / imported APIs
□ Restore algorithms in the Decompile window
□ Rename functions/vars; write Plate comments
□ If dynamic is needed, hand off to Frida/GDB (reverse-engineering dynamic chapter)
```

### 3. Headless (batch)

```bash
# Example: analyzeHeadless path varies by install; MUST take it from tool-index
analyzeHeadless /path/to/project Proj -import sample.bin -postScript ExportDecomp.py
```

### 4. MCP (if configured)

```text
□ Confirm ghidra MCP port (often 8765; tool-index is authoritative)
□ Pull decompile / xrefs with MCP tools; MUST NOT guess the port
```

## Toolchain

| Tool | Purpose | Bootstrap |
|------|------|------|
| Ghidra | Primary decompiler | Manual release / package manager |
| ghidra-mcp | AI bridge | bootstrap capability name `ghidra-mcp` |
| ghidriff | Patch diff | see `patch-diff-exploit` |

## References

- `references/ghidra-cheatsheet.md`
- `../ida-reverse/` `../radare2/` `../binary-diff/`

## Routing context

**Upstream**: MASTER R22  
**Downstream**: dynamic verify → Frida/GDB; exploit → `pwn-chain`  
**Peer**: `ida-reverse` (commercial deep dive)

## Task-complete self-check

- [ ] Based on real Ghidra/tool-index paths?
- [ ] Function addresses and renames annotated?
- [ ] Reproducible steps present?
- [ ] Checklist / journal?
