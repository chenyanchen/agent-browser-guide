# Skill vs MCP for agent-browser

Both Claude Code Skills and MCP (Model Context Protocol) servers can wrap agent-browser. This document compares them in depth to help you choose.

## The Core Difference

- **Skill:** A markdown file loaded on demand. Zero cost when idle. Instructions + tool permissions.
- **MCP server:** A persistent process that registers tools. Tools are always in context once enabled.

## Token Economics

This is the decisive factor for most use cases.

### MCP Tool Registration Cost

When an MCP server registers tools, each tool's schema (name, description, parameters) is injected into every API call as part of the system prompt. This is a **permanent per-message tax**.

| MCP Server Complexity | Tools Registered | Token Cost (per message) |
|---|---|---|
| Simple (3-5 tools) | 5 | ~1,500-3,000 tokens |
| Medium (10-20 tools) | 15 | ~6,000-12,000 tokens |
| Complex (30-50 tools) | 40+ | ~12,000-24,000 tokens |

A complex MCP server with 40+ tools burns 12,000-24,000 tokens on **every single message** in the conversation, whether or not you use any of those tools.

### Skill Loading Cost

A skill loads its SKILL.md into context **only when invoked**. Before invocation: 0 tokens. After the task is done, the instructions can be dropped from active context.

| Scenario | Skill Cost | MCP Cost |
|---|---|---|
| 20-message conversation, browser used in 2 messages | ~2,000 tokens total | ~30,000-60,000 tokens total |
| 50-message conversation, browser used in 5 messages | ~5,000 tokens total | ~75,000-150,000 tokens total |
| 100-message conversation, never use browser | 0 tokens | ~150,000-300,000 tokens total |

### Real Numbers

In a typical development session (50 messages), an MCP server with 15 tools costs roughly **50 x 8,000 = 400,000 extra tokens**. At Claude's API pricing, that's real money. With a skill, you pay only for the messages where you actually use browser automation.

## Pre-Authorization

### Skills

```yaml
allowed-tools:
  - Bash(agent-browser:*)
```

One line in SKILL.md. When the skill is invoked, all `agent-browser` commands run without per-command user approval. The user approved the skill invocation itself.

### MCP

Each MCP tool call requires individual user approval (unless the user has configured auto-approve rules). For a workflow that chains 5 agent-browser operations, that's 5 approval prompts vs. 1.

## Command Chaining

### Skills: Natural Composition

A skill is instructions in natural language. Claude reads them and executes a sequence of commands intelligently:

```markdown
## Workflow
1. Check CDP availability
2. Navigate to the API subdomain
3. Fetch the watchlist
4. Compare with database
5. Add missing stocks
6. Verify changes
```

Claude executes all 6 steps as a coherent sequence, adapting if step 3 returns an error.

### MCP: Individual Tool Calls

An MCP server exposes atomic tools. Claude must call them one at a time:

```
Tool call 1: browser_navigate(url="...")
Tool call 2: browser_eval(script="...")
Tool call 3: browser_eval(script="...")
...
```

Each tool call is a separate round-trip. The model must plan the sequence itself, without the benefit of documented workflow instructions.

## Skill Chaining vs MCP Composition

Skills can invoke other skills via the Skill tool:

```
/stock-trade --> /stock-watchlist --> agent-browser
```

This is a clean dependency chain where each skill brings its own context and permissions.

MCP servers can call each other, but the composition model is less structured -- it's up to the model to figure out that tool X from server A should be followed by tool Y from server B.

## Vercel's Position

Vercel notably closed PRs proposing MCP server implementations for their tools and instead ships an official `SKILL.md`:

```
.claude/skills/vercel-skill/SKILL.md
```

Their reasoning aligns with the token economics argument: MCP's always-on tool registration is wasteful for tools that are used intermittently. Skills provide the same capability with on-demand loading.

This isn't a fringe opinion -- it reflects a pattern emerging among tool authors who optimize for real-world Claude Code usage.

## When MCP Is Still Appropriate

MCP is the right choice when:

1. **High-frequency usage:** If you use browser automation in >80% of messages, the amortized cost of always-on tools is acceptable.

2. **Shared infrastructure:** MCP servers can serve multiple clients (not just Claude Code). If you have a browser automation server used by multiple AI agents, MCP makes sense.

3. **Complex state management:** MCP servers can maintain state (open connections, session tokens) across tool calls. Skills rely on external state (files, databases, the browser itself).

4. **Non-Claude clients:** MCP is a protocol. Skills are Claude Code-specific. If you need to support other AI coding assistants, MCP is the portable choice.

5. **Discovery:** MCP tools are always visible to the model. Skills require the user to know the slash command or the model to know to invoke them.

## Decision Framework

```
Do you use browser automation in most messages?
  YES --> Consider MCP (amortized cost is acceptable)
  NO  --> Use a Skill (pay only when used)

Do you need to serve multiple AI clients?
  YES --> MCP (it's a protocol, not Claude-specific)
  NO  --> Skill (simpler, cheaper)

Do you need complex multi-step workflows?
  YES --> Skill (natural language instructions > atomic tools)
  NO  --> Either works

How many tools would the MCP server register?
  < 5  --> MCP is fine (low overhead)
  5-15 --> Skill preferred (meaningful savings)
  > 15 --> Skill strongly preferred (MCP cost is prohibitive)
```

## Summary Table

| Dimension | Skill | MCP |
|---|---|---|
| Idle cost | 0 tokens | 1,500-24,000 tokens/message |
| Load model | On-demand | Always-on |
| Authorization | 1 approval per skill | 1 approval per tool call |
| Composition | Skill chaining (structured) | Ad-hoc tool sequencing |
| Portability | Claude Code only | Any MCP client |
| State management | External (DB, files, browser) | Internal (server process) |
| Workflow docs | Built-in (SKILL.md) | Separate (README, comments) |
| Multi-client | No | Yes |

## Next Steps

- [Building a skill](../examples/03-claude-code-skill.md) -- hands-on skill creation
- [Skill chaining](../examples/04-skill-chaining.md) -- multi-skill orchestration
- [Gotchas](gotchas.md) -- practical pitfalls to avoid
