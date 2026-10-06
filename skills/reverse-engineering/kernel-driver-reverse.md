# Kernel-driver reverse-engineering reference

> Covers Windows/Linux kernel-driver RE, rootkit analysis, C/C++ binary pattern ID.

---

## Windows driver RE

### Driver types

| Type | Features | Analysis focus |
|------|------|---------|
| WDM (Windows Driver Model) | legacy; manual IRP management | DriverEntry → device create → Dispatch routines |
| KMDF (Kernel Mode Driver Framework) | modern; event-driven | EvtDriverDeviceAdd → Queue → I/O callbacks |
| WDF (Windows Driver Foundation) | KMDF + UMDF umbrella | look at WdfDriverCreate calls |
| Minifilter | filesystem filter driver | FltRegisterFilter → Pre/Post callbacks |

### WDM driver analysis flow

```text
1. Find DriverEntry (entry)
   - IDA auto-IDs, or search IoCreateDevice / IoCreateSymbolicLink

2. Find device name and symlink
   - IoCreateDevice → DeviceName (e.g. \Device\MyDriver)
   - IoCreateSymbolicLink → SymLink (e.g. \DosDevices\MyDriver)

3. Find Dispatch routines
   - DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchIoctl
   - this is the user-mode DeviceIoControl entry

4. Analyze IOCTL handling
   - switch(IoControlCode) dispatches functions
   - IOCTL encoding: CTL_CODE(DeviceType, Function, Method, Access)
   - Method: METHOD_BUFFERED / METHOD_IN_DIRECT / METHOD_OUT_DIRECT / METHOD_NEITHER

5. Find vulns
   - user-controlled buffer length not checked → overflow
   - METHOD_NEITHER uses user pointers directly → arbitrary R/W
   - IOCTL access not checked → unprivileged user can call
```

### IOCTL code decode

```python
# decode IOCTL code
def decode_ioctl(code):
    device_type = (code >> 16) & 0xFFFF
    access = (code >> 14) & 0x3
    function = (code >> 2) & 0xFFF
    method = code & 0x3
    
    methods = {0: "BUFFERED", 1: "IN_DIRECT", 2: "OUT_DIRECT", 3: "NEITHER"}
    access_types = {0: "ANY", 1: "READ", 2: "WRITE", 3: "READ|WRITE"}
    
    return f"DevType=0x{device_type:X} Func=0x{function:X} Method={methods[method]} Access={access_types[access]}"

# example
decode_ioctl(0x80002034)
# DevType=0x8000 Func=0x80D Method=BUFFERED Access=ANY
```

### IDA plugins

| Plugin | Use | Link |
|------|------|------|
| **Driver Buddy Reloaded** | auto-ID IOCTL, Dispatch, device names | https://github.com/VoidSec/DriverBuddyReloaded |
| **WinDbg + IDA** | kernel debug + static together | built-in |
| **FLIRT/Lumina** | ID WDK library functions | IDA built-in |

### Reference articles

- [Windows Drivers RE Methodology (VoidSec)](https://voidsec.com/windows-drivers-reverse-engineering-methodology/) — most complete WDM driver RE methodology
- [Driver Reversing 101](https://eversinc33.com/posts/driver-reversing.html) — WDM vs KMDF compare
- [Methodology of Reversing Vulnerable Killer Drivers](https://whiteknightlabs.com/2025/10/28/methodology-of-reversing-vulnerable-killer-drivers/) — vulnerable-driver analysis

---

## Linux kernel-module RE

### LKM (Loadable Kernel Module) structure

```text
Key functions:
- init_module / module_init → runs on load
- cleanup_module / module_exit → runs on unload

Key structures:
- struct file_operations → char-device open/read/write/ioctl
- struct net_device_ops → net-device ops
- struct block_device_operations → block-device ops
```

### Analysis flow

```text
1. Confirm it is a kernel module
   file module.ko → "ELF 64-bit ... relocatable" (note: relocatable, not executable)

2. Find init/exit
   readelf -s module.ko | grep -E "init_module|cleanup_module"
   or find module info in .modinfo section

3. Find file_operations
   search register_chrdev / cdev_add / misc_register
   → find fops struct → locate ioctl/read/write handlers

4. Analyze ioctl handling
   unlocked_ioctl / compat_ioctl
   → switch(cmd) dispatch

5. Find rootkit behavior
   - modify sys_call_table → syscall hook
   - modify /proc filesystem → hide processes/files
   - register netfilter hook → hide network connections
   - modify VFS layer → hide files
```

### Common rootkit techniques

| Technique | Feature | Detection |
|------|------|---------|
| syscall table hook | modify `sys_call_table` entries | compare in-memory table vs on-disk vmlinux |
| VFS hook | modify `file_operations` fn ptrs | check whether fops ptrs point outside kernel code |
| Netfilter hook | `nf_register_net_hook` | walk netfilter hook list |
| kprobe/ftrace hook | register kprobe or ftrace callback | check ftrace registration list |
| eBPF rootkit | load malicious BPF programs | `bpftool prog list` |
| DKOM | directly modify kernel objects (process list) | walk task_struct list vs /proc |

### Tools

| Tool | Use |
|------|------|
| `crash` | kernel dump analysis |
| `volatility3` | memory forensics (Linux profile) |
| `dmesg` / `journalctl` | kernel log |
| `lsmod` / `/proc/modules` | loaded-module list |
| `modinfo` | module metadata |
| `strace` | syscall trace (user-mode view) |

---

## C/C++ RE pattern ID

### Common C patterns

| Source pattern | Disasm feature |
|---------|-----------|
| `if-else` | `cmp` + `jcc` (conditional jump) |
| `switch-case` | jump table (`jmp [rax*8 + table]`) or consecutive `cmp` |
| `for` loop | `cmp` + `jl/jle` + body + `inc/add` + `jmp` back |
| `while` loop | condition at loop top |
| `do-while` | condition at loop bottom |
| function-pointer call | `call rax` or `call [reg+offset]` |
| `struct` access | `[reg+fixed offset]` (e.g. `[rdi+0x10]`) |
| `malloc` + use | `call malloc` → return stored in register → later access via that register+offset |
| string compare | `call strcmp` or `repe cmpsb` |

### C++-specific patterns

| Source pattern | Disasm feature |
|---------|-----------|
| **virtual call** | `mov rax, [rcx]` (load vtable) → `call [rax+offset]` (call virtual) |
| **constructor** | alloc memory → write vtable ptr → init members |
| **destructor** | clean members → may call `operator delete` |
| **this pointer** | first arg (rcx/rdi) is object pointer |
| **inheritance** | vtable contains parent virtuals + child overrides |
| **multiple inheritance** | object has multiple vtable ptrs (different offsets) |
| **RTTI** | `type_info` pointer before vtable |
| **exceptions** | `__cxa_throw` / `_CxxThrowException` |
| **STL containers** | `std::vector`: `{begin, end, capacity}` three-pointer struct |
| **std::string** | SSO: short strings inline, long strings heap |

### vtable RE method

```text
1. Find vtable
   - search consecutive function-pointer arrays (in .rodata or .rdata)
   - constructor `mov [rcx], offset vtable` writes the vtable pointer

2. Determine class hierarchy
   - offset -8 before vtable is usually RTTI pointer (if not stripped)
   - multiple vtables sharing the first few entries → inheritance

3. Label virtuals
   - vtable[0] is usually destructor (or deleting destructor)
   - then by offset: vtable[1] = func1, vtable[2] = func2...

4. IDA ops
   - create a struct at the vtable address (each field a fn ptr)
   - comment `call [rax+offset]` with the virtual being called
```

### Struct recovery

```text
Method 1: infer from access pattern
  mov eax, [rdi+0x00]  → field_0: int/ptr (4/8 bytes)
  mov ecx, [rdi+0x08]  → field_8: int/ptr
  movss xmm0, [rdi+0x10] → field_10: float

Method 2: infer from sizeof
  call malloc(0x30) → struct size 0x30 (48 bytes)
  
Method 3: infer from constructor
  constructor inits all fields → types and offsets are obvious

Method 4: IDA "Create struct"
  select access pattern → Edit → Struct → Create struct from selection
```

---

## Common compiler fingerprints

| Compiler | ID features |
|--------|---------|
| MSVC | `_security_cookie` check, `__fastcall` calling convention, Rich Header |
| GCC | `__stack_chk_fail`, `-fstack-protector`, `.note.GNU-stack` |
| Clang/LLVM | GCC-like but different opt patterns; `__asan_*` (if sanitizer on) |
| MinGW | GCC features + Windows API calls |
| AOSP Clang | Android-specific `__android_log_print`, PGO marks |

### Opt-level ID

| Opt level | Features |
|---------|------|
| -O0 | lots of redundant mov; every var on stack; no inlining |
| -O1 | basic opts; some vars in registers |
| -O2 | loop unroll, function inline, tail-call opt |
| -O3 / -Os | aggressive inline, vectorize (SIMD), hard to read |
| PGO | hot-path opt; cold code split to `.text.cold` |
| LTO | cross-module inline; global DCE |

---

## Kernel debug environments

### Windows

```text
Debugger: WinDbg Preview
Connect: network debug (preferred) or serial

Debuggee setup:
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.x.x port:50000

Debugger connect:
WinDbg → File → Attach to Kernel → Net → Port:50000 Key:xxx

Common commands:
!analyze -v          # auto-analyze crash
lm                   # list loaded modules
!drvobj \Driver\xxx  # view driver object
dt nt!_DRIVER_OBJECT # display struct
bp module!function   # breakpoint
```

### Linux

```text
Debugger: GDB + QEMU or kgdb

QEMU kernel debug:
qemu-system-x86_64 -kernel bzImage -s -S ...
gdb vmlinux -ex "target remote :1234"

Common commands:
info threads         # kernel threads
lx-symbols           # load kernel symbols (needs scripts/gdb/)
p init_task          # inspect init process
lx-dmesg             # kernel log
```

---

## Agent action anchors (Issue #65 U–AV)

Aligned with `references/nonpe-format-cookbook.md` §5 (short table; does not replace the flow above):

| ID | Action | Evidence |
|----|------|----------|
| AG | `DriverEntry` short → scan non-empty `MajorFunction` slots; prefer DEVICE_CONTROL/CREATE | `E-driver-irp-handlers` |
| AH | build IOCTL control-code → handler table and METHOD_* | `E-driver-ioctl` |
| AI | suspected BYOVD: compare against public vulnerable-driver lists; record name/hash/signature and call intent; **do not write exploit steps** | `E-driver-byovd` |

## References

| Resource | Notes | Link |
|------|------|------|
| VoidSec driver RE methodology | full Windows WDM driver analysis flow | https://voidsec.com/windows-drivers-reverse-engineering-methodology/ |
| Elastic Rootkit series | Linux rootkit taxonomy + detection | https://security-labs.elastic.co/security-labs/linux-rootkits-1-hooked-on-linux |
| Driver Buddy Reloaded | IDA driver-analysis plugin | https://github.com/VoidSec/DriverBuddyReloaded |
| LOLDrivers | known vulnerable-driver list | https://www.loldrivers.io/ |
| Windows Driver Samples | Microsoft official driver samples | https://github.com/microsoft/Windows-driver-samples |
| Linux Kernel Module Programming | kernel-module development tutorial | https://sysprog21.github.io/lkmpg/ |
| Trail of Bits - Devirtualizing C++ | vtable RE method | https://blog.trailofbits.com/2017/02/13/devirtualizing-c-with-binary-ninja/ |
