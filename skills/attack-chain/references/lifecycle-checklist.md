# Pentest / attack-chain lifecycle checklist

> Align community pentest skill packs (e.g. Orizon claude-code-pentest six-phase) with this pack's `attack-chain` + `ops`.
> Inspired by public Claude pentest lifecycle skills (retrieved 2026-07); **commands and authorization follow this pack's scope**.
> Date: 2026-07-17

## Before use

- [ ] `case-init` done, `auth.status=granted`
- [ ] `network_profile` ≠ misusing unrestricted against production
- [ ] `lead` has assigned specialist_roles (`ops/role-map.md`)

## Phase gates

| Phase | Role | Pack skill | Done when |
|-------|------|------------|-----------|
| 0 Scope | lead | ops/scope-contract | ready_for_act |
| 1 Recon | cie | pentest-tools | assets list + timeline |
| 2 Enum/Vuln | cpe | pentest-tools / api-security | candidate F-* drafts |
| 3 Validate | cpe | pentest-tools | E-* + validated Finding |
| 4 Post-ex (if authorized) | cpe/lead | attack-chain latter half | stay inside out_of_scope |
| 5 RE assist | cre | ida/apk/js/… | only if client/binary needed |
| 6 Report | doc | docs-generator | Evidence→Finding→Path |
| 7 Journal | lead | field-journal | redacted |

## vs "give a domain, auto-pwn" skills (this pack)

| Typical external auto pack | reverse-skill |
|----------------------------|---------------|
| Default spray the domain | MUST scope an asset list |
| Weak evidence into the report | Mandatory E/F/P chain |
| Single session, no roles | role-map handoff |
| No tool index | tool-index + bootstrap |

## At least one timeline entry per phase

Format: `ops/timeline-workitem.md`.
