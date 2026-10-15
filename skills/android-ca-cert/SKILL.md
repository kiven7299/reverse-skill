---
name: android-ca-cert
description: Install a custom CA certificate (Burp/mitmproxy/own) as a SYSTEM-trusted certificate on Android to decrypt HTTPS traffic that ignores user CAs. Covers Android 7-13 (system store), Android 14+ (Conscrypt APEX + per-namespace injection via Zygote/app mount namespaces), and Android 16 (new store location). For apps using default system trust AND for pinning that only checks the chain against the store. Trigger keywords: install system CA, trust user cert, Android 14 APEX cacerts, conscrypt, inject certificate, ssl pinning system cert, emulated cert trust, http toolkit android cert.
---

# Install a System-Trusted CA on Android (for HTTPS decryption)

## What this skill is for

You want Burp / mitmproxy / your own CA to be trusted by an app so its HTTPS
traffic decrypts in your proxy. A **user CA** (`Settings > Install cert > CA`) is
NOT enough for many apps, especially on Android 7+ where apps can opt out of
user-added CAs. You need the CA as a **system trust**.

This skill installs the CA into the Android **system/store CA store** so that any
network stack that reads the platform trust store (Conscrypt, BoringSSL,
OkHttp default, WebView, dart:io/Flutter) will validate against it.

> Scope note: this fixes chain verification against the platform store. If an
> app additionally pins a leaf/public-key hash in a custom verify callback, this
> alone won't bypass it — see the "pinning vs chain" caveat at the end.

## Android version → where CAs live

| Android | System CA path | Default user CA honored? |
|---------|----------------|--------------------------|
| < 7     | `/system/etc/security/cacerts` | yes (all apps) |
| 7 – 13  | `/system/etc/security/cacerts` | no (opt-out apps) |
| 14 – 15 | `/system/etc/security/cacerts` **+** `/apex/com.android.conscrypt/cacerts` (Conscrypt APEX mirror), must also inject into per-app mount namespaces | no |
| 16+     | new location (see Method 5/16 note) | no |

- Android **7 – 13**: enough to overlay `/system/etc/security/cacerts`.
- Android **14+**: because user/app trust is served from the **Conscrypt APEX**
  (`/apex/com.android.conscrypt/cacerts`) and each app runs in its **own mount
  namespace**, you must ALSO bind the store into the APEX path inside the
  Zygote **and** each already-running app namespace. Skipping the APEX/namespace
  step is the #1 reason "I installed the system cert but it still doesn't trust"
  — the app reads the APEX mirror, not your `/system/etc` overlay.

## Requirements

- Rooted device (Magisk or KernelSU). USB debugging on.
- `adb`, `bash` available, and the device shell has `su`.
- Your CA cert as a PEM (and its hashed filename — see below).
- Because the app may be recreated after the fixture, re-run the injection
  whenever the target app restarts (see "Keeping it applied" section).

## Step 0 — compute the CA hashed filename

The CA file must be named `<subject-hash>.0` (OpenSSL legacy
`X509_NAME_hash_old`, first 8 hex of MD5 over the DER Subject). Compute it
with Python (`cryptography`) if no `openssl` is on the host:

```python
import hashlib, glob
from cryptography import x509
from cryptography.hazmat.primitives.serialization import Encoding
cert = x509.load_pem_x509_certificate(open("your_ca.pem","rb").read())
subj_der = cert.subject.public_bytes()
print("filename:", hashlib.md5(subj_der).hexdigest()[:8] + ".0")
```

With `openssl` on the host it is simply: `openssl x509 -in your_ca.pem -subject_hash -noout` → append `.0`.

Push it onto the device as e.g. `/data/local/tmp/<hash>.0`.

## Method A — Android 7–13 (simple system overlay)

```sh
adb push your_ca.pem /data/local/tmp/<hash>.0
adb shell su -c '
  mkdir -p /data/local/tmp/cacerts
  cp /system/etc/security/cacerts/* /data/local/tmp/cacerts/
  cp /data/local/tmp/<hash>.0 /data/local/tmp/cacerts/
  chmod 644 /data/local/tmp/cacerts/*
  mount --bind /data/local/tmp/cacerts /system/etc/security/cacerts
'
```

Verify: `adb shell su -c 'ls /system/etc/security/cacerts/<hash>.0'`.

## Method B — Android 14+ (Conscrypt APEX + per-namespace injection) [Method 4]

> Source: HTTP Toolkit's Android CA injection method.
> Manual writeup: https://httptoolkit.com/blog/android-14-install-system-ca-certificate/

This is the definitive method for Android 14+. It overlays `/system/etc`
with a tmpfs holding the full store + your cert, then bind-mounts that overlay
into the **Conscrypt APEX path** inside the shell, the **Zygote** namespaces, and
every **already-running app** namespace.

1. Push your cert to the device and make a copy matching `<hash>.0`:
   `adb push your_ca.pem /data/local/tmp/<hash>.0`

2. Create/run this script on the device as root (the `CERTIFICATE_PATH` is the
   directory holding your `<hash>.0` file):

```sh
#!/system/bin/sh
set -e  # fail on error

CERTIFICATE_PATH="/data/local/tmp"

# Snapshot the existing store
mkdir -p /data/local/tmp/htk-ca-copy
chmod 700 /data/local/tmp/htk-ca-copy
rm -rf /data/local/tmp/htk-ca-copy/*

if [ -d "/apex/com.android.conscrypt/cacerts" ]; then
    cp /apex/com.android.conscrypt/cacerts/* /data/local/tmp/htk-ca-copy/
else
    cp /system/etc/security/cacerts/* /data/local/tmp/htk-ca-copy/
fi

# In-memory overlay over the system store path
mount -t tmpfs tmpfs /system/etc/security/cacerts
mv /data/local/tmp/htk-ca-copy/* /system/etc/security/cacerts/

# Put your CA in
cp "${CERTIFICATE_PATH}"/*.0 /system/etc/security/cacerts/

# Fix permissions + SELinux context
chown root:root /system/etc/security/cacerts/*
chmod 644 /system/etc/security/cacerts/*
chcon u:object_r:system_file:s0 /system/etc/security/cacerts/
chcon u:object_r:system_file:s0 /system/etc/security/cacerts/*

# ---- Android 14+: inject into the Conscrypt APEX in each mount namespace ----
if [ -d "/apex/com.android.conscrypt/cacerts" ]; then
    # shell's own namespace
    mount --bind /system/etc/security/cacerts /apex/com.android.conscrypt/cacerts

    # Zygote daemons: every app forked from them inherits the mount
    ZYGOTE_PID=$(pidof zygote || true)
    ZYGOTE64_PID=$(pidof zygote64 || true)
    Z_PIDS="$ZYGOTE_PID $ZYGOTE64_PID"
    for Z_PID in $Z_PIDS; do
        if [ -n "$Z_PID" ]; then
            nsenter --mount=/proc/$Z_PID/ns/mnt -- \
                /bin/mount --bind /system/etc/security/cacerts /apex/com.android.conscrypt/cacerts
        fi
    done

    # Already-running apps (descendants of the Zygotes)
    APP_PIDS=$(
        echo $Z_PIDS | \
        xargs -n1 ps -o PID= -P | \
        awk '{print $1}'
    )
    for PID in $APP_PIDS; do
        nsenter --mount=/proc/$PID/ns/mnt -- \
            /bin/mount --bind /system/etc/security/cacerts /apex/com.android.conscrypt/cacerts &
    done
    wait

    echo "APEX certificates remounted for $(echo $APP_PIDS | wc -w) apps"
fi

rm -rf /data/local/tmp/htk-ca-copy
echo "System cert injected"
```

3. `adb push inject.sh /data/local/tmp/` then
   `adb shell su -c 'sh /data/local/tmp/inject.sh'`.

4. Restart the target app (or let the injection hit its namespace if it was
   already running). Verify the app now shows decrypted traffic in Burp.

## Keeping it applied (per-app restart caveat)

- The Zygote namespace injection means **newly forked apps inherit** the overlay.
- The "already-running app" loop only covers apps running **at injection time**.
  An app that was killed and respawns from a Zygote snapshot taken before the
  overlay may not carry it. If the target app respawns (or you `force-stop` +
  relaunch it), **re-run the injection script while the new process is alive**
  so its namespace gets the bind mount.
- tmpfs overlays are lost on reboot → re-run after every boot.

## Verification checklist

- `su -c 'ls /system/etc/security/cacerts/<hash>.0'` exists.
- `su -c 'ls /apex/com.android.conscrypt/cacerts/<hash>.0'` exists after injection.
- Proxy shows decrypted HTTPS for the target host.
- If it still fails with a cert error even after this, it is **not** a plain
  chain-verify problem: it's an app-level pin (leaf/public-key hash in a custom
  verify callback, common in banking/Flutter/dart:io `badCertificateCallback`).
  In that case: (a) patch the app's verify callback to always accept, or
  (b) replace the pinned cert with your proxy's cert signed by a CA the app
  trusts, or (c) patch at the BoringSSL/Conscrypt verify layer in-process.

## Troubleshooting

- "still untrusted after install": you skipped the APEX/namespace step, or the
  app restarted after the loop. Re-run while the app is alive (Method B).
- `mount -t tmpfs` fails: ensure `su` grants mount; on some kernels use
  `mount --bind` of a prepared directory (Method A style) for
  `/system/etc/security/cacerts` instead of tmpfs, then still do the APEX
  bind-mount + namespace injection.
- `chcon` fails: your device may be permissive or lack the context; try
  `chcon u:object_r:system_file:s0` ignoring failure, or skip if store already
  has that context.
- SELinux denials for `nsenter`/mount: run everything under `su`; if a custom
  rule blocks it, temporarily `setenforce 0` (test device) and restore.
