# .NET reverse common workflow

End-to-end flow, IL-patch reliability, string-decryptor extract, state-machine ID, dnlib scripting.

## End-to-end workflow

```text
1. Identify  → confirm managed .NET (not native)
2. Detect    → DIE / de4dot --detect packer
3. Deobf     → de4dot (keep original sample)
4. Static    → dnSpyEx C# view to locate, IL view for key logic
5. Dynamic   → dnSpyEx debugger break on key methods; read runtime plaintext
6. Patch     → IL editor, Save Module
```

Persist artifacts each step: original `target.exe` → unpacked `target-clean.exe` → patched `target-patched.exe`.

## IL patch vs C# patch reliability

**Rule: key edits use the IL editor, not the C# editor.**

| Dimension | C# editor (Edit Method C#) | IL editor (Edit IL) |
|------|---------------------------|---------------------|
| Compile-fail risk | High (missing refs, syntax, lambda rewrite) | Near zero |
| Fidelity | Compiler regenerates IL; may differ | In-place, instruction-level |
| Fit | String/constant, simple logic | Branch, drop checks, control flow |
| async/await/state machine | Often fail or distort | Edit state-machine fields; reliable |

dnSpyEx C# is read-only decompile plus attempted recompile. Compiler-generated code (state machines, closures, `yield`) often fails to recompile. IL editor is what-you-see.

### Typical IL patch patterns

```text
Force check true (if (check) → always true):
  orig: call bool Foo::Check()
        brfalse.s SKIP
  patch: ldc.i4.1            ; push true
        brfalse.s SKIP      ; never jumps; SKIP unused
  or:
        ldc.i4.1
        ret                 ; method returns true

Force check false:
  ldc.i4.0
  ret

Drop a whole check:
  nop all, or ret with the correct value

String constants:
  C# editor on strings is usually OK (ldstr token swap); encrypted/resource strings need decrypt logic

Numeric constants:
  Change ldarg / ldc operands
```

## State-machine ID (async/await / yield)

C# `async/await` and `IEnumerator` yield compile to a **state machine**: nested class, `MoveNext()` switch on `state`. dnSpyEx C# may restore async, but it can distort; IL `MoveNext` is ground truth.

```text
async/await MoveNext:
  switch(this.<>1__state) {
    case 0: ... pre-await; this.<>1__state = 1; await MoveNext;
    case 1: ... post-await;
  }

To patch async: edit state transitions in MoveNext or the case body.
C# editor on async almost always fails → MUST use IL.
```

## String-decryptor extract

See `obfuscators.md`. dnlib bulk string decrypt:

```csharp
// dnlib: scan string-decryptor calls, restore at runtime, write back
// Usage: dotnet script decrypt.csproj target.exe 0x06000012
using System;
using System.Reflection;
using dnlib.DotNet;
using dnlib.DotNet.Writer;
using dnlib.DotNet.Emit;

var module = ModuleDefMD.Load(args[0]);
var decryptorToken = uint.Parse(args[1], System.Globalization.NumberStyles.HexNumber);

// Find decrypt method; invoke via reflection (load assembly into AppDomain)
// Walk methods; replace call Decryptor(token) with ldstr "plaintext"
foreach (var type in module.GetTypes())
    foreach (var method in type.Methods)
    {
        if (!method.HasBody) continue;
        var instrs = method.Body.Instructions;
        for (int i = 0; i < instrs.Count; i++)
        {
            // Detect call-decryptor; invoke; replace with ldstr
            // (reflection boilerplate omitted: load original assembly →
            //   MethodInfo.Invoke for plaintext → instrs[i] = OpCodes.Ldstr + operand=plaintext)
        }
    }

var opts = new ModuleWriterOptions(module);
module.Write("target-decrypted.exe", opts);
```

dnlib is the de-facto .NET metadata API; de4dot uses it internally. Prefer it for custom deobfuscators.

## Dynamic debug notes

dnSpyEx debugger is far friendlier than native:

- **Break at method entry**: right-click method → Add Breakpoint
- **Object values**: Locals / Watch for fields and strings
- **Memory write**: Edit Value on runtime vars
- **Exception breakpoints**: Debug → Exceptions — obfuscators often drive control flow with exceptions; break on them to see the real path

### Exception-driven control flow

Some obfuscators put real logic in `try` and jump via `throw` + `catch`. Static IL looks like EH; it is control flow:

```text
try { throw new CustomException(0x42); }
catch (CustomException e) {
    switch(e.Code) {
        case 0x42: real logic A; break;
        case 0x43: real logic B; break;
    }
}
```

Break on `CustomException`, follow `Code`. Faster than grinding IL.

## Module initializer (`<module>` .cctor)

The module static ctor (`<module>` `.cctor`) runs first on assembly load. Obfuscators put anti-tamper / decrypt init here. Order:

```text
1. Read <module>.cctor — decrypt / anti-debug init
2. Then Program.Main / Startup
3. If anti-tamper is in .cctor → patch .cctor first, then unpack
```

## Config / C2 / key extract pattern

Red-team tools and loaders often encrypt config in resources or fields and decrypt at runtime:

```text
Locate:
1. strings for cleartext URL/IP (usually gone after obfuscation)
2. byte[] fields + decrypt (AES/XOR)
3. Break on decrypt return; dump plaintext
4. Common: AES-256-CBC with Key==IV (Codegate 2013; see reverse-engineering/tools.md .NET)
```

See `references/sharp-tools.md` for red-team tool config layouts.

## Boundary vs reverse-engineering

- **IL2CPP / NativeAOT** → native, no CLR metadata → `reverse-engineering/` (IDA/r2); this skill only IDs it
- **Managed .NET** (standard C# exe/dll, Mono/Unity managed, Xamarin) → this skill
- **Hybrid (native loader + .NET payload)** → loader in `reverse-engineering/`; after dump, switch here

## Artifact list

Per .NET reverse task:

- `target-original.exe` (untouched)
- `target-clean.exe` (de4dot)
- `notes.md` (obfuscator, decryptor token, key methods, config/C2/key)
- `target-patched.exe` (if patched)
- `il-diff.txt` (pre/post IL if patched)
