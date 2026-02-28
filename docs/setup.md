# Setup Guide

## Prerequisites

- Chrome (or Chromium-based browser)
- Node.js 18+ (for npm install) or Homebrew (macOS)

## Enabling Chrome DevTools Protocol (CDP)

### Chrome 145+ (Recommended)

No restart needed. Just:

1. Open Chrome
2. Navigate to `chrome://inspect/#remote-debugging`
3. Check **"Allow remote debugging for this browser instance"**

Chrome now exposes CDP on `localhost:9222`. CDP stays enabled until Chrome restarts.

**Note:** After a Chrome restart, you must re-visit `chrome://inspect/#remote-debugging` to re-enable CDP.

### Older Chrome (< 145) or Custom Port

Launch Chrome with a command-line flag:

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-debug-profile"
```

```bash
# Linux
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-debug-profile"
```

**Important (Chrome 136+):** The `--user-data-dir` flag is required. Without it, Chrome silently ignores `--remote-debugging-port`. This was a breaking change in Chrome 136. The directory doesn't need to exist -- Chrome creates it. But this means you get a fresh profile (no cookies, no logins) unless you point it to your real profile directory.

## Installing agent-browser

### macOS (Homebrew)

```bash
brew install agent-browser
```

### macOS / Linux (npm)

```bash
npm install -g agent-browser && agent-browser install
```

### Verify installation

```bash
agent-browser --version
```

## Configure Auto-Connect

By default, agent-browser launches a new headless browser. If you need your browser's cookies and login sessions (authenticated APIs, sites that block automation), configure auto-connect to use your running Chrome instead:

```bash
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{
  "autoConnect": true
}
EOF
```

With this config, all `agent-browser` commands automatically connect to your running Chrome.

## Verifying the Connection

With Chrome running and CDP enabled:

```bash
agent-browser get url
```

Expected output: the URL of your current Chrome tab. If you see a URL, everything is working.

## Quick Smoke Test

```bash
# Navigate to a page
agent-browser open 'https://example.com'

# Read the title
agent-browser eval 'document.title'
# Expected: "Example Domain"

# Read the heading
agent-browser eval 'document.querySelector("h1").textContent'
# Expected: "Example Domain"
```

## Troubleshooting

### "No browser found" or connection refused

```
Error: connect ECONNREFUSED 127.0.0.1:9222
```

**Cause:** CDP is not enabled.

**Fix:**
- Chrome 145+: Visit `chrome://inspect/#remote-debugging`
- Older Chrome: Relaunch with `--remote-debugging-port=9222`

### "No pages found"

```
No pages available
```

**Cause:** Chrome has no regular tabs open (only internal pages like `chrome://settings`).

**Fix:** Open at least one regular webpage (e.g., `https://example.com`).

### Port already in use

```
Error: listen EADDRINUSE :::9222
```

**Cause:** Another Chrome instance or process is using port 9222.

**Fix:**
```bash
# Find what's using the port
lsof -i :9222
# Kill it or use a different port
```

### Chrome 136+ ignoring --remote-debugging-port

**Cause:** Missing `--user-data-dir` flag.

**Fix:** Always pass `--user-data-dir` when using the command-line approach:
```bash
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
```

### eval returns undefined

**Cause:** The expression doesn't return a value, or the element doesn't exist.

**Fix:** Ensure your JavaScript expression returns something:
```bash
# Wrong: assignment doesn't return
agent-browser eval 'const x = 1'

# Right: expression returns a value
agent-browser eval '1 + 1'
agent-browser eval 'document.title'
```

### Navigation timeout

**Cause:** The page takes too long to load or redirects infinitely.

**Fix:** Check the URL manually in Chrome first. Some pages require authentication or block automated access.

## Next Steps

- [Quick start example](../examples/01-quick-start.md)
- [Gotchas from real usage](gotchas.md)
