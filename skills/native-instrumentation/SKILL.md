---
name: native-instrumentation
description: Use for Android app-shield / RASP bypass WITHOUT Frida — in-memory patch, native no-Frida instrumentation, Xqsig/Xqcode request-signer reuse, SSL certificate-pinning bypass via Conscrypt/BoringSSL callback patch, and Zygisk path hiding. Applies when Frida is detected, the app hashes its own files, or an OkHttp/native interceptor must be re-used for signing. General principles plus per-app lessons drawn from real cases in the canonical workspace (BShield/VNeID/ZaloPay/MB Bank).
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the task is a client-side shield bypass on a device/app already in scope
3. `NEXT`: read the canonical real-case workspace:
   `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\`
   - `WORKSPACE_PLAN.md`, `_agent_worklog/TODO.md`, `AGENTS.md`
   - `_methodology/07_native_c_hooking.md`, `06_ssl_pinning_bypass.md`, `02_dynamic_analysis.md`, `10_api_protection_bypass.md`
   - match the app class to a real case below before touching a device
4. `NEXT`: verify tool availability (`../tool-index.md` and `../../TOOLS.md`); rebuild patchers from workspace source, never keep stray binaries on the device
5. `ACT`: choose a delivery mechanic, apply the general principle, then reuse the matching real case

# Native shield instrumentation (no-Frida)

## Scope

Prefer this skill when the client app:

- detects / rejects Frida (maps scan, port 27042, `frida-agent` string, ZygiskFrida injection)
- server-hashes the app's own `.so`/dex so repacking is impossible
- signs or encrypts requests via a native `OkHttp.Interceptor` (Xqsig/Xqcode/Xqplatform, BShield, V-Key)
- pins TLS at Conscrypt / BoringSSL, not just OkHttp Java pins

Toolchain: external memory patch, Zygisk module, `app_process` standalone harness, or
`ptrace` + remote `dlopen` in-process reuse. No Frida attach, no `Interceptor.attach`.

> How to read this skill: **General principles** below apply to ANY shield. **Per-app lessons**
> are observations from ONE concrete app; they illustrate a general point but their codes,
> offsets, and flags are NOT universal — re-derive per build and never assert them as the norm.

---

## PART A — General principles (apply to any shield)

### A1. Live-address discipline
- Target address = library **`r--p` load base** from `/proc/<pid>/maps` + VA (offset-0), NOT the
  `r-xp` map start (that reads wrong bytes/zeros).
- Reliable live read = `open("/proc/<pid>/mem")` + `pread`; a shell `dd` read can return zeros.
- Rebuild patchers from workspace source; do not carry a stale binary that encodes wrong offsets.

### A2. Multi-process / respawn handling
- Shielded apps may spawn many short-lived pids (a respawn storm) and run network in child
  isolates. Patch **every** pid that can send, verify each read-back, before the first
  event/handshake of that pid.
- A patch that lands after the event already fired does NOT falsify the function — check the live
  process for the other event objects before switching hypotheses.

### A3. SSL pinning is layered — test each separately
| Layer | Where | Attack |
|---|---|---|
| Proxy ignore | app never uses system proxy | transparent/VPN redirect |
| CA trust store | `/apex/com.android.conscrypt/cacerts/` | mount CA in |
| Java pin | OkHttp `CertificatePinner`, Conscrypt `TrustManagerImpl` | no-op check |
| Native/Conscrypt | cert-verify callback in `libjavacrypto.so` | patch the **installed callback** |
| Flutter bank TLS | BoringSSL inside `libflutter.so` | patch the cert-verify callback branch |
| Second verifier | post-handshake response-cert check | locate by runtime stack trace |

Generic rules:
- Patch the **callback the library installed**, NOT `SSL_set_custom_verify` / `SSL_CTX_set_custom_verify`.
  Killing the setter removes the custom callback and default verification fails the intercepted cert.
- There can be a SECOND, post-handshake response-cert verifier that still blocks after the
  handshake patch. If a "certificate of the response is not approved"-style reject persists, locate
  the second verifier by stack-tracing the instant it logs — do not assume one patch is enough.
- Distinguish a **certificate reject** from an **connection/network error**; they are different
  failure classes (see B3 for a concrete example). Diagnose by the actual error text, not the number.

### A4. Request-signer reuse
- If the signer needs app runtime state (Application, real ClassLoader, VM), **reuse the in-process
  signer** — do not reimplement offline. Feeding valid input to the real signer beats recovering the
  algorithm.
- "Signer reuse" is NOT decrypting keys or reversing a private key; it only constructs a valid
  request and lets the runtime signer sign it.

### A5. Server-side flags are not cleared locally
- A server-pushed detection/flag tied to the client IP/device is not cleared by `pm clear` or a
  local patch. Reaching clean state may need a clean reinstall + reboot.

---

## PART B — Delivery mechanics (ranked by detectability)

From `_methodology/07_native_c_hooking.md`. Rank 1 is the default.

| Rank | Method | What it leaves in the target | Illustrated by |
|---|---|---|---|
| 1 | External memory write (`pwrite /proc/<pid>/mem`) | changed bytes only; no map, no tracer | ZaloPay SSL callback, MB submit patch |
| 2 | Zygisk `.so` (`postAppSpecialize`) | extra map, zygote injection | MB path-hide (observation), Zygisk hide |
| 3 | `ptrace` + remote `dlopen` | `TracerPid` during attach + payload map | VNeID signer-reuse `injector.c`/`xqpayload.cpp` |
| 4 | `app_process` standalone harness | runs in own process | ZaloPay `XqDirectSigner` |
| 5 | `LD_PRELOAD` | env var + mapped `.so` | avoid against shields |

### B1. External memory patch (default)
Only changes instructions — cannot build OkHttp objects or call Java. Works for `mov w0,#0; ret`,
branch flips, NOPs. Reusable tool `_tools/native_hooks/mem_patch/mem_patch.c`:
```
mem_patch <pid> <libname> <vaddr_hex> [hexbytes...]     # no bytes = read-only dump
```
Apply A1–A2 (base, read-back, every pid) when using it.

### B2. Zygisk path hide
Hide `su`/`magisk`/`kernelsu`/`frida`/`/data/adb`/`/proc/modules` probes from a denylisted package.
See `_tools/native_hooks/zygisk_hide/`. Requires NDK r26+ for `android-34`. Load into the target
package only; do NOT also enable ZygiskFrida.

### B3. In-process signer reuse
When the signer needs full app runtime state (Application object, real PathClassLoader, VM/state),
standalone harnesses hang/crash. Reuse the real signer in the running app:
1. Injector: ptrace attach → remote `mmap` → write payload path → remote `dlopen` → detach.
2. Payload native: constructor thread → attach JavaVM → `ActivityThread.currentApplication()` →
   load the Java payload Dex via `DexClassLoader` with the app's **real** ClassLoader.
3. Payload Java: `new OkHttpClient.Builder().build()` so the shield self-injects its signer
   interceptor → append a dump interceptor **after** the signer → `execute()` → capture the signed
   request → throw a stop exception to block network.
- Android 16 blocks writable DEX: `chmod 444` the payload DEX before load.

### B4. `app_process` standalone harness
Use when native state is simple enough to bootstrap manually (no full app runtime needed). Common
patches: JNIEnv helper, manual state init, force a URL gate, NOP a numeric-mismatch branch,
synthesize an `okio.Buffer` via a Java helper when the native buffer comes back null.

---

## PART C — Per-app lessons (illustrative — ONE app each, NOT universal)

Read these as examples of how the general principles manifest. Code values, offsets, and flags are
**build-specific** and must be re-derived, not reused verbatim. Each line names its source app.

| Lesson (app) | What happened | General point it illustrates |
|---|---|---|
| MB Bank 6.5.23: `1200` = "certificate of the response is not approved" still fired after the handshake branch was patched; `1005`/`NW1017` = "Connection refused" | two different result codes in the same log | A3: second (response-cert) verifier; A3: cert-reject vs connection-refused are different classes |
| MB Bank: server `getSHFiles` returns GW934 if any `.so`/dex/manifest bit changes | repack rejected | A5 + scope: server integrity → memory-only |
| MB Bank: `network_security_config.xml` pin had an expiry in the past | an expired XML pin was not the active blocker | A3: verify WHICH layer actually rejects; don't assume the stale pin |
| MB Bank: `pm clear` did not clear a server-flagged IP state | screen persisted until clean reinstall/reboot | A5 |
| ZaloPay: SSL pinning patched via `libjavacrypto.so+0x354F8` callback (`3f2303d5 fd7bbca9` → `00 00 80 52 c0 03 5f d6`) | a byte patch on the installed callback | A3: patch the installed callback, not the setter |
| ZaloPay: PID auto-detected by scanning `/proc/*/maps` (process name masquerades) | `cmdline` was misleading | A1: identify the process by mapped libs, not name |
| VNeID: built OkHttp with the app's real ClassLoader so BShield auto-added the signer interceptor | reuse in-process, not standalone | A4 |
| BShield: `19` = system proxy is a detection flag | setting a global `http_proxy` triggers it | Do NOT set a global proxy on shielded apps |

---

## Full real-case reference (canonical workspace)

| Case | Dir | Deliverable |
|---|---|---|
| BShield (VNEID + ZaloPay) | `BShield/docs/` + `BShield/tools/` | Xq signer reuse, SSL patch offsets, injector/harness |
| VNeID | `BShield/docs/VNEID_RESEARCH_FLOW_TEAM_WRITEUP.md` | in-process signer reuse `injector.c`/`xqpayload.cpp`/`PayloadMain.java` |
| ZaloPay | `BShield/docs/ZaloPay_Xqsig_research_journey.md`, `ZaloPay_Xqsig_Xqcode_RE_report.md` | standalone harness, callback pad, PID via `/proc/*/maps` |
| MB Bank 6.5.23 | `MB_Bank/` | multi-layer (Zimperium+V-Key+AnTian), server-hash, memory-only submit patch |
| MoMo | `momo/` | passive frida-file scan, `process_vm_readv` scanner |

Offsets/codes above are build-specific — re-derive per new build.

---

## MUST NOT

- Attach Frida when the shield scans maps/threads; use native mechanics only.
- Repack an app the server hashes (integrity reject) — patch memory, never the `.so` file.
- Patch `SSL_set_custom_verify` to disable pinning — it breaks the handshake.
- Flush unrelated `iptables -t nat` chains or set a global `http_proxy` on a shield-aware app.
- Leave `.c`/`.so`/`.dex` in `/data/local/tmp` after the read (workspace AGENTS.md).
- Clear Java exceptions globally without understanding the callsite — ART is sensitive.
- Repeat a per-app code (e.g. a bank's `1200`/`1005`) as if it were a universal rule.

## Task-complete self-check (MUST pass before claiming done)

- [ ] Which delivery mechanic was used and why (anti-frida / server hash / in-process state)?
- [ ] Address recorded as lib + offset/VA; base = `r--p` map start, verified by live read-back
- [ ] Every pid that can send is patched before its first event/handshake
- [ ] A real request decrypts / signer returns fresh valid output (sent flag, not a catalog string)
- [ ] Cert-reject vs connection-refused vs integrity-check distinguished by error text, not assumed
- [ ] Cleanup done: redirect/sniproxy/DNAT removed; no stray tool binary on device
