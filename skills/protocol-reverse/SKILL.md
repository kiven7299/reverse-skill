---
name: protocol-reverse
description: Use for authorized reverse engineering of custom binary protocols, Protobuf/gRPC, WebSocket frames, and PCAP-driven protocol recovery.
---

# Protocol Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm authorization and routine-ops boundary
2. `NOW`: confirm this is **protocol/traffic/serialization** reverse (pure web param signing → `js-reverse/`)
3. `NOW`: if the target has network interaction → finish `../scripts/case-init.ps1` scope; MUST NOT ACT on the target while `auth` is not granted
4. `NEXT`: read `../tool-index.md`; missing tools → bootstrap (tshark/wireshark may be manual)
5. `ACT`: enter workflow Phase 1; produce a frame-layout or message-dictionary draft

## When to use

- Custom TCP/UDP binary protocols
- Protobuf / gRPC / FlatBuffers / MessagePack
- WebSocket / MQTT / private RPC
- PCAP / PCAPNG field and state-machine recovery
- Client-server checks, sequence numbers, encrypted frame headers

## Do not use this skill

| Case | Go to |
|------|------|
| HTTP param signing / JS crypto only | `js-reverse/` |
| TLS cert issues only | `pentest-tools/` or a browser proxy |
| Deep firmware protocol-stack + emulation | `firmware-pentest/` then return here |

## Workflow

### Phase 1 — Collect and triage

```text
□ Obtain samples: PCAP / proxy export / client logs / binary
□ Mark direction: C→S / S→C; handshake, heartbeat, reconnect?
□ Fixed header? magic? length field? TLV? fixed length?
□ Compression (zlib/gzip/lz4) or crypto (in-frame AES/ChaCha)?
□ tshark -r cap.pcap -T fields -e frame.number -e ip.src -e tcp.payload
```

### Phase 2 — Recover frame layout

```text
□ Align multiple same-class messages; find invariant bytes / incrementing seq
□ Length field: big/little endian, includes header or not
□ Checksum: CRC16/32, checksum, HMAC location
□ Draw the state machine: Connect → Auth → Ready → Request/Response → Close
□ Tools: Wireshark custom dissector draft / ImHex / 010 Editor template / Kaitai Struct
```

### Phase 3 — Serialization and crypto

```text
□ Protobuf: recover .proto (blackboxprotobuf / pbtk / protoc --decode_raw)
□ gRPC: HTTP/2 headers + protobuf body
□ Crypto: find key derivation (client so/dll/JS) → pair ida-reverse / js-reverse / apk-reverse
□ Replay: authorized scope only; harmless fields first, then sensitive ops
```

### Phase 4 — Artifacts

```text
MUST produce:
- Message-type table (name / opcode / fields)
- At least 1 reproducible decode command or script
- Evidence: raw hex excerpt + decoded result (redacted)
```

## Toolchain

| Tool | Required | Purpose | Bootstrap |
|------|------|------|------|
| tshark / Wireshark | strongly recommended | PCAP parse | manual / winget |
| Python3 | yes | decode scripts | system |
| blackboxprotobuf | optional | unknown protobuf | pip |
| ImHex / 010 | optional | structure templates | manual |
| IDA / r2 / Ghidra | as needed | client serialization functions | see matching skill |

## References

- `references/protocol-workflow.md` — frame layout and Protobuf cheat sheet
- Related: `../ida-reverse/` `../js-reverse/` `../firmware-pentest/` `../pentest-tools/`

## Routing context

**Upstream**: `MASTER-ROUTING` R21 · `routing.md`
**Downstream**: need client algorithm → `ida-reverse`/`js-reverse`; need exploit replay → `pentest-tools`/`api-security`
**Peer**: `malware-analysis` (C2 protocol), `digital-forensics` (traffic forensics)

## Task-complete self-check

- [ ] Recovered message layout or state machine (not just pasted hex)?
- [ ] Reproducible decode command?
- [ ] Honored scope / redaction?
- [ ] Wrote back field-journal / report Checklist?
