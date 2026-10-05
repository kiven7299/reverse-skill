# Fork workflow (kiven7299/reverse-skill)

Keep personal changes while tracking upstream `zhaoxuya520/reverse-skill`.

## Remotes

| Name | URL | Push |
|---|---|---|
| `origin` | https://github.com/kiven7299/reverse-skill.git | yes |
| `upstream` | https://github.com/zhaoxuya520/reverse-skill.git | no |

```text
git remote -v
```

## Daily sync

Agents MUST follow `UPSTREAM-SYNC.md`: fetch, classify real vs language-only deltas, ask the user which buckets to apply. Do not merge `upstream/main` first.

```text
git fetch upstream
git log --oneline HEAD..upstream/main
# then UPSTREAM-SYNC.md — wait for the user
git push origin main
```

Rebase instead of merge if the history must stay linear:

```text
git fetch upstream
git rebase upstream/main
git push origin main
```

## What stays local

| Path | Why |
|---|---|
| `TOOLS.md` | Machine tool map. Edit this when porting machines. |
| `FORK.md` | This workflow. |
| `skills/scripts/lib/ToolDiscovery.ps1` overlay helpers | Reads `TOOLS.md` JSON. Re-apply after a large upstream rewrite of that file. |
| English `SKILL.md` / `AGENTS.md` / `ops/*.md` | LLM-facing text. Re-translate only the upstream delta. |

`skills/tool-index.md` and `skills/tool-index.json` stay gitignored. They are generated per machine.

## Conflict rule

1. Upstream routing/scripts win unless the hunk is the `TOOLS.md` overlay.
2. Keep English ACTION REQUIRED wording if upstream only changed Chinese phrasing with the same meaning.
3. If upstream adds a new skill, translate that `SKILL.md` in a follow-up commit.

## First-time clone of this fork

```text
git clone https://github.com/kiven7299/reverse-skill.git
cd reverse-skill
git remote add upstream https://github.com/zhaoxuya520/reverse-skill.git
# edit TOOLS.md paths for this machine
powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/refresh-tool-index.ps1
```
