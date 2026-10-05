---
name: radio-sdr
description: Use for authorized RF/SDR security research including signal identification, replay feasibility study in shielded labs, and wireless protocol analysis outside classic Wi-Fi.
---

# RF / SDR Security Research

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: **spectrum and transmit are strictly regulated**; authorized bands / Faraday cage / lab targets only
2. `NOW`: scope MUST name device, band, and whether transmit is allowed (receive-only by default)
3. `ACT`: receive-only ID → demod analysis → lab replay feasibility

## When to use

- Wireless remotes/sensors and other non-Wi-Fi RF (authorized)
- ADS-B/remote-control protocol research (legal receive)
- Split with wifi-wireless: this skill is **general SDR RF**; Wi-Fi offense/defense goes to R29

## Workflow

```text
□ Confirm regulations and licenses
□ Receive-only: identify center frequency and modulation
□ GNU Radio / URH analysis
□ Replay only in a shielded room with written permission
□ Conclusion focus: unauthorized-control feasibility / hardening advice
```

## Toolchain

| Tool | Purpose |
|------|------|
| RTL-SDR / HackRF (compliant) | RX/TX hardware |
| URH / GNU Radio | analysis |
| Inspectrum | signals |

## References

- `references/sdr-lab-rules.md`
- `../wifi-wireless/` `../ot-ics/` `../hardware-security/`

## Routing context

**Upstream**: MASTER R38
**MUST NOT**: interfere with public comms, unauthorized transmit

## Task-complete self-check

- [ ] Default receive-only and recorded the legal boundary?
- [ ] Checklist?
