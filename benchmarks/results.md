# Benchmark Results

## Environment

- **OS:** macOS Sequoia 15.3
- **Chrome:** 145.0.6422.60
- **agent-browser:** 0.15.1
- **Claude Code:** latest (Opus 4 model)
- **Network:** Residential broadband, ~50ms latency to target APIs

## Methodology

Four tasks were tested, each run 5 times. We measured:

- **Wall time:** From command invocation to result returned
- **Response chars:** Raw character count of the returned data
- **Estimated tokens:** Characters / 4 (rough approximation for token counting)

Each task was also attempted via WebFetch (Claude Code's built-in URL fetcher) and Chrome in Claude (desktop app's browser integration) where applicable.

## Tasks

### Task 1: Stock API (Authenticated)

Fetch portfolio data from a trading platform's internal API. Requires active session cookies.

### Task 2: Financial Data Page

Scrape key financial metrics from a stock detail page (public, no auth).

### Task 3: Search Results Page

Navigate to a financial search engine, extract top 10 results.

### Task 4: WeChat Article

Read a WeChat public article (requires specific cookies/headers, blocks most scrapers).

## Raw Results: agent-browser

| Task | Run 1 | Run 2 | Run 3 | Run 4 | Run 5 | Median |
|---|---|---|---|---|---|---|
| **Task 1: Stock API** | | | | | | |
| Time (ms) | 1,240 | 1,180 | 1,310 | 1,150 | 1,220 | 1,220 |
| Chars | 2,847 | 2,851 | 2,849 | 2,850 | 2,848 | 2,849 |
| **Task 2: Financial Page** | | | | | | |
| Time (ms) | 2,310 | 2,540 | 2,180 | 2,450 | 2,290 | 2,310 |
| Chars | 1,523 | 1,519 | 1,525 | 1,521 | 1,524 | 1,523 |
| **Task 3: Search Results** | | | | | | |
| Time (ms) | 3,120 | 2,980 | 3,240 | 3,050 | 3,180 | 3,120 |
| Chars | 3,412 | 3,408 | 3,415 | 3,410 | 3,411 | 3,411 |
| **Task 4: WeChat Article** | | | | | | |
| Time (ms) | 1,890 | 1,760 | 1,920 | 1,810 | 1,850 | 1,850 |
| Chars | 5,234 | 5,230 | 5,238 | 5,232 | 5,236 | 5,234 |

## Comparison Baselines

### WebFetch

| Task | Result | Notes |
|---|---|---|
| Task 1: Stock API | **Failed** | 401 Unauthorized -- no browser cookies |
| Task 2: Financial Page | 8,450 chars | Returns full HTML-to-markdown conversion |
| Task 3: Search Results | 12,200 chars | Includes navigation, ads, boilerplate |
| Task 4: WeChat Article | **Failed** | Blocked by WeChat anti-scraping |

### Chrome in Claude (Desktop App)

| Task | Result | Notes |
|---|---|---|
| Task 1: Stock API | **Not accessible** | Cannot navigate to internal API endpoints |
| Task 2: Financial Page | ~15,000 chars | Full page screenshot + text extraction |
| Task 3: Search Results | ~18,000 chars | Full page with all UI elements |
| Task 4: WeChat Article | **Failed** | wise.com, reddit, WeChat are blocked |

## Token Efficiency Analysis

| Task | agent-browser (tokens) | WebFetch (tokens) | Chrome in Claude (tokens) | agent-browser advantage |
|---|---|---|---|---|
| Task 1 | ~712 | N/A (failed) | N/A (failed) | Only option that works |
| Task 2 | ~381 | ~2,113 | ~3,750 | 5.5x fewer vs WebFetch |
| Task 3 | ~853 | ~3,050 | ~4,500 | 3.6x fewer vs WebFetch |
| Task 4 | ~1,309 | N/A (failed) | N/A (failed) | Only option that works |

### Why agent-browser Is More Token-Efficient

1. **Targeted extraction:** JavaScript eval returns only the data you ask for, not the entire page.
2. **No markup overhead:** WebFetch converts HTML to markdown, keeping navigation, headers, footers. agent-browser returns raw data.
3. **JSON over prose:** API responses are structured JSON. WebFetch returns prose summaries.

## Access Comparison

Sites that require authenticated access or block standard scrapers:

| Site/Resource | agent-browser | WebFetch | Chrome in Claude |
|---|---|---|---|
| Internal APIs (with SSO) | Works | Blocked | Blocked |
| WeChat articles | Works | Blocked | Blocked |
| Reddit (logged in) | Works | Limited | Blocked |
| wise.com | Works | Blocked | Blocked |
| Banking portals | Works | Blocked | Blocked |
| Public websites | Works | Works | Works |

## Key Takeaways

1. **agent-browser is the only option for authenticated APIs.** Neither WebFetch nor Chrome in Claude can use your browser's session cookies.

2. **3-6x token savings on public pages.** Targeted JavaScript extraction returns only what you need, avoiding the boilerplate that WebFetch includes.

3. **Consistent performance.** Median times are stable across runs. The browser is a reliable execution environment.

4. **Access to walled gardens.** WeChat, Reddit, banking portals, and internal tools are only accessible through the real browser session.

## Next Steps

- [Quick start](../examples/01-quick-start.md) -- try it yourself
- [Gotchas](../docs/gotchas.md) -- avoid common pitfalls
- [Skill vs MCP](../docs/skill-vs-mcp.md) -- architectural decisions
