# Connect AI Agents to Your Real Chrome via CDP

> Use [agent-browser](https://github.com/vercel-labs/agent-browser) + Chrome DevTools Protocol to give AI agents access to your browser's cookies, login sessions, and extensions — at 10-15x fewer tokens than screenshot-based approaches.

[中文版](README.zh-CN.md)

---

## Table of Contents

| I want to... | Jump to |
|---|---|
| **Set up now** | [Quick Start](#quick-start) |
| **Understand the problem first** | [The Problem](#the-problem) → [Benchmarks](#benchmarks) |
| **Integrate into my workflow** | [Integration Patterns](#integration-patterns) → [Gotchas](#gotchas) |
| **Compare Skill vs MCP** | [Skill vs MCP](#skill-vs-mcp) |
| **See a real-world system** | [Case Study](#real-world-case-study) |
| **Look up a term** | [Glossary](#glossary) |

---

## Quick Start

### What

Connect agent-browser to your running Chrome via CDP, so AI agents inherit your browser's cookies, login sessions, and extensions.

### Why

By default, agent-browser launches a **new headless browser** — no cookies, no login state. That's fine for public pages.

But if your agent needs to:
- **Call authenticated APIs** using your browser's cookies and SSO sessions
- **Access restricted sites** that block automated browsers (WeChat, Reddit, wise.com)
- **Interact with internal tools** behind corporate authentication

...you need to connect to your **real Chrome**. That's what this setup does.

> **Security note:** CDP grants full access to your browser — all tabs, cookies, and localStorage. Only enable on your personal machine. [Details →](#security)

### How

#### 1. Enable Chrome Remote Debugging

In your Chrome address bar, go to:

```
chrome://inspect/#remote-debugging
```

Check **"Allow remote debugging for this browser instance"**. No restart needed. (Chrome 145+)

#### 2. Install agent-browser

```bash
# macOS
brew install agent-browser

# or via npm
npm install -g agent-browser && agent-browser install
```

#### 3. Configure Auto-Connect

```bash
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{
  "autoConnect": true
}
EOF
```

This tells agent-browser to connect to your running Chrome instead of launching a headless instance.

#### 4. Verify

```bash
agent-browser get url
# Should print the URL of your current Chrome tab
```

You're done. Your AI agent can now control your Chrome.

## The Problem

Claude in Chrome connects to your real browser — it has your cookies and login sessions. But the screenshot-based mechanism creates two problems:

| Problem | Impact |
|---------|--------|
| **Screenshot-heavy** | Each action: render → screenshot → base64 → vision model → reason. One click costs ~2,000-8,000 tokens, takes 8-15 seconds. |
| **Site restrictions** | Some sites still block or behave differently under Claude in Chrome's automation: wise.com, reddit.com, mp.weixin.qq.com, and more. |

And when things go wrong? The agent takes more screenshots to debug, doubling the token cost and time.

## The Solution

```bash
agent-browser eval 'document.title'
```

One command connects to your running Chrome via CDP — inheriting all cookies and login sessions.

```
AI Agent (Claude Code / Cursor / Codex / Windsurf / ...)
  └── bash: agent-browser <command>
        └── Chrome DevTools Protocol (CDP)
              └── Your Chrome (with all cookies, extensions, login state)
```

**This is agent-agnostic.** agent-browser is a CLI tool — any AI agent that can execute shell commands can use it. Claude Code, Codex, Cursor, Windsurf, Copilot — the setup is the same. No vendor lock-in, no SDK integration. If your agent can run `bash`, it can control your browser.

## Benchmarks

> **Environment:** macOS Sequoia 15.3, Chrome 145+, residential broadband (China mainland)
> **Date:** 2026-02-28 | **Sample:** 4 tasks from a stock trading system, single run each
> **Token estimate:** CLI output chars ÷ 4. Claude in Chrome baselines from prior usage sessions
> **Note:** Results vary with page content, network, and Chrome version. [Full methodology →](benchmarks/results.md)

### Per-Task Comparison

| Task | agent-browser Time | agent-browser Tokens | Claude in Chrome Tokens | Reduction |
|------|-------------------|---------------------|------------------------|-----------|
| Stock price JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| Authenticated API call | 3.5s | **217** | 3,000 | 14x |
| Financial page snapshot | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| Search page snapshot | 1.8s | **41** | 5,000-8,000 | 122-195x |
| **Total (4 tasks)** | **10.8s** | **2,092** | **20,000-32,000** | **10-15x** |

### Overall Comparison

| Metric | Claude in Chrome | agent-browser + CDP |
|--------|-----------------|-------------------|
| Tokens per action | ~2,000-8,000 | **~50-1,800** |
| Speed per action | ~8-15s (screenshots + vision) | **~2-4s** |
| When stuck | More screenshots, 2x cost | Returns error text, minimal overhead |
| Site access (in our tests) | wise.com, reddit, WeChat restricted | All tested sites accessible |
| Cookies / login state | ✅ Yes (your real Chrome) | ✅ Yes (your real Chrome via CDP) |
| Mechanism | Screenshot → vision model | JavaScript eval / accessibility tree |
| Idle token overhead | MCP: 12-24K tokens permanent | Skill: ~150 tokens |

## Integration Patterns

### Pattern A: Fetch JSON APIs (eval + fetch)

Use your browser's cookies to call authenticated APIs directly:

```bash
# Navigate to the API's domain first (for same-origin cookies)
agent-browser open "https://api.example.com/"
agent-browser wait 2000

# Call the API — cookies are sent automatically
agent-browser eval --stdin <<'EOF'
fetch("https://api.example.com/user/watchlist")
  .then(r => r.json())
  .then(d => JSON.stringify(d))
EOF
```

**Output:** Pure JSON. ~100-300 tokens. No HTML, no markdown, no noise.

### Pattern B: Navigate + Interact (snapshot + refs)

For pages that need clicking, filling forms, or reading content:

```bash
agent-browser open "https://example.com/dashboard"
agent-browser wait --load networkidle
agent-browser snapshot -i
# Output:
# - button "Sign Out" [ref=e1]
# - link "Settings" [ref=e2]
# - textbox "Search" [ref=e3]

agent-browser fill @e3 "query"
agent-browser press Enter
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
    → agent-browser(calls authenticated API)
      → Chrome CDP → Broker API (with user's cookies)
```

See [examples/04-skill-chaining.md](examples/04-skill-chaining.md) for the full architecture.

## Gotchas

### CORS: Navigate to the Right Subdomain

APIs only receive cookies from the **same origin**. If the API is on `t.example.com`, you must navigate there first:

```bash
# WRONG — CORS failure
agent-browser open "https://www.example.com"
agent-browser eval 'fetch("https://t.example.com/api/data")'
# TypeError: Failed to fetch

# RIGHT — same origin
agent-browser open "https://t.example.com/"
agent-browser eval 'fetch("https://t.example.com/api/data")'
# Works!
```

### Chrome Restart

After restarting Chrome, you need to re-enable remote debugging at `chrome://inspect/#remote-debugging`. This is by design — Chrome doesn't persist CDP access for security.

### Shell Escaping

Use `<<'EOF'` (single-quoted heredoc) to avoid shell interpretation of your JavaScript:

```bash
# WRONG — shell expands $, !, backticks
agent-browser eval "fetch(`https://api.com/${path}`)"

# RIGHT — heredoc bypasses shell
agent-browser eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

### Rate Limiting

Add delays between rapid API calls:

```bash
agent-browser eval '...'  # Call 1
agent-browser wait 2000    # Wait 2s
agent-browser eval '...'  # Call 2
```

### Tab Contention (Agent vs Human)

You and the agent share the same Chrome — will you fight over tabs? In practice, no. Once agent-browser connects to a tab, it locks onto it. You can freely use other tabs.

Rules: don't close the agent's tab, don't navigate it manually, and avoid `agent-browser close` mid-workflow.

For heavy usage, consider a [dedicated Chrome profile](docs/gotchas.md#7-tab-contention-agent-vs-human).

### Security

CDP gives **full access** to your browser — all tabs, all cookies, all localStorage. Only use this on your own machine, and only log into sites you need for automation in the CDP-enabled Chrome session.

More details: [docs/gotchas.md](docs/gotchas.md)

## Skill vs MCP

agent-browser offers two integration paths for Claude Code:

| Dimension | Skill (CLI) | MCP Server |
|-----------|-------------|------------|
| Idle token cost | ~150 tokens (description only) | **12,000-24,000 tokens** (40+ tool schemas, always loaded) |
| Per-action cost | ~50-200 tokens (bash one-liners) | ~500-2,000 (structured tool calls) |
| Command chaining | `&&` chains reduce round-trips | Each action = separate tool call |
| Pre-authorization | Built into SKILL.md frontmatter | Manual config, known bugs |
| Maturity | Official (Vercel-maintained) | Community wrapper (single author) |
| Vercel's recommendation | **Yes** (ships official SKILL.md) | No (closed multiple MCP PRs) |

**In our experience:** MCP permanently consumes 12-24K tokens even when the browser is idle. The Skill approach costs near-zero when idle. For intermittent browser use (which is most development sessions), Skill is the more token-efficient choice.

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

With agent-browser (as of 2026-02-28): **217 tokens, 3.5 seconds, consistent across 20+ daily runs**.

## When NOT to Use This

- **Exploring unknown sites** — use MCP browser tools or computer use (you need vision)
- **Visual verification** — use `screenshot --annotate` (agent-browser supports this too)
- **CI/CD pipelines** — use headless mode with profiles, not CDP to a live browser
- **Multi-user systems** — CDP connects to one user's browser; use session management for multi-tenant

## Glossary

| Term | Definition |
|------|-----------|
| **CDP** | Chrome DevTools Protocol — the same protocol Chrome DevTools (F12) uses internally. Enables external programs to control a running Chrome instance. |
| **MCP** | Model Context Protocol — Anthropic's protocol for connecting AI models to external tools. MCP servers register tool schemas that are loaded into every API call. |
| **Skill** | A Claude Code feature: a markdown file (SKILL.md) loaded on demand, declaring instructions and tool permissions. Zero token cost when idle. |
| **WebFetch** | Claude Code's built-in tool for fetching web pages. Converts HTML to markdown and summarizes via a fast model. |
| **snapshot** | agent-browser command that returns a compact accessibility tree of the page — structured text, not a screenshot. |
| **eval** | agent-browser command that executes JavaScript in the browser's page context, like running code in the DevTools console. |
| **Headless browser** | A browser running without a visible window. Used by automation tools (Puppeteer, Playwright). No access to the user's cookies or login sessions unless explicitly configured. |

## Platform Support

| Platform | Status | Notes |
|----------|--------|-------|
| macOS (Homebrew) | ✅ Verified | `brew install agent-browser` |
| macOS (npm) | ✅ Verified | `npm install -g agent-browser && agent-browser install` |
| Linux (npm) | Should work | Not yet tested by the author |
| Windows (npm) | Should work | Not yet tested by the author |
| Chrome 145+ | ✅ Required | For no-restart CDP via `chrome://inspect` |
| Chrome < 145 | Supported | Requires `--remote-debugging-port` flag. [Setup →](docs/setup.md#older-chrome--145-or-custom-port) |
| Chromium-based (Edge, Brave) | Should work | Via CDP, untested |

## Resources

- [agent-browser](https://github.com/vercel-labs/agent-browser) — The CLI tool
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) — The underlying protocol
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — Building reusable skills

## License

[MIT](LICENSE)
