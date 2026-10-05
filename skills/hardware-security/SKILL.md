---
name: hardware-security
description: Use for authorized hardware and embedded interface security research including UART/JTAG discovery, debug pad triage, secure boot overview, and offline firmware extraction support.
---

# Hardware / Embedded Interface Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm **physical-access authorization** and device ownership
2. `NOW`: ESD/power safety; read-only probing by default
3. `NEXT`: hand off image analysis to firmware-pentest
4. `ACT`: identify chassis and debug ports → consoles → extract

## When to use

- UART / JTAG / SWD debug-port discovery
- Boot logs, root shell, boot interrupt
- Pair with teardown to extract Flash
- Secure-boot / encrypted-Flash feasibility (non-destructive first)

## Workflow

```text
□ Disassemble the authorized device; photo-label test points
□ Multimeter for GND/VCC/TX/RX; logic levels 1.8/3.3/5V
□ USB-TTL read-only logs; record baud rate
□ JTAG: enumerate IDCODE; assess whether locked
□ Extract image → hand off firmware-pentest / ghidra
```

## Toolchain

| Tool | Purpose |
|------|------|
| USB-TTL / logic analyzer | UART |
| J-Link / CMSIS-DAP | debug |
| bus pirate / flipper (lab) | multi-protocol |
| binwalk / flashrom | extract |

## References

- `references/debug-interface-triage.md`
- `../firmware-pentest/` `../ot-ics/`

## Routing context

**Upstream**: MASTER R34
**MUST NOT**: unauthorized teardown/damage of others' devices

## Task-complete self-check

- [ ] Recorded interface levels and pinout?
- [ ] Image hash-preserved?
- [ ] Checklist?
