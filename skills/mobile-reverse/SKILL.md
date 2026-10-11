---
name: mobile-reverse
description: Use for Android/iOS app shield analysis, RASP bypass, root/jailbreak detection bypass, SSL pinning bypass, anti-frida bypass, native C hooking (Frida alternative), and app protection assessment. Covers shield identification, static+dynamic analysis, bypass development, and verification. References the canonical mobile security research workspace.
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm authorization
2. `NOW`: confirm the task involves mobile app security analysis, shield bypass, or protection assessment
3. `NEXT`: read the research workspace overview at `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\WORKSPACE_PLAN.md`
4. `NEXT`: check `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\_agent_worklog\TODO.md` for current task status
5. `ACT`: enter the appropriate workflow phase

# Mobile App Shield Reverse Engineering & Bypass

## Scope

This skill covers the full lifecycle of mobile app protection analysis:

- Identify which shield/RASP/packer is protecting an Android/iOS app
- Analyze detection mechanisms (root, hook, debug, emulator, proxy, integrity)
- Develop targeted bypasses for each detection layer
- Verify bypass success (app runs on rooted device + traffic in proxy)
- Document findings and create reusable bypass artifacts

## Canonical Research Workspace

All mobile shield research, reports, tools, scripts, and methodologies live at:

```
D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\
```

### Key paths:
| Path | Content |
|------|---------|
| `_methodology/` | Reusable playbooks and technique guides |
| `_tools/` | Frida scripts, native hooks, setup guides |
| `_references/` | Curated external research |
| `_agent_worklog/` | Task tracker and session logs |
| `BShield/` | BShield research (VNEID, ZaloPay) |
| `momo/` | MoMo ProtectionSDK research |
| `MB_Bank/` | MBBank multi-shield research (Zimperium + V-Key + AnTian + BlueShield) |

## Research Environment

| Component | Detail |
|-----------|--------|
| Android device | Samsung, rooted (Magisk/KernelSU), USB ADB |
| HTTP proxy | Burp Suite at `http://*:8088`, invisible mode |
| Static analysis | JADX (MCP available), IDA Pro (MCP available) |
| Dynamic analysis | Frida, Objection, custom native C hooks |
| Build | Android NDK for native C cross-compilation |

## Success Criteria

A shield is **bypassed** when:
1. App runs normally on rooted/jailbroken device (no crash, no warning, full functionality)
2. SSL pinning bypassed — HTTPS traffic visible in Burp proxy
3. Core features work (login, transactions, main screens)

## Workflow

### Phase 0: Shield Identification

1. Install APK on clean (non-rooted) device, capture baseline behavior
2. Install on rooted device, observe: crash? warning dialog? blocked feature?
3. Static triage:
   - `apktool d` + JADX decompile
   - Check `assets/` for shield data files (`bshield.dat`, randomized model files, etc.)
   - Check `lib/` for shield .so files (names, sizes, exports)
   - Use APKiD for packer/protector fingerprinting
   - Search for known SDK strings: `zimperium`, `bshield`, `vkey`, `promon`, `guardsquare`, `talsec`, etc.
4. Map the detection surface from `AndroidManifest.xml` (services, receivers, activities with security names)

### Phase 1: Static Analysis (Java/Kotlin layer)

→ Route to `apk-reverse` skill for JADX + apktool workflows

Key targets:
- `System.loadLibrary` calls → identify native libs loaded by shield
- Root detection classes (search: `root`, `su`, `magisk`, `superuser`)
- Hook detection (search: `frida`, `xposed`, `substrate`, `hook`)
- SSL pinning config (`network_security_config.xml`, `CertificatePinner`, `TrustManager`)
- Device attestation (SafetyNet, Play Integrity API calls)
- Shield callback interfaces (result codes, detection flags)

### Phase 2: Static Analysis (Native layer)

→ Route to `ida-reverse` or `ghidra-reverse` skill for .so analysis

Key targets:
- JNI registration table (`RegisterNatives` → map Java methods to native functions)
- Detection function signatures (file checks, /proc reads, port scans, memory scans)
- String references: `/proc/self/maps`, `/data/local/tmp`, `frida`, `su`, `magisk`
- Anti-debug: `ptrace`, `PTRACE_TRACEME`, timing checks
- Integrity checks: CRC, SHA256, HMAC on memory/files

### Phase 3: Dynamic Analysis

Two approaches, choose based on anti-frida strength:

#### 3A: Frida-based (when anti-frida is weak or bypassable)
- Spawn mode (`frida -U -f <pkg>`) for early injection
- Hook detection functions → return benign values
- Trace JNI calls to map native behavior
- SSL pinning bypass scripts (OkHttp, TrustManager, Conscrypt, BoringSSL)
- Use `hluda` or renamed/reported frida-server to reduce detection surface

#### 3B: Native C hooking (when Frida is detected)
This is the **stealth alternative** to Frida. Compile a native .so that hooks target functions via PLT/GOT or inline patching, inject via:
- Zygisk module (hooks `postAppSpecialize`, before app anti-tamper arms)
- `LD_PRELOAD` injection
- `ptrace` + `dlopen` injection
- Process memory patching (`process_vm_writev`)

Key frameworks:
- `fake-linker` — PLT hook + soinfo manipulation
- `bhook` (ByteHook) — PLT hook by ByteDance
- Custom Zygisk modules for per-app hooks

Reference: `_methodology/07_native_c_hooking.md`

#### 3C: Read-only analysis (when no injection is desired)
- Custom native C scanner using `process_vm_readv()` to read ART heap
- `/proc/<pid>/maps` analysis
- `strace` / `ltrace` for syscall tracing
- `logcat` filtering for shield SDK output

### Phase 4: Bypass Development

For each detected check, develop a targeted bypass:

| Detection | Bypass Strategy |
|-----------|----------------|
| File-based root (`su`, Magisk paths) | Hook `faccessat`/`access`/`open` → return ENOENT for root paths |
| `/proc/mounts` Magisk traces | Hook `read()` on mounts fd → filter Magisk entries |
| `/proc/self/maps` frida traces | Hook `read()` on maps fd → filter frida/gum entries |
| Frida port scan (27042) | Rename frida-server + change port (`-l 0.0.0.0:12345`) |
| Frida memory signature | Use hluda (no frida string in .so name), or native C hooks instead |
| `ptrace` anti-debug | Hook `ptrace()` → return 0 |
| SSL CertificatePinner | Hook `check`/`check$okhttp` → no-op |
| SSL TrustManager | Replace with permissive TrustManager via `SSLContext.init` |
| SSL native BoringSSL | Hook `SSL_CTX_set_custom_verify` → callback returns 0 |
| Integrity/checksum | Patch comparison logic (cmp → always-equal), or provide correct hashes |
| Server-side attestation | Cannot bypass client-side; need to understand what client reports |

### Phase 5: Verification

1. Clean app data: `pm clear <package>`
2. Start app on rooted device
3. Verify: no crash, no warning, login works
4. Configure proxy, verify: traffic appears in Burp
5. Test core features through proxy
6. Document exact bypass steps for reproducibility

## Known Shield Signatures

| Shield | Key Indicators |
|--------|---------------|
| **BShield** | `assets/bshield.dat`, `libfumfucsdh.so`, `io.bshield.*` classes, Xqsig/Xqcode headers |
| **Zimperium zDefend** | `libempirememorial.so` (12MB), `d_zdevice_event_submit`, `zimperium.com` CDN |
| **V-Key V-OS** | `libvosWrapperEx.so`, `libchecks.so`, VTAP MethodChannel |
| **AnTian** | `libanTian37.so`, checks `/proc/mounts`, `/proc/fs/jbd2` |
| **Promon SHIELD** | `libshield.so`, encrypted string resolution |
| **DexProtector** | `libdpboot.so`, `libdexprotector.so`, anonymous executable segments |
| **Bangbang/Jiagu** | `libsecexe.so`, `libSecShell.so` |
| **Tencent Legu** | `libshell*.so`, `libtxAppProtect.so` |
| **360 Jiagu** | `libjiagu*.so` |

## Real-case reference (canonical workspace)

Live shield-breakdown cases live in the workspace
`D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\`.
Consult the matching real case before / while analyzing a new app of the same shield class —
offsets are build-specific but the decision tree generalizes.

| Shield / app | Case dir | What to copy |
|---|---|---|
| BShield (VNEID, ZaloPay) | `BShield/docs/` | Xqsig/Xqcode signer reuse, SSL callback patch, injector/harness |
| Zimperium + V-Key + AnTian (MB Bank) | `MB_Bank/` | multi-layer shield, server-hash GW934, `d_zdevice_event_submit` memory patch |
| MoMo ProtectionSDK | `momo/` | passive frida-file scan, `process_vm_readv` scanner |
| DexProtector | `dexprotector/` | packer notes |
| V-Key V-OS | `vkey-vos/` | V-Key notes |

Read `_agent_worklog/TODO.md` for the current open state before starting a new case.

## Routing to Other Skills

| Condition | Route To |
|-----------|----------|
| APK Java/Kotlin layer analysis | `apk-reverse` |
| Shield detected + Frida rejected / server-hashes files / native Xq signer | `native-instrumentation` |
| Native .so deep analysis | `ida-reverse` or `ghidra-reverse` |
| Binary patching with radare2 | `radare2` |
| WebView/hybrid JS analysis | `js-reverse` |
| Protocol/traffic reverse | `protocol-reverse` |
| Full report generation | `docs-generator` |
| General RE methodology | `reverse-engineering` |

## Agent Handoff

Before ending a session:
1. Update `_agent_worklog/TODO.md` with current status
2. Write session log to `_agent_worklog/sessions/YYYY-MM-DD_HH-MM.md`
3. Commit any new scripts/tools to appropriate directory
4. Note what the next session should start with

## Task-complete self-check (MUST pass before claiming done)

- [ ] Shield SDK identified (name/version) with evidence, or "unknown — needs analysis"
- [ ] Detection→response chain mapped, and each part labelled **local** vs **server-side**
      (a server-pushed flag is not cleared by a local patch)
- [ ] SSL pinning layer(s) enumerated (Java / Conscrypt native / Flutter / second verifier)
- [ ] Decision recorded: Frida OK vs must use native instrumentation (anti-frida strength)
- [ ] A real app action produced observable proof (traffic decrypts / screen changes / sent flag),
      not a catalog string
- [ ] Build/offset recorded as lib+offset/VA and read-back verified live on the right pid
- [ ] Workspace artifacts + session log updated; stray binaries removed from device
