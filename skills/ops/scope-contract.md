# Generic scope contract (hard startup gate)

> **MUST**: before **ACT** on any security / reverse / pentest task, land `scope.md` under the current analysis project's `work/<case>/`.
> No scope → docs/routing only. **MUST NOT** scan, hook, or exploit a target.
> Template may be copied. Keep English field keys so scripts can validate.

## How to initialize

Windows:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File skills\scripts\case-init.ps1 -Hint "<one-line task>" -CaseName "my-case"
# Default output: current analysis project's work/<case>/scope.md
# When calling the skill from another directory: -ProjectRoot "C:\path\to\analysis-project"

# Legal local offline sample: auth granted + offline + explicit sample → ready_for_act=true
powershell -NoProfile -ExecutionPolicy Bypass -File skills\scripts\case-init.ps1 `
  -Hint "offline apk" -CaseName "my-sample" -Preset offline-sample -Sample ".\app.apk"
```

Linux / macOS / Kali:

```bash
bash skills/scripts/case-init.sh --hint "<one-line task>" --case-name "my-case"
# Default output: caller analysis project's work/<case>/scope.md
# When calling from another directory: --project-root "/path/to/analysis-project"

# Legal local offline sample
bash skills/scripts/case-init.sh \
  --hint "offline apk" --case-name "my-sample" \
  --preset offline-sample --sample ./app.apk
```

`-PackageRoot` / `--package-root` remain compatibility flags. New flows SHOULD use `ProjectRoot` / `--project-root` for case-artifact ownership.

## scope.md full template

```markdown
# Case Scope

## meta
- case_id: {YYYYMMDD-short}
- created: {ISO-8601}
- operator: {name or local}
- project_root: {caller analysis project}
- primary_skill: {from master-route}
- lead_role: lead   # see ops/role-map.md
- specialist_roles: []  # e.g. cie, cpe, cre

## auth
- status: granted | pending | denied
- basis: written_contract | bug_bounty_scope | ctf_public | own_system | lab_only
- evidence_of_auth: {ticket/path or "CTF public" or "owner-operated"}
- MUST NOT proceed if status != granted

## in_scope
- assets: []          # hosts, domains, APK paths, binaries, URLs
- surfaces: []        # web, mobile, binary, network, api
- activities: []      # recon, reverse, exploit_validate, report

## out_of_scope
- assets: []
- activities: []      # e.g. DoS, phishing real users, data exfil

## network_profile
- mode: offline | lab_only | authorized_target_only | unrestricted_lab
- notes: |
    offline = no outbound packets (static / local sample only)
    lab_only = lab/VM IPs only
    authorized_target_only = in_scope assets only
- MUST NOT use unrestricted against production without written auth

## deliverables
- report: true
- field_journal: true
- diagrams: true
- timeline: true

## constraints
- timebox: {}
- stealth: low | medium | high
- data_handling: anonymize | no_user_pii

## signoff
- ready_for_act: false
- checklist:
  - [ ] auth.status = granted
  - [ ] in_scope.assets non-empty OR offline sample path set
  - [ ] network_profile.mode chosen
  - [ ] out_of_scope reviewed
```

## Routing hook (AI MUST execute)

```text
RULES / MASTER-ROUTING / SKILL:
  1) master-route → PRIMARY
  2) platform-native case-init or handwritten scope.md
  3) auth not granted → STOP; only collect authorization evidence
  4) ready_for_act = true → open PRIMARY SKILL.md → ACT
```

`case-guard -Force` / `case-guard --force` is a compatibility flag. It **MUST NOT** bypass `auth.status`, a legal scope, network profile, or `ready_for_act`.

## network_profile cheat sheet

| mode | Allowed | Forbidden |
|---|---|---|
| `offline` | static analysis, local files, simulation | any egress, public RPC |
| `lab_only` | lab/CTF target nets | production / unauthorized IPs |
| `authorized_target_only` | in_scope list | assets outside the list |
| `unrestricted_lab` | isolated lab net (written) | internet production |

## Traits

- Pure Markdown, **no database**
- Orthogonal to `TOOLS.md` / `tool-index` / bootstrap: scope answers "may we hit it"; tool-index answers "what do we hit it with"
