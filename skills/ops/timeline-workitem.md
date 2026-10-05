# Timeline + WorkItem / Coverage

> Replayable ops log (Z3r0 timeline idea) + coverage checkboxes (WorkItem idea).
> All of it lives under **`work/<case>/`** (repo gitignore), not in skill-pack prose.

## Directory contract

```text
work/<case>/
  scope.md           # contract (ops/scope-contract.md)
  timeline.md        # append-only; do not rewrite history
  workitems.md       # work items and coverage
  evidence/          # raw artifacts (screenshots, pcap, logs)
  notes/
  report/            # final report draft or copy
```

Initialize:

```powershell
powershell -File skills\scripts\case-init.ps1 -Hint "full pentest" -CaseName "acme-2026"
```

## timeline.md format

Each record is **append-only**:

```markdown
## {ISO-8601} | {role} | {phase}
- action:
- command_or_ref:
- result_summary:
- artifacts: []      # relative paths under this case
- evidence_ids: []   # E-xxx when promoted
- decision_delta: [] # only decisions changed since the previous transition
- carry_forward_refs: [scope.md] # unchanged authoritative state is referenced, not re-serialized
- next:
```

**MUST NOT** delete or rewrite an existing `##` time block (corrections: new entry + `corrects: {timestamp}`).

### Decision delta boundary

`scope.md`, `workitems.md`, and existing Evidence are the current authoritative state. `timeline.md` records the transition; it does not copy a full snapshot.

- Every real stage/turn transition **MUST** write `decision_delta`; list only decisions that actually changed and will change the next action. Write `[]` when nothing changed.
- Unchanged route, auth, scope, network profile, tool capability, existing hypothesis/Evidence **MUST NOT** be expanded again for handoff; put them in `carry_forward_refs`.
- A consumer **MUST** resolve `carry_forward_refs` first, then overlay `decision_delta`. Do not treat the delta as full state.
- A genuine decision boundary exists only when two or more materially different, evidence-supported branches exist and the user's choice changes the next action. Deterministic transitions continue; do not restated context just to manufacture a menu.

Representative transition: `Triage -> Static` with unchanged auth/scope/route records only `decision_delta: [phase=triage->static]` and inherits the rest via `carry_forward_refs: [scope.md, evidence/E-triage.md]`.

## workitems.md template

```markdown
# Work Items

| ID | title | role | targets | surface | status | evidence | notes |
|----|-------|------|---------|---------|--------|----------|-------|
| WI-001 | Port scan edge | cie | {ip} | network | done | E-001 | |
| WI-002 | Auth bypass check | cpe | /api/login | web | blocked | | need creds |

status: pending | in_progress | blocked | done | cancelled

## Coverage
- [ ] Recon complete for in_scope assets
- [ ] Critical/High candidates triaged
- [ ] Validated findings have Evidence
- [ ] Path documented (attack/call/solve)
- [ ] Timeline continuous (no silent gaps >1 major phase)
- [ ] Report exported via docs-generator
- [ ] field-journal written (anonymized)
```

## attack-chain / pentest hooks

| Skill | MUST |
|---|---|
| `attack-chain/` | multi-stage tasks create a case dir; end of each stage updates workitems + timeline |
| `pentest-tools/` | at least 1 timeline entry per tool batch; discoveries → Evidence draft |
| other RE skills | timeline recommended; Evidence chain complete before the report |

## Traits

- Agent-friendly plain text, easy to diff/review
- Cross-referenceable with `TOOLS.md` / tool-index command paths
- No WebSocket live feed; paste timeline into the report when needed
