---
name: dsl-vm-reverse
description: Reverse JavaScript-based custom DSL/VM interpreters, non-standard WASM-like runtimes, and risk-control engines. Use when analyzing IIFE or switch-based opcode dispatchers, extracting instruction tables, recovering bytecode semantics, capturing VM state at runtime, or reconstructing execution flow.
---

# DSL custom virtual machine reverse (DSL VM Reverse Engineering)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: confirm this is a custom JS opcode VM / risk engine, not standard WASM or ordinary webpack
2. `NOW`: `case-init` until `scope.md` is ready; offline samples use `offline` / `lab`
3. `ACT`: start file classification at "3. Generic reverse workflow" Phase 1; do not stop at the directory listing

> For reversing JS-implemented custom WASM VMs / risk engines

---

## Contents

- [1. Scope](#1-scope)
- [2. DSL VM identification traits](#2-dsl-vm-identification-traits)
- [3. Generic reverse workflow](#3-generic-reverse-workflow)
- [4. Opcode extract and classify](#4-opcode-extract-and-classify)
- [5. Runtime capture options](#5-runtime-capture-options)
- [6. Common status codes](#6-common-status-codes)
- [7. Skill self-check](#7-skill-self-check)

---

## 1. Scope

Use this skill when the target file matches **any** of these traits:

| # | Trait | Notes |
|---|------|------|
| 1 | IIFE start + many single-letter variable names | `!function(){var U=void 0,y=parseInt,E0=Function,...}` |
| 2 | Contains `DG()` or similar with a switch-case loop | Interpreter main loop, `d[7]&31` decodes opcode |
| 3 | Large file (500KB+) but zero-byte ratio < 1% | Nonstandard WASM, pure JS |
| 4 | Contains `C[number]` constant-table refs | `C[9][xxx]` function table/string table |
| 5 | Single-line compressed code | 583KB one line, obfuscated names |

### Exclusion rules

| Condition | Not this skill | Route to |
|------|-----------|------|
| File starts with `\x00asm` | Standard WASM binary | `reverse-engineering/languages.md` |
| File contains `Uint8Array([0,97,115,109])` WASM magic | Embedded WASM | Extract .wasm then IDA/Ghidra |
| Standard Webpack bundle (`function(e,t,n){...}`) | Ordinary JS | `js-reverse/` |
| Zero-byte ratio > 20% | WASM binary | `reverse-engineering/languages.md` |

---

## 2. DSL VM identification traits

### Code traits

```javascript
// Trait 1: IIFE entry, single-letter vars mapped to numeric constants
!function(){
    var U=void 0, y=parseInt, E0=Function, AN=Uint8Array;
    var E=15, l=10, m=12, x=16, S=13, $=11;
    // Numeric constants mapped to var names, replacing raw numbers
    ...
}

// Trait 2: interpreter main loop DG()
function DG(C, d, ...) {
    var d = [];  // array simulating WASM stack/locals
    for (d[7] = x; d[7] !== U;) {
        var aE = d[7] & 31;         // low 5 bits = opcode
        var O = d[7] >> 5 & 31;      // high 5 bits = sub-operation
        switch (aE) {
            case 0: /* ... */ d[7] = 612; break;
            case 1: /* ... */
            // ... N cases
        }
    }
}

// Trait 3: constant table C[9] stores function indices and strings
// C[9][0] = ["pc"]      → function parameter description
// C[9][667] = "string"  → string constant
// C[9][x] = number      → function index

// Trait 4: W(C[index], null, ...) call pattern
// W = Function.prototype.call.bind(call)
// All builtins are called via C[index]

// Trait 5: instruction encoding
// d[7] = opcode(bit 0-4) | subop(bit 5-9) | operand(bit 10+)
```

### Opcode encoding

Each instruction is a 32-bit integer:

```
bit 0-4:   opcode (0-N)
bit 5-9:   sub-operation (0-31)
bit 10-31: operand/immediate

Decode:
  aE = d[7] & 31        → opcode
  O  = d[7] >> 5 & 31   → sub-operation
  d[other] = d[7] >> 10  → operand
```

---

## 3. Generic reverse workflow

### Phase 1: File classification (5 minutes)

```bash
# Check whether this is a DSL VM
python3 << 'EOF'
with open('target.js', 'rb') as f:
    head = f.read(100)

# 1. Check WASM magic
if head[:4] == b'\x00asm':
    print("standard WASM binary")
    exit()

# 2. Check zero-byte ratio
data = open('target.js', 'rb').read()
zero_pct = data.count(b'\x00') / len(data) * 100
print(f"zero-byte ratio: {zero_pct:.1f}%")

if zero_pct > 20:
    print("WASM binary")
elif head[:2] == b'!f':
    # Check single-letter var pattern
    if b'var U=void 0' in head or b'U=void 0,y=parseInt' in head:
        print("→ DSL VM!")
    else:
        print("ordinary JS IIFE")
EOF
```

### Phase 2: Variable mapping extract (10 minutes)

```python
import re

with open('target.js', 'r', errors='replace') as f:
    s = f.read()

# Extract var X=number mappings from the first 2000 chars
mappings = re.findall(r'var\s+(\w+)\s*=\s*(\d+)', s[:2000])
print('constant mapping:')
for name, val in mappings:
    print(f"  {name:4s} = {val:3d} (0x{int(val):02x})")
```

### Phase 3: Opcode extract and classify (15 minutes)

```python
# 1. Extract all cases
all_cases = re.findall(r'case\s+(\d+):', s)
unique = sorted(set(int(c) for c in all_cases))

print(f"total cases: {len(all_cases)}")
print(f"unique opcodes: {len(unique)}: {unique}")

# 2. Classify each opcode
for op in unique:
    idx = s.find(f'case {op}:')
    snippet = s[idx:idx+200]
    if 'd[7]=' in snippet:
        op_type = 'BRANCH'
    elif 'return' in snippet:
        op_type = 'RETURN'
    elif 'W(C[' in snippet:
        op_type = 'CALL'
    elif 'new' in snippet:
        op_type = 'ALLOC'
    elif 'try' in snippet or 'catch' in snippet:
        op_type = 'EXCEPTION'
    else:
        op_type = 'ARITH/STORE'
    print(f"  opcode {op:2d}: {op_type}")
```

### Phase 4: Constant-table analysis (30 minutes)

```python
const_refs = re.findall(r'C\[9\]\[(\d+)\]', s)
unique_refs = sorted(set(int(x) for x in const_refs))

print(f"C[9] refs: {len(unique_refs)} indices")
print(f"range: {min(unique_refs)} - {max(unique_refs)}")

# Analyze context of each ref
for ref in unique_refs[:20]:
    idx = s.find(f'C[9][{ref}]')
    ctx = s[max(0,idx-50):idx+80]
    clean = ''.join(c if c.isprintable() else ' ' for c in ctx)
    print(f"  C[9][{ref}] → {clean}")
```

### Phase 5: Export-function tracing (1-2 hours)

Locate exported functions (e.g. `getToken`) via:

```
1. Find AWSCInner.register() or similar register calls
2. Determine the registered module and factory function
3. Find the object the factory returns → export function definition site
4. If the function name is not in JS → stored as bytecode in the C[9] constant table
5. Trace the call chain:
   AWSCInner._modules['fy'].getToken()
   → W(C[function_index], null, ...)
   → DG() interpreter runs the encoded instruction sequence
```

### Phase 6: Runtime inject (if pure static is not enough)

```javascript
// Inject a minimal AWSC-compatible environment
const fakeEnv = {
    AWSCInner: {
        _modules: {},
        register(name, moduleName, factory) {
            this._modules[moduleName] = factory();
        }
    }
};

// Execute DSL VM code
dslVmCode();

// Get exports
const token = fakeEnv.AWSCInner._modules['fy'].getToken({});
```

---

## 4. Opcode extract and classify

### References opcode table (from existing cases)

| Opcode | Operation type | Traits |
|--------|---------|------|
| 0 | **BRANCH** | `d[7]=xxx` unconditional jump |
| 1 | **CALL** | `W(C[Y],null,function(){...})` embedded function call |
| 2 | **ARITH** | `d[4]=0`, `d[7]=72` variable assign |
| 3 | **ARITH** | `d[0]=d[1][C[x]]`, `d[5]=d[0]<d[3]` compare |
| 4 | **STORE** | `d[8]=d[5]in d[4]` property access/existence check |
| 5 | **ARITH** | `d[8]=d[4]-d[8]` arithmetic |
| 6 | **RETURN** | `return gV`, `throw` return/throw |
| 7 | **ALLOC** | `d[6]=[]`, `d[6][C[8]](...)` push |
| 8 | **BRANCH** | `d[7]=d[k]?512:425` conditional jump |
| 9 | **STRING** | `d[6][C[t]]=d[m]`, `new fh(...)` regex |
| 10 | **ALLOC** | function-arg prep, call-stack create |
| 11 | **STRING** | `new fh("\\s",d[5])` regex match |
| 12 | **STORE** | `P[d[9]]=d[4][C[H]](d[3])` data pass |
| 13 | **CALL** | `C[9][113]=d[9]` module init |
| 14 | **STRING** | `d[8]=d[9]+d[m]` string concat |
| 15 | **RETURN** | `return EL;` function return |
| 16 | **ALLOC** | `var r,P,Z,B...` local decls |
| 17 | **ALLOC** | `(Z=[])[C[8]](69,T,445)` static array init |
| 18 | **TABLE** | function table/type table init |
| 19 | **EXCEPTION** | `try{for(var RK=x;...` try-catch loop |
| 20 | **DOM** | `Is[d[o]]` DOM ops |
| 21 | **STORE** | safe get of global/object properties |
| 22 | **STRING** | `new fh(r,v)` string/regex handling |
| 23 | **BRANCH** | `try...catch` safe get + conditional jump |
| 24 | **CALL** | `W(C[2],null,8,z,FL)` multi-arg function call |
| 25 | **EXCEPTION** | `try{...}catch(C){...}` exception catch + jump |

---

## 5. Runtime capture options

### Option A: Selenium + CDP native events (recommended, highest success)

```python
from selenium import webdriver

driver = webdriver.Chrome()

# Inject anti-detect
driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {
    "source": r"""
        Object.defineProperty(navigator, 'webdriver', {get: () => false});
        Object.defineProperty(navigator, 'plugins', {get: () => [1,2,3,4,5]});
        Object.defineProperty(navigator, 'languages', {get: () => ['zh-CN','zh','en']});
    """
})

# Send CDP native mouse events
driver.execute_cdp_cmd("Input.dispatchMouseEvent", {
    "type": "mousePressed",
    "x": 549.5, "y": 441.2,
    "button": "left", "buttons": 1,
    "clickCount": 1, "pointerType": "mouse"
})
```

### Option B: Playwright headless browser

```javascript
const { chromium } = require('playwright');

async function run() {
    const browser = await chromium.launch();
    const page = await browser.newPage();

    // Intercept network requests
    await page.route('**/api/**', async route => {
        await route.continue_();
    });

    await page.goto('https://target-page.com');

    // Wait for DSL VM init
    await page.waitForFunction(() => {
        return window.AWSCInner &&
               window.AWSCInner._modules &&
               window.AWSCInner._modules['fy'];
    });

    // Perform actions
    await page.mouse.move(500, 400);
    await page.mouse.down();
    // ... action sequence
    await page.mouse.up();
}
```

### Option C: Pure protocol verify (very low success)

> Tokens generated by a DSL VM are usually tightly bound to browser context (TLS JA3 fingerprint, IP, Cookie, request headers, etc.). After leaving the browser the server can detect a context mismatch. **Do not prefer a pure-protocol approach**.

---

## 6. Common status codes

| Code | Meaning | Handling |
|------|------|------|
| 0 | **Verify passed** | Take sessionId + sig |
| 300 | **Risk-control intercept** | Blocked, cannot pass |
| 8778 | **Verify failed, retry** | Retry the action |
| 8776 | **Too fast, retry** | Add delay then retry |
| 69634 | **Generic failure** | Check whether args are correct |

---

## 7. Skill self-check

- [ ] Did I finish DSL VM identification (IIFE + single-letter vars + DG() interpreter)?
- [ ] Did I extract the variable mapping table (`var X=number`)?
- [ ] Did I extract and classify the opcode list?
- [ ] Did I analyze the C[9] constant-table ref range?
- [ ] Did I locate the export-function registration point?
- [ ] If pure static was not enough, did I try a runtime inject option?
- [ ] After the task, did I write back the field-journal?
- [ ] New tools/scenes found → update routing.md?

---

## Routing registration

| Type | Route |
|------|------|
| **Target type**: WASM / DSL VM / custom ISA | `reverse-engineering/dsl-vm-reverse/SKILL.md` |
| **User intent**: "DSL VM / risk-engine reverse" | this skill |
| **Toolchain**: Playwright / Selenium CDP | browser inject options |

### Path crossings

```
DSL VM reverse path:
  reverse-engineering/dsl-vm-reverse/ → Phase 1-6 workflow
  ↓ if runtime data capture is needed
  browser-automation/ → Playwright/Selenium CDP
  ↓ if API protocol layer analysis is needed
  js-reverse/ → Observe→Capture→Rebuild
```
