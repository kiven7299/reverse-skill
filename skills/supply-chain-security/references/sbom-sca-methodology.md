# SBOM + SCA methodology

## SBOM standard comparison

| Standard | Format | Ecosystem | Prefer when |
|------|------|------|---------|
| SPDX | JSON/YAML/tag-value | Linux Foundation, Yocto | License compliance first |
| CycloneDX | JSON/XML | OWASP, Kubernetes | Security analysis first |
| SWID | XML | ISO | Enterprise asset management |

## SBOM generation toolchain

```bash
# cdxgen: CycloneDX SBOM from source
cdxgen -o bom.json -t cyclonedx

# Syft: from container/filesystem
syft nginx:latest -o spdx-json > sbom.spdx.json

# SBOM-Tool: Microsoft toolchain
sbom-tool generate -b ./build -bc ./src -pn MyApp -pv 1.0
```

## SCA tool comparison

| Tool | Free | Speed | Database | Reachability |
|------|:--:|------|--------|:--:|
| OSV-Scanner | ✅ | very fast | OSV.dev | ❌ |
| Trivy | ✅ | fast | multi-source | ❌ |
| Dependency-Track | ✅ | medium | NVD+OSV+GitHub | ❌ (plugin) |
| Snyk | ❌ | medium | proprietary | ✅ |
| CodeQL | ✅ | slow | code-level | ✅ |

## Vuln priority

```
CVSS ≥ 9.0 + public PoC + reachable → P0 fix now
CVSS ≥ 7.0 + PoC + reachable → P1 this week
CVSS ≥ 7.0 + no PoC or not reachable → P2 next iteration
Rest → normal process
```

## Manual verify (3 steps)

```bash
# 1. Confirm version (do not trust SBOM fields blindly)
# In container: dpkg -l | grep <package>
# Node: cat node_modules/<pkg>/package.json | jq .version
# Python: pip show <package>

# 2. Confirm vuln
# Search CVE: https://osv.dev / https://nvd.nist.gov
# Check affected version range
# Find GitHub Advisory / oss-security list

# 3. Verify impact
# Search public PoC: GitHub/Exploit-DB
# Exploit conditions: auth/local access/specific config required?
# Isolated verify: docker run --rm -it vulnerable-image bash
```

## Continuous monitoring

```yaml
# Daily SBOM refresh + scan
schedule:
  - cron: "0 6 * * *"  # 06:00 daily
    steps:
      - cdxgen -o bom.json
      - osv-scanner scan --sbom bom.json
      - trivy fs --exit-code 1 --severity CRITICAL .
```

Source: OWASP CycloneDX, SPDX, Google OSV, CISA SBOM Guidance
