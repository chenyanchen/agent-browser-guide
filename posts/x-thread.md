# X/Twitter Thread: agent-browser + CDP

### Tweet 1

I benchmarked my AI agent's browser automation: agent-browser + CDP vs Claude in Chrome.

3 tasks. Real measurements. agent-browser was 1.3x–1.8x faster every time — and works with ANY agent.

Here's what I found (and WHY it's faster): 🧵

---

### Tweet 2

Three tasks, back-to-back on the same machine:

| Task | agent-browser | Chrome | Speedup |
|------|--------------|--------|---------|
| Data extraction | 28s | 50s | 1.8x |
| Form login | 28s | 50s | 1.8x |
| x.com post+delete | 2m19s | 3m04s | 1.3x |

Token usage per task? Comparable. Both use JS eval under the hood.

---

### Tweet 3

So WHY is agent-browser faster? Three reasons:

1⃣ Command chaining: `fill && click && wait && snapshot` = 1 tool call. Chrome MCP needs separate calls. Login: 4 vs 8 API round-trips.

2⃣ Text vs screenshots: agent-browser uses accessibility tree (~2k tokens). Chrome takes screenshots. x.com JSONL: 129 KB vs 2,383 KB (18.5x).

---

### Tweet 4

3⃣ Lower baseline compounds: Chrome MCP loads ~6.4k extra tokens EVERY turn. Over 24 turns (x.com task) = ~154k extra cumulative input.

Plus the always-on idle cost:
- Skill: ~586 tokens idle
- MCP: ~5,600 tokens EVERY turn, even when just writing code

---

### Tweet 5

And it's universal:

- Any agent: CLI tool. Works with Codex, Cursor, Windsurf, Copilot — not just Claude. No vendor lock-in.
- Any site: Connects to YOUR Chrome. Any site you can open, the agent can access. No "this site blocks automation."

I built a skill chain: /stock-trade → /stock-watchlist → agent-browser → Chrome CDP → Broker API.

---

### Tweet 6

Biggest gotcha: CORS.

If your API is on t.example.com, you MUST navigate there first. Navigating to www.example.com then fetching from t.example.com = silent failure.

Cost me 2 hours. Don't repeat my mistake.

---

### Tweet 7

When NOT to use this:

- Exploring unknown sites (Claude in Chrome has vision fallback — you need it)
- CI/CD pipelines (use headless mode, not CDP to a live browser)
- Multi-user systems (CDP = one user's browser)

Both tools have their place.

---

### Tweet 8

Full guide with real benchmarks, gotchas, and skill templates:

github.com/chenyanchen/agent-browser-guide

The honest finding: tokens are comparable for JS eval tasks. The real wins are speed (1.3x–1.8x from command chaining + text vs screenshots), idle cost (10x lower), and dual universality — any agent, any site.
