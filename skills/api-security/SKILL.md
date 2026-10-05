---
name: api-security
description: Use for authorized security assessment of REST, GraphQL, WebSocket, or SOAP APIs, including discovery, authentication, authorization, rate-limit, and CI/CD testing.
---
# API security testing

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

> Covers REST / GraphQL / WebSocket / SOAP
> 10-phase methodology, discovery through CI/CD integration

## When to use

- REST API security testing (OpenAPI/Swagger-driven or blind)
- GraphQL security audit (introspection, batch queries, alias overload)
- WebSocket security testing
- JWT / OAuth 2.0 auth testing
- BOLA/IDOR/BFLA authorization vulns
- API rate-limit bypass and DoS testing

## 10-phase test flow

### Phase 1: API discovery and recon

```text
Active discovery:
□ Vespasian: headless crawl → auto-generate OpenAPI 3.0 / GraphQL SDL specs
□ Entropy --discover: extract endpoints from robots.txt + JS files
□ Kiterunner / ffuf: brute undocumented endpoint paths
□ Check common paths: /swagger.json, /openapi.json, /graphql, /api-docs

GraphQL introspection (three attempts):
  1. Standard introspection query
  2. Minimal query (bypass WAF full-block)
  3. Query only __schema { types { name } } (smallest probe)
```

### Phase 2: Auth testing

```text
JWT analysis (jwt_tool / Burp):
□ alg:none attack: set header "alg":"none", empty signature
□ Key confusion: RS256 public key → HS256 symmetric key
□ Weak HMAC key brute: jwt_tool -C -d wordlist.txt
□ Expiry/claim tamper: change exp/iat/sub/role claims
□ kid injection: ../../etc/passwd → HMAC signature bypass

OAuth 2.0:
□ redirect_uri manipulation → auth-code leak
□ CSRF via missing state param
□ Token leak in Referer header
□ Missing PKCE detection

GraphQL auth:
□ mutation via GET to bypass auth (CSRF)
□ Batch-query auth bypass
```

### Phase 3: Authorization testing (BOLA/IDOR/BFLA)

```text
BOLA (object-level authz bypass):
□ Enumerate numeric IDs: /user/1 → /user/2 → /user/3
□ Enumerate UUIDs
□ Enumerate usernames/emails
□ Burp Autorize: dual-session replay compare

BFLA (function-level authz bypass):
□ Regular user calls admin API
□ HTTP method switch: GET → PUT → PATCH → DELETE
□ API version downgrade: /v2/admin → /v1/admin
□ Bulk-op injection: {"users": [1,2,3]} → {"users": [1,2,3,admin_id]}

Tools: Burp Autorize, AuthMatrix, Entropy (malicious_insider persona)
```

### Phase 4: GraphQL specials

```text
Introspection leak → info exposure
Alias overload → 100+ alias DoS
Batch queries → 10+ concurrent query DoS
Field repeat → __typename × 500
Directive overload → recursive @skip/@include
Cyclic queries → deep nested introspection recursion
Field suggestions → error-message info leak
GraphiQL/Playground exposed → public IDE risk
GET mutations → CSRF risk
Trace/debug mode → metadata leak

Tools: FireTail, Escape DAST, api.sh (Phases 1-3)
```

### Phase 5: REST input validation

```text
□ HTTP method switch: GET→POST→PUT→DELETE→OPTIONS→PATCH
□ Content-Type tamper: JSON→XML→multipart
□ NoSQL injection: {"username": {"$gt": ""}}
□ SSRF via URL params: webhook URL/avatar URL/import URL
□ XXE in XML endpoints
□ Parameter pollution: /api?role=user&role=admin
□ Mass assignment: add is_admin: true to the body
```

### Phase 6: Business logic and differential testing

```text
□ Entropy compare: diff v1 vs v2 API → status-code change/field drop/latency regression
□ Multi-role workflow tests: admin/user/readonly permission matrix
□ Coupon/points/price manipulation
□ Race: concurrent requests for TOCTOU
```

### Phase 7: WebSocket testing

```text
□ Endpoint discovery
□ Message injection (inject payload, prototype pollution)
□ Oversized message handling
□ Type confusion
□ Cross-site WebSocket hijacking (CSWH)
```

### Phase 8: Rate limit and DoS

```text
□ Rate-limit bypass via headers: X-Forwarded-For, X-Real-IP
□ Path variants: /api/ → /api → /Api/ → /API/
□ Slowloris low-bandwidth exhaustion
□ GraphQL batch deep-nest DoS
□ IP rotation tests (ProxyCat proxy pool)
```

### Phase 9: Data exposure

```text
□ Over-exposed responses: compare API return vs UI display
□ Pagination enum: ?page=1&limit=10000
□ Error-message info leak: stack traces/internal paths/SQL errors
□ GraphQL nested traversal to unauthorized data
□ OpenAPI spec exposes sensitive endpoints
```

### Phase 10: CI/CD integration

```text
□ Entropy --ci --watch: auto-rerun on spec change
□ Escape DAST: auto-block builds by severity threshold
□ Persist discoveries as regression tests
□ StackHawk (developer-first, ZAP kernel)
```

## Toolchain

| Tool | Use | Get |
|------|------|------|
| Vespasian | Traffic → OpenAPI/GraphQL spec | GitHub: praetorian-inc/vespasian |
| Entropy | LLM-generated attack scenarios, 5 personas | GitHub: arjinexe/entropy-chaos |
| Escape DAST | Business-logic security testing | escape.tech |
| api.sh | 8-phase all-protocol attack pipeline | GitHub: Sharon-Needles/api |
| FireTail | GraphQL 12 specials | firetail.ai |
| jwt_tool | Full JWT testing | GitHub: ticarpi/jwt_tool |
| Burp Autorize | Dual-session authz compare | Burp BApp Store |

## References

- `references/rest-graphql-testing.md` — REST + GraphQL deep testing
- `references/jwt-oauth-testing.md` — JWT + OAuth security testing


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the RULES Checklist items?
