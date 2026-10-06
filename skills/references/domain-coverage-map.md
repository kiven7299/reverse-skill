# Pack domain coverage map (depth first)

> Versus community "hundreds of micro-skills": we cover the main field with **few deep skills + routing + ops**.
> Date: 2026-07-18

## Domain → pack entry

| Domain | PRIMARY / module | Notes |
|----|----------------|------|
| Mobile Android | `apk-reverse/` `mobile-reverse/` | |
| Mobile iOS | `mobile-reverse/` | |
| Binary deep dive | `ida-reverse/` `radare2/` `ghidra-reverse/` | Ghidra = open-source main path |
| General RE / anti-debug / OLLVM | `reverse-engineering/` | |
| .NET | `dotnet-reverse/` | |
| Frontend JS / signing | `js-reverse/` | |
| Browser extensions | `browser-extension-reverse/` | |
| DSL / risk-control VM | `reverse-engineering/dsl-vm-reverse/` | |
| Protocol / PCAP protocol | `protocol-reverse/` | |
| Firmware IoT | `firmware-pentest/` | |
| Malware samples | `malware-analysis/` | |
| Digital forensics / IR | `digital-forensics/` | |
| Threat hunting / blue team | `threat-hunting/` | |
| Pentest tools | `pentest-tools/` (+ src-hunter) | |
| Windows / AD | `windows-ad/` | |
| Cloud / containers / K8s | `cloud-k8s/` | |
| Code audit / SAST | `code-audit/` | |
| Wi-Fi / wireless | `wifi-wireless/` | |
| OT / ICS | `ot-ics/` | Passive first; write-register forbidden by default |
| macOS | `macos-reverse/` | iOS still via mobile-reverse |
| Binary Ninja | `binary-ninja-reverse/` | Commercial GUI/Python API; community MCP only if explicitly enabled, loopback bind by default |
| Thick client | `thick-client/` | |
| Go / Rust binaries | `go-rust-reverse/` | |
| Hardware debug ports | `hardware-security/` | Hand off to firmware-pentest |
| Database | `database-security/` | |
| Email / phishing | `email-security/` | |
| Federated identity SSO | `identity-federation/` | Complements api-security JWT |
| RF / SDR | `radio-sdr/` | RX only by default; not Wi-Fi |
| Multi-stage attack | `attack-chain/` | |
| Pwn | `pwn-chain/` | |
| N-day patch | `patch-diff-exploit/` | |
| EDR research | `edr-bypass-re/` | |
| API | `api-security/` | |
| Supply-chain SBOM | `supply-chain-security/` | |
| LLM/Agent | `llm-security/` | + `ops/skill-supply-chain.md` |
| Browser automation | `browser-automation/` | |
| Reports / diagrams | `docs-generator/` `diagram-generator/` | |
| Symbol migration | `binary-diff/` | |
| Ops contract | `ops/` | **Distinctive** |
| CTF orchestration | `CTF-Sandbox-Orchestrator/` | |
| Crypto pattern ID | `reverse-engineering` pattern docs | Shared with RE tasks; no separate extension pack |

## Domains we explicitly do not vendor-merge (strategy on routing miss)

| Domain | Strategy |
|----|------|
| Pure game-cheat development | Not a product direction; Unity samples still via `reverse-engineering` + seed-014 |
| Deep auto/aviation certification-grade | External link OK; this pack has RF/OT entry-level only |
| Pure GRC/compliance long-form | Does not replace specialist GRC tools; may cite report templates |
| 800+ ATT&CK micro-skills | This table + optional ATT&CK tags (Finding field) |

## MITRE ATT&CK (optional)

Finding template allows `optional_attack: Txxxx` (see `ops/evidence-finding-path.md`). Full ATT&CK engine is **not** required.
