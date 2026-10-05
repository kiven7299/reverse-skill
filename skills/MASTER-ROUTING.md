# reverse-skill PRIMARY fast path

> `scripts/master-route.ps1` and `scripts/master-route.sh` MUST keep the same routing contract. Platform changes the entry, not the routing semantics.

Tool paths: `TOOLS.md` (this fork) then generated `tool-index.md`. Do not guess.

## Execution contract

```text
1. Route before acting
2. Emit PRIMARY path + one-line rationale
3. case-init / scope.md (ops/scope-contract) — no target ACT until auth.status=granted
4. Assign lead + specialist roles (ops/role-map)
5. Open PRIMARY SKILL.md → ACTION REQUIRED immediately
6. Tool paths only from TOOLS.md / tool-index; missing → bootstrap (manifest capabilities only)
7. Append timeline / workitems; conclusions use Evidence→Finding→Path
8. No hit → read routing.md full table or propose a new skill
```

### Windows

```powershell
powershell -File skills\scripts\master-route.ps1 -Hint "<user task>"
# Default writes the current project's work/master-route-<ts>/route-scope.md; when calling from another dir, set project root
powershell -File skills\scripts\master-route.ps1 -Hint "<user task>" -ProjectRoot "C:\path\to\analysis-project"
powershell -File skills\scripts\case-init.ps1 -Hint "<user task>" -CaseName "my-case"
# Case defaults to current project's work/<case>/; -PackageRoot remains compatible, -ProjectRoot wins
powershell -File skills\scripts\case-init.ps1 -Hint "<user task>" -CaseName "my-case" -ProjectRoot "C:\path\to\analysis-project"
# One-shot ACT-ready (auth + target + network profile):
powershell -File skills\scripts\case-init.ps1 -Hint "<task>" -CaseName "my-case" -AuthGranted -TargetUrl "https://target/" -NetworkProfile authorized_target_only
# Local offline sample:
powershell -File skills\scripts\case-init.ps1 -Hint "offline apk" -CaseName "my-sample" -Preset offline-sample -Sample ".\app.apk"
# Smoke: verify + script parse + routing matrix (includes Chinese Hint)
powershell -File skills\scripts\smoke.ps1
# Light scope gate before ACT (not ready → exit 2; -Force is compatibility only and cannot bypass the hard gate)
powershell -File skills\scripts\case-guard.ps1 -CaseRoot work\my-case
# Append Evidence
powershell -File skills\scripts\append-evidence.ps1 -CaseRoot work\my-case -Id E-001 -Title "..." -ReproCommand "..."
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

### Linux / macOS / Kali

PowerShell is not required for the core route/case path:

```bash
bash skills/scripts/master-route.sh --hint "<user task>"
bash skills/scripts/master-route.sh --hint "<user task>" --project-root "/path/to/analysis-project"
bash skills/scripts/case-init.sh --hint "<user task>" --case-name "my-case"
bash skills/scripts/case-init.sh --hint "<user task>" --case-name "my-case" --project-root "/path/to/analysis-project"
# Local offline sample:
bash skills/scripts/case-init.sh --hint "offline apk" --case-name "my-sample" --preset offline-sample --sample ./app.apk
# Light scope gate before ACT (--force is compatibility only and cannot bypass the hard gate):
bash skills/scripts/case-guard.sh --case-root work/my-sample
# Routing parity:
bash skills/scripts/test-routing.sh
bash skills/scripts/test-bootstrap-manifest.sh
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

## Ops contracts

| Doc | Purpose |
|---|---|
| `ops/IDENTITY.md` | We are a router pack, not a Z3r0 platform |
| `ops/scope-contract.md` | Startup gate |
| `ops/evidence-finding-path.md` | Evidence chain |
| `case-review/SKILL.md` | Evidence-graph review and report handoff |
| `ops/role-map.md` | Role → skill |
| `ops/timeline-workitem.md` | Timeline and coverage |
| `ops/sandbox-profile.md` | Tool coverage |
| `ops/skill-supply-chain.md` | External skill/MCP install gate |
| `references/community-security-skills.md` | Community skill map (borrow, do not vendor) |
| `reverse-engineering/references/re-agent-workflow.md` | RE: triage→static→dynamic→synthesis |
| `pentest-tools/references/recon-pipeline.md` | Authorized recon pipeline + evidence gate |
| `../../TOOLS.md` | Machine tool paths (this fork) |

## Priority (high → low)

> Order MUST match the `priority` array in `config/routing.json`. Change routing only in the JSON, then this table. `verify-routing-coherence.ps1` parses this table.

| ID | Condition | PRIMARY |
|---|---|---|
| **R4** | DSL VM / fireye / custom opcode VM | `reverse-engineering/dsl-vm-reverse/` |
| **R1** | APK / smali / jadx / apktool | `apk-reverse/` |
| **R2** | IPA / iOS / Objection / MobSF / mobile | `mobile-reverse/` |
| **R3** | JS signing / frontend crypto / jshook / CDP | `js-reverse/` |
| **R30** | Browser extension reverse | `browser-extension-reverse/` |
| **R31** | macOS / Mach-O | `macos-reverse/` |
| **R33** | Go / Rust binary | `go-rust-reverse/` |
| **R5** | .NET / dnSpy / de4dot / ConfuserEx | `dotnet-reverse/` |
| **R9** | Malware sample / YARA / sandbox | `malware-analysis/` |
| **R21** | Protocol / Protobuf / PCAP protocol | `protocol-reverse/` |
| **R22** | Ghidra / open-source decompile | `ghidra-reverse/` |
| **R45** | Binary Ninja / Binja / HLIL / MLIL / Binary Ninja MCP | `binary-ninja-reverse/` |
| **R6** | IDA / deep decompile / disassembly | `ida-reverse/` |
| **R7** | radare2 / r2 | `radare2/` |
| **R8** | Firmware / binwalk / IoT / EMBA | `firmware-pentest/` |
| **R34** | Hardware debug port / UART/JTAG | `hardware-security/` |
| **R28** | OT / ICS / industrial control | `ot-ics/` |
| **R17** | pwn / ROP / heap-stack exploit | `pwn-chain/` |
| **R16** | N-day / patch diff | `patch-diff-exploit/` |
| **R18** | EDR / AV bypass / syscall | `edr-bypass-re/` |
| **R24** | Windows / AD / Kerberos / AD CS | `windows-ad/` |
| **R37** | Federated identity SAML/OIDC | `identity-federation/` |
| **R23** | Cloud / container / K8s | `cloud-k8s/` |
| **R35** | Database security | `database-security/` |
| **R25** | Forensics / memory dump / timeline | `digital-forensics/` |
| **R44** | OSINT / threat intel / public X IOC enrichment | `threat-intelligence/` |
| **R36** | Email / phishing analysis | `email-security/` |
| **R29** | Wi-Fi / wireless pentest | `wifi-wireless/` |
| **R38** | RF / SDR research | `radio-sdr/` |
| **R32** | Thick-client security | `thick-client/` |
| **R26** | Code audit / SAST / Semgrep | `code-audit/` |
| **R27** | Threat hunting / detection engineering / blue team | `threat-hunting/` |
| **R10** | Attack chain / red team / lateral / full pentest | `attack-chain/` |
| **R11** | Nmap / Nuclei / SQLMap / SRC / pentest tools | `pentest-tools/` |
| **R12** | API / GraphQL / BOLA / JWT attacks | `api-security/` |
| **R13** | SBOM / Trivy / supply chain | `supply-chain-security/` |
| **R14** | LLM / prompt injection / agent security | `llm-security/` |
| **R15** | bindiff / symbol migration / PDB | `binary-diff/` |
| **R19** | Browser/desktop automation | `browser-automation/` |
| **R40** | Case / Evidence-graph review | `case-review/` |
| **R20** | Report / writeup | `docs-generator/` |
| **R39** | Diagrams / Mermaid / Graphviz / PlantUML / architecture | `diagram-generator/` |
| **R41** | CTF / AWD / range (single entry, do not expand 40 sub-skills) | `ctf-sandbox/` |
| **R0** | Generic reverse / anti-debug / OLLVM / unknown binary | `reverse-engineering/` |

No strong keyword hit → PRIMARY=`R0`, and prompt opening `routing.md` (ambiguity appendix, not a second router).

## Boundaries

| Task | Handling |
|---|---|
| Pure CTF multi-type orchestration | PRIMARY `ctf-sandbox/` → sidecar `../CTF-Sandbox-Orchestrator/` |

## Read order

```text
RULES.md → MASTER-ROUTING.md → PRIMARY SKILL.md
  → (optional) routing.md three-axis / field-journal
  → TOOLS.md → tool-index.md → bootstrap → ACT
```
