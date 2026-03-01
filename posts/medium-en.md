# I Made My AI Agent's Browser Automation 2.5x Faster — And Agent-Agnostic

*How connecting to my real Chrome via CDP gave any AI agent fast, authenticated browser access*

---

## The Problem With AI Browser Automation

I built a stock trading automation system with Claude Code. Every morning, it analyzes overnight market data, generates pre-market plans, executes intraday trades, and runs post-market reviews. Eight Claude Code skills orchestrate the entire daily workflow.

One of those skills syncs my watchlist to my brokerage account. Simple enough: call an authenticated API to add or remove stock symbols. But here is where things went sideways.

My first approach was "Claude in Chrome" -- Anthropic's feature that embeds Claude into your Chrome browser. It has access to your cookies and login sessions, which is great. It uses DOM inspection and accessibility-based approaches first, but falls back to screenshots when it can't operate directly. When that happens, the browser renders the page, takes a screenshot, converts it to base64, sends it to a vision model, which reasons about what it sees and decides the next action.

For a simple API call that returns 200 bytes of JSON, this vision model overhead added up. And when the page didn't load as expected, Claude would take more screenshots to debug, increasing the cost and time.

Worse, I couldn't even reach some sites. My brokerage platform (10jqka.com.cn) intermittently blocked the automated browser. wise.com, reddit.com, mp.weixin.qq.com -- all refused to serve pages.

I also tried WebFetch -- Claude Code's built-in tool for fetching web pages. It converts HTML to markdown and feeds it back. For simple public pages, it works. But for authenticated APIs, it was unreliable: 503 rate limiting, ~3,000 tokens per call, and a success rate around 70%. Not good enough for a trading system that runs daily.

The core issue: Claude in Chrome has the cookies I need, but it's slow and Claude-only. WebFetch is unreliable and has no cookies. I needed something faster, more reliable, and that would work with any AI agent -- not just Claude.

## What Is CDP?

Chrome DevTools Protocol (CDP) is the same protocol that Chrome DevTools uses to inspect pages, set breakpoints, and profile performance. It's been around since 2017 and is battle-tested -- every Chrome extension and debugging tool relies on it.

The key insight: **CDP can connect to an already-running Chrome instance**. Not a new headless browser. Your actual Chrome, with your cookies, your login sessions, your extensions, your browsing history. Everything.

[agent-browser](https://github.com/vercel-labs/agent-browser) is a CLI tool from Vercel Labs that wraps CDP into clean, scriptable commands. Instead of screenshots and vision models, it gives you:

- `eval` -- execute JavaScript in the page context (including `fetch()` with cookies)
- `snapshot` -- get a compact accessibility tree (not a screenshot)
- `open` / `click` / `fill` -- navigate and interact with pages

The architecture shift looks like this:

```
BEFORE (Claude in Chrome):
  Claude → Your Chrome → JS eval / screenshot fallback → Result
  Speed: ~64s per task | Portability: Claude only

AFTER (agent-browser + CDP):
  Any Agent → Bash: agent-browser eval '...'
           → CDP → Your Chrome → JSON response
  Speed: ~26s per task | Portability: Any agent that can run bash
```

## The Benchmark That Surprised Me

I expected agent-browser to be dramatically cheaper on tokens. I designed a controlled benchmark: open Hacker News, extract the top 10 story titles, return a JSON array. Both approaches ran back-to-back in fresh Claude Code sessions on the same machine. I measured tokens via `/context` before and after.

The result was not what I expected.

### Task: Extract Hacker News Top 10 Titles

| Metric | agent-browser + CDP | Claude in Chrome |
|--------|---------------------|------------------|
| **Total time** | **26s** | **64s** |
| Context delta | +7k tokens | +2k tokens |
| Message tokens | +8k (incl. ~4.9k skill load) | +3.9k |
| Mechanism used | JS eval via Bash CLI | JS eval via MCP tool |

Both approaches used the **same underlying mechanism**: JavaScript evaluation. Both executed `document.querySelectorAll('.titleline > a')` and returned identical results. No screenshots in either session.

Token usage per task was **comparable**. Claude in Chrome actually used fewer context tokens (+2k vs +7k), because agent-browser had to load skill instructions (~4.9k one-time cost). Subtracting the skill load, task messages were similar (~3.1k vs ~3.9k).

### So Where's the Real Advantage?

**Idle cost.** This is the architectural win. Claude in Chrome loads 18 MCP tools (~5,600 tokens) into every API call for the entire session — even when you're writing code, not using the browser. agent-browser's skill descriptions cost ~586 tokens when idle. That's a 10x difference that compounds over a full coding session.

**Works with any AI agent.** agent-browser is a CLI tool. Any AI agent that can run bash — Claude Code, Codex, Cursor, Windsurf, Copilot — can use it. Claude in Chrome only works with Claude. This is not a Claude-specific optimization; it's a universal solution.

**Works with any site.** Because agent-browser connects to your real Chrome via CDP, any site you can open in Chrome, the agent can access. Claude in Chrome restricts access on some sites — wise.com, reddit.com, mp.weixin.qq.com all refused to serve pages under its automation.

**Speed.** agent-browser was 2.5x faster in our benchmark (26s vs 64s). The browser operations were similar (~11s vs ~7s), but agent-browser had less API roundtrip overhead.

### What Claude in Chrome Does Better

To be fair: Claude in Chrome has a **vision fallback**. When JavaScript can't handle a task (complex visual UIs, unfamiliar page layouts), it can fall back to screenshots + vision model. agent-browser has `snapshot` (accessibility trees) but no vision capability. For visual exploration of unknown sites, Claude in Chrome is the better tool.

## Setup in 5 Minutes

### Step 1: Enable Chrome Remote Debugging

In your Chrome address bar, go to:

```
chrome://inspect/#remote-debugging
```

Check **"Allow remote debugging for this browser instance"**. No restart needed. This requires Chrome 145+.

One caveat: after restarting Chrome, you need to re-enable this. Chrome intentionally does not persist CDP access for security reasons.

### Step 2: Install and Configure

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install

# Connect to your real Chrome instead of launching headless
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{ "autoConnect": true }
EOF
```

### Step 3: Verify Connection

```bash
agent-browser get url
# Should print the URL of your current Chrome tab
```

### Step 4: Try It Out

```bash
# Navigate to a site
agent-browser open "https://news.ycombinator.com"
agent-browser wait --load networkidle

# Get a compact accessibility tree (not a screenshot!)
agent-browser snapshot
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
agent-browser open "https://api.example.com/"
agent-browser wait 2000

# Call the authenticated API -- cookies are sent automatically
agent-browser eval --stdin <<'EOF'
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
   agent-browser open "https://t.10jqka.com.cn/"
   agent-browser wait 2000

2. Call the authenticated API:
   agent-browser eval --stdin <<'EOF'
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
    → agent-browser(calls authenticated API)
      → Chrome CDP → Broker API (with user's cookies)
```

When I execute a trade, the `/stock-trade` skill records it in the database and then automatically invokes `/stock-watchlist` to update my brokerage's watchlist. The entire chain -- database write, API call, verification -- completes in under 5 seconds and costs about 500 tokens total.

This is a pattern I call **"skill chaining"** -- each skill focuses on one responsibility and delegates to the next. The trade skill knows about PostgreSQL but nothing about browser automation. The watchlist skill knows about agent-browser but nothing about database schemas. Clean separation of concerns, orchestrated by Claude Code.

The skill approach also gives you a natural place to encode domain knowledge. My watchlist skill knows that the brokerage API uses `marketid=17` for Shanghai stocks and `marketid=33` for Shenzhen stocks. That mapping lives in the skill's markdown instructions, not in hardcoded config files. When the brokerage changes their API, I update one markdown file.

## Skill vs MCP: A Token Economics Analysis

agent-browser ships with both a CLI (for the Skill approach) and an MCP server. You might think MCP is the "proper" integration. Let me walk through why the Skill approach wins on token economics.

### Idle Cost

When you register an MCP server with Claude Code, its tool schemas are loaded into the context window **at all times**. In our benchmark, Claude in Chrome loaded 18 MCP tools costing **~5,600 tokens** — permanently consumed even when you're writing code, not using the browser.

A Skill, by contrast, is just a markdown file with a name and description. When not active, it costs **~586 tokens** (just the one-line descriptions in the skill registry). The full instructions (~4.9k) only load when the skill is invoked.

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
agent-browser eval 'document.title'
```

~50 tokens. And you can chain commands with `&&` to reduce round-trips further.

### The Math

Consider a typical session where you do 30 minutes of coding, then 2 minutes of browser automation (5 actions), then 30 more minutes of coding.

**MCP approach (measured):**
- Idle cost: 5,600 tokens (18 MCP tools, loaded into every API call)
- Action cost: 5 x 800 tokens = 4,000 tokens
- Total: **9,600 tokens**

**Skill approach (measured):**
- Idle cost: 586 tokens (skill descriptions)
- Skill load: ~4,900 tokens (loaded on invocation, unloaded after)
- Action cost: 5 x 600 tokens = 3,000 tokens
- Total: **8,486 tokens**

The per-session difference is modest. But **MCP's 5,600 tokens load into every API call** — even during pure coding work. Over a full session with intermittent browser use, the idle overhead compounds.

### Maturity

There's also a practical maturity consideration. The CLI is officially maintained by Vercel Labs and ships with an official SKILL.md template. The MCP wrapper is a community contribution by a single author. Vercel has closed multiple MCP-related PRs, signaling that the CLI/Skill path is their recommended integration.

To be fair, MCP has its strengths. If you're building a multi-tool integration that needs to expose browser capabilities to *any* MCP-compatible client (not just Claude Code), the MCP path gives you broader compatibility. But for the common case -- a single developer using Claude Code who occasionally needs browser access -- the Skill approach is strictly better on every metric that matters.

## Gotchas From Production

After running this in production for several weeks, here are the hard-won lessons.

### CORS: The Subdomain Trap

This one cost me two hours. My brokerage's API lives at `t.10jqka.com.cn`, but their main site is `www.10jqka.com.cn`. I navigated to the main site and tried to fetch from the API subdomain:

```bash
# WRONG
agent-browser open "https://www.10jqka.com.cn"
agent-browser eval 'fetch("https://t.10jqka.com.cn/api/data")'
# TypeError: Failed to fetch
```

Same-origin policy. The browser treats `www.` and `t.` as different origins. The fix:

```bash
# RIGHT
agent-browser open "https://t.10jqka.com.cn/"
agent-browser eval 'fetch("https://t.10jqka.com.cn/api/data")'
# Works!
```

**Rule:** Always navigate to the *exact* subdomain of the API you're calling.

### Shell Escaping With Heredocs

JavaScript is full of characters that shells love to interpret: `$`, backticks, `!`, `{}`. Use single-quoted heredocs to prevent shell expansion:

```bash
# WRONG — shell expands $, !, backticks
agent-browser eval "fetch(`https://api.com/${path}`)"

# RIGHT — heredoc bypasses shell
agent-browser eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

The `<<'EOF'` (with quotes around EOF) is critical. Without quotes, the shell still expands variables inside the heredoc.

### Rate Limiting

When chaining multiple API calls, add explicit delays:

```bash
agent-browser eval '...'  # Call 1
agent-browser wait 2000    # Wait 2s
agent-browser eval '...'  # Call 2
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
result=$(agent-browser eval --stdin <<'EOF'
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

This isn't about "agent-browser vs Claude in Chrome." Each tool has its place:

- **Claude in Chrome**: Best for visual exploration of unknown sites, and tasks where the agent needs to *see* the page. Has a vision fallback that agent-browser lacks.
- **agent-browser + CDP**: Best for speed, portability, and keeping your context window clean during long coding sessions.

The insight is simpler than I originally thought. I expected a massive token gap. What I found was: **for JavaScript eval tasks, both approaches use comparable tokens**. The real wins are idle cost (10x lower), and dual universality — any agent, not just Claude; any site, not just automation-friendly ones.

That universality point is worth emphasizing. agent-browser is a CLI tool. Any AI agent that can execute shell commands — Claude Code, Codex, Cursor, Windsurf, Copilot — gets the same capabilities with the same setup. No SDK, no vendor lock-in. And because it connects to your real Chrome, any site you can access manually, your agent can access too. No more "this site blocks automated browsers."

The tools are maturing fast. A year ago, giving an AI agent browser access meant Puppeteer scripts and fragile selectors. Today, agent-browser gives you a clean CLI that connects to your real Chrome in one line.

Start with one task. Measure the speed. Compare to your current approach. I think you'll appreciate the difference.

---

*The full guide, with benchmarks, code examples, and Claude Code skill templates, is at [github.com/chenyanchen/agent-browser-guide](https://github.com/chenyanchen/agent-browser-guide).*
