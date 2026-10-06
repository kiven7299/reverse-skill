# Attack-chain playbook cheat sheet

> Pick the playbook by target type. Each defines a standard path from initial access to the objective.

---

## Playbook 1: External web app → domain controller

```
1. Subdomain enum + port scan
2. Web fingerprint → known-vuln components
3. Exploit for webshell / RCE
4. Intranet recon (ipconfig/ifconfig, arp, net user)
5. Tunnel (frp/chisel/ssh)
6. Intranet scan (live hosts, open ports)
7. Credential harvest (mimikatz/hashdump/config files)
8. Lateral movement (PTH/WMI/PsExec)
9. Domain recon (BloodHound)
10. Domain privesc (Kerberoasting/DCSync/constrained delegation)
11. Domain-controller access
```

**Tool chain**: subfinder → httpx → nuclei → sqlmap/sstimap → frp → nmap → mimikatz → crackmapexec → bloodhound → certipy

---

## Playbook 2: Phish → intranet pentest

```
1. Target employee OSINT (LinkedIn/MaiMai)
2. Craft phish (spoofed sender / legitimate subject)
3. Build payload (macro doc/LNK/ISO/HTML smuggling)
4. Send phish
5. Wait for callback (C2 beacon)
6. Local recon + privesc
7. Credential extraction
8. Lateral movement
9. Persistence
10. Objective
```

**Tool chain**: theHarvester → gophish → msfvenom/cobalt-strike → mimikatz → bloodhound

---

## Playbook 3: Near-source → intranet

```
1. Physical recon (WiFi signal, badge type, USB ports)
2. WiFi attack (Fluxion fake AP / WPA crack)
   or BadUSB implant (Rubber Ducky keystroke inject)
   or network implant (Raspberry Pi / LAN Turtle)
3. Get an intranet foothold
4. Intranet scan
5. Continue from Playbook 1 steps 5-11
```

**Tool chain**: fluxion/aircrack-ng → rubber-ducky → frp → nmap → crackmapexec

---

## Playbook 4: Cloud pentest

```
1. Cloud asset discovery (subdomain → CNAME → cloud vendor)
2. Bucket enum (S3/OSS/Blob public access)
3. SSRF → cloud metadata (169.254.169.254)
4. Get temp creds (AK/SK/Token)
5. Cloud API enum (IAM/EC2/Lambda/RDS)
6. Privilege escalation (PassRole/AssumeRole)
7. Lateral (cross-account/cross-region)
8. Data access
```

**Tool chain**: subfinder → nuclei(ssrf) → aws-cli → pacu → ScoutSuite

---

## Playbook 5: Bug bounty / SRC quick hits

```
1. Asset collection (subdomains + ports + JS files)
2. Fingerprint → fast known-vuln verify (nuclei)
3. Parameter discovery (arjun/paramspider)
4. Per-class tests:
   - IDOR/authz (change ID/role)
   - SSRF (intranet probe/cloud metadata)
   - SQLi (sqlmap)
   - XSS (xsstrike)
   - File upload (bypass detection)
   - Logic flaws (payment/captcha/password reset)
5. Write PoC + submit report
```

**Tool chain**: subfinder → httpx → nuclei → arjun → sqlmap → xsstrike → burpsuite

---

## Playbook 6: AD CS certificate attack

```
1. Discover AD CS (certipy find)
2. Identify vulnerable templates (ESC1-ESC8)
3. Request a malicious certificate
4. Authenticate as the target user with the cert
5. Get NTLM hash or TGT
6. DCSync all credentials
```

**Tool chain**: certipy → rubeus → mimikatz → secretsdump

---

## Decision matrix

| Current state | Next priority |
|---------------|---------------|
| Domain only | subdomain enum → port scan → web fingerprint |
| Web vuln in hand | get shell → intranet recon |
| Low-priv shell | privesc → credential extract |
| One intranet host | tunnel → intranet scan → lateral |
| Domain-user creds | BloodHound → attack path |
| Domain-admin hash | DCSync → Golden Ticket |
| Cloud AK/SK | enum perms → privesc → data access |
| Phish callback | local privesc → creds → lateral |
| Near-source access | intranet scan → same as above |
