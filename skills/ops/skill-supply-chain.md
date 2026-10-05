# Agent skill supply-chain security (this pack)

> Sources: OWASP Agentic Skills Top 10 (AST10), Anthropic Agent Skills security notes, public poisoning incidents (e.g. ClawHavoc; see AST10 timeline)
> Retrieved: 2026-07-17
> Applies when installing / writing / merging **any** skill, MCP, or bootstrap script

Static audit of this pack's **executable script surface** (backdoors / wipe / pipe-to-shell): [`docs/PACKAGE-SECURITY-AUDIT.md`](../../docs/PACKAGE-SECURITY-AUDIT.md).

## 1. Why reverse-skill owns this

This pack will:

- instruct the AI to **run commands and bootstrap downloads**
- touch local and network resources via MCP
- write field-journal / reports

A malicious skill can steal credentials, persist prompts, or plant a supply-chain backdoor.
We use **document gates + a tool source of truth**, not another skill app store.

## 2. Threat map (AST10, shrunk)

| Risk | Symptom | Control here |
|---|---|---|
| Malicious / poisoned skill | induce exfil, write memory/backdoors | Trust this repo + user-written external sources only; humans read SKILL.md and scripts first |
| Over-privilege | indiscriminate `curl \| bash`, whole-disk reads | bootstrap only manifest capabilities; scope `network_profile` |
| Dependency poison | malicious pip/npm | prefer official releases; record versions in tool-index |
| Blind MCP trust | unaudited MCP servers | tool-index registration + port probe; do not default-trust remote MCP |
| MCP/CLI auto-exec poison | repo `.env` rewriting `CODEX_HOME` so startup runs a hostile MCP (HackTricks / CVE-class) | do not trust in-repo default MCP config; check env + MCP list before starting the agent |
| Prompt injection inside a skill | hidden instructions in SKILL body | review diffs; no "HTML-comment execution instructions" without the user |
| Scope drift | skill induces wider scans / "own the whole domain automatically" | ops/scope-contract: out_of_scope + auth; no in_scope → no spray scan |
| Skill-stack overload | too many skills loaded, misses increase (public evals) | load PRIMARY + necessary secondary only (MASTER-ROUTING) |

## 3. MUST checklist for installing an external skill

```text
□ Source: official org / audited list (e.g. ToB curated) / user-owned
□ Read every SKILL.md + scripts/* + package deps
□ No mystery egress, no default steps that read ~/.ssh or browser DBs
□ On routing conflict: this pack's MASTER-ROUTING + scope wins
□ Do not copy into the monorepo unless CONTRIBUTING + anonymization
□ Update skills/references/community-security-skills.md with source date
```

## 4. Boundary vs bootstrap / MCP

| Action | Allowed | Forbidden |
|---|---|---|
| `bootstrap-reverse.ps1 -Capability X` | X ∈ bootstrap-manifest.json | arbitrary new names without editing the manifest |
| Register MCP | user confirm + tool-index refresh | silent global MCP pointing at unknown URLs |
| Run a community one-click pentest Python | authorized lab + source read first | production target + unknown script |

Prefer paths already in `TOOLS.md` over a new download.

## 5. Authors / contributors of this pack

- New skill: CONTRIBUTING + ACTION REQUIRED + completion checklist
- Community quotes: URL + date (this file / community-security-skills.md)
- Suspicious behavior: stop, tell the user, do not auto-"try to bypass"

## 6. Fast self-check (before merging external material)

```powershell
# List script extensions about to be introduced
Get-ChildItem -Recurse -Include *.ps1,*.sh,*.py,*.js | Select-Object FullName
# Coarse danger-pattern search (human review, not complete)
# In the external dir: Select-String -Pattern 'Invoke-WebRequest|curl .\||wget .\||~/.ssh|exfil'
```

## 7. Related

- Identity: `IDENTITY.md`
- External index: `../references/community-security-skills.md`
- Auth: `scope-contract.md` + `field-journal/precedent-auth.md`
