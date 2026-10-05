---
name: js-reverse
description: Use when reversing frontend JavaScript with js-reverse-mcp. Covers signature-chain location, page observation for evidence, runtime sampling, local env-patch replay, and evidence-based output. Prefer js-reverse_* tools in the current environment; switch to jshookmcp when a stronger browser/CDP/Hook surface is needed.
---

# MCP frontend JS reverse work spec

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: confirm the current task matches this skill's scope
3. `NEXT`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
4. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
5. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

## Scope

Prefer this skill when the task is:

- Locate API signatures, encrypted parameters, risk-control fields
- Observe page request chains and script sources
- Capture function args and return values at runtime
- Trace the trigger of a given XHR/Fetch/WebSocket
- Bring page evidence back to Node for local replay and env patching

If the target is a binary, APK, PE, ELF, DLL, or SO, use `ida-reverse`, `radare2`, or `reverse-engineering` instead.

## Default tool mapping in this environment

This skill does not assume bare tool names. It binds to `js-reverse_*` tools available in the current client environment.

If the task explicitly mentions `jshookmcp`, `JS hook`, `CDP`, browser breakpoints, network intercept, SourceMap, or AST deobfuscation, still use this skill; only switch the underlying MCP surface to `jshookmcp`. Do not treat that as a new master entry.

Prerequisite: `jshookmcp` is not a local bare CLI tool. It is an MCP server that MUST be downloaded, explicitly registered, and enabled. Related tools are callable only after the chosen client (Claude, Codex, etc.) MCP config has it attached and enabled.

Common mapping:

- `list_scripts` -> `js-reverse_list_scripts`
- `get_script_source` -> `js-reverse_get_script_source`
- `search_in_sources` -> `js-reverse_search_in_sources`
- `break_on_xhr` -> `js-reverse_break_on_xhr`
- `evaluate_script` -> `js-reverse_evaluate_script`
- `get_paused_info` -> `js-reverse_get_paused_info`
- `set_breakpoint_on_text` -> `js-reverse_set_breakpoint_on_text`
- `list_network_requests` -> `js-reverse_list_network_requests`
- `get_request_initiator` -> `js-reverse_get_request_initiator`
- `get_websocket_messages` -> `js-reverse_get_websocket_messages`
- `take_screenshot` -> `js-reverse_take_screenshot`
- `new_page` -> `js-reverse_new_page`
- `navigate_page` -> `js-reverse_navigate_page`
- `select_page` -> `js-reverse_select_page`
- `select_frame` -> `js-reverse_select_frame`
- `pause/resume` -> `js-reverse_pause_or_resume`

If the tool-name prefix changes later, update this section first. Do not guess at execution time.

### jshookmcp role

- Role: enhanced execution surface for `js-reverse`, not an independent master
- Fit: browser automation, CDP debug, JS Hook, network intercept, SourceMap rebuild, AST-assisted understanding
- Call prerequisite: download and register `@jshookmcp/jshook` in the MCP client config, then ensure that server is enabled
- Suggested entry: still run `Observe → Capture → Rebuild`; in `Observe/Capture` prefer jshookmcp browser and Hook capabilities
- vs anything-analyzer: both can do browser/network evidence; anything-analyzer leans packet capture and HTTP analysis; jshookmcp leans JS runtime, CDP, Hook, and source understanding

## Core principles

- `Observe-first`
- `Hook-preferred`
- `Breakpoint-last`
- `Rebuild-oriented`
- `Evidence-first`

Observe the page first, then minimize sampling, then local env patch. Do not skip evidence and guess the environment.

## Five-phase workflow

### 1. Observe

Goal: confirm the target request, related scripts, and candidate functions. Do not guess the environment.

Default actions:

- Open the target page with `js-reverse_new_page` or `js-reverse_navigate_page`
- Find the target request with `js-reverse_list_network_requests`
- Trace the call source with `js-reverse_get_request_initiator`
- Narrow scripts with `js-reverse_list_scripts`, `js-reverse_search_in_sources`

MUST produce:

- Target request URL or fingerprint
- initiator clues
- Suspicious script URLs
- Initial task record

### 2. Capture

Goal: minimally invasive sampling of the target request: parameter samples, call order, runtime evidence.

Rules:

- Prefer `js-reverse_break_on_xhr`
- Prefer `js-reverse_evaluate_script` for light runtime observation
- On hit, read `js-reverse_get_paused_info` first
- Use `js-reverse_set_breakpoint_on_text` only if needed

### 3. Rebuild

Goal: turn page evidence into locally iterable Node replay material.

Rules:

- Local env patches MUST be based on page observation evidence
- Do not invent patches for `window/document/navigator/crypto/storage`
- Record only one minimal causal patch decision at a time

### 4. Patch

Goal: drive env patches from errors and first divergence until the local script stably emits the target parameters.

Rules:

- See what is missing, then patch that
- One minimal patch decision at a time
- Retest immediately after each patch
- Write every patch into the task record

### 5. DeepDive

Goal: after local run succeeds, deobfuscate, restore control flow, and purify business logic.

Rules:

- If the task is only to emit a signature, this phase MAY be downgraded
- If the algorithm chain MUST be reused long-term, this phase MUST be done
- Issue #65 obfuscation bypass (U–AV §4): JSVMP (AD) → `E-js-vmp`; CFF+string array (AE) → `E-js-deobf`; DevTools/debugger anti-debug (AF) → `E-js-anti-debug`. Full trigger table: `../reverse-engineering/references/nonpe-format-cookbook.md`; AST detail still uses `references/ast-deobfuscation.md`

## Execution requirements

- Write all important steps into the local task artifact
- If you cannot explain why a tool is called, do not call it
- Prefer ready MCP capabilities of `js-reverse_*` or jshookmcp for evidence; do not rewrite those capabilities in scripts first
- On failure, fall back per `references/fallbacks.md`
- Output follows `references/output-contract.md`

## Required reading

- Automation entry: `references/automation-entry.md`
- Parameter defaults: `references/tool-defaults.md`
- Task input template: `references/task-input-template.md`
- MCP-specific task orchestration: `references/mcp-task-template.md`
- Task artifacts: `references/task-artifacts.md`
- Local replay: `references/local-rebuild.md`
- Env patching: `references/env-patching.md`
- Node replay: `references/node-env-rebuild.md`
- Instrumentation: `references/instrumentation.md`
- AST deobfuscation: `references/ast-deobfuscation.md`
- Non-PE/JS obfuscation cookbook U–AV: `../reverse-engineering/references/nonpe-format-cookbook.md` (AD/AE/AF)
- Fallback: `references/fallbacks.md`
- Output contract: `references/output-contract.md`

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Upstream fallbacks**:
- anything-analyzer MCP (port 23816) browser tools as alternative or complement
- jshookmcp as a stronger browser/CDP/Hook/Network/SourceMap/AST execution surface
- `reverse-engineering/SKILL.md` (if the target is not frontend JS)

**Downstream exits**:
- Need env patch → `references/env-patching.md`
- Need local replay → `references/local-rebuild.md` / `references/node-env-rebuild.md`
- Need deobfuscation → `references/ast-deobfuscation.md`
- When blocked, fall back → `references/fallbacks.md`

**Peer modules**: anything-analyzer MCP (browser automation and HTTP capture can complement)

---

## On-Demand Bootstrap

MCP capabilities this skill depends on can be installed via the unified bootstrap system; MCP client registration MUST explicitly choose a target. By default no client global config is written.

### Automation bounds

| Capability | Auto-register | Method | Notes |
|------|-----------|------|------|
| jshookmcp | ✓ | npm-mcp (npx start) | Register after explicitly choosing Claude / Codex / Both |
| anything-analyzer | ✓ | local-http-mcp | Service MAY auto-start; client registration MUST be explicit |
| Node.js | ✓ | winget install | Runtime dependency |

### Bootstrap method

```powershell
# Install and register jshookmcp; Codex MAY be replaced with Claude or Both
powershell -File "<skill-root>\scripts\bootstrap-reverse.ps1" -Capability @('jshookmcp') -McpHostTarget Codex

# Register and start anything-analyzer
powershell -File "<skill-root>\scripts\bootstrap-reverse.ps1" -Capability @('anything-analyzer') -StartServices -McpHostTarget Codex
```

### Notes

- After `jshookmcp` is registered, the MCP server still MUST be **enabled** in the AI client before it can be called
- Without `-McpHostTarget`, only install/prepare the capability and return registration-required; do not modify Claude or Codex config
- `anything-analyzer` needs pnpm and project source; bootstrap auto-clones and installs deps
- If Node.js is missing, bootstrap installs Node.js 22 via winget first

<br><br>## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES checklist items?
