# X/Twitter Thread: agent-browser + CDP

### Tweet 1

I made my AI agent's browser automation 2.5x faster — and agent-agnostic.

Not a new model. Not prompt engineering. Just connecting to my real Chrome via CDP instead of going through Claude in Chrome.

Here's what I measured (and what surprised me): 🧵

---

### Tweet 2

I expected a massive token gap. I ran a controlled benchmark: same task (extract HN top 10 titles), same machine, back-to-back.

agent-browser + CDP: 26s, +7k context tokens
Claude in Chrome: 64s, +2k context tokens

Wait — Claude in Chrome used FEWER tokens?

---

### Tweet 3

Yes. Because both approaches used the same mechanism: JavaScript eval. No screenshots in either session.

Claude in Chrome's `javascript_tool` ≈ agent-browser's `eval`. For JS-capable tasks, token usage is comparable.

---

### Tweet 4

So where's the win?

1. Idle cost: Skill loads ~586 tokens idle. MCP loads ~5,600 tokens EVERY turn, even when you're just writing code. 10x difference.
2. Any agent: agent-browser is a CLI. Works with Codex, Cursor, Windsurf, Copilot — not just Claude. No vendor lock-in.
3. Any site: Connects to YOUR Chrome, so any site you can open, the agent can access. No more "this site blocks automation."
4. Speed: 2.5x faster in our benchmark (26s vs 64s).

---

### Tweet 5

I built a Claude Code skill chain around it:

/stock-trade (records in PostgreSQL)
  -> /stock-watchlist (syncs to brokerage)
    -> agent-browser (calls authenticated API)
      -> Chrome CDP -> Broker API (with my cookies)

26s. Any agent that can run bash can do this.

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

The honest finding: tokens are comparable for JS eval tasks. The real wins are idle cost (10x lower) and dual universality — any agent, any site.
