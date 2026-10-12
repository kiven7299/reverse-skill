---
name: reverse-skill-router
description: Use the reverse-skill repository for authorized reverse engineering, security analysis. Requires the reverse-skill repository to be available as the current workspace or an explicitly supplied local path.
---

# Reverse Skill Codex adapter

This plugin is an optional Codex entry point. The repository remains the canonical, client-neutral implementation.

When the current workspace is the `reverse-skill` repository:

1. Read `RULES.md` at the repository root.
2. Run the platform-native `skills/scripts/master-route` entry with the user's task to select the PRIMARY skill from `skills/config/routing.json`.
3. Before any target action, create and validate `work/<case>/scope.md` with the platform-native `case-init` and `case-guard` scripts.
4. Open the selected `skills/<PRIMARY>/SKILL.md` and follow its task-specific instructions.
5. Resolve tools only through the generated `skills/tool-index.md`; do not register MCP servers or install tools unless the user requested that action.
