---
name: native-instrumentation
description: Use for Android app-shield / RASP bypass WITHOUT Frida — in-memory patch, native no-Frida instrumentation, Xqsig/Xqcode request-signer reuse, SSL certificate-pinning bypass via Conscrypt/BoringSSL callback patch, and Zygisk path hiding. Applies when Frida is detected, the app hashes its own files (server SHA256 integrity), or an OkHttp/native interceptor must be re-used for signing. Backed by real cases in the canonical workspace (BShield/VNeID/ZaloPay/MB Bank).
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the task is a client-side shield bypass on a device/app already in scope
3. `NEXT`: read the canonical real-case workspace:
   `D:\OneDrive_nguyenchilam\OneDrive\CyberSecurity\Research\Mobile_security\App_protection\`
   - `WORKSPACE_PLAN.md`, `_agent_worklog/TODO.md`, `AGENTS.md`
   - `_methodology/07_native_c_hooking.md`, `06_ssl_pinning_bypass.md`, `02_dynamic_analysis.md`
   - per-app real cases: `BShield/`, `MB_Bank/`, `momo/`
4. `NEXT`: verify tool availability (`../tool-index.md` and `../../TOOLS.md`); rebuild patchers from workspace source, never keep stray binaries on the device
5. `ACT`: choose a delivery mechanic below and execute

# Native shield instrumentation (no-Frida)

## Scope

Prefer this skill when the client app:

- detects / rejects Frida (maps scan, port 27042, `frida-agent` string, ZygiskFrida injection)
- server-hashes the app's own `.so`/dex (one changed bit → `getSHFiles`/GW934) so repacking is impossible
- signs or encrypts requests via a native `OkHttp.Interceptor` (Xqsig/Xqcode/Xqplatform, BShield, V-Key)
- pins TLS at Conscrypt / BoringSSL (inside `libjavacrypto.so` / `libflutter.so`), not just OkHttp Java pins

Toolchain: external memory patch, Zygisk module, `app_process` standalone harness, or
`ptrace` + remote `dlopen` in-process reuse. No Frida attach, no `Interceptor.attach`.

## Delivery mechanics (ranked by detectability)

From `_methodology/07_native_c_hooking.md`. Rank 1 is the default.

| Rank | Method | What it leaves in the target | Real case |
|---|---|---|---|
| 1 | External memory write (`pwrite /proc/<pid>/mem`) | changed bytes only; no map, no tracer | ZaloPay SSL callback, MB `zpatch`/`flpatch2`, V-Key submit |
| 2 | Zygisk `.so` (`postAppSpecialize`) | extra map, zygote injection | MB `mbpathlog` (observation), path-hide |
| 3 | `ptrace` + remote `dlopen` | `TracerPid` during attach + payload map | VNeID signer-reuse `injector.c`/`xqpayload.cpp` |
| 4 | `app_process` standalone harness | runs in own process | ZaloPay `XqDirectSigner` |
| 5 | `LD_PRELOAD` | env var + mapped `.so` | avoid against shields |

### 1. External memory patch (default)

Only changes instructions — cannot build OkHttp objects or call Java. Works for
`mov w0,#0; ret` / branch flips / NOPs.

Reusable tool: `_tools/native_hooks/mem_patch/mem_patch.c` (ARM64, base + VA, live read-back).

```
mem_patch <pid> <libname> <vaddr_hex> [hexbytes...]     # no bytes = read-only dump
mem_patch 29362 libempirememorial.so a9cd34 00 00 80 52 c0 03 5f d6
```

Rules (from real cases):
- Live address = `/proc/<pid>/maps` **`r--p` load base** + VA (offset-0), NOT the `r-xp` map start (that reads zeros).
- Reliable live read = `open("/proc/<pid>/mem") + pread`; a shell `dd` read returns zeros even on pid 1 on this device.
- Multi-process apps respawn a storm of short-lived pids; Flutter network runs in child isolates → patch **every** pid that can send, verify read-back each time, before the first handshake/event.
- A patch that lands after the event already fired does not falsify the function.

### 2. Zygisk path hide

Hide `su`/`magisk`/`kernelsu`/`frida`/`/data/adb`/`/proc/modules` probes from a denylisted
package. See `_tools/native_hooks/zygisk_hide/`. Requires NDK **r26+** (device NDK r21 too old
for `android-34`). Load only into the target package; do NOT also enable ZygiskFrida.

### 3. In-process signer reuse (VNeID pattern)

When the signer needs full app runtime state (Application object, real PathClassLoader, BShield
VM/state), standalone hangs/crashes. Reuse the real signer:

1. `injector.c` — ptrace attach, remote `mmap`, write payload path, remote `dlopen`, detach.
2. `xqpayload.cpp` — constructor thread → attach JavaVM → `ActivityThread.currentApplication()` →
   load DEX via `DexClassLoader` with the app's **real** ClassLoader.
3. `PayloadMain.java` — `new OkHttpClient.Builder().build()` so BShield auto-injects its signer
   interceptor → add a dump interceptor **after** the signer → `execute()` → capture signed request →
   `throw IOException("STOP_AFTER_SIGN_PRINT")` to block network.

Real case: `BShield/docs/VNEID_RESEARCH_FLOW_TEAM_WRITEUP.md` §16.

Android 16 blocks writable DEX: `chmod 444` the payload DEX before load.

### 4. `app_process` standalone harness (ZaloPay pattern)

When native state is simple enough to bootstrap manually. Real case:
`BShield/docs/ZaloPay_Xqsig_research_journey.md`. Patches: JNIEnv helper, manual state init,
force URL gate, NOP numeric-mismatch branch, synthesize `okio.Buffer` via a Java helper when the
native one returns null (POST signing depends on real body bytes).

## SSL pinning by-pass, no-Frida (`_methodology/06_ssl_pinning_bypass.md`)

Layered; test each separately and keep a per-host map.

| Layer | Where | Patch |
|---|---|---|
| Proxy ignore | app never uses system proxy | invisible/VPN redirect |
| CA trust store | `/apex/com.android.conscrypt/cacerts/` | mount CA in |
| Java pin | OkHttp `CertificatePinner`, Conscrypt `TrustManagerImpl` | hook to no-op (use Frida only if no anti-frida) |
| Native/Conscrypt | `libjavacrypto.so` cert-verify callback | patch the **installed callback**, not the setter |
| Flutter bank TLS | BoringSSL inside `libflutter.so` | patch the cert-verify callback branch |
| Second verifier | post-handshake response-cert check | locate by stack trace the instant it logs |

Hard rules from real cases:
- **Do NOT patch `SSL_set_custom_verify` / `SSL_CTX_set_custom_verify`** — breaking it removes the
  custom callback and default verify fails with `CERTIFICATE_VERIFY_FAILED`. Patch the callback
  the library installed (ZaloPay `libjavacrypto.so+0x354F8`: `3f2303d5 fd7bbca9` → `00 00 80 52 c0 03 5f d6`).
- **`1200` is a SECOND post-handshake response-cert verifier**, not `1005/NW1017` (Connection
  refused). Patching only the handshake branch (`libflutter.so 0x6e8718`) still leaves `1200`.
- An expired `network_security_config.xml` pin is not the active blocker (MB: expired 2026-04-29).
- `pm clear` does not clear a server-side IP/device flag; that needs clean reinstall+reboot.

## Xqsig / request-signer reuse (`_methodology/10_api_protection_bypass.md`)

- The signer needs runtime state → reuse the in-process signer, do not reimplement offline.
- "Signer reuse" ≠ decrypt/recover private keys; it only feeds valid input and asks the runtime
  signer to sign. Real cases: ZaloPay `sign_and_replay.py` (standalone oracle), VNeID `gen_vneid_signed_curl.py` (in-process reuse).

## Real-case reference (canonical workspace)

| Case | Dir | Deliverable |
|---|---|---|
| BShield (VNEID + ZaloPay) | `BShield/docs/` + `BShield/tools/` | Xqsig signer reuse, SSL patch offsets, injector/harness |
| VNeID | `BShield/docs/VNEID_RESEARCH_FLOW_TEAM_WRITEUP.md` | in-process signer reuse, `injector.c`, `xqpayload.cpp`, `PayloadMain.java` |
| ZaloPay | `BShield/docs/ZaloPay_Xqsig_research_journey.md`, `ZaloPay_Xqsig_Xqcode_RE_report.md` | standalone harness, `libjavacrypto.so+0x354F8`, PID auto-detect via `/proc/*/maps` |
| MB Bank 6.5.23 | `MB_Bank/` | `libempirememorial.so d_zdevice_event_submit` patch `0xa9cd34`, Flutter verifier `0x6e8718`, server-hash GW934 |
| MoMo | `momo/` | passive frida-file scan (1005), `process_vm_readv` scanner |

Offsets are **build-specific** — re-derive per new build; never reuse across versions.

## MUST NOT

- Do not attach Frida when the shield scans maps/threads; use native mechanics only.
- Do not repack an app the server hashes (GW934). Patch memory, never the `.so` file.
- Do not patch `SSL_set_custom_verify` to disable pinning — breaks the handshake.
- Do not flush unrelated `iptables -t nat` chains or set a global `http_proxy` (flag 19).
- Do not leave `.c`/`.so`/`.dex` in `/data/local/tmp` after the read (workspace AGENTS.md).
- Do not clear Java exceptions globally without understanding the callsite — ART is sensitive.

## Task-complete self-check (MUST pass before claiming done)

- [ ] Which delivery mechanic was used, and why (not Frida, server hash, or in-process state)?
- [ ] Target address recorded as lib + offset/VA, not just name; base = `r--p` map start
- [ ] Patch bytes confirmed by live read-back on the pid that actually sends
- [ ] Every pid that can send the host is patched before its first event/handshake
- [ ] A real app request **decrypts** in Burp / signer returned a fresh valid output (sent flag, not a catalog string)
- [ ] `1200` / `1005` / GW codes disambiguated (cert vs connection vs integrity)
- [ ] Cleanup: redirect/sniproxy/DNAT removed; no stray tool binary left on device
