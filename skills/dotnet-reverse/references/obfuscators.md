# .NET obfuscator deobfuscation

ID, unpack, anti-tamper bypass for mainstream .NET obfuscators. Core tools: **de4dot** (auto most packers) + **dnSpyEx** (manual patch) + **dnlib** (scripted).

## Decision table

| Obfuscator | de4dot type | Fingerprint | Auto unpack | Manual notes |
|--------|-------------|---------|---------|---------|
| ConfuserEx 1.x/2.x | `cfze` | anti-tamper, CF flatten, string encrypt, anti-debug | ✅ usually | Newer: patch anti-tamper first |
| ConfuserEx 3.x / forks | `cfze` | same + custom protector | ⚠️ partial | runtime dump / dnlib |
| SmartAssembly | `sa` | string encode, resource compress, call hide | ✅ auto | resource decompress |
| Babel.NET | `babel` | method-body encrypt, CF, strings | ✅ auto | — |
| Eazfuscator.NET | `eaz` | string/resource encrypt, expr obfuscation | ⚠️ partial | string decryptor |
| .NET Reactor | `reactor` | necrobit (code encrypt) + anti-tamper | ⚠️ hard on new | dump + rebuild metadata |
| Themida .NET | — | outer pack + virtualize | ❌ de4dot fails | dump memory; native path |
| Agile.NET / CliSecure | `agile` | method-body encrypt | ✅ auto | — |

## de4dot standard usage

```powershell
# Auto-detect (enough most of the time)
de4dot target.exe -o target-clean.exe

# Explicit type (auto-detect failed)
de4dot --type cfze target.exe -o target-clean.exe

# Probe packer type
de4dot --detect target.exe

# Batch
de4dot *.exe

# Strings only; leave CF (minimal)
de4dot --strtyp delegate --strtok METHOD_TOKEN target.exe
```

de4dot `--strtyp` / `strtok`: decrypt strings only (decrypt method token), keep original CF. Use when you want plaintext strings without touching anti-tamper.

---

## ConfuserEx (most common)

### Fingerprints

- Entry `<module>` class with `[MethodImpl(NoInlining)]` anti-tamper
- Many `Dictionary<string, T>` string-decryptor calls
- CF flatten (switch dispatch + state)
- Nested `.cmp` compressed resources
- dnSpyEx C# view: garbled names (`\uXXXX` or junk); bodies full of `int num = ...; switch(num)`

### Unpack flow

```powershell
# 1. Standard unpack
de4dot target.exe -o target-clean.exe

# 2. If de4dot says "unknown" or output will not open → new/forked ConfuserEx
#    Confirm anti-tamper:
dnSpyEx open → Module .cctor or Main integrity check
```

### Anti-tamper bypass (common on new ConfuserEx)

ConfuserEx `anti tamper` hashes method bodies at runtime and crashes on edit. de4dot handles old builds; new builds need manual work:

```text
Method A — dnSpyEx patch the checker:
  1. Find anti-tamper method (usually called from <module> static ctor)
  2. IL edit: body → ret
  3. Save → feed de4dot

Method B — runtime dump:
  1. MegaDumper / ExtremeDumper dump in-memory assembly
  2. Dump is already decrypted; de4dot leftover cleanup
```

### After CF restore

de4dot restores flattened switch dispatch to if/while. If incomplete (leftover state machine), re-run de4dot or follow IL by hand.

---

## SmartAssembly

```powershell
de4dot --type sa target.exe -o target-clean.exe
```

Fingerprints:
- Strings via `SmartAssembly.Runtime.Strong*`
- Resource compress (`{assembly}.Resources`)
- Hidden calls (`ProcessCaller` / indirect call)

de4dot compatibility is best here; usually one-shot.

---

## .NET Reactor (necrobit)

**.NET Reactor** **necrobit** encrypts real method bodies into resources, decrypts at runtime; original bodies are stubs. de4dot works on old versions; 4.x+ often fails.

```text
When de4dot fails:
1. Run the program (dotnet target.exe or double-click)
2. MegaDumper / ExtremeDumper dump process → decrypted assembly
3. de4dot leftover obfuscation on the dump
4. If metadata is broken, rebuild with dnlib (see common-workflow.md)
```

---

## Manual string-decryptor extract

Obfuscators encrypt strings and restore via a decrypt method. de4dot usually finds it; if not:

```text
1. dnSpyEx find decrypt method (often static string Decrypt(int) or Decrypt(string, int))
   - Fingerprint: many calls, numeric constant args, returns string
2. Note method token (e.g. 0x06000012)
3. Point de4dot at it:
   de4dot --strtyp delegate --strtok 0x06000012 target.exe -o target-clean.exe
```

If the decrypt method itself is CF-flattened, unpack CF first, then locate it.

## Common anti-debug

| Technique | Where | Bypass |
|------|------|------|
| `Debugger.IsAttached` | any method | IL `ldc.i4.0; ret` or patch getter |
| `Debugger.IsLogging` | — | same |
| Timing (`DateTime.Now` delta) | method entry | patch the compare |
| `CheckRemoteDebuggerPresent` P/Invoke | — | nop the call |
| Exception-driven CF (try/catch path) | main logic | do not nop; analyze catch real path |

> .NET anti-debug is simpler than native — mostly managed APIs; one IL line in dnSpyEx.

## Fallback when de4dot fails

1. **de4dot --detect** vs the table above
2. **Runtime dump** (MegaDumper / ExtremeDumper / Process Hacker module export)
3. **dnlib script** (see common-workflow.md dnlib)
4. **Dynamic first**: run, break at decrypt, read plaintext without unpacking

Community: Washi *misconceptions-about-dotnet* (IL pitfalls), Kanxue .NET reverse board, Guided Hacking *Top 5 .NET RE Tools*.
