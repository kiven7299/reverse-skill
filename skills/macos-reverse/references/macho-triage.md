# Mach-O triage

```bash
file ./app
otool -hv ./app
otool -l ./app | head
codesign -d --entitlements :- ./app
```

Watch: `com.apple.security.*` entitlements, Library Validation, flags that disable library injection.
