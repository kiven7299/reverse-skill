# Root / jailbreak / anti-debug / SSL pinning bypass

## Detection layer model

```
Layer 1: static (install / startup)
  ├─ Package manager (Cydia, apt, Magisk)
  ├─ File checks (su, busybox, frida-server)
  └─ Property checks (ro.debuggable, ro.secure)

Layer 2: runtime (continuous)
  ├─ Process (frida-server, magiskd)
  ├─ Port (27042 Frida default)
  ├─ Memory (/proc/self/maps inject traces)
  └─ Stack (Frida frames)

Layer 3: environment (on demand)
  ├─ ptrace (TracerPid)
  ├─ /proc/self/status
  ├─ build.prop (test-keys)
  └─ Direct syscall (bypass libc)
```

## Android root-detection bypass

### Common libraries

| Library | Method | Bypass |
|--------|---------|---------|
| RootBeer | 8 combined checks | Hook each check → false |
| SafetyNet | Google Play Services remote attest | Magisk Hide / Shamiko / Play Integrity Fix |
| Google Play Integrity | SafetyNet successor | Trickystore + PIF |
| Custom native | syscall read /proc/self/status | Hook syscall or remount /proc |

### Combined Frida bypass

```javascript
Java.perform(function() {
    // RootBeer
    var RootBeer = Java.use("com.scottyab.rootbeer.RootBeer");
    var methods = ["isRooted", "isRootedWithBusyBox", "checkSuExists",
        "detectRootManagementApps", "detectPotentiallyDangerousApps",
        "detectTestKeys", "checkForDangerousProps", "checkForRWPaths"];
    methods.forEach(function(m) {
        RootBeer[m].implementation = function() { return false; };
    });

    // Generic Build.TAGS
    var Build = Java.use("android.os.Build");
    var original = Build.TAGS.value;
    Build.TAGS.value = "release-keys";

    // PackageManager → hide packages
    var PackageManager = Java.use("android.content.pm.PackageManager");
    PackageManager.getPackageInfo.overload('java.lang.String', 'int').implementation = function(pkg, flags) {
        if (pkg == "de.robv.android.xposed.installer" || 
            pkg.includes("magisk") || pkg.includes("frida")) {
            throw Java.use("android.content.pm.PackageManager$NameNotFoundException").$new();
        }
        return this.getPackageInfo(pkg, flags);
    };
});
```

## iOS jailbreak-detection bypass

### Multi-layer Frida hook

```javascript
// 1. Filesystem
var NSFileManager = ObjC.classes.NSFileManager;
var paths = [
    "/Applications/Cydia.app", "/var/lib/apt", "/bin/bash",
    "/usr/sbin/sshd", "/etc/apt", "/Library/MobileSubstrate"
];
// Hook fileExistsAtPath → NO

// 2. fork (forbidden in sandbox)
var fork_ptr = Module.findExportByName("libSystem.B.dylib", "fork");
Interceptor.replace(fork_ptr, new NativeCallback(function() {
    return -1;
}, 'int', []));

// 3. Scheme
// Via MobileSubstrate hook
var LSApplicationWorkspace = ObjC.classes.LSApplicationWorkspace;
// Hook defaultWorkspace → canOpenURL → NO for cydia://

// 4. Signature
var MISValidateSignature = Module.findExportByName(null, "MISValidateSignature");
Interceptor.attach(MISValidateSignature, {
    onLeave: function(retval) { retval.replace(0); }
});
```

## Anti-debug bypass

### Android

```javascript
// 1. ptrace self → block attach
// Native: ptrace(PTRACE_TRACEME, 0, NULL, 0)
// Bypass: Hook ptrace → return 0

// 2. TracerPid
// /proc/self/status → TracerPid: 0
var fopen = Module.findExportByName(null, "fopen");
Interceptor.attach(fopen, {
    onEnter: function(args) {
        this.path = Memory.readUtf8String(args[0]);
    },
    onLeave: function(retval) {
        if (this.path && this.path.includes("status")) {
            // Rewrite FILE* contents
        }
    }
});

// 3. isDebuggerConnected (Java)
var Debug = Java.use("android.os.Debug");
Debug.isDebuggerConnected.implementation = function() { return false; };
```

### iOS

```javascript
// 1. PT_DENY_ATTACH
// ptrace(PT_DENY_ATTACH, 0, NULL, 0) → block debugger attach
var ptrace = Module.findExportByName(null, "ptrace");
Interceptor.replace(ptrace, new NativeCallback(function(request, pid, addr, data) {
    if (request == 31) return 0; // PT_DENY_ATTACH → ignore
    return ptrace(request, pid, addr, data);
}, 'int', ['int', 'int', 'pointer', 'int']));

// 2. sysctl
var sysctl = Module.findExportByName(null, "sysctl");
Interceptor.attach(sysctl, {
    onLeave: function(retval) {
        // Clear kinfo_proc p_flag P_TRACED
    }
});

// 3. getppid (parent should be launchd)
// Under debugger getppid() != 1
```

## SSL pinning bypass

### Android five layers

```text
Layer 1 — TrustManager: accept all certs
Layer 2 — OkHttp CertificatePinner: hook and empty pins
Layer 3 — WebView SSL error handler: ignore cert errors
Layer 4 — Network Security Config: edit xml → trust user CAs
Layer 5 — Native SSL (OpenSSL/BoringSSL): Hook SSL_get_verify_result → X509_V_OK
```

### iOS four layers

```text
Layer 1 — NSURLSession: Hook SecTrustEvaluate → kSecTrustResultProceed
Layer 2 — Alamofire: Hook ServerTrustManager
Layer 3 — AFNetworking: Hook AFSecurityPolicy
Layer 4 — libcurl: LD_PRELOAD SSL verify callback
```

### Generic Objection commands

```bash
# Android
objection -g "com.app" explore
android sslpinning disable
# Equivalent: auto-hook the 5 layers above

# iOS
objection -g "com.app" explore
ios sslpinning disable
# Equivalent: auto-hook the 4 layers above
```

Source: OWASP MSTG, Frida CodeShare, objection wiki
