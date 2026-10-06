# Non-PE / multi-format agent response recipes U–AV + AW–DN

> Parallel to PE anti-debug recipes A–T (`../anti-analysis.md`): **file type** → "trigger → one-line action → Evidence".  
> **Not** a second main flow. After Triage IDs the type, jump to the matching skill + this table.  
> Default **authorized isolated lab / authorized samples and devices**. Wiper, BYOVD, reflective inject: write **detection and forensics**, not unauthorized destruction/exploit tutorials.  
> Bypass or restore failure still MUST record Evidence; MUST NOT silently treat as "harmless".

> §1–§8 / U–AV = original rules (Issue #65). §9–§23 / AW–DN = extended rules (Issue #87, after dedup + semantic enhance + edge-case patches).

## 0. Routing cheat sheet

| Type clue | Primary skill | This-table section |
|------|------|------|
| .bat / .cmd / batch | malware-analysis | §1, §19 |
| Office macros / VBA / XLM / .docm/.xlsm | malware-analysis | §3 (incl. DD OLE extract, DJ XLM macros) |
| .docx/.xlsx/.pptx OOXML outbound / DDE / .rtf OLE | malware-analysis | §10 (incl. DK RTF) |
| Web/frontend JS obfuscation, JSVMP | js-reverse | §4, §21 (incl. DE/DF) |
| .sys / kernel driver | reverse-engineering/kernel-driver-reverse.md + cre | §5 |
| .dll emphasis | malware-analysis / re-agent-workflow | §6 (deduped vs A–T) |
| APK / Magisk / hidden icon | apk-reverse | §7–§8, §23 |
| .pdf / PDF document | malware-analysis | §9 |
| .jar/.class / Java bytecode | reverse-engineering | §12 |
| .reg / registry script | malware-analysis | §17 |
| Xposed/LSPosed module | apk-reverse | §22 |
| ELF / Linux binary | reverse-engineering | → elf-analysis.md, anti-analysis.md |
| Python bytecode | reverse-engineering | → languages.md |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **U** | lots of SET single-char vars + %a%%b% concat, or ^ line-continue splitting commands | expand SET line by line; restored command list; batch deobf tools OK; **MUST NOT** treat as "no action" before restore | E-batch-deobf | P0 |
| **V** | text opens as garbage; HEX header FF FE (UTF-16 LE BOM) | confirm BOM → convert UTF-8 then parse; or chcp 65001 + type | E-batch-encoding | P2 |
| **W** | lots of REM/::, redundant GOTO/labels drowning real logic | strip comments; comb true GOTO paths; isolate-run and capture actual cmd logs | E-batch-deadcode | P1 |

> Numbering keeps the proposer's habit: **no patch Y**.

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **X** | multi-layer FromBase64String / Gzip / Compress / nested -replace | decode **layer by layer**; record each layer separately; tools optional (PowerDecode etc.); if none, hand/script | E-ps-decode-layer-N | P0 |
| **Z** | reversed strings, fragments + concat then Invoke-Expression/IEX | restore full string; BP IEX or script-block log; plaintext commands into Evidence | E-ps-string-restore | P1 |

## 3. VBA macros / XLM (AA AB AC DD DJ)

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AA** | olevba/OLEDump sees only P-Code; source stream empty (VBA Stomping) | P-Code decompile tools; if incomplete, Word/Excel macro debug observe; write limits | E-vba-pcode | P0 |
| **AB** | lots of Chr() concat or Base64 strings; suspected shellcode/nested script | Immediate Window/script restore strings; type after decode; dynamically watch CreateObject/Shell | E-vba-str-decode | P1 |
| **AC** | meaningless If 1=2, or InsertLines/DeleteLines self-modify | statically follow real branch; dynamically bp self-mod APIs and dump post-change macros | E-vba-selfmod | P2 |
| **DD** | olevba/oledump finds VBA project (vbaProject.bin); ext .docm/.xlsm/.pptm | oledump.py check OLE stream structure; olevba extract VBA source and detect suspect APIs; check AutoOpen/Workbook_Open etc. auto-run macros | E-office-vba | P0 |
| **DJ** | .xls/.xlsm contains Excel 4.0/XLM macros (hidden in cell formulas, not VBA stream); olevba finds XLM markers | olevba --xlm extract XLM formulas; check hidden sheets for EXEC/CALL/REGISTER; XLMMacroDeobfuscator dynamic emulate restore | E-office-xlm | P0 |

## 4. JavaScript (AD AE AF) → primary path js-reverse

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AD** | custom bytecode array + while/switch interpreter (JSVMP) | find VM entry and opcode dispatch; dynamic log traces; AST+dynamic dual track; see js-reverse DeepDive | E-js-vmp | P0 |
| **AE** | while(1){switch} + large string-array indices | AST/Babel reconstruct; restore strings from array indices; wakaru etc. optional; **do not** paste the whole PE ollvm-deobfuscation longform | E-js-deobf | P0 |
| **AF** | debugger, hijacked console, performance.now delta, DevTools detect | disable BPs / freeze time source / headless browser; patch detection points; authorized pages | E-js-anti-debug | P1 |

## 5. SYS kernel driver (AG AH AI)

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AG** | DriverEntry very short; logic not at entry | scan non-empty MajorFunction[] slots; prefer IRP_MJ_DEVICE_CONTROL/CREATE; address list into Evidence | E-driver-irp-handlers | P0 |
| **AH** | DeviceIoControl / IOCTL dispatch present | build control-code→handler table; label METHOD_* and buffer direction; user-mode comms surface | E-driver-ioctl | P0 |
| **AI** | sample loads/drops a known vulnerable driver or oddly signed driver (BYOVD pattern) | compare **public** lists such as LOLDrivers; record driver name/hash/signature; analyze **call intent**; **do not** expand exploit steps | E-driver-byovd | P1 |

See kernel-driver-reverse.md for the flow; this table only adds agent action anchors.

## 6. DLL (AJ–AQ) — dedup vs A–T / #72

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AJ** | DLL analysis only looked at exports/EP; ignored TLS or DllMain | **TLS callbacks + DllMain MUST both be inspected**; dynamic BP order still follows the four-stage rocket (TLS→EP/DllMain→API→ExitProcess) | E-dll-tls-dllmain | P0 |
| **AK** | export names good-then-evil, misnamed, or exports mismatch behavior | cross exports vs actual calls; list anomalous exports | E-exports-anomaly | P0 |
| **AL** | no exports or very few, still gets loaded | locate from entry, strings, xrefs, callers; do not give up because "no exports" | E-dll-noexport | P0 |
| **AM** | static IAT missing DLL; used only at runtime | **See A–T patch R** (Delay-Load / E-delay-import); do not dual-write the longform here | E-delay-import | P0 pointer |
| **AN** | need restore export params and calling convention | xrefs + dynamically watch registers/stack; label stdcall/fastcall etc. | E-dll-export-abi | P1 |
| **AO** | suspected DLL hijack/sideload | check same-name DLL in app dir, search path, KnownDLLs; legit program + anomalous DLL combo | E-dll-sideload | P1 |
| **AP** | fileless mapping / reflective-load clues | memory features, loader behavior, pathless modules; authorized-env forensics | E-dll-reflective | P1 |
| **AQ** | lower risk only because export names "don't look malicious" | **MUST NOT** judge safe from export names alone; combine section perms, entry, strings, dynamic behavior | E-dll-export-priority | P1 |

DLL/SYS hard gate remains: E-imports + E-exports (see re-agent-workflow).

## 7. Android wiper / persistence (AR AS AT) → apk-reverse

> **Authorized samples, images, or test devices only.** Actions are detect, extract IOC and persistence paths — not carry out destruction.

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AR** | Magisk module/script contains delete-libs, flash, bulk rm of system partitions etc. **wiper-feature commands** | feature-command table + module path; label high-risk destructive capability; do not execute wiper commands | E-android-wiper-cmd | P0 |
| **AS** | loop curl\|sh / remotely pull scripts, unconventional C2 URL | extract URL; analyze whether download body contains wiper commands; record temp paths | E-android-wiper-backdoor | P0 |
| **AT** | /data/adb/service.d, post-fs-data.d, suspect /system/priv-app etc. | list persistence scripts/APKs; content summary into Evidence | E-android-persistence | P1 |

## 8. Android transparent/hidden icon (AU AV) → apk-reverse

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AU** | LAUNCHER icon fully transparent/empty label, Theme.NoDisplay, no LAUNCHER category, component disabled | aapt dump badging + manifest; decompile check icon pixels; anomalies into Evidence | E-android-hidden-icon-manifest | P0 |
| **AV** | installed but no desktop icon; background traffic/auto-start/high-risk perms/dynamically restore icon | pm list vs desktop; dumpsys package; broadcasts and device_admin; behavior into Evidence | E-android-hidden-icon-behavior | P1 |

> **§9–§23 below are Issue #87 extended rules (AW–DC).**
> Removed chapters that duplicated existing files: ELF (→ elf-analysis.md), Mach-O (→ platforms.md), Python (→ languages.md).

## 9. PDF malicious documents (AW AX AY AZ)

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **AW** | pdfid finds /JS, /JavaScript, /OpenAction, /AA, /Launch count >0 (incl. hex-encoded names like /4A#61#76#61... obfuscated counts) | pdfid -e stats (plain vs obfuscated counts); pdf-parser extract suspect objects; peepdf interactive + JS emulate | E-pdf-autoaction | P0 |
| **AX** | pdfid finds /EmbeddedFile >0; object streams have FlateDecode/ASCIIHexDecode cascade filter chains; or encoded payload hidden in /Annot objects | pdf-parser extract stream data; peepdf decode multi-layer cascade filters (incl. AES-encrypted stream security handler r5/r6); check Annotation objects; file ID decoded type | E-pdf-embedded | P0 |
| **AY** | extracted PDF JS has lots of eval, unescape, String.fromCharCode, atob | peepdf JS emulate env execution trace; layer-decode Base64/Hex/ROT13; CyberChef assist | E-pdf-js-deobf | P1 |
| **AZ** | PDF structure anomalies: /JBIG2Decode, XREF table manipulated, object-number jumps | pdfid -d rename suspect keywords; check known CVE exploit patterns; extract exploit trigger conditions | E-pdf-exploit | P1 |

## 10. Office OOXML / DDE / RTF (BA BB DK) → complementary to §3 VBA

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BA** | after unzipping docx/xlsx/pptx, word/_rels/ or xl/_rels/ contain suspect external relationships (incl. remote template injection) | check *.rels external links; check vbaData.xml; extract embedded OLE objects; check protocol-handler abuse (ms-msdt: / search-ms: / ms-officecmd:) | E-office-ooxml | P0 |
| **BB** | document contains DDEAUTO or DDEEXEC field codes executing external commands via fields | olevba --dde scan; extract DDE command args; check whether they point at PowerShell/external exe | E-office-dde | P0 |
| **DK** | .rtf contains embedded OLE objects (not OOXML, not classic OLE compound) | rtfobj extract embedded OLE; oleobj analyze object type; check Equation Editor exploits (CVE-2017-11882 etc.); file ID extracted type | E-rtf-ole | P0 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BC** | file starts with \x00asm magic; or JS contains WebAssembly instantiate logic | wasm2wat to text; check import section for host imports; wasm-decompile to pseudocode; check Emscripten glue signatures (__wasm_call_ctors) to judge JS-compiled origin | E-wasm-struct | P0 |
| **BD** | many WASM functions but simple logic; bodies split into tiny functions; or meaningless block/loop nesting | diswasm assess function-minimization level; JEB Pro / IDA WASM plugin deep analysis; dynamically trace execution logs | E-wasm-obfuscation | P1 |
| **BE** | WASM module talks to the browser via JS import/export; WebSocket, fetch, WebGL calls present | analyze JS glue and WASM together; browser DevTools trace data exchange; extract net-comm URL/domain | E-wasm-c2 | P1 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BF** | JD-GUI/jadx opens JAR with meaningless short class/method names (a.a.a / _0x prefix / numeric class names); or lots of while/switch CF obfuscation | ID obfuscator type (ProGuard / Allatori / ZKM); Java Deobfuscator static deobf; if high intensity, dynamic-debug key logic | E-java-obfuscation | P0 |
| **BG** | lots of Class.forName(), Method.invoke(), Constructor.newInstance(); or custom ClassLoader + defineClass() loading classes from byte arrays in memory; import table harmless but runtime dynamically loads malicious classes | javap -c -v for reflection-call detail; trace Class.forName arg strings; check defineClass() byte-array origin; dynamically bp Method.invoke | E-java-reflection | P0 |
| **BH** | JAR contains .so (Linux/Android) or .dll (Windows); or System.loadLibrary() calls | extract native libs; file ID format; hand off to ELF/PE independent analysis | E-java-native | P1 |
| **BI** | after JAR/ZIP extract, nested JAR/WAR/EAR; high-entropy .dat/.bin/.img under /resources, /assets | recursively unzip all nested archives; entropy for encrypted/compressed; check META-INF/MANIFEST.MF and pom.xml | E-java-nested | P1 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BJ** | PE strings contain AutoIt / AU3 / EA05 / EA06 signatures; or resource section contains AutoIt script resources (distinguish AutoHotKey; MITRE T1059.010 shares both) | autoit-ripper extract compiled script; ID encoding family (EA05 = AutoIt3.00 / EA06 = AutoIt3.26); after EA06 header extract 8-byte decrypt key for payload; restore source | E-autoit-extract | P0 |
| **BK** | extracted script has lots of StringEncrypt/_StringEncrypt; or Execute dynamic exec + meaningless var names | myAutToExe static decompile; ID anti-debug techniques; analyze obfuscated CF | E-autoit-deobf | P1 |
| **BL** | script contains RegWrite (registry persistence), FileInstall (file drop), InetGet (net download), Run/RunWait | mark sensitive API call sequences; analyze InetGet URL; trace FileInstall drop paths | E-autoit-malicious | P0 |
| **DM** | AutoIt as loader does process hollowing: CallWindowProc/EnumWindows callback + shellcode + inject into legit process (regsvcs.exe etc.), drop .NET payload (DarkGate / Snake Keylogger / ArechClient2 pattern) | check DllCall/DllCallbackRegister call chains into kernel32 inject APIs; extract shellcode data; ID injected target process; extract .NET payload for independent analysis | E-autoit-hollowing | P0 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BM** | HTML contains HTA:APPLICATION tag, window.execScript, or CreateObject calls | check HTA:APPLICATION attrs (Application, WindowState); extract VBS/JS from script tags | E-hta-bypass | P0 |
| **BN** | after mshta.exe launch, HTA uses XMLHttpRequest / ActiveXObject to remotely pull payload and execute | extract net-request URL; track ActiveXObject creation (ADODB.Stream etc.); restore full download-exec chain | E-hta-download-chain | P0 |
| **BO** | HTA is a single extremely long obfuscated string executed via eval / execScript | extract Base64/Hex encoded payload and decode; CyberChef recursive encoding detect; restore payload | E-hta-oneline | P1 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BP** | .wsf contains \<job\> + \<script language="..."\> tags; mixed JScript/VBScript/Python | split code blocks by \<script language\>; analyze each by its language rules | E-wsf-multi | P0 |
| **BQ** | .jse/.vbe starts with #@~^ signature; Microsoft Script Encoder | screnc-decoder decode; if no tool, dynamic exec + dump decoded script | E-jse-decode | P0 |
| **BR** | WSF multiple \<script\> blocks + \<package\> refs external resources + \<component\> refs COM | build cross-block call graph; trace function calls between \<script\>; restore full exec flow | E-wsf-call-chain | P1 |
| **BS** | WSF contains WshShell.SendKeys UAC bypass, WshShell.Run with 0 hidden window, WScript.Sleep delay bypass | check whether user-sim ops bypass security prompts; record stealth-exec params | E-wsf-anti-detect | P1 |

## 16. MSI installer (BT BU BV)

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BT** | MSI contains CustomAction table (Binary / Script / DLL custom actions) | msiexec /a or lessmsi extract; check CustomAction table; extract custom-action binaries | E-msi-custom-action | P0 |
| **BU** | MSI Binary table contains VBScript/JScript custom-action scripts | extract script binary from Binary table and decode to readable script; analyze per VBS/JS rules | E-msi-script | P1 |
| **BV** | MSI silent install via /quiet /passive /qn; ALLUSERS=1 elevation | record install command-line args; analyze Property-table permission settings; mark silent+elevate combo | E-msi-privilege | P1 |

## 17. REG registry scripts (BW BX BY)

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BW** | .reg writes HKCU\...\Run or HKLM\...\Run etc. autostart paths | extract all paths; mark Run-path entries as persistence; record full path and value | E-reg-persistence | P0 |
| **BX** | .reg modifies HKCR\...\shell\open\command (file association) or HKCR\CLSID\{...}\InprocServer32 (DLL inject) | check whether shell\open\command is a nonstandard exe; check InprocServer32 DLL path | E-reg-hijack | P0 |
| **BY** | .reg modifies HKLM\...\Policies\System (UAC level), EnableLUA, ConsentPromptBehaviorAdmin | check default security config before change; analyze UAC impact; mark downgrade behavior | E-reg-uac-bypass | P1 |

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **BZ** | .vbs/.js parsed by both VBScript and JScript; conditional compile (@_win32) or language-feature cross-exec | split VBScript/JScript blocks; syntax-analyze separately; ID mixed-exec logic | E-vbs-mixed | P1 |
| **CA** | script contains CreateObject("WScript.Shell") / CreateObject("Shell.Application") / Scripting.FileSystemObject | label high-risk COM object calls; track Run/Exec args; trace FSO-created file paths | E-vbs-com-abuse | P0 |
| **CB** | script starts with #@~^ signature; Microsoft Script Encoder (VBS-specific) | screnc-decoder decode; if no tool, dynamic exec + dump decoded script | E-vbs-encoded | P0 |
| **CC** | VBA/VBScript contains WScript.Shell.Run + cmd /c + PowerShell, then process inject | trace CreateObject COM chain; analyze inject features in Run args; record full process-create chain | E-vbs-inject-chain | P0 |
| **DN** | VBScript/JScript fileless persistence via WMI ActiveScriptEventConsumer (no Startup folder / registry Run key) | check WMI event subscription (__EventFilter + __FilterToConsumerBinding + ActiveScriptEventConsumer); extract bound script content; mark fileless persistence | E-vbs-wmi-persist | P0 |

## 19. BAT/CMD advanced obfuscation (CD–CI) → complementary to §1 U–W

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **CD** | setlocal enabledelayedexpansion + !var! + dynamic var names (!var_%i%!) | expand one by one after delayed expansion; Batch-Dump --expand auto-expand | E-bat-delayed-expand | P0 |
| **CE** | type/more/findstr reads self or file :stream ADS alternate data stream then executes | check : suffix refs (file.bat:payload); dir /r list ADS; type file:stream extract | E-bat-ads-hidden | P0 |
| **CF** | lots of echo line-by-line write .tmp/.cmd temp files then call | extract all echo redirects; restore temp-file content; monitor scripts generated in temp dir | E-bat-temp-gen | P1 |
| **CG** | for %%i in (...) do set var=%%i accumulating vars; for /f line-parse command output | expand for loops one by one; record each iteration assign; serialize-restore for /f results | E-bat-for-expand | P1 |
| **CH** | main batch receives args via %1 %*; parent/downloader passes obfuscated instructions | check call context; record incoming args; Base64-decode args; restore full call chain | E-bat-param-call | P1 |
| **CI** | certutil -decode / powershell -Command / echo \| findstr combo decode-exec | extract Base64/Hex strings and decode; check whether decode result is executable script/PE | E-bat-encoded-exec | P0 |

## 20. PowerShell advanced bypass (CJ–CO, DL) → complementary to §2 X–Z

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **CJ** | [Ref].Assembly.GetType('...AmsiUtils') / amsiInitFailed / GetTypes() etc. AMSI bypass (incl. hardware-BP bypass: CPU debug registers, no memory write/VirtualProtect) | ID bypass pattern (Patch / registry / env var / hardware BP); dynamically confirm it took effect; label bypass-technique type | E-ps-amsi | P0 |
| **CK** | [PSConstraintLanguage] type ops or DefaultRunspace session-state change to bypass CLM | ID CLM-bypass pattern; mark bypass-clm; analyze post-bypass exec context | E-ps-clm-bypass | P0 |
| **CL** | [ScriptBlock]::Create / $ExecutionContext.InvokeCommand constructors; or overwrite ScriptBlock log settings | check whether script disables logging; dynamically verify whether logs were bypassed | E-ps-sb-log-bypass | P1 |
| **CM** | IEX (New-Object Net.WebClient).DownloadString(...) or [Reflection.Assembly]::Load(FromBase64...) fileless exec | extract download URL; check domain/IP reputation; PS logs capture in-memory loaded code; isolated-net simulate extract payload | E-ps-reflect-load | P0 |
| **CN** | 3+ nested encodings: outer Base64 → Gzip → XOR → plaintext (beyond §2 X two-layer range) | recurse decode to plaintext or cannot continue; record each layer intermediate; PowerDecode automate; each layer into Evidence | E-ps-multi-decode | P0 |
| **CO** | Set-Alias maps IEX to a single-char alias; Get-ChildItem variable: dynamically get var values | expand all alias maps back to original command names; AST analysis restore vars | E-ps-alias-decode | P1 |
| **DL** | script patches ntdll.dll EtwEventWrite (stomping) to silence telemetry; often combined with AMSI bypass | check EtwEventWrite address obtain + memory patch (ret 0xC3); check together with CJ AMSI bypass; mark dual-bypass combo | E-ps-etw-bypass | P0 |

## 21. JavaScript advanced obfuscation (CP CQ DE DF) → complementary to §4 AD–AF

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **CP** | JS uses Proxy to intercept property access + Reflect API dynamic method calls to defeat static analysis | ID Proxy get/set/apply trap functions; trace Reflect.get actual target; mark dynamic intercept behavior | E-js-proxy | P1 |
| **CQ** | JS contains _0x... hex string array + while(!![]) dead loop + for+switch CF (obfuscator.io features) | ID obfuscator.io features (string array + dead loop); de4js / jsnice auto-deobf; restored code into Evidence | E-js-obfuscator | P0 |
| **DE** | JS body is a large bytecode array + VM interpreter loop (many while/switch); entry points at eval/Function constructor; business logic fully unreadable (deepening of §4 AD) | ID VM entry; track opcode→handler map; browser dynamic exec Hook eval output; JSimplifier AST reconstruct; record opcode map | E-jsvmp-deep | P0 |
| **DF** | JS eval dynamically generates new code and immediately executes, document.write rewrites the page, or Function constructor dynamically builds function bodies | Hook eval and Function constructor; record generated code; browser dynamic exec capture self-mod content | E-js-selfmod | P1 |

## 22. Xposed/LSPosed module analysis (CR–CX) → apk-reverse

> Analyze the Xposed/LSPosed **module itself** as the RE target (not a tool-usage scene).

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **CR** | AndroidManifest.xml has no android:name entry Activity; meta-data sets xposedmodule=true | check assets/xposed_init for entry class; search IXposedHookLoadPackage/ZygoteInit/CmdInit interface impls | E-xp-entry | P0 |
| **CS** | code contains XposedHelpers.findAndHookMethod / XposedBridge.hookMethod / findClass | extract findAndHookMethod first arg (target class) + second arg (target method); build target-app inventory | E-xp-hook-targets | P0 |
| **CT** | module contains DexClassLoader/PathClassLoader dynamic load; or Runtime.exec / ProcessBuilder command exec | trace DexClassLoader constructor args; extract dynamically loaded DEX for independent analysis; check exec command args | E-xp-dynamic-load | P0 |
| **CU** | Hook targets involve payment/biometrics/SMS/contacts/location/crypto-key etc. sensitive APIs | classify Hook target classes/methods by sensitivity; mark payment/biometrics/SMS-contacts classes; summarize threat | E-xp-sensitive-hooks | P0 |
| **CV** | code contains XposedBridge detection evasion / Zygote inject-trace wipe / custom net comms | check stacktrace modify / XposedBridge class-ref wipe; check independent net requests (OkHttp/Socket); ID C2 target | E-xp-anti-detection | P1 |
| **CW** | code contains Resources dynamic replace / View draw intercept / AccessibilityService declared | check AssetManager replace / Resources.updateConfiguration; check AccessibilityService config; ID UI hijack | E-xp-ui-hijack | P1 |
| **CX** | AndroidManifest.xml declares lsposed xposedscope meta-data; or code contains package-name whitelist check | parse xposedscope target-app range; check dynamic whitelist bypass (reflect-modify scope); ID global-Hook overreach | E-xp-scope-bypass | P1 |

## 23. Magisk module deep analysis (CY–DC, DG–DI) → complementary to §7 AR–AT

> §7 focuses on wiper/destructive behavior. This section covers non-destructive but suspect module behavior: install-script analysis, file drop, Zygisk inject, anti-detect, persistence, privilege, lateral infection.

| ID | Trigger | Action (summary) | Evidence | Priority |
|------|------|------|------|------|
| **DG** | Magisk module ZIP root contains config.sh / install.sh; META-INF/com/google/android/update-binary is a nonstandard installer | extract on_install/print_modname/set_permissions from config.sh/install.sh; check whether update-binary contains extra payload; mark pm install / dd block device / mount -o remount,rw ops | E-mg-install-script | P0 |
| **DH** | ZIP contains system/ / vendor/ / data/ dir structure; or post-fs-data.sh / service.sh boot-exec scripts | extract drop-file paths; ID whether APKs drop to /system/priv-app/; check service.sh + post-fs-data.sh for boot persist/keep-alive/C2; mark all writes to system partitions | E-mg-file-drop | P0 |
| **CY** | module contains zygisk/ dir (arm64-v8a.so etc. native libs); or config.sh declares IS_ZYGISK=true | extract zygisk/ native libs; analyze ZygiskModule callbacks (onLoad / preAppSpecialize / postAppSpecialize); check JNI Hook | E-mg-zygisk | P0 |
| **DI** | module scripts write /data/adb/service.d/ or /data/adb/post-fs-data.d/; or modify crontab/init.rc (deepening of §7 AT) | extract scripts written to service.d + post-fs-data.d; check uninstall-time auto-infect of other modules (post-uninstall.sh / module-dir watch); check magisk --remove-modules trigger-protection | E-mg-persistence | P0 |
| **CZ** | module scripts contain resetprop system-prop modify / magiskhide / DenyList; or integrate Shamiko (hide Zygisk itself) / TrickyStore (tamper cert chain) / PlayIntegrityFork (forge Play Integrity API) | extract all resetprop calls; ID modified props (ro.debuggable / ro.build.tags etc.); check DenyList hide-self; ID Shamiko/TrickyStore/PlayIntegrityFork module-level anti-detect | E-mg-anti-detect | P0 |
| **DA** | module scripts contain setenforce 0 / mount -o rw,remount /system / chmod 777 sensitive dirs | check SELinux ops (setenforce/chcon/restorecon); check system-partition mount + dm-verity disable; mark high-risk privilege | E-mg-privilege | P0 |
| **DB** | dropped APK/script contains curl/wget/HTTP client; or dropped APK requests INTERNET + READ_CONTACTS/SMS etc. sensitive perms | extract net-request target URL/IP; analyze dropped APK permission declarations; ID data-exfil logic | E-mg-c2 | P0 |
| **DC** | script walks /data/adb/modules/, modifies other module files, or writes a copy of itself into other modules | check module.prop inject of malicious instructions; check other modules' service.sh append of malicious code; ID "parasite" logic | E-mg-cross-infect | P0 |

## 24. Constraints (global)

1. **No parallel main flow**: stage gates still follow re-agent-workflow / each skill.  
2. **Evidence MUST be recorded**: including failure, half-restore, quality= labels.  
3. **Dedup vs A–T**: do not repeat PE anti-debug; AM→R; AJ adds the DLL view and does not overturn the TLS rocket.  
4. **Missing tools**: record n/a + hand equivalent; do not pretend a commercial suite was used.  
5. **Authorization**: destructive/inject/driver-vuln class is defense analysis and forensics wording only.
6. **Extended-rule dedup**: ELF → elf-analysis.md; Mach-O → platforms.md; Python → languages.md. This table does not repeat those format rules.

## 25. P0 minimum checklist (when type hits)

```text
□ bat/cmd → U (+ V/W if needed; advanced CD–CI)
□ ps1 → X (+ Z; advanced CJ–CO + DL ETW)
□ office ooxml/rtf → BA + DK (+ BB if DDE suspected)
□ js strong obfuscation → AD or AE (+ AF; advanced CP/CQ/DE/DF)
□ sys → AG + AH (+ AI if BYOVD suspected)
□ dll → AJ + AK/AL; Delay-Load goes R
□ apk destroy/hide → AR/AS or AU (+ AT/AV)
□ xposed module → CR + CS + CT + CU (+ CV–CX)
□ magisk deep → DG + DH + CY + DI + CZ + DA (+ DB/DC)
```
