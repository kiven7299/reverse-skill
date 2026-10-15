---
name: native-instrumentation
description: >
  Native C process instrumentation on Android without Frida.
  Covers all techniques for reading, patching, and hooking a running process
  from outside or inside using compiled C tools and Linux procfs primitives.
  Use when Frida is detected/blocked, when the app has integrity checks that
  reject repackaging, or when lower detectability is required.
  Trigger keywords: native hook, mem_patch, process_vm_readv, /proc/pid/mem,
  ptrace inject, GOT hook, inline patch, no-frida, C instrumentation,
  memory patch, heap patch, code patch, ARM64 patch, native bypass.
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the task targets a device/app already in scope (`work/<case>/scope.md`)
3. `NEXT`: verify tool availability (`../tool-index.md` and `../../TOOLS.md`); rebuild patchers from workspace source
4. `NEXT`: read the canonical reference workspace for real implementations:
   `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\`
   - `_methodology/07_native_c_hooking.md` — delivery mechanics, decision matrix
   - `_methodology/02_dynamic_analysis.md` — read-only observation-first methodology
   - `_tools/native_hooks/mem_patch/mem_patch.c` — generic patcher
   - `MB_Bank/native-c/` — 39 real C tools covering every technique below
5. `ACT`: choose a delivery mechanic, write/adapt a C tool, cross-compile, push, execute

---

# Native C Process Instrumentation (no-Frida)

## What this skill is

A general-purpose methodology for **instrumenting a running Android process using compiled C programs** instead of Frida. The C tools interact with the target process through Linux kernel interfaces (`/proc/<pid>/maps`, `/proc/<pid>/mem`, `process_vm_readv`/`writev`, `ptrace`) to:

- **Read** process memory (diagnostic scanning, string search, verification)
- **Patch code** (replace instructions: NOP functions, force branches, change return values)
- **Patch data** (flip config strings/flags in heap memory)
- **Hook functions** (GOT slot redirection, inline trampoline)
- **Inject code** (ptrace + dlopen a `.so` with a constructor into the running process)

All patches are **transient** — they exist only in the process's virtual memory and die when the process exits. No APK files are modified.

## When to use this skill

| Condition | Why native C, not Frida |
|---|---|
| App scans `/proc/self/maps` for `frida-agent` | No frida strings in memory |
| App checks `/proc/net/tcp` for port 27042 | No listening port |
| App enumerates threads for `gum-js-loop` | No JS runtime thread |
| App hashes its own `.so`/dex (server integrity) | Memory-only patch, files unchanged |
| Need lowest detectable footprint | External write leaves changed bytes only |
| Must run code inside the app (Java objects, signer reuse) | ptrace+dlopen injection |

## When NOT to use

- App has no anti-Frida and no integrity checks → use Frida, it's faster to iterate
- Pure Java-layer analysis → use Frida's Java.perform or Xposed
- Need rapid prototyping / REPL → Frida's JS console is better for exploration

---

## PART A — Core Linux primitives

Every technique below builds on these four kernel interfaces. Understand them first.

### A1. `/proc/<pid>/maps` — Process memory layout

```
7b1a200000-7b1a400000 r--p 00000000 fe:20 12345  /data/app/.../lib/arm64/libtarget.so
7b1a400000-7b1a800000 r-xp 00200000 fe:20 12345  /data/app/.../lib/arm64/libtarget.so
7b1a800000-7b1a900000 rw-p 00600000 fe:20 12345  /data/app/.../lib/arm64/libtarget.so
```

Fields: `start-end perms file_offset dev inode pathname`

Key rules:
- **Load base** = the mapping where `file_offset == 0` (usually `r--p`). All virtual addresses (VA) from IDA/Ghidra are offsets from this base.
- **RX segment** (`r-xp`) contains executable code — this is where you read/patch instructions.
- **RW segment** (`rw-p`) contains GOT, `.data`, `.bss` — this is where you read/patch GOT slots and writable data.
- Anonymous regions (`[anon:libc_malloc]`, `[anon:dalvik-...]`, blank) contain heap data.

```c
// Find load base of a library in a process
static uintptr_t find_base(pid_t pid, const char *lib) {
    char path[64], line[1024];
    snprintf(path, sizeof path, "/proc/%d/maps", pid);
    FILE *f = fopen(path, "r");
    if (!f) return 0;
    uintptr_t base = 0;
    while (fgets(line, sizeof line, f)) {
        if (!strstr(line, lib)) continue;
        unsigned long long lo;
        unsigned long long foff;
        if (sscanf(line, "%llx-%*llx %*s %llx", &lo, &foff) == 2 && foff == 0) {
            base = (uintptr_t)lo;
            break;
        }
    }
    fclose(f);
    return base;
}
```

### A2. `/proc/<pid>/mem` — File-based memory I/O

Seek to any virtual address and read/write. Requires same-UID or root (CAP_SYS_PTRACE).

```c
int fd = open("/proc/<pid>/mem", O_RDWR);

// Read 8 bytes at address
unsigned char buf[8];
pread(fd, buf, 8, (off_t)address);

// Write patch bytes
unsigned char patch[] = {0x00, 0x00, 0x80, 0x52, 0xC0, 0x03, 0x5F, 0xD6};
pwrite(fd, patch, sizeof patch, (off_t)address);

close(fd);
```

**Important**: `pwrite` on `/proc/<pid>/mem` bypasses page permissions — you can write to `r-xp` code pages without `mprotect`. The kernel handles it. This is the primary mechanism for code patching.

### A3. `process_vm_readv` / `process_vm_writev` — Kernel cross-process I/O

Faster than `/proc/mem` for bulk operations (no fd, no seek). Preferred for scanning large regions.

```c
#include <sys/uio.h>

// Read remote memory into local buffer
struct iovec local  = { .iov_base = buf,              .iov_len = size };
struct iovec remote = { .iov_base = (void *)address,  .iov_len = size };
ssize_t n = process_vm_readv(pid, &local, 1, &remote, 1, 0);

// Write local buffer to remote memory
ssize_t n = process_vm_writev(pid, &local, 1, &remote, 1, 0);
```

### A4. `ptrace` — Process control and register manipulation

Used only for SO injection (delivery method 3). Attaching sets `TracerPid` which some shields detect.

```c
ptrace(PTRACE_ATTACH, tid, 0, 0);    // stop the thread
waitpid(tid, &status, 0);            // wait for stop

// Read/write registers (ARM64)
struct iovec iov = { &regs, sizeof regs };
ptrace(PTRACE_GETREGSET, tid, (void*)NT_PRSTATUS, &iov);  // save
ptrace(PTRACE_SETREGSET, tid, (void*)NT_PRSTATUS, &iov);  // modify

ptrace(PTRACE_CONT, tid, 0, 0);      // resume
ptrace(PTRACE_DETACH, tid, 0, 0);    // release
```

---

## PART B — Techniques (what you can do)

### B1. Code patching — Replace instructions in a loaded library

The most common operation. Replace a function's prologue or a branch instruction.

**Pattern 1: NOP a function (make it return 0 immediately)**
```c
// ARM64: mov w0, #0; ret
unsigned char patch[8] = {
    0x00, 0x00, 0x80, 0x52,  // MOV W0, #0
    0xC0, 0x03, 0x5F, 0xD6   // RET
};
uintptr_t addr = base + offset_from_ida;
pwrite(fd, patch, 8, (off_t)addr);
```

**Pattern 2: Force a conditional branch (make it unconditional)**
```c
// Change B.LE → unconditional B
unsigned char patch[4] = {0x01, 0x00, 0x00, 0x14};  // B +4
pwrite(fd, patch, 4, (off_t)(base + branch_offset));
```

**Pattern 3: Bare RET (kill a function, no return value change)**
```c
unsigned char ret[4] = {0xC0, 0x03, 0x5F, 0xD6};  // RET
pwrite(fd, ret, 4, (off_t)(base + offset));
```

**Always verify before and after:**
```c
unsigned char before[8];
pread(fd, before, 8, (off_t)addr);
// Check before[] matches expected prologue — safety against wrong offset
pwrite(fd, patch, sizeof patch, (off_t)addr);
unsigned char after[8];
pread(fd, after, 8, (off_t)addr);
// Confirm after[] matches patch — verify write succeeded
```

### B2. Heap/data patching — Modify runtime strings and config values

Scan anonymous `rw-p` regions for known strings, patch in-place. Useful for flipping config flags, disabling feature gates, or corrupting detection results.

```c
// Scan all anonymous rw-p regions of a process
// For each region: read with process_vm_readv, search for needle, patch with pwrite

static int scan_and_patch(pid_t pid, const char *needle, size_t needle_len,
                          size_t patch_offset, const void *patch, size_t patch_len) {
    char maps_path[64];
    snprintf(maps_path, sizeof maps_path, "/proc/%d/maps", pid);
    FILE *f = fopen(maps_path, "r");
    char line[512];
    int hits = 0;

    while (fgets(line, sizeof line, f)) {
        uint64_t start, end;
        char perms[8], name[256] = {0};
        sscanf(line, "%lx-%lx %7s %*s %*s %*s %255[^\n]", &start, &end, perms, name);

        // Only anonymous rw-p regions (heap, new-space, config buffers)
        if (perms[0] != 'r' || perms[1] != 'w') continue;
        if (strchr(name, '/')) continue;  // skip file-mapped
        if (strstr(name, "stack") || strstr(name, "vdso")) continue;

        uint64_t len = end - start;
        if (len > 8 * 1024 * 1024) len = 8 * 1024 * 1024;  // cap per region
        unsigned char *buf = malloc(len);
        struct iovec lo = {buf, len}, ro = {(void*)start, len};
        ssize_t got = process_vm_readv(pid, &lo, 1, &ro, 1, 0);
        if (got <= 0) { free(buf); continue; }

        for (ssize_t i = 0; i + (ssize_t)needle_len <= got; i++) {
            if (memcmp(buf + i, needle, needle_len) == 0) {
                uint64_t addr = start + i + patch_offset;
                int wfd = open("/proc/...", O_RDWR);  // open /proc/<pid>/mem
                pwrite(wfd, patch, patch_len, (off_t)addr);
                close(wfd);
                hits++;
            }
        }
        free(buf);
    }
    fclose(f);
    return hits;
}
```

**Daemon mode**: for apps that parse configs repeatedly or respawn processes, run in a tight poll loop:
```c
for (;;) {
    scan_all_pids();    // pidof <package> → scan each pid
    usleep(10000);      // 10ms poll
}
```

### B3. GOT hooking — Redirect function calls via the Global Offset Table

Replace a GOT slot to redirect calls through a trampoline. Works for any PLT-resolved function (`send`, `connect`, `write`, `SSL_write`, etc.).

```c
// 1. Find the GOT slot address (from IDA: .got section, or readelf -r)
uintptr_t got_slot = rx_base + got_offset;

// 2. Read current (real) function pointer
unsigned long real_addr;
pread(fd, &real_addr, 8, (off_t)got_slot);

// 3. Write a trampoline into spare RW space (end of .data segment)
uintptr_t tramp = rw_base + spare_offset;
unsigned char code[16];
// ARM64: ldr x16, #8; br x16; .quad real_addr
code[0..3] = 0x58000050;  // ldr x16, [pc, #8]
code[4..7] = 0xd61f0200;  // br x16
memcpy(code + 8, &real_addr, 8);
pwrite(fd, code, 16, (off_t)tramp);

// 4. Overwrite GOT slot to point to trampoline
pwrite(fd, &tramp, 8, (off_t)got_slot);
```

The trampoline can be extended to increment a counter, log, or branch to different behavior before jumping to the real function.

### B4. Read-only diagnostics — Scan without modifying

Always start with read-only observation before patching. Scan for:
- Detection result strings (`"JAILBROKEN"`, `"frida"`, `"su"`, error codes)
- Config values and feature flags
- Encryption keys on the stack/heap
- DEX magic bytes in memory (decrypted DEX)
- GOT integrity (compare live GOT values against pristine file)

```c
// Generic string scanner across all readable regions
char line[512];
while (fgets(line, sizeof line, maps_file)) {
    // parse start, end, perms
    if (perms[0] != 'r') continue;
    // read region via process_vm_readv
    // memmem(buf, len, needle, needle_len)
}
```

### B5. Memory diffing — Detect in-memory hooks/tampering

Compare live `r-xp` code against the pristine `.so` file to detect:
- Hooks installed by the shield
- GOT entries pointing to unexpected libraries
- Inline patches

```c
// Read pristine bytes from the file at file_offset
// Read live bytes from /proc/<pid>/mem at runtime address
// Compare byte-by-byte → any diff = in-memory modification
```

---

## PART C — Delivery mechanics (how the tool reaches the target)

Ranked by detectability. Choose the lowest rank that can express the needed change.

| Rank | Method | What it leaves | Use when |
|---|---|---|---|
| 1 (default) | External memory write (`pwrite /proc/<pid>/mem`) | Changed bytes only | Patching instructions or data; no new code needed in target |
| 2 | Zygisk `.so` (`postAppSpecialize`) | Extra `.so` map, zygote injection | Must run before `Application.onCreate` (e.g., hide paths) |
| 3 | `ptrace` + remote `dlopen` | `TracerPid` during attach + payload `.so` map | Must run real code inside app (Java calls, OkHttp objects, signer reuse) |
| 4 | `app_process` standalone harness | Runs in own process | Native state is simple enough to bootstrap manually |
| 5 | `LD_PRELOAD` | Env var + mapped `.so` | Avoid against shields; only for non-shielded apps |

### C1. External memory write (rank 1, default)

A standalone ARM64 binary runs as root, reads `/proc/<pid>/maps`, computes the target address, writes via `pwrite`. The target process never sees a tracer, a new mapping, or an environment variable.

**Limitation**: can only change existing bytes. Cannot call functions, build Java objects, or execute new code in the target.

**Daemon variant**: for apps that respawn processes (multi-process architecture), poll `pidof` and patch every new PID before its first event fires.

### C2. Zygisk module (rank 2)

A Magisk/KernelSU Zygisk module loads into the target via `postAppSpecialize`. Runs before the app's own code. Good for path hiding (`faccessat`/`fstatat` interpose via `RTLD_NEXT`).

```c
// hide.c — Zygisk/LD_PRELOAD interpose
#define _GNU_SOURCE
#include <dlfcn.h>
#include <errno.h>
#include <string.h>

static const char *HIDE[] = {"su","magisk","kernelsu","frida","zygisk","/data/adb", NULL};

int faccessat(int fd, const char *path, int mode, int flags) {
    for (int i = 0; HIDE[i]; i++)
        if (strstr(path, HIDE[i])) { errno = ENOENT; return -1; }
    static int (*real)(int,const char*,int,int);
    if (!real) real = dlsym(RTLD_NEXT, "faccessat");
    return real(fd, path, mode, flags);
}
```

### C3. ptrace + dlopen injection (rank 3)

Full remote code injection for when you need to run code *inside* the target process (call Java methods, use the app's ClassLoader, reuse in-process signers).

Sequence:
1. `ptrace(PTRACE_ATTACH)` a non-main thread (main often fails dlopen)
2. Save all registers via `PTRACE_GETREGSET`
3. Write `.so` path to the thread's stack via `process_vm_writev`
4. Set registers: `x0 = path_addr, x1 = RTLD_NOW, x30 = 0, pc = __dl_dlopen`
5. `PTRACE_CONT` → thread calls `dlopen`, payload `.so` constructor runs
6. Read result from `x0`, call `__dl_dlerror` for error message
7. Restore original registers, `PTRACE_DETACH`

The payload `.so`'s constructor can:
- Attach to JavaVM via `JNI_GetCreatedJavaVMs`
- Get `ActivityThread.currentApplication()`
- Load a DEX payload via `DexClassLoader` using the app's real ClassLoader
- Build OkHttp clients that inherit the app's interceptors (signers auto-attach)

**Android 16 note**: blocks writable DEX. `chmod 444` the payload DEX before `DexClassLoader`.

### C4. `app_process` standalone harness (rank 4)

Run a Java+native program in its own process using the Android runtime. Use when the native state can be bootstrapped manually without the full app context.

---

## PART D — Process discovery and multi-process handling

### D1. Finding target PIDs

```c
FILE *pp = popen("pidof com.example.app 2>/dev/null", "r");
// Parse all PIDs — shielded apps often spawn multiple processes
```

For masquerading processes (cmdline is misleading), identify by mapped libraries:
```c
// Scan /proc/*/maps for a library unique to the target app
// e.g., strstr(line, "libtarget_shield.so")
```

### D2. Multi-process / respawn storm

Shielded apps may:
- Spawn multiple processes (main, `:shield`, `:webview`, worker isolates)
- Kill and respawn processes rapidly (respawn storm)
- Run network operations in child processes

**Rule**: patch **every** PID that can send the event/request. Verify each read-back. A patch that lands after the event already fired is too late for that PID but still needed for the next request.

**Daemon pattern**:
```c
for (;;) {
    // pidof com.example.app → list of all PIDs
    // For each PID: check if already patched (read-back), patch if not
    usleep(10000);  // 10ms poll
}
```

---

## PART E — Build and deploy

### E1. Cross-compile (Android NDK)

All tools are single-file C programs. No Makefile needed.

```bash
# Standalone executable (static — no dynamic linker dependency)
aarch64-linux-android21-clang -static -O2 -Wall tool.c -o tool

# Or with PIE (smaller, needs linker on device — always available)
aarch64-linux-android26-clang -O2 -Wall -fPIE -pie tool.c -o tool

# Shared library for injection
aarch64-linux-android26-clang -O2 -Wall -shared -fPIC payload.c -o libpayload.so -ldl
```

### E2. Deploy to device

```bash
adb push tool /data/local/tmp/
adb shell chmod 755 /data/local/tmp/tool
adb shell su -c /data/local/tmp/tool <pid>
```

### E3. Cleanup

Remove all tools after the session. Do not leave `.c`, `.so`, or `.dex` files on the device.

```bash
adb shell su -c rm /data/local/tmp/tool
```

---

## PART F — Address discipline (critical)

### F1. Computing the target address

```
target_address = load_base + virtual_address_from_IDA
```

Where:
- `load_base` = first mapping of the library in `/proc/<pid>/maps` where `file_offset == 0`
- `virtual_address_from_IDA` = the offset/VA shown in IDA/Ghidra (relative to image base 0)

**Common mistake**: using the `r-xp` map start instead of the `r--p` (file_offset=0) map start. The `r-xp` segment has a non-zero file offset, so adding the VA to its start gives a wrong address.

### F2. Read-back verification

Always read back after writing to confirm the patch landed:
```c
pwrite(fd, patch, n, addr);
pread(fd, readback, n, addr);
if (memcmp(readback, patch, n) != 0) {
    // Write failed — wrong permissions, wrong address, or process died
}
```

### F3. Prologue verification

Before writing, read the current bytes and verify they match the expected prologue. This catches:
- Wrong library version (offsets shifted)
- Already-patched process (idempotency)
- Wrong PID (different app)

---

## PART G — ARM64 instruction reference (common patches)

| Instruction | Hex (little-endian) | Use |
|---|---|---|
| `MOV W0, #0` | `00 00 80 52` | Set return value to 0 |
| `MOV W0, #1` | `20 00 80 52` | Set return value to 1 |
| `RET` | `C0 03 5F D6` | Return from function |
| `NOP` | `1F 20 03 D5` | No operation (skip instruction) |
| `B <imm>` | `0x14000000 \| ((offset>>2) & 0x03FFFFFF)` | Unconditional branch |
| `MOV X0, #0` | `00 00 80 D2` | Set 64-bit return to 0 |
| `MOV W0, #0; RET` | `00 00 80 52 C0 03 5F D6` | NOP function (return 0) |

---

## Canonical reference implementation

The workspace at `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\` contains:

| Tool | Location | Technique |
|---|---|---|
| `mem_patch.c` | `_tools/native_hooks/mem_patch/` | Generic code patcher (B1 + C1) |
| `hide.c` | `_tools/native_hooks/zygisk_hide/jni/` | Zygisk path interpose (C2) |
| `inject.c` | `MB_Bank/native-c/` | ptrace+dlopen injector (C3) |
| `gotpatch.c` | `MB_Bank/native-c/` | GOT slot redirection (B3) |
| `heapgate.c` | `MB_Bank/native-c/` | Heap config string flip (B2) |
| `diffcode.c` | `MB_Bank/native-c/` | Live vs file memory diff (B5) |
| `thscan.c` / `cfgscan.c` | `MB_Bank/native-c/` | Read-only string scanners (B4) |
| `sendlog.c` | `MB_Bank/native-c/` | Injected `.so` that interposes `send()` (C3 payload) |

---

## Workflow

```
1. OBSERVE (read-only)
   ├── Parse /proc/<pid>/maps → identify loaded libraries and segments
   ├── Scan heap for detection strings, config flags, error codes
   ├── Read code at target offset → verify expected prologue bytes
   └── Diff live code vs pristine file → detect shield's own hooks

2. IDENTIFY target
   ├── Find the function to patch in IDA/Ghidra → note VA (offset from base 0)
   ├── Determine: code patch (B1), data patch (B2), GOT hook (B3), or injection (C3)?
   └── Determine: external write sufficient, or need in-process code?

3. WRITE the C tool
   ├── Single-file .c with find_base(), pread/pwrite or process_vm_readv/writev
   ├── Prologue check before write, read-back after write
   ├── Multi-PID support if the app is multi-process
   └── Daemon mode if the app respawns or re-parses config

4. BUILD + DEPLOY
   ├── Cross-compile with NDK (aarch64-linux-android)
   ├── adb push + chmod + su -c execute
   └── Verify: read back patched bytes, observe app behavior

5. CLEANUP
   └── Remove all tools from device
```

---

## MUST NOT

- Use Frida when the shield scans maps/threads — use native mechanics only.
- Modify APK files when the server hashes them — patch memory, not files.
- Use the `r-xp` map start as the base — use the `r--p` (file_offset=0) start.
- Leave tool binaries on the device after the session.
- Assume a single PID — check for multi-process and respawn.
- Skip read-back verification — confirm every write.
- Hardcode absolute addresses — always compute `base + offset`.
- Use `LD_PRELOAD` against a shield-aware app.
- Globally clear Java exceptions without understanding the callsite.
- Repeat app-specific codes/offsets as universal rules — always re-derive per build.

## Task-complete self-check (MUST pass before claiming done)

- [ ] Which delivery mechanic was used and why?
- [ ] Target address recorded as `lib + offset`; base = first mapping with file_offset=0
- [ ] Prologue verified before write; read-back verified after write
- [ ] Every PID that can trigger the target behavior is patched
- [ ] Observed behavior confirms the patch works (not just that bytes changed)
- [ ] Cleanup done: no tool binaries left on device
