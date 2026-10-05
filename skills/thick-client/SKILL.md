---
name: thick-client
description: Use for authorized security testing of desktop thick clients including local storage, update channels, IPC, traffic, and client-side trust boundaries.
---

# Thick Client Security Testing

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md`
2. `NOW`: confirm the target is a **desktop thick client** (Win/macOS/Linux GUI or companion service), not pure web
3. `NOW`: case-init; write installer source and test accounts into scope
4. `NEXT`: tools (Burp upstream proxy, process monitor, RE tools)
5. `ACT`: trust-boundary diagram → local surface → network surface → update/supply-chain

## When to use

- C/S-architecture clients, Electron/Qt/.NET WinForms/WPF
- Local config/credential storage, IPC, named pipes
- Client-enforced check bypass research (authorized)
- Auto-update channel and code-signing verification

## Workflow

### 1. Draw the boundary

```text
□ Process tree, child processes, drivers/services
□ Listening ports and outbound domains
□ Local sensitive paths: %APPDATA%, Keychain, registry
```

### 2. Local attack surface

```text
□ Cleartext config, hardcoded keys, debug switches
□ DLL hijack/search order (Windows)
□ Database files (SQLite) permissions and encryption
□ IPC: who can connect? is there authn?
```

### 3. Network surface

```text
□ System proxy / app custom TLS
□ Cert pinning → pair with mobile/js methodology or Frida
□ API over-privilege: admin APIs hidden in the client
```

### 4. Reverse validation

```text
□ .NET → dotnet-reverse; native → ida/ghidra; Electron → asar + js-reverse
```

## Toolchain

| Tool | Purpose |
|------|------|
| Process Monitor / API Monitor | behavior |
| Burp / mitmproxy | traffic |
| dnSpy / IDA / Ghidra | reverse |
| Sysinternals | Windows surface |
| asar / nexe detect | Electron |

## References

- `references/thick-client-checklist.md`
- `../dotnet-reverse/` `../ida-reverse/` `../js-reverse/` `../api-security/`

## Routing context

**Upstream**: MASTER R32
**Downstream**: pure protocol `protocol-reverse`; supply-chain updates `supply-chain-security`

## Task-complete self-check

- [ ] Drew the trust boundary?
- [ ] Covered both local and network surfaces?
- [ ] Checklist?
