# OLLVM deobfuscation / Obfuscator-LLVM Deobfuscation

> OLLVM unpack workflow for APK .so, ELF binaries, and control-flow flattening.
> Tool and variant info from 2026 community-active project survey, not training memory.
> Use for: Android NDK hardening, CTF RE, packed .so analysis, commercial obfuscator defense.

---

## 0. Fast decision: which tool?

Match your environment and the target obfuscation type:

| Your situation | First choice | Backup | Notes |
|------|------|------|------|
| Have IDA Pro 7.5-7.7 + Hex-Rays; want one-click unflatten | **obpo-plugin** | d810-ng | obpo uses microcode + dataflow + concolic; strongest, but cloud plugin (needs network; core closed-source) |
| Have IDA Pro (any recent); want local one-stop deobf | **d810-ng** | original D-810 | local, open-source, Z3; covers OLLVM/Tigress/Hodur/Approov variants |
| Have Binary Ninja | **ollvm-breaker** | — | Android .so field (libvdog and similar hardened samples) |
| No IDA/BN, script-only, target x86/x64 | **ollvm-unflattener** (Miasm) | angr deflat | Miasm symbolic execution; BFS multi-layer |
| Pure Python symbolic execution, CTF | **angr** Deobfuscator | Triton | no GUI; scriptable |
| Target ARM64 .so, no IDA | **deollvm** (Unicorn) | angr | Unicorn ARM64 deflat |
| Hit BR obfuscation (indirect branch) | **DeObfBR** | mark data segment RO | Goron/Arkari-style BR often defeated by data-segment RO |
| Hit Tigress | d810-ng `UnflattenerSwitchCase`/`UnflattenerTigressIndirect` | — | d810-ng ships Tigress-specific unflatteners |

> **Core advice:** prefer **d810-ng** (local, actively maintained, wide variant coverage). When cloud is allowed, **obpo-plugin** is strongest. If both fail, custom **angr/Miasm** symbolic execution.

---

## 1. Modern OLLVM variant ecosystem (2026 community survey)

OLLVM is no longer just the 2017 original repo. Active obfuscator forks below. **ID the variant before deobf** — counters differ a lot:

### 1.1 Obfuscator fork lineage

| Variant | LLVM baseline | New vs original OLLVM | Counter points |
|------|------|------|------|
| **Obfuscator** (original) | 3.3~4.0 | sub + bcf + fla (three base passes) | standard tools handle it |
| **Hikari** | 6~8 | Anti Class Dump, Function Call Obfuscate, Function Wrapper, Indirect Branching, Split BB, String Encryption | decrypt strings first + fix indirect jumps |
| **Hikari-LLVM15** | 15~19 | + Anti Debugging, Anti Hook, Constant Encryption | closed-source now; Constant Encryption hardens static analysis |
| **goron** | 7~10 | Indirect Branch/Call/GlobalVariable | ⚠️ Goron-style indirect obfuscation often defeated by "mark data segment RO" |
| **Arkari** (komimoe/Hikari) | 14~latest | goron-based, still maintained | same as goron; data-segment RO is a partial counter |
| **Pluto** | 14 | MBA Obfuscation, Random CF, Split BB, **Trap Angr** (deliberately breaks angr) | ⚠️ Trap Angr pass kills angr SE; switch tools or skip the trap |
| **Polaris** (ex-Pluto) | 16 | Alias Access, Indirect Branch/Call, String Encryption, Merge Function, Linear MBA, Dirty Bytes Insertion, Function Splitting, Junk Insertion | Hikari+Pluto combo; hardest; handle in layers |
| **O-MVLL** | open-obfuscator | Python-driven pass manager; Anti Hooking, Arithmetic(MBA), BB Duplicate, CF Breaking, Function Outline, Indirect Branch/Call, Opaque Constants | common in modern Android hardening; Python config is easy to customize |
| **amice** (Rust) | Rust impl | full set + VM Flatten, Instruction Virtualization, Delayed Offset Loading, Parameter Aggregation | includes VM-ization; restore VM handlers, do not treat as plain deflat |
| **VMP family** (SmallVmp/VMPilot/xVMP/VMPacker) | — | instruction virtualization | **not OLLVM**; needs VM RE; see VM-specific tools |

### 1.2 Key ID clues

- **Trap Angr** (Pluto/Polaris): if angr explodes or path-explodes, suspect Trap Angr pass → switch to d810-ng or Unicorn dynamic
- **Goron/Arkari indirect jump**: if dispatcher uses indirect jump (BR x8, not switch), first mark related data segments RO; jump targets often become statically solvable
- **Constant Encryption** (Hikari-LLVM15/Polaris/O-MVLL): constants decrypt at runtime; pure static cannot see real values → Unicorn-execute the decrypt stub
- **VM Flatten** (amice): CF becomes a VM dispatch loop; **do not treat as ordinary fla**; ID the VM handler table first

---

## 2. OLLVM obfuscation-type detection

ID features of the three core OLLVM passes:

### 2.1 Control Flow Flattening (`fla`)

**IDA view:**
- function entry jumps first to a unique dispatcher block
- main logic split into many basic blocks; each block ends by jumping back to the dispatcher
- dispatcher picks the next block via a **state variable**
- huge `switch`; cases have no logical relation

**Variant forms (dispatchers d810-ng IDs):**
- O-LLVM: switch / if-chain + state variable
- Tigress: `m_jtbl` (switch-case) or `m_ijmp` (indirect jump; needs `goto_table_info` config)
- Hodur (PlugX): nested `while(1)` state machine, `jnz state, #CONST`, **no switch dispatcher**
- Approov: `while(v8 != C)`; state constants clustered in `0xF6000–0xF6FFF`

### 2.2 Bogus Control Flow (`bcf`)

- **unreachable fake branches** inserted between real branches
- fake branches protected by **opaque predicates** (always true/false, but static analysis cannot prove it)
- lots of dead code inflates function size

```c
// classic opaque predicate: x(x+1) is always even; compiler cannot prove it
if ((x * (x + 1)) % 2 == 0) {
    // real logic
} else {
    // unreachable junk
}
```

### 2.3 Instruction Substitution (`sub`) → MBA

- simple arithmetic/bitwise replaced by equivalent complex expressions (MBA, Mixed Boolean-Arithmetic)

### 2.4 Fast classification

| Type | IDA feature | Main counter |
|------|------|------|
| fla (flattening) | huge switch + dispatcher | obpo / d810-ng / deflat |
| bcf (bogus CF) | unreachable branches + dead code | d810-ng opaque predicate removal / symbolic execution |
| sub/MBA | complex arithmetic | d810-ng MBA simplifier / SiMBA (Z3) |
| fla + bcf + sub | all of the above, huge bloat | **layered deobf (bcf then fla then sub)** |

---

## 3. Mainstream tools (community-active)

### 3.1 obpo-plugin — strongest effect, cloud plugin

> [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) · 629⭐ · active 2026-06

Hex-Rays **microcode** pseudocode optimizer. Rebuilds flattened CF via **dataflow tracking + program slicing + concolic**. Community-recognized among the strongest.

**Key traits:**
- operates at microcode; optimizes decompiler output (does not rewrite ASM)
- IDA 7.5.0 / 7.6.0 / 7.7.0 + Hex-Rays
- arch: ARM, ARM64, x86, x86_64, PowerPC, PowerPC64, MIPS (7.6/7.5)
- **cloud plugin**: target-function binary uploaded to obpo-server (core closed-source; plugin free/open)
- server self-funded; timeout 600s; **no multithread / abusive calls**

**Install and use:**

1. download obpo_plugin.py and the obpoplugin directory
2. copy into IDA plugins path
3. restart IDA; open target binary
4. locate dispatcher block in CFG; typically looks like:
   [screenshot: repo assets/dispatchblock.png]
5. right-click → OBPO → Mark and process function
6. after processing, refresh decompiler
7. mark newly appearing dispatcher blocks from decompiler changes (iterate nested fla)

**Fit and limits:**
- ✅ standard and nested fla; good results
- ⚠️ needs network; careful with sensitive samples (unreleased vulns, trade secrets) — binary is uploaded
- ⚠️ server may be down; depends on author maintenance
- ❌ cannot solve all obfuscation (author states this)

### 3.2 d810-ng — local one-stop first choice

> [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) · 223⭐ · updated 2026-06-26

Modern maintained/rewritten D-810 (Next Generation). Local, open-source, **Z3 SMT**. Widest variant coverage.

**Core capabilities (from d810-ng README):**

*Instruction-level opts:*

| Class | Notes |
|------|------|
| MBA simplification | `(a+b)-2*(a&b) => a^b`, Z3-verified DSL rules |
| Hacker's Delight | bitwise equivalences (from *Hacker's Delight*) |
| O-LLVM patterns | Obfuscator-LLVM-specific MBA patterns |
| Constant folding | 22 constant-simplify rules |
| Predicate simplification | opaque-predicate removal (setz/setnz/lnot/smod) |
| Z3 rules | SMT when template match fails |
| Hodur-specific | PlugX (Hodur) malware MBA patterns |

*CF unflatteners (by target obfuscation):*

| Unflattener | Target | Notes |
|------|------|------|
| `Unflattener` | O-LLVM | standard switch/if-chain + state variable |
| `UnflattenerSwitchCase` | Tigress | Tigress switch-case dispatch (`m_jtbl`) |
| `UnflattenerTigressIndirect` | Tigress | Tigress indirect jump (`m_ijmp`); needs `goto_table_info` |
| `HodurUnflattener` | Hodur (PlugX) | nested `while(1)` + `jnz state, #CONST`; no switch |
| `BadWhileLoop` | Approov | `while(v8 != C)`; state constants in 0xF6000–0xF6FFF |
| `UnflattenerFakeJump` | generic | remove always-true/false conditional jumps |
| `SingleIterationLoopUnflattener` | leftover | clean single-iter loops where `INIT == CHECK` and `UPDATE != CHECK` |
| `UnflattenControlFlowRule` (experimental) | generic | CFG unflattener based on path emulation |

**Install and use:**

2. install deps (including Z3)
3. copy into IDA plugins dir
4. in IDA press Ctrl-Shift-D to load plugin
5. tick the rule sets to apply in the GUI
6. apply to the target function

**Why d810-ng over original D-810:**
- original D-810 is less maintained
- d810-ng has CI tests, rewritten code, new Tigress/Hodur/Approov unflatteners
- Z3 fallback when template match fails; higher success rate

### 3.3 ollvm-unflattener — Miasm symbolic execution, script-only

> [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) · 265⭐ · active 2026-06

**Miasm** SE engine. No IDA/BN. Pure Python CLI.

**Traits:**
- recover original CF via Miasm SE (vs MODeflattener's pure-static method)
- **BFS multi-layer**: auto-follow calls of the target function; recursive deobf
- Windows/Linux x86/x64
- outputs a new deobfuscated binary

**Install and use:**

```bash
# basic usage
# -a: auto-follow calls for multi-layer
```

**Fit:** no IDA, target x86/x64, need batch scripted processing.

### 3.4 ollvm-breaker — Binary Ninja field

Uses **Binary Ninja** to unflatten. Repo ships Android hardened sample `libvdog.so`; already fixed JNI_OnLoad, crazy::GetPackageName, prevent_attach_one, etc.

**Fit:** Binary Ninja users, Android .so field.

### 3.5 deollvm

Unicorn-based ARM64 OLLVM deflat. Fallback for ARM64 .so without IDA.

### 3.6 DeObfBR — BR obfuscation specialist

Removes **BR obfuscation** (indirect-branch obfuscation, Goron/Arkari style).

**⚠️ cheap counter (from awesome-ollvm):** Goron/Arkari-style indirect-related obfuscation can often be defeated by **marking data segments read-only** — indirect jump targets often depend on a runtime-writable data segment; RO makes them statically solvable.

### 3.7 angr — generic symbolic-execution framework

```python
# built-in Deobfuscator
```

**⚠️ Pluto/Polaris Trap Angr pass:** these variants ship a trap against angr SE. If angr path-explodes or faults, suspect Trap Angr → switch to d810-ng or Unicorn dynamic.

---

## 4. Full deobf workflow (by scene)

### 4.1 Generic decision tree

```text
Target binary

1. ID OLLVM variant (see §1.2 clues)
  ├── original OLLVM / Hikari / O-MVLL  → standard fla/bcf/sub
  ├── Pluto / Polaris                → watch Trap Angr; avoid angr
  ├── Goron / Arkari                 → try data-segment RO first, then BR
  └── amice (has VM)                  → not plain fla; restore VM handlers

2. Pick tool (see §0 decision table)
  ├── have IDA + network OK + non-sensitive sample → obpo-plugin
  ├── have IDA + local              → d810-ng
  ├── have Binary Ninja            → ollvm-breaker
  ├── no GUI + x86/x64           → ollvm-unflattener (Miasm)
  ├── no GUI + ARM64             → deollvm (Unicorn) / angr
  └── pure SE / CTF           → angr

3. Layered deobf (order matters)
  a) remove opaque predicates (bcf) first   → d810-ng opaque predicate removal
  b) then unflatten CF (fla) → unflattener
  c) finally simplify MBA (sub)       → d810-ng MBA simplifier / SiMBA

4. Verify
  ├── function size dropped a lot?
  ├── CFG went from star/radial to chain/tree?
  └── Frida-hook key functions; verify logic?
```

### 4.2 Android NDK .so deobf specialty

OLLVM-hardened NDK .so is the most common APK RE scene.

**Step 1 — extract .so:**

```bash
# or unzip APK directly: unzip target.apk -d out/ ; find out -name "*.so"
```

**Step 2 — ID OLLVM and variant:**

```text
readelf -a libnative.so | grep -E "Size|text"   # .text huge but few functions → likely OLLVM
# open in IDA; look at function features:
#   huge switch → fla
#   unreachable branches → bcf
#   complex arithmetic → sub/MBA
#   indirect jump BR x8 → Goron/Arkari; try data-segment RO
#   while(1) + jnz state → Hodur; use d810-ng HodurUnflattener
```

**Step 3 — deobf (layered):**

```text
a) bcf: d810-ng opaque predicate removal  (or obpo handles automatically)
```

**Step 4 — Frida dynamic verify:**

```javascript
// Trace OLLVM state variable; help deflat find the state-var address

// hook dispatcher entry; observe state change sequence
        // read state variable (need register/stack location from decompile)
        console.log("[state]", this.context.x8);  // assume state in x8
```

### 4.3 CTF fast deobf

CTF is time-tight; prefer the fastest path:

```text
# find the largest functions (most likely obfuscated)
        # angr fails → suspect Trap Angr → switch d810-ng / Unicorn
```

---

## 5. MBA expression simplification

### 5.1 Common OLLVM MBA patterns

```text
# these equalities are the simplification targets of OLLVM sub-pass expressions
```

### 5.2 Tool choice

| Tool | Method | Fit |
|------|------|------|
| **d810-ng MBA simplifier** | batch inside IDA; Z3-verified | first choice; in the decompile flow |
| **SiMBA** (`pip install simba-simplifier`) | CLI/library | pure expression simplify; batch |
| **Arybo** | symbolic bitvectors | lots of MBA expressions |
| **Z3 direct** | SMT | most generic; when all templates fail |

```python
# SiMBA example
```

---

## 6. Full deobf case script

```bash
#!/bin/bash
# OLLVM deobfuscation pipeline (2026 community tools)
# for standard OLLVM / Hikari / O-MVLL hardened ELF/.so

BINARY=$1

echo "[*] Stage 0: basic analysis and variant ID"
file $BINARY
readelf -h $BINARY 2>/dev/null | head -5
echo "    → confirm variant in IDA (see §1)"

echo "[*] Stage 1: d810-ng local deobf (first choice)"
echo "    IDA → Ctrl-Shift-D load d810-ng"
echo "    tick: MBA + Opaque predicate + Unflattener"
echo "    Apply to target functions"
echo "    save IDB"

echo "[*] Stage 2: obpo-plugin (if d810-ng is weak and network is OK)"
echo "    IDA → right-click dispatcher → OBPO → Mark and process"
echo "    ⚠️ do not use on sensitive samples (binary uploaded to cloud)"

echo "[*] Stage 3: no-IDA fallback (x86/x64)"
echo "    python unflattener -i $BINARY -o deobf.bin -t <func_addr> -a"

echo "[*] Stage 4: ARM64 .so no-IDA fallback"
echo "    deollvm (Unicorn) or angr Deobfuscator"

echo "[+] Done. Re-analyze in IDA to verify."
```

---

## 7. Common pitfalls (community field notes)

| Problem | Cause | Fix |
|------|------|---------|
| angr path explosion / abort | Pluto/Polaris **Trap Angr** pass | switch d810-ng or Unicorn dynamic |
| obpo-plugin cannot connect | server self-funded; may be down | fall back to local d810-ng; open an issue on the obpo repo |
| Goron/Arkari indirect-jump deflat fails | dispatcher uses BR x8 not switch | mark data segment RO first, then DeObfBR |
| function still messy after d810-ng | OLLVM customized pass params/seed | SE-remove opaque predicates first, then unflatten |
| nested fla not fully cleared in one pass | obpo/d810-ng clears one layer per pass | **iterate**: mark each newly appearing dispatcher |
| ARM64 .so deflat errors | old deflat scripts are x86-only | use d810-ng / obpo (ARM64) / deollvm |
| Hikari strings invisible | String Encryption pass | Unicorn-emulate decrypt stub; dump decrypted strings |
| amice target: deflat totally ineffective | includes VM Flatten / Instruction Virtualization | **not OLLVM fla**; restore VM handlers (see VM RE) |
| Hodur(PlugX) sample has no switch dispatcher | nested while(1) + jnz state | d810-ng **HodurUnflattener**; do not use ordinary Unflattener |
| Approov sample state constants look patternless | constants clustered in 0xF6000–0xF6FFF | d810-ng **BadWhileLoop** unflattener |
| sensitive sample sent to obpo by mistake | binary uploaded to cloud | classified / unreleased-vuln samples **local tools only** (d810-ng/angr) |
| Frida hook of OLLVM function hangs | state var mutated into infinite loop | conditional BP at dispatcher entry; cap execution count |

---

## 8. Tool cheat sheet (2026 community activity)

| Tool | Platform | Method | Stars/price | Last update | Open source | Notes |
|------|------|------|---------|---------|------|------|
| **obpo-plugin** | IDA | microcode+concolic (cloud) | 629 | 2026-06 | plugin open / core closed | strongest; needs network |
| **ollvm-breaker** | Binary Ninja | BN API | 441 | 2026-06 | ✅ | Android .so field |
| **ollvm-unflattener** | CLI | Miasm SE | 265 | 2026-06 | ✅ | x86/x64, BFS multi-layer |
| **d810-ng** | IDA | microcode+Z3 | 223 | 2026-06 | ✅ | **local first choice**; wide variant coverage |
| **DeObfBR** | — | BR obfuscation specialist | 96 | 2026-06 | ✅ | Goron/Arkari indirect branch |
| **IDA_Ollvm-unflattener** | IDA | Miasm plugin edition | 90 | 2026-04 | ✅ | IDA plugin wrap of ollvm-unflattener |
| **deollvm** | CLI | Unicorn | 34 | 2026-04 | ✅ | ARM64 specialist |
| **angr** | CLI | symbolic execution | — | active | ✅ | generic; beaten by Trap Angr |
| **SiMBA** | CLI/lib | MBA simplify | — | — | ✅ | expression simplify |
| **Triton** | CLI | SE + taint | — | active | ✅ | dynamic SE |

---

## 9. References

**Obfuscators (to understand the adversary):**
- [obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator) — original OLLVM
- [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari) — Hikari
- [komimoe/Hikari](https://github.com/komimoe/Hikari) — Arkari (goron-based, LLVM 14+)
- [amimo/goron](https://github.com/amimo/goron) — goron
- [bluesadi/Pluto](https://github.com/bluesadi/Pluto) — Pluto
- [za233/Polaris-Obfuscator](https://github.com/za233/Polaris-Obfuscator) — Polaris (ex-Pluto)
- [open-obfuscator/o-mvll](https://github.com/open-obfuscator/o-mvll) — O-MVLL
- [fuqiuluo/amice](https://github.com/fuqiuluo/amice) — Rust OLLVM passes
- [lich4/awesome-ollvm](https://github.com/lich4/awesome-ollvm) — **variant ecosystem overview (strongly recommended first)**

**Deobf tools:**
- [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) — strongest cloud plugin
- [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) — local first choice
- [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) — Miasm script-only
- [amimo/ollvm-breaker](https://github.com/amimo/ollvm-breaker) — Binary Ninja
- [GeT1t/deollvm](https://github.com/GeT1t/deollvm) — ARM64 Unicorn
- [Mrack/DeObfBR](https://github.com/Mrack/DeObfBR) — BR obfuscation specialist
- [maskelihileci/IDA_Ollvm-unflattener](https://github.com/maskelihileci/IDA_Ollvm-unflattener) — IDA plugin edition
- [angr](https://angr.io/) — symbolic-execution framework
- [SiMBA](https://github.com/tech-srl/simba) — MBA simplify

**Papers/blogs:**
- [Quarkslab: Deobfuscation: Recovering an OLLVM-protected program](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html) — classic deflat theory
- [MODeflattener](https://github.com/mrT4ntr4/MODeflattener) — static deflat (contrast to ollvm-unflattener)

> Related: [[anti-analysis.md]] (anti-debug/anti-analysis master table), [[tools-advanced.md]] (advanced toolset), [[elf-analysis.md]] (ELF file analysis), [[ai-assisted-re.md]] (AI-assisted RE)
