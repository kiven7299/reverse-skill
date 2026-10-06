# Security / RE / pentest technical document templates

Templates for reverse engineering, pentest, vuln analysis, and similar security work. After a task, the AI MUST create a document in the user's project directory and fill the matching template.

---

## 0. Evidence Chain (all security reports MUST include)

> Full contract: `skills/ops/evidence-finding-path.md`
> Case dir: `work/<case>/` (`case-init.ps1`)

Report body **MUST** include these sections (may merge into "core findings"; fields MUST NOT be omitted):

### 0.1 Scope summary
- Link `scope.md`: `auth` / `in_scope` / `network_profile`
- No scope → MUST NOT claim the task done

### 0.2 Evidence
At least 1 row: `E-id` / `source_ref` / `repro_command` / `content_hash|n/a`

### 0.3 Findings
Each: `F-id` / `severity|n/a_re` / `evidence_ids` / `confidence` / `location` / `status`

### 0.4 Path
At least 1 `P-id`: `path_type=attack|callflow|solve`; steps may attach E/F

### 0.5 Timeline summary
Link `timeline.md` or embed 3–10 key append records

---

---

## 0.6 Vendor structure overlay (professional vendor report structure)

> Full rules: `references/vendor-report-rules.md` (Issue #65)
> **MUST** read and select flavor when emitting a formal security report; **structure only; forbid copying vendor prose/IOC instances**.

| Flavor / Overlay | When | Skeleton in one line |
|------------------|------|------------|
| `malware` | Clear malware sample / common trojan / white-plus-black | Huorong-style: overview→flow→sample analysis→IR→IOC |
| `apt` | APT/campaign/multi-stage chain | Kaspersky-style: summary→infection chain→investigation→Interesting findings→tech analysis→detect/mitigate→IOC |
| `flavor = null` | Ordinary RE/pentest/CTF/JS signing | This section's task template + applicable Base elements |
| thin `vuln` | Vuln/patch/CVE technical analysis (explicit) | Overview→impact/repro→crash and patch analysis→mitigation |

**Base elements (G1–G7)**: G1 exec summary MUST · G2 Scope MUST · G3 E/F/P MUST · G4 IOC MUST only for `malware`/`apt` · G5 recs MUST for `malware`/`apt`/`vuln` · G6 appendix SHOULD · G7 ATT&CK MUST for `apt`

Flavor and section order follow `vendor-report-rules.md`. On conflict with §0.1–0.5, **Evidence contract wins**.

## 1. Reverse-engineering report template

```markdown
# [Target name] reverse-engineering report

> Analysis date: YYYY-MM-DD
> Analyst: [AI / human]
> Toolchain: [jadx / IDA / radare2 / Frida / ...]

## 1. Target overview

| Attribute | Value |
|------|---|
| Filename | |
| File type | APK / ELF / PE / Mach-O / ... |
| Size | |
| MD5 | |
| SHA256 | |
| Package/entry | |

## 2. Analysis goals

<!-- Core questions this RE pass must answer -->

## 3. Static analysis

### 3.1 Basics
<!-- Arch, compiler, protections, string traits -->

### 3.1.1 Import table / deps (binary MUST)
<!-- Write E-imports / E-triage-imports summary; log Evidence even on failure; no skip -->

### 3.2 Key functions/classes
<!-- Located logic + code snippets -->

### 3.3 Crypto/signing algorithms
<!-- If crypto: algorithm, key origin, param construction -->

## 4. Dynamic analysis

### 4.1 Hook log
<!-- Frida / xposed / other hook targets and results -->

### 4.2 Runtime behavior
<!-- Network, files, process behavior -->

## 5. Core findings

<!-- Numbered key conclusions -->

1. ...
2. ...
3. ...

## 6. Reproduction steps

<!-- A third party can reproduce the analysis -->

```bash
# Key commands
```

## 7. Open issues

<!-- Unresolved points -->

## 8. Attachments

<!-- hook scripts, decrypt code, screenshots -->
```

---

---

## 1b. Malware / APT report (vendor flavor)

When the task is malware analysis, a virus report, or APT/campaign analysis, **do not** ship only the RE skeleton above. Ordinary RE keeps the original template and does not auto-pick a vendor flavor:

1. Read `vendor-report-rules.md` and pick `malware` or `apt`
2. Emit in that section order
3. Still **MUST** include §0 Evidence chain; `malware` / `apt` flavor also **MUST** include an IOC table
4. Binary static analysis **MUST** include import-table Evidence (same hard gate as radare2/ida/malware)

## 1c. 漏洞技术分析 (thin `vuln` overlay)

When the task is **OS/component vuln, patch diff, CVE technical analysis**, or the user explicitly asks for a "vuln technical analysis report":

1. Read `vendor-report-rules.md` §3b; use thin `vuln` section order (**not** full malware/apt flavor)
2. **MUST** include: impact, in-scope repro or explicit n/a, crash/root-cause or patch-diff Evidence, mitigation/patch recs
3. **MUST** include §0 Evidence→Finding→Path
4. **MUST NOT** expand PoC on unauthorized targets, or copy external weaponized exploit details

## 2. Pentest report template

```markdown
# [Target] pentest report

> Test date: YYYY-MM-DD
> Scope: [URL / IP / app name]
> Authorization: [granted / CTF / lab]

## 1. Executive summary

<!-- One paragraph: what was tested, what was found, risk level -->

## 2. Test scope

| Item | Detail |
|------|------|
| Target | |
| Test type | black-box / gray-box / white-box |
| Window | |
| Tools | |

## 3. Finding summary

| # | Vuln name | Risk | Status |
|---|---------|---------|------|
| 1 | | high/medium/low/info | verified/pending |

## 4. Finding details

### 4.1 [Vuln name]

**Risk**: high / medium / low

**Description**:

**Impact**:

**Reproduction**:

1. ...
2. ...
3. ...

**Evidence**:

```
<!-- request/response/screenshot/payload -->
```

**Remediation**:

## 5. Attack path

<!-- If a full chain exists, draw it -->

```
entry → recon → exploit → privilege escalation → objective
```

## 6. Tools and environment

| Tool | Version | Use |
|------|------|------|
| | | |

## 7. Remediation summary

| Priority | Rec |
|--------|------|
| P0 | |
| P1 | |
| P2 | |

## 8. Appendix

<!-- Full payloads, scripts, configs -->
```

---

## 3. CTF Writeup template

```markdown
# [Contest] - [Challenge] Writeup

> Category: Web / Reverse / Pwn / Crypto / Misc / Forensics
> Difficulty: Easy / Medium / Hard
> Points: N pts
> Solve time:

## Challenge statement

<!-- Original description -->

## Approach

### Step 1: Recon
<!-- What was observed -->

### Step 2: Vuln / foothold
<!-- Key insight -->

### Step 3: Exploit
<!-- How it was used -->

## Key code/Payload

```python
# exploit code
```

## Flag

```
flag{...}
```

## Landmines

<!-- Dead ends -->

## Takeaways

<!-- Concepts for later review -->
```

---

## 4. JS/Web signing RE report template

```markdown
# [Site/app] signing-parameter reverse report

> Analysis date: YYYY-MM-DD
> Target API: [URL]
> Sign field: [field name]

## 1. Target request

```http
POST /api/xxx HTTP/1.1
Host: example.com

param1=xxx&sign=<target field>
```

## 2. Location process

### 2.1 Breakpoint/Hook method
<!-- How the sign generator was found -->

### 2.2 Call stack
<!-- Key call chain -->

## 3. Algorithm recovery

### 3.1 Algorithm type
<!-- HMAC-SHA256 / AES / custom / ... -->

### 3.2 Param construction
<!-- Fields in the sign, sort rules, separators -->

### 3.3 Key origin
<!-- Hardcoded / API response / timestamp-derived / ... -->

## 4. Local reproduction

```javascript
// Node.js reproduction
```

## 5. Verification

<!-- Generated sign vs live request -->

## 6. Anti-bot / risk-control notes

<!-- Rate limits, device fingerprint, env checks -->
```

---

## 5. Document output rules

### Location

- Default: **user current project directory** (not the skill-pack directory)
- Filename: `YYYY-MM-DD_[type]-[target-short]-report.md`
- If the user project has `docs/`, prefer `docs/`

### When

The AI auto-invokes this skill when:

1. RE task finished with core conclusions
2. Pentest finished with verified vulns
3. CTF challenge solved with flag
4. User explicitly asks for "a report/document"

### Quality

- All code blocks MUST be runnable or have clear context
- No placeholder/TODO (if incomplete, mark "to fill" and why)
- Key findings MUST have evidence (command output, screenshot description, snippets)
- Reproduction steps MUST let a third party reproduce independently
