# 停止在浏览器自动化上浪费 95% 的 Token

> 通过 CDP 将 AI Agent 连接到你的真实 Chrome。同一个浏览器，同一套 Cookie，成本降低 20 倍。

[English Version](README.md)

AI Agent 需要与网页交互——查价格、调 API、填表单。但现有方案的 Token 消耗高得离谱。

**本指南展示一种更好的方式：** 用 [agent-browser](https://github.com/vercel-labs/agent-browser) 通过 CDP（Chrome DevTools Protocol）连接你**正在使用的 Chrome**。你的 Cookie、你的登录态、你的扩展——Token 成本只要原来的几十分之一。

## 问题

"Chrome in Claude"（计算机使用 / MCP 浏览器工具）有三个根本性问题：

| 问题 | 为什么严重 |
|------|----------|
| **截图驱动** | 每次操作：渲染 → 截图 → Base64 → 视觉模型 → 推理。一次点击消耗 ~2,000-8,000 Token |
| **没有登录态** | 全新浏览器实例 = 没有 Cookie。无法调用需要鉴权的 API |
| **网站拦截** | 很多网站拒绝自动化浏览器：wise.com、reddit.com、mp.weixin.qq.com 等 |

出了问题怎么办？Agent 多截几张图来排查，Token 和时间翻倍。

## 解决方案

```bash
agent-browser --auto-connect eval 'document.title'
```

一行命令。通过 CDP 连接你正在运行的 Chrome。你的 Cookie。你的登录态。一切直接可用。

```
AI Agent（Claude Code / Cursor / Codex 等）
  └── bash: agent-browser --auto-connect <command>
        └── Chrome DevTools Protocol (CDP)
              └── 你的 Chrome（包含所有 Cookie、扩展、登录态）
```

## 性能对比

来自股票交易自动化系统的真实测量数据（2026-02-28）：

### 单任务对比

| 任务 | agent-browser 耗时 | agent-browser Token | Chrome in Claude Token | 降幅 |
|------|-------------------|---------------------|----------------------|------|
| 股价 JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| 带鉴权的 API 调用 | 3.5s | **217** | 3,000 | 14x |
| 财经页面快照 | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| 搜索结果页面 | 1.8s | **41** | 5,000-8,000 | 122-195x |
| **合计（4 个任务）** | **10.8s** | **2,092** | **20,000-32,000** | **10-15x** |

### 整体对比

| 指标 | Chrome in Claude | agent-browser + CDP |
|------|-----------------|-------------------|
| 每次操作 Token | ~2,000-8,000 | **~50-1,800** |
| 每次操作速度 | ~8-15 秒（截图 + 视觉模型） | **~2-4 秒** |
| 卡住时 | 更多截图，速度减半 | 不存在此问题 |
| 网站访问 | wise.com、reddit、微信被拦截 | **全部可访问** |
| 登录态 | 无（全新浏览器） | **完整（你的真实 Chrome）** |
| 空闲 Token 开销 | MCP：12-24K Token 常驻 | Skill：~150 Token |

> 测量方法：[benchmarks/results.md](benchmarks/results.md)

## 快速开始

### 1. 启用 Chrome 远程调试

在 Chrome 地址栏输入：

```
chrome://inspect/#remote-debugging
```

点击 **"Enable"**。完成。不需要重启 Chrome。（需要 Chrome 145+）

### 2. 安装 agent-browser

```bash
# macOS
brew install agent-browser

# 或用 npm
npm install -g agent-browser && agent-browser install
```

### 3. 验证连接

```bash
agent-browser --auto-connect get url
# 应该输出你当前 Chrome 标签页的 URL
```

搞定。你的 AI Agent 现在可以控制你的 Chrome 了。

## 集成模式

### 模式 A：获取 JSON API（eval + fetch）

利用浏览器的 Cookie 直接调用带鉴权的 API：

```bash
# 先导航到 API 的域名（确保同源 Cookie）
agent-browser --auto-connect open "https://api.example.com/"
agent-browser --auto-connect wait 2000

# 调用 API — Cookie 自动带上
agent-browser --auto-connect eval --stdin <<'EOF'
fetch("https://api.example.com/user/watchlist")
  .then(r => r.json())
  .then(d => JSON.stringify(d))
EOF
```

**输出：** 纯 JSON。~100-300 Token。没有 HTML，没有 Markdown，没有噪音。

### 模式 B：导航 + 交互（snapshot + refs）

需要点击、填表或读取页面内容时：

```bash
agent-browser --auto-connect open "https://example.com/dashboard"
agent-browser --auto-connect wait --load networkidle
agent-browser --auto-connect snapshot -i
# 输出：
# - button "Sign Out" [ref=e1]
# - link "Settings" [ref=e2]
# - textbox "Search" [ref=e3]

agent-browser --auto-connect fill @e3 "query"
agent-browser --auto-connect press Enter
```

**输出：** 紧凑的无障碍树，带引用标记。每页 ~200-2,000 Token（WebFetch 需要 8,000-15,000）。

### 模式 C：Claude Code Skill

构建可复用的 Skill，底层调用 agent-browser：

```yaml
---
name: my-browser-skill
description: 通过用户浏览器完成 X 操作
allowed-tools: Bash(agent-browser:*)
---
```

`allowed-tools` frontmatter 预授权 agent-browser 命令——不再逐次弹出权限确认。

完整模板见 [examples/03-claude-code-skill.md](examples/03-claude-code-skill.md)。

### 模式 D：Skill 链式调用

多个 Skill 串联实现复杂工作流：

```
/stock-trade（记录交易到 PostgreSQL）
  → /stock-watchlist（同步到券商自选股）
    → agent-browser --auto-connect（调用鉴权 API）
      → Chrome CDP → 券商 API（带用户 Cookie）
```

完整架构见 [examples/04-skill-chaining.md](examples/04-skill-chaining.md)。

## 避坑指南

### CORS：必须导航到正确的子域名

API 只能接收**同源**的 Cookie。如果 API 在 `t.example.com`，你必须先导航到那里：

```bash
# 错误 — CORS 失败
agent-browser --auto-connect open "https://www.example.com"
agent-browser --auto-connect eval 'fetch("https://t.example.com/api/data")'
# TypeError: Failed to fetch

# 正确 — 同源
agent-browser --auto-connect open "https://t.example.com/"
agent-browser --auto-connect eval 'fetch("https://t.example.com/api/data")'
# 成功！
```

### Chrome 重启

重启 Chrome 后，需要重新在 `chrome://inspect/#remote-debugging` 启用远程调试。这是设计如此——Chrome 出于安全考虑不会持久化 CDP 访问权限。

### Shell 转义

用 `<<'EOF'`（单引号 heredoc）避免 shell 解释 JavaScript 中的特殊字符：

```bash
# 错误 — shell 会展开 $、!、反引号
agent-browser --auto-connect eval "fetch(`https://api.com/${path}`)"

# 正确 — heredoc 绕过 shell
agent-browser --auto-connect eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

### 限流

连续 API 调用之间加延迟：

```bash
agent-browser --auto-connect eval '...'  # 调用 1
agent-browser --auto-connect wait 2000    # 等 2 秒
agent-browser --auto-connect eval '...'  # 调用 2
```

### 安全

CDP 给予浏览器的**完全访问权限**——所有标签页、所有 Cookie、所有 localStorage。仅在个人电脑上使用，且只在需要自动化的网站上保持登录。

更多细节：[docs/gotchas.md](docs/gotchas.md)

## Skill vs MCP：为什么 Skill 更优

agent-browser 为 Claude Code 提供两种集成路径：

| 维度 | Skill (CLI) | MCP Server |
|------|-------------|------------|
| 空闲 Token 成本 | ~150 Token（仅描述信息） | **12,000-24,000 Token**（40+ 工具 Schema，始终加载） |
| 每次操作成本 | ~50-200 Token（bash 单行命令） | ~500-2,000（结构化工具调用） |
| 命令链接 | `&&` 链接减少往返 | 每个操作 = 独立工具调用 |
| 预授权 | SKILL.md frontmatter 内置 | 手动配置，已知 Bug |
| 成熟度 | 官方维护（Vercel） | 社区包装（单人作者） |
| Vercel 推荐 | **是**（官方发布 SKILL.md） | 否（关闭了多个 MCP PR） |

**结论：** MCP 即使你不用浏览器也会常驻消耗 12-24K Token。Skill 方式空闲时成本近乎为零。

深入分析：[docs/skill-vs-mcp.md](docs/skill-vs-mcp.md)

## 真实案例

本指南来自用 Claude Code 构建自动化股票交易系统的实践。该系统使用 8 个 Skill 编排完整的每日工作流：

```
盘前分析 → 盘中交易 → 盘后复盘
   ↓           ↓          ↓
/stock-pre-market  /stock-trade  /stock-post-market
(隔夜数据分析)    (DB + 告警 +    (快照 + 截图
                  自选股同步)     + 次日计划)
```

agent-browser + CDP 负责**自选股同步**——调用券商（同花顺）的鉴权 API 来添加和删除股票。具体实现：

1. 通过 CDP 导航到 `t.10jqka.com.cn`（同花顺 API 子域名）
2. 利用浏览器中已有的登录 Cookie，调用 `fetch()` 请求自选股 API
3. API 返回纯 JSON，Agent 直接解析结果

在使用 agent-browser 之前，这个操作要么：
- 手动操作（违背自动化的初衷）
- WebFetch 带 Cookie 调用（不稳定，503 限流，每次调用约 3,000 Token）

使用 agent-browser 后：**217 Token、3.5 秒、100% 可靠**。

完整的 Skill 链路是这样的：

```
/stock-trade（记录交易到 PostgreSQL）
  → /stock-watchlist（同步到同花顺自选股）
    → agent-browser --auto-connect（调用鉴权 API）
      → Chrome CDP → 同花顺 API（带用户 Cookie）
        → 返回 {"errcode":0,"errmsg":"success"}
```

整个链路——数据库写入、API 调用、结果验证——在 5 秒内完成，总共约 500 Token。

深入了解：[posts/zhihu-cn.md](posts/zhihu-cn.md)

## 什么时候不该用

- **探索未知网站** — 用 MCP 浏览器工具或计算机使用（你需要视觉能力）
- **视觉验证** — 用 `screenshot --annotate`（agent-browser 也支持此功能）
- **CI/CD 流水线** — 用无头模式 + Profile，不要用 CDP 连接活跃浏览器
- **多用户系统** — CDP 连接的是单个用户的浏览器；多租户需要会话管理

## 资源

- [agent-browser](https://github.com/vercel-labs/agent-browser) — CLI 工具
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) — 底层协议
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — 构建可复用 Skill

## 许可证

[MIT](LICENSE)
