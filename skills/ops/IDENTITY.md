# reverse-skill identity (vs Z3r0)

> This file pins **who we are**. Absorb Z3r0 ideas on evidence / scope / roles / timeline. **Do not** become a Z3r0 platform.

## We are

| Axis | reverse-skill |
|---|---|
| Shape | **Skill router pack** — methodology + on-demand toolchain for any AI client (Claude / Cursor / Codex / Kilo…) |
| Entry | `RULES.md` → `MASTER-ROUTING` / `master-route.ps1` → child skill |
| Tool truth | `TOOLS.md` (this fork) + generated `tool-index.md` + `bootstrap-manifest.json` (real paths, no guessing) |
| Evolution | `field-journal/` anonymized write-back |
| Artifacts | Markdown report + `work/<case>/` (gitignored) |
| Deploy | `git clone` is enough; no required PG / UI / Docker pool |

## We are not

| Z3r0 has | reverse-skill **deliberately skips** |
|---|---|
| React ops console | no |
| FastAPI control plane + WebSocket sessions | no |
| PostgreSQL evidence DB | no |
| LightRAG service | no |
| Docker host pool / noVNC control proxy | no (may **document** optional sandbox profiles) |
| Multi-agent process runtime | no (only **role → skill map + handoff protocol**) |

## What we take from Z3r0 (shrunk)

| Idea | reverse-skill form |
|---|---|
| Auth and project boundary | `ops/scope-contract.md` → per-case `scope.md` |
| Evidence→Finding→Path | `ops/evidence-finding-path.md` + report templates |
| Specialist split | `ops/role-map.md` (Lead / cie / cpe / cre… → skill) |
| Replayable log | append-only `work/<case>/timeline.md` |
| WorkItem / coverage | `workitems.md` + coverage checkboxes |
| Tool completeness | `ops/sandbox-profile.md` vs bootstrap-manifest vs `TOOLS.md` |
| Egress control | `network_profile` (`offline` / `lab` / `authorized`) |

## Traits that MUST stay

1. **Three-axis routing + PRIMARY fast path** (target type / intent / toolchain)
2. **On-demand bootstrap** across Windows / Kali / Linux / macOS
3. **MCP-friendly** (IDA / Burp / jshook / anything-analyzer / jadx-mcp)
4. **Anonymized field-journal evolution**
5. **Compliance engineering**: ACTION REQUIRED / completion checklist / no fake stops

## Healthy relationship with Z3r0

```text
Z3r0 = red-team OS / team collaboration platform
reverse-skill = an agent's security-task router + playbook

Optional future: mount this pack into a Z3r0 sandbox-local skills dir
Current: zero Z3r0 install required for a complete workflow
```

## Relationship to "800+ community micro-skills"

- **Do not** submodule a giant skill dump (poison surface + maintenance; see `skill-supply-chain.md`)
- **Do** keep `references/community-security-skills.md` as an index and borrow rules
- **Do** use `domain-coverage-map.md` to show: deep skill + routing > skill-stack spam
- External skill install: AST10 thinking + curated sources only (e.g. Trail of Bits curated)
