# Quick Start: Hello World with agent-browser

Get Claude Code controlling your browser in 4 steps.

## What

Connect agent-browser to your running Chrome via CDP, so AI agents inherit your cookies, login sessions, and extensions.

## Why

By default, agent-browser launches a new headless browser — no cookies, no login state. That's fine for public pages. But if your agent needs authenticated APIs, restricted sites (WeChat, Reddit, wise.com), or internal tools behind SSO — you need to connect to your **real Chrome**.

## How

### Step 1: Enable Chrome DevTools Protocol (CDP)

Open Chrome and navigate to:

```
chrome://inspect/#remote-debugging
```

Check **"Allow remote debugging for this browser instance"**. No restart required. (Chrome 145+)

### Step 2: Install agent-browser

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install
```

### Step 3: Configure Auto-Connect

```bash
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{ "autoConnect": true }
EOF
```

This tells agent-browser to connect to your running Chrome instead of launching a headless instance.

### Step 4: Verify

```bash
agent-browser get url
```

You should see the URL of your current Chrome tab. If you see a URL, you're connected.

## Your First Commands

### Get the page title

```bash
agent-browser eval 'document.title'
```

```
Google
```

### Get the current URL

```bash
agent-browser eval 'window.location.href'
```

```
https://www.google.com/
```

### Navigate to a page and extract content

```bash
agent-browser open 'https://example.com'
agent-browser eval 'document.querySelector("h1").textContent'
```

```
Example Domain
```

### Run a multi-line script

```bash
agent-browser eval --stdin <<'EOF'
const links = Array.from(document.querySelectorAll('a'));
links.map(a => `${a.textContent.trim()} -> ${a.href}`).join('\n');
EOF
```

## What Just Happened?

You used CDP (Chrome DevTools Protocol) to talk to your real Chrome browser -- the same one where you're logged in to everything. This means agent-browser can:

- Read any page you can see
- Execute JavaScript in any tab
- Use your authenticated sessions (cookies, tokens, everything)

No puppeteer. No headless browser. No re-authentication.

## Next Steps

- [Authenticated API calls](02-authenticated-api.md) -- use your browser's cookies to call APIs
- [Building a Claude Code skill](03-claude-code-skill.md) -- wrap agent-browser in a reusable skill
- [Setup details & troubleshooting](../docs/setup.md) -- if something didn't work
