# AI-assisted reverse engineering

> LLM-driven decompilation / multi-agent verification / neural semantic recovery
> Largest paradigm shift 2025–2026

## Core tools and models

### LLM4Decompile
- First open-source framework using LLMs for binary→source decompilation
- x86/ARM/MIPS
- Input: assembly → output: C source
- Training: million-scale source-assembly pairs

### Decaf (2026)
- **Compiler-feedback verification**: LLM source → compile → compare to original binary
- Result: decompile rate 26% → 83.9% (ExeBench Real -O2)
- Key insight: feedback loop beats a larger model

### Constraint-Guided Multi-Agent (2026)
- Three-stage verification pipeline:
  1. Syntax correctness (parse)
  2. Compilability (GCC)
  3. Behavioral equivalence (LLM-generated tests)
- 84–97% re-executable; $0.03–0.05 each

### REMEND (2026)
- Specialty: extract math equations from binaries
- 89.8–92.4% accuracy (across 3 ISAs × 3 opt levels × 2 languages)
- Speed: 0.132s/function, only 12M params

### Glaurung
- Open-source Ghidra alternative; Rust core + Python bindings
- **AI-native architecture**: LLM agent embedded in every analysis layer
- Evidence artifacts: plain/rich/JSON/JSONL for LLM consumption
- Supports: ELF/PE/Mach-O, x86/ARM/RISC-V, IOC detection, entropy analysis

## Workflow: AI-augmented binary analysis

### 1. LLM-assisted fast recon

```text
□ strings extract → LLM semantic class (URL/key/path/protocol)
□ import table → LLM infers function (crypto=OpenSSL? net=libcurl?)
□ disasm snippets → LLM IDs patterns (cipher, anti-debug, VM detect)
□ error messages → LLM infers context ("Invalid license" → auth logic location)
```

### 2. Neural decompile

```bash
# LLM4Decompile
python llm4decompile.py --binary target.so --arch arm64 --output target.c

# verify (recompile + compare)
gcc -O2 -o target_recompiled target.c -fPIC -shared
# → verify output behavioral equivalence
```

### 3. Multi-Agent verification

```text
Agent 1 (syntax): check generated C parses
  ↓ fail → feed error back to LLM retry
Agent 2 (compile): GCC → check warnings/errors
  ↓ fail → feed compile errors to LLM
Agent 3 (behavior): LLM generates inputs → run original vs recompiled → compare
  ↓ mismatch → feed diffs to LLM → iterate
```

### 4. LLM-assisted static analysis

```text
□ function rename: feed decompiled pseudocode → LLM suggests semantic names
□ type recovery: context → LLM infers struct/class defs
□ algorithm ID: asm snippets → LLM IDs cipher (AES/TEA/RC4/custom)
□ protocol RE: packet sequence → LLM infers format
□ comments: decompiled code → LLM generates Chinese/English comments
```

### 5. macOS/iOS private-framework RE (MOTIF)

```text
Problem: macOS private frameworks undocumented; types missing
Approach: LLM analyzes usage patterns → infers method signatures and param types
Result: ObjC signature recovery 15% → 86% (vs static analysis)
```

## LLM prompt templates

### Function semantics

```
You are a reverse engineering expert. Analyze this decompiled function:

[pseudocode]

1. What does this function do? (one sentence)
2. Suggest a meaningful function name.
3. What are the input parameters and their likely types?
4. What is the return value?
5. What external APIs/functions does it depend on?
6. Any security-relevant operations (crypto, auth, network, file I/O)?
```

### Algorithm ID

```
Analyze this assembly/disassembly for cryptographic operations:

[assembly]

1. Is this a known cryptographic algorithm? (AES/DES/RC4/TEA/ChaCha20/custom?)
2. Identify the key schedule and round structure.
3. What is the key size?
4. Are there any hardcoded constants that identify the algorithm?
```

### Protocol format inference

```
Given this network packet sequence, infer the protocol structure:

[hex dump]

1. Identify magic bytes and length fields.
2. Propose a struct definition for the packet header.
3. What field(s) appear to be checksums/CRCs?
4. Is this a known protocol or custom?
```

## Tool choice

| Scene | Recommended | Cost |
|------|---------|------|
| Fast decompile | LLM4Decompile | free (local GPU) |
| High-precision decompile | Constraint-Guided Multi-Agent | ~$0.05/binary |
| Math-function extract | REMEND | free |
| Cross-platform RE | Glaurung (Rust) | free open source |
| LLM interaction | Claude API / GPT-4 / DeepSeek | ~$0.01-0.10/call |

## Limits

- **Complex CF**: virtualized/obfuscated code still hard (CFF, VMProtect)
- **Indirect calls**: vtables, fn ptrs hard to recover
- **Inlined functions**: compiler inlining blurs boundaries
- **FP**: vectorized-insn semantic recovery still weak
- **Context window**: large functions (>1000 lines) exceed LLM context

Source: Decaf (2026), REMEND (2026), Constraint-Guided Multi-Agent Decompilation (2026), LLM4Decompile, Glaurung
