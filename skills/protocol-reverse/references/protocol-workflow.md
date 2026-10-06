# Protocol reverse cheat sheet

> For: `protocol-reverse` skill · 2026-07-18

## Common layout patterns

| Pattern | Trait | Hint |
|------|------|------|
| Fixed header+body | First 2/4 bytes = length | Check whether length includes header |
| Magic | Fixed `0xDEAD` etc. | Helps stream resync |
| TLV | Repeating type-length-value | Type enum = message dictionary |
| Protobuf | Field-number varint | `protoc --decode_raw` |
| Encrypted frame | High entropy, no plaintext URL | Hunt nonce/IV neighborhood first |

## Minimal Python skeleton

```python
import struct
def parse_frame(buf: bytes):
    magic, length, msg_type = struct.unpack_from(">IHI", buf, 0)
    body = buf[10:10+length]
    return {"magic": magic, "type": msg_type, "body": body}
```

## Extract TCP payload from PCAP

```bash
tshark -r cap.pcap -Y "tcp.port==4433" -T fields -e tcp.payload | head
```
