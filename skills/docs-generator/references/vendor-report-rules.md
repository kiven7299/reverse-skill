# Vendor Report Rules (professional vendor report structure overlay)

> Issue #65 question 2.
> **Structure and writing rules only. Forbid copying any vendor report body, figures, real IOC instances, or long passages.**
> This file is an **overlay**: it does not replace `security-report-templates.md` task templates, and does not weaken §0 Evidence→Finding→Path.

Structure references (public samples, skeleton only):

| Flavor | Primary reference | When |
|--------|--------|------|
| `malware` | Huorong malware/technical analysis reports | Clear common trojan, white-plus-black, phishing drop, malware sample |
| `apt` | Kaspersky Securelist / APT campaign reports (e.g. MATA) | APT, group campaign, multi-stage infection chain, industry targeting |

Principle: **few templates, high quality** — 仅 2 个厂商全文 flavor (`malware` / `apt`) + Base elements + **optional thin overlay** (e.g. `vuln` technical analysis). Ordinary RE, pentest, CTF, and JS reports keep the task template and MUST NOT default-masquerade as malware reports; `vuln` 不是第 3 个默认全文 flavor.

---

## 0. When to enable

When `docs-generator` emits a **security** report (RE / malware / pentest wrap-up / user asks for "professional report" / "vendor style") it **MUST** read this file. Pick a vendor flavor only when task evidence or the user explicitly supports it; otherwise `flavor = null` and overlay only Base professional elements plus the original task template.

| Signal | Flavor / Overlay |
|------|------------------|
| APT / group / campaign / multi-stage C2 / industry targeting / ICS / spear-phish campaign | `apt` |
| Clear malware sample, trojan, stealer, white-plus-black, fake site | `malware` |
| User explicitly wants vuln/patch/CVE technical analysis, or evidence is OS/component vuln research | `flavor = null` + **thin overlay `vuln`** (see §3b) |
| Ordinary APK/ELF/PE/Mach-O RE, algorithm analysis, firmware, pentest, CTF, JS signing | `flavor = null`; original task template + Base professional minimum |

User explicit "Kaspersky/APT", "Huorong/virus report", or "vuln technical analysis" overrides auto-select.
**Forbid** defaulting ordinary malware/APT/ordinary RE into the `vuln` outline.

---

## 1. Base professional elements

Apply Base elements by report type. **MUST** items cannot be omitted. Flavor-specific elements MUST NOT appear on unrelated tasks just to fill a template. If nothing applies, use `n/a` plus reason.

| # | Element | Requirement |
|---|------|------|
| G1 | Exec summary / overview | **MUST**: 3–8 sentences: what was analyzed, worst conclusion, impact, recommended action |
| G2 | Scope and authorization | **MUST**: link case `scope.md` (template §0.1) |
| G3 | Evidence→Finding→Path | **MUST**: `security-report-templates.md` §0 and `skills/ops/evidence-finding-path.md` |
| G4 | IOC table | `malware` / `apt` **MUST**; other tasks only if relevant indicators exist |
| G5 | Recs / response | `malware` / `apt` **MUST**: at least 1 actionable rec; other tasks follow original template |
| G6 | Appendix metadata | **SHOULD**: tools and versions, sample hashes, full repro commands |
| G7 | ATT&CK mapping | **MUST** (under `apt`; `n/a` + reason if none); other tasks **SHOULD** |

### 1.1 IOC table minimum columns

```markdown
| Type | Value | Context | First/last seen | Source evidence / 来源证据 | Confidence |
|------|----|--------|---------------|----------|--------|
| file_sha256 / file_md5 / domain / ip:port / url / mutex / path / registry | … | Where found | YYYY-MM-DD / n/a | E-id | high/med/low |
```

### 1.2 Copyright and security boundary

- Do not paste vendor PDF/web paragraphs or figure captions as your analysis.
- Real tokens, internal URLs, customer IDs → placeholders.
- Unauthorized targets: no directly exploitable attack-step detail (follow case scope / RULES).

---

## 2. Flavor: `malware` (Huorong-style · explicit pick)

**Narrative goal**: in 5 minutes the reader knows "what it is → how it arrived → what the sample does → how to respond → which IOCs".

### 2.1 Recommended section order

```markdown
# [Title: one-line threat characterization]

> Analysis date / party / sample ID (hash)

## 1. Overview
(G1: discovery channel, disguise, core tech, product detection if known — else n/a)

## 2. Attack / infection flow
(Flow: Mermaid or numbered list; Path `path_type=attack`)

## 3. Sample analysis
### 3.1 Provenance
### 3.2 Static analysis
(**MUST** include import table / identity Evidence: E-imports or equivalent; see radare2/ida/malware hard gates)
### 3.3 Dynamic analysis / behavior
(If no dynamic conditions: n/a + reason)
### 3.4 Core findings (Findings table or numbered list, with evidence_ids)

## 4. Incident response
(Execute only in-scope: 先确认 scope 并保全 sample, memory, process tree, network, logs first; then isolate the host; terminate processes, isolate/remove files, check hosts/startup, full-disk scan and recheck only after owner approval. 不得在证据保全前直接删除文件.)

## 5. Closing notes
(Risk reminder and prevention for operators/end users)

## 6. IOC
(G4 table)

## 7. Evidence-chain summary
(§0: E / F / P / Timeline; may merge with §3.4; fields stay)

## 8. Appendix
(Tool versions, repro commands, script paths)
```

### 2.2 Voice

- Default Chinese for Chinese users; conclusion first, then detail.
- Static analysis layered by component/stage; no unstructured log dumps.
- Response steps MUST be independently executable; no "raise security awareness" filler.

---

## 3. Flavor: `apt` (Kaspersky Securelist style)

**Narrative goal**: campaign story — who hit whom, when, with what chain; how the investigation moved; how components split; what defenders hunt with.

### 3.1 Recommended section order

```markdown
# [Campaign/cluster name]: [one-line impact]

> Date / team / industry and region (if known)

## 1. Executive summary
(G1: time window, victim profile, entry, family/cluster attribution, duration, top conclusions)

## 2. The infection chain
(Stages: delivery → exploit/loader → main implant → post-exfil; unknown segments as "limited visibility"
maps to Path; chain diagram recommended)

## 3. Incident investigation
(Investigation narrative: turning points, internal proxy/C2 traits, how scope expanded; attach Timeline)

## 4. Interesting findings
(3–7 non-obvious points; attach E-id / F-id when possible)

## 5. Technical analysis
### 5.1 Component overview table (loader / trojan / stealer / …)
### 5.2 Per-component behavior and config
### 5.3 Static highlights (IAT/pack/persistence Evidence)
### 5.4 Network and C2
(optional ATT&CK table G7)

## 6. Detection and mitigation
(Detection ideas / hunt clues / mitigation priority; no slogans)

## 7. IOC
(G4; grouped by type)

## 8. Evidence-chain summary
(§0 fields)

## 9. Appendix
(Sample list and hashes, tool versions, public IDs; do not copy external report body)
```

### 3.2 Voice

- Timeline and "limited visibility" MUST be honest.
- Interesting findings ≠ restated overview; write the investigation's real anomalies.
- Component analysis: table of role / persistence / C2 / deps, then expand.

---


## 3b. Thin overlay: `vuln` (vuln technical analysis · optional)

> Issue #65 addendum. Structure from public "OS/component vuln technical analysis" outlines. **Skeleton only.** Forbid copying screenshot/body PoC packets, exploit details, or unauthorized attack steps.
> **Not** a 3rd default full vendor flavor; overlay only on vuln-research tasks or explicit user request.

**Narrative goal**: "who is affected → how to confirm/repro (in-scope) → root cause and patch delta → how to mitigate".

### Suggested section order

```markdown
## 1. Vuln overview
### 1.1 Impact (version/component/config prerequisites)
### 1.2 Reproduction (authorized env; third-party repeatable; no weaponized-tutorial tone)

## 2. Vuln analysis
### 2.1 Crash / anomaly analysis (Evidence: crash logs, trigger conditions)
### 2.2 Patch analysis (diff/guard conditions/fix point — attach E-*)
### 2.3 PoC or trigger analysis (in-scope existing material only; protocol/input construction level)

## 3. Mitigation
### 3.1 Mitigations (config/kill-switches)
### 3.2 Official patch and verification

## 4. Evidence → Finding → Path (inline or standalone table)
```

### Hard constraints

- **MUST** scope/auth: no repro or PoC expansion on unauthorized targets
- **MUST** E/F/P: repro, crash, patch conclusions attach evidence_ids
- **MUST NOT** use `vuln` as the default malware/APT shell
- **MUST NOT** copy exploit code or full weaponized steps from external reports/screenshots
- IOC table: only if network/file indicators exist; else n/a or omit

---
## 4. Hook to existing task templates

| Task template (`security-report-templates.md`) | Overlay |
|------------------------------------------|----------|
| 1. RE report | Default `flavor = null`; keep original static/dynamic/repro skeleton and IAT hard-gate Evidence; §2 only for clear malware samples |
| 2. Pentest report | `flavor = null`; add applicable Base G1–G3; attack path aligns to §0 Path; IOC not required |
| 3. CTF Writeup | `flavor = null`; keep original challenge/approach/repro; IOC/ATT&CK not required |
| 4. JS/Web signing RE | `flavor = null`; original overview → locate → algorithm → repro; no malware shell |
| Malware / APT dedicated | Explicit full `malware` or `apt` skeleton |

**Conflict**: §0 Evidence-chain fields and scope gates **always win**. Flavor changes narrative order and professional shell only; MUST NOT drop E/F/P.

---

## 5. Selection pseudocode

```
if user_requests_kaspersky or apt or threat_campaign:
    flavor = apt
elif user_requests_huorong or vir_report or explicit_malware:
    flavor = malware
else:
    flavor = null  # original task template + applicable Base
overlay = null
if user_requests_vuln_tech_report or cve_patch_analysis:
    overlay = vuln  # thin only; never a third default full flavor
emit(base_report)
if flavor in (malware, apt):
    emit(report with flavor outline)
elif overlay == vuln:
    emit(report with vuln thin outline)
```

---

## 6. Completion checklist (self-check at report end)

- [ ] Flavor chosen, or explicit "task template + minimum set"
- [ ] G1 overview present and not filler
- [ ] §0 E/F/P fields complete
- [ ] `malware` / `apt` reports have IOC table (or n/a+reason)
- [ ] `malware` / `apt` reports have actionable recs/response
- [ ] Non-flavor tasks were not forced into malware/APT-only sections
- [ ] vuln only on vuln tasks; overview/analysis/mitigation skeleton + E/F/P; no unauthorized weaponized PoC
- [ ] No vendor-prose paste, no placeholder/TODO
- [ ] IAT and other hard-gate Evidence entered static/tech analysis (if this task did binary analysis)

---

## 7. Source register

- Kaspersky Securelist, “Updated MATA attacks industrial companies in Eastern Europe”: <https://securelist.com/updated-mata-attacks-industrial-companies-in-eastern-europe/110829> (structure reference; accessed 2026-08-11)
- Huorong public technical articles: <https://www.huorong.cn/> (site entry; accessed 2026-08-11. Register specific article URL, title, and access date when actually citing)
- ATT&CK technique IDs are normalized mapping only and MUST be backed by this engagement's Evidence; do not auto-import IOCs from external reports.

---

## 8. Non-goals

- Do not maintain extra full templates for Mandiant/CrowdStrike/QiAnXin (dual flavor + optional thin overlay covers common needs).
- Do not promote `vuln` to a default full flavor alongside malware/apt.
- Do not auto-crawl vendor sites to fill reports.
- Do not weaken the Evidence contract or authorization scope because of flavor.
