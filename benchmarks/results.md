# Benchmark Results

Real measurements from a stock trading automation system (2026-02-28).

## Environment

- **OS:** macOS Sequoia 15.3
- **Chrome:** 145+ with `chrome://inspect/#remote-debugging` enabled
- **agent-browser:** latest (installed via `brew install agent-browser`)
- **Claude Code:** Opus 4 model
- **Network:** Residential broadband, China mainland

## Methodology

Four representative tasks from the trading system. Each task was measured via agent-browser (CLI output character count → token estimate) and compared against Chrome in Claude / WebFetch baselines from prior usage.

- **agent-browser tokens:** Raw character count of CLI output / 4 (conservative estimate)
- **Chrome in Claude tokens:** Measured from actual Claude computer use sessions (screenshot + vision model round-trips)
- **WebFetch tokens:** Measured from actual WebFetch tool calls (HTML-to-markdown conversion)

## Tasks

### Task 1: Stock Price JSON API

Fetch real-time stock price data from a public financial API endpoint. Returns structured JSON.

- **agent-browser approach:** `eval` with `fetch()`, returns raw JSON
- **Chrome in Claude approach:** Navigate to page, screenshot, vision model extracts price

### Task 2: Authenticated API Call (Watchlist Sync)

Call the brokerage's (10jqka.com.cn) watchlist API to add/remove stocks. Requires active session cookies.

- **agent-browser approach:** Navigate to `t.10jqka.com.cn`, `eval` with `fetch()`, cookies attached automatically
- **Chrome in Claude approach:** Cannot authenticate (fresh browser, no cookies)
- **WebFetch approach:** Unreliable, ~70% success rate, 503 rate limiting

### Task 3: Financial Page Snapshot

Extract key financial metrics from a stock detail page (earnings, PE, revenue growth, etc.).

- **agent-browser approach:** `snapshot` command returns compact accessibility tree
- **Chrome in Claude approach:** Full page screenshot + vision model extraction
- **WebFetch approach:** HTML-to-markdown conversion, includes navigation/boilerplate

### Task 4: Search Results Page

Query a financial search engine (iwencai.com) and extract top results.

- **agent-browser approach:** `snapshot` returns structured accessibility tree of results
- **Chrome in Claude approach:** Full page screenshot + vision model
- **WebFetch approach:** Full HTML-to-markdown with ads, navigation, boilerplate

## Results

### Per-Task Comparison

| Task | agent-browser Time | agent-browser Tokens | Chrome in Claude Tokens | Reduction |
|------|-------------------|---------------------|------------------------|-----------|
| Stock price JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| Authenticated API call | 3.5s | **217** | 3,000 (WebFetch baseline) | 14x |
| Financial page snapshot | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| Search results page | 1.8s | **41** | 5,000-8,000 | 122-195x |
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

## Analysis

### Why the Token Gap Is So Large

1. **JSON API (57 tokens):** agent-browser returns raw JSON from `fetch()`. Chrome in Claude must render a page, screenshot it, base64-encode it, send to vision model, then extract text. 4,000-6,000 tokens for 200 bytes of data.

2. **Search results (41 tokens):** Search result pages have highly structured content (title, URL, snippet) that maps perfectly to accessibility tree nodes. The `snapshot` command captures this structure with near-zero overhead.

3. **Financial page (1,777 tokens):** This is the narrowest gap because you're genuinely fetching substantial page content either way. But the accessibility tree is still 4.5-8x more compact than a screenshot-based approach.

4. **Authenticated API (217 tokens):** Chrome in Claude simply cannot do this — it has no cookies. The WebFetch baseline (3,000 tokens) was unreliable. agent-browser is the only approach that works consistently.

### Access Comparison

| Site/Resource | agent-browser | WebFetch | Chrome in Claude |
|---|---|---|---|
| Public JSON APIs | Works | Works | Works (but 70-105x more tokens) |
| Authenticated APIs (SSO/cookies) | **Works** | Unreliable | **Blocked** (no cookies) |
| WeChat articles | **Works** | Blocked | Blocked |
| Reddit (logged in) | **Works** | Limited | Blocked |
| wise.com | **Works** | Blocked | Blocked |
| Chinese financial sites (10jqka) | **Works** | Rate limited | Intermittent |

## Key Takeaways

1. **10-15x overall token reduction.** Across 4 representative tasks, agent-browser used 2,092 tokens vs 20,000-32,000 for Chrome in Claude.

2. **agent-browser is the only option for authenticated APIs.** Neither WebFetch nor Chrome in Claude can reliably use your browser's session cookies.

3. **Speed compounds.** A 5-action workflow: Chrome in Claude = 40-75 seconds, agent-browser = 10-20 seconds.

4. **No debugging tax.** When Chrome in Claude gets confused, it takes more screenshots to debug, doubling token cost. agent-browser errors are plain text — cheap to process.

## Next Steps

- [Quick start](../examples/01-quick-start.md) — try it yourself
- [Gotchas](../docs/gotchas.md) — avoid common pitfalls
- [Skill vs MCP](../docs/skill-vs-mcp.md) — architectural decisions
