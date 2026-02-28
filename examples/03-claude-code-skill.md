# Building a Claude Code Skill with agent-browser

Skills are reusable capabilities that Claude Code loads on demand. This example shows how to wrap agent-browser into a skill that any conversation can invoke.

## How Skills Work

- A skill is a `SKILL.md` file inside `.claude/skills/` in your project
- It gets loaded when invoked via the `Skill` tool or a `/slash-command`
- Skills declare which tools they need (Bash, Read, Write, etc.)
- Skills are **not** loaded until called -- zero idle cost

## Full SKILL.md Template

Create `.claude/skills/stock-watchlist/SKILL.md`:

```yaml
---
name: stock-watchlist
description: Sync watchlist to trading platform via browser automation
allowed-tools:
  - Bash(agent-browser:*)
  - Bash(echo:*)
  - Read
---
```

```markdown
# Stock Watchlist Sync

Syncs the screened stock list to the trading platform's browser-based watchlist.

## Precondition Check

Before any browser operation, verify CDP is available:

\```bash
agent-browser list-pages > /dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "ERROR: Chrome CDP not available."
  echo "Open chrome://inspect/#remote-debugging in Chrome."
  exit 1
fi
\```

## Workflow

### 1. Navigate to the trading platform

\```bash
agent-browser navigate 'https://trade.example.com/watchlist'
\```

### 2. Read the current watchlist

\```bash
agent-browser eval "$(cat <<'JSEOF'
(async () => {
  const resp = await fetch('/api/watchlist/list', {
    credentials: 'include'
  });
  if (!resp.ok) return JSON.stringify({ error: resp.status });
  return JSON.stringify(await resp.json());
})();
JSEOF
)"
\```

### 3. Diff and update

Compare the API response with the desired stock list. Add missing stocks, remove extras.

\```bash
agent-browser eval "$(cat <<'JSEOF'
(async () => {
  const resp = await fetch('/api/watchlist/add', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({ symbols: ['600519', '000858'] })
  });
  return JSON.stringify({ ok: resp.ok, status: resp.status });
})();
JSEOF
)"
\```

### 4. Verify

Re-read the watchlist and confirm the changes took effect.

## Error Handling

- **CDP unavailable:** Print setup instructions, exit
- **401 Unauthorized:** Tell user to log in via Chrome, exit
- **503 Rate limited:** Wait 2s, retry up to 3 times
- **Network error:** Report and exit
```

## The `allowed-tools` Pattern

```yaml
allowed-tools:
  - Bash(agent-browser:*)
```

This grants the skill permission to run any `agent-browser` subcommand (`navigate`, `eval`, `list-pages`, etc.) without prompting the user each time. The wildcard `*` covers all subcommands.

You can also be more restrictive:

```yaml
allowed-tools:
  - Bash(agent-browser:navigate *)
  - Bash(agent-browser:eval *)
  - Bash(agent-browser:list-pages)
```

## Precondition Check Pattern

Always check CDP availability before doing browser work:

```bash
if ! agent-browser list-pages > /dev/null 2>&1; then
  echo "Chrome CDP not available. Enable at chrome://inspect/#remote-debugging"
  exit 1
fi
```

This prevents confusing errors mid-workflow when Chrome isn't ready.

## How Skills Get Invoked

There are two ways users trigger a skill:

1. **Slash command:** User types `/stock-watchlist sync 600519 000858`
2. **Skill tool:** Claude Code's orchestrator calls the Skill tool internally, e.g., when `/stock-trade` chains into `/stock-watchlist`

Both load the SKILL.md into context and execute its instructions.

## Navigate, Action, Error Handling Flow

Every agent-browser skill follows the same pattern:

```
1. Precondition check  (is CDP available?)
        |
        v
2. Navigate             (go to correct subdomain)
        |
        v
3. Action               (eval fetch / DOM manipulation)
        |
        v
4. Error check          (401? 503? CORS? network?)
       / \
      /   \
   OK      Error
   |         |
   v         v
5. Return   Retry or report
```

## Next Steps

- [Skill chaining](04-skill-chaining.md) -- how one skill calls another
- [Skill vs MCP comparison](../docs/skill-vs-mcp.md) -- why skills over MCP servers
