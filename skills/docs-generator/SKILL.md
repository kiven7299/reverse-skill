---
name: docs-generator
description: |
  Creates task-oriented technical documentation with progressive disclosure. Use when writing READMEs, API docs, architecture docs, or markdown documentation.
  Also use this skill at the END of any completed reverse engineering, penetration testing, CTF, or security analysis task to generate a formal report in the user's project directory.
  Trigger keywords: write report, write docs, produce report, writeup, technical docs, report, documentation.
---

# Technical Documentation

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm the current task matches this skill's scope
2. `NOW`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
3. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
4. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

For writing style, tone, and voice guidance, use `Skill(ce:writer)` with **The Engineer** persona.

## Security/RE task document output

When a reverse/pentest/CTF/security-analysis task finishes, this skill generates a formal technical document in the **user project directory**.

### When to trigger

1. Reverse task done, core conclusions produced (algorithm recovery, signature break, bypass, etc.)
2. Pentest done, vulns found and validated
3. CTF challenge solved, flag obtained
4. User explicitly asks for "a report/document/writeup"

### Template choice

| Task type | Template |
|---------|---------|
| APK/binary/so reverse | `references/security-report-templates.md` → reverse-engineering report |
| Pentest / vuln hunting | `references/security-report-templates.md` → pentest report |
| CTF solve | `references/security-report-templates.md` → CTF Writeup |
| JS/Web signature reverse | `references/security-report-templates.md` → signature-reverse report |
| Malware / APT / virus analysis report | `references/security-report-templates.md` + **`references/vendor-report-rules.md`** |
| General technical docs | `references/templates.md` → README / API docs |

### Vendor-report structure (Issue #65)

Security formal reports **MUST** read `references/vendor-report-rules.md` (structure only; do not copy vendor prose). Pick a vendor flavor only when task evidence or the user explicitly requires it; ordinary reverse and other tasks use `flavor = null`.

| Flavor / Overlay | When | Primary skeleton |
|------------------|--------|------------|
| `malware` | clear malware sample, trojan, white-plus-black, phishing poison | Huorong-style: overview → flow → sample analysis → IR → IOC |
| `apt` | APT/campaign/group/multi-stage infection chain/industry targeting | Kaspersky Securelist-style: abstract → infection chain → investigation narrative → Interesting findings → technical analysis → detection/mitigation → IOC |
| `flavor = null` | ordinary APK/ELF/PE/Mach-O reverse, algorithm/firmware analysis, pentest / CTF / JS signing | original task template + Base common elements; do not wrap malware/APT-only sections |
| thin `vuln` | user explicitly asks for vuln/patch/CVE technical analysis | overview → impact/repro → crash and patch analysis → hardening (overlay on null; not a 3rd default full-text flavor) |

Principle: **few templates, high quality** — only 2 vendor full-text flavors; `vuln` is an optional thin overlay, not a third default full-text template.
Applies **together with** §0 Evidence→Finding→Path; on conflict the Evidence contract wins.

### Output rules

- **Output location**: user's current project directory (not the skill pack)
- **Filename format**: `YYYY-MM-DD_[type]-[target-short]-report.md`
- **If the project has `docs/`**: prefer `docs/`
- **Encoding**: UTF-8
- **Language**: follow the conversation language (Chinese chat → Chinese report, English chat → English report)

### Quality requirements

- All code blocks MUST be directly runnable or have clear context
- No placeholder/TODO
- Key findings MUST have evidence
- Repro steps MUST let a third party reproduce independently
- Sensitive data (real tokens, passwords, internal URLs) replaced with placeholders
- **MUST** include the Evidence → Finding → Path chain (see `../ops/evidence-finding-path.md` and template §0)
- **MUST** read `references/vendor-report-rules.md`: pick `malware` / `apt` or `flavor = null` (vuln tasks may overlay thin `vuln`); with no flavor, emit only the original task template and applicable Base elements; do not force IOC/ATT&CK（不强制 IOC/ATT&CK）
- **SHOULD** cite case `scope.md` / `timeline.md` (`../scripts/case-init.ps1`)

### Diagram integration

When generating a report, call the `diagram-generator` skill at appropriate places for visuals:

| Report type | Suggested diagram | Diagram type |
|---------|---------|---------|
| Reverse-engineering report | function-call graph, data-flow diagram | Mermaid flowchart / sequenceDiagram |
| Pentest report | attack-path diagram, network topology | Mermaid flowchart / Graphviz |
| CTF Writeup | solve-flow diagram | Mermaid flowchart |
| JS signature-reverse report | request-chain sequence, algorithm flowchart | Mermaid sequenceDiagram / flowchart |

Embed diagrams as Mermaid code blocks in the report markdown so they render on GitHub/GitLab.

---

## Core Principles

### 1. Progressive Disclosure

Reveal information in layers:

| Layer | Content | User Question |
|-------|---------|---------------|
| 1 | One-sentence description | What is it? |
| 2 | Quick start code block | How do I use it? |
| 3 | Full API reference | What are my options? |
| 4 | Architecture deep dive | How does it work? |

**Warnings, breaking changes, and prerequisites go at the TOP.**

### 2. Task-Oriented Writing

```markdown
<!-- Bad: Feature-oriented -->
## AuthService Class
The AuthService class provides authentication methods...

<!-- Good: Task-oriented -->
## Authenticating Users
To authenticate a user, call login() with credentials:
```

### 3. Show, Don't Tell

Every concept needs a concrete example.

## Formatting Standards

- **Sentence case headings**: "Getting started" not "Getting Started"
- **Max 3 heading levels**: Deeper means split the doc
- **Always specify language** in code blocks
- **Relative paths** for internal links
- **Tables** for structured data with 3+ attributes

## Quality Checklist

- [ ] Code examples tested and runnable
- [ ] No placeholder text or TODOs
- [ ] Matches actual code behavior
- [ ] Scannable without reading everything
- [ ] Reader knows what to do next

## Anti-Patterns

| Problem | Fix |
|---------|-----|
| Wall of text | Break up with headings, bullets, code, tables |
| Buried critical info | Warnings/breaking changes at TOP |
| Missing error docs | Always document what can go wrong |

## Templates

For README, API endpoint, and file organization templates, see [references/templates.md](references/templates.md).

## Related Skills

- `Skill(ce:writer)` - Writing style, tone, and voice (load The Engineer persona)
- `Skill(ce:visualizing-with-mermaid)` - Architecture and flow diagrams


---

## On-Demand Bootstrap

This skill has no external-tool dependency; it is pure text generation. No bootstrap.

If diagrams must be rendered into the report, call the `diagram-generator/` skill.

---

## Routing context

**Upstream entry**: every security/RE skill auto-calls this skill after the task completes
**Trigger**:
- Auto: after the task, as behavior-chain step 9
- Manual: user says "write report", "produce docs", "writeup"

**Peer modules**:
- `apk-reverse/` — APK reverse done → reverse report
- `ida-reverse/` — binary analysis done → reverse report
- `radare2/` — CLI analysis done → reverse report
- `js-reverse/` — JS signature reverse done → signature report
- `reverse-engineering/` — general reverse done → reverse report
- `field-journal/` — report content is also a source for the evolution log

**Security report templates**: `references/security-report-templates.md`
**Vendor report rules**: `references/vendor-report-rules.md` (flavor: malware | apt | null; optional overlay: vuln)
**General doc templates**: `references/templates.md`


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Does the report include Evidence / Finding / Path (ops contract)?
- [ ] Did I complete and write back RULES Checklist items?
