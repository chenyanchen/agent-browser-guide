# Benchmark Results

Real measurements comparing agent-browser + CDP vs Claude in Chrome (2026-02-28).

## Environment

- **OS:** macOS Sequoia 15.3
- **Chrome:** 145+ with `chrome://inspect/#remote-debugging` enabled
- **agent-browser:** latest (installed via `brew install agent-browser`)
- **Claude Code:** 2.1.63, Opus 4.6 model
- **Network:** Residential broadband, China mainland
- **Permission mode:** bypassPermissions (both sessions)

## Task

**Open https://news.ycombinator.com, extract the top 10 story titles, return as JSON array.**

Both sessions ran back-to-back on the same machine with the same Chrome instance. Both approaches used JavaScript evaluation to extract data — no screenshots were involved in either session.

## Methodology

- Each approach ran in a fresh Claude Code session
- Token usage measured via `/context` command before and after the task
- Wall-clock time measured from prompt submission to final response
- Session IDs:
  - agent-browser: `e459ee27-bfee-4b8c-b56e-e47ecda1143f`
  - Claude in Chrome: `d41f39e3-daa6-4b8e-810e-8062822ed683`

## Results

### Task Comparison

| Metric | agent-browser + CDP | Claude in Chrome |
|--------|---------------------|------------------|
| Context baseline | 21k tokens | 28k tokens |
| Context after task | 28k tokens | 30k tokens |
| **Context delta** | **+7k** | **+2k** |
| Message tokens (before → after) | 8 → 8k | 8 → 3.9k |
| **Total time (prompt → answer)** | **26s** | **64s** |
| Browser operation time | ~11s | ~7.4s |
| Tool calls | 4 | 3 |
| Mechanism used | JS eval via Bash CLI | JS eval via MCP tool |
| Result correct | ✅ | ✅ |

### Why agent-browser's Context Delta Is Larger

agent-browser's +7k includes a one-time skill loading cost (~4.9k tokens for SKILL.md instructions injected into context). Subtracting that, the actual task messages were ~3.1k — slightly less than Chrome's 3.9k.

This cost is paid once per skill invocation. Multiple browser operations within one invocation share the same skill load.

### Baseline Breakdown

| Component | agent-browser session | Claude in Chrome session |
|-----------|----------------------|--------------------------|
| System prompt | 3.5k | 4.2k |
| System tools | 16.9k | 17.1k |
| MCP tools | — | 5.6k (18 tools, always loaded) |
| Skills | 586 (8 skill descriptions) | 502 |
| Memory files | 58 | 58 |
| **Total baseline** | **21k** | **28k** |

### Idle Cost: Skill vs MCP

This is where the architectural difference matters most — not during browser tasks, but during the rest of the session when the browser is idle.

| Metric | Skill (CLI) approach | MCP (Claude in Chrome) |
|--------|---------------------|----------------------|
| Idle token overhead | ~586 tokens | ~5,600 tokens |
| Loaded when | On demand (skill invocation) | Always (session start) |
| Unloaded when | After skill completes | Never (entire session) |

In a typical 60-minute coding session with 2 minutes of browser use, MCP loads 5,600 tokens into every API call for the entire session. The Skill approach loads ~586 tokens idle, plus ~4.9k only during the browser task.

## Analysis

### What Both Approaches Did

Surprisingly, both approaches used the **same underlying mechanism**: JavaScript evaluation in the browser's page context.

- **agent-browser:** `agent-browser eval --stdin` → executed `document.querySelectorAll('.titleline > a')`
- **Claude in Chrome:** `javascript_tool` MCP tool → executed `document.querySelectorAll('.titleline > a')`

No screenshots, no vision model, no base64 encoding in either session. Claude in Chrome uses vision model as a fallback when it can't operate via DOM/JavaScript — but for this structured data extraction task, JavaScript was sufficient.

### Where agent-browser Wins

1. **Speed (2.5x faster):** 26s vs 64s. The browser operations themselves were comparable (~11s vs ~7.4s), but agent-browser had less API roundtrip overhead.

2. **Idle cost (10x lower):** ~586 tokens vs ~5,600 tokens when not using the browser. This compounds over long coding sessions.

3. **Agent-agnostic:** Any AI agent that can run bash (Claude Code, Codex, Cursor, Windsurf, Copilot) can use agent-browser. Claude in Chrome only works with Claude.

### Where Claude in Chrome Wins

1. **Lower per-task context delta:** +2k vs +7k. No skill loading overhead — MCP tools are always available.

2. **Simpler setup:** No CLI installation or autoConnect configuration needed.

3. **Vision fallback:** When JavaScript can't accomplish the task, Claude in Chrome can fall back to screenshots + vision model. agent-browser has `snapshot` (accessibility tree) but no vision capability.

## Key Takeaways

1. **For JS eval tasks, token usage is comparable.** Both approaches execute JavaScript in the browser. The difference is in overhead structure (skill load vs MCP idle).

2. **Speed is the clearest advantage.** agent-browser completes tasks 2.5x faster.

3. **Idle cost matters in long sessions.** MCP's 5,600 tokens are loaded into every API call, every turn, even during pure coding work. The Skill approach pays near-zero when the browser is idle.

4. **Portability is the strategic advantage.** agent-browser works with any AI agent that can run shell commands. This is a universal solution, not tied to any specific AI vendor.

## Next Steps

- [Quick start](../examples/01-quick-start.md) — try it yourself
- [Gotchas](../docs/gotchas.md) — avoid common pitfalls
- [Skill vs MCP](../docs/skill-vs-mcp.md) — architectural decisions
