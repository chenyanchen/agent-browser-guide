# Quick Start: Hello World with agent-browser

Get Claude Code controlling your browser in 3 steps.

## Step 1: Enable Chrome DevTools Protocol (CDP)

Open Chrome and navigate to:

```
chrome://inspect/#remote-debugging
```

Click **"Enable"**. That's it -- no restart required. (Requires Chrome 145+)

## Step 2: Install and Configure

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install

# Enable auto-connect (so you don't need --auto-connect on every command)
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{ "autoConnect": true }
EOF
```

## Step 3: Verify the Connection

```bash
# Get the URL of your current Chrome tab
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
