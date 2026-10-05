# Optional sandbox tool profile (vs bootstrap-manifest)

> Z3r0 default images ship many tools. reverse-skill **does not bundle an image**. Use this table as coverage contrast plus optional Docker advice.

Machine paths for this fork live in `../../TOOLS.md`. Prefer those over guessing.

## Capabilities this pack can auto-bootstrap

Source: `skills/scripts/bootstrap-manifest.json` (file wins):

| Capability | Typical scene |
|---|---|
| jadx / apktool / adb / frida / frida-ps | Android |
| r2 / rabin2 | binary CLI |
| idalib-mcp / idapro | IDA MCP |
| jeb-pro | commercial Android / ARM decompiler (manual license) |
| jshookmcp / reqable-mcp / anything-analyzer / agent-browser | Web / JS / capture / browser |
| ghidra-mcp | Ghidra |
| nmap / seclists / proxycat / burpsuite-mcp / pentestswarm | pentest |
| binwalk / pwntools / yara | firmware / pwn / malware |

```powershell
powershell -File skills\scripts\bootstrap-reverse.ps1 -Capability @('jadx','nmap','yara') -StartServices
powershell -File skills\scripts\refresh-tool-index.ps1
```

On this fork, check `TOOLS.md` first. Do not reinstall a tool that already has a path there.

## Common in Z3r0 sandboxes but not auto-installed here

| Tool | reverse-skill policy |
|---|---|
| subfinder / amass / httpx / ffuf / nuclei / sqlmap | documented install / Kali / external MCP; **do not pretend bootstrap has them**. This machine already has amass + nuclei under `D:\Tools` — see `TOOLS.md` |
| full Ghidra GUI | ghidra-mcp capability + manual plugin steps; Ghidra 10.4 is already on this machine |
| gdb / pwndbg | platform docs, manual; pwntools may bootstrap |
| hydra / hashcat | manual or Kali |
| JEB Pro | user-owned license, manual install; third-party MCP bridges need supply-chain review first |
| Reqable desktop | user install; `reqable-mcp` only registers a pinned official MCP runtime. This machine already has Reqable under `D:\Tools\reqable` |
| SecLists | seclists capability |

## Recommended light Docker ops profile (optional, not a dependency)

Only when the user **already** has Docker and an authorized lab:

```text
min: nmap + nuclei + sqlmap container or a pentestMCP-class image
mobile: jadx + apktool + frida on the host
RE: host IDA/r2 + TOOLS.md / tool-index
```

**MUST NOT** require a Z3r0 install to use reverse-skill.

## network_profile coupling

Scans inside a sandbox still obey the case `scope.md` `network_profile`:

- `offline` → do not start outbound-scan containers
- `authorized_target_only` → containers may hit in_scope only
