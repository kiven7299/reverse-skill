# Cloud / K8s checklist (compact)

## IMDS
- [ ] SSRF can reach 169.254.169.254
- [ ] IMDSv2 enforced
- [ ] IAM role permission surface returned

## K8s high risk
- [ ] Excessive cluster-admin bindings
- [ ] Secrets in plaintext env vars
- [ ] privileged + hostPID/hostPath combo
- [ ] Anonymous auth / insecure apiserver port

## Container
- [ ] Running as root
- [ ] Kernel-module load / docker.sock mount
