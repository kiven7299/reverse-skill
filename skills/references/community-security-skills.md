# Community security Skill ecosystem map (2026-07)

> Source retrieval date: **2026-07-17**
> Purpose: reverse-skill **knows what exists outside**, borrow on demand, **do not** vendor-merge giant external libraries into this pack.
> Pack identity: routing + tool bootstrap + evidence/scope contract + field-journal (see `ops/IDENTITY.md`).

## 1. High-value external repos (learn; do not blind-install)

| Repo | Scale / role | Value to this pack | Risk |
|------|-----------|------------|------|
| [trailofbits/skills](https://github.com/trailofbits/skills) | ToB security-research Claude plugin marketplace | Audit/vuln-analysis/RE plugin quality bar | Install via ToB marketplace; do not default-trust non-curated copies |
| [trailofbits/skills-curated](https://github.com/trailofbits/skills-curated) | Reviewed plugin list | Prefer over arbitrary community skills | Same |
| [Orizon-eu/claude-code-pentest](https://github.com/Orizon-eu/claude-code-pentest) | 6 pentest-lifecycle skills + pure Python scripts | Recon→exploit→report pipeline vs our `attack-chain`+`pentest-tools` | Recheck auth boundary; sandbox scripts |
| [trilwu/secskills](https://github.com/trilwu/secskills) | 16 skills + 6 expert subagents | Multi-role split vs `ops/role-map.md` | Plugin shape, not this monorepo |
| [Masriyan/Claude-Code-CyberSecurity-Skill](https://github.com/Masriyan/Claude-Code-CyberSecurity-Skill) | ~15–19 domain skills (RE/OT/CSOC) | Domain-coverage checklist | Shallower than this pack's per-domain skills |
| [mukul975/Anthropic-Cybersecurity-Skills](https://github.com/mukul975/Anthropic-Cybersecurity-Skills) | **800+** skills · ATT&CK/NIST map | **Framework mapping** and domain catalog; do not depend on the whole library | Huge volume; maintenance and poison surface |
| [Eyadkelleh/awesome-skills-security](https://github.com/Eyadkelleh/awesome-claude-skills-security) | SecLists packed as agent skills | Wordlist/payload entry | Overlaps seclists bootstrap |
| [securityfortech/awesome-security-skills](https://github.com/securityfortech/awesome-security-skills) | Curated security-skill list | Index for discovering new skills | List-only; audit each |
| [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) | 1000+ cross-vendor skill index | Discover official/community skills | Not security-specific |
| [anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review) | PR security-review GitHub Action | Compare to our docs/report "change audit" | CI product, not RE routing |
| [agentskills.io](https://agentskills.io) | Agent Skills open standard | Align frontmatter/dir conventions | Standard has no offense/defense content |

### 1.1 Second-pass retrieval (2026-07-17)

| Repo / resource | Role | Landing in this pack |
|-------------|------|----------|
| [trailofbits/skills](https://github.com/trailofbits/skills) plugins: `audit-context-building` `differential-review` `semgrep-rule-creator` `sharp-edges` `dwarf-expert` `burpsuite-project-parser` | Audit context, diff security review, dangerous APIs, DWARF, Burp project parse | Compare `ida-reverse`/`docs-generator`/audit workflow; **do not** merge the library |
| [HexRaysSA/ida-claude-code-plugins](https://github.com/HexRaysSA/ida-claude-code-plugins) | Official IDA Claude plugins (domain automation, marked unsafe) | Compare `ida-reverse` MCP path; unsafe plugins off by default |
| [P4nda0s/reverse-skills](https://github.com/P4nda0s/reverse-skills) | IDA-NO-MCP: export decompile then analyze; rev-frida/dex-dump/u3d | Complements offline export when MCP is unavailable |
| [2389-research/binary-re](https://github.com/2389-research/binary-re) | triage→static(r2/Ghidra)→dynamic(QEMU/GDB/Frida)→synthesis | `reverse-engineering` phase gates in `re-agent-workflow.md` |
| [incogbyte/android-reverse-engineering-claude-skill](https://github.com/incogbyte/android-reverse-engineering-claude-skill) | APK unpack, endpoint extract, adaptive Frida bypass | Compare `apk-reverse`; dynamic scripts need scope |
| [OwenPawl/cerberus-re-skill](https://github.com/OwenPawl/cerberus-re-skill) | Apple-oriented Ghidra+LLDB+Frida loop | Reference for macOS/iOS dynamic loop |
| [ljagiello/ctf-skills](https://github.com/ljagiello/ctf-skills) | CTF reverse/pwn; install tools on demand | Compare CTF-Sandbox + `pwn-chain` |
| [shuvonsec/claude-bug-bounty](https://github.com/shuvonsec/claude-bug-bounty) | /recon→/hunt→/validate→/report | Compare `recon-pipeline.md` + scope gate |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) | Web payload + Prompt Injection chapter | Prefer `pentest-tools/payloads`; LLM in `llm-security` |
| [HackTricks](https://hacktricks.wiki/) | Pentest methodology + **AI/MCP abuse** | See skill-supply-chain MCP section |
| [appsecsanta AI pentesting agents 2026](https://appsecsanta.com/research/ai-pentesting-agents-2026) | 39+ open-source AI pentest agent architectures | Multi-agent is not required; we use role-map |
| Snyk review "more skills ≠ better" | Skill stacking can lower audit quality | Reinforce "deep skill + routing" |

## 2. Standards and threats (2025–2026)

| Source | Point | Landing |
|------|------|----------|
| [OWASP Agentic Skills Top 10](https://owasp.org/www-project-agentic-skills-top-10/) | Malicious skills, supply chain, privilege abuse, memory poison | `ops/skill-supply-chain.md` |
| [Anthropic Agent Skills engineering](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | Install trusted sources only; review scripts and deps | Same + bootstrap MUST NOT guess paths |
| ClawHavoc-class poison campaigns (AST10) | Bulk malicious skills in registries | Forbid one-click install from unknown registry into this pack |

## 3. This pack vs external "wide" coverage

| Domain | reverse-skill | Why we do not merge whole external libraries |
|------|---------------|--------------------------------|
| APK/JS/IDA/r2/firmware/pwn | **Deep** skill + scripts | Keep depth bound to tool-index |
| Pentest/attack-chain/SRC | pentest-tools + attack-chain + src-hunter | Orizon-class as methodology compare |
| LLM/Agent security | llm-security | AST10 hardens skill self-security |
| Evidence/scope/roles | **ops/** (distinctive) | Most skill packs have no case contract |
| OT/ICS / pure GRC / fraud F3 | No standalone skill | Routing miss → propose new skill or external link; do not force-fit |
| 800+ micro-skills | Do not copy | MASTER routing + domain skills instead of fragments |

## 4. Borrowing rules (MUST)

```text
1. Forbid git submodule of 800+ skills as a runtime dependency
2. When borrowing: extract "phase/checklist/command pattern" into this pack's references or an existing skill
3. External scripts: inspect deps and network in isolation before bootstrap-manifest
4. New scenarios: CONTRIBUTING to add a skill; update routing + RULES keywords
5. Cite source URL + retrieval date (this file's format)
6. Before install/merge, walk ops/skill-supply-chain.md
7. At runtime load only MASTER-ROUTING PRIMARY (+ needed secondary); avoid skill-stack overload
```

## 4.1 Borrowed artifacts already in this pack (not external deps)

| Artifact | Path |
|------|------|
| RE four phases | `reverse-engineering/references/re-agent-workflow.md` |
| Authorized recon | `pentest-tools/references/recon-pipeline.md` |
| Attack-chain gates | `attack-chain/references/lifecycle-checklist.md` |
| Skill supply chain | `ops/skill-supply-chain.md` |
| Domain coverage | `references/domain-coverage-map.md` |

## 5. Suggested priority (later iterations)

| Priority | Action |
|--------|------|
| P0 done | ops contract, MASTER routing, skill supply-chain security docs |
| P1 | Compare Orizon/ToB; add pentest phase checklists to attack-chain references |
| P2 | Optional "external skill allowlist" config; not on the default path |
