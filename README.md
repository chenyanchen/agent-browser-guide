# Stop Wasting 95% of Your Tokens on Browser Automation

> Connect AI agents to your real Chrome via CDP. Same browser, same cookies, 20x cheaper.

[中文版](README.zh-CN.md)

AI agents need to interact with the web — checking prices, calling APIs, filling forms. But current approaches burn through tokens like there's no tomorrow.

**This guide shows a better way:** use [agent-browser](https://github.com/vercel-labs/agent-browser) to connect to your **existing Chrome** via CDP (Chrome DevTools Protocol). Your cookies, your login sessions, your extensions — at a fraction of the token cost.

## The Problem

"Chrome in Claude" (computer use / MCP browser tools) has three fundamental problems:

| Problem | Why It Hurts |
|---------|-------------|
| **Screenshot-heavy** | Each action: render → screenshot → base64 → vision model → reason. One click costs ~2,000-8,000 tokens. |
| **No login state** | Fresh browser instance = no cookies. You can't call authenticated APIs. |
| **Site restrictions** | Many sites block automated browsers: wise.com, reddit.com, mp.weixin.qq.com, and more. |

And when things go wrong? The agent takes more screenshots to debug, doubling the token cost and time.

## The Solution

```bash
agent-browser --auto-connect eval 'document.title'
```

One line. Connects to your running Chrome via CDP. Your cookies. Your login sessions. Everything just works.

```
AI Agent (Claude Code / Cursor / Codex / etc.)
  └── bash: agent-browser --auto-connect <command>
        └── Chrome DevTools Protocol (CDP)
              └── Your Chrome (with all cookies, extensions, login state)
```

## Benchmarks

Real measurements from a stock trading automation system (2026-02-28):

### Per-Task Comparison

| Task | agent-browser Time | agent-browser Tokens | Chrome in Claude Tokens | Reduction |
|------|-------------------|---------------------|------------------------|-----------|
| Stock price JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| Authenticated API call | 3.5s | **217** | 3,000 | 14x |
| Financial page snapshot | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| Search page snapshot | 1.8s | **41** | 5,000-8,000 | 122-195x |
| **Total (4 tasks)** | **10.8s** | **2,092** | **20,000-32,000** | **10-15x** |

### Overall Comparison

| Metric | Chrome in Claude | agent-browser + CDP |
|--------|-----------------|-------------------|
| Tokens per action | ~2,000-8,000 | **~50-1,800** |
| Speed per action | ~8-15s (screenshots + vision) | **~2-4s** |
| When stuck | More screenshots, 2x slower | No such issue |
| Site access | wise.com, reddit, WeChat blocked | **Everything works** |
| Login state | None (fresh browser) | **Full (your real Chrome)** |
| Idle token overhead | MCP: 12-24K tokens permanent | Skill: ~150 tokens |

> Methodology: [benchmarks/results.md](benchmarks/results.md)

## Quick Start

### 1. Enable Chrome Remote Debugging

In your Chrome address bar, go to:

```
chrome://inspect/#remote-debugging
```

Click **"Enable"**. That's it. No restart needed. (Requires Chrome 145+)

### 2. Install agent-browser

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install
```

### 3. Verify Connection

```bash
agent-browser --auto-connect get url
# Should print the URL of your current Chrome tab
```

You're done. Your AI agent can now control your Chrome.

## Integration Patterns

### Pattern A: Fetch JSON APIs (eval + fetch)

Use your browser's cookies to call authenticated APIs directly:

```bash
# Navigate to the API's domain first (for same-origin cookies)
agent-browser --auto-connect open "https://api.example.com/"
agent-browser --auto-connect wait 2000

# Call the API — cookies are sent automatically
agent-browser --auto-connect eval --stdin <<'EOF'
fetch("https://api.example.com/user/watchlist")
  .then(r => r.json())
  .then(d => JSON.stringify(d))
EOF
```

**Output:** Pure JSON. ~100-300 tokens. No HTML, no markdown, no noise.

### Pattern B: Navigate + Interact (snapshot + refs)

For pages that need clicking, filling forms, or reading content:

```bash
agent-browser --auto-connect open "https://example.com/dashboard"
agent-browser --auto-connect wait --load networkidle
agent-browser --auto-connect snapshot -i
# Output:
# - button "Sign Out" [ref=e1]
# - link "Settings" [ref=e2]
# - textbox "Search" [ref=e3]

agent-browser --auto-connect fill @e3 "query"
agent-browser --auto-connect press Enter
```

**Output:** Compact accessibility tree with refs. ~200-2,000 tokens per page (vs 8,000-15,000 for WebFetch).

### Pattern C: Claude Code Skill

Build reusable skills that use agent-browser under the hood:

```yaml
---
name: my-browser-skill
description: Does X via the user's browser
allowed-tools: Bash(agent-browser:*)
---
```

The `allowed-tools` frontmatter pre-authorizes agent-browser commands — no per-call permission prompts.

See [examples/03-claude-code-skill.md](examples/03-claude-code-skill.md) for a full template.

### Pattern D: Skill Chaining

Chain multiple skills for complex workflows:

```
/stock-trade (records trade in PostgreSQL)
  → /stock-watchlist (syncs to brokerage watchlist)
    → agent-browser --auto-connect (calls authenticated API)
      → Chrome CDP → Broker API (with user's cookies)
```

See [examples/04-skill-chaining.md](examples/04-skill-chaining.md) for the full architecture.

## Gotchas

### CORS: Navigate to the Right Subdomain

APIs only receive cookies from the **same origin**. If the API is on `t.example.com`, you must navigate there first:

```bash
# WRONG — CORS failure
agent-browser --auto-connect open "https://www.example.com"
agent-browser --auto-connect eval 'fetch("https://t.example.com/api/data")'
# TypeError: Failed to fetch

# RIGHT — same origin
agent-browser --auto-connect open "https://t.example.com/"
agent-browser --auto-connect eval 'fetch("https://t.example.com/api/data")'
# Works!
```

### Chrome Restart

After restarting Chrome, you need to re-enable remote debugging at `chrome://inspect/#remote-debugging`. This is by design — Chrome doesn't persist CDP access for security.

### Shell Escaping

Use `<<'EOF'` (single-quoted heredoc) to avoid shell interpretation of your JavaScript:

```bash
# WRONG — shell expands $, !, backticks
agent-browser --auto-connect eval "fetch(`https://api.com/${path}`)"

# RIGHT — heredoc bypasses shell
agent-browser --auto-connect eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

### Rate Limiting

Add delays between rapid API calls:

```bash
agent-browser --auto-connect eval '...'  # Call 1
agent-browser --auto-connect wait 2000    # Wait 2s
agent-browser --auto-connect eval '...'  # Call 2
```

### Tab Contention (Agent vs Human)

You and the agent share the same Chrome — will you fight over tabs? **No.** Once agent-browser connects to a tab, it locks onto it. You can freely use other tabs.

Rules: don't close the agent's tab, don't navigate it manually, and avoid `agent-browser close` mid-workflow. Everything else is fine.

For heavy usage, consider a [dedicated Chrome profile](docs/gotchas.md#7-tab-contention-agent-vs-human).

### Security

CDP gives **full access** to your browser — all tabs, all cookies, all localStorage. Only use this on your own machine, and only log into sites you need for automation in the CDP-enabled Chrome session.

More details: [docs/gotchas.md](docs/gotchas.md)

## Skill vs MCP: Why Skill Wins

agent-browser offers two integration paths for Claude Code:

| Dimension | Skill (CLI) | MCP Server |
|-----------|-------------|------------|
| Idle token cost | ~150 tokens (description only) | **12,000-24,000 tokens** (40+ tool schemas, always loaded) |
| Per-action cost | ~50-200 tokens (bash one-liners) | ~500-2,000 (structured tool calls) |
| Command chaining | `&&` chains reduce round-trips | Each action = separate tool call |
| Pre-authorization | Built into SKILL.md frontmatter | Manual config, known bugs |
| Maturity | Official (Vercel-maintained) | Community wrapper (single author) |
| Vercel's recommendation | **Yes** (ships official SKILL.md) | No (closed multiple MCP PRs) |

**Bottom line:** MCP permanently burns 12-24K tokens even when you're not using the browser. The Skill approach costs near-zero when idle.

Deep dive: [docs/skill-vs-mcp.md](docs/skill-vs-mcp.md)

## Real-World Case Study

This guide comes from building an automated stock trading system with Claude Code. The system uses 8 skills orchestrating a complete daily workflow:

```
Pre-market analysis → Intraday trading → Post-market review
     ↓                      ↓                    ↓
  /stock-pre-market    /stock-trade         /stock-post-market
  (overnight data)     (DB + alerts +       (snapshot + screen
                        watchlist sync)      + next-day plan)
```

agent-browser + CDP handles the **watchlist synchronization** — calling the brokerage's authenticated API to add/remove stocks. Before agent-browser, this required either:
- Manual operation (defeats the purpose of automation)
- WebFetch with cookies (unreliable, 503 rate limiting, ~3,000 tokens per call)

With agent-browser: **217 tokens, 3.5 seconds, 100% reliable**.

## When NOT to Use This

- **Exploring unknown sites** — use MCP browser tools or computer use (you need vision)
- **Visual verification** — use `screenshot --annotate` (agent-browser supports this too)
- **CI/CD pipelines** — use headless mode with profiles, not CDP to a live browser
- **Multi-user systems** — CDP connects to one user's browser; use session management for multi-tenant

## Resources

- [agent-browser](https://github.com/vercel-labs/agent-browser) — The CLI tool
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) — The underlying protocol
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — Building reusable skills

## License

[MIT](LICENSE)
