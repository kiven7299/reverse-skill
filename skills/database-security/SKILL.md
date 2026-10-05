---
name: database-security
description: Use for authorized database security assessment covering PostgreSQL/MySQL/MSSQL/Mongo/Redis exposure, authz, UDF/command paths, and misconfiguration review.
---

# Database Security Assessment

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read precedent-pentest; **MUST NOT run destructive statements on production DBs** unless explicitly allowed
2. `NOW`: scope MUST name instance, account rights, and whether write/delete is allowed
3. `NEXT`: client-tool paths
4. `ACT`: exposure → authn → authz → config → exploit-chain validation (safe)

## When to use

- DB unauthorized/weak password/wrong bind 0.0.0.0
- Over-privilege, dangerous features (xp_cmdshell, COPY PROGRAM, UDF)
- Lateral: from app account to DBA
- NoSQL injection and Redis write-file (authorized env)

## Workflow

```text
□ Network exposure and TLS
□ Account roles and grantee
□ Sensitive-table access control
□ Dangerous config: file_priv, xp_cmdshell, load_file
□ Whether audit logs are on
□ Backup and snapshot privileges
```

## Toolchain

| Tool | Purpose |
|------|------|
| Official CLI | connect and enumerate |
| sqlmap | injection validation (authorized) |
| nuclei | known-exposure templates |
| Cloud RDS console audit | config |

## References

- `references/db-misconfig-checklist.md`
- `../pentest-tools/` `../cloud-k8s/`

## Routing context

**Upstream**: MASTER R35
**Downstream**: got OS command → attack-chain; cloud-hosted → cloud-k8s

## Task-complete self-check

- [ ] Avoided unauthorized write/delete?
- [ ] Distinguished config issues from exploitable chains?
- [ ] Checklist?
