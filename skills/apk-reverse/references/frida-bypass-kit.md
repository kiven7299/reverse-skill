# Frida Bypass Kit — Android generic security-bypass framework

> Source: [FridaBypassKit](https://github.com/okankurtuluss/FridaBypassKit) (2025)
> Use when APK dynamic analysis needs root, SSL pinning, emulator, or anti-debug bypass.

## Overview

FridaBypassKit is a Frida script that bundles four bypasses. No per-app customization; drop in and run.

## Four bypasses

### 1. Root-detection bypass

- Hook `File.exists()` to hide su binaries
- Intercept `Runtime.exec()` root-check calls
- Hide root packages from PackageManager (Magisk, SuperSU, etc.)
- Patch system properties so the device looks unrooted

### 2. SSL pinning bypass

- Hook `TrustManagerImpl.verifyChain()`
- Hook `TrustManagerImpl.checkTrustedRecursive()`
- Skip certificate-chain verification
- Return an empty chain to skip checks
- Works with OkHttp, Retrofit, and custom stacks

### 3. Emulator-detection bypass

- Fake TelephonyManager return values
- Return fake phone numbers and carrier names
- Patch Build properties

### 4. Anti-debug bypass

- Hook `Debug.isDebuggerConnected()`
- Block debugger detection
- Skip anti-debug checks

## Usage

```bash
# Prerequisites
pip install frida-tools
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell su -c /data/local/tmp/frida-server &

# Inject into the target app
frida -U -f com.example.app -l FridaBypassKit.js
```

## Other recommended Frida bypass scripts

| Project | Notes | Link |
|------|------|------|
| httptoolkit/frida-interception-and-unpinning | MitM all HTTPS traffic | [GitHub](https://github.com/httptoolkit/frida-interception-and-unpinning) |
| 0xCD4/SSL-bypass | Generic non-custom SSL bypass | [GitHub](https://github.com/0xCD4/SSL-bypass) |
| incogbyte/ssl-bypass gist | Bypass common SSL pinning methods | [Gist](https://gist.github.com/incogbyte/1e0e2f38b5602e72b1380f21ba04b15e) |
| Zero3141/Frida-OkHttp-Bypass | OkHttp CertificatePinner | [GitHub](https://github.com/Zero3141/Frida-OkHttp-Bypass) |

## Integration with this pack

In the `apk-reverse` workflow, use when:

1. App detects root and refuses to run → enable Root Detection Bypass
2. HTTPS capture shows no plaintext → enable SSL Pinning Bypass
3. App detects emulator and refuses to run → enable Emulator Detection Bypass
4. App crashes after Frida attach → enable Debug Detection Bypass

Preferred combo: run full FridaBypassKit first, then tune per target.
