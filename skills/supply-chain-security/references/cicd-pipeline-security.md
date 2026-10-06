# CI/CD pipeline security audit

## Pipeline attack surface

```text
Threat model (STRIDE):
□ Spoofing: fake build/signature/provenance
□ Tampering: alter source/artifacts/deps
□ Repudiation: malicious ops with no audit log
□ Info disclosure: pipeline logs/artifacts leak secrets
□ Denial of service: exhaust CI / break builds
□ Elevation of privilege: runner escape / secret theft
```

## Audit checklist

### 1. Pipeline as Code config

```yaml
# GitHub Actions audit points
# ❌ Dangerous
on:
  pull_request_target:  # PR trigger with secrets access
    types: [opened]

# ❌ Script injection
- run: echo "${{ github.event.issue.title }}"  # user input → shell

# ❌ Unbounded token perms
permissions: write-all

# ✅ Safer
on:
  pull_request:  # no secrets
    types: [opened]

# ✅ Pin to SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

# ✅ Least privilege
permissions:
  contents: read
```

### 2. Secret management

```bash
# Scan historical commits for secrets
gitleaks detect --source . --verbose
trufflehog git file://. --only-verified

# Inspect Actions Secrets use
gh secret list
# Confirm: no hardcoded secrets, rotation, least privilege

# Runtime secret injection
# ✅ OIDC instead of long-lived keys
# ✅ Secrets only on steps that need them
```

### 3. Build integrity

```bash
# Build provenance
# Immutable build record (SLSA L2+)
slsa-provenance generate --source . --output provenance.json

# Artifact sign
cosign sign-blob --key cosign.key artifact.tar.gz

# Verify
cosign verify-blob --key cosign.pub --signature artifact.tar.gz.sig artifact.tar.gz
```

### 4. Runner security

```text
□ GitHub-hosted runner? (preferred: fresh env each job)
□ Self-hosted: isolated VM/container?
□ Ever ran fork PRs? (self-hosted risk is high)
□ Runner egress restricted?
□ Build cache leak across builds?
```

### 5. Dependency pull security

```text
□ npm: package-lock.json committed? forbid --force / --legacy-peer-deps
□ pip: requirements.txt version-pinned? forbid pip install <unverified source>
□ Docker: FROM pinned to digest? forbid latest tag
□ Go: go.sum committed?
□ Private packages: registry auth via short-lived token?
```

## Automated check pipeline

```yaml
# .github/workflows/supply-chain.yml
name: Supply Chain Security
on: [push, pull_request]

jobs:
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: SBOM Generate
        run: |
          npm install -g @cyclonedx/cdxgen
          cdxgen -o sbom.json
      
      - name: OSV Scan
        run: |
          go install github.com/google/osv-scanner/cmd/osv-scanner@latest
          osv-scanner scan --sbom sbom.json --format sarif > osv-results.sarif
      
      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          exit-code: 1
      
      - name: Secret Scan
        run: |
          docker run --rm -v $PWD:/src ghcr.io/gitleaks/gitleaks:latest \
            detect --source /src --verbose
      
      - name: Dependency-Track Upload
        run: |
          curl -X POST https://dtrack.example.com/api/v1/bom \
            -H "X-Api-Key: ${{ secrets.DTRACK_API_KEY }}" \
            -F "autoCreate=true" -F "project=myapp" -F "bom=@sbom.json"
```

Source: SLSA Framework, OWASP CI/CD Top 10, GitHub Security Lab
