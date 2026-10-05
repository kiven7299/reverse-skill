---
name: windows-ad
description: Use for authorized Active Directory and Windows identity attacks including Kerberos, AD CS, BloodHound paths, NTLM relay, and domain privilege escalation research.
---

# Windows / Active Directory Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md`
2. `NOW`: **domain/AD tests MUST have explicit authorized scope** (including DC, whether poisoning/relay is allowed)
3. `NOW`: case-init; write network_profile and MUST NOT actions clearly
4. `NEXT`: tool-index(impacket/certipy/bloodhound are often manual)
5. `ACT`: start from identity enum and the BloodHound graph; do not lead with destructive exploits

## When to use

- Domain pentest, Kerberoasting, AS-REP, delegation
- AD CS (ESC1–ESC8 etc.) certificate attacks
- BloodHound / SharpHound attack paths
- NTLM Relay / Coercer forced auth
- Local priv-esc to domain path (Potato etc. as a pivot)

## Relation to attack-chain

- **Multi-stage from internet to DC** → PRIMARY MAY still be `attack-chain/`; this skill is the **AD specialist**
- **Already in-domain, identity-focused** → PRIMARY = this skill

## Workflow

### 1. Enum

```bash
# example Impacket / built-in (needs creds and authorization)
nxc smb <range> -u user -p pass
bloodhound-python -d domain.local -u user -p pass -c All -ns <DC>
```

### 2. Common paths (graph first, gun second)

```text
□ Kerberoast / AS-REP → offline crack
□ ACL abuse (GenericAll/WriteDacl)
□ Delegation (unconstrained/constrained/resource-based)
□ AD CS template misconfig → Certipy
□ Relay: LLMNR/NBT-NS + ntlmrelayx（confirm authorization）
```

### 3. Creds and lateral

```text
□ secretsdump / lsassy / mimikatz (strict auth and cleanup)
□ PtH / PtT / golden ticket only in authorized red-team scope
□ Write Evidence each step; wait for user confirm on high-risk
```

## Toolchain

| Tool | Use |
|------|------|
| BloodHound / SharpHound | Path graph |
| Certipy | AD CS |
| Impacket / NetExec | Lateral and enum |
| Rubeus / Mimikatz | Tickets and creds (authorized) |
| Coercer / Responder | Forced auth / poisoning |

## References

- `references/ad-attack-paths.md`
- `../pentest-tools/references/network-attack-defense.md`
- `../attack-chain/`
- seeds: `field-journal/seed-005_ad-certipy-esc1.md` `seed-007_ntlm-relay-coercer.md` `seed-013_kerberoasting-spn.md`

## Routing context

**Upstream**: MASTER R24
**Downstream**: report `docs-generator`; need EDR research `edr-bypass-re`
**MUST NOT**: unauthorized DCSync / golden ticket against production

## Task-complete self-check

- [ ] Graph/enum before exploit?
- [ ] Reproducible commands recorded and redacted?
- [ ] Scope MUST NOT items honored?
- [ ] Checklist?
