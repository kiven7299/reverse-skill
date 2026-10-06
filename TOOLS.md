# Local Tool Path Reference

**This is the only file to edit when porting this skill pack to another machine or when a tool moves.**

LLM rule: resolve every CLI/GUI path from this file first. Do not guess. Do not search `PATH` until this file has no entry. After editing, run `skills/scripts/refresh-tool-index.ps1` so `skills/tool-index.md` matches.

Git remotes for this fork:

| Remote | URL | Role |
|---|---|---|
| `origin` | https://github.com/kiven7299/reverse-skill.git | Personal fork. Push local English overlay, TOOLS.md, and machine hooks here. |
| `upstream` | https://github.com/zhaoxuya520/reverse-skill.git | Upstream. Pull updates. Never push here. |

Sync:

```text
git fetch upstream
git merge upstream/main
# resolve conflicts, keep local/ and TOOLS.md
git push origin main
```

---

## Machine

| Key | Value |
|---|---|
| Host OS | Windows 10/11 |
| Tools root | `D:\Tools` |
| Java | `D:\c_drive_extends\java_21\bin\java.exe` (21.0.2 LTS) |
| Python | `D:\c_drive_extends\python3.11\python.exe` (3.11.4) |
| Node | `D:\c_drive_extends\NodeJS\node.exe` (v22.14.0) |
| Git | `D:\c_drive_extends\Git\cmd\git.exe` |

On another machine: change **Tools root** and every path in the JSON block below. Keep tool `name` keys unchanged.

---

## Overlay (machine-readable)

Discovery reads the first `json` fence in this file. Prepend these fallbacks ahead of upstream catalog paths.

```json
{
  "toolsRoot": "D:\\Tools",
  "runtimes": {
    "java": "D:\\c_drive_extends\\java_21\\bin\\java.exe",
    "python": "D:\\c_drive_extends\\python3.11\\python.exe",
    "pip": "D:\\c_drive_extends\\python3.11\\Scripts\\pip.exe",
    "node": "D:\\c_drive_extends\\NodeJS\\node.exe",
    "npx": "D:\\c_drive_extends\\NodeJS\\npx.cmd"
  },
  "tools": [
    {
      "name": "jadx",
      "skill": "apk-reverse",
      "purpose": "Java/Android decompiler (CLI)",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\jadx-1.5.6\\bin\\jadx.bat" }
      ]
    },
    {
      "name": "jadx-gui",
      "skill": "apk-reverse",
      "purpose": "Java/Android decompiler (GUI)",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\jadx-gui-1.5.3-win\\jadx-gui-1.5.3.exe" },
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\jadx-1.5.6\\bin\\jadx-gui.bat" }
      ]
    },
    {
      "name": "apktool",
      "skill": "apk-reverse",
      "purpose": "APK decode/rebuild",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\APKTool\\apktool.bat" },
        { "type": "java-jar", "value": "D:\\Tools\\Android_platform-tools\\APKTool\\apktool.jar" }
      ]
    },
    {
      "name": "adb",
      "skill": "apk-reverse",
      "purpose": "Device / logcat",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\platform-tools\\adb.exe" },
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\Sdk\\platform-tools\\adb.exe" }
      ]
    },
    {
      "name": "apksigner",
      "skill": "apk-reverse",
      "purpose": "APK signing",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\Sdk\\build-tools\\36.1.0\\apksigner.bat" }
      ]
    },
    {
      "name": "zipalign",
      "skill": "apk-reverse",
      "purpose": "APK alignment",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\Sdk\\build-tools\\36.1.0\\zipalign.exe" },
        { "type": "path", "value": "D:\\Tools\\Android_platform-tools\\APK.Tool.GUI.v3.3.2.0\\Resources\\zipalign.exe" }
      ]
    },
    {
      "name": "java",
      "skill": "apk-reverse",
      "purpose": "JAR / Java toolchain",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\java_21\\bin\\java.exe" }
      ]
    },
    {
      "name": "frida",
      "skill": "apk-reverse",
      "purpose": "Dynamic instrumentation",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\python3.11\\Scripts\\frida.exe" }
      ]
    },
    {
      "name": "frida-ps",
      "skill": "apk-reverse",
      "purpose": "Frida process list",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\python3.11\\Scripts\\frida-ps.exe" }
      ]
    },
    {
      "name": "ida",
      "skill": "ida-reverse",
      "purpose": "IDA Pro 9.1",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\IDA_pro_91\\ida.exe" },
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\IDA Pro 7.6\\ida.exe" }
      ]
    },
    {
      "name": "ida-pro-mcp",
      "skill": "ida-reverse",
      "purpose": "IDA Pro MCP 2.0 plugin (user plugins dir)",
      "fallbacks": [
        { "type": "directory", "value": "C:\\Users\\nguye\\AppData\\Roaming\\Hex-Rays\\IDA Pro\\plugins\\ida_mcp" }
      ]
    },
    {
      "name": "analyzeHeadless",
      "skill": "ghidra-reverse",
      "purpose": "Ghidra headless",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\ghidra_10.4_PUBLIC_20230928\\ghidra_10.4_PUBLIC\\support\\analyzeHeadless.bat" }
      ]
    },
    {
      "name": "r2",
      "skill": "radare2",
      "purpose": "radare2 analyzer",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\r2.exe" },
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\radare2.exe" }
      ]
    },
    {
      "name": "rabin2",
      "skill": "radare2",
      "purpose": "Binary recon",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\rabin2.exe" }
      ]
    },
    {
      "name": "rasm2",
      "skill": "radare2",
      "purpose": "Assemble / disassemble",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\rasm2.exe" }
      ]
    },
    {
      "name": "radiff2",
      "skill": "radare2",
      "purpose": "Binary diff",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\radiff2.exe" }
      ]
    },
    {
      "name": "rahash2",
      "skill": "radare2",
      "purpose": "Hash / checksum",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\rahash2.exe" }
      ]
    },
    {
      "name": "rax2",
      "skill": "radare2",
      "purpose": "Base conversion",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\rax2.exe" }
      ]
    },
    {
      "name": "r2pm",
      "skill": "radare2",
      "purpose": "radare2 package manager",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\R.E. Tools\\radare2\\bin\\r2pm.exe" }
      ]
    },
    {
      "name": "nmap",
      "skill": "pentest-tools",
      "purpose": "Port scan / service detect",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\Nmap\\nmap.exe" }
      ]
    },
    {
      "name": "python",
      "skill": "reverse-engineering",
      "purpose": "Helper scripts",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\python3.11\\python.exe" }
      ]
    },
    {
      "name": "pip",
      "skill": "reverse-engineering",
      "purpose": "Python packages",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\python3.11\\Scripts\\pip.exe" }
      ]
    },
    {
      "name": "node",
      "skill": "js-reverse",
      "purpose": "JS replay / MCP runtime",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\NodeJS\\node.exe" }
      ]
    },
    {
      "name": "npx",
      "skill": "js-reverse",
      "purpose": "npm MCP launcher",
      "fallbacks": [
        { "type": "path", "value": "D:\\c_drive_extends\\NodeJS\\npx.cmd" }
      ]
    },
    {
      "name": "reqable-mcp",
      "skill": "pentest-tools",
      "purpose": "Reqable capture MCP",
      "fallbacks": [
        { "type": "path", "value": "D:\\Tools\\reqable\\mcp-server.exe" }
      ]
    }
  ]
}
```

---

## Catalog (human)

Prefer the JSON `name` when a skill asks for a catalog tool. Extra local tools are listed here for LLM routing even if they are not in upstream `Get-ReverseToolCatalog`.

### Reverse engineering

| Name | Path | Notes |
|---|---|---|
| jadx CLI | `D:\Tools\R.E. Tools\jadx-1.5.6\bin\jadx.bat` | Installed for this pack (v1.5.6) |
| jadx GUI | `D:\Tools\R.E. Tools\jadx-gui-1.5.3-win\jadx-gui-1.5.3.exe` | Existing GUI |
| IDA Pro 9.1 | `D:\Tools\R.E. Tools\IDA_pro_91\ida.exe` | Prefer this over 7.6 |
| IDA Pro 7.6 | `D:\Tools\R.E. Tools\IDA Pro 7.6\ida.exe` | Fallback |
| IDA MCP 2.0 plugin | `C:\Users\nguye\AppData\Roaming\Hex-Rays\IDA Pro\plugins\` | `ida_mcp.py` + `ida_mcp\`. IDA loads this user dir. |
| Ghidra | `D:\Tools\R.E. Tools\ghidra_10.4_PUBLIC_20230928\ghidra_10.4_PUBLIC\ghidraRun.bat` | 10.4 PUBLIC |
| Ghidra headless | `D:\Tools\R.E. Tools\ghidra_10.4_PUBLIC_20230928\ghidra_10.4_PUBLIC\support\analyzeHeadless.bat` | |
| radare2 | `D:\Tools\R.E. Tools\radare2\bin\r2.exe` | Full r2 suite in `bin\` |
| iaito | `D:\Tools\R.E. Tools\iaito-6.0.4-w64\iaito\iaito.exe` | r2 GUI |
| dnSpy | `D:\Tools\R.E. Tools\dnSpy-6.5.1\dnSpy-net-win64\dnSpy.exe` | .NET |
| ILSpy | `D:\Tools\R.E. Tools\ILSpy_9.0.0.7833-preview3-x64\ILSpy.exe` | .NET |
| JD-GUI | `D:\Tools\R.E. Tools\JD-GUI-1.6.6\jd-gui-1.6.6.jar` | Java GUI |
| GDA | `D:\Tools\R.E. Tools\GDA 4.02\GDA4.02.exe` | Android |
| CFR | `D:\Tools\R.E. Tools\cfr-0.152.jar` | Java decompiler |
| jadx-mcp-server | `D:\Tools\jadx-mcp-server\jadx_mcp_server.py` | Existing MCP |

### Android / mobile

| Name | Path | Notes |
|---|---|---|
| apktool | `D:\Tools\Android_platform-tools\APKTool\apktool.bat` | |
| adb | `D:\Tools\Android_platform-tools\platform-tools\adb.exe` | |
| zipalign | `D:\Tools\Android_platform-tools\Sdk\build-tools\36.1.0\zipalign.exe` | |
| apksigner | `D:\Tools\Android_platform-tools\Sdk\build-tools\36.1.0\apksigner.bat` | |
| aapt / aapt2 | `D:\Tools\Android_platform-tools\Sdk\build-tools\36.1.0\aapt.exe` | |
| scrcpy | `D:\Tools\Android_platform-tools\scrcpy\scrcpy.exe` | |
| APK Tool GUI | `D:\Tools\Android_platform-tools\APK.Tool.GUI.v3.3.2.0\APKToolGUI.exe` | |
| SignApk | `D:\Tools\Android_platform-tools\SignApk\signapk.jar` | |
| jnitrace | `D:\Tools\jnitrace` | Python tool tree |

### Web / pentest

| Name | Path | Notes |
|---|---|---|
| nmap | `D:\Tools\Nmap\nmap.exe` | |
| nuclei | `D:\Tools\nuclei_2.7.3\nuclei.exe` | 2.7.3 |
| amass | `D:\Tools\Amass\amass.exe` | |
| dirsearch | `D:\Tools\dirsearch\dirsearch.py` | |
| jwt_tool | `D:\Tools\jwt_tool\jwt_tool.py` | |
| BurpSuite Pro | `D:\Tools\BurpSuite\BurpSuite_Desktop\BurpSuitePro\BurpSuitePro.exe` | Licensed install already on disk |
| Burp MCP | `D:\Tools\BurpSuite\BurpSuite-MCP-Server\main.py` | |
| Reqable | `D:\Tools\reqable\Reqable.exe` | |
| Reqable MCP | `D:\Tools\reqable\mcp-server.exe` | |
| ngrok | `D:\Tools\ngrok\ngrok.exe` | |
| Proxifier | `D:\Tools\Proxifier` | |
| IIS ShortName Scanner | `D:\Tools\IIS-ShortName-Scanner` | |
| php_filter_chains_oracle_exploit | `D:\Tools\php_filter_chains_oracle_exploit` | |

### Data / general

| Name | Path | Notes |
|---|---|---|
| DBeaver | `D:\Tools\dbeaver\dbeaver.exe` | |
| DB Browser for SQLite | `D:\Tools\DB.Browser.for.SQLite-v3.13.1-win64\DB Browser for SQLite.exe` | |
| WinMerge | `D:\Tools\WinMerge\WinMergeU.exe` | |
| Maven | `D:\Tools\Apache_Maven\apache-maven-3.8.5\bin\mvn.cmd` | |
| HxD installer | `D:\Tools\HxDPortableSetup\HxDPortableSetup.exe` | Installer only; not a portable exe |
| MobaXterm | `D:\Tools\MobaXterm_Portable_v24.2` | |

### Runtimes (not under D:\Tools)

| Name | Path |
|---|---|
| java | `D:\c_drive_extends\java_21\bin\java.exe` |
| python | `D:\c_drive_extends\python3.11\python.exe` |
| frida / frida-ps | `D:\c_drive_extends\python3.11\Scripts\` |
| node / npx | `D:\c_drive_extends\NodeJS\` |

---

## Missing / do not auto-install

These are absent or not a drop-in CLI. Do **not** download cracked IDA/Burp. Commercial tools stay as already present on disk.

| Tool | Status |
|---|---|
| yara | Not installed. Add a licensed/official Windows build under `D:\Tools\yara\yara.exe` and extend the JSON overlay. |
| binwalk | Not installed. Prefer WSL/Kali or a later official Windows build. |
| Binary Ninja | Not on this machine. |
| JEB Pro | Not on this machine. |
| SecLists | Not under `D:\Tools`. |

---

## Agent contract

1. Read this file before any tool invocation.
2. Use the JSON `name` + `fallbacks[0].value` for catalog tools.
3. Use the Catalog tables for extra local tools (dnSpy, nuclei, Burp, …).
4. After a path change: edit this file only, then refresh `tool-index`.
5. New installs go under `D:\Tools` (or the new tools root), then add a row here.
