---
name: mobile-reverse
description: Use for authorized Android or iOS application reverse engineering and security testing, including APK or IPA analysis, runtime instrumentation, SSL pinning, and platform protection checks.
---
# Mobile Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

> Unified Android + iOS reverse methodology
> Frida / Objection / OWASP MSTG / SSL Pinning Bypass

## When to use

- Android APK reverse and security testing
- iOS IPA reverse and security testing
- Mobile app runtime dynamic instrumentation
- SSL Pinning / Root detect / jailbreak-detect bypass
- Mobile crypto extract (AES/RSA/HMAC keys)
- Mobile app pentest (OWASP MASTG)
- App testing without Root/jailbreak

## Four-phase workflow

### Phase 1: Information gathering

```text
Android:
□ Obtain APK (Google Play / APKMirror / adb pull)
□ Manifest analysis: permissions, exported components, Intent Filter, backup flags
□ androguard: androguard analyze APK → components/permissions/signature
□ APKLeaks: hardcoded API Key / Token / Secret scan
□ Packer detect: whether packed (360/Tencent/Bangcle/ijiami)

iOS:
□ Obtain IPA (App Store / ipatool / Apple Configurator)
□ Decrypt App Store binary: frida-ios-dump / Clutch
□ Info.plist analysis: ATS config, URL Scheme, Queries Schemes
□ class-dump: export ObjC class structure
□ Packer detect: whether Swift/ObjC obfuscation is used
```

### Phase 2: Static analysis

```text
Cross-platform:
□ JADX-GUI: APK → Java source (Android)
□ Ghidra / Hopper: .so / Mach-O decompile
□ radare2 / Cutter: CLI fast recon

Android-specific:
□ apktool d app.apk → smali + resources
□ dex2jar: DEX → JAR → JD-GUI
□ smali/baksmali: Dalvik bytecode edit

iOS-specific:
□ class-dump: export ObjC headers
□ Swift symbol recover: swift-demangle
□ dsymutil: debug-symbol extract
□ otool -L: dynamic library deps
□ jtool2: Mach-O analysis
```

### Phase 3: Dynamic analysis

```text
Frida — generic dynamic instrumentation:
□ frida-ps -U: list device processes
□ frida-trace -U -i "open*" com.app: trace function calls
□ Custom Hook scripts: mutate args/returns, call private methods

Objection — Frida enhancement (no script required):
□ objection -g "com.app" explore
□ android root disable / ios jailbreak disable
□ android sslpinning disable / ios sslpinning disable
□ android keystore list / ios keychain dump
□ env / ls / sqlite connect

Frida Gadget (no Root/jailbreak):
□ Inject frida-gadget.so / FridaGadget.dylib into APK/IPA
□ Resign → install → Hook without device privileges
□ objection patchapk --source app.apk (fully automatic)
```

### Phase 4: Network analysis

```text
□ Burp Suite: intercept HTTP/HTTPS, mutate request/response
□ mitmproxy: scripted proxy (Python API)
□ Wireshark: PCAP capture analysis
□ Cert install: Android user cert → system cert (Magisk + MoveCert)
□ SSL Pinning bypass: Frida/Objection/Xposed/SSL Kill Switch 2
□ WebSocket / gRPC traffic analysis
```

## Common bypass quick lookup

### SSL Pinning

```bash
# Objection (simplest)
objection -g "com.app" explore
android sslpinning disable

# Frida generic script
frida -U -l ssl_pinning_bypass.js -f com.app

# Xposed (Android)
TrustMeAlready module → globally disable cert checks
```

### Root / jailbreak detect

```bash
# Objection
android root disable
ios jailbreak disable

# Frida custom (multi-layer detect)
Java.perform(function() {
    var RootBeer = Java.use("com.scottyab.rootbeer.RootBeer");
    RootBeer.isRooted.implementation = function() { return false; };
    // Extra bypass: Magisk su detect, frida-server detect, /proc/self/maps detect
});
```

### Anti-debug

```bash
# Android
frida -U -l anti_debug_bypass.js -f com.app
# Bypass: ptrace(TracerPid), /proc/self/status, isDebuggerConnected()

# iOS
# Bypass: PT_DENY_ATTACH, sysctl CTL_KERN/KERN_PROC/KERN_PROC_PID
frida -U -l ios_anti_debug.js -f com.app
```

## Mobile crypto extract

```javascript
// Android — Hook Cipher.getInstance for key+algorithm
Java.perform(function() {
    var Cipher = Java.use("javax.crypto.Cipher");
    Cipher.getInstance.overload('java.lang.String').implementation = function(algo) {
        console.log("[Cipher] Algorithm: " + algo);
        return this.getInstance(algo);
    };
    Cipher.init.overload('int', 'java.security.Key').implementation = function(mode, key) {
        console.log("[Cipher] Key: " + bytesToHex(key.getEncoded()));
        return this.init(mode, key);
    };
});

// iOS — Hook CCCrypt
Interceptor.attach(Module.findExportByName("libcommonCrypto.dylib", "CCCrypt"), {
    onEnter: function(args) {
        console.log("CCCrypt op: " + args[0] + " alg: " + args[1]);
        console.log("Key: " + hexdump(args[3], { length: args[4].toInt32() }));
    }
});
```

## Toolchain

| Tool | Platform | Purpose |
|------|:--:|------|
| JADX-GUI | A | Java decompile |
| apktool | A | APK unpack/rebuild |
| Ghidra | A+I | Multi-arch decompile |
| Hopper | I | iOS-specific disassembly |
| Frida | A+I | Dynamic instrumentation |
| Objection | A+I | Frida REPL enhancement |
| MobSF | A+I | Automated SAST+DAST |
| class-dump | I | ObjC class export |
| frida-ios-dump | I | IPA decrypt |
| jtool2 | I | Mach-O analysis |
| Burp Suite | A+I | HTTP intercept |
| mitmproxy | A+I | Scripted proxy |

> A=Android, I=iOS

## References

- `references/frida-objection-deep.md` — Frida + Objection deep usage
- `references/ios-reverse-guide.md` — iOS reverse special
- `references/anti-detection-bypass.md` — Root/jailbreak/anti-debug/SSL Pinning bypass


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES checklist items?
