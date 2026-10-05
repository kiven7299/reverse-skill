---
name: digital-forensics
description: Use for authorized digital forensics including memory dumps, disk timelines, PCAP investigation, artifact triage, and IR evidence preservation.
---

# Digital Forensics & IR Artifacts

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` or the org IR authorization note
2. `NOW`: confirm this is **forensics/attribution**, not offensive scanning
3. `NOW`: create the case; prefer read-only evidence copies (write-protect original media)
4. `NEXT`: tool-index; Volatility etc. are often manual
5. `ACT`: preserve hashes → timeline → key artifacts

## When to use

- Memory-dump analysis (Volatility 2/3)
- Disk / E01 / on-disk file timelines
- PCAP attribution and protocol recovery (may pair with `protocol-reverse/`)
- Host artifacts: Prefetch, Shimcache, Event Log, browser history
- IR IOC extraction (pair with `malware-analysis/` / `threat-hunting/`)

## Workflow

### 1. Preserve

```text
□ Compute SHA256; record timezone and collection command
□ Work on copies; originals read-only
□ Write chain-of-custody notes into timeline
```

### 2. Memory

```bash
vol -f mem.dmp windows.info
vol -f mem.dmp windows.pslist
vol -f mem.dmp windows.netscan
vol -f mem.dmp windows.cmdline
```

### 3. Host artifacts

```text
□ Event logs: Security / PowerShell / Sysmon
□ Persistence: Run keys, services, scheduled tasks, WMI
□ Execution traces: Amcache, Prefetch, BAM
```

### 4. Network

```text
□ tshark session and DNS stats
□ Export suspicious flows → protocol-reverse or malware C2 analysis
```

## Toolchain

| Tool | Purpose |
|------|------|
| Volatility 3 | memory |
| Timeline Explorer / Plaso | super timeline |
| tshark | PCAP |
| Eric Zimmerman tools | Windows artifacts |
| Autopsy / FTK Imager | disk |

## References

- `references/forensics-triage.md`
- `../malware-analysis/` `../threat-hunting/` `../protocol-reverse/`

## Routing context

**Upstream**: MASTER R25
**Downstream**: deep malware sample work → malware-analysis; rules → threat-hunting

## Task-complete self-check

- [ ] Preserved hashes and copy strategy?
- [ ] Timeline reviewable?
- [ ] IOCs classified/redacted?
- [ ] Checklist?
