# OWASP LLM & Agentic AI Top 10 (2025-2026)

## OWASP Top 10 for LLM Applications v2.0 (2025)

| # | Risk | Core issue | Test direction |
|---|------|---------|---------|
| LLM01 | Prompt Injection | Crafted input steers model behavior | Direct/indirect injection, encoding bypass |
| LLM02 | Sensitive Information Disclosure | PII/API Key/training-data leak | Prompt extraction, output analysis |
| LLM03 | Supply Chain | Poisoned model/lib/dataset | Model provenance, dependency scan |
| LLM04 | Data & Model Poisoning | Train/fine-tune data backdoors | Data lineage, behavior anomaly |
| LLM05 | Improper Output Handling | Output causes XSS/SQLi/RCE | Downstream injection tests |
| LLM06 | Excessive Agency | Over-powered tools/autonomy cause real harm | Permission audit, human-in-loop tests |
| LLM07 | System Prompt Leakage | Extract hidden instructions/secrets/business logic | Cascaded extraction, canary token |
| LLM08 | Vector & Embedding Weaknesses | RAG pipeline attacks, embedding inversion | Retrieval poisoning, semantic-similarity attacks |
| LLM09 | Misinformation | Hallucination is a safety risk in high-stakes use | Factuality checks, confidence calibration |
| LLM10 | Unbounded Consumption | DoS/Denial-of-Wallet | Token-burn tests, rate limits |

## OWASP Top 10 for Agentic Applications (ASI 2026)

| # | Risk | Core harm | Test direction |
|---|------|---------|---------|
| ASI01 | Agent Goal Hijack | Malicious input/tool output hijacks goal | Instruction override, goal tamper |
| ASI02 | Tool Misuse & Exploitation | Unintended use of legitimate tools | Tool-chain stitching, param injection |
| ASI03 | Identity & Privilege Abuse | Agent acts beyond authorization | Credential theft, delegation-chain tests |
| ASI04 | Agentic Supply Chain | MCP descriptors/third-party tools as live risk | Dynamic supply-chain scan |
| ASI05 | Unexpected Code Execution | Prompt→tool→script RCE chain | Multi-layer code-exec tests |
| ASI06 | Memory & Context Poisoning | Long-term memory/embedding poison | Persistent memory attacks |
| ASI07 | Insecure Inter-Agent Communication | Tamper agent-to-agent messages | MITM, replay |
| ASI08 | Cascading Failures | Single fault collapses the system | Fault-propagation tests |
| ASI09 | Human-Agent Trust Exploitation | Manipulate operators into approving danger | Authority-bias / urgency tests |
| ASI10 | Rogue Agents | Agent self-replicates / persistent malice | Persistent-backdoor detection |

## Observed finding mix

Share of issues in real assessments:
- LLM01 Prompt Injection: ~45%
- LLM06 Sensitive Info Disclosure: ~20%
- LLM08 Excessive Agency: ~15%
- Remaining 7: ~20%

## Defense principles

1. Separate planning from execution — the model that interprets intent ≠ the model that acts
2. Bind identity/purpose/scope/TTL — no broad environment privileges
3. Log everything — tool calls/memory/comms as first-class security telemetry
4. Blast-radius control — circuit break/rollback/kill switch over convenience
5. Treat all natural-language input (including retrieved content) as untrusted
6. Output is untrusted too — sanitize before render/exec/query
