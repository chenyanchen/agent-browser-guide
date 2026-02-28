# Multi-Skill Orchestration with agent-browser

Real workflows don't live in a single skill. This example shows how multiple skills chain together, using a stock trading system as the concrete case.

## Architecture

```
User: "Buy 200 shares of 600519 at 1580"
          |
          v
  /stock-trade (orchestrator)
          |
          +---> PostgreSQL: INSERT INTO stock_portfolio_holdings ...
          |     PostgreSQL: INSERT INTO stock_portfolio_transactions ...
          |
          +---> /stock-watchlist (chained skill)
                    |
                    +---> agent-browser: navigate to trading platform
                    +---> agent-browser: eval fetch('/api/watchlist/sync')
                    +---> agent-browser: verify changes
                    |
                    +---> Return result to /stock-trade
          |
          v
  /stock-trade: Update daily analysis file
          |
          v
  Done. User sees confirmation.
```

## The Chain in Detail

### Skill 1: /stock-trade

The entry point. Handles the business logic:

```
SKILL.md declares:
  allowed-tools:
    - Bash(psql:*)         # database operations
    - Bash(agent-browser:*)  # for chained browser skills
    - Read
    - Write
    - Skill                # can invoke other skills
```

When `/stock-trade` processes a buy order:

1. Validates the order (price, shares, risk limits)
2. Inserts records into PostgreSQL
3. Updates the daily analysis file
4. **Calls `/stock-watchlist`** to sync the browser-based watchlist

### Skill 2: /stock-watchlist

Called by `/stock-trade` via the Skill tool. It:

1. Checks CDP availability
2. Navigates to the trading platform
3. Reads current watchlist via authenticated API
4. Computes the diff (what to add, what to remove)
5. Makes API calls to sync
6. Returns success/failure to the caller

### Skill 3 (optional): /stock-alerts

After a trade, the system may also chain into `/stock-alerts` to set up price monitoring:

```
/stock-trade
    |
    +---> /stock-watchlist  (browser sync)
    +---> /stock-alerts     (set stop-loss alert)
```

## How Chaining Works

A skill invokes another skill via the `Skill` tool:

```
# Inside /stock-trade's execution, Claude Code calls:
Skill(name="stock-watchlist", args="sync 600519")
```

The chained skill loads its own SKILL.md, gets its own tool permissions, and executes independently. When it finishes, control returns to the parent skill.

## Data Flow Between Skills

Skills don't share memory directly. They communicate through:

| Channel | Example |
|---|---|
| Database | /stock-trade writes holdings; /stock-watchlist reads them |
| File system | /stock-trade writes daily analysis; /stock-post-market reads it |
| Arguments | /stock-trade passes stock codes to /stock-watchlist |
| Tool output | /stock-watchlist returns success/failure text |

## Full Sequence Diagram

```
User           /stock-trade      PostgreSQL      /stock-watchlist    Chrome
  |                 |                |                  |               |
  |-- "buy 600519"->|                |                  |               |
  |                 |-- INSERT ----->|                  |               |
  |                 |<-- OK ---------|                  |               |
  |                 |                |                  |               |
  |                 |-- Skill ------>|                  |               |
  |                 |                |-- list-pages --->|               |
  |                 |                |                  |-- CDP ------->|
  |                 |                |                  |<-- tabs ------|
  |                 |                |-- navigate ----->|               |
  |                 |                |                  |-- CDP ------->|
  |                 |                |                  |<-- OK --------|
  |                 |                |-- eval fetch --->|               |
  |                 |                |                  |-- fetch() --->|
  |                 |                |                  |<-- JSON ------|
  |                 |<-- "synced" ---|                  |               |
  |                 |                |                  |               |
  |<-- "Done" ------|                |                  |               |
```

## Why Not One Giant Skill?

Separation gives you:

- **Reusability:** `/stock-watchlist` works standalone or chained
- **Testability:** Test browser automation independently from trade logic
- **Token efficiency:** Each skill loads only when needed, then unloads
- **Permissions:** Each skill declares only the tools it needs

## Next Steps

- [Setup guide](../docs/setup.md) -- get agent-browser running
- [Skill vs MCP](../docs/skill-vs-mcp.md) -- why this architecture over MCP
