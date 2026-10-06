# EDR hook survey cheat sheet

> Authorized red team / adversary emulation / own-product tests only. Forbidden on unauthorized targets.

Mainstream EDR / AV user-mode and kernel monitor points, for red-team recon: "what to handle".

## 1. Mainstream EDR fingerprints and hook patterns

| Vendor / product | User-mode components | Kernel drivers | Main monitor surface |
|------------------|----------------------|----------------|----------------------|
| CrowdStrike Falcon | `CSFalconService.exe`, `CSAgent.sys` injected into targets | `CSAgent.sys`, `CSBoot.sys` | Heavy kernel callbacks + ETW-TI; few user-mode hooks (cloud lookup) |
| Microsoft Defender for Endpoint (MDE) | `MsMpEng.exe`, `MpClient.dll` | `WdFilter.sys`, `WdBoot.sys`, `WdNisDrv.sys` | AMSI + ETW-TI + ntdll inline hook + full kernel callbacks |
| SentinelOne | `SentinelAgent.exe`, `SentinelHelperService.exe` | `SentinelMonitor.sys`, `SentinelDeviceControl.sys` | Heavy ntdll user-mode hooks + kernel callbacks + own ETW provider |
| Elastic Defend (ex Endpoint Security) | `elastic-endpoint.exe` | `elastic-endpoint-driver.sys` | Mostly ETW + light ntdll hooks, Elastic Agent upload |
| ESET | `ekrn.exe`, `eamsi.dll` | `eamonm.sys`, `epfwwfp.sys` | Very many user-mode hooks (NtCreateFile / NtOpenProcess, etc.) |
| Sophos Intercept X | `SophosFileScanner.exe`, `SophosNtpService.exe` | `SophosED.sys`, `hmpalert.sys` | ntdll hooks + HMPA memory protect + kernel callbacks |
| Kaspersky | `avp.exe`, `klif.sys` | `klif.sys`, `klhk.sys` | Heavy user-mode hooks + KLIF minifilter + network filter driver |
| Trend Micro Apex One | `TmListen.exe`, `TmCCSF.dll` | `tmcomm.sys`, `tmactmon.sys` | User-mode hooks + behavior-monitor driver |
| Carbon Black | `RepMgr.exe`, `RepWAV.exe` | `ParityDriver.sys` | Kernel-callback-heavy + ETW |

### Quick fingerprint script

```powershell
$edrSigs = @{
    'CSAgent'           = 'CrowdStrike Falcon'
    'SentinelAgent'     = 'SentinelOne'
    'elastic-endpoint'  = 'Elastic Defend'
    'ekrn'              = 'ESET'
    'MsMpEng'           = 'Microsoft Defender'
    'SophosFileScanner' = 'Sophos Intercept X'
    'avp'               = 'Kaspersky'
    'TmListen'          = 'Trend Micro Apex One'
    'cb'                = 'Carbon Black'
}

Get-Process | ForEach-Object {
    foreach ($k in $edrSigs.Keys) {
        if ($_.ProcessName -match $k) {
            "[+] $($edrSigs[$k]) detected: $($_.ProcessName) (PID $($_.Id))"
        }
    }
}

Get-ChildItem 'C:\Windows\System32\drivers\*.sys' |
    Where-Object { $_.Name -match 'CSAgent|Sentinel|elastic|eam|WdFilter|Sophos|klif|tmcomm|Parity' } |
    Select-Object Name, VersionInfo
```

## 2. High-priority user-mode ntdll hooks

EDR almost always hooks these `ntdll.dll` exports (grouped by ATT&CK behavior):

| Function | Behavior watched | ATT&CK |
|----------|------------------|--------|
| `NtCreateThreadEx` | Remote-thread inject, QueueUserAPC inject | T1055.002 / T1055.004 |
| `NtAllocateVirtualMemory` | shellcode alloc RWX | T1055 |
| `NtAllocateVirtualMemoryEx` | Cross-process alloc (Win10+ new API) | T1055 |
| `NtProtectVirtualMemory` | Change page perms RW→RX | T1055 |
| `NtWriteVirtualMemory` | Cross-process write shellcode | T1055.012 |
| `NtMapViewOfSection` | Section-based inject (Process Doppelganging / Ghosting) | T1055.013 |
| `NtCreateSection` | With MapViewOfSection | T1055.013 |
| `NtOpenProcess` | Open target process handle | T1057 |
| `NtQueueApcThread` / `NtQueueApcThreadEx` | APC inject | T1055.004 |
| `NtCreateProcess` / `NtCreateProcessEx` / `NtCreateUserProcess` | Create child (incl. PPID spoof) | T1106 |
| `NtSetContextThread` | Rewrite thread context (thread-hijack inject) | T1055.003 |
| `NtResumeThread` | Resume after inject | T1055 |
| `NtQuerySystemInformation` | Enum processes / drivers / handles | T1057 / T1082 |
| `NtAdjustPrivilegesToken` | Privesc SeDebugPrivilege etc. | T1134 |
| `NtLoadDriver` | Load kernel driver (BYOVD) | T1543.003 |

### Confirm a hook exists

```powershell
# Simple: disasm-diff disk ntdll vs in-process ntdll
# 1. Grab disk ntdll
copy C:\Windows\System32\ntdll.dll C:\temp\ntdll_clean.dll

# 2. In windbg attach any process, dump live ntdll .text
# .writemem c:\temp\ntdll_live.bin ntdll!.text L?<size>

# 3. IDA / radare2 disasm NtAllocateVirtualMemory; clean should be:
#    mov r10, rcx
#    mov eax, <SSN>
#    test byte ptr [...]
#    jne ...
#    syscall
#    ret
# If the first insn is jmp <some addr>, it is hooked
```

## 3. Kernel callback monitor points

Common EDR-registered kernel callbacks (all can be unregistered via the BYOVD path in `attack-chain`, at high cost):

| API | When the callback fires | Defender use |
|-----|-------------------------|--------------|
| `PsSetCreateProcessNotifyRoutineEx` | Process create / exit | Intercept suspicious child processes |
| `PsSetCreateThreadNotifyRoutine` | Thread create / exit | Detect remote-thread inject |
| `PsSetLoadImageNotifyRoutine` | DLL / EXE load into any process | Module integrity / unsigned intercept |
| `CmRegisterCallback` / `CmRegisterCallbackEx` | Registry ops | Persistence detect |
| `ObRegisterCallbacks` | `OpenProcess` / `OpenThread` handle requests | Block LSASS handle grab (T1003.001) |
| `MmRegisterPhysicalMemoryCallback` | Physical-memory map | Anti-DMA / memory forensics |
| `IoRegisterFsRegistrationChange` | Filesystem register | Minifilter coordination |
| `KeRegisterNmiCallback` | NMI (rare EDR use) | Exception monitor |
| `EtwRegister` (kernel side) | Kernel ETW report | Coexists with ETW-TI |

### Enumerate registered callbacks in windbg

```text
0: kd> dx -r1 nt!PspCreateProcessNotifyRoutine
0: kd> dx -r1 nt!PspCreateThreadNotifyRoutine
0: kd> dx -r1 nt!PspLoadImageNotifyRoutine

0: kd> !object \Callback
0: kd> !object \Callback\ProcessObject
```

Or PChunter / DRVHV for a user-visible callback list.

## 4. Static dump of the hook table (IDA + windbg)

### Flow A: single-process compare

```text
1. Find a process already injected with the EDR user-mode component (any live process)
2. windbg attach (-pn target.exe)
3. lm m ntdll  → module base
4. .writemem c:\temp\ntdll_live.bin ntdll+0x0 L?<image size>
5. Copy C:\Windows\System32\ntdll.dll to c:\temp\ntdll_disk.dll
6. Load both in IDA, jump to NtAllocateVirtualMemory:
     - disk: standard prologue
     - live: first insn jmp <0x7FFE000000xx>
7. Follow the jmp target → that is the EDR trampoline; dump it
8. Inside the trampoline, see which DLL it lands in; confirm the EDR module name
```

### Flow B: batch hook-table generation

Use `HookHunter` or a custom script:

```powershell
# pseudo workflow; see scripts mentioned in references
$disk = Get-Content C:\Windows\System32\ntdll.dll -Encoding Byte
$live = # via OpenProcess + ReadProcessMemory
# compare first 16 bytes of each export in .text
```

## 5. pe-sieve auto-detect

`pe-sieve` is first choice for reconning EDR hooks and implant self-check:

```powershell
# basic scan
pe-sieve64.exe /pid 1234

# recommended combo (shellcode + hook detect)
pe-sieve64.exe /pid 1234 /shellc 3 /modules 3 /imp 3 /data 3 /dir hooks_dump

# key flags:
#   /shellc N    shellcode scan level (0-3)
#   /modules N   module integrity (0-3)
#   /imp N       IAT hook check
#   /data N      data-section scan
#   /dir <path>  dump output dir
```

Output under `hooks_dump/<pid>.<name>/` produces `*.tag` files listing hook addresses:

```text
modified_modules.tag example:
71f10000;ntdll.dll
71f1a3b0;hook;jmp_far
71f1c020;hook;jmp_near
```

Feed those RVAs straight into IDA for follow-up.

### Embed pe-sieve in the implant (self-check)

In the field, compile `pe-sieve` as a lib (`libpe-sieve`) so the implant self-checks at start: if ntdll is hooked, trigger unhook; if *you* are hooked, be careful — maybe a sandbox.

## 6. API Monitor v2 dynamic watch

API Monitor v2 (Rohitab) is good in lab for seeing when/where EDR inserts hooks:

```text
1. Start API Monitor v2 (admin)
2. API Filter check:
     - NT Native API → Memory Management
     - NT Native API → Process and Thread
     - Windows Defender / AMSI (if visible)
3. Monitor New Process → pick the implant test sample
4. Watch:
     - NtAllocateVirtualMemory call order
     - whether an EDR DLL intermediates
5. Modules tab: which EDR DLLs were LoadLibrary-injected
```

## 7. Common EDR DLLs (user-mode)

| DLL | Vendor | Notes |
|-----|--------|-------|
| `umppc*.dll` | Microsoft Defender | MpClient userland |
| `mpoav.dll` | Microsoft Defender | AMSI provider |
| `aswAMSI.dll` | Avast | AMSI provider |
| `eamsi.dll` | ESET | AMSI provider |
| `IDPMServiceClient.dll` | Sophos | HMPA inject |
| `klsihk64.dll` | Kaspersky | Injected into target process |
| `CrowdStrike.Sensor.dll` | CrowdStrike | Older; newer mainly kernel |
| `SentinelInjection64.dll` | SentinelOne | User-mode inject |
| `TmUmEvt64.dll` | Trend Micro | Behavior monitor |

Confirm the target EDR, then decide which DLL to reverse for the hook table.

## References

- pe-sieve: <https://github.com/hasherezade/pe-sieve>
- HollowsHunter: <https://github.com/hasherezade/hollows_hunter>
- API Monitor v2: <http://www.rohitab.com/apimonitor>
- MITRE ATT&CK T1562: <https://attack.mitre.org/techniques/T1562/>
- MITRE ATT&CK T1055: <https://attack.mitre.org/techniques/T1055/>
- ired.team EDR notes: <https://www.ired.team/offensive-security/defense-evasion>

## Routing callback

After the hook survey, return to Step 3 in `SKILL.md`, pick a bypass combo, then execute `references/unhook-techniques.md` and `references/telemetry-blinding.md`.
