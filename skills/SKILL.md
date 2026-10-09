---
name: reverse-skill-router
description: Routes reverse engineering, exploitation, penetration testing, malware, mobile, firmware, browser automation, documentation, and security tasks to the appropriate specialist skill. Use when a task spans modules or the correct reverse-skill entrypoint is unclear.
---
# Reverse Engineering Skills Master Control

This directory holds reverse-engineering skill modules. Each subdirectory is independent and contains a `SKILL.md` describing when to use it, the toolchain, and the workflow.

Machine tool paths for this fork: `../TOOLS.md`. Generated index: `tool-index.md`.

## CRITICAL: routing execution contract (run immediately)

After reading this file, do not reply "read/understood" only. Execute in order:

1. `NOW`: run the platform-native router (Windows `scripts/master-route.ps1`; Linux/macOS/Kali `scripts/master-route.sh`). PRIMARY comes from `config/routing.json`. On hard cases, read the `routing.md` three-axis appendix.
2. `NOW`: platform-native `case-init` lands `work/<case>/scope.md` in the current analysis project. **No target ACT until auth is granted.** Local offline samples use the `offline-sample` preset + an explicit sample. Force MUST NOT bypass the hard gate.
3. `ACT`: open PRIMARY `SKILL.md` and execute ACTION REQUIRED.
4. `NEXT`: tool paths only from `TOOLS.md` then `tool-index.md`. Missing tools → platform-native bootstrap (manifest only).
5. Conclusions use Evidence→Finding→Path. Report/journal is SHOULD unless the user asked for a deliverable.

**Identity**: see `ops/IDENTITY.md` (light router pack + tool bootstrap + journal; **not** a Z3r0-style platform).

If routing cannot hit, research the method online and propose a new skill. MUST NOT force-fit a mismatched module.

## Directive levels (RFC 2119)

- `MUST`: required. Violation fails the task.
- `MUST NOT`: forbidden. Violation is a security breach.
- `SHOULD`: do it by default; skip only with a reason.
- `MAY`: optional.

## Current modules

| Module | Directory | When |
|---|---|---|
| **Generic reverse** | `reverse-engineering/` | GDB / Frida / angr / Unicorn / Qiling / anti-analysis / multi-language RE / CTF pattern library |
| **APK reverse** | `apk-reverse/` | Android APK unpack, jadx decompile, smali edit, Frida hook, rebuild/sign/install |
| **.NET / C# reverse** | `dotnet-reverse/` | Managed PE, dnSpyEx + de4dot deobfuscation (ConfuserEx/SmartAssembly/Babel), IL patch, Sharp* red-team tool analysis, dnSpy MCP |
| **IDA Pro reverse** | `ida-reverse/` | IDA Pro MCP HTTP server (72 tools): decompile, disassemble, data-flow, xrefs |
| **Frontend JS reverse** | `js-reverse/` | Browser signing location, encrypted params, runtime sampling, Node env replay; prefer existing `js-reverse_*`; stronger browser/CDP/Hook surface via jshookmcp only after that MCP is downloaded/registered/enabled |
| **radare2 analysis** | `radare2/` | CLI binary recon, disassembly, patch: r2 / rabin2 / rasm2 / radiff2 |
| **CTF entry** | `ctf-sandbox/` | Single PRIMARY; downstream still lives in sidecar `../CTF-Sandbox-Orchestrator/` |
| **Technical docs** | `docs-generator/` | After a task, generate reverse reports, pentest reports, CTF writeups, signing-reverse reports |
| **Evidence-graph review** | `case-review/` | Validate scope, Evidence→Finding→Path traceability, workitems, timeline, artifact hashes |
| **Browser and desktop automation** | `browser-automation/` | Browser (Playwright) + Windows desktop apps (OpenReverse UIA/CUA) + network observation |
| **Cross-version symbol migration** | `binary-diff/` | Migrate old symbols onto a new build, PDB-less recovery, bulk rename after an update |
| **N-day patch-diff → exploit** | `patch-diff-exploit/` | Locate the bug from a vendor patch, write PoC, N-day weaponize (vs binary-diff: this skill is attack-side) |
| **RE → exploit chain** | `pwn-chain/` | From reverse to a working exploit: stack/heap/kernel pwn, pwntools, libc-database, CTF-to-remote stability |
| **Firmware pentest chain** | `firmware-pentest/` | OWASP FSTM nine stages: extract → EMBA → Firmadyne/QEMU → AFL++ fuzz → device exploit |
| **EDR-bypass reverse** | `edr-bypass-re/` | Reverse EDR hook tables/ETW/AMSI → direct syscall / Hell's Gate / hardware BP / call-stack spoof |
| **Pentest toolchain** | `pentest-tools/` | Nmap/Nuclei/SQLMap/FFUF/Hashcat/Pentest Swarm and 20+ tools, MCP-exposed |
| **Diagrams** | `diagram-generator/` | Mermaid/Graphviz/PlantUML from natural language (attack paths, data flow, architecture, state machines) |
| **Attack-chain orchestration** | `attack-chain/` | Multi-stage attack-path planning; full pentest, HW exercise, external-to-DC starts here |
| **LLM/AI security** | `llm-security/` | OWASP LLM + ASI Top 10: prompt injection, tool abuse, memory poison, agent hijack, system-prompt extract, **agent compliance engineering** |
| **API security** | `api-security/` | REST/GraphQL/WebSocket: BOLA/IDOR, JWT/OAuth, 10-phase method |
| **Supply-chain security** | `supply-chain-security/` | SBOM/SCA/CI-CD: dep scan, container security, build integrity, vuln reachability |
| **Mobile reverse** | `mobile-reverse/` | Android + iOS: Frida/Objection, SSL pinning/root/jailbreak bypass, OWASP MASTG |
| **Native shield instrumentation** | `native-instrumentation/` | bShield/Xq dynamic signing; SSL pinning bypass on Conscrypt; signer reuse (harness + live in-process ptrace); no-Frida native C path |
| **Malware analysis** | `malware-analysis/` | Six-stage sample analysis, YARA/Sigma, anti-analysis, sandbox orchestration |
| **DSL VM reverse** | `reverse-engineering/dsl-vm-reverse/` | JS custom ISA VM (IIFE + switch-case opcode); risk-control / captcha engines |
| **Ops contracts** | `ops/` | Scope / evidence chain / roles / timeline / identity / skill supply-chain |
| **Community skill map** | `references/community-security-skills.md` | External security-skill index and borrow rules (no blind install) |
| **Skill supply chain** | `ops/skill-supply-chain.md` | External skill/MCP install latch (AST10 subset) |
| **RE stage gates** | `reverse-engineering/references/re-agent-workflow.md` | triage→static→dynamic→synthesis |
| **Authorized recon pipeline** | `pentest-tools/references/recon-pipeline.md` | scope gate + hit ≠ validated |
| **Protocol reverse** | `protocol-reverse/` | Custom binary protocol / Protobuf / gRPC / PCAP frame layout |
| **Ghidra reverse** | `ghidra-reverse/` | Open-source decompile, headless, Ghidra MCP (main entry when IDA is absent) |
| **Binary Ninja reverse** | `binary-ninja-reverse/` | HLIL/MLIL/LLIL, Python API, optional community MCP/localhost HTTP |
| **Cloud / container / K8s** | `cloud-k8s/` | IMDS/IAM, container escape surface, Kubernetes RBAC |
| **Windows / AD** | `windows-ad/` | Kerberos, AD CS, BloodHound, relay and domain paths |
| **Digital forensics** | `digital-forensics/` | Memory/disk timeline, PCAP attribution, IR preservation |
| **Code audit / SAST** | `code-audit/` | Semgrep/CodeQL, white-box, dangerous APIs and authz review |
| **Threat intel / OSINT** | `threat-intelligence/` | Public-source IOC enrichment, campaign correlation, independent corroboration |
| **Threat hunting** | `threat-hunting/` | Hypothesis-driven hunting, Sigma detection engineering, blue-team validation |
| **OT / ICS** | `ot-ics/` | Purdue zoning, PLC/SCADA, passive-first assessment |
| **Wi-Fi / wireless** | `wifi-wireless/` | Authorized wireless assessment, handshake/PMKID, lab rules |
| **Browser extension reverse** | `browser-extension-reverse/` | Chrome/Firefox extensions, MV3 worker, permission surface |
| **macOS / Mach-O** | `macos-reverse/` | Signing, ObjC/Swift, LaunchAgent, macOS samples |
| **Thick client** | `thick-client/` | Desktop C/S, local storage, IPC, update channel |
| **Go / Rust reverse** | `go-rust-reverse/` | Stripped Go/Rust, pclntab, panic strings |
| **Hardware debug** | `hardware-security/` | UART/JTAG/SWD, read-only extract, firmware handoff |
| **Database security** | `database-security/` | MySQL/PG/MSSQL/Mongo/Redis exposure and config |
| **Email security** | `email-security/` | Phishing teardown, SPF/DKIM/DMARC, BEC |
| **Federated identity** | `identity-federation/` | SAML/OIDC/OAuth SSO flows and mismatches |
| **RF / SDR** | `radio-sdr/` | Authorized RF research, receive-only by default |

## Unified entry

On reverse, CTF, capture, frontend signing, APK rebuild, or binary-analysis tasks, enter in this order:

1. Platform-native router (Windows `scripts/master-route.ps1`; Linux/macOS/Kali `scripts/master-route.sh`) → PRIMARY (`config/routing.json`)
2. Platform-native `case-init` → `scope.md`
3. Open PRIMARY `SKILL.md`
4. Hard cases: `routing.md`. Need host paths: `TOOLS.md` then `tool-index.md`

## Working method

Combine modules as needed:

1. **Got a target** → file type first, then the matching analyzer
2. **Quick wins** → strings / rabin2 -z / ltrace for direct clues
3. **Go deep** → decompile → IDA; dynamic hook → Frida; symbolic → angr
4. **If one path dies, switch** → static fails → dynamic; Java fails → `.so`; page watch is not enough → breakpoints

## Next-step menu pattern

A child skill `MUST` offer 3–6 numbered options only at a **genuine decision boundary** (two or more materially different, evidence-supported branches, and the user's choice changes the next action). If the next step is uniquely decided by a gate / Evidence, `MUST` continue and, per `ops/timeline-workitem.md`, record only `decision_delta` + `carry_forward_refs`. `MUST NOT` re-emit unchanged route/scope/auth/context just to manufacture a menu.

Format:

- Number each option 1–6
- Each option is a concrete executable action (not an abstract direction)
- Include at least one "export report / write writeup" option
- Include at least one "go deeper" or "switch method" option
- Include a "stop / pause / ask something else" exit when needed

Example:

```
## Suggested next step (pick a number)

1. Deep-decompile sub_140001000 and recover the algorithm
2. Frida-hook to validate the parameter hypothesis
3. Export currently named functions as symbol-migration YAML
4. Generate the current-stage analysis report
5. Switch to radare2 for a light recon comparison
6. Pause; I will confirm the evidence so far
```

## This catalog grows

When a new subdirectory appears, read its `SKILL.md`.

When adding a skill, follow `CONTRIBUTING.md` so that:

- the routing matrix still splits correctly
- bootstrap can fill dependencies
- tool-index reflects the new tool

## Related resources

- This host may also run **anything-analyzer** (port 23816) MCP for browser automation, HTTP capture, and AI analysis
- `TOOLS.md` is the portable path map for this fork; `tool-index.md` records availability, real paths, versions, and script refs
- Package-root `README.md` covers install and client attach for Claude Code, Codex CLI, and other code AI clients

## On-demand bootstrap

When a workflow is missing a tool, do not fail immediately. Call platform-native bootstrap:

Windows:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "<skill-root>\scripts\bootstrap-reverse.ps1" -Capability @('tool-name') -StartServices
```

Linux / macOS:

```bash
bash <skill-root>/scripts/bootstrap-reverse.sh tool-name --start-services
```

Kali:

```bash
bash <package-root>/kali/scripts/bootstrap-reverse.sh tool-name --start-services
```

Supported capabilities (see `scripts/bootstrap-manifest.json`): jadx, apktool, jeb-pro, binaryninja, frida, frida-ps, idalib-mcp, reqable-mcp, jshookmcp, xquik-mcp, anything-analyzer, idapro, r2, rabin2, adb, agent-browser, ghidra-mcp, seclists, proxycat, burpsuite-mcp, nmap, pentestswarm, binwalk, yara, pwntools, bkcrack

> JEB Pro is registered as a **manual licensed install**: bootstrap only prints guidance and MUST NOT download or bypass a commercial license. Reqable MCP only registers a pinned official runtime; the user still installs the Reqable desktop client.
>
> Tools not in the manifest (unblob/EMBA, …) `MUST` follow manual install steps in the skill doc. Do not pretend they bootstrap.

Bootstrap refreshes `tool-index` when it finishes. Prefer `TOOLS.md` paths over a new download.

## Precedent files

Before any reverse/pentest action, MUST read in order:

| Order | File | When |
|---|---|---|
| **#1** | `ops/scope-contract.md` + `case-init.ps1` | Executable auth gate. `precedent-auth.md` does not write granted |
| **#2** | `field-journal/precedent-reverse.md` or `precedent-pentest.md` | On demand — load only when the AI is hesitating |

**#1 first, #2 lazy.**

## Auto-evolution

After each reverse/pentest task, write experience back to `field-journal/`. See the "hard checklist after task completion" in `RULES.md`.

- Template: `field-journal/_template.md`
- Index: `field-journal/_index.md`
- Precedents: `field-journal/precedent-auth.md` → `precedent-reverse.md` → `precedent-pentest.md`
- Before a new task, check the index and precedents and reuse what exists

## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I finish three-axis routing (target type + user intent + toolchain)?
- [ ] After a routing hit, did I read the target skill's SKILL.md?
- [ ] On a miss, did I propose a new skill instead of force-fitting?
- [ ] Did I use real tool paths from `TOOLS.md` / `tool-index`?
