---
name: browser-automation
description: |
  Unified automation entry. Covers browser automation (Playwright) and Windows desktop automation (OpenReverse).
  Browser: open pages, click, fill forms, scrape, screenshot, auto-login, pentest page interaction.
  Desktop: drive IDA/x64dbg and other GUI tools, Windows UI Automation, vision-driven interaction, desktop-app packet capture.
  Trigger keywords: browser automation, desktop automation, open page, fill form, scrape, screenshot, auto-login, Playwright, agent-browser, headless, OpenReverse, UIA, CUA, desktop control, Windows automation.
---

# Automation (Desktop & Browser Automation)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm the current task matches this skill's scope
2. `NOW`: read `../tool-index.md` and `../../TOOLS.md`; verify tool availability and real paths
3. `NEXT`: if a tool is missing, call bootstrap; do not guess paths
4. `ACT`: enter step 1 of the workflow and execute; do not stop at confirmation

## Scope

Use this skill when the task is one of:

### Browser (Playwright / agent-browser)
- Open pages and operate elements (click, fill, submit)
- Scrape page content or take screenshots
- Automate login flows
- Interact with web pages in pentests (submit payload, trigger XSS)
- Automate captcha pages
- Batch form submit

### Desktop apps (OpenReverse)
- Drive Windows desktop apps (IDA Pro, x64dbg, Wireshark, etc.)
- Vision-driven interaction (CUA mode)
- Structured UI operations (UIA mode)
- Observe desktop-app network traffic (built-in mitmproxy)
- Automate reverse-tool GUI operations
- Black-box test desktop software

### Split with other tools

| Scenario | Use |
|------|--------|
| Operate a web page (in-browser) | **Playwright / agent-browser** |
| Operate a desktop app (Windows GUI) | **OpenReverse** |
| Packet analysis, HTTP capture | anything-analyzer or OpenReverse network lane |
| JS breakpoints, Hook, CDP debug | jshookmcp |
| Locate signing algorithms, replay with a patched environment | js-reverse |

Simple rule:
- Target is a web page → Playwright
- Target is a Windows desktop app → OpenReverse
- Both needed → combine

---

## Part 1: Browser automation (Playwright / agent-browser)

### Core workflow

```bash
# 1. Open the page
agent-browser open <url>

# 2. Get interactive elements (returns @e1, @e2... refs)
agent-browser snapshot -i

# 3. Operate elements by ref
agent-browser click @e1
agent-browser fill @e2 "text"

# 4. Close when done
agent-browser close
```

### Command reference

```bash
# Navigation
agent-browser open <url>
agent-browser close

# Page snapshot
agent-browser snapshot        # full accessibility tree
agent-browser snapshot -i     # interactive elements only (preferred)

# Interaction
agent-browser click @e1
agent-browser fill @e2 "text"
agent-browser type @e2 "text"
agent-browser press Enter
agent-browser scroll down 500

# Get info
agent-browser get text @e1
agent-browser get title
agent-browser get url

# Wait
agent-browser wait @e1
agent-browser wait 2000
agent-browser wait --load networkidle
```

### Notes
- MUST run `agent-browser close`, else process leak
- Snapshot before operating; do not guess element refs
- After form submit, use `wait --load networkidle` until the page is stable

---

## Part 2: Desktop app automation (OpenReverse)

### Overview

[OpenReverse](https://github.com/zhexulong/openreverse) is a desktop-interaction and evidence-capture framework for AI Agents. It supports:
- **UIA mode**: Windows UI Automation, structured desktop-control ops
- **CUA mode**: vision-driven interaction (Computer Use Agent), for complex GUIs
- **Network observe**: built-in mitmproxy plus local capture

### Interaction-mode choice

| Mode | Fit | Layer |
|------|---------|------|
| UIA | target has standard Windows controls (buttons, text boxes, lists) | Windows UI Automation API |
| CUA | complex UI or non-standard controls (IDA disasm view, custom-rendered UI) | vision + mouse/keyboard |

### Network-observe modes

| Mode | Fit |
|------|---------|
| Proxy Lane | target can be configured to use a proxy (preferred) |
| Local Lane | target cannot use a proxy; need local capture |

### Install and config

```bash
# 1. Clone the project
git clone https://github.com/zhexulong/openreverse.git
cd openreverse

# 2. Install deps
npm install

# 3. Wire Agent hosts (Claude Code / Codex / Zed)
npm run init:agents -- --target=all /path/to/project

# 4. Install CUA runtime (if vision-driven mode is needed)
npm run install:cua-runtime
npm run doctor:cua-runtime

# 5. Install network-observe deps (if capture is needed)
npm run install:mitmproxy
npm run doctor:network
```

### Common combinations

| Need | Config |
|------|------|
| Desktop app only | UIA or CUA, no network lane |
| Desktop app + capture | UIA/CUA + proxy lane |
| Desktop app + local capture | UIA/CUA + local lane |

### Reverse-engineering examples

```text
Scenario: automate IDA Pro for batch analysis

1. Open IDA Pro with OpenReverse CUA mode
2. Auto-load the target binary
3. Wait for analysis to finish
4. Export the function list via UI
5. Observe IDA network behavior with network lane (e.g. Lumina requests)
```

```text
Scenario: automate x64dbg debugging

1. Launch x64dbg with OpenReverse UIA mode
2. Load the target program
3. Set breakpoints
4. Run and observe register/memory changes
5. Screenshot to save evidence
```

---

## On-Demand Bootstrap

### Automation capability boundary

| Tool | Auto-install | How | Notes |
|------|-----------|---------|------|
| Playwright | ✓ | npm + npx playwright install | browser automation engine |
| agent-browser CLI | ✓ | npm install -g agent-browser | browser-ops CLI |
| Node.js | ✓ | winget | prerequisite |
| OpenReverse | ✗ | manual clone + npm install | experimental, heavy deps |
| mitmproxy | ✗ | manual install | OpenReverse network-observe dep |

### Bootstrap trigger

- Browser ops missing Playwright → auto bootstrap
- Desktop ops need OpenReverse → guide the user through a manual install (full steps)

### OpenReverse manual-install guide

If the AI detects desktop-app automation is needed but OpenReverse is not installed:

```markdown
⚠️ **OpenReverse is required for desktop-app automation**

**Install steps**:
1. `git clone https://github.com/zhexulong/openreverse.git`
2. `cd openreverse && npm install`
3. `npm run init:agents -- --target=all <your-project-path>`
4. For vision mode: `npm run install:cua-runtime`
5. For network observe: `npm run install:mitmproxy`

**Verify**: `npm run doctor:cua-runtime` and `npm run doctor:network`
```

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Fit**: any task that needs automated browser or desktop-app control
**Downstream exit**:
- captured requests need analysis → `anything-analyzer` or `js-reverse`
- need JS debug/Hook → `jshookmcp`
- need to recover a signing algorithm → `js-reverse`
- desktop app is an RE tool → `ida-reverse/`

**Peer modules**: `js-reverse` (browser ops may need JS analysis), `ida-reverse` (OpenReverse can automate the IDA GUI)

## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not only read)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back RULES Checklist items?
