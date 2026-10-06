# reverse-skill overview

A **skill router** for AI coding clients. It classifies a security or reverse-engineering task, picks one PRIMARY playbook, then runs real tools. It is not a single-tool installer and not a red-team operating system.

This fork: https://github.com/kiven7299/reverse-skill  
Upstream: https://github.com/zhaoxuya520/reverse-skill  
Machine paths: [`TOOLS.md`](../TOOLS.md). Sync: [`FORK.md`](../FORK.md), [`UPSTREAM-SYNC.md`](../UPSTREAM-SYNC.md).

Agents start at [`README_AI.md`](../README_AI.md) and [`AGENTS.md`](../AGENTS.md). Humans start here, then [`README.md`](../README.md).

## What it is

When an agent sees an APK, binary, encrypted frontend param, CTF challenge, or authorized pentest target, this pack:

1. routes by target type, user intent, and toolchain (`skills/config/routing.json`);
2. lands a case (`work/<case>/scope.md`) and blocks ACT until `auth.status=granted`;
3. opens the PRIMARY `SKILL.md` and runs ACTION REQUIRED;
4. resolves tools from `TOOLS.md` then generated `skills/tool-index.md`;
5. records Evidence → Finding → Path, then a report / field-journal.

```text
User task
  → RULES.md / AGENTS.md
  → MASTER-ROUTING / master-route (PRIMARY)
  → case-init / scope.md
  → PRIMARY SKILL.md
  → TOOLS.md → tool-index → bootstrap (manifest only)
  → timeline + Evidence→Finding→Path → report
```

It exists because general agents guess commands, mix APK/ELF/JS/PCAP playbooks, scatter tool paths across machines, and repeat the same mistakes.

## Status

| Item | Value |
|---|---|
| Version | 1.0.1 |
| Routing rules | 44 (R0–R45), SSoT `skills/config/routing.json` |
| Regression | 175+ hint → PRIMARY cases |
| Tracked modules | 45 `SKILL.md` entries |
| Fallback | R0 = `reverse-engineering/` |
| Clients | Client-neutral (Kilo, Claude Code, Codex, Cursor, OpenCode, …) |

Change routing only in `routing.json`. `verify-routing-coherence.ps1` keeps `MASTER-ROUTING.md` in lockstep.

## How to run a task

Open this repository as the workspace. Do not copy `skills/` into another client folder.

Windows:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File skills\scripts\master-route.ps1 -Hint "<task>"
powershell -NoProfile -ExecutionPolicy Bypass -File skills\scripts\case-init.ps1 -Hint "<task>" -CaseName "my-case"
```

Linux / macOS / Kali:

```bash
bash skills/scripts/master-route.sh --hint "<task>"
bash skills/scripts/case-init.sh --hint "<task>" --case-name "my-case"
```

Then open the printed PRIMARY `SKILL.md`. Offline local samples: `-Preset offline-sample` / `--preset offline-sample` plus an explicit sample path. `-Force` never bypasses the scope gate.

Tool paths: edit [`TOOLS.md`](../TOOLS.md) when porting machines, then:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File skills\scripts\refresh-tool-index.ps1
```

## Skill catalog (45)

Grouped by job. Router priority is **not** this list order. See [Priority](#priority-high--low).

### Reverse engineering

| Module | Use when |
|---|---|
| `apk-reverse/` | Android APK unpack, jadx, smali, Frida, rebuild/sign |
| `mobile-reverse/` | Android + iOS, Objection, pinning / root / jailbreak checks |
| `js-reverse/` | Frontend signing, encrypted params, CDP / jshook |
| `reverse-engineering/dsl-vm-reverse/` | Custom JS opcode VM / risk-control engine |
| `dotnet-reverse/` | Managed PE, dnSpy / de4dot, IL patch |
| `ida-reverse/` | IDA Pro deep decompile / MCP |
| `ghidra-reverse/` | Ghidra GUI / headless / MCP when IDA is absent |
| `binary-ninja-reverse/` | HLIL / MLIL / LLIL |
| `radare2/` | CLI r2 / rabin2 / rasm2 / radiff2 |
| `go-rust-reverse/` | Stripped Go / Rust, pclntab |
| `macos-reverse/` | Mach-O, codesign, ObjC / Swift |
| `browser-extension-reverse/` | Chrome / Firefox extensions, MV3 |
| `protocol-reverse/` | Custom binary protocol, Protobuf / gRPC, PCAP frames |
| `malware-analysis/` | Sample analysis, YARA / Sigma, anti-analysis |
| `reverse-engineering/` | Generic RE, anti-debug, OLLVM, unknown binary (R0) |

### Reverse → exploit

| Module | Use when |
|---|---|
| `binary-diff/` | Cross-version **symbol** migration (not exploit writing) |
| `patch-diff-exploit/` | N-day from vendor patch → PoC |
| `pwn-chain/` | Stack / heap / kernel pwn to a **working** exploit |
| `edr-bypass-re/` | Reverse EDR hooks / ETW / AMSI (authorized) |
| `firmware-pentest/` | Firmware / IoT, OWASP FSTM extract → emulate → exploit |
| `hardware-security/` | UART / JTAG / SWD, read-only extract |

### Pentest / identity / infra

| Module | Use when |
|---|---|
| `attack-chain/` | Multi-stage lead (recon → access → lateral). Dispatch, do not do every stage here |
| `pentest-tools/` | Nmap, Nuclei, SQLMap, FFUF, Hashcat, Burp MCP |
| `pentest-tools/src-hunter/` | SRC / bug-bounty hunting playbooks (needs granted scope) |
| `api-security/` | REST / GraphQL / WebSocket, BOLA / JWT |
| `windows-ad/` | Kerberos, AD CS, BloodHound, relay |
| `identity-federation/` | SAML / OIDC / OAuth SSO mismatches |
| `cloud-k8s/` | IMDS / IAM, container escape, K8s RBAC |
| `database-security/` | MySQL / PG / MSSQL / Mongo / Redis |
| `supply-chain-security/` | SBOM / SCA / CI-CD / image integrity |
| `thick-client/` | Desktop C/S, local storage, IPC, update channel |
| `ot-ics/` | Purdue zoning, PLC / SCADA, **passive-first** |
| `wifi-wireless/` | Authorized Wi-Fi, handshake / PMKID, lab only |
| `radio-sdr/` | RF / SDR, receive-only by default |

### Blue / intel / AI

| Module | Use when |
|---|---|
| `digital-forensics/` | Memory / disk timeline, PCAP, IR preservation |
| `threat-hunting/` | Hypothesis hunt, Sigma / YARA detection engineering |
| `threat-intelligence/` | Public OSINT / IOC enrichment |
| `email-security/` | Phishing teardown, SPF / DKIM / DMARC, BEC |
| `code-audit/` | Semgrep / CodeQL, white-box |
| `llm-security/` | Prompt injection, tool abuse, agent hijack |

### CTF and deliverables

| Module | Use when |
|---|---|
| `ctf-sandbox/` | Single PRIMARY latch. Downstream is `CTF-Sandbox-Orchestrator/` (~40 competition skills). Do not dump those into `routing.json` |
| `ops/` | Scope, roles, evidence chain, timeline, identity, skill supply-chain |
| `case-review/` | Read-only Evidence-graph check before report handoff |
| `docs-generator/` | Reverse / pentest / CTF reports |
| `diagram-generator/` | Mermaid / Graphviz / PlantUML |
| `browser-automation/` | Playwright + Windows desktop UIA |

## Priority (high → low)

Must match `routing.json` `priority`. First strong keyword hit with highest score wins.

| ID | Condition | PRIMARY |
|---|---|---|
| R4 | DSL VM / custom opcode VM | `dsl-vm-reverse/` |
| R1 | APK / smali / jadx / apktool | `apk-reverse/` |
| R2 | IPA / iOS / Objection / MobSF | `mobile-reverse/` |
| R3 | JS signing / frontend crypto / CDP | `js-reverse/` |
| R30 | Browser extension | `browser-extension-reverse/` |
| R31 | macOS / Mach-O | `macos-reverse/` |
| R33 | Go / Rust binary | `go-rust-reverse/` |
| R5 | .NET / dnSpy / de4dot | `dotnet-reverse/` |
| R9 | Malware / YARA / sandbox | `malware-analysis/` |
| R21 | Protocol / Protobuf / PCAP protocol | `protocol-reverse/` |
| R22 | Ghidra | `ghidra-reverse/` |
| R45 | Binary Ninja / HLIL / MLIL | `binary-ninja-reverse/` |
| R6 | IDA / deep decompile | `ida-reverse/` |
| R7 | radare2 / r2 | `radare2/` |
| R8 | Firmware / binwalk / IoT | `firmware-pentest/` |
| R34 | UART / JTAG | `hardware-security/` |
| R28 | OT / ICS | `ot-ics/` |
| R17 | pwn / ROP / heap | `pwn-chain/` |
| R16 | N-day / patch diff | `patch-diff-exploit/` |
| R18 | EDR / syscall bypass | `edr-bypass-re/` |
| R24 | Windows / AD / Kerberos | `windows-ad/` |
| R37 | SAML / OIDC | `identity-federation/` |
| R23 | Cloud / K8s | `cloud-k8s/` |
| R35 | Database | `database-security/` |
| R25 | Forensics | `digital-forensics/` |
| R44 | OSINT / threat intel | `threat-intelligence/` |
| R36 | Email / phishing | `email-security/` |
| R29 | Wi-Fi | `wifi-wireless/` |
| R38 | RF / SDR | `radio-sdr/` |
| R32 | Thick client | `thick-client/` |
| R26 | SAST / Semgrep | `code-audit/` |
| R27 | Threat hunting | `threat-hunting/` |
| R10 | Full attack chain | `attack-chain/` |
| R11 | Nmap / Nuclei / pentest tools | `pentest-tools/` |
| R12 | API / GraphQL / BOLA | `api-security/` |
| R13 | SBOM / supply chain | `supply-chain-security/` |
| R14 | LLM / prompt injection | `llm-security/` |
| R15 | Bindiff / symbol migration | `binary-diff/` |
| R19 | Browser / desktop automation | `browser-automation/` |
| R40 | Case / Evidence review | `case-review/` |
| R20 | Report / writeup | `docs-generator/` |
| R39 | Diagrams | `diagram-generator/` |
| R41 | CTF / AWD / range | `ctf-sandbox/` |
| R0 | Generic / unknown binary | `reverse-engineering/` |

No strong hit → R0, then read `skills/routing.md`.

Chinese **hint keywords** stay in `routing.json` so Chinese user phrasing still matches. Playbooks are English.

## Ops contracts

| File | Role |
|---|---|
| `skills/ops/IDENTITY.md` | Router pack, not a Z3r0 platform |
| `skills/ops/scope-contract.md` | Auth + `network_profile` before ACT |
| `skills/ops/evidence-finding-path.md` | Evidence → Finding → Path |
| `skills/ops/role-map.md` | lead / cie / cpe / cre in one session |
| `skills/ops/timeline-workitem.md` | Append-only timeline, coverage |
| `skills/ops/sandbox-profile.md` | Bootstrap vs manual tools |
| `skills/ops/skill-supply-chain.md` | External skill / MCP install gate |

`network_profile`: `offline` | `lab_only` | `authorized_target_only` | `unrestricted_lab`.

## This machine

Prefer paths in [`TOOLS.md`](../TOOLS.md). Already mapped: jadx CLI/GUI, apktool, adb, zipalign, apksigner, IDA 9.1, Ghidra 10.4, radare2, Frida, nmap, nuclei, Burp, Reqable, dnSpy, ILSpy.

Not mapped / missing: YARA, binwalk, Binary Ninja, JEB Pro, SecLists. Do not install cracks. Bootstrap only names in `skills/scripts/bootstrap-manifest.json`.

## What this is not

- Not a required Postgres / React / Docker control plane
- Not 800 vendor-copied micro-skills
- Not permission to scan the internet. Naming a host is not `auth.status=granted`
- MCP (IDA, Burp, jadx, Reqable, jshook) is opt-in. No silent global client config

## Related docs

| Doc | Purpose |
|---|---|
| [README.md](../README.md) | Project entry |
| [README_AI.md](../README_AI.md) | Agent bootstrap |
| [AGENTS.md](../AGENTS.md) | Client-neutral repo rules |
| [RULES.md](../RULES.md) | Behavior chain |
| [skills/MASTER-ROUTING.md](../skills/MASTER-ROUTING.md) | PRIMARY ladder |
| [skills/routing.md](../skills/routing.md) | Full matrix |
| [skills/INDEX.md](../skills/INDEX.md) | Generated module index |
| [docs/QUICKSTART.md](QUICKSTART.md) | Clone, open workspace, client notes |
| [docs/PLATFORMS.md](PLATFORMS.md) | OS matrix |
| [TOOLS.md](../TOOLS.md) | Local tool paths |

## License and use

MIT for this pack. `CTF-Sandbox-Orchestrator/` is GPLv3. Third-party tools keep their own licenses.

Authorized research, CTF, teaching, and systems you own or have written permission to test. Unauthorized access, scanning, or exploitation is forbidden.
