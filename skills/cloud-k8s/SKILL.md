---
name: cloud-k8s
description: Use for authorized cloud, container, and Kubernetes security assessment including metadata SSRF, IAM misconfig, container escape paths, and cluster RBAC review.
---

# Cloud / Container / Kubernetes Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-pentest.md` — **cloud/K8s tests MUST have written authorization**
2. `NOW`: case-init + scope; pin account boundary, MUST NOT destructive ops
3. `NOW`: confirm cloud metadata/container/K8s/IAM, not a generic web scan (that is `pentest-tools/`)
4. `NEXT`: tool-index; kubectl/aws/gcloud are usually manual installs
5. `ACT`: start from identity and exposure; MUST NOT default to full-net scan

## When to use

- Cloud metadata SSRF (169.254.169.254 / IMDS)
- IAM over-privilege, public buckets, bad security groups
- Docker/containerd escape-path assessment
- Kubernetes RBAC, Secrets, Admission, supply-chain images
- Container image vulns (MAY chain `supply-chain-security/`)

## Workflow

### Phase 1 — Identity and boundary

```text
□ Current identity: cloud AK/SK, K8s SA, node SSH?
□ Scope: single account / single cluster / single namespace
□ Network profile: authorized_target_only
```

### Phase 2 — Cloud control plane

```bash
# example (swap per vendor; MUST stay in the authorized account)
aws sts get-caller-identity
aws s3 ls
# Azure / GCP matching identity commands
```

```text
□ Public buckets / bad ACLs
□ Metadata: IMDSv1 vs v2; SSRF chain
□ Role assumption (PassRole) and lateral
```

### Phase 3 — Containers

```text
□ privileged / hostPath / hostNetwork?
□ capabilities (SYS_ADMIN etc.)
□ Writable host paths → escape candidates
□ Image history and known CVEs → Trivy
```

### Phase 4 — Kubernetes

```bash
kubectl auth can-i --list
kubectl get pods,secrets,svc -A
kubectl get clusterrolebindings
```

```text
□ SA token mounts and permissions
□ Missing dangerous admission webhooks
□ etcd / dashboard exposure
□ NetworkPolicy default-allow?
```

## Toolchain

| Tool | Use | Bootstrap |
|------|------|------|
| kubectl | Cluster interact | Manual |
| trivy | Image/IaC | bootstrap `trivy` if available |
| kube-bench / kubeaudit | CIS/config | Manual |
| pacu / scoutsuite | Cloud audit (authorized) | Manual |
| nuclei | Known cloud vuln templates | bootstrap nmap/nuclei ecosystem |

## References

- `references/k8s-cloud-checklist.md`
- CTF counterpart: `../../CTF-Sandbox-Orchestrator/competition-agent-cloud/`
- `../supply-chain-security/` `../pentest-tools/`

## Routing context

**Upstream**: MASTER R23
**Downstream**: node shell → `attack-chain` / `windows-ad`; image vulns → supply-chain
**MUST NOT**: unauthorized scan of other public-cloud tenants

## Task-complete self-check

- [ ] Limited to authorized account/cluster?
- [ ] Findings include repro and impact?
- [ ] Destructive ops avoided?
- [ ] Report / journal?
