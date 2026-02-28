# I Cut My AI Agent's Browser Tokens by 95% With One Architecture Change

*How connecting Claude to my real Chrome via CDP replaced screenshots with surgical API calls*

---

## The Problem With AI Browser Automation

I build a stock trading automation system with Claude Code. Every morning, it analyzes overnight market data, generates pre-market plans, executes intraday trades, and runs post-market reviews. Eight Claude Code skills orchestrate the entire daily workflow.

One of those skills syncs my watchlist to my brokerage account. Simple enough: call an authenticated API to add or remove stock symbols. But here is where things went sideways.

My first approach was "Chrome in Claude" -- Anthropic's computer use feature that gives Claude a browser. It works like this: the agent sends a command, the browser renders the page, takes a screenshot, converts it to base64, sends it to a vision model, which reasons about what it sees and decides the next action.

For a simple API call that returns 200 bytes of JSON, this process consumed **4,000-6,000 tokens** and took **8-15 seconds**. And that was the *happy path*. When the page didn't load as expected, Claude would take more screenshots to debug, doubling the cost and time.

Worse, I couldn't even reach some sites. My brokerage platform (10jqka.com.cn) intermittently blocked the automated browser. wise.com, reddit.com, mp.weixin.qq.com -- all refused to serve pages to headless Chrome instances.

And the authentication problem was the real killer. "Chrome in Claude" spins up a fresh browser instance. No cookies. No login sessions. Every authenticated API call required me to somehow inject credentials, which is both fragile and a security concern.

I needed a fundamentally different approach.

I also tried WebFetch -- Claude Code's built-in tool for fetching web pages. It converts HTML to markdown and feeds it back. For simple public pages, it works. But for authenticated APIs, it was unreliable: 503 rate limiting, ~3,000 tokens per call, and a success rate around 70%. Not good enough for a trading system that runs daily.

The core problem with all these approaches is the same: they try to **create a new browser environment** to simulate the user. But the user already has a perfectly good browser environment -- their running Chrome instance, with all the cookies and login sessions already in place. Why not just use that?

## What Is CDP?

Chrome DevTools Protocol (CDP) is the same protocol that Chrome DevTools uses to inspect pages, set breakpoints, and profile performance. It's been around since 2017 and is battle-tested -- every Chrome extension and debugging tool relies on it.

The key insight: **CDP can connect to an already-running Chrome instance**. Not a new headless browser. Your actual Chrome, with your cookies, your login sessions, your extensions, your browsing history. Everything.

[agent-browser](https://github.com/vercel-labs/agent-browser) is a CLI tool from Vercel Labs that wraps CDP into clean, scriptable commands. Instead of screenshots and vision models, it gives you:

- `eval` -- execute JavaScript in the page context (including `fetch()` with cookies)
- `snapshot` -- get a compact accessibility tree (not a screenshot)
- `open` / `click` / `fill` -- navigate and interact with pages

The architecture shift looks like this:

```
BEFORE (Chrome in Claude):
  Agent → Headless Chrome → Screenshot → Base64 → Vision Model → Reasoning
  Cost: 2,000-8,000 tokens per action, 8-15s

AFTER (agent-browser + CDP):
  Agent → Bash: agent-browser --auto-connect eval '...'
       → CDP → Your Chrome → JSON response
  Cost: 50-200 tokens per action, 2-4s
```

## The Benchmark That Changed My Mind

I ran four representative tasks from my stock trading system and measured tokens and time for each approach. These are real measurements, not synthetic benchmarks.

### Per-Task Comparison

| Task | agent-browser Time | agent-browser Tokens | Chrome in Claude Tokens | Reduction |
|------|-------------------|---------------------|------------------------|-----------|
| Stock price JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| Authenticated API call | 3.5s | **217** | 3,000 | 14x |
| Financial page snapshot | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| Search page snapshot | 1.8s | **41** | 5,000-8,000 | 122-195x |
| **Total (4 tasks)** | **10.8s** | **2,092** | **20,000-32,000** | **10-15x** |

The numbers speak for themselves. But there are a few things worth highlighting:

**The JSON API case is the most dramatic.** When all you need is structured data from an API, agent-browser returns the raw JSON -- 57 tokens. Chrome in Claude has to render a page, screenshot it, OCR the content, and reason about it. 4,000-6,000 tokens for the same 200 bytes of data.

**The financial page case shows the floor.** When you genuinely need page content (not just an API response), agent-browser's `snapshot` command returns a compact accessibility tree. At 1,777 tokens, it's still 4.5-8x cheaper than a screenshot-based approach, but the gap is narrower because you're actually fetching substantial content either way.

**The search page result is surprising.** A search results page compresses remarkably well into an accessibility tree -- the structured nature of search results (title, URL, snippet) maps perfectly to accessibility nodes. 41 tokens vs 5,000-8,000.

### Overall Experience

| Metric | Chrome in Claude | agent-browser + CDP |
|--------|-----------------|-------------------|
| Tokens per action | ~2,000-8,000 | **~50-1,800** |
| Speed per action | ~8-15s (screenshots + vision) | **~2-4s** |
| When stuck | More screenshots, 2x slower | No such issue |
| Site access | wise.com, reddit, WeChat blocked | **Everything works** |
| Login state | None (fresh browser) | **Full (your real Chrome)** |

The speed difference compounds. A workflow that chains 5 browser actions goes from 40-75 seconds to 10-20 seconds. And when Chrome in Claude gets confused (which happens often on complex pages), it enters a screenshot loop that can easily burn 20,000+ tokens before giving up.

## Setup in 5 Minutes

### Step 1: Enable Chrome Remote Debugging

In your Chrome address bar, go to:

```
chrome://inspect/#remote-debugging
```

Click **"Enable"**. That's it. No restart needed. This requires Chrome 145+.

One caveat: after restarting Chrome, you need to re-enable this. Chrome intentionally does not persist CDP access for security reasons.

### Step 2: Install agent-browser

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install
```

### Step 3: Verify Connection

```bash
agent-browser --auto-connect get url
# Should print the URL of your current Chrome tab
```

### Step 4: Try It Out

```bash
# Navigate to a site
agent-browser --auto-connect open "https://news.ycombinator.com"
agent-browser --auto-connect wait --load networkidle

# Get a compact accessibility tree (not a screenshot!)
agent-browser --auto-connect snapshot
```

You should see something like:

```
- heading "Hacker News" [level=1]
- link "new" [ref=e1]
- link "past" [ref=e2]
- ...
```

That's an accessibility tree. Compact, structured, and costs a fraction of a screenshot.

### Step 5: Call an Authenticated API

This is where the magic happens. Navigate to a site you're already logged into, then use `eval` with `fetch()`:

```bash
# Navigate to the API's domain (required for same-origin cookies)
agent-browser --auto-connect open "https://api.example.com/"
agent-browser --auto-connect wait 2000

# Call the authenticated API -- cookies are sent automatically
agent-browser --auto-connect eval --stdin <<'EOF'
fetch("https://api.example.com/user/data")
  .then(r => r.json())
  .then(d => JSON.stringify(d))
EOF
```

The `fetch()` runs inside your Chrome's page context, so it inherits all the cookies and session state. No credential injection, no OAuth flows, no token management. If you can see the data when you visit the page manually, the agent can fetch it programmatically.

## Building a Claude Code Skill

Here's where it gets interesting. Claude Code has a "skills" system -- reusable instructions that the agent can invoke. You can build a skill that wraps agent-browser commands.

Here's a simplified version of my watchlist sync skill:

```yaml
---
name: stock-watchlist
description: Sync stock watchlist to brokerage via browser automation
allowed-tools: Bash(agent-browser:*)
---
```

The `allowed-tools` frontmatter is critical. It pre-authorizes any bash command starting with `agent-browser:` -- so the agent won't pause to ask for permission on every browser action.

The skill body (written in markdown) contains the instructions:

```markdown
## Steps

1. Navigate to the brokerage domain (required for CORS):
   agent-browser --auto-connect open "https://t.10jqka.com.cn/"
   agent-browser --auto-connect wait 2000

2. Call the authenticated API:
   agent-browser --auto-connect eval --stdin <<'EOF'
   fetch("https://t.10jqka.com.cn/api/watchlist/add", {
     method: "POST",
     headers: { "Content-Type": "application/json" },
     body: JSON.stringify({ codes: ["600519", "000858"] })
   }).then(r => r.json()).then(d => JSON.stringify(d))
   EOF

3. Verify the response and report results.
```

### Skill Chaining

The real power emerges when skills chain together:

```
/stock-trade (records trade in PostgreSQL)
  → /stock-watchlist (syncs to brokerage watchlist)
    → agent-browser --auto-connect (calls authenticated API)
      → Chrome CDP → Broker API (with user's cookies)
```

When I execute a trade, the `/stock-trade` skill records it in the database and then automatically invokes `/stock-watchlist` to update my brokerage's watchlist. The entire chain -- database write, API call, verification -- completes in under 5 seconds and costs about 500 tokens total.

This is a pattern I call **"skill chaining"** -- each skill focuses on one responsibility and delegates to the next. The trade skill knows about PostgreSQL but nothing about browser automation. The watchlist skill knows about agent-browser but nothing about database schemas. Clean separation of concerns, orchestrated by Claude Code.

The skill approach also gives you a natural place to encode domain knowledge. My watchlist skill knows that the brokerage API uses `marketid=17` for Shanghai stocks and `marketid=33` for Shenzhen stocks. That mapping lives in the skill's markdown instructions, not in hardcoded config files. When the brokerage changes their API, I update one markdown file.

## Skill vs MCP: A Token Economics Analysis

agent-browser ships with both a CLI (for the Skill approach) and an MCP server. You might think MCP is the "proper" integration. Let me walk through why the Skill approach wins on token economics.

### Idle Cost

When you register an MCP server with Claude Code, its tool schemas are loaded into the context window **at all times**. agent-browser's MCP server exposes 40+ tools with detailed schemas. That's **12,000-24,000 tokens** permanently consumed, even when you're doing non-browser work like editing code or writing tests.

A Skill, by contrast, is just a markdown file with a name and description. When not active, it costs **~150 tokens** (just the one-line description in the skill registry). The full instructions only load when the skill is invoked.

### Per-Action Cost

MCP tool calls require structured input/output with JSON schemas. A single `evaluate` call might look like:

```json
{
  "tool": "browser_evaluate",
  "arguments": {
    "expression": "document.title",
    "awaitPromise": true
  }
}
```

That's ~100-200 tokens just for the call structure. The Skill approach uses bash one-liners:

```bash
agent-browser --auto-connect eval 'document.title'
```

~50 tokens. And you can chain commands with `&&` to reduce round-trips further.

### The Math

Consider a typical session where you do 30 minutes of coding, then 2 minutes of browser automation (5 actions), then 30 more minutes of coding.

**MCP approach:**
- Idle cost: 18,000 tokens (average) x 60 minutes = 18,000 tokens (loaded once, persistent)
- Action cost: 5 x 1,000 tokens = 5,000 tokens
- Total: **23,000 tokens**

**Skill approach:**
- Idle cost: 150 tokens (just the description)
- Skill load: ~800 tokens (loaded on invocation, unloaded after)
- Action cost: 5 x 150 tokens = 750 tokens
- Total: **1,700 tokens**

That's a **13.5x difference**. And the gap widens the more time you spend *not* using the browser, which is most of the time.

### Maturity

There's also a practical maturity consideration. The CLI is officially maintained by Vercel Labs and ships with an official SKILL.md template. The MCP wrapper is a community contribution by a single author. Vercel has closed multiple MCP-related PRs, signaling that the CLI/Skill path is their recommended integration.

To be fair, MCP has its strengths. If you're building a multi-tool integration that needs to expose browser capabilities to *any* MCP-compatible client (not just Claude Code), the MCP path gives you broader compatibility. But for the common case -- a single developer using Claude Code who occasionally needs browser access -- the Skill approach is strictly better on every metric that matters.

## Gotchas From Production

After running this in production for several weeks, here are the hard-won lessons.

### CORS: The Subdomain Trap

This one cost me two hours. My brokerage's API lives at `t.10jqka.com.cn`, but their main site is `www.10jqka.com.cn`. I navigated to the main site and tried to fetch from the API subdomain:

```bash
# WRONG
agent-browser --auto-connect open "https://www.10jqka.com.cn"
agent-browser --auto-connect eval 'fetch("https://t.10jqka.com.cn/api/data")'
# TypeError: Failed to fetch
```

Same-origin policy. The browser treats `www.` and `t.` as different origins. The fix:

```bash
# RIGHT
agent-browser --auto-connect open "https://t.10jqka.com.cn/"
agent-browser --auto-connect eval 'fetch("https://t.10jqka.com.cn/api/data")'
# Works!
```

**Rule:** Always navigate to the *exact* subdomain of the API you're calling.

### Shell Escaping With Heredocs

JavaScript is full of characters that shells love to interpret: `$`, backticks, `!`, `{}`. Use single-quoted heredocs to prevent shell expansion:

```bash
# WRONG — shell expands $, !, backticks
agent-browser --auto-connect eval "fetch(`https://api.com/${path}`)"

# RIGHT — heredoc bypasses shell
agent-browser --auto-connect eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

The `<<'EOF'` (with quotes around EOF) is critical. Without quotes, the shell still expands variables inside the heredoc.

### Rate Limiting

When chaining multiple API calls, add explicit delays:

```bash
agent-browser --auto-connect eval '...'  # Call 1
agent-browser --auto-connect wait 2000    # Wait 2s
agent-browser --auto-connect eval '...'  # Call 2
```

Without delays, rapid-fire requests can trigger rate limiting on the target API, leading to 429 errors or temporary bans.

### Security Considerations

CDP gives **full access** to your browser -- all tabs, all cookies, all localStorage. This is by design (you're connecting to your own browser), but it means:

- Only enable CDP on your personal machine
- Be mindful of which sites you're logged into
- Don't share your CDP port over the network
- After restarting Chrome, CDP access is disabled by default (a good safety feature)

### Error Handling in Automation

When agent-browser commands fail (network timeout, element not found, API error), the error is returned as plain text in the bash output. Claude Code sees the error and can reason about it without taking screenshots. This is a subtle but important advantage: debugging costs tokens proportional to the error message, not proportional to a re-rendered page screenshot.

In practice, I wrap critical agent-browser calls in bash conditionals:

```bash
result=$(agent-browser --auto-connect eval --stdin <<'EOF'
fetch("https://api.example.com/data")
  .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
  .then(d => JSON.stringify(d))
  .catch(e => JSON.stringify({ error: e.message }))
EOF
)
echo "$result"
```

The agent sees either clean JSON or a structured error. No ambiguity, no screenshot-based debugging loops.

## Conclusion

This isn't about "agent-browser vs MCP" or "CDP vs computer use." Each tool has its place:

- **Computer use / MCP browser tools**: Best for visual exploration of unknown sites, visual verification, and tasks where the agent needs to *see* the page.
- **agent-browser + CDP**: Best for structured data extraction, authenticated API calls, and repeated workflows where you know the target pages.

The insight is simple: **most browser automation tasks don't need vision**. They need data. When you replace "render page -> screenshot -> base64 -> vision model -> extract text" with "fetch() -> JSON," you cut tokens by 95% and time by 75%.

If your AI agent regularly interacts with the web -- especially authenticated APIs -- try the CDP approach. The setup takes 5 minutes, and the token savings compound with every action.

The tools are maturing fast. A year ago, giving an AI agent browser access meant Puppeteer scripts and fragile selectors. Today, agent-browser gives you a clean CLI that connects to your real Chrome in one line. The developer experience is already good. The token economics make it a no-brainer for structured, repeatable browser tasks.

Start with one task. Measure the tokens. Compare to your current approach. I think you'll be surprised by the difference.

---

*The full guide, with benchmarks, code examples, and Claude Code skill templates, is at [github.com/chenyanchen/agent-browser-guide](https://github.com/chenyanchen/agent-browser-guide).*
