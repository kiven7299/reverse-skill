---
name: ot-ics
description: Use for authorized OT/ICS security assessment covering Purdue model zoning, PLC/SCADA exposure, industrial protocol discovery, and safe passive-first evaluation.
---

# OT / ICS Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` — **OT mis-operation can cause physical harm**
2. `NOW`: written authorization MUST state: site, net segment, whether active scan / register writes are allowed
3. `NOW`: case-init; default **passive-first**; MUST NOT write PLC before `ready_for_act`
4. `NEXT`: tool-index; most ICS tools are manual and need an isolated lab net
5. `ACT`: asset and zone ID → exposure → read-only validation

## When to use

- ICS/SCADA/DCS security assessment (authorized)
- Purdue-model zoning and cross-zone channels
- Modbus/DNP3/S7/EtherNet/IP protocol exposure
- Engineering station, HMI, historian, jump host
- IT/OT fusion boundary (firewall rules, unidirectional gate)

## Safety iron rules (MUST)

```text
MUST NOT without explicit permission:
- Write PLC coils/registers
- High-rate full-net scan of production OT
- Interrupt SIS-related paths
Prefer: read-only ID, traffic mirror, offline firmware/config analysis
```

## Workflow

### Phase 1 — Zoning and assets

```text
□ Purdue L0–L5 sketch: field devices → control → supervisory → site DMZ → enterprise
□ Asset list: PLC/RTU/HMI/engineering station/historian/Jump host
□ Protocol and port baseline (authorized segments only)
```

### Phase 2 — Passive and read-only

```text
□ SPAN/mirror PCAP → protocol-reverse / Wireshark ICS dissectors
□ Offline audit of config and engineering files (TIA/RSLogix exports etc.)
□ Default passwords and cleartext protocols (unauthenticated Modbus) as Findings; do not write values
```

### Phase 3 — Restricted active (authorized only)

```text
□ Low-rate identify, maintenance window
□ Read-only function codes first
□ Evidence each step; stop and report immediately on anomaly
```

### Phase 4 — Firmware/patch surface

```text
□ Controller firmware version → CVE map (do not blindly flash firmware)
□ Joint firmware-pentest offline image analysis
```

## Toolchain

| Tool | Use | Notes |
|------|------|------|
| Wireshark ICS dissectors | Passive parse | Mirrored traffic |
| Nmap NSE (restricted) | Identify | Rate and time window |
| Claroty/Nozomi etc. | Asset discovery | Commercial/on-site |
| PLC vendor engineering software | Config audit | Offline first |
| binwalk / Ghidra | Firmware | Offline |

## References

- `references/ot-safe-assessment.md`
- `../firmware-pentest/` `../protocol-reverse/` `../network` via pentest-tools

## Routing context

**Upstream**: MASTER R28
**Downstream**: firmware deep-dive `firmware-pentest`; protocol `protocol-reverse`; IT lateral `windows-ad`/`attack-chain`
**Peer**: do not hit OT with generic web-scan defaults

## Task-complete self-check

- [ ] Defaulted to passive/read-only and recorded auth boundary?
- [ ] Avoided writes to control loops (unless explicitly allowed)?
- [ ] Findings include physical/process impact?
- [ ] Checklist / journal?
