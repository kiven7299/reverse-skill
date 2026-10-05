---
name: apk-reverse
description: Use when reversing Android APKs from the CLI. Covers APK unpacking, Java decompilation, smali edits, rebuild, Frida dynamic hooks, and switching to so/native analysis as needed. Prefer locally installed jadx, apktool, frida, adb, ida-reverse, and radare2.
---

## ACTION REQUIRED (execute immediately after reading)

> Endpoint extract / adaptive Frida community map: ../references/community-security-skills.md; dynamic analysis requires a scope-authorized device.

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

# APK reverse CLI work spec

## Scope

Prefer this skill when the task is:

- Analyze APK Java business logic
- Locate login, signing, risk control, certificate checks, root detection
- View and edit `AndroidManifest.xml`
- View and edit smali
- Rebuild an APK
- Frida Java/native dynamic hooks
- Switch to native analysis when the APK contains `.so`

## CLI tools verified on this machine

- `jadx` `1.5.5`
- `apktool` `3.0.2`
- `frida-ps` `17.9.6`
- `adb`
- `java`

## Prefer bundled scripts when

These flows are high-frequency and easy to mis-parameterize; prefer the skill scripts:

- One-shot `jadx + apktool` dump plus summary: `scripts/decode.ps1`
- Frida device check, process list, spawn/attach inject: `scripts/frida-run.ps1`
- Rebuild, align, sign, install APK: `scripts/rebuild-sign-install.ps1`
- Fast extract of Manifest components and permissions: `scripts/manifest-summary.ps1`

Keep these one-liners as direct calls; do not wrap them:

- `adb devices`
- `adb logcat`
- `frida-ps -U`
- `jadx --version`
- `apktool --version`

## Bundled scripts

### `scripts/decode.ps1`

Purpose:

- Run `jadx` and `apktool` together
- Default: create a task output dir next to the original APK
- Emit summary fields: `package`, `java_files`, `smali_dirs`, `so_files`
- Tolerate partial `jadx` decompile errors when usable artifacts remain

Example:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Name demo -SkipJadx
```

### `scripts/frida-run.ps1`

Purpose:

- Unify Frida device, process, spawn/attach entry points
- Avoid mixing `-f`, `-n`, `-U` when writing args by hand

Example:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -ListDevices
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -ListProcesses
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -Spawn -Package com.example.app -ScriptPath "D:\hooks\test.js"
```

### `scripts/rebuild-sign-install.ps1`

Purpose:

- `apktool b` rebuild APK
- `zipalign` align
- `apksigner` sign and verify
- Optional direct `adb install`

Example:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- Default: generate and reuse a debug keystore
- Default output next to `ProjectDir`, so it sits with the original package and unpack dir

### `scripts/manifest-summary.ps1`

Purpose:

- Extract package name
- List permissions
- List activity/service/receiver/provider
- Mark the main launcher activity

Example:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\manifest-summary.ps1" -ManifestPath "C:\work\apktool_out\AndroidManifest.xml"
```

To analyze `.so`, `lib/arm64-v8a/*.so`, `lib/armeabi-v7a/*.so`, also use:

- `ida-reverse`
- `radare2`

## Tool roles

### `jadx`

Use for:

- Java decompile reading
- Package, class, method name search
- High-level APK logic first

Common commands:

```bash
jadx -d jadx_out app.apk
jadx --single-class com.example.LoginActivity -d jadx_out app.apk
jadx --deobf -d jadx_out app.apk
```

### `JEB Pro` (optional commercial tool)

Use for:

- Cross-check and deep decompile of Android DEX / APK / ARM
- Extra static analysis when JADX output is incomplete or heavily obfuscated
- Second-toolchain check of classes, methods, and call relations on the same target

Bounds:

- JEB Pro is commercial software. MUST be obtained and licensed by the user; this pack will not download, crack, or bypass licensing.
- Call it only when `tool-index` confirms JEB is available on this machine; otherwise keep using `jadx`, `apktool`, Ghidra, IDA, or radare2.
- Third-party JEB MCP bridges are not a pack dependency. Before install, MUST review source, permissions, network behavior, and version per `../ops/skill-supply-chain.md`, then get explicit user confirmation to register.

### `apktool`

Use for:

- Unpack APK
- View and edit `AndroidManifest.xml`
- View and edit smali
- Rebuild APK

Common commands:

```bash
apktool d app.apk -o apktool_out
apktool b apktool_out -o rebuilt.apk
```

### `frida`

Use for:

- Dynamically observe Java method calls
- Hook native exports
- Bypass root detection, certificate checks, debug detection

Common commands:

```bash
frida-ps -U
frida -U -f com.example.app -l hook.js
frida-trace -U -f com.example.app -j '*!*certificate*'
```

### `adb`

Use for:

- Device connection
- Install APK
- View logs
- Pull files

Common commands:

```bash
adb devices
adb install -r app.apk
adb shell pm list packages
adb logcat
adb pull /data/local/tmp/file .
```

## Recommended workflow

### 1. Triage

Map APK structure first. Do not rush to patch or hook.

Suggested actions:

1. Export Java with `jadx -d jadx_out app.apk`
2. Export smali and resources with `apktool d app.apk -o apktool_out`
3. Read first:
   - `AndroidManifest.xml`
   - Main `package`
   - `application`, `activity`, `service`, `receiver`
   - Whether `lib/` contains `.so`
4. Issue #65 threat-shape lookup (authorized sample/device; see `../reverse-engineering/references/nonpe-format-cookbook.md` §7–8):
   - Transparent/hidden icon (AU): `aapt dump badging` + manifest theme/label/icon → `E-android-hidden-icon-manifest`
   - Magisk/script brick traits and remote curl|sh (AR/AS) → record traits and URLs as evidence; **do not execute** destructive commands
   - Persistence paths (AT): `service.d` / `priv-app` etc. → `E-android-persistence`

### 2. Java logic observation

Prefer reading `jadx_out`:

- `MainActivity`
- `Application`
- Login, network, crypto, risk-control classes
- Third-party SDK init classes

Common keywords:

- `login`
- `sign`
- `encrypt`
- `cipher`
- `token`
- `root`
- `certificate`
- `trust`
- `okhttp`
- `retrofit`
- `webview`

If Java is readable, locate business logic here first.

### 3. Smali and resource confirmation

When `jadx` is incomplete, obfuscation is heavy, or a real patch is needed, switch to `apktool_out`:

- Read `smali*/`
- Read `res/values/strings.xml`
- Read `AndroidManifest.xml`

Prefer patching:

- `android:exported`
- Debug flags
- Root-detection return values
- Login verification logic
- Certificate-check branches

### 4. Rebuild and install

After edits:

```bash
apktool b apktool_out -o rebuilt.apk
```

Or close the loop with the script:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- This skill only guarantees the `apktool` rebuild chain
- Device install usually still needs a signing flow
- If the task enters sign/align, add `apksigner` / `zipalign`

### 5. Dynamic Hook

When static analysis is not enough, use Frida:

- Hook login functions
- Hook `OkHttp` / `Retrofit` / `WebView` key points
- Hook `javax.crypto`, `MessageDigest`
- Hook root-detection functions
- Hook SSL pinning logic

Principles:

- Hook Java first, then decide if native hooks are needed
- Print args and return values first, then decide whether to mutate returns

Suggestions:

- One-shot commands: use `frida-*` directly
- Stable reusable inject flows: prefer `scripts/frida-run.ps1`

### 6. Native `.so` split

If the APK contains critical `.so`:

- Find `lib/**/*.so` with `apktool` or `jadx`
- For export symbols, strings, fast triage: `radare2`
- For long-term deep analysis, decompile, rename, type recovery: `ida-reverse`

Switch to native quickly when:

- Java is only a JNI wrapper
- Core signing logic is not in Java
- Key logic disappears after `System.loadLibrary()`
- Certificate checks / risk control live in `.so`

## Output requirements

At minimum state:

- Entry components and key classes
- Whether key logic is in Java, smali, or `.so`
- Confirmed sensitive points: login, signing, root, SSL, WebView, JNI
- If patched, what changed
- If hooked, which class/method/export

## MUST NOT

- Do not blindly edit smali at the start
- Do not write hooks before reading the manifest and main entry
- Do not treat incomplete Java decompile as "logic cannot be analyzed"
- Do not keep grinding Java when `.so` clearly holds core logic

## Quick command memo

```bash
# Decompile Java
jadx -d jadx_out app.apk

# Unpack APK
apktool d app.apk -o apktool_out

# Rebuild APK
apktool b apktool_out -o rebuilt.apk

# Device and processes
adb devices
frida-ps -U

# Spawn and inject
frida -U -f com.example.app -l hook.js
```

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Downstream exits**:
- Core logic in `.so` → `ida-reverse/` or `radare2/`
- Need dynamic Hook/verify → `reverse-engineering/tools-dynamic.md` (Frida section)
- General RE methodology → `reverse-engineering/SKILL.md`

**Peer modules**: `reverse-engineering/` (.so analysis and advanced Frida)

---

## On-Demand Bootstrap

This skill's entry scripts are wired to the unified bootstrap system. Missing tools do not fail immediately; install is attempted automatically.

### Automation bounds

| Tool | Auto-install | Method | Notes |
|------|-----------|---------|------|
| jadx | ✓ | GitHub Release ZIP | Download and extract to `%USERPROFILE%\Tools\jadx\` |
| apktool | ✓ | GitHub Release JAR + wrapper | Download jar and generate bat under `%USERPROFILE%\Tools\apktool\` |
| JEB Pro | ✗ | User installs and provides a valid license | Optional Android / ARM cross-check; third-party MCP bridge needs a separate audit |
| frida / frida-ps | ✓ | pip install frida-tools | Requires Python |
| adb | ✓ | winget / fallback path | Install Android Platform-Tools |
| zipalign | ✗ | Manual Android Build-Tools | `sdkmanager "build-tools;35.0.0"` |
| apksigner | ✗ | Manual Android Build-Tools | Same as above |

### Bootstrap triggers

- `scripts/decode.ps1`: missing jadx or apktool → auto-call `bootstrap-reverse.ps1`
- `scripts/rebuild-sign-install.ps1`: missing adb or apktool → auto-call bootstrap
- `scripts/frida-run.ps1`: still a manual check (frida is usually already installed via pip)

### On bootstrap failure

If auto-install fails, scripts throw a clear error plus a manual install link. Common causes:
- No network (GitHub API / PyPI unreachable)
- winget unavailable (Windows too old)
- Java not installed (apktool needs JDK)


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES checklist items?
- [ ] If hidden-icon / brick / persistence clues hit: did I record E-android-* Evidence per U–AV cookbook (in-scope)?
