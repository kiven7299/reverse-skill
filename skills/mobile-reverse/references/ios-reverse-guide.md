# iOS reverse-engineering notes

## IPA acquire and decrypt

```bash
# Download from App Store
ipatool search "Target App"
ipatool purchase -b com.target.app
ipatool download -b com.target.app -o app.ipa

# Extract installed app from device
# Jailbroken device
scp root@device:/private/var/containers/Bundle/Application/*/Target.app .

# Decrypt (App Store binary is encrypted FAT)
# frida-ios-dump (preferred)
python3 dump.py com.target.app -o decrypted.ipa

# Clutch
Clutch -i  # list installed
Clutch -d 1  # decrypt #1

# dumpdecrypted
DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /path/to/App
```

## Mach-O analysis

```bash
# Basics
otool -l TargetBinary | grep crypt    # encryption state
otool -L TargetBinary                 # dylib deps
otool -hv TargetBinary                # header
jtool2 --pages TargetBinary           # pages

# Thin fat binary
lipo -info TargetBinary
lipo TargetBinary -thin arm64 -output TargetBinary_arm64

# Symbols
nm -g TargetBinary                    # exports
nm -a TargetBinary                    # all
swift-demangle <mangled_name>         # Swift demangle

# class-dump
class-dump -H TargetBinary -o headers/
# ObjC class/method decls into headers/
```

## Objective-C runtime

```text
Message dispatch:
objc_msgSend(id self, SEL op, ...)  →  dynamic method dispatch
  ↓
Runtime lookup:
1. Class method cache
2. Class method list
3. Superclass walk
4. +resolveInstanceMethod / +resolveClassMethod
5. forwardingTargetForSelector
6. methodSignatureForSelector + forwardInvocation
```

### Frida ObjC hook

```javascript
// Instance method
var hook = ObjC.classes.ClassName["- instanceMethod:"];
Interceptor.attach(hook.implementation, {
    onEnter: function(args) {
        // args[0] = self, args[1] = selector, args[2+] = method args
        console.log("self: " + new ObjC.Object(args[0]));
        console.log("arg: " + args[2].toInt32());
    }
});

// Class method
var hook = ObjC.classes.ClassName["+ classMethod:"];
Interceptor.attach(hook.implementation, { ... });

// Call ObjC
var NSString = ObjC.classes.NSString;
var str = NSString.stringWithString_("test");
console.log(str.UTF8String());
```

## Swift reverse

```text
Swift name mangling:
$s10ModuleName5ClassC6method3argSi_tF
  │ │         │     │ │      │  │   └─ arg type
  │ │         │     │ │      │  └───── return type  
  │ │         │     │ │      └──────── arg name
  │ │         │     │ └─────────────── method name
  │ │         │     └──────────────── class (len+name)
  │ │         └────────────────────── module
  │ └──────────────────────────────── identifier
  └────────────────────────────────── global

Tools: swift-demangle, Hopper (auto demangle)
```

## Jailbreak-detection bypass

```text
Detection classes:

1. Filesystem:
   □ /Applications/Cydia.app
   □ /var/lib/apt/
   □ /bin/bash
   □ /usr/sbin/sshd
   → Hook NSFileManager.fileExistsAtPath:

2. Sandbox escape:
   □ fork() succeeds (forbidden in sandbox)
   □ system()
   → Hook fork → return -1

3. Dyld inject:
   □ _dyld_get_image_count above limit
   → Cap the return value

4. Scheme:
   □ cydia:// URL Scheme
   → Hook UIApplication.canOpenURL:

5. sysctl:
   □ CTL_KERN/KERN_PROC/KERN_PROC_PID → kinfo_proc
   → Hook sysctl → clear p_flag P_TRACED
```

### Unified Frida bypass

```javascript
// File check bypass
var NSFileManager = ObjC.classes.NSFileManager;
var defaultManager = NSFileManager.defaultManager();
Interceptor.attach(defaultManager["- fileExistsAtPath:"].implementation, {
    onLeave: function(retval) {
        var path = ObjC.Object(args[2]).toString();
        if (path.includes("Cydia") || path.includes("apt") || 
            path.includes("sshd") || path.includes("bash")) {
            retval.replace(0); // false
        }
    }
});

// fork bypass
Interceptor.replace(Module.findExportByName(null, "fork"), 
    new NativeCallback(function() { return -1; }, 'int', []));

// dyld bypass
var _dyld_get_image_count = Module.findExportByName(null, "_dyld_get_image_count");
Interceptor.attach(_dyld_get_image_count, {
    onLeave: function(retval) {
        if (retval.toInt32() > 200) retval.replace(200);
    }
});
```

## Protection bypass list

| Control | iOS bypass |
|------|-------------|
| App Store encrypt | frida-ios-dump / Clutch |
| SSL pinning | Objection `ios sslpinning disable` / SSL Kill Switch 2 |
| Jailbreak detect | Objection `ios jailbreak disable` / custom Frida hook |
| Anti-debug (PT_DENY_ATTACH) | Inject after Frida spawn / debugserver |
| Integrity | Hook MAC check / code-sign verify |
| Anti-inject | Strip Mach-O `__RESTRICT` segment |
| Swift obfuscation | swift-demangle + LLM semantic restore |
| Screenshot protect | Hook UIScreen.mainScreen.snapshotViewAfterScreenUpdates |

Source: OWASP MSTG, frida-ios-dump, The iPhone Wiki
