# reverse-skill quick start

`reverse-skill` is a set of routing rules, playbooks, and tool indexes for AI clients. It is not a single executable. Analyze only targets you are authorized to test, or legal CTF / lab systems.

## Clone and open

```text
git clone https://github.com/kiven7299/reverse-skill.git
cd reverse-skill
git remote add upstream https://github.com/zhaoxuya520/reverse-skill.git
```

Open the **repository root** as the workspace. Do not copy only `skills/` into a client plugin folder.

Core files:

- `AGENTS.md` / `RULES.md` — behavior and consent gates
- `skills/MASTER-ROUTING.md` — PRIMARY routing
- `skills/*/SKILL.md` — specialist playbooks
- `TOOLS.md` — this machine's tool paths
- `docs/platforms/` — OS install notes

Then:

```text
Windows:       powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/refresh-tool-index.ps1
Linux / macOS: bash skills/scripts/refresh-tool-index.sh
Kali:          bash kali/scripts/refresh-tool-index.sh
```

If the client does not auto-load project rules, cite `RULES.md` and the PRIMARY `SKILL.md` in the chat.

## Client sync failures

If the client shows `Not Synchronizable`, confirm the workspace is the repo root, files are readable, and the client supports that layout. Open the repo locally and cite the needed files. When reporting, include client version, OS, full error, and minimal repro.

## Agent refusals

A client safety policy is not overridden by “I am authorized” in the prompt. Stay on lawful targets. Do not ask for unauthorized intrusion, credential theft, persistence, or destructive ops. Narrow the request to code understanding, sample analysis, patching, CTF, or defensive validation. For a specific APK, site, or account, prepare a verifiable scope first.

## Python tools and uv

Standalone CLIs:

```text
uv tool install PACKAGE_NAME
```

Project deps:

```text
uv venv
uv pip install -r requirements.txt
```

Do not replace every `pip` with `uv pip`. `python -m pip`, `pipx` bootstrap, and existing venvs have different jobs. Do not install security tools into system-global Python.

Archive and download safety: [UV-AND-DOWNLOAD-SECURITY.md](UV-AND-DOWNLOAD-SECURITY.md).

## ZIP / antivirus heuristics

RE tool archives often trip AV heuristics (binaries, debuggers, packed samples). A warning is not a verdict. Do not disable AV. Download from the expected HTTPS repo or release, check digest if published, inspect contents, scan, then run only what you intend.

## Accounts and contributions

Follow the AI client, GitHub, tool vendor, and target ToS. This repo cannot prevent a third-party platform from limiting an account. New skills: `skills/CONTRIBUTING.md`, small verifiable PRs.

Reports need a full error, environment, and repro. One-word reports (“virus”, “test”) need a filename, URL, scanner, version, and steps.

## External models

This repo does not bundle, proxy, or resell Grok/xAI or other APIs and does not collect API keys. Configure models in your client. Vet third-party relays yourself.
