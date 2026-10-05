---
name: go-rust-reverse
description: Use for reverse engineering stripped Go and Rust binaries including runtime recognition, pclntab/moduel data recovery, panic strings, and idiomatic decompilation recovery.
---

# Go / Rust Binary Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: read `../field-journal/precedent-reverse.md`
2. `NOW`: confirm the sample is a Go/Rust build (`file`/strings/runtime traits)
3. `NEXT`: check GoReSym / related plugins
4. `ACT`: runtime ID → symbol/metadata recovery → business logic

## When to use

- Stripped-symbol Go malware/tools
- Rust release binaries, panic-string-driven analysis
- Language-specific methods that complement generic ida/ghidra

## Workflow

### Go

```text
□ Identify go.buildid, leftover runtime symbols, pclntab
□ GoReSym / redress / IDA Go plugins to recover function names
□ Watch how interface, slice, string structs appear in decompile
□ Network/crypto library paths: crypto/* net/http
```

### Rust

```text
□ panic strings, rust_begin_unwind, crate-path hints
□ Generic instantiation causes code bloat; locate string xrefs first
□ async/tokio state machines need xrefs
```

### Dynamic

```text
□ Frida still works; watch Go stacks and scheduling
□ Prefer log and config strings to drive breakpoints
```

## Toolchain

| Tool | Purpose |
|------|------|
| GoReSym | Go metadata |
| IDA/Ghidra + Go/Rust plugins | Decompile |
| radare2 | Fast strings |
| strings / rabin2 | Triage |

## References

- `references/go-rust-notes.md`
- `../reverse-engineering/go-reverse.md` `../ida-reverse/` `../ghidra-reverse/`
- seed: `field-journal/seed-002_go-malware-stripped.md`

## Routing context

**Upstream**: MASTER R33  
**Downstream**: malware sample flow `malware-analysis`; generic RE `reverse-engineering`

## Task-complete self-check

- [ ] Recovered key function names or an equivalent mapping?
- [ ] Language-runtime evidence annotated?
- [ ] Checklist?
