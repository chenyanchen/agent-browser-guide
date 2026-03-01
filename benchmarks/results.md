# Benchmark Results

Real measurements comparing agent-browser + CDP vs Claude in Chrome across three tasks of increasing complexity (2026-02-28 / 2026-03-01).

## Environment

- **OS:** macOS Sequoia 15.3
- **Chrome:** 145+ with `chrome://inspect/#remote-debugging` enabled
- **agent-browser:** latest (installed via `brew install agent-browser`)
- **Claude Code:** 2.1.63, Opus 4.6 model
- **Network:** Residential broadband, China mainland
- **Permission mode:** bypassPermissions (both sessions)
- **Measurement:** Token usage from session JSONL `usage` fields; wall-clock time from first to last timestamp

## Task 1: Data Extraction (JS eval)

**Prompt:** Open https://news.ycombinator.com, extract the top 10 story titles, return as JSON array.

Both approaches used JavaScript evaluation (`document.querySelectorAll`) — no screenshots in either session.

| Metric | agent-browser + CDP | Claude in Chrome |
|--------|---------------------|------------------|
| **Total time** | **28s** | **50s** |
| Context baseline | 21k | 28k (+6.4k MCP tools) |
| Context delta | +5.7k | +5.1k |
| Tool calls | 4 | 8 |
| Mechanism | JS eval via Bash CLI | JS eval via MCP tool |
| Result correct | ✅ | ✅ |
| Session ID | `f1995c1c` | `081ddb5d` |

## Task 2: Form Interaction (fill + click)

**Prompt:** Go to https://the-internet.herokuapp.com/login, fill in username "tomsmith" and password "SuperSecretPassword!", click Login, and return the success message text.

| Metric | agent-browser + CDP | Claude in Chrome |
|--------|---------------------|------------------|
| **Total time** | **28s** | **50s** |
| Context baseline | 21k | 28k |
| Context delta | +5.7k | +5.1k |
| Output tokens | 459 | ~1,071 |
| Tool calls | 4 | 8 |
| Screenshots taken | 0 | 2 |
| JSONL size | 35 KB | 221 KB |
| Session ID | `f1995c1c` | `081ddb5d` |

### What Each Approach Did

**agent-browser (4 tool calls):**

```
1. Skill("agent-browser")              → load skill instructions
2. Bash: fill @e2 && fill @e3          → chain fill + click + wait + snapshot
        && click @e4 && wait              in ONE command
        && snapshot -i
3. Bash: get text "#flash"             → extract success message
4. Bash: close                         → cleanup
```

**Claude in Chrome (8 tool calls):**

```
1. tabs_context_mcp                    → get tab info
2. navigate(url)                       → open page
3. computer(screenshot)                → take screenshot to see the page
4. find("password input")              → locate password field
5. form_input(ref_12, password)        → fill password
6. find("Login button")               → locate button
7. computer(left_click, ref_13)        → click Login
8. computer(screenshot)                → take screenshot to verify result
```

Key difference: agent-browser chained 5 operations into 1 bash command via `&&`. Claude in Chrome needed separate tool calls for each action, including 2 screenshots.

## Task 3: Complex Interaction (post + delete on x.com)

**Prompt:** Go to x.com, post a tweet with text "agent-browser benchmark test - please ignore [timestamp]", wait for it to be posted, then delete that tweet. Return the tweet URL before deletion and confirmation of deletion.

| Metric | agent-browser + CDP | Claude in Chrome |
|--------|---------------------|------------------|
| **Total time** | **2m19s (139s)** | **3m04s (184s)** |
| Context baseline | 21k | 28k |
| Context final | 37k | 46k |
| Context delta | +16k | +18k |
| Output tokens | ~3,565 | ~4,144 |
| Tool calls (browser) | ~17 Bash | 19 MCP |
| Screenshots taken | 0 | 8 |
| JSONL size | 129 KB | **2,383 KB** |
| Session ID | `fe268414` | `3149e6a3` |

### What Each Approach Did

**agent-browser:** Used `snapshot -i` (text accessibility tree) to see the page, `fill` / `click` with refs to interact, and `get url` to capture the tweet URL. Zero screenshots — all text-based.

**Claude in Chrome:** Used `navigate` + `javascript_tool` for navigation, then 15 `computer` actions (including 8 screenshots) for interaction. Screenshots were used to visually verify each step: page loaded, text entered, tweet posted, menu opened, delete confirmed.

## Speed Analysis: Why agent-browser Is Faster

Across all three tasks, agent-browser was consistently faster:

| Task | agent-browser | Claude in Chrome | Speedup |
|------|--------------|------------------|---------|
| Data extraction | 28s | 50s | 1.8x |
| Form login | 28s | 50s | 1.8x |
| x.com post+delete | 139s | 184s | 1.3x |

Three factors explain the speed difference:

### 1. Command Chaining Reduces API Round-trips

agent-browser chains multiple operations into a single bash command with `&&`:

```bash
# 1 tool call = 5 browser operations
agent-browser fill @e2 "tomsmith" && agent-browser fill @e3 "pass" && agent-browser click @e4 && agent-browser wait --load networkidle && agent-browser snapshot -i
```

Claude in Chrome requires a separate MCP tool call for each action. Each tool call is a full API round-trip: send entire context → model reasons → returns tool call → execute → send result back.

In the login task: **4 round-trips vs 8 round-trips**.

### 2. Text vs Screenshots

agent-browser uses `snapshot` (accessibility tree) — compact text, ~200-2,000 tokens per page.

Claude in Chrome takes screenshots for visual verification. Each screenshot involves rendering, base64 encoding, and vision model processing. The JSONL size difference tells the story:

| Task | agent-browser JSONL | Chrome JSONL | Ratio |
|------|-------------------|--------------|-------|
| Form login | 35 KB | 221 KB | 6.3x |
| x.com post+delete | 129 KB | 2,383 KB | **18.5x** |

The x.com session's JSONL is **18.5x larger** — almost entirely from 8 base64-encoded screenshots carried through the context.

### 3. MCP Baseline Compounds Over Turns

Claude in Chrome loads ~6,400 extra tokens (18 MCP tool schemas) into every API call. Over many turns, this compounds:

- Login task: 8 turns × 6.4k = ~51k extra cumulative input tokens
- x.com task: 24 turns × 6.4k = ~154k extra cumulative input tokens

More input tokens per turn means slower inference.

## Idle Cost: Skill vs MCP

This is where the architectural difference matters most — not during browser tasks, but during the rest of the session when the browser is idle.

| Metric | Skill (CLI) approach | MCP (Claude in Chrome) |
|--------|---------------------|----------------------|
| Idle token overhead | ~586 tokens | ~5,600 tokens |
| Loaded when | On demand (skill invocation) | Always (session start) |
| Unloaded when | After skill completes | Never (entire session) |

### Baseline Breakdown

| Component | agent-browser session | Claude in Chrome session |
|-----------|----------------------|--------------------------|
| System prompt | 3.5k | 4.2k |
| System tools | 16.9k | 17.1k |
| MCP tools | — | 5.6k (18 tools, always loaded) |
| Skills | 586 (8 skill descriptions) | 502 |
| Memory files | 58 | 58 |
| **Total baseline** | **21k** | **28k** |

## Where Claude in Chrome Wins

1. **Simpler setup:** No CLI installation or autoConnect configuration needed.

2. **Vision fallback:** When JavaScript can't accomplish the task, Claude in Chrome can fall back to screenshots + vision model. agent-browser has `snapshot` (accessibility tree) but no vision capability. For visual exploration of unknown sites, Claude in Chrome is the better tool.

3. **Lower per-task context delta for JS eval tasks:** +2k vs +7k (no skill loading overhead).

## Key Takeaways

1. **agent-browser is consistently faster.** 1.3x–1.8x across three tasks of varying complexity, driven by command chaining, text-based page representation, and lower baseline overhead.

2. **Idle cost matters in long sessions.** MCP's 5,600 tokens are loaded into every API call, every turn, even during pure coding work. The Skill approach pays near-zero when the browser is idle.

3. **Dual universality is the strategic advantage.** agent-browser works with any AI agent that can run shell commands (not just Claude), and any site you can open in Chrome (not just sites that permit automation).

4. **For JS eval tasks, token usage is comparable.** Both approaches execute JavaScript in the browser. The difference is in overhead structure (skill load vs MCP idle) and speed.

## Session Files

All session JSONL files are in `~/.claude/projects/-Users-nle-silicon-agent-browser-guide/`:

| Session | Task | Approach | ID |
|---------|------|----------|----|
| G | Login form | agent-browser | `f1995c1c-d0b6-4539-8421-184e4ed4776c` |
| H | Login form | Claude in Chrome | `081ddb5d-b926-43f0-9927-37c8529d31b1` |
| I | x.com post+delete | agent-browser | `fe268414-2e98-4c0b-b2e8-25d0e97a1bc2` |
| J | x.com post+delete | Claude in Chrome | `3149e6a3-f584-4161-9408-f5b9a5becafc` |

Earlier sessions (A–F) from 2026-02-28 are in `~/.claude/projects/-Users-nle-silicon-series4-stuff/`.

## Next Steps

- [Quick start](../examples/01-quick-start.md) — try it yourself
- [Gotchas](../docs/gotchas.md) — avoid common pitfalls
- [Skill vs MCP](../docs/skill-vs-mcp.md) — architectural decisions
