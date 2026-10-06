# Go binary reverse-engineering guide

> Go binaries have unique challenges: static linking makes them huge, tens of thousands of functions, special string format, hard symbol recovery after strip.
> This doc covers toolchain, recovery tricks, and field workflow.

---

## ID a Go binary

Fast check whether a binary is Go-compiled:

```bash
# string features
strings binary | grep -E "runtime\.|go\.buildid|GOROOT"

# rabin2 recon
rabin2 -z binary | grep -i "runtime"

# file size abnormally large (statically linked runtime)
# typical Hello World: C ~20KB, Go ~2MB
```

Common features:
- lots of functions with `runtime.` prefix
- `go.buildid` section
- `GOROOT`, `GOPATH` path strings
- 5000–50000+ functions (entire runtime + stdlib)

---

## Core toolchain

### Symbol recovery

| Tool | Use | Link |
|------|------|------|
| **GoReSym** | Mandiant; parse Go symbols (pclntab/moduledata) | https://github.com/mandiant/GoReSym |
| **GoResolver** | Volexity; CFG-similarity auto-deobf of Garble binaries | https://github.com/volexity/GoResolver |
| **redress** | analyze stripped Go binaries; recover types/interfaces/packages | https://github.com/goretk/redress |
| **GoStringUngarbler** | Google; recover Garble-obfuscated strings | https://github.com/mandiant/GoStringUngarbler |

### IDA plugins

| Tool | Use | Link |
|------|------|------|
| **go_parser** | IDA plugin; parse moduledata/pclntab/types | https://github.com/0xjiayu/go_parser |
| **IDAGolangHelper** | IDA scripts; parse Go types | https://github.com/sibears/IDAGolangHelper |
| **AlphaGolang** | SentinelLabs IDAPython scripts | https://github.com/SentineLabs/AlphaGolang |
| **IDA 9.2+ native** | Hex-Rays official Go decompile improvements | https://hex-rays.com/blog/stop-guessing-and-start-going |

### Ghidra plugins

| Tool | Use | Link |
|------|------|------|
| **Ghidra + GoReSym output** | export symbols with GoReSym then import into Ghidra | use together |
| **golang_loader_assist** | Ghidra Go load assist | community script |

### Standalone analysis tools

| Tool | Use | Link |
|------|------|------|
| **gore** | Go RE library (redress backend) | https://github.com/goretk/gore |
| **garble** | Go obfuscator (know it to fight it) | https://github.com/burrowers/garble |

---

## Key Go binary structures

### pclntab (PC Line Table)

Most important structure in a Go binary. Contains:
- all function names and address maps
- source file paths
- line numbers
- stack-frame sizes

Even after strip, pclntab usually remains (Go runtime depends on it).

```text
How to locate:
1. search magic bytes: 0xFFFFFFF0 (Go 1.16+) or 0xFFFFFFFB (Go 1.18+)
2. GoReSym auto-locate
3. go_parser IDA plugin auto-parse
```

### moduledata

Contains:
- pclntab pointer
- type-info table
- itab (interface table)
- global-variable info

### String format

Go strings are not C-style null-terminated; they are `(pointer, length)`:

```text
C string:   "hello\0"
Go string:  struct { ptr *byte; len int } → ptr points at "hello" (no \0)
```

IDA/Ghidra default string ID therefore misses lots of Go strings.

**Fix**:
- `go_parser` auto-ID Go strings
- GoReSym export string list
- manual: find `runtime.stringtable` or locate via xrefs

---

## Field workflow

### Scene 1: unstripped Go binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → export all function names, types, source paths
2. load into IDA/Ghidra
3. import GoReSym symbols
4. filter out runtime.* and stdlib; focus on user code
5. start at main.main
```

### Scene 2: stripped Go binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → even after strip, pclntab is usually still there
2. if GoReSym fails → use redress
   redress -src binary    # recover source paths
   redress -pkg binary    # recover package structure
   redress -type binary   # recover types
3. load into IDA + go_parser plugin
4. run go_parser auto-recovery
5. start from recovered main.main
```

### Scene 3: Garble-obfuscated Go binary

```text
Garble will:
- randomize function names (main.main → main.a3f2b1c)
- encrypt strings
- strip file-path info
- obfuscate package names

Counter:
1. GoResolver (CFG signature match)
   → recover stdlib function names via CFG similarity
2. GoStringUngarbler (string decrypt)
   → auto-ID Garble string-encryption patterns and decrypt
3. dynamic (Frida/dlv)
   → hook runtime functions; observe real behavior
4. compare
   → compile same-version Go Hello World; binary-diff the runtime part
```

### Scene 4: CGo mixed compile

```text
1. ID CGo boundary (_cgo_* functions)
2. recover Go part with go_parser
3. analyze C part with normal IDA
4. watch bridge fns: _cgo_topofstack, crosscall2, etc.
```

---

## Common command cheat sheet

```bash
# GoReSym: export symbols
GoReSym -t -d -p binary > symbols.json
GoReSym -t -d -p binary -o ida_script.py  # generate IDA script

# redress: analyze stripped binary
redress -src binary          # source paths
redress -pkg binary          # package structure
redress -type binary         # types
redress -interface binary    # interfaces
redress -filepath binary     # full file paths

# GoResolver: deobf Garble
GoResolver -binary binary -output resolved.json

# GoStringUngarbler: decrypt Garble strings
GoStringUngarbler -i binary -o deobfuscated_binary

# fast Go version
strings binary | grep "go1\."
GoReSym -p binary | grep "Version"
```

---

## Go analysis flow in IDA

```text
1. load binary (correct architecture)
2. wait for auto-analysis
3. run go_parser plugin:
   - File → Script File → go_parser.py
   - or Edit → Plugins → Go Parser
4. plugin auto:
   - parse pclntab
   - recover function names
   - mark Go strings
   - parse types
5. filter view:
   - hide runtime.* functions
   - focus main.* and third-party packages
6. reverse from main.main
```

---

## Common traps

| Trap | Notes | Fix |
|------|------|------|
| too many functions | Go static link → 5000–50000 fns | filter by package; only main.* and business pkgs |
| incomplete string ID | Go strings are not null-terminated | recover with go_parser or GoReSym |
| unreadable decompile | defer/goroutine/interface make pseudocode messy | IDA 9.2+ improved, or assist with dynamic |
| Garble | function names/strings all randomized | GoResolver + GoStringUngarbler |
| version diffs | pclntab format differs by Go version | GoReSym supports Go 1.2–1.23+ |
| CGo boundary | mixed Go and C | treat _cgo_* functions as the boundary |

---

## Coordination with other skills

| Need | Use |
|------|--------|
| deep IDA analysis of Go binaries | `ida-reverse/` + go_parser plugin |
| Ghidra (free) | Ghidra + GoReSym symbol import |
| fast recon | `radare2/` — `rabin2 -z` for strings |
| dynamic hook | Frida (hook runtime fns) or dlv (native Go debugger) |
| cross-version compare | `binary-diff/` — migrate symbols from old (with symbols) to new |
| Garble deobf | GoResolver + GoStringUngarbler |
