# Frida + Objection deep usage

## Frida core API

### Java runtime (Android)

```javascript
Java.perform(function() {
    // Get class
    var String = Java.use("java.lang.String");

    // Hook static method
    var System = Java.use("java.lang.System");
    System.getProperty.overload('java.lang.String').implementation = function(key) {
        console.log("System.getProperty: " + key);
        return this.getProperty(key);
    };

    // Hook constructor
    var File = Java.use("java.io.File");
    File.$init.overload('java.lang.String').implementation = function(path) {
        console.log("File opened: " + path);
        return this.$init(path);
    };

    // Enumerate loaded classes
    Java.enumerateLoadedClasses({
        onMatch: function(className) { console.log(className); },
        onComplete: function() {}
    });

    // Change return value
    var RootDetector = Java.use("com.app.security.RootDetector");
    RootDetector.isDeviceRooted.implementation = function() {
        return false;
    };
});
```

### Native (Android + iOS)

```javascript
// Hook exported function
Interceptor.attach(Module.findExportByName(null, "open"), {
    onEnter: function(args) {
        this.path = Memory.readUtf8String(args[0]);
    },
    onLeave: function(retval) {
        console.log("open(" + this.path + ") = " + retval);
    }
});

// Hook any address (by offset)
var base = Module.findBaseAddress("libnative.so");
var target = base.add(0x12345);
Interceptor.attach(target, {
    onEnter: function(args) {
        console.log("Function called from: " + Thread.backtrace(this.context, Backtracer.ACCURATE)
            .map(DebugSymbol.fromAddress).join('\n'));
    }
});

// Change return value
Interceptor.attach(Module.findExportByName(null, "strcmp"), {
    onLeave: function(retval) {
        if (retval.toInt32() === 0) return; // strings equal, skip
        // Force match
        retval.replace(0);
    }
});
```

### ObjC runtime (iOS)

```javascript
// Hook ObjC method
var hook = ObjC.classes.ViewController["- viewDidLoad"];
Interceptor.attach(hook.implementation, {
    onEnter: function(args) {
        console.log("viewDidLoad called");
    }
});

// Enumerate all classes
ObjC.enumerateLoadedClasses({
    onMatch: function(className) { console.log(className); },
    onComplete: function() {}
});

// Call ObjC
var NSString = ObjC.classes.NSString;
var str = NSString.stringWithString_("Hello from Frida");
```

## Objection command cheat sheet

### Generic

```bash
objection -g "com.app" explore           # start
objection -g "com.app" explore -q        # quiet (inject, no wait)
objection patchapk --source app.apk      # auto-inject Frida Gadget
objection signapk --source app.apk       # sign only

# Filesystem
env              # app data dir
ls               # list files
file download /path/to/file  # download
file upload local.txt /remote/path  # upload

# SQLite
sqlite connect /path/to/db.sqlite
.tables          # list tables
select * from users;  # query
```

### Android-only

```bash
android root disable              # bypass root detect
android sslpinning disable        # bypass SSL pinning
android hooking list classes      # enumerate classes
android hooking list class_methods com.app.Main  # enumerate methods
android hooking watch class com.app.Main  # hook all methods
android intent launch_activity com.app.MainActivity  # launch Activity
android heap search instances com.app.User  # heap search
android keystore list             # Keystore entries
```

### iOS-only

```bash
ios jailbreak disable             # bypass jailbreak detect
ios sslpinning disable            # bypass SSL pinning
ios keychain dump                 # dump Keychain
ios nsuserdefaults get            # NSUserDefaults
ios nsurlcache dump               # HTTP cache
ios cookies get                   # cookies
ios pasteboard monitor            # clipboard
ios ui dump                       # UI hierarchy
ios plist cat Info.plist          # read plist
```

## No-root / no-jailbreak deploy

### Android — Frida Gadget inject

```bash
# 1. Unpack APK
apktool d app.apk -o app_unpacked

# 2. Drop frida-gadget into lib
cp frida-gadget-17.x.x-android-arm64.so \
   app_unpacked/lib/arm64-v8a/libfrida-gadget.so

# 3. Inject System.loadLibrary("frida-gadget") in smali
# Edit main Activity onCreate or attachBaseContext

# 4. Rebuild and sign
apktool b app_unpacked -o app_patched.apk
uber-apk-signer -a app_patched.apk

# 5. Objection automation
objection patchapk --source app.apk --skip-resources
```

### iOS — Frida Gadget inject

```bash
# 1. Decrypt App Store IPA
python3 frida-ios-dump.py -u -p com.app.target

# 2. Inject FridaGadget.dylib
# Edit Mach-O Load Commands, add @executable_path/FridaGadget.dylib

# 3. Resign
codesign -f -s "Apple Development" Payload/App.app

# 4. Install via Xcode sideload or AltStore
```

## SSL pinning bypass (advanced)

### Multi-layer (Android)

```javascript
// 1. OkHttp CertificatePinner
var CertificatePinner = Java.use("okhttp3.CertificatePinner");
CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function() {};

// 2. Custom TrustManager
var TrustManagerImpl = Java.use("com.android.org.conscrypt.TrustManagerImpl");
TrustManagerImpl.verifyChain.implementation = function() { return []; };

// 3. WebView SSL Error
var SslErrorHandler = Java.use("android.webkit.SslErrorHandler");
SslErrorHandler.proceed.implementation = function() { return this.proceed(); };

// 4. Network Security Config
// Edit AndroidManifest.xml → android:networkSecurityConfig="@xml/network_security_config"
// Trust user CAs in the xml
```

### Multi-layer (iOS)

```javascript
// 1. NSURLSession
var SecTrustEvaluate = Module.findExportByName("Security", "SecTrustEvaluate");
Interceptor.replace(SecTrustEvaluate, new NativeCallback(function(trust, result) {
    Memory.writeU32(result, 1); // kSecTrustResultProceed = 1
    return 0; // errSecSuccess
}, 'int', ['pointer', 'pointer']));

// 2. Alamofire
// Hook ServerTrustManager.evaluate → always success
```

Source: Frida docs, Objection wiki, OWASP MSTG
