---
name: macos-reverse
description: Use for authorized macOS and Mach-O reverse engineering including codesign, Objective-C/Swift recovery, endpoint security surfaces, and Apple platform malware analysis.
---

# macOS / Mach-O Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md`
2. `NOW`: confirm the target is macOS/Mach-O/App bundle (iOS IPA → `mobile-reverse/`)
3. `NEXT`: tool-index; jtool2/lldb etc.
4. `ACT`: codesign and load info → static → dynamic (lldb/Frida)

## When to use

- Mach-O executables / dylib / framework
- .app bundle, LaunchAgent/Daemon
- Objective-C / Swift symbols and runtime
- Notarization/signing, Hardened Runtime, TCC-related behavior analysis
- macOS malware static/dynamic analysis (with malware-analysis)

## Workflow

### 1. Bundle and signature

```bash
file target
codesign -dv --verbose=4 target
spctl -a -vv target 2>&1
otool -L target
```

### 2. Static

```text
□ class-dump / swift-demangle / Hopper / Ghidra / IDA
□ Strings and XPC service names, TCC-sensitive APIs
□ LC_LOAD_dylib deps and rpath
```

### 3. Dynamic

```text
□ lldb / Frida
□ Observe with fs_usage / log stream
□ Network: pair with protocol-reverse or a proxy
```

## Toolchain

| Tool | Purpose |
|------|------|
| otool / nm / codesign | System-bundled |
| Hopper / Ghidra / IDA | Decompile |
| class-dump / dsdump | ObjC |
| Frida / lldb | Dynamic |
| jtool2 | Mach-O |

## References

- `references/macho-triage.md`
- `../mobile-reverse/` (iOS) `../ghidra-reverse/` `../malware-analysis/`

## Routing context

**Upstream**: MASTER R31  
**Downstream**: iOS → mobile-reverse; generic samples → malware-analysis

## Task-complete self-check

- [ ] Signature / Hardened Runtime status recorded?
- [ ] Address-level / symbol-level conclusions present?
- [ ] Checklist?
