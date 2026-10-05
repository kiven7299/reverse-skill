---
name: wifi-wireless
description: Use for authorized wireless security assessment including Wi-Fi capture, WPA handshake analysis, rogue AP detection research, and lab-only deauth testing.
---

# Wi-Fi / Wireless Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read precedent-pentest; **wireless attacks have high legal risk**, MUST have written authorization and physical range
2. `NOW`: scope MUST name target SSID/BSSID/site; MUST NOT scan neighbor networks
3. `NEXT`: confirm the adapter can monitor mode
4. `ACT`: recon → capture → analyze (lab first)

## When to use

- Authorized Wi-Fi security assessment
- WPA/WPA2 handshake capture and offline assessment
- Rogue AP / phishing hotspot detection research
- Enterprise wireless isolation and portal security

## Workflow

```text
□ iwconfig / airmon-ng enter monitor (legal env)
□ airodump-ng lock target BSSID channel
□ Handshake or PMKID capture (target only)
□ hashcat/aircrack offline password-policy assess
□ Report: crypto type, isolation, portal bypass, advice
```

## Toolchain

| Tool | Use |
|------|------|
| aircrack-ng suite | Capture/assess |
| hcxdumptool / hcxtools | PMKID |
| hashcat | Password-policy assess |
| Wireshark | Management-frame analysis |

## References

- `references/wireless-lab-rules.md`
- `../pentest-tools/` `../attack-chain/` (near-source chapter)

## Routing context

**Upstream**: MASTER R29
**MUST NOT**: unauthorized deauth, ops against non-target client networks

## Task-complete self-check

- [ ] Strictly locked to target BSSID?
- [ ] Hardening advice in the report?
- [ ] Checklist?
