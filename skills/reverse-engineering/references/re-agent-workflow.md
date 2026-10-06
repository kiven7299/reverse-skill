# RE Agent workflow gates (static ↔ dynamic)

> Inspired by: binary-re stage split, community RE skill (Frida/r2/Ghidra/IDA loop), Cerberus three-head loop (static/dynamic/instrument)  
> Issue #65 increments: IAT-repair iron rule / IAT 修复铁律, six-stage map, .NET/DLL·SYS equivalent paths; user-instruction feasibility gate / 可行性门闩; bypass patches 6–10; anti-debug/obfuscation recipes A–T; non-PE multi-format recipes U–AV (2026-08-12)  
> Applies to: `reverse-engineering/`, `ida-reverse/`, `radare2/`, `malware-analysis/`, and cre role handoff. Overlay IDs: `ops/analysis-decision-framework.md` (ADF) and `ops/analysis-blindspot-cookbook.md`.

## 0. Start

```text
□ scope.md: offline sample path or authorized device/target
□ tool-index: real paths for file/strings/r2/ida/frida etc.
□ role: cre (ops/role-map)
```

Do not re-inject full case context between stages. `scope.md` / `workitems.md` / Evidence stay authoritative; `timeline.md` only carries the transition delta:

1. At stage/turn end write only `decision_delta` that actually changes next actions; no change → `[]`.
2. Unchanged route/auth/scope/network profile/tool state/hypothesis go only in `carry_forward_refs`; consumer reads by ref, does not re-serialize / emit.
3. `decision_delta` is not full state; consumer 必须先继承 refs first, then apply delta.
4. Stop at a next-step menu only when two or more evidence-supported branches would cause different next actions; a deterministic gate advances directly.

Example: Triage done and the only legal next step is Static → transition needs only `decision_delta: [phase=triage->static]` + `carry_forward_refs: [scope.md, evidence/E-triage.md]`.

## 0.5 User-instruction feasibility gate / 可行性门闩 (Issue #65)

**Principle**: obey the user's **goal**, do not blindly follow the user's **step order**. Before skipping a step, state the prerequisite and ask confirmation; after confirmation, forced steps MUST be done, and Evidence quality MUST be labeled honestly.

| Situation | Agent MUST |
|------|------|
| User wants X, and current state can produce **valid** Evidence | do X, update Evidence |
| User wants X, but a **known blocking prerequisite** exists (e.g. already judged packed and static IAT unreadable) | **MUST NOT** pretend a meaningful IAT is done; ① one sentence stating the block; ② recommended order (unpack/repair IAT first, or grab APIs dynamically); ③ **ask the user** whether to "still force-read the current junk table" or "follow recommended order" |
| User **forces** the current step (e.g. inspect IAT while still packed) | do it and record Evidence; MUST label `quality=unreadable` / `packed` (or equivalent); **MUST NOT** conclude "no network capability" from that |
| User accepts recommended order | do prerequisite steps first; then do X automatically or on request; **MUST NOT** pass off a prerequisite (e.g. unpack) as "import-table check already done" |

**Relation to "redo X"**: redo X still = redo the named step (or the confirmed legal-prerequisite negotiation). Unpack is a **prerequisite** of the import table, not a **substitute** for it.

Typical conflict: user says on a packed sample "don't unpack yet, look at imports first" → packers often tamper the import directory / encrypt descriptors; static table is junk and meaningless → use this table's "blocking prerequisite" row; do not silently unpack and pretend, and do not silently hand over a junk table as done.

## 1. Triage (5–15 min · mandatory start)

```text
□ hash the sample (MD5/SHA256) → unique ID
□ ID file type: EXE / DLL / SYS / ELF / Mach-O / .NET / script(bat/ps1/vba) / JS / APK etc.
□ non-PE/script/APK/driver specialty: see §3.4 and `references/nonpe-format-cookbook.md` (U–AV)
□ file / DIE / entropy / packer features (PEiD / DIE / Exeinfo etc.)
□ arch: x86 / x64 / ARM; compiler-language clues (VC++ / Delphi / .NET / Go / Rust)
□ packer-type clues: UPX / ASPack / VMProtect / Themida / unknown obfuscation
□ strings / rabin2 -z skim
□ MUST import/export anchors (see "Import-table hard gate and equivalent paths" below); if user jumped ahead and it is packed → §0.5 first
□ produce: E-triage (MUST include imports or equivalent-anchor class summary, with quality label if applicable) + hypothesis list
```

**Stage gate (Triage → Static/Dynamic)**: MUST NOT enter Dynamic until E-triage records imports **or** a legal equivalent-anchor summary (unless IAT repair failure is recorded and dynamic bypass is chosen, see §1.2), and MUST NOT claim "basic triage complete". On parse failure still MUST write the failure output into Evidence; MUST NOT skip. When the user asks to "redo import-table check", MUST redo the imports/equivalent step itself (or first finish the §0.5-negotiated prerequisite); MUST NOT swap in other analysis steps as a stand-in.

### 1.1 Import-table hard gate and equivalent paths

| Sample type | MUST anchor (Evidence) | Notes |
|------|------|------|
| Native PE/ELF/Mach-O (IAT readable) | `E-imports` / `E-triage-imports`: import class summary | `rabin2 -i` / IDA imports / equivalent |
| DLL / SYS / shared lib | **in parallel** `E-imports` + `E-exports` (`rabin2 -i` + `rabin2 -E`) | export table priority equals import table (external entry) |
| .NET managed (no classic IAT) | **equivalent path**: dnSpy/IL/metadata/assembly refs and sensitive-API summary → still write into `E-imports` or `E-triage-imports` semantic slot | **MUST NOT** skip the hard gate because "there is no IAT"; dnSpy view = native "check imports" |
| Import parse failed / empty / packed junk table | still record failure or junk-table output as Evidence, and label `quality` | MUST NOT silently skip; junk table MUST NOT support capability-negation conclusions |

**Clean-import warning (MUST remind)**: if the import table is "too clean" (only kernel32/ntdll etc. base DLLs, almost no business APIs), strongly suspect `LoadLibrary` + `GetProcAddress` dynamic load → note the suspicion in Evidence, and **SHOULD** go Dynamic to capture in-memory APIs; MUST NOT claim "no network/no file capability" from static IAT alone.

**High-risk API combos (patch 8 · SHOULD)**: when the import table is long, prefer **malicious-combo clustering**; filter pure system-base calls. Examples (not exhaustive):

- High-risk cluster: `FindWindowA/W` + `WriteProcessMemory` + `CreateRemoteThread` (inject)
- High-risk cluster: `CryptEncrypt` / `CryptAcquireContext` + lots of `FindFirstFile` / `DeleteFile` (ransomware tendency)
- High-risk cluster: `InternetOpen` / `WinHttp` / `URLDownloadToFile` + persistence APIs (`RegSetValue` / `CreateService`)
- Lone `CreateFile` / `ReadFile` etc. are mostly benign noise unless co-occurring with the clusters above

### 1.2 Unpack and IAT handling (high-risk fork · Issue #65)

```text
Branch A: unpacked / .NET managed
  → go straight to §2 Static (.NET uses equivalent anchors)

Branch B: packed / strong obfuscation
  Step 1: try unpack (auto unpacker / manual OEP) — authorized isolated env only
  Step 2: try repair IAT
    Tools: x86 → ImportREC (or equivalent); x64 → Scylla (or equivalent). MUST NOT grind ImportREC on 64-bit samples.
    Case B1: repair succeeds and is parseable → record E-imports (post-repair) → §2 Static
    Case B2: ImportREC/Scylla errors, repaired binary will not run, or IAT is all garbage (VMP/encrypted packer)
      → [IAT-repair iron rule / IAT 修复铁律] immediately stop further static IAT repair
      → MUST record E-iat-repair-fail (command, tool, failure symptom, decision to go dynamic)
      → go straight to §3 Dynamic: API BP / hardware exec BP / memory search to capture imports
      → this is not "skipping the import table": the import-table path was tried and Evidence recorded
    Case B3 (patch 6): after unpack+IAT repair, double-click flash-crash / BSOD (suspect file CRC/size self-check)
      → abandon further static file repair; record E-self-check-crash or fold into E-iat-repair-fail
      → go §3 Dynamic: BP CreateFile / GetFileSize / hash-related APIs; locate the check-bypass point
```

**IAT-repair iron rule (MUST)**: prefer auto/semi-auto repair; once the repair tool errors or the repaired program will not run, **stop immediately** grinding the static import table, switch to dynamic debug, and capture imported functions at runtime with API breakpoints (e.g. `bp CreateFile` / key network APIs).

## 2. Static (basic static anchors → deep dive)

| Tool | When |
|------|------|
| radare2 / rabin2 | fast functions/imports/strings (imports already MUST-done in Triage or failure-bypass recorded) |
| IDA / Ghidra (MCP or headless) | deep dive, xrefs, types; survey-stage recheck of import classes |
| OLLVM docs | suspected CFF |

```text
□ confirm E-imports / E-triage already contain import-table or equivalent-anchor Evidence (if missing, fill first; MUST NOT postpone)
□ if DLL/SYS: confirm E-exports recorded
□ sensitive-API grouping + high-risk combo clustering (patch 8)
□ hardcoded domain/IP/URL strings; resource section hiding payload?
□ locate key functions (crypto/check/net/auth) → write addresses/symbols into Evidence
□ one path blocked → switch tools (IDA↔r2↔Ghidra)
□ timebox (patch 9 · SHOULD default): static deep-dive ~15 min still no key path → force §3 Dynamic (user/task may override duration)
```

**No MCP**: export decompile text then analyze (see P4nda0s reverse-skills / IDA-NO-MCP approach); still write Evidence paths.

## 3. Dynamic (cross-validation loop)

Core idea: **static supplies clues → dynamic verifies → verify stalls → back to static re-review** (no single fixed order).

### 3.0 Breakpoint opening (patches 7 + 10 · MUST order)

Before launching the sample in a user-mode debugger (x64dbg etc.), pre-set breakpoints as a "four-stage rocket" (names may differ by arch/tool; order does not):

1. **TLS callback** BP (may already have run before debugger EP)
2. **Entry point EP** BP
3. **Sensitive API** BP (e.g. `CreateRemoteThread` / net / file write)
4. **Safety net**: `ExitProcess` / process-exit path BP (patch 10) — if anti-debug exits immediately, **do not rush to restart**; dump memory at once; write the pre-crash image path into Evidence for string/data recovery

```text
□ Frida / x64dbg / gdb / emulator: verify static hypotheses
□ pre-set BPs per §3.0 then run; single-step stack/registers (white-box)
□ behavior monitor: sandbox / Procmon / RegShot (black-box)
□ IAT-repair-fail / self-check flash-crash samples: hardware exec BP or memory search to force-capture APIs; CreateFile/GetFileSize for CRC
□ anti-debug / anti-Frida → reverse-engineering/anti-analysis
□ Android: root-detect / SSL-pinning bypass scripts as needed, **authorized device only**
□ crash logs drive the next hook round (adaptive loop)
□ timebox (patch 9 · SHOULD default): single-step ~200 insns still no malware-behavior clue → force back to static string search / new anchors (overridable)
```

### 3.1 Sandbox / dynamic no-behavior fallback (MUST)

```text
No behavior or immediate exit / infinite sleep
  → check anti-debug / anti-VM routines (CPUID, high-precision timing, sandbox artifacts, etc.)
  → try hardware-BP bypass, patch detection points, or switch to physical / higher-fidelity env
  → write "no behavior + suspected anti-VM" into Evidence; MUST NOT write "sample is harmless" without conditions
```

### 3.2 Timebox / 时间盒 policy (patch 9 · SHOULD)

| Stage | Default threshold (user/task overridable) | Action |
|------|------|------|
| Static deep-dive, no key path | ~15 min | go Dynamic |
| Dynamic single-step, no progress | ~200 insns | back to Static strings/xrefs re-anchor |
| Any path fails repeatedly | record Evidence then switch tool or bypass | MUST NOT spin on the same failed method |

### 3.3 Anti-debug / obfuscation bypass cheat sheet (Issue #65 patches A–T · high frequency)

Full index and action detail: `reverse-engineering/anti-analysis.md` "Agent response recipes A–T". Here only **P0 must-check + common transitions**. Default **authorized isolated lab**; patch/flag-edit is not an unauthorized production action.

| Trigger feature | Preferred action (summary) | Evidence |
|------|------|------|
| `cpuid` then jz/jnz (A) | lab: flip flags or patch to the real branch; record detection-point address | `E-anti-debug-cpuid` |
| `rdtsc` + sub/cmp (B) | bp rdtsc or hook time source; MUST NOT treat infinite idle waiting for sandbox timeout as "harmless" | `E-anti-debug-rdtsc` |
| PEB BeingDebugged / NtGlobalFlag (K) | ScyllaHide or hand-edit PEB; patch conditional jump | `E-anti-debug-peb` |
| `NtQueryInformationProcess` DebugPort/Flags/Object (P) | ScyllaHide / hook return; record class param | `E-anti-debug-ntqip` |
| tiny imports but rich behavior → API hashing (N) | bp GetProcAddress; reverse hash and re-inject into IDA | `E-api-hash` |
| strings empty but net/file behavior → string encryption (I) | find decode routine xref; dump decrypted and re-inject | `E-string-decrypt` |
| has signature but source suspect (F) | SigCheck: valid/revoked/time; **invalid does not lower** threat | `E-sig-forge` |
| standard strings no IOC → try wide chars (T) | `strings -el` / UTF-16LE; Alt+A unicode | `E-wide-strings` |
| debugger-name strings / Toolhelp scan (C) | bp CreateToolhelp32Snapshot chain | `E-anti-debug-procscan` |
| AddVectoredExceptionHandler + deliberate exception (D) | bp VEH register; analyze handler | `E-anti-debug-veh` |
| int3 / DR0–DR7 (M) | patch int3; soft BP or ScyllaHide hide hardware BP | `E-anti-debug-bp` |
| multiple PE headers / overlapping sections (G) | real section-table mapping + entropy; do not trust section names | `E-pe-anomaly` |
| file tail > section sum Overlay (J) | extract overlay; file/entropy; find load-offset xref | `E-overlay` |
| .rsrc abnormally large / high-entropy RT_RCDATA (Q) | extract resource; FindResource chain + decrypt dump | `E-rsrc-payload` |
| DLL loaded only at runtime (R) | check Delay Import; bp delay-load helper | `E-delay-import` |
| while+switch star CFG (H) | **See** `ollvm-deobfuscation.md`; if plugins fail, dynamic path | `E-cff` |
| always-true/false branches (S) | **See** ollvm / SE; dynamic wins | `E-opaque-pred` |
| `/proc/self/status` TracerPid (L) | **Linux/ELF**; hook or patch; not mandatory on Windows main path | `E-anti-debug-tracerpid` |

**Constraint**: bypass failure still records Evidence; MUST NOT write "anti-debug triggered exit" as "sample is harmless". Full A–T and P2 (E compile time, O junk insns) live in the anti-analysis recipe section.

### 3.4 Non-PE / multi-format bypass (Issue #65 patches U–AV · routing)

Full index: `reverse-engineering/references/nonpe-format-cookbook.md`. Here only **type → entry**; action detail is in the cookbook / matching skill.

| Type | Jump | P0 Evidence anchors (examples) |
|------|------|------|
| VBA macros | cookbook §3 + malware | `E-vba-pcode` |
| JS strong obfuscation / JSVMP | **js-reverse** + cookbook §4 | `E-js-vmp` / `E-js-deobf` |
| SYS driver | kernel-driver-reverse + cookbook §5 | `E-driver-irp-handlers` / `E-driver-ioctl` |
| DLL emphasis | cookbook §6 (AM≡A–T **R**) | `E-dll-tls-dllmain` / `E-exports` |
| Android wiper / hidden icon | **apk-reverse** + cookbook §7–8 | `E-android-wiper-*` / `E-android-hidden-icon-*` |

**Constraint**: do not invent a second "non-PE six-stage"; split with §3.3 A–T (PE anti-debug vs multi-format). Authorized lab; wiper/BYOVD/reflective = detection/forensics wording.

## 4. Synthesis (IOC / attack chain / report)

```text
□ Finding: algorithm/check logic/exploitables / behavior conclusions
□ Path: callflow or solve steps hung on E-*
□ IOC: network fingerprints + host fingerprints (table if present; else n/a + reason)
□ report docs-generator (malware/apt/null/vuln overlay by task) + optional diagrams
□ optional: YARA / Snort·Suricata rule precipitation
□ field-journal desensitize
```

## 5. Six-stage field map (Issue #65 mind-map → this file)

| Field stage | This-file section | Hard gate / iron rule |
|------|------|------|
| 1 initial fast judgment | §0–§1 Triage | Hash, arch, file type, packer check; imports/equivalent anchors; §0.5 instruction gate |
| 2 unpack and IAT | §1.2 | IAT iron rule; fail/self-check flash-crash → Evidence → Dynamic |
| 3 basic static anchors | §2 Static | high-risk API combos; timebox SHOULD |
| 4 deep cross-validation | §3 Dynamic | 4-stage BP rocket; no-behavior fallback; timebox; §3.3 A–T; §3.4 U–AV type routing |
| 5 extract IoC and attack chain | §4 Synthesis | IOC + Kill Chain / Path |
| 6 archive and rules | §4 + docs-generator / YARA | structured report; rules optional |

## 6. Difference vs "pile RE skill plugins"

- this pack uses **stage gates + tool-index**; does not default-enable Hex-Rays "unsafe fully automatic execute" class plugins  
- dynamic instrumentation defaults to **offline/lab** network_profile  
- IAT/imports: **try + record** beats "infinite static grind" or "silent skip"  
- user instructions: **goal first + prerequisite negotiate**; MUST NOT pass off unrelated steps as the named step
