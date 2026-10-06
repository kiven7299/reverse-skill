# Reverse-engineering resource index

> Curated from several awesome lists, ranked by usefulness. Use these for RE methodology and tool guidance.

---

## General libraries

| Project | Stars | Coverage | Link |
|------|-------|------|------|
| **awesome-reversing** (tylerha97) | 3k+ | RE tools/books/courses/labs | https://github.com/tylerha97/awesome-reversing |
| **awesome-reverse-engineering** (alphaSeclab) | 4k+ | 3500+ tools + 2300 articles, all platforms | https://github.com/alphaSeclab/awesome-reverse-engineering |
| **Reverse-Engineering** (mytechnotalent) | 10k+ | Free tutorials: x86/x64/ARM/AVR/RISC-V | https://github.com/mytechnotalent/Reverse-Engineering |
| **awesome-malware-analysis** (rshipp) | 12k+ | Malware analysis tools/resources | https://github.com/rshipp/awesome-malware-analysis |
| **reversingBits** | — | RE/binary-analysis cheat-sheet collection | https://github.com/mohitmishra786/reversingBits |
| **awesome-arm-exploitation** | — | ARM exploit resources (video/articles/books) | https://github.com/HenryHoggard/awesome-arm-exploitation |
| **Binary-Analysis-Automation** | — | Automated binary analysis (ML/scripts/static/dynamic) | https://github.com/user1342/Awesome-Binary-Analysis-Automation |

---

## ELF / Linux RE

| Resource | Notes | Link |
|------|------|------|
| **libelfmaster** | Safe ELF parser (forensics/malware rebuild) | https://github.com/elfmaster/libelfmaster |
| **ELF spec** | Official ELF format docs | https://refspecs.linuxfoundation.org/elf/elf.pdf |
| **Linux Internals** | /proc, memory layout, syscall | https://0xax.gitbooks.io/linux-insides/ |
| **Compiler Explorer** | See C/C++/Rust/Go compile to assembly | https://godbolt.org/ |

---

## ARM / AArch64

| Resource | Notes | Link |
|------|------|------|
| **ARM architecture manuals** | Full ISA reference | https://developer.arm.com/documentation |
| **Azeria Labs** | ARM assembly/exploit tutorials (best intro) | https://azeria-labs.com/writing-arm-assembly-part-1/ |
| **ARM64 syscall table** | Linux AArch64 syscall numbers | https://arm64.syscall.sh/ |
| **QEMU user-mode** | Analyze ARM binaries without a real device | `qemu-aarch64 -strace ./binary` |

---

## Malware analysis

| Resource | Notes | Link |
|------|------|------|
| **YARA** | Malware signature matching | https://github.com/VirusTotal/yara |
| **Volatility 3** | Memory forensics framework | https://github.com/volatilityfoundation/volatility3 |
| **FLOSS** | Auto-extract obfuscated strings | https://github.com/mandiant/flare-floss |
| **Detect It Easy (DiE)** | File type/packer/compiler ID | https://github.com/horsicq/Detect-It-Easy |
| **PE-bear** | PE analyzer | https://github.com/hasherezade/pe-bear |
| **Capa** | Auto-ID binary capabilities (net/file/crypto) | https://github.com/mandiant/capa |
| **Unpacker** | Generic unpacking framework | https://github.com/malwaretech/UnpackerFramework |

---

## Dynamic analysis / sandbox

| Resource | Notes | Link |
|------|------|------|
| **Frida** | Cross-platform dynamic instrumentation | https://frida.re/ |
| **strace** | Linux syscall trace | OS built-in |
| **ltrace** | Library-call trace | OS built-in |
| **QEMU** | User/system emulation | https://www.qemu.org/ |
| **Unicorn** | Programmable CPU emulator | https://www.unicorn-engine.org/ |
| **Qiling** | High-level binary emulation | https://qiling.io/ |
| **angr** | Symbolic execution + binary analysis | https://angr.io/ |
| **Triton** | Dynamic binary analysis | https://triton-library.github.io/ |

---

## Deobfuscation / unpacking

| Resource | Notes | Link |
|------|------|------|
| **UPX** | Most common packer; `upx -d` unpacks | https://upx.github.io/ |
| **unipacker** | Generic PE unpacker | https://github.com/unipacker/unipacker |
| **de4dot** | .NET deobfuscation | https://github.com/de4dot/de4dot |
| **JADX** | Android DEX deobfuscation | https://github.com/skylot/jadx |
| **JEB** | Commercial Android/ARM decompiler | https://www.pnfsoftware.com/ |
| **Miasm** | RE framework (IR/symbolic exec/deobf) | https://github.com/cea-sec/miasm |
| **OLLVM deobf** | CFF / bogus-CF recovery | Recover with angr/Triton symbolic execution |

---

## Online analysis platforms

| Platform | Notes | Link |
|------|------|------|
| **VirusTotal** | Multi-engine scan + behavior | https://www.virustotal.com/ |
| **Joe Sandbox** | Automated malware analysis | https://www.joesandbox.com/ |
| **ANY.RUN** | Interactive online sandbox | https://any.run/ |
| **Hybrid Analysis** | Free malware analysis | https://www.hybrid-analysis.com/ |
| **Compiler Explorer** | Compiler output | https://godbolt.org/ |
| **Dogbolt** | Multi-decompiler compare (IDA/Ghidra/Binary Ninja) | https://dogbolt.org/ |

---

## Learning path

### Intro (0–3 months)

1. [Reverse Engineering for Beginners](https://beginners.re/) — free ebook
2. [Azeria Labs ARM tutorials](https://azeria-labs.com/) — ARM assembly basics
3. [Nightmare](https://guyinatuxedo.github.io/) — CTF RE/Pwn tutorials
4. [crackmes.one](https://crackmes.one/) — RE practice

### Intermediate (3–12 months)

1. [Practical Binary Analysis](https://practicalbinaryanalysis.com/) — hands-on binary analysis
2. [The IDA Pro Book](https://nostarch.com/idapro2.htm) — deep IDA
3. [Malware Unicorn RE101](https://malwareunicorn.org/workshops/re101.html) — malware RE
4. [pwnable.kr](http://pwnable.kr/) / [pwnable.tw](https://pwnable.tw/) — Pwn practice

### Advanced

1. [Modern Binary Exploitation](https://github.com/RPISEC/MBE) — RPI course
2. [How to Hack Like a Ghost](https://nostarch.com/how-hack-ghost) — advanced ops
3. [Windows Internals](https://docs.microsoft.com/en-us/sysinternals/) — Windows kernel
4. Practice: real malware samples (MalwareBazaar)

---

## Cheat sheets

| Cheat sheet | Link |
|--------|------|
| x86/x64 instructions | https://www.felixcloutier.com/x86/ |
| ARM64 instructions | https://developer.arm.com/documentation/ddi0602/latest |
| Linux syscall table (x64) | https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/ |
| Linux syscall table (ARM64) | https://arm64.syscall.sh/ |
| GDB cheat sheet | https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf |
| radare2 cheat sheet | this pack `radare2/references/cheatsheet.md` |
| IDA shortcuts | https://hex-rays.com/products/ida/support/freefiles/IDA_Pro_Shortcuts.pdf |
| Ghidra shortcuts | Ghidra built-in Help → Keyboard Shortcuts |
