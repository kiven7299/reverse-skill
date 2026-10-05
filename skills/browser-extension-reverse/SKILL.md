---
name: browser-extension-reverse
description: Use for authorized reverse engineering of browser extensions (Chrome/Firefox) including manifest analysis, background workers, and extension-based credential or traffic logic recovery.
---

# Browser Extension Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md`
2. `NOW`: confirm the target is a **browser extension** (crx/xpi/unpacked dir), not ordinary page JS (ordinary → `js-reverse/`)
3. `NEXT`: unpack the extension; read the manifest
4. `ACT`: permission surface → background scripts → network/storage hooks

## When to use

- Chrome/Edge MV2/MV3 extension analysis
- Firefox extensions
- Malicious-extension IOC, supply-chain extension poisoning
- Recover signing/crypto/proxy logic implemented in an extension

## Workflow

### 1. Package

```text
□ Unpack crx / take the extension dir from the profile
□ manifest.json: permissions, host_permissions, background, content_scripts
□ Assess over-permission (<all_urls>, webRequest, debugger)
```

### 2. Logic

```text
□ service_worker / background entry
□ content_script injection points and world (isolated)
□ chrome.storage / IndexedDB keys
□ Same as `js-reverse`: Observe network and messaging (runtime.sendMessage)
```

### 3. Dynamic

```text
□ Load the unpacked dir in developer mode
□ Check errors in chrome://extensions
□ Attach DevTools to the service worker
□ Frida/browser CDP (jshookmcp) if needed
```

## Toolchain

| Tool | Purpose |
|------|------|
| unzip/jq | manifest |
| Chrome DevTools | worker debug |
| js-reverse toolchain | deep JS |
| YARA | malicious-extension rules |

## References

- `references/extension-analysis.md`
- field-journal entries on extension recovery
- `../js-reverse/` `../malware-analysis/`

## Routing context

**Upstream**: MASTER R30
**Downstream**: heavily obfuscated JS → `js-reverse`; poisoning investigation → supply-chain / malware

## Task-complete self-check

- [ ] Listed the permission surface and entry scripts?
- [ ] Recovered the key data flow?
- [ ] Checklist?
