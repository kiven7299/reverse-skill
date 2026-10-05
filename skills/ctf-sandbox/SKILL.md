---
name: ctf-sandbox
description: Thin PRIMARY for CTF / AWD / range multi-type orchestration. Hands off to the sidecar CTF-Sandbox-Orchestrator. Use when the user says CTF, AWD, range, or contest challenge and no more specific pwn/APK/IDA route already won.
---

# CTF sandbox entry (sidecar, not a second router)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: run `../scripts/case-init.ps1`; MUST NOT ACT against real external networks before `auth.status=granted`. Contests/ranges use `-NetworkProfile lab` or `offline`.
2. `NOW`: open package-root `../../CTF-Sandbox-Orchestrator/ctf-sandbox-orchestrator/SKILL.md` and continue under its sandbox assumptions.
3. `MUST NOT` dump 40+ `competition-*` sub-skills into `routing.json`. This entry is one PRIMARY latch only.
4. `ACT`: let the orchestrator pick a downstream `competition-*`. When the challenge type is already clear (pwn/ROP, APK, IDA), an earlier `routing.json` rule SHOULD already have won; do not steal it.

## Why a separate layer

`CTF-Sandbox-Orchestrator/` is a **GPL sidecar pack**; default authorization is inside the sandbox. The core routing pack stays MIT + `scope.md` gate. This skill is a keyword latch only; it does not merge the contest tree into core.

## Task-complete self-check (MUST pass before claiming done)

- [ ] Did I run case-init / scope first, instead of treating "user said CTF" as granted external-network auth?
- [ ] Did I open the sidecar orchestrator, instead of treating 40 sub-skills as PRIMARY?
- [ ] If the task is actually pwn/APK/IDA, did I let the more specific PRIMARY take over?
