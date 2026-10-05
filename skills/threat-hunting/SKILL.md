---
name: threat-hunting
description: Use for blue-team threat hunting, detection engineering with Sigma/YARA, SIEM query design, and incident detection validation.
---

# Threat Hunting & Detection Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm blue-team/hunting auth and data-source scope (SIEM, EDR export)
2. `NOW`: state a hypothesis before querying; do not blindly dump alerts
3. `NEXT`: tools and data-access method
4. `ACT`: hypothesis → query → validate → rule-ize

## When to use

- Threat hunting (hypothesis-driven)
- Sigma / YARA detection engineering
- Alert tuning, false-positive analysis
- With `malware-analysis/`: sample-side IOC → this skill lands detection
- With `digital-forensics/`: case artifacts → lateral hunting

## Workflow

### 1. Build a hypothesis

```text
Example: attacker uses living-off-the-land for lateral movement
→ Data sources: Sysmon 1/3/10, Windows Security 4624/4648
→ Success criteria: anomalous parent process or rare account logon source
```

### 2. Query and stack

```text
□ Baseline: normal admin hours and hosts
□ Anomaly: new services, encoded PowerShell, unusual outbound
□ Correlate: same account, many hosts, short-time logons
```

### 3. Rule-ize

```yaml
# Sigma skeleton lives in malware-analysis; this skill stresses:
# - false-positive surface
# - data-source field mapping
# - response playbook links
```

### 4. Validate

```text
□ Atomic tests (Atomic Red Team) in authorized labs only
□ Replay historical logs to validate recall
```

## Toolchain

| Tool | Purpose |
|------|------|
| Sigma CLI / sigmac | rule convert |
| YARA | file/memory |
| SIEM (ELK/Splunk etc.) | query |
| osquery | endpoint hunting |
| Atomic Red Team | detection validation (lab) |

## References

- `references/hunting-loop.md`
- `../malware-analysis/references/yara-sigma-rules.md`
- `../digital-forensics/`

## Routing context

**Upstream**: MASTER R27
**Downstream**: confirmed intrusion → forensics; malware sample → malware-analysis
**MUST NOT**: run attack simulation on unauthorized production

## Task-complete self-check

- [ ] Clear hypothesis and conclusion?
- [ ] Rules note false positives and data sources?
- [ ] Checklist?
