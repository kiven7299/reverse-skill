# Code-audit checklist (compact)

- [ ] All external input entry points listed
- [ ] Auth / authz middleware coverage
- [ ] Multi-tenant ID bound to session
- [ ] Deserialization / pickle / YAML load
- [ ] SSRF egress and protocol restrictions
- [ ] Secret and token storage
- [ ] File-upload path and type
- [ ] Dangerous exec/system/Runtime
