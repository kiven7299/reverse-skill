---
name: identity-federation
description: Use for authorized assessment of federated identity systems including SAML, OIDC, OAuth2 flows, SSO misconfiguration, and token confusion issues.
---

# Identity Federation (SAML / OIDC / OAuth)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read precedent-pentest; put SSO test accounts and IdP/SP scope into scope.md
2. `NOW`: MUST NOT lock real user accounts with brute-force attempts
3. `NEXT`: capture tools and docs (metadata URLs)
4. `ACT`: map the protocol flow → common mismatches → validate

## When to use

- SAML Response signature/assertion tampering surface (classic defect patterns)
- OIDC implicit/auth-code + missing PKCE
- redirect_uri / state / nonce issues
- IdP vs SP metadata, multi-tenant issuer confusion
- Complements `api-security` JWT attacks (this skill is federation and SSO flow)

## Workflow

```text
□ Draw: User → SP → IdP → Token → SP
□ Collect: /.well-known/openid-configuration, SAML metadata
□ Check: exact redirect_uri match, state binding, PKCE
□ Check: SAML signature coverage, algorithm downgrade
□ Session fixation and logout failure
```

## Toolchain

| Tool | Purpose |
|------|------|
| Burp + SAML Raider etc. | assertion edit (authorized) |
| jwt_tool | JWT segments |
| Browser DevTools | redirect chain |
| IdP admin logs | audit |

## References

- `references/sso-flow-checklist.md`
- `../api-security/` `../windows-ad/` (enterprise IdP)

## Routing context

**Upstream**: MASTER R37
**Downstream**: pure API JWT → api-security; cloud IdP → cloud-k8s

## Task-complete self-check

- [ ] Mapped the full SSO flow?
- [ ] Each Finding has repro and impact?
- [ ] Checklist?
