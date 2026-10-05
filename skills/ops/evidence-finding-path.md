# Evidence → Finding → Path chain

> Inspired by the Z3r0 Evidence Plane, landed as a **Markdown field contract**.
> reverse-skill trait: bind to `docs-generator` report templates, anonymized `field-journal` write-back, and reproducible commands.

## 1. Evidence (immutable observation)

Each evidence item is its own section or table row:

```markdown
### E-{nnn}
- title:
- observed_at:
- source_type: command | screenshot | file | log | memory | network | manual
- source_ref: {path or command id}
- content_hash: {sha256 of artifact if file, else n/a}
- artifact_path: {relative path under case root when content_hash is recorded, else n/a}
- repro_command: |
    {exact command}
- raw_excerpt: |
    {anonymized excerpt}
- linked_workitem: WI-{nnn} | n/a
- supersedes: E-{nnn} | none
```

**MUST**: a Finding cites at least 1 Evidence. `repro_command` MUST be runnable by a third party or marked offline-limited.

**CLI helper** (writes `work/<case>/evidence/E-*.md`):

```powershell
powershell -File skills/scripts/append-evidence.ps1 -CaseRoot work/<case> `
  -Id E-001 -Title "..." -ReproCommand "..." -Severity info -Status observed
```

When the evidence is a case-local file, pass `-ArtifactPath` to record SHA-256 fixity and a relative artifact path. Review the full case graph before handoff:

```bash
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

The review is read-only. It checks scope fields, Evidence records, work-item and timeline refs, structured Findings, Paths, and artifact hash matches.

## 2. Finding (security / reverse conclusion)

```markdown
### F-{nnn}
- title:
- severity: critical | high | medium | low | info | n/a_re
- category: vuln | misconfig | design | reverse_algo | bypass | other
- status: candidate | validated | false_positive | accepted_risk
- evidence_ids: [E-001, E-002]
- location: {file:line | addr | url | class.method}
- impact:
- confidence: high | medium | low
- repro_steps:
  1.
  2.
- remediation: {or n/a for pure RE}
- optional_attack: {ATT&CK ID or empty}
```

**MUST**: `evidence_ids` is non-empty. When `status=validated`, confidence MUST NOT be low unless residual risk is recorded.

## 3. Path (attack / call / solve)

Always called **Path**. Interpret by task type:

| Task | Path meaning |
|---|---|
| pentest / attack-chain | attack-path steps |
| reverse | key call / data-flow steps |
| CTF | solve steps |

```markdown
### P-{nnn}
- title:
- path_type: attack | callflow | solve
- start:
- goal:
- steps:
  1. action: — evidence: E-xxx — finding: F-xxx | none
  2. action: — evidence: E-xxx — finding: F-yyy | none
- residual_risks:
```

**MUST**: every step can bind Evidence. If an attack path claims "got access/data", the terminal Finding MUST have validated evidence.

## 4. Where this lives in the report

A `docs-generator` security report **MUST** contain:

1. Scope summary (link to case `scope.md`)
2. Evidence table or chapter
3. Findings list (with evidence_ids)
4. At least 1 Path (attack / call / solve)
5. Timeline summary (optional full link to `timeline.md`)

See the **Evidence Chain** section in `docs-generator/references/security-report-templates.md`.

## 5. field-journal hook

When writing the journal, **SHOULD** excerpt:

- up to 3 key Evidence ids + commands
- 1 core Finding
- one-line reusable Path pattern

Full sensitive content stays in the user's project report. Journal **MUST** be anonymized (`anonymization.md`).

## 6. Difference vs Z3r0

| Z3r0 | reverse-skill |
|---|---|
| PG immutable rows + API | Markdown files + hash fields |
| UI review queue | report + next-step menu + journal |
| deep ATT&CK binding | optional tags, no required UI |

## Validated sufficiency (Issue #77 / R4*)

Global bind rule: every Finding references **>=1** Evidence.

Promotion to `status=validated` is stricter (decision cookbook):

| status | Evidence bar |
|---|---|
| preliminary / candidate | >=1 (unchanged) |
| **validated** | **SHOULD >=2 independent** Evidence (best: 1 static + 1 dynamic). A single Evidence item MUST NOT silently promote to validated — keep candidate/preliminary, or record residual_risk + human confirm. |
| blocked promotion | record Evidence E-insufficient-evidence |

Full recipes: [analysis-decision-framework.md](analysis-decision-framework.md) (R4*, R1, R41, R44).
