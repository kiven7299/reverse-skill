# Specialist role → skill map (no multi-agent server)

> Role codes are inspired by the Z3r0 specialist team. **Implementation** is reverse-skill routing plus a handoff protocol, not process orchestration.

## Role table

| Code | Name | Duty | PRIMARY / tool skill |
|---|---|---|---|
| **lead** | Lead | Split work, set scope, stage-gate, roll up the report | `attack-chain/` or current PRIMARY hub; finish → `docs-generator/` |
| **cie** | Collection / intel | Asset discovery, exposure, relationships | `pentest-tools/` (recon); browser → `browser-automation/`; cloud → `cloud-k8s/` |
| **cpe** | Penetration validation | Scan, exploit validation, impact | `pentest-tools/`; API → `api-security/`; AD → `windows-ad/`; wireless → `wifi-wireless/`; DB → `database-security/`; SSO → `identity-federation/`; OT → `ot-ics/` |
| **cre** | Reverse engineering | Binary / firmware / mobile / frontend logic | `ida-reverse/` `ghidra-reverse/` `binary-ninja-reverse/` `radare2/` `apk-reverse/` `mobile-reverse/` `macos-reverse/` `js-reverse/` `browser-extension-reverse/` `dotnet-reverse/` `go-rust-reverse/` `firmware-pentest/` `hardware-security/` `malware-analysis/` `protocol-reverse/` `thick-client/` `reverse-engineering/` |
| **cae** | Code audit | Source / deps / supply chain | `code-audit/` + `supply-chain-security/` |
| **cbe** | Blue team / forensics | Hunt, detect, IR artifacts | `threat-hunting/` `digital-forensics/` |
| **cce** | Crypto | Algorithm / protocol / key misuse | `reverse-engineering` pattern docs |
| **llm** | AI security | Prompt / agent | `llm-security/` |
| **doc** | Documentation | Report / writeup / diagrams | `docs-generator/` + `diagram-generator/` |

## Lead MUST protocol

```text
1. Emit PRIMARY (master-route) + lead_role=lead
2. Write scope.md (ops/scope-contract)
3. Set specialist_roles[] and handoff conditions
4. End of each stage: update timeline + workitems; continue / switch role / report
5. MUST NOT skip scope and jump to cpe against production
```

## Handoff rules

| From → to | Trigger | Deliverable |
|---|---|---|
| lead → cie | need asset surface | scope + known domains/IPs |
| cie → cpe | live surface/service | asset list + ports/URLs |
| cpe → cre | need RE / client logic | sample path + suspicion |
| cre → cpe | recovered protocol / key / check | algorithm notes + repro command |
| any → doc | stage or task complete | Evidence/Finding/Path draft |
| any → lead | blocked / out of scope / path change | timeline note + blocked reason |

## Single-agent usage

Do not spawn six agents:

```text
Same session:
  [lead] plan
  [cie] run recon skill
  [cpe] switch pentest-tools
  …
Prefix output with the role so timeline search works:
  [cpe] nuclei high findings → E-003
```

## Relation to master-route

- `master-route` sets the **PRIMARY skill**
- `role-map` sets **who owns the current stage** (may be written in scope.md)
- Multi-stage tasks often PRIMARY=`attack-chain/`, then lead dispatches

## MUST NOT

- Do not assume a Z3r0 session API exists
- Do not start extra scans against unauthorized targets "for the role"
