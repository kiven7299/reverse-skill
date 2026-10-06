# Android advanced reverse reference

> Native SO analysis, advanced Frida, SSL pinning bypass, root-detection evasion, packer unpack, Flutter/React Native reverse.

---

## Native SO reverse

### Analysis flow

```text
1. Extract .so files from the APK
   unzip app.apk lib/arm64-v8a/*.so -d extracted/

2. Confirm architecture and basics
   file libxxx.so
   rabin2 -I libxxx.so

3. Find JNI entry points
   - Search JNI_OnLoad (dynamic register)
   - Search Java_com_xxx_yyy (static register)
   - nm -D libxxx.so | grep -i java

4. Load in IDA/Ghidra
   - Import JNI headers (jni.h types)
   - Annotate JNIEnv* args
   - Find RegisterNatives (dynamic native table)

5. Locate key logic
   - Trace from Java native method names
   - Xref strings (keys, URLs, errors)
   - Trace crypto lib calls (AES/MD5/SHA)
```

### JNI function registration

```c
// Static register: name = Java_package_class_method
JNIEXPORT jstring JNICALL Java_com_example_app_Security_getSign(
    JNIEnv *env, jobject thiz, jstring input) { ... }

// Dynamic register: RegisterNatives in JNI_OnLoad
static JNINativeMethod methods[] = {
    {"getSign", "(Ljava/lang/String;)Ljava/lang/String;", (void*)native_getSign},
};

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    vm->GetEnv((void**)&env, JNI_VERSION_1_6);
    jclass clazz = env->FindClass("com/example/app/Security");
    env->RegisterNatives(clazz, methods, sizeof(methods)/sizeof(methods[0]));
    return JNI_VERSION_1_6;
}
```

### IDA JNI tips

```text
1. Import JNI type library
   File → Load File → Parse C Header → jni.h

2. Type first arg as JNIEnv*
   Right-click arg → Set type → JNIEnv*
   Then env->FindClass / env->GetMethodID resolve automatically

3. Find RegisterNatives
   Search JNIEnv vtable offset 0x35C (ARM64)
   → 3rd arg is JNINativeMethod array
   → Extract all native function addresses from the array
```

---

## Advanced Frida

### Hook native functions

```javascript
// Hook libc
Interceptor.attach(Module.findExportByName("libc.so", "open"), {
    onEnter: function(args) {
        this.path = args[0].readUtf8String();
        console.log("[open] " + this.path);
    },
    onLeave: function(retval) {
        if (this.path.includes("su") || this.path.includes("magisk")) {
            console.log("[open] Blocked root check: " + this.path);
            retval.replace(-1);  // fail
        }
    }
});

// Hook a function in a custom SO
var base = Module.findBaseAddress("libsecurity.so");
var targetFunc = base.add(0x1234);  // offset
Interceptor.attach(targetFunc, {
    onEnter: function(args) {
        console.log("arg0: " + args[0].readUtf8String());
    },
    onLeave: function(retval) {
        console.log("return: " + retval.readUtf8String());
    }
});
```

### Hook Java methods

```javascript
Java.perform(function() {
    // Instance method
    var Security = Java.use("com.example.app.Security");
    Security.getSign.implementation = function(input) {
        console.log("[getSign] input: " + input);
        var result = this.getSign(input);  // original
        console.log("[getSign] output: " + result);
        return result;
    };

    // Constructor
    Security.$init.overload('java.lang.String').implementation = function(key) {
        console.log("[Security.<init>] key: " + key);
        this.$init(key);
    };

    // Overload
    Security.encrypt.overload('java.lang.String', 'int').implementation = function(data, mode) {
        console.log("[encrypt] data=" + data + " mode=" + mode);
        return this.encrypt(data, mode);
    };
});
```

### Memory search and patch

```javascript
// Scan for a string in memory
Process.enumerateModules().forEach(function(module) {
    if (module.name === "libtarget.so") {
        Memory.scan(module.base, module.size, "48 65 6C 6C 6F", {  // "Hello"
            onMatch: function(address, size) {
                console.log("Found at: " + address);
            }
        });
    }
});

// Patch instructions
var addr = Module.findBaseAddress("libsecurity.so").add(0x5678);
Memory.patchCode(addr, 4, function(code) {
    var writer = new Arm64Writer(code, {pc: addr});
    writer.putNop();  // replace with NOP
    writer.flush();
});
```

---

## SSL pinning bypass

### Generic (preferred)

```javascript
// Generic Frida SSL pinning bypass
// Source: https://github.com/0xCD4/SSL-bypass
Java.perform(function() {
    // 1. TrustManager bypass
    var TrustManager = Java.registerClass({
        name: 'com.custom.TrustManager',
        implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: {
            checkClientTrusted: function(chain, authType) {},
            checkServerTrusted: function(chain, authType) {},
            getAcceptedIssuers: function() { return []; }
        }
    });

    // 2. SSLContext swap
    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    var sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, [TrustManager.$new()], null);

    // 3. OkHttp CertificatePinner bypass
    try {
        var CertificatePinner = Java.use('okhttp3.CertificatePinner');
        CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function() {};
    } catch(e) {}
});
```

### Per-framework bypass

| Framework | Bypass |
|------|---------|
| OkHttp3 | Hook `CertificatePinner.check` to no-op |
| Retrofit | Same as OkHttp (uses OkHttp) |
| Volley | Hook `HurlStack` SSL factory |
| Flutter | Hook `dart:io` `SecurityContext` (needs a special script) |
| React Native | Hook `OkHttpClientProvider` |
| WebView | Hook `WebViewClient.onReceivedSslError` |

### Flutter-specific

```javascript
// Flutter SSL pinning bypass (find ssl_verify_peer_cert)
var flutter_lib = Module.findBaseAddress("libflutter.so");
// Signature for ssl_verify_peer_cert
var pattern = "FF 03 05 D1 FD 7B 0F A9";  // ARM64
Memory.scan(flutter_lib, Module.findModuleByName("libflutter.so").size, pattern, {
    onMatch: function(address) {
        Interceptor.replace(address, new NativeCallback(function() {
            return 0;  // success
        }, 'int', []));
    }
});
```

---

## Root-detection bypass

### Common checks

| Check | Bypass |
|---------|---------|
| `/system/app/Superuser.apk` | Hook `File.exists()` → false |
| `su` command | Hook `Runtime.exec()` and block su |
| `/proc/self/mounts` | Hook file read; filter magisk |
| SafetyNet/Play Integrity | Magisk Hide / Zygisk + Shamiko |
| Magisk package name | Randomize Magisk package |
| `/data/adb/` | Hook `opendir`/`access` |

### Generic Frida root bypass

```javascript
Java.perform(function() {
    // Hook File.exists
    var File = Java.use("java.io.File");
    File.exists.implementation = function() {
        var path = this.getAbsolutePath();
        var blacklist = ["su", "Superuser", "magisk", "busybox", "xposed"];
        for (var i = 0; i < blacklist.length; i++) {
            if (path.toLowerCase().includes(blacklist[i])) {
                return false;
            }
        }
        return this.exists();
    };

    // Hook System.getProperty
    var System = Java.use("java.lang.System");
    System.getProperty.overload('java.lang.String').implementation = function(key) {
        if (key === "ro.debuggable" || key === "ro.secure") {
            return "1";
        }
        return this.getProperty(key);
    };
});
```

---

## Packer ID and unpack

### Common packers

| Packer | Fingerprint | Unpack |
|------|---------|---------|
| 360 Jiagu | `libjiagu.so`, `com.stub.StubApp` | FART / Frida dump dex |
| Tencent Legu | `libshell*.so`, `com.tencent.StubShell` | FART / BlackDex |
| Bangcle | `libDexHelper.so`, `com.secneo.apkwrapper` | FART |
| ijiami | `libexec.so`, `s.h.e.l.l` | Frida dump |
| NetEase Yidun | `libnesec.so` | Frida dump |
| Naga | `libnaga.so` | Frida dump |

### Generic unpack methods

```text
Method 1: FART (ART unpack)
- Flash FART ROM or use Frida FART
- Auto-dump all ClassLoader DEX

Method 2: Frida DEX dump
- frida -U -f com.target.app -l dex_dump.js
- Hook DexFile::OpenMemory and dump in-memory DEX

Method 3: BlackDex
- No-root unpacker
- Install BlackDex APK and pick the target app

Method 4: Manual dump
- Frida-enumerate all ClassLoaders
- App ClassLoader → DexFile objects
- Read DEX memory and save
```

### Frida DEX dump script

```javascript
Java.perform(function() {
    Java.enumerateClassLoaders({
        onMatch: function(loader) {
            try {
                var dexFiles = loader.getDexFileList();
                console.log("ClassLoader: " + loader);
                console.log("  DEX files: " + dexFiles);
            } catch(e) {}
        },
        onComplete: function() {}
    });
});
```

---

## React Native / Flutter reverse

### React Native

```text
1. Unzip APK → assets/index.android.bundle (JS)
2. Pretty-print JS → search APIs, keys, signing
3. If Hermes bytecode (.hbc) → hermes-dec
4. Hook: Frida on Java ReactBridge
```

### Flutter

```text
1. Flutter compiles to libapp.so (Dart AOT)
2. No direct Dart source decompile
3. Methods:
   - reFlutter: patch libflutter.so for snapshot
   - Doldrums: parse Dart snapshot for class/function info
   - Frida-hook key functions in libflutter.so
4. Network: Flutter skips system proxy; SSL needs special handling
```

---

## Tool cheat sheet

| Tool | Use | Install |
|------|------|------|
| jadx | Java decompile | In bootstrap |
| apktool | Unpack/repack | In bootstrap |
| Frida | Dynamic hook | `pip install frida-tools` |
| Objection | Frida wrapper (easier) | `pip install objection` |
| MobSF | Automated mobile security | Docker |
| BlackDex | No-root unpack | APK |
| FART | ART unpack | Flash ROM or Frida build |
| hermes-dec | Hermes bytecode decompile | npm |
| reFlutter | Flutter reverse helper | pip |
| Magisk + Shamiko | Root hide | Flash |

---

## References

| Resource | Notes | Link |
|------|------|------|
| OWASP MASTG | Mobile security testing guide | https://mas.owasp.org/ |
| FridaBypassKit | Generic bypass framework | https://github.com/okankurtuluss/FridaBypassKit |
| SSL-bypass | Generic SSL pinning bypass | https://github.com/0xCD4/SSL-bypass |
| awesome-frida | Frida resource index | https://github.com/dweinstein/awesome-frida |
| Android Security Awesome | Android security resources | https://github.com/ashishb/android-security-awesome |
