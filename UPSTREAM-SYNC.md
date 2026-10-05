# Upstream sync playbook (agent MUST follow)

Use this file every time the user asks to update from the original repo (`upstream` = https://github.com/zhaoxuya520/reverse-skill).

Do **not** merge blindly. Fetch, classify the delta, then ask the user in natural language which buckets to apply.

## 0. Fetch only

```text
git fetch upstream
git log --oneline HEAD..upstream/main
git diff --stat HEAD...upstream/main
```

Stop here. No merge until the user picks buckets.

## 1. Classify each changed file

For every path in `git diff --name-status HEAD...upstream/main`:

| Bucket | Meaning | Default |
|---|---|---|
| **A — real behavior** | Routing, scripts, tests, JSON, new skills, bugfixes, new gates | Offer; recommend yes |
| **B — local overlay** | `TOOLS.md`, `FORK.md`, this file, ToolDiscovery overlay helpers | Keep ours unless upstream rewrote the same function |
| **C — language-only** | Same meaning, Chinese vs our English | Skip by default. Re-translate later if the user wants |
| **D — mixed** | Upstream changed meaning **and** our file is English | Show a short English summary of the **behavior** delta only |

Language-only (bucket C) examples: heading rename 适用范围 → Scope with the same rules; ACTION REQUIRED rephrased but same 5 steps; comment translation. If the only diff after ignoring CJK/English wording is whitespace or equivalent RFC 2119 terms, it is C.

## 2. Present to the user (required)

Sort **most real change → least**. Omit pure language-only files from the spoken list (mention only the count).

Speak naturally in the user's language. Do **not** dump a git patch. Template:

```text
Upstream has N commits. Here is what would actually change (language-only files skipped: K files).

1. <path> — <one line: what behavior changes>
2. <path> — <one line>
…

Local files I will keep unless you say otherwise: TOOLS.md, FORK.md, UPSTREAM-SYNC.md.

Which of these do you want applied? You can say “all real changes”, “only routing/scripts”, “skip 3 and 5”, or name files.
```

Wait for the answer. Then merge / cherry-pick / checkout only the chosen paths.

## 3. After applying

1. If a chosen English `SKILL.md` / ops doc gained new **behavior**, translate only the new sentences. Do not revert the whole file to Chinese.
2. If upstream added a skill, add an English `SKILL.md` in a follow-up (ask first).
3. If `ToolDiscovery.ps1` conflicted, keep `Get-ReverseLocalToolsMarkdownPath` / overlay merge, then re-insert upstream catalog entries.
4. Run:

```text
powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/verify-routing-coherence.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/extract-summaries.ps1
```

5. Refresh `tool-index` only if tool paths or discovery changed.
6. Push `origin` only if the user asked to push.

## 4. Known local overlay (this fork)

Treat as bucket B unless the user explicitly wants upstream's version:

| File | Why it exists |
|---|---|
| `TOOLS.md` | Machine path map (`D:\Tools`). Porting machines = edit this file only |
| `FORK.md` | origin/upstream remotes |
| `UPSTREAM-SYNC.md` | This playbook |
| `skills/scripts/lib/ToolDiscovery.ps1` | Overlay reader prepends `TOOLS.md` JSON |
| `skills/scripts/refresh-tool-index.ps1` | English generated headers |
| `skills/scripts/extract-summaries.ps1` | English INDEX headers |
| English `AGENTS.md`, `skills/SKILL.md`, `skills/MASTER-ROUTING.md`, `skills/ops/*.md`, module `SKILL.md` | LLM-facing English |

`routing.json` / tests / `RULES.md` were not rewritten for language. Prefer upstream there when they change.

## 5. MUST NOT

- Do not `git merge upstream/main` before the user picks buckets.
- Do not restore Chinese text for a file whose only upstream delta is wording.
- Do not overwrite `TOOLS.md` with upstream (upstream does not ship it).
- Do not push to `upstream`.
