---
name: pwn-chain
description: |
  Full-chain engineered method from reverse to a working exploit.
  Fit: have binary + vuln point + target env; need a stable exploit (not a local-only script that dies remotely).
  Three directions: stack overflow / heap exploit / kernel pwn. Stresses the engineering gap "CTF local pass → real remote stable": libc mismatch, heap-spray timing, SMEP/SMAP/KASLR, stack align, remote buffering.
  Core toolchain: pwntools + GEF/pwndbg + ROPgadget/Ropper + one_gadget + libc-database + qemu-system kernel debug.
  Trigger keywords: pwn, stack overflow, heap overflow, ROP, ret2libc, ret2csu, one_gadget, libc-database, heap exploit, tcache, fastbin, unsorted bin, kernel pwn, kROP, SMEP, SMAP, KASLR, modprobe_path, pwntools, GEF, pwndbg.
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

# From vuln point to Working Exploit (Pwn Chain)

## Scope

Use this skill when the task is:

1. **Have binary + known vuln point** — static/audit/fuzz already found overflow/UAF/double free; need trigger-to-shell
2. **CTF local pass, remote fail** — remote env mismatch kills the script; need stabilize
3. **Real-target binary exploit** — SRC / red-team, memory-corruption vuln identified, need RCE
4. **Linux kernel driver ioctl bug** — user-mode trigger, goal is root

**Prerequisite**: you already know "where it crashes". This skill does not find vulns (that is fuzzing / audit); it writes the exploit from the vuln point.

### Split vs other skills

| Scenario | Use |
|------|--------|
| Identify custom VM / anti-debug / complex obfuscation | `reverse-engineering/` |
| Open a binary from zero for static analysis | `ida-reverse/` or `radare2/` |
| **Have a vuln point, write exploit that works remote** | **this skill** |
| Fold the pwned shell into a full attack chain | `attack-chain/` (downstream) |

`reverse-engineering/` focuses on "what the program does" (pattern ID, protocol restore, weird CTF mechanisms); this skill focuses on "turn a understood vuln into an executable attack". Often paired, split is clear.

## Core workflow

```text
Step 1: confirm vuln class + protections
   ├─ checksec ./vuln (NX / Canary / PIE / RELRO / Fortify)
   ├─ file ./vuln  + readelf -d ./vuln
   ├─ classify: stack overflow / format string / heap (UAF/DF/OF) / integer / race / kernel
   └─ → pick which references/

Step 2: pick exploit strategy
   ├─ NX off + no ASLR → raw shellcode
   ├─ NX on + libc given → ret2libc / one_gadget
   ├─ NX on + no libc → leak then libc-database lookup
   ├─ heap → glibc-version techniques (tcache/fastbin/unsorted/large)
   └─ kernel → commit_creds / modprobe_path / core_pattern

Step 3: prepare libc + gadgets
   ├─ libc-database: ./find puts 0x6f0
   ├─ ROPgadget --binary ./libc.so.6 --only "pop|ret"
   ├─ one_gadget ./libc.so.6
   └─ compute base: leak_addr - libc.sym['puts']

Step 4: write pwntools template (local process)
   ├─ context.binary = ELF('./vuln')
   ├─ p = process('./vuln')  /  p = gdb.debug('./vuln','b *main+xx')
   ├─ payload = cyclic(N) + p64(ret) + ...
   └─ p.interactive()

Step 5: local pass
   ├─ attach repeatedly + watch registers + tune offset
   ├─ use pwndbg/GEF vmmap / heap / bins / telescope
   └─ after local pass, switch to remote()

Step 6: remote stabilize
   ├─ libc offset: leak + libc-database lookup; do not guess
   ├─ stack align: 16-byte misalign → movaps crash → add a ret gadget
   ├─ remote latency → recvuntil exact anchor string; disable fuzzy sleep
   ├─ remote buffering: sendlineafter is more stable than sendline
   ├─ heap-spray success: increase spray count + leave padding chunks against coalesce
   └─ loop: while True until success rate ≥ 95%
```

## Typical scenarios

### Scenario 1: remote 64-bit binary (NX+PIE+canary, libc given)

```text
Have: ./vuln (64-bit ELF, NX, PIE, canary) + ./libc.so.6 + nc host port
Vuln: read(buf, 0x200) but buf is only 0x40 bytes → stack overflow
Protect: canary blocks, PIE randomizes .text

Strategy:
1. leak canary first (stack/format-string/partial read)
2. leak a libc function addr (puts@got)
3. libc.address = leaked - libc.sym['puts'] for libc base
4. one_gadget ./libc.so.6 pick a magic gadget whose constraints hold
5. payload = padding + canary + saved_rbp + (pop_rdi + bin_sh + system) or one_gadget
6. add a ret gadget to fix stack align (critical!)
```

Full template: `references/stack-pwn.md`.

### Scenario 2: Linux kernel driver ioctl OOB write → root

```text
Have: vmlinux + bzImage + initramfs.cpio.gz + custom vuln.ko
Vuln: ioctl(0x1337, ptr) copy_from_user length controllable → kernel heap overflow (kmalloc-64 slab)
Protect: SMEP, SMAP, KASLR, KPTI

Strategy:
1. edit init script for root shell (CTF) or leak KASLR base first (real)
2. leak kernel base via /proc/kallsyms (may be restricted) or uninitialized heap spray
3. spray tty_struct / msg_msg / pipe_buffer in kmalloc-64 slab
4. overwrite vtable ptr to userland → blocked (SMEP); switch to stack pivot + kernel ROP
5. ROP: prepare_kernel_cred(0) → commit_creds → swapgs+iretq → userland execve("/bin/sh")
6. or cheaper: overwrite modprobe_path to "/tmp/x", write /tmp/x, then trigger modprobe
```

Full template: `references/kernel-pwn.md`.

## On-Demand Bootstrap

### Tool dependencies

| Tool | Use | Install |
|------|------|---------|
| pwntools | exploit framework | `pip install pwntools` |
| GEF | gdb enhance (kernel + userland) | `git clone https://github.com/bata24/gef` (active fork) |
| pwndbg | gdb enhance (best heap debug UX) | `git clone https://github.com/pwndbg/pwndbg && ./setup.sh` |
| ROPgadget | gadget search | `pip install ropgadget` |
| Ropper | gadget search (alt, more archs) | `pip install ropper` |
| one_gadget | libc magic gadget find | `gem install one_gadget` (needs ruby) |
| libc-database | libc fingerprint lookup | `git clone https://github.com/niklasb/libc-database && ./get` |
| qemu-system-x86_64 | kernel-challenge debug | `apt install qemu-system-x86` |
| binwalk / cpio | initramfs unpack | `apt install binwalk cpio` |
| patchelf | switch libc version | `apt install patchelf` |

### Bootstrap check script

```bash
# one-shot check + install core tools
for t in pwntools ropgadget ropper; do
  pip show $t >/dev/null 2>&1 || pip install $t
done

command -v one_gadget >/dev/null || gem install one_gadget

[ -d ~/tools/libc-database ] || git clone https://github.com/niklasb/libc-database ~/tools/libc-database
[ -d ~/tools/libc-database/db ] || (cd ~/tools/libc-database && ./get ubuntu debian)

[ -d ~/tools/pwndbg ] || (git clone https://github.com/pwndbg/pwndbg ~/tools/pwndbg && cd ~/tools/pwndbg && ./setup.sh)
```

### After the same tool auto-install fails twice

Stop retrying. Emit structured manual install steps (pip mirror / gem mirror / git China mirror / apt mirror) for user confirm.

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Trigger**: have binary + identified vuln point, need to write exploit

**Upstream skills (use them first, then return here)**:
- Do not yet understand what the binary does → `reverse-engineering/`
- Need detailed static analysis → `ida-reverse/`
- Fast recon of arch/protections → `radare2/`

**Downstream skills (after shell)**:
- Fold into full attack chain (lateral, priv-esc, persist) → `attack-chain/`

**Submodule nav**:
- Stack exploits (ret2libc / ret2csu / one_gadget / stack align) → `references/stack-pwn.md`
- Heap exploits (tcache / fastbin / unsorted / large bin / FILE struct) → `references/heap-pwn.md`
- Kernel pwn (kROP / SMEP-SMAP bypass / KASLR leak / modprobe_path) → `references/kernel-pwn.md`

## Notes

- **Do not ship after local pass** — local libc / ASLR / network differ from remote; MUST run 20+ consecutive remote() trials for stability
- **libc version MUST be confirmed** — leak + libc-database lookup; do not assume Ubuntu 22.04 default libc
- **Stack align is the common 64-bit pit** — `movaps xmm0, [rsp]` faults if rsp is not 16-byte aligned; add an empty `ret` gadget
- **Heap exploits are extremely glibc-version sensitive** — tcache in 2.27, safe-linking in 2.32, hooks removed in 2.34; each version has a different path
- **Kernel pwn MUST confirm CPU flags first** — qemu boot args +smep +smap +pku decide how the ROP chain is written
- **One KASLR leak is enough** — after one kernel address, all others are offsets; do not leak repeatedly

## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the RULES Checklist items?
