# APK security-test cheat sheet

> Based on OWASP MASTG (Mobile Application Security Testing Guide).
> Six dimensions: static, dynamic, network, storage, authz, code protection.

---

## Static analysis checklist

### Manifest audit

```text
□ android:debuggable="true" → debuggable (must not ship in production)
□ android:allowBackup="true" → data can be backed up and extracted
□ android:exported="true" components → exposed Activity/Service/Receiver/Provider
□ Custom permission protectionLevel → is it normal (should be signature)
□ intent-filter scheme → custom deeplink hijackable
□ android:usesCleartextTraffic="true" → cleartext HTTP allowed
□ minSdkVersion too low → missing security features
```

### Code-audit hot spots

```text
□ Hardcoded keys/tokens (search "key", "secret", "password", "api_key")
□ Weak RNG (java.util.Random instead of SecureRandom)
□ Weak crypto (ECB, DES, MD5 for passwords)
□ WebView config (setJavaScriptEnabled + addJavascriptInterface = RCE)
□ SQL injection (rawQuery concatenating user input)
□ Path traversal (ContentProvider openFile without path checks)
□ Log leaks (Log.d/Log.i of secrets)
□ Clipboard leaks (ClipboardManager holding secrets)
□ Implicit Intent leaks (sendBroadcast without package)
```

### Third-party library audit

```text
□ Outdated OkHttp/Retrofit (known CVEs)
□ Outdated WebView engine
□ SDKs with known CVEs
□ Ad SDK collection scope
□ Push SDK config (token leak)
```

---

## Dynamic analysis checklist

### Frida hook priority targets

| Target | Hook | Goal |
|------|---------|------|
| Login | `LoginActivity.login()` | Watch credential handling |
| Signature | `*Sign*`, `*sign*`, `*encrypt*` | Recover signing algo |
| SSL pinning | `CertificatePinner.check` | Bypass capture |
| Root detect | `*root*`, `*su*`, `*magisk*` | Bypass detect |
| Crypto | `javax.crypto.Cipher` | Extract key/IV |
| Token store | `SharedPreferences.getString` | Watch token R/W |
| Network | `OkHttpClient.newCall` | Watch request build |

### Common Frida one-liners

```bash
# Trace all crypto
frida-trace -U -f com.target.app -j '*Cipher*!*'

# Trace all HTTP
frida-trace -U -f com.target.app -j '*OkHttp*!*'

# Trace SharedPreferences R/W
frida-trace -U -f com.target.app -j '*SharedPreferences*!*'

# Trace all native Java_* calls
frida-trace -U -f com.target.app -i 'Java_*'
```

### Objection quick commands

```bash
# Connect
objection -g com.target.app explore

# Common
android hooking list activities
android hooking list services
android sslpinning disable
android root disable
android clipboard monitor
env                              # app dirs
sqlite connect <db_path>         # connect DB
```

---

## Network security

### Capture setup

```text
Method 1: system proxy + Burp/mitmproxy
- WiFi proxy → Burp listen address
- Install CA on device
- Android 7+ needs network_security_config or Frida bypass

Method 2: VPN mode (preferred)
- HttpCanary / Packet Capture
- No root, no proxy config
- Cannot decrypt SSL-pinned traffic

Method 3: Frida + r2frida
- Intercept in-process
- Independent of proxy/VPN
```

### Checks

```text
□ HTTPS on all API calls
□ SSL pinning present
□ Cert validation correct (no self-signed accept)
□ Certificate Transparency (CT) checks
□ API keys in cleartext requests
□ Token expiry
□ Request signing against tamper
□ Replay protection (nonce/timestamp)
□ WebSocket encrypted
□ Secrets in URL query (logged)
```

---

## Data storage security

### Locations

| Location | Risk | Check |
|------|------|---------|
| SharedPreferences | Cleartext token/password | `adb shell cat /data/data/pkg/shared_prefs/*.xml` |
| SQLite | Unencrypted secrets | `adb pull /data/data/pkg/databases/` |
| External storage | World-readable | `adb shell ls /sdcard/Android/data/pkg/` |
| App logs | Debug leak | `adb logcat \| grep pkg` |
| Backup | allowBackup=true | `adb backup -f backup.ab pkg` |
| Keyboard cache | Input history | Check `inputType` is `textPassword` |
| Screenshot protect | Sensitive screens capturable | Check `FLAG_SECURE` |

### Encrypted storage comparison

| Scheme | Security | Notes |
|------|--------|------|
| SharedPreferences cleartext | ❌ | Readable after root |
| EncryptedSharedPreferences | ✓ | AndroidX Security |
| SQLCipher | ✓ | Encrypted SQLite |
| Android Keystore | ✓✓ | Hardware-backed keys |
| Custom AES | ⚠️ | Depends on key management |

---

## Authn / authz

### Common bugs

| Bug | Test |
|------|---------|
| Weak password policy | Try 123456, password, etc. |
| No lockout | Brute login API |
| Token never expires | Replay old token after logout |
| IDOR | Change user_id in request |
| SMS OTP brute | 4/6-digit, no rate limit |
| OAuth misconfig | Tamper redirect_uri |
| Biometric bypass | Hook BiometricPrompt |
| Device-bind bypass | Change device_id |

### Test payloads

```bash
# IDOR
curl -H "Authorization: Bearer USER_A_TOKEN" \
     "https://api.target.com/users/USER_B_ID/profile"

# Token replay
# 1. Login, get token
# 2. Logout
# 3. Replay old token → expect 401

# SMS OTP brute
for code in $(seq 0000 9999); do
    curl -X POST "https://api.target.com/verify" \
         -d "phone=13800138000&code=$code"
done
```

---

## Code-protection assessment

| Control | Detect | Bypass difficulty |
|---------|---------|---------|
| ProGuard | jadx class names a/b/c | Low (rename only) |
| String encrypt | Find decryptor, hook plaintext | Medium |
| Anti-debug | Attach debugger | Medium (Frida) |
| Root detect | Run on rooted device | Medium (generic scripts) |
| Emulator detect | Run on emulator | Low–medium |
| Integrity check | Modify APK then install | Medium (patch checker) |
| Packer | Entry class and .so | Medium–high (unpack) |
| Native protect | Core logic in .so | High (IDA) |
| VMP | Virtualized code | Very high |

---

## 30-minute quick test

```text
1. [5min] Unpack + Manifest audit
   apktool d app.apk
   Check debuggable/allowBackup/exported/cleartext

2. [10min] Fast code audit
   jadx -d out app.apk
   Search: password, key, secret, token, http://

3. [5min] Network
   Set proxy → use app → check cleartext/weak crypto

4. [5min] Storage
   adb shell → shared_prefs and databases

5. [5min] Dynamic confirm
   Frida-hook key functions → confirm findings
```
