# AI Agent obedience engineering — make the Agent actually work after reading the workflow

> Source: 2026 multi-source synthesis (Anthropic Skill Engineering, Microsoft Code Words, Strands Steering Hooks, Gradient Flow Harness Engineering)
> Use when: coding Agents (Claude Code / Codex / Cursor / Cline / Windsurf / Kiro, etc.) read README/RULES.md then only acknowledge, skip steps, or silently omit required ops

---

## Root-cause diagnosis

"Read the workflow, did not work" is not a capability gap. Natural-language instructions have semantic escape room:

| Cause | Meaning |
|------|------|
| **Context attention decay** | Mid-document content is down-weighted; Agent mostly "sees" head and tail |
| **Semantic override** | Helpfulness optimization reinterprets explicit orders (MUST DO X → "maybe do X") |
| **Passive language treated as optional** | "Ready for next step → invoke X" is read as a suggestion |
| **No state enforcement** | No external state machine checks order; skipped steps go unnoticed |
| **Silent state rot** | Structurally valid, semantically wrong output; errors accumulate quietly |

---

## Technique 1: Critical-First Pattern

**Put "what to do next" first. Context later.**

```
WRONG (Agent ignores):
  [70 lines of project background and tool list]
  → "Next: run bootstrap to install missing tools"

CORRECT (Agent executes):
  "## Execute now: run `bootstrap-reverse.ps1` to check and install missing tools
   → then read routing.md and pick the skill"
  [then project background and tool list]
```

**Why**: LLMs weight prompt head and tail highest. Middle content may be ignored.

**Apply here**:
- RULES.md routing-entry section after trigger keywords, before execution principles
- First section of each SKILL.md is "Execute now", not "When to use"

---

## Technique 2: Directive Over Suggestive

Replace suggestive language with RFC 2119 directives:

| Weak (Agent may skip) | Strong (Agent MUST execute) |
|---|---|
| "You can try..." | **MUST**: you must execute... |
| "Ready for next step → invoke X" | **NOW**: invoke X immediately; do not wait for confirmation |
| "Suggest reading routing.md first" | **REQUIRED**: finish routing.md before any submodule |
| "If tools are missing you can bootstrap" | **NO EXCUSE**: missing tools → bootstrap only; no guessed manual installs |
| "Remember to update field-journal" | **CHECKLIST ENFORCED**: tick Checklist after the task; unfinished = not done |
| "Should..." | **MUST** / **MUST NOT** |

**Key pattern**:
```
MUST — violation = task fail
MUST NOT — violation = security violation
SHOULD — skip only with a reason
MAY — truly optional
```

---

## Technique 3: Excuse Rebuttal Table

**Highest-leverage patch in this pack.** Agents invent "reasonable excuses" under friction. Pre-list them and rebut:

| Common Agent excuse | Rebuttal (forced) |
|---|---|
| "I can skip this and just..." | **No skip.** Every behavior-chain step is required. If you think skip is valid, output the reason first; user decides. |
| "In my judgment this is not required" | **Your judgment does not apply here.** Name the criterion and why it allows skipping a written step. |
| "The user probably does not need this" | **Never decide for the user.** Present all options; mark a recommendation; do not hide alternatives. |
| "I already know how; no need to read X" | **Read X then act.** X may hold task-specific constraints. Reading takes ~2s. |
| "To save time I can skip in parallel..." | **Save time by paralleling independent steps, not by skipping.** Dependent steps stay sequential. |
| "I used this tool before; I know the path" | **No guessed paths.** Resolve from tool-index; install locations differ per machine. |
| "Task is basically done; skip checklist" | **Done = Checklist fully ticked.** Unticked Checklist is not done. |
| "No tool-index; I will guess paths" | **Missing file is 100x safer than a wrong path.** Run refresh-tool-index.ps1 first. |
| "User did not ask for a report, so I skip it" | **Report is default, not optional.** Security tasks MUST emit a report unless the user says "no report". |
| "Too simple for a journal" | **Simple tasks still have landmines.** At least: target type + what was used + surprises; one line is enough. |
| "User asked to redo IAT/one step; I did something more useful" | **Redo = redo the named step** (or a user-confirmed legal prerequisite path). MUST update the matching Evidence; no substitute steps; no silent skip. Unpacking is a **prerequisite** of a readable IAT, not a **substitute** for IAT Evidence. |
| "User said packed sample: skip unpack, just look at IAT; I submitted a junk table as done" | **Feasibility gate:** if X is blocked, MUST state the block, recommend order, **ask the user**. If forced, execute and mark `quality=unreadable/packed`; no capability-denial from a junk table. |
| "Unpacked binary crashes; I keep patching files on disk" | **Patch 6:** log E-self-check-crash / E-iat-repair-fail, go dynamic (bp CreateFile/GetFileSize). No infinite static file edits. |
| "IAT will not repair; I keep trying more packer tools statically" | **IAT repair iron rule:** prefer auto/semi-auto repair; on tool error or unreproducible run → stop static IAT, log E-iat-repair-fail, capture APIs via dynamic breakpoints. No infinite static grind. |
| ".NET / no IAT, hard gate N/A, I skip" | **Equivalent anchors still MUST:** .NET writes dnSpy/IL/metadata summary into the E-imports semantic slot; DLL/SYS MUST also have E-exports. No empty skip. |


**How to use**: put this table near the end of RULES.md or other instruction files (high-attention zone). Agent sees rebuttals before inventing excuses.

---

## Technique 4: Five Skill-engineering patterns (Anthropic 2026)

| Pattern | When | Key trick |
|---|---|---|
| **Linear Flow** | Clear sequences (deploy, install) | Safe defaults + negative orders ("MUST NOT use --force") |
| **Decision Tree** | Platform nav, diagnosis | Tree nav + progressive `references/` load |
| **Iterative Loop** | TDD, review-fix loops | Hard rules first + **excuse rebuttal table** to block shortcuts |
| **Baton Loop** | Multi-session, multi-Agent | Externalize state to `next-prompt.md` (MUST write before exit) |
| **Multi-Phase + Checkpoints** | Multi-day complex work | Orchestrator parent skill + human Go/No-Go; mark time cost |

**This pack**:
- Full behavior chain = Linear Flow (15 sequential steps)
- Routing matrix = Decision Tree (3-dimension match)
- Checklist = Multi-Phase Checkpoint (tick every step)
- Field Journal = Baton Loop (cross-session state)

---

## Technique 5: In-band enforcement (Steering Hooks)

Do not rely on Agent "conscience". Embed self-check in the prompt:

```
Before claiming "task done", MUST self-check:
1. Did I skip any behavior-chain step? Which?
2. Did I guess any tool path? If yes, what is the tool-index path?
3. Is the Checklist fully ticked? Why not?
4. If any answer is "yes"/"not ticked", the task is not done;
   return to that step; do not declare done.
```

Self-audit before "done" is faster than external checks.

---

## Technique 6: Opaque identifiers (Code Words) — API/tool params

Microsoft 2026: semantic param names trigger "helpful optimization".

```
WRONG: { "query": "...", "top": 9 }        → 68.4% param compliance
CORRECT: { "query": "...", "code": "alpha" } → 100% param compliance
```

**Use**:
- Short codes instead of semantic params when bootstrap needs exact config
- Code-word maps for params that MUST be followed

---

## Technique 7: Dual Validation

```
AI A (executor) writes output
  ↓
AI B (reviewer) checks against rules
  ↓ pass
Ship to user
  ↓ fail
Return to A with the specific rule citation
```

**Here**:
- Embed a self-review step in RULES.md: Agent checks Checklist item-by-item before the report
- Unfinished items → return to that step and finish

---

## Technique 8: Context-window layout

LLM attention (high→low):
```
[head 10%] ████████████ ← highest; put "act now"
[middle 80%]  ████░░░░░░░░ ← decay; put references
[tail 10%]  ████████████ ← rebound; put "no skip" + Checklist
```

**Apply**:
1. **Head 10%**: execute-now + trigger keywords
2. **Middle 80%**: detailed workflow, links, tool lists
3. **Tail 10%**: excuse rebuttal + hard Checklist + forbidden list

---

## Field Prompt templates

### Template A: Forced-start (embed at RULES.md head)

```markdown
## CRITICAL: after reading this you MUST execute (do not only acknowledge)

1. **NOW**: detect this file's directory → pack root
2. **NOW**: on first use, write these rules into global config (see global-inject section)
3. **NEXT**: read `skills/SKILL.md` → `skills/routing.md` → pick the sub-skill
4. **NEXT**: read `skills/tool-index.md` for tool state
5. **THEN**: start the actual task; do not stay in "I read it" mode

If you only reply "read", "done", "I understand" without executing the steps,
you failed. The user needs tools installed, code analyzed, vulns verified —
not an acknowledgement.
```

### Template B: Submodule entry (embed at each SKILL.md head)

```markdown
## ACTION REQUIRED (execute immediately after reading; do not wait)

After this file:
1. Confirm you understand when this skill applies
2. Check required tools (`../tool-index.md`)
3. Missing tools → bootstrap
4. Tools present → start workflow step 1
5. If unsure → list concrete questions; do not go silent
```

### Template C: Done self-check (embed at each SKILL.md tail)

```markdown
## Done self-check (MUST confirm item-by-item before claiming done)

□ I executed every behavior-chain step (no skip)
□ I guessed no tool paths (all from tool-index.md)
□ I produced reproducible commands/scripts/reports (not only described steps)
□ I updated field-journal (if there were landmines)
□ I ran the completion Checklist (report + diagrams + lessons)
```

---

## Forbidden (obedience extras)

- Forbidden: after RULES.md, reply only "understood, give me a task"
  → Correct: global inject → SKILL.md → routing.md → pick entry
- Forbidden: claim "steps 1-4 done" when you only read them
  → Correct: separate "read docs" from "executed ops"; the latter has side effects
- Forbidden: say "task done" without Checklist
  → Checklist is the only definition of done
- Forbidden: replace tool-index with "from experience"
  → Paths differ per machine; tool-index is the only locator

---

## If you change only one thing

**Add an "act now" block at the very top of RULES.md**, using bold, CRITICAL, NOW.

Highest ROI. Most "did not work" is: after reading, the Agent enters wait-for-user mode. A forced act-now block breaks that.

Second change: **add the excuse rebuttal table**. First friction → Agent stops; pre-block those excuses.
