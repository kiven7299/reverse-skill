# Browser and desktop automation cheat sheet

> Playwright (browser) and OpenReverse (Windows desktop) commands and patterns.
> For pentest, reverse engineering, automated collection.

---

## Playwright / agent-browser commands

### Navigation and lifecycle

```bash
# Open page
agent-browser open "https://target.com/login"

# Wait for load
agent-browser wait --load networkidle

# Close browser (required; else process leak)
agent-browser close
```

### Page snapshot

```bash
# Full a11y tree (debug)
agent-browser snapshot

# Interactive elements only (preferred; returns @e1, @e2...)
agent-browser snapshot -i
```

### Element interaction

```bash
# Click
agent-browser click @e1

# Fill textbox
agent-browser fill @e2 "admin"

# Per-character type (JS-listening inputs)
agent-browser type @e2 "password123"

# Keys
agent-browser press Enter
agent-browser press Tab
agent-browser press Escape

# Scroll
agent-browser scroll down 500
agent-browser scroll up 300
```

### Info

```bash
# Element text
agent-browser get text @e1

# Title
agent-browser get title

# Current URL
agent-browser get url
```

### Wait strategy

```bash
# Wait for element
agent-browser wait @e1

# Fixed wait (ms)
agent-browser wait 2000

# Network idle
agent-browser wait --load networkidle

# Navigation complete
agent-browser wait --load domcontentloaded
```

---

## Pentest patterns

### Automated login

```bash
agent-browser open "https://target.com/login"
agent-browser snapshot -i
agent-browser fill @username "admin"
agent-browser fill @password "password123"
agent-browser click @login_button
agent-browser wait --load networkidle
agent-browser get url                    # confirm backend redirect
```

### XSS payload inject

```bash
agent-browser open "https://target.com/search"
agent-browser snapshot -i
agent-browser fill @search_input "<script>alert(1)</script>"
agent-browser click @search_button
agent-browser wait --load networkidle
agent-browser snapshot                   # check if payload rendered
```

### Bulk form submit (with script)

```powershell
$payloads = @("' OR 1=1--", "<img src=x onerror=alert(1)>", "{{7*7}}")
foreach ($p in $payloads) {
    agent-browser open "https://target.com/form"
    agent-browser snapshot -i
    agent-browser fill @input "$p"
    agent-browser click @submit
    agent-browser wait --load networkidle
    agent-browser snapshot              # inspect response
}
agent-browser close
```

### Cookie / LocalStorage extract

```bash
# Via Playwright API (Node.js script mode)
# agent-browser does not expose cookies; use script mode
```

```javascript
// playwright-extract.js
const { chromium } = require('playwright');
(async () => {
    const browser = await chromium.launch();
    const context = await browser.newContext();
    const page = await context.newPage();
    await page.goto('https://target.com');
    
    // cookies
    const cookies = await context.cookies();
    console.log(JSON.stringify(cookies, null, 2));
    
    // localStorage
    const storage = await page.evaluate(() => JSON.stringify(localStorage));
    console.log(storage);
    
    await browser.close();
})();
```

### Screenshot evidence

```bash
# agent-browser mode
agent-browser open "https://target.com/admin"
agent-browser wait --load networkidle
# Screenshot depends on agent-browser version
```

```javascript
// playwright script mode
await page.screenshot({ path: 'evidence.png', fullPage: true });
```

---

## Playwright Node.js API cheat sheet

### Base template

```javascript
const { chromium } = require('playwright');

(async () => {
    const browser = await chromium.launch({
        headless: true,           // headless
        // proxy: { server: 'http://127.0.0.1:8080' }  // Burp
    });
    const context = await browser.newContext({
        ignoreHTTPSErrors: true,  // ignore cert errors
        userAgent: 'Mozilla/5.0 ...',
    });
    const page = await context.newPage();
    
    await page.goto('https://target.com');
    // ... actions ...
    
    await browser.close();
})();
```

### Common selectors

```javascript
// CSS
await page.click('#login-btn');
await page.fill('input[name="username"]', 'admin');

// Text
await page.click('text=Submit');
await page.click('button:has-text("Login")');

// XPath
await page.click('xpath=//button[@type="submit"]');

// Combined
await page.click('form >> input[type="submit"]');
```

### Network intercept

```javascript
// Intercept request
await page.route('**/api/**', route => {
    console.log('API call:', route.request().url());
    route.continue();
});

// Modify request
await page.route('**/api/auth', route => {
    route.continue({
        headers: { ...route.request().headers(), 'X-Admin': 'true' }
    });
});

// Intercept response
await page.route('**/api/user', async route => {
    const response = await route.fetch();
    const json = await response.json();
    json.role = 'admin';  // tamper response
    route.fulfill({ response, json });
});
```

### Wait and assert

```javascript
// Wait for element
await page.waitForSelector('#result');
await page.waitForSelector('.error', { state: 'visible' });

// Wait for network
const [response] = await Promise.all([
    page.waitForResponse('**/api/login'),
    page.click('#login-btn'),
]);
console.log(response.status(), await response.json());

// Wait for navigation
await Promise.all([
    page.waitForNavigation(),
    page.click('a[href="/admin"]'),
]);
```

---

## OpenReverse desktop automation cheat sheet

### Mode choice

| Mode | Prefix | Fit |
|------|---------|---------|
| UIA | `openreverse uia ...` | Standard Windows controls (button, textbox, list) |
| CUA | `openreverse cua ...` | Complex/custom GUI (IDA disasm view, custom render) |

### UIA (structured controls)

```bash
# Launch app
openreverse uia launch "C:\Tools\x64dbg\x64dbg.exe"

# Window tree
openreverse uia tree

# Click button
openreverse uia click "Button:Open"

# Fill textbox
openreverse uia fill "Edit:FilePath" "C:\sample.exe"

# Menu
openreverse uia menu "File > Open"

# Control text
openreverse uia get-text "Edit:Output"
```

### CUA (vision-driven)

```bash
# Screenshot
openreverse cua screenshot

# Click coords
openreverse cua click 500 300

# Double-click
openreverse cua dblclick 500 300

# Type
openreverse cua type "search string"

# Keys
openreverse cua key "ctrl+g"    # IDA: Go to address
openreverse cua key "F5"        # IDA: Decompile
openreverse cua key "F9"        # x64dbg: Run
```

### Network observe (mitmproxy)

```bash
# Proxy observe
openreverse network start --mode proxy --port 8888

# Local capture
openreverse network start --mode local --filter "target.exe"

# Captured requests
openreverse network list

# Export HAR
openreverse network export har output.har

# Stop
openreverse network stop
```

---

## Reverse-tool automation combos

### IDA Pro (OpenReverse + ida-reverse)

```text
Scene: batch-analyze samples

1. openreverse cua launch "ida64.exe"
2. For each sample:
   a. openreverse cua key "ctrl+o"        # Open file dialog
   b. openreverse uia fill "Edit:FileName" "sample_N.exe"
   c. openreverse uia click "Button:Open"
   d. Wait for analysis (poll IDA title bar)
   e. Extract via ida-reverse MCP
   f. openreverse cua key "ctrl+w"        # Close database
```

### x64dbg automated debug

```text
Scene: auto breakpoints and data capture

1. openreverse uia launch "x64dbg.exe"
2. openreverse cua key "F3"               # Open file
3. openreverse uia fill "Edit:FileName" "target.exe"
4. openreverse uia click "Button:Open"
5. openreverse cua key "ctrl+g"           # Go to address
6. openreverse cua type "0x401000"
7. openreverse cua key "F2"               # Set breakpoint
8. openreverse cua key "F9"               # Run
9. openreverse cua screenshot             # Save state
```

---

## Common issues

| Issue | Cause | Fix |
|------|------|------|
| agent-browser hung | Process leak | `agent-browser close`, then open |
| Element ref stale | Page refreshed | Re-run `snapshot -i` |
| Fill no-op | JS listens to input | Use `type` instead of `fill` |
| HTTPS cert error | Self-signed | Playwright: `ignoreHTTPSErrors: true` |
| Page load timeout | Slow net / heavy assets | Raise timeout or `domcontentloaded` |
| UIA miss | Owner-draw controls | Switch to CUA |
| CUA click offset | Resolution/DPI mismatch | screenshot first, then coords |

---

## Install and deps

### Playwright

```powershell
# Node.js if missing
winget install OpenJS.NodeJS.LTS

# Playwright
npm install -g playwright
npx playwright install          # browser engines

# agent-browser CLI
npm install -g agent-browser
```

### OpenReverse

```powershell
git clone https://github.com/zhexulong/openreverse.git
cd openreverse
npm install
npm run init:agents -- --target=all <project-path>

# Optional: CUA runtime
npm run install:cua-runtime
npm run doctor:cua-runtime

# Optional: network observe
npm run install:mitmproxy
npm run doctor:network
```

---

## Related

| Resource | Notes | Link |
|------|------|------|
| Playwright docs | API | https://playwright.dev/docs/intro |
| OpenReverse | Desktop automation | https://github.com/zhexulong/openreverse |
| mitmproxy | HTTP/HTTPS proxy | https://mitmproxy.org/ |
| Windows UI Automation | UIA docs | https://learn.microsoft.com/en-us/windows/win32/winauto/entry-uiauto-win32 |
