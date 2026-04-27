# Chrome CDP CLI

Lightweight Chrome DevTools Protocol CLI tool. No Puppeteer dependency, connects directly via WebSocket. Supports 100+ tabs with instant connection.

## Prerequisites

- **Node.js 22+** (uses built-in WebSocket)
- **Chrome** (or Chromium, Brave, Edge, Vivaldi)

## Setup

### 1. Enable Remote Debugging

Open Chrome and navigate to:

```
chrome://inspect/#remote-debugging
```

Click **"Allow remote debugging for this browser instance"** to enable it.

> This only needs to be done once per browser session. Chrome will show an "Allow debugging?" prompt the first time you access each tab.

### 2. Verify Connection

```bash
scripts/cdp.mjs list
```

If successful, you'll see a list of open tabs with their target IDs.

## Quick Start

```bash
# List open pages — get target ID prefixes
scripts/cdp.mjs list

# Screenshot
scripts/cdp.mjs shot <target> [file]

# Accessibility tree snapshot
scripts/cdp.mjs snap <target>

# Evaluate JavaScript
scripts/cdp.mjs eval <target> "document.title"

# Navigate to URL
scripts/cdp.mjs nav <target> https://example.com
```

## Network Capture & Replay

Network requests are automatically captured when a tab daemon starts.

```bash
# List captured requests (filter by keyword)
scripts/cdp.mjs netlog <target> [filter]

# Full request/response headers + body
scripts/cdp.mjs netdetail <target> <reqId>

# Replay a request with modified parameters
scripts/cdp.mjs replay <target> <reqId> '{"url":"...","headers":{},"body":"..."}'

# Export as HAR file (importable by Chrome DevTools / Burp Suite)
scripts/cdp.mjs export <target> [file]

# Compare two requests
scripts/cdp.mjs diff <target> <id1> <id2>

# Filter netlog/export by target URL patterns
scripts/cdp.mjs scope <target> "*api*"

# List cookies
scripts/cdp.mjs cookie <target> [name]
```

## WebSocket Monitoring

```bash
scripts/cdp.mjs wslog <target> [filter]       # List connections
scripts/cdp.mjs wsdetail <target> <wsId>       # Show frames
scripts/cdp.mjs wsclear <target>               # Clear log
```

## Interaction

```bash
scripts/cdp.mjs click   <target> <selector>        # Click element
scripts/cdp.mjs clickxy <target> <x> <y>            # Click at coordinates
scripts/cdp.mjs type    <target> <text>              # Type text
scripts/cdp.mjs hover   <target> <selector>          # Hover element
scripts/cdp.mjs key     <target> <keyName>           # Keyboard (Enter, Escape, Tab...)
scripts/cdp.mjs select  <target> <selector> <value>  # Dropdown select
scripts/cdp.mjs wait    <target> <selector> [ms]     # Wait for element (default 10s)
scripts/cdp.mjs scroll  <target> [dir] [px]          # Scroll (up/down/left/right)
scripts/cdp.mjs loadall <target> <selector> [ms]     # Click "load more" until gone
```

## Request Interception

Intercept and modify in-flight requests/responses via CDP Fetch domain.

```bash
# Mock API response
scripts/cdp.mjs intercept <target> "*api/user*" '{"responseBody":"{\"mock\":true}","responseStatus":200}'

# Inject auth header
scripts/cdp.mjs intercept <target> "*api*" '{"requestHeaders":{"Authorization":"Bearer xxx"}}'

# Unlock CORS
scripts/cdp.mjs intercept <target> "*api*" '{"responseHeaders":{"Access-Control-Allow-Origin":"*"}}'

# Manage rules
scripts/cdp.mjs intercept <target> list
scripts/cdp.mjs intercept <target> remove <id>
scripts/cdp.mjs intercept <target> off
```

## Page & Storage

```bash
scripts/cdp.mjs html           <target> [selector]     # Get HTML
scripts/cdp.mjs storage        <target> [key] [value]   # localStorage
scripts/cdp.mjs sessionstorage <target> [key] [value]   # sessionStorage
scripts/cdp.mjs pdf            <target> [file]           # Export PDF
scripts/cdp.mjs dialog         <target> <action>         # Handle dialogs
```

## Encode / Decode

No target needed.

```bash
scripts/cdp.mjs encode base64 "hello world"    # aGVsbG8gd29ybGQ=
scripts/cdp.mjs decode base64 aGVsbG8gd29ybGQ=  # hello world
scripts/cdp.mjs encode url "a=1&b=2"           # a%3D1%26b%3D2
scripts/cdp.mjs encode hex "hello"              # 68656c6c6f
scripts/cdp.mjs decode hex 68656c6c6f           # hello
```

Supported: `base64`, `url`, `html`, `hex`

## Coordinate System

`shot` captures at native resolution. CDP Input events use CSS pixels.

```
CSS px = screenshot px / DPR
```

`shot` prints the DPR for the current page. Typical Retina (DPR=2): divide screenshot coords by 2.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `No DevToolsActivePort found` | Open `chrome://inspect/#remote-debugging` and enable remote debugging |
| `Daemon failed to start` | Click "Allow" in Chrome's debugging prompt |
| Non-standard browser config | Set `CDP_PORT_FILE` env var to your `DevToolsActivePort` path |
