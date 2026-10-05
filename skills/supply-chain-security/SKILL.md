---
name: supply-chain-security
description: Use for software supply-chain security assessment covering SBOM, SCA, CI/CD pipelines, container images, build integrity, dependency provenance, and vulnerability reachability.
---
# Supply Chain Security Testing

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

> SBOM / SCA / CI/CD pipeline / dependency provenance
> Regulation-driven: US EO SBOM, China GB, EU CRA

## When to use

- Software supply-chain security assessment
- OSS dependency vuln scan and validation
- CI/CD pipeline security audit
- Container-image security analysis
- Third-party component compliance review
- Build-artifact provenance and integrity

## Six-layer supply-chain governance

```text
Layer 1: Source-trust assessment → upstream repo/maintainers/release-history review
Layer 2: Build-pipeline integration → CI/CD security gates, signature verify
Layer 3: Artifact-distribution integrity → signatures, checksums, attached SBOM
Layer 4: Runtime protection → container scan, admission control
Layer 5: Continuous monitoring → real-time CVE tracking, vuln reachability
Layer 6: Incident response → supply-chain attack IR, rollback strategy
```

## Workflow

### 1. SBOM generate and audit

```text
Generate SBOM:
□ CycloneDX format: cdxgen → bom.json
□ SPDX format: sbom-tool generate
□ Syft: syft <image|dir> -o spdx-json

Audit points:
□ Unknown/unauthorized dependencies
□ Deprecated/unmaintained packages
□ License-conflict detection
□ Direct vs transitive dependency inventory
□ Each component's release timeline and maintainer status
```

### 2. Software composition analysis (SCA)

```bash
# OSV-Scanner (free, Google-maintained)
osv-scanner scan -r . --format json

# OWASP Dependency-Track (enterprise continuous monitoring)
docker run -p 8080:8080 dependencytrack/apiserver
# → upload SBOM → auto-match NVD/OSV/GitHub Advisory

# Snyk (commercial)
snyk test --all-projects
snyk monitor  # continuous monitoring

# Trivy (container + deps + IaC)
trivy fs .          # filesystem scan
trivy image nginx   # container image
trivy config .      # IaC config
```

### 3. Vulnerability reachability

```text
SCA alert ≠ actual risk! Most SCA tools have only ~15% of alerts actually reachable.

Validation steps:
1. Get CVE list via Dependency-Track or Trivy
2. Filter CVSS ≥ 7.0
3. Reachability analysis for CVEs with a PoC
   - Code Property Graph slice: trace user input to the vuln function
   - DEPTEX method: EPD (Execution Path Dominance) + LLM semantic validation
4. Validate the PoC in an isolated env
5. Rank reachable vulns by actual impact for fix priority
```

Tool refs:
- CodeQL: GitHub code query → dataflow analysis
- Snyk Code: reachability flags
- DEPTEX: LLM-assisted context-aware risk assessment

### 4. CI/CD pipeline security

```text
Security checkpoints:
□ Commit → pre-commit hook: gitleaks (secret scan)
□ PR stage → SCA scan (Trivy/OSV-Scanner)
□ Build stage → artifact sign (cosign)
□ Push stage → attach SBOM (syft + attest)
□ Deploy stage → admission control (OPA/Kyverno + image scan)
□ Runtime → continuous vuln monitoring (Dependency-Track)

Pipeline self-security:
□ Pipeline as Code audit (GitHub Actions / GitLab CI config injection)
□ Runner isolation (stop malicious builds escaping the container)
□ Secret management (Actions Secrets / Vault, MUST NOT hardcode)
□ Third-party Action review (pin commit SHA, not tag)
```

### 5. Container-image security

```bash
# Dockerfile audit
hadolint Dockerfile

# Image scan (multi-layer: OS + app deps + config)
trivy image --severity HIGH,CRITICAL nginx:latest

# Minimal base image
# Prefer: distroless → alpine → slim → avoid latest
docker scout quickview nginx:latest

# Image signing
cosign sign --key cosign.key myimage:tag
cosign verify --key cosign.pub myimage:tag
```

### 6. Third-party dependency review

```text
New-dependency Checklist:
□ Maintenance: commits in last 6 months? maintainer activity?
□ Security history: ever shipped malicious code?
□ Dep tree: how many new transitives after adding?
□ License: compatible with the project license?
□ Alternatives: safer options (Snyk Advisor / Socket.dev scores)?

Risk matrix:
  high maintenance × low dep count × compatible license → low risk
  low maintenance × high dep count × license conflict → high risk
```

## Toolchain

| Tool | Purpose | Get |
|------|------|------|
| OWASP Dependency-Track | enterprise continuous SCA | `docker pull dependencytrack/apiserver` |
| OSV-Scanner | free SCA (OSV.dev ecosystem) | `go install github.com/google/osv-scanner` |
| Trivy | image + deps + IaC scan | `apt install trivy` |
| Syft | SBOM generate | `curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh` |
| cdxgen | CycloneDX SBOM generate | `npm install -g @cyclonedx/cdxgen` |
| Cosign | container sign | `go install github.com/sigstore/cosign/v2/cmd/cosign` |
| Gitleaks | secret/credential scan | `go install github.com/gitleaks/gitleaks/v8` |
| Snyk | commercial SCA + reachability | `npm install -g snyk` |
| CodeQL | code query + dataflow | built into GitHub Actions |

## References

- `references/sbom-sca-methodology.md` — SBOM + SCA methodology
- `references/cicd-pipeline-security.md` — CI/CD pipeline security audit


## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES Checklist items?
