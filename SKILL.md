---
name: chrome-cdp
description: Interact with local Chrome browser session (only on explicit user approval after being asked to inspect, debug, or interact with a page open in Chrome)
---

# Chrome CDP

Lightweight Chrome DevTools Protocol CLI. Connects directly via WebSocket — no Puppeteer, works with 100+ tabs, instant connection.

## Prerequisites

- Chrome (or Chromium, Brave, Edge, Vivaldi) with remote debugging enabled: open `chrome://inspect/#remote-debugging` and toggle the switch
- Node.js 22+ (uses built-in WebSocket)
- If your browser's `DevToolsActivePort` is in a non-standard location, set `CDP_PORT_FILE` to its full path

## Commands

All commands use `scripts/cdp.mjs`. The `<target>` is a **unique** targetId prefix from `list`; copy the full prefix shown in the `list` output (for example `6BE827FA`). The CLI rejects ambiguous prefixes.

### List open pages

```bash
scripts/cdp.mjs list
```

### Take a screenshot

```bash
scripts/cdp.mjs shot <target> [file]    # default: screenshot-<target>.png in runtime dir
```

Captures the **viewport only**. Scroll first with `eval` if you need content below the fold. Output includes the page's DPR and coordinate conversion hint (see **Coordinates** below).

### Accessibility tree snapshot

```bash
scripts/cdp.mjs snap <target>
```

### Evaluate JavaScript

```bash
scripts/cdp.mjs eval <target> <expr>
```

> **Watch out:** avoid index-based selection (`querySelectorAll(...)[i]`) across multiple `eval` calls when the DOM can change between them (e.g. after clicking Ignore, card indices shift). Collect all data in one `eval` or use stable selectors.

### Network capture & replay

Network & WebSocket traffic is automatically captured when a tab daemon starts.

```bash
scripts/cdp.mjs netlog    <target> [filter]       # list captured requests (filter by URL keyword)
scripts/cdp.mjs netdetail <target> <reqId>         # full request/response headers + body
scripts/cdp.mjs netclear  <target>                 # clear captured request log
scripts/cdp.mjs cookie    <target> [name]          # list all cookies or get a specific one
scripts/cdp.mjs replay    <target> <reqId> [json]  # replay request with optional overrides
scripts/cdp.mjs export    <target> [file]          # export captured requests as HAR file
scripts/cdp.mjs diff      <target> <id1> <id2>     # compare two captured requests
scripts/cdp.mjs scope     <target> <pattern>       # add URL scope filter (* = wildcard)
scripts/cdp.mjs scope     <target> list|off|remove <id>  # manage scope filters
```

`<reqId>` is a request ID prefix from `netlog`. Replay uses `fetch()` in page context with `credentials: include`, inheriting cookies automatically. Export produces HAR 1.2 format (importable by Chrome DevTools, Burp Suite, etc.). Scope filters limit `netlog` and `export` to matching URLs only.

### WebSocket monitoring

```bash
scripts/cdp.mjs wslog     <target> [filter]       # list WebSocket connections
scripts/cdp.mjs wsdetail  <target> <wsId>          # show frames for a connection
scripts/cdp.mjs wsclear   <target>                 # clear WebSocket log
```

### Interaction

```bash
scripts/cdp.mjs click   <target> <selector>        # click element by CSS selector
scripts/cdp.mjs clickxy <target> <x> <y>            # click at CSS pixel coords
scripts/cdp.mjs type    <target> <text>              # Input.insertText at current focus
scripts/cdp.mjs hover   <target> <selector>          # hover over element (triggers menus, tooltips)
scripts/cdp.mjs key     <target> <keyName>           # keyboard event: Enter, Escape, Tab, ArrowDown, etc.
scripts/cdp.mjs select  <target> <selector> <value>  # select dropdown option by value
scripts/cdp.mjs wait    <target> <selector> [ms]     # wait for element to appear (default 10s)
scripts/cdp.mjs scroll  <target> [direction] [px]    # scroll: up/down/left/right (default: down 500px)
scripts/cdp.mjs loadall <target> <selector> [ms]     # click "load more" until gone
```

### Page & storage

```bash
scripts/cdp.mjs html           <target> [selector]     # full page or element HTML
scripts/cdp.mjs nav            <target> <url>           # navigate and wait for load
scripts/cdp.mjs net            <target>                 # resource timing entries (basic)
scripts/cdp.mjs storage        <target> [key] [value]   # read/write localStorage
scripts/cdp.mjs sessionstorage <target> [key] [value]   # read/write sessionStorage
scripts/cdp.mjs pdf            <target> [file]           # export page as PDF
scripts/cdp.mjs dialog         <target> <action>         # handle JS dialogs: accept|dismiss|auto-accept|auto-dismiss|off
```

### Request interception

Intercept and modify in-flight requests/responses using the CDP Fetch domain:

```bash
scripts/cdp.mjs intercept <target> <urlPattern> [rulesJson]  # add intercept rule
scripts/cdp.mjs intercept <target> list                       # list active rules
scripts/cdp.mjs intercept <target> remove <id>                # remove a rule
scripts/cdp.mjs intercept <target> off                        # clear all rules
```

`<urlPattern>` uses `*` as wildcard. Rules JSON fields:
- **Request mods**: `requestHeaders`, `requestUrl`, `requestMethod`, `requestBody`
- **Response mods**: `responseStatus`, `responseHeaders`, `responseBody`

Examples:
```bash
# Mock an API response
scripts/cdp.mjs intercept <target> "*api/user*" '{"responseBody":"{\"name\":\"test\"}","responseStatus":200}'

# Inject auth header into all API requests
scripts/cdp.mjs intercept <target> "*api*" '{"requestHeaders":{"Authorization":"Bearer xxx"}}'

# Unlock CORS
scripts/cdp.mjs intercept <target> "*api*" '{"responseHeaders":{"Access-Control-Allow-Origin":"*"}}'
```

### Encode/decode (no target needed)

```bash
scripts/cdp.mjs encode base64 "hello world"       # → aGVsbG8gd29ybGQ=
scripts/cdp.mjs decode base64 aGVsbG8gd29ybGQ=    # → hello world
scripts/cdp.mjs encode url "a=1&b=2"              # → a%3D1%26b%3D2
scripts/cdp.mjs encode hex "hello"                 # → 68656c6c6f
```

Supported encodings: `base64`, `url`, `html`, `hex`.

### Advanced

```bash
scripts/cdp.mjs evalraw <target> <method> [json]  # raw CDP command passthrough
scripts/cdp.mjs open    [url]                      # open new tab (each triggers Allow prompt)
scripts/cdp.mjs stop    [target]                   # stop daemon(s)
```

## Coordinates

`shot` saves an image at native resolution: image pixels = CSS pixels × DPR. CDP Input events (`clickxy` etc.) take **CSS pixels**.

```
CSS px = screenshot image px / DPR
```

`shot` prints the DPR for the current page. Typical Retina (DPR=2): divide screenshot coords by 2.

## Tips

- Prefer `snap --compact` over `html` for page structure.
- Use `type` (not eval) to enter text in cross-origin iframes — `click`/`clickxy` to focus first, then `type`.
- Chrome shows an "Allow debugging" modal once per tab on first access. A background daemon keeps the session alive so subsequent commands need no further approval. Daemons auto-exit after 20 minutes of inactivity.
