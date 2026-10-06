# ELF binary deep-analysis reference

> Structure parse, anti-analysis ID, and analysis tricks for Linux/Android ELF RE.

---

## ELF structure cheat sheet

### File header (ELF Header)

```text
Offset Size Field              Notes
0x00  4    e_ident[EI_MAG]   Magic: 7f 45 4c 46 ("\x7fELF")
0x04  1    e_ident[EI_CLASS] 1=32bit, 2=64bit
0x05  1    e_ident[EI_DATA]  1=LE, 2=BE
0x10  2    e_type            2=EXEC, 3=DYN(PIE/SO), 4=CORE
0x12  2    e_machine         0x03=x86, 0x3E=x86_64, 0xB7=AArch64, 0x28=ARM
0x18  8    e_entry           entry VA
0x20  8    e_phoff           program-header table offset
0x28  8    e_shoff           section-header table offset (may be 0 after strip)
0x38  2    e_phnum           program-header count
0x3C  2    e_shnum           section-header count
```

### Program header (Program Header)

```text
Type       Name       Notes
0x01   PT_LOAD    loadable segment (code/data)
0x02   PT_DYNAMIC dynamic-link info
0x03   PT_INTERP  interpreter path (/lib/ld-linux.so)
0x04   PT_NOTE    auxiliary info
0x06   PT_PHDR    program-header table itself
0x6474e550 PT_GNU_EH_FRAME  exception handling
0x6474e551 PT_GNU_STACK     stack-executable flag
0x6474e552 PT_GNU_RELRO     read-only relocs
```

### Common sections

| Section | Notes |
|------|------|
| `.text` | code |
| `.rodata` | read-only data (string constants) |
| `.data` | initialized globals |
| `.bss` | uninitialized globals |
| `.plt` / `.got` | dynamic-link jump tables |
| `.init_array` | constructor pointer array |
| `.fini_array` | destructor pointer array |
| `.dynamic` | dynamic-link info |
| `.symtab` / `.dynsym` | symbol tables |
| `.strtab` / `.dynstr` | string tables |

---

## Anti-analysis ID

### Common ELF anti-analysis

| Technique | Feature | Counter |
|------|------|---------|
| Corrupt program headers | PHDR filled with junk (e.g. 0x0a) | repair by hand or ignore broken PHDR |
| No section header | `e_shoff = 0`, `e_shnum = 0` | analyze via program headers only; do not depend on sections |
| Strip | no `.symtab`; function names gone | GoReSym(Go) / signature match / FLIRT |
| Static link | no `.dynamic`; huge size | FLIRT/Lumina for library fns |
| Fake file type | suffix .sh/.txt/.jpg | `file` / magic bytes |
| UPX packed | `UPX!` marker | `upx -d` unpack |
| Custom packer | entry jumps to decompress code | run dynamically to OEP then dump |
| Anti-debug | ptrace(TRACEME) | LD_PRELOAD hook / patch |
| Anti-VM | check /proc/cpuinfo | edit cpuinfo or hook the read |
| Code encryption | decrypt .text at runtime | breakpoint after decrypt then dump |

### ID self-extract / self-modify

```text
Features:
1. mmap(PROT_READ|PROT_WRITE|PROT_EXEC) near entry
2. immediately memcpy or copy loop
3. then mprotect to change perms
4. finally br/jmp to the new mapping

Analysis:
1. find mmap → record returned address
2. breakpoint after mprotect(PROT_EXEC)
3. dump decompressed memory
4. analyze as a new binary
```

---

## ARM64 (AArch64) RE cheat sheet

### Registers

| Register | Use |
|--------|------|
| x0-x7 | args/return |
| x8 | indirect result (syscall number) |
| x9-x15 | temporaries |
| x16-x17 | IP0/IP1 (PLT jump) |
| x18 | platform register (Android: shadow call stack) |
| x19-x28 | callee-saved |
| x29 (FP) | frame pointer |
| x30 (LR) | link register (return address) |
| SP | stack pointer |
| PC | program counter |

### Common insn patterns

```text
Prologue:
  stp x29, x30, [sp, #-N]!    # save FP and LR
  mov x29, sp                  # set frame pointer

Epilogue:
  ldp x29, x30, [sp], #N      # restore FP and LR
  ret                          # return (br x30)

Syscall:
  mov x8, #NR                  # syscall number
  svc #0                       # trap

Conditional branch:
  cmp x0, #0
  b.eq label                   # equal
  b.ne label                   # not equal
  cbz x0, label                # x0 == 0
  cbnz x0, label               # x0 != 0

Address load:
  adrp x0, page                # page high bits
  add x0, x0, #offset          # add low 12-bit offset
  ldr x0, [x1, #offset]        # load from memory
```

### Linux ARM64 syscall numbers

| Number | Name | Notes |
|------|------|------|
| 56 | openat | open file |
| 63 | read | read |
| 64 | write | write |
| 57 | close | close |
| 222 | mmap | mmap |
| 226 | mprotect | change memory perms |
| 117 | ptrace | process trace |
| 220 | clone | create process/thread |
| 221 | execve | exec |
| 93 | exit | exit |
| 94 | exit_group | exit process group |

---

## Common compression/pack ID

| Algorithm | ID features | Decompress |
|------|---------|---------|
| **LZSS** | bitstream + literal/match flags | custom decompressor (as in this report) |
| **ZLIB/Deflate** | Magic: `78 01`/`78 9C`/`78 DA` | `zlib.decompress()` |
| **GZIP** | Magic: `1F 8B` | `gzip -d` / `gunzip` |
| **LZ4** | Magic: `04 22 4D 18` | `lz4 -d` |
| **LZMA/XZ** | Magic: `FD 37 7A 58 5A 00` (XZ) | `xz -d` / `lzma -d` |
| **Brotli** | no fixed magic; use context | `brotli -d` |
| **Zstandard** | Magic: `28 B5 2F FD` | `zstd -d` |
| **UPX** | string `UPX!` | `upx -d` |
| **Custom** | decompress loop at entry | reverse algorithm then write decompressor |

### Clues for custom compression

```text
1. loop + bit ops (shift, AND, OR) near entry
2. "sliding window" copy-back (read backward from output buffer) → LZ family
3. frequency table / Huffman tree build → Deflate/Huffman
4. fixed-size block processing → block compression (LZ4/Snappy)
5. arithmetic-coding traits (interval shrink) → LZMA/ANS
```

---

## Linux process-injection techniques

### mmap + code inject

```text
Flow:
1. mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_ANON|MAP_PRIVATE, -1, 0)
2. write shellcode/payload into the mapping
3. mprotect(addr, size, PROT_READ|PROT_EXEC)  # make executable
4. jump to mapped address

Features:
- mmap return value saved
- immediately memcpy or write loop
- then mprotect changes perms
- finally br/blr to that address
```

### ptrace inject

```text
Flow:
1. ptrace(PTRACE_ATTACH, target_pid)
2. waitpid(target_pid)
3. ptrace(PTRACE_GETREGS, target_pid, &regs)
4. set regs.pc to injected code
5. ptrace(PTRACE_SETREGS, target_pid, &regs)
6. ptrace(PTRACE_CONT, target_pid)

Features:
- open /proc/<pid>/mem or use ptrace
- read/modify target registers
- write shellcode into target address space
```

### /proc/self/mem self-modify

```text
Flow:
1. open("/proc/self/mem", O_RDWR)
2. lseek(fd, target_addr, SEEK_SET)
3. write(fd, new_code, size)

Use:
- bypass W^X (mmap pages cannot be W+X together)
- modify own code segment (.text is usually RO)
- runtime patch instructions
```

---

## Strategy for large ELFs

For 5MB+ binaries:

```text
1. Fast recon (5 min)
   - file / rabin2 -I → arch, type, protections
   - strings | grep -i "error\|fail\|http\|/proc\|/dev" → key strings
   - rabin2 -i → imports (if any)
   - rabin2 -E → exports

2. Structure (10 min)
   - readelf -l → program headers (LOAD layout)
   - code near entry → decompress/decrypt?
   - find .init_array → constructors (may include anti-debug)

3. Locate key logic
   - start from string xrefs
   - start from syscalls (mmap/ptrace/open)
   - start from net fns (connect/send/recv)

4. Divide and conquer
   - if self-extract → decompress first, analyze payload
   - if multi-module → analyze by function
   - binary-diff across versions
```

---

## Tool command cheat sheet

```bash
# basics
file binary
readelf -h binary          # ELF header
readelf -l binary          # program headers
readelf -S binary          # section headers (if present)
rabin2 -I binary           # combined info

# strings
strings -a binary | less
rabin2 -z binary           # data-segment strings
rabin2 -zz binary          # whole-file strings

# disasm
r2 -A binary               # radare2 analysis
objdump -d binary          # GNU disasm
aarch64-linux-gnu-objdump -d binary  # ARM64 cross-disasm

# dynamic
strace -f ./binary         # syscall trace
ltrace -f ./binary         # library-call trace
qemu-aarch64 -strace ./binary  # ARM64 emulate

# memory dump
gdb -p <pid> -ex "dump memory out.bin 0xADDR 0xADDR+SIZE" -ex quit

# repair broken ELF
# hand-edit e_phnum or patch broken PHDR
python -c "
import struct
with open('binary', 'r+b') as f:
    f.seek(0x38)  # e_phnum offset (64-bit)
    f.write(struct.pack('<H', 2))  # set correct PHDR count
"
```
