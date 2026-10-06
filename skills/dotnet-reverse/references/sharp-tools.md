# Red-team Sharp* analysis & tool install matrix & dnSpy MCP

## Red-team Sharp* analysis

Many red-team tools are C# (Sharp* family). Reverse them to understand detect logic, change signatures, extract embedded config.

### Common Sharp* cheat sheet

| Tool | Function | Reverse focus |
|------|------|-----------|
| **Rubeus** | Kerberos (AS-REP roast / Kerberoast / S4U / pass-the-ticket) | Fixed project layout; `Interop.*` P/Invoke for native calls |
| **SharpHound** | BloodHound collector | LDAP queries, collected attributes |
| **SharpShell / SharpWS** | Remote exec, lateral | WMI / WinRM, command obfuscation |
| **Seatbelt** | Recon | Collection items, decision logic |
| **SharpRoast** | Kerberoasting | Ticket request/parse |
| **Inveigh / SharpSploit** | MitM / exploit framework | Reflection load, API call chain |

### Generic analysis loop

```text
1. Open in dnSpyEx (usually unobfuscated; some teams add ConfuserEx)
2. Read Program.Main or command dispatch (Rubeus uses switch(command))
3. Find the command implementation class/method
4. Read P/Invoke (Interop.* namespace) — native APIs live here
5. Extract embedded resources (some tools embed config/templates)
6. If changing signatures (EDR evade): command strings, API calls, string constants
```

### Rubeus structure example

Rubeus dispatches by command; one class per subcommand. Kerberoasting:

```text
Entry: Rubeus.CommandLineParser → parse args
Dispatch: switch(command) → "kerberoast" → Ask.TGS(...)
P/Invoke: Rubeus.Interop.Lsa* / Native.cs → native Kerberos API
Key: LsaCallAuthenticationPackage (KERB_RETRIEVE_TKT_REQUEST)
```

Signature evade: rename `"kerberoast"`, change `Rubeus` banner strings, reorder P/Invoke.

### Embedded config extract

Many loaders/tools encrypt C2, keys, certs into resources or fields:

```powershell
# dnSpyEx Resources tree
# or CLI
powershell -c "[System.Reflection.Assembly]::LoadFile('target.exe').GetManifestResourceNames()"
# After finding a resource: dnSpyEx right-click → Extract / Save
```

Runtime-decrypted config → break on decrypt return and dump plaintext (see `common-workflow.md`).

---

## Tool install matrix

### Windows (preferred; dnSpyEx is GUI)

```powershell
# A: Chocolatey
choco install dnspy ilspy de4dot detect-it-easy

# B: Manual release download (preferred; pin versions)
# dnSpyEx:    https://github.com/dnSpyEx/dnSpy/releases
# de4dot:     https://github.com/de4dot/de4dot/releases
# ILSpy:      https://github.com/icsharpcode/ILSpy/releases
# DIE:        https://github.com/horsicq/Detect-It-Easy/releases
# dnlib:      dotnet add package dnlib  (NuGet)
```

### Linux / macOS (no dnSpyEx GUI; CLI)

```bash
# ILSpy CLI decompile
dotnet tool install -g ilspycmd
ilspycmd target.exe -p -o outdir/         # decompile to dir

# de4dot cross-platform (mono or dotnet)
# Download de4dot .dll from release; run with dotnet
dotnet de4dot.dll target.exe -o target-clean.exe

# dnlib (scripted; needs dotnet SDK)
dotnet new console -o dnclean && cd dnclean
dotnet add package dnlib

# DIE CLI (diec)
# Linux: install from https://github.com/horsicq/Detect-It-Easy
diec target.exe
```

### .NET runtime prerequisite

```bash
# Linux
sudo apt install dotnet-runtime-8.0        # or 6.0/7.0 per target
# macOS
brew install --cask dotnet-sdk
```

> dnSpyEx (IL editor + debugger) is Windows GUI only. Linux/macOS .NET reverse is `ilspycmd` decompile + `dnlib` script patch; no equivalent interactive debug GUI. Prefer Windows when patching.

---

## dnSpy MCP integration

Several community dnSpy MCP projects expose decompile/IL inspect as MCP tools so an AI can call them — same MCP philosophy as reverse-skill.

### Mainstream dnSpy MCP projects

| Project | Notes | Fit |
|------|------|------|
| **soufianetahiri/dnspy-mcp** | Core MCP server: decompile, IL inspection | Claude Code / Cursor |
| **AgentSmithers/DnSpy-MCPserver-Extension** | Runs as dnSpyEx extension; deep GUI | Load inside dnSpyEx |
| **malwarecakefactory/dnspy-mcp-extension** | 33 tools, triage → deobfuscation | Full-flow automation |

### Register in Claude MCP config

Install the dnSpyEx extension per project README, then register in `~/.claude/mcp.json` (command/args per README):

```json
{
  "mcpServers": {
    "dnspy": {
      "command": "dotnet",
      "args": ["path/to/dnspy-mcp.dll"]
    }
  }
}
```

After register, this skill's AI path: user says "analyze this .NET" → route to `dotnet-reverse/` → prefer `dnspy_decompile` / `dnspy_inspect_il` → fall back to GUI.

> dnSpy MCP is not a reverse-skill built-in bootstrap. User installs the extension and registers it. Consider adding to `bootstrap-manifest.json` later.

---

## Community resource index

### Strongly recommended

- **Washi blog** — .NET reverse: https://blog.washi.dev/posts/misconceptions-about-dotnet/
  - Core: **do not over-rely on dnSpy C# decompiler; know the IL editor** (matches this pack's IL-first rule)
- **dnSpyEx** — active dnSpy fork: https://github.com/dnSpyEx/dnSpy
- **de4dot** — .NET deobfuscate: https://github.com/de4dot/de4dot
- **dnlib** — metadata programming: https://github.com/dnlib/dnlib

### Field tutorials

- Medium *De-obfuscating and reversing a .NET/C# spyware* — dnSpy + de4dot vs info-stealer
- YouTube *dnSpy Patch .NET EXEs & DLLs* — patch + keygen walkthrough
- Kanxue .NET reverse board — search ".net reverse" / "dnSpy" / "ConfuserEx" for field posts, Nuitka reverse, AV evade
- Guided Hacking *Top 5 .NET Reverse Engineering Tools* — dnSpy still #1
- StackExchange / Reverse Engineering — `DynamicMethod` debug and similar

### .NET resources already in this repo

- `reverse-engineering/tools.md` `.NET Analysis` — dnSpy/ILSpy cheat + Codegate 2013 two-stage XOR+AES-CBC
- `reverse-engineering/field-notes.md` `.NET` — tool notes
- `reverse-engineering/awesome-re-resources.md` — de4dot listed
- `field-journal/seed-014_unity-il2cpp-reverse.md` — Unity IL2CPP (native; complements managed .NET)

Deep .NET content lives in this module; keep a cheat index in `reverse-engineering/`.
