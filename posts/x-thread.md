# X/Twitter Thread: agent-browser + CDP

### Tweet 1

I reduced my AI agent's browser automation tokens by 95%.

Not by switching models. Not by prompt engineering.

By connecting Claude to my real Chrome instead of spinning up a headless one.

Here's the architecture change (and real numbers):

---

### Tweet 2

The problem with "Chrome in Claude" (computer use / MCP browser):

1. Screenshot-heavy: render -> screenshot -> base64 -> vision. One click = 2K-8K tokens
2. No login state: fresh browser = no cookies
3. Sites blocked: wise.com, reddit, WeChat refuse automated browsers

---

### Tweet 3

The fix is one line:

```
agent-browser --auto-connect eval 'document.title'
```

This connects to your running Chrome via CDP (Chrome DevTools Protocol). Your cookies. Your sessions. Your extensions. Everything just works.

---

### Tweet 4

Real benchmark from my stock trading system:

Stock API: 57 tokens vs 4,000-6,000 (70-105x)
Auth API: 217 vs 3,000 (14x)
Financial page: 1,777 vs 8,000-15,000 (4.5-8x)
Search page: 41 vs 5,000-8,000 (122-195x)

Total: 2,092 vs 20,000-32,000. ~15x cheaper.

---

### Tweet 5

I built a Claude Code skill chain around it:

/stock-trade (records in PostgreSQL)
  -> /stock-watchlist (syncs to brokerage)
    -> agent-browser (calls authenticated API)
      -> Chrome CDP -> Broker API (with my cookies)

217 tokens. 3.5 seconds. 100% reliable.

---

### Tweet 6

Biggest gotcha: CORS.

If your API is on t.example.com, you MUST navigate there first. Navigating to www.example.com then fetching from t.example.com = silent failure.

Cost me 2 hours. Don't repeat my mistake.

---

### Tweet 7

When NOT to use this:

- Exploring unknown sites (use MCP / computer use -- you need vision)
- CI/CD pipelines (use headless mode, not CDP to a live browser)
- Multi-user systems (CDP = one user's browser)

It's not about replacing MCP. It's about picking the right tool.

---

### Tweet 8

Full guide with benchmarks, gotchas, and Claude Code skill templates:

github.com/chenyanchen/agent-browser-guide

MCP idle cost: 12-24K tokens (always loaded).
Skill idle cost: ~150 tokens.

Choose accordingly.
