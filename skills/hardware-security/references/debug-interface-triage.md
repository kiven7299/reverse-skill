# Debug-interface triage

1. Find silkscreen: TX RX GND VCC TDI TDO TCK TMS
2. Match voltage before connecting
3. Read-only serial logs first
4. Record U-Boot interrupt key and env vars (do not saveenv casually)
5. SHA256 after image extract
