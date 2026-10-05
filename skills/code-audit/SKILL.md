---
name: code-audit
description: Use for authorized source-code security review and SAST workflows including Semgrep, CodeQL patterns, dangerous API hunting, and fix verification.
---

# Source Code Security Audit

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` or code-audit authorization
2. `NOW`: confirm **source/repo access** (binary-only → switch to an RE skill)
3. `NOW`: pin the language stack and scope (dir/service/PR diff)
4. `NEXT`: tool-index; semgrep etc.
5. `ACT`: threat-model sketch → auto scan → human validation

## When to use

- White-box audit, PR/diff security review
- Semgrep / CodeQL / Bandit / gosec and other SAST
- Dangerous APIs, injection points, missing authz, crypto misuse
- Split with `supply-chain-security/`: this skill is **first-party code logic**; supply-chain is deps and pipelines

## Workflow

### 1. Scope and threat model

```text
□ Trust boundaries: user input, files, deserialization, SSRF, auth middleware
□ High-value assets: auth, payment, admin, key handling
```

### 2. Auto scan

```bash
semgrep --config auto .
# or project rule packs
semgrep --config p/owasp-top-ten .
```

### 3. Human validation (MUST)

```text
□ Each SAST hit: reachable? exploitable? false positive?
□ Authz: IDOR/privilege, missing checks, broken multi-tenant isolation
□ Injection: SQL/command/template/LDAP
□ Crypto: hardcoded keys, ECB, homemade crypto
```

### 4. Output

```text
Finding: location + data flow + PoC + fix advice
Optional ATT&CK / CWE IDs
```

## Toolchain

| Tool | Language/scenario |
|------|-----------|
| Semgrep | multi-language fast rules |
| CodeQL | deep dataflow (GitHub) |
| Bandit | Python |
| gosec / staticcheck | Go |
| SpotBugs / FindSecBugs | Java |

## References

- `references/sast-review-checklist.md`
- `../supply-chain-security/` `../api-security/` `../llm-security/` (Agent code)

## Routing context

**Upstream**: MASTER R26
**Role**: `ops/role-map.md` cae
**Downstream**: dependency vulns → supply-chain; runtime validation → pentest-tools

## Task-complete self-check

- [ ] Human-validated, not just pasted scanner output?
- [ ] Includes fix advice?
- [ ] Limited to the authorized repo scope?
- [ ] Checklist?
