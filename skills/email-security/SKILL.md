---
name: email-security
description: Use for authorized email security review including phishing analysis, header authentication (SPF/DKIM/DMARC), BEC patterns, and mailbox token abuse research.
---

# Email Security & Phishing Analysis

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm authorization (analyze sample mail / tenant-config review)
2. `NOW`: do not re-deliver malware samples to real users
3. `ACT`: header auth → content/URL → attachment sandbox → tenant control-plane advice

## When to use

- Phishing-mail teardown and IOC
- SPF/DKIM/DMARC config assessment
- BEC (business email compromise) patterns
- OAuth-app phishing / mailbox-token abuse (pair with llm/cloud identity)
- Security-awareness exercise design (authorized)

## Workflow

```text
□ Full raw headers: Received chain, From/Return-Path consistency
□ SPF/DKIM/DMARC alignment results
□ URL sandbox and attachment static (pair with malware-analysis)
□ Brand impersonation vs reply-address mismatch
□ Tenant: anti-phish policy, external tagging, MFA, OAuth app consent
```

## Toolchain

| Tool | Purpose |
|------|------|
| Mail client "view source" | headers |
| dig/nslookup | SPF/DMARC records |
| urlscan / sandbox | links and attachments |
| Tenant admin center | policy |

## References

- `references/email-auth-checklist.md`
- `../malware-analysis/` `../attack-chain/` (phish stage) `../windows-ad/` (tokens)

## Routing context

**Upstream**: MASTER R36
**MUST NOT**: unauthorized bulk test-phish against third-party domains

## Task-complete self-check

- [ ] Header-auth conclusions complete?
- [ ] IOCs detection-ready (pair with threat-hunting)?
- [ ] Checklist?
