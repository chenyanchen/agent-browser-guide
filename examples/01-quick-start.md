# Quick Start: Hello World with agent-browser

Get Claude Code controlling your browser in 3 steps.

## Step 1: Enable Chrome DevTools Protocol (CDP)

Open Chrome and navigate to:

```
chrome://inspect/#remote-debugging
```

Click **"Configure..."** if needed, ensure `localhost:9222` is listed. That's it -- no restart required (Chrome 145+).

## Step 2: Install agent-browser

```bash
brew install anthropics/tap/agent-browser
# or
npm install -g @anthropic-ai/agent-browser
```

## Step 3: Verify the Connection

```bash
# List all open tabs
agent-browser list-pages
```

You should see output like:

```
Page 0: "Google" - https://www.google.com/
Page 1: "GitHub" - https://github.com/
```

If you see your open tabs, you're connected.

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
agent-browser navigate 'https://example.com'
agent-browser eval 'document.querySelector("h1").textContent'
```

```
Example Domain
```

### Run a multi-line script

```bash
agent-browser eval "$(cat <<'JSEOF'
const links = Array.from(document.querySelectorAll('a'));
links.map(a => `${a.textContent.trim()} -> ${a.href}`).join('\n');
JSEOF
)"
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
