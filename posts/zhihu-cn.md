# 用 agent-browser + CDP 连接真实 Chrome，AI Agent 浏览器 Token 消耗降低 95%

> 把 AI Agent 接到你正在用的 Chrome 上，同一个浏览器、同一套 Cookie，Token 消耗降到原来的 1/20。

---

## 1. 背景：AI Agent 为什么需要浏览器

我用 Claude Code 构建了一套 A 股交易自动化系统。每天的工作流是这样的：

```
盘前分析 → 盘中交易 → 盘后复盘
   ↓           ↓          ↓
/stock-pre-market  /stock-trade  /stock-post-market
(隔夜数据分析)    (DB + 告警 +    (快照 + 截图
                  自选股同步)     + 次日计划)
```

八个 Claude Code Skill 编排整个流程。其中，`/stock-watchlist` 负责把交易信号同步到券商自选股。逻辑很简单：调一个带鉴权的 API，添加或删除股票代码。

问题是——虽然 Claude in Chrome 能访问浏览器的登录态，但视觉模型的介入让操作成本偏高。

## 2. 现有方案的痛点

### Claude in Chrome 的两大问题

Claude in Chrome 连接的是你的真实 Chrome——有 Cookie、有登录态。它优先通过 DOM 和无障碍树操作页面，无法直接操作时才回退到截图。但视觉模型的介入仍然带来额外开销：

| 问题 | 具体表现 |
|------|---------|
| **视觉模型开销** | 当回退到截图时，每次往返涉及渲染、Base64 编码和视觉模型推理——增加显著的 Token 消耗和延迟 |
| **访问受限** | 部分网站在自动化模式下仍然被拦截或行为异常：wise.com、reddit.com、mp.weixin.qq.com 等 |

更要命的是错误恢复：页面没加载出来？Agent 多截几张图来排查，Token 和时间成倍增加。

我用 WebFetch 也试过——带 Cookie 调 API。结果是 503 限流、3,000 Token 一次调用、成功率不到 70%。

### 根本矛盾

这些方案都在试图**新建一个浏览器环境**去模拟用户。但用户已经有一个完美的浏览器环境了——正在运行的 Chrome，里面有完整的登录态、Cookie、插件。为什么不直接用它？

## 3. 解决方案：agent-browser + CDP

### 什么是 CDP

Chrome DevTools Protocol（CDP）是 Chrome DevTools 使用的协议——你按 F12 打开开发者工具，底层就是 CDP 在通信。它从 2017 年就存在了，每个 Chrome 扩展和调试工具都依赖它。

关键能力：**CDP 可以连接到一个正在运行的 Chrome 实例**。不是新建的无头浏览器，是你正在用的那个 Chrome——你的 Cookie、你的登录态、你的插件，全都在。

### agent-browser 是什么

[agent-browser](https://github.com/vercel-labs/agent-browser) 是 Vercel Labs 维护的 CLI 工具，把 CDP 封装成干净的命令行接口：

```bash
# 一行连接你的 Chrome
agent-browser eval 'document.title'
```

架构对比：

```
之前（Claude in Chrome）：
  Agent → 你的 Chrome → DOM/无障碍树 + 截图回退 → 视觉模型
  成本：视觉模型介入时 Token 开销显著增加

之后（agent-browser + CDP）：
  Agent → Bash: agent-browser eval '...'
       → CDP → 你的 Chrome → JSON 响应
  成本：每次操作 50-200 Token，2-4 秒
```

### 安装（5 分钟）

**第一步：启用 Chrome 远程调试**

地址栏输入：

```
chrome://inspect/#remote-debugging
```

勾选 **"Allow remote debugging for this browser instance"**。不需要重启 Chrome。（需要 Chrome 145+）

**第二步：安装并配置**

```bash
# macOS
brew install agent-browser

# 或用 npm
npm install -g agent-browser && agent-browser install

# 连接你的真实 Chrome（而不是启动无头浏览器）
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{ "autoConnect": true }
EOF
```

**第三步：验证连接**

```bash
agent-browser get url
# 应该输出你当前 Chrome 标签页的 URL
```

## 4. 实测数据

以下是我的股票交易系统中四个真实任务的实测数据（2026-02-28）：

### 单任务对比

| 任务 | agent-browser 耗时 | agent-browser Token | Claude in Chrome Token | 降幅 |
|------|-------------------|---------------------|----------------------|------|
| 股价 JSON API | 2.6s | **57** | 4,000-6,000 | 70-105x |
| 带鉴权的 API 调用 | 3.5s | **217** | 3,000 | 14x |
| 财经页面快照 | 2.9s | **1,777** | 8,000-15,000 | 4.5-8x |
| 搜索结果页面 | 1.8s | **41** | 5,000-8,000 | 122-195x |
| **合计（4 个任务）** | **10.8s** | **2,092** | **20,000-32,000** | **10-15x** |

### 整体对比

| 指标 | Claude in Chrome | agent-browser + CDP |
|------|-----------------|-------------------|
| 每次操作 Token | ~2,000-8,000 | **~50-1,800** |
| 每次操作速度 | ~8-15 秒（截图 + 视觉模型） | **~2-4 秒** |
| 卡住时 | 更多截图，成本翻倍 | 返回错误文本，开销极小 |
| 网站访问 | wise.com、reddit、微信被拦截 | 所有测试站点均可访问 |
| Cookie / 登录态 | ✅ 有（你的 Chrome） | ✅ 有（通过 CDP 连接你的 Chrome） |
| 工作机制 | DOM + 无障碍树，截图回退 | JavaScript eval / 无障碍树 |

几个值得注意的点：

- **JSON API 场景差距最大**：只需要结构化数据时，agent-browser 直接返回原始 JSON（57 Token），而 Claude in Chrome 需要导航、交互、可能回退到视觉模型——Token 开销显著更高。
- **搜索页面压缩效果惊人**：搜索结果的结构化特征（标题、URL、摘要）完美映射到无障碍树节点，41 Token vs 5,000-8,000。
- **速度差异会叠加**：5 个浏览器操作链起来，Claude in Chrome 要 40-75 秒，agent-browser 只要 10-20 秒。

## 5. 实战：同花顺自选股管理

这是我实际在用的场景——把交易信号同步到同花顺自选股。

### 核心流程

```bash
# 第一步：导航到 API 所在的子域名（CORS 要求，后面会详细说）
agent-browser open "https://t.10jqka.com.cn/"
agent-browser wait 2000

# 第二步：调用自选股 API（Cookie 自动带上）
agent-browser eval --stdin <<'EOF'
fetch("https://t.10jqka.com.cn/openapi/stockconcern/v2/add", {
  method: "POST",
  headers: {
    "Content-Type": "application/x-www-form-urlencoded"
  },
  body: "stockcode=600519&marketid=17"
})
.then(r => r.json())
.then(d => JSON.stringify(d))
EOF

# 第三步：验证结果
# 输出: {"errcode":0,"errmsg":"success"}
```

### 完整的删除操作

```bash
# 从自选股删除
agent-browser eval --stdin <<'EOF'
fetch("https://t.10jqka.com.cn/openapi/stockconcern/v2/remove", {
  method: "POST",
  headers: {
    "Content-Type": "application/x-www-form-urlencoded"
  },
  body: "stockcode=600519&marketid=17"
})
.then(r => r.json())
.then(d => JSON.stringify(d))
EOF
```

### 批量操作

```bash
# 批量添加多只股票
agent-browser eval --stdin <<'EOF'
const codes = [
  { code: "600519", market: "17" },
  { code: "000858", market: "33" },
  { code: "601318", market: "17" }
];

const results = [];
for (const { code, market } of codes) {
  const r = await fetch("https://t.10jqka.com.cn/openapi/stockconcern/v2/add", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: `stockcode=${code}&marketid=${market}`
  });
  const d = await r.json();
  results.push({ code, ...d });
  await new Promise(resolve => setTimeout(resolve, 500)); // 限流保护
}
JSON.stringify(results);
EOF
```

效果：**217 Token、3.5 秒、100% 成功率**。对比之前用 WebFetch 的 3,000 Token、不确定的成功率，体验天壤之别。

## 6. Claude Code Skill 集成

Claude Code 的 Skill 系统允许你把常用操作封装成可复用的指令。下面是自选股同步 Skill 的关键部分：

### SKILL.md 模板

```yaml
---
name: stock-watchlist
description: 通过浏览器自动化同步自选股到券商
allowed-tools: Bash(agent-browser:*)
---
```

`allowed-tools` 是关键——它预授权所有以 `agent-browser:` 开头的 bash 命令，Agent 不会每次都停下来问权限。

### Skill 内容（Markdown 格式）

```markdown
## 连接浏览器

先确认 Chrome 已启用远程调试（chrome://inspect/#remote-debugging）。

## 添加自选股

1. 导航到 t.10jqka.com.cn（必须是 t. 子域名，不能是 www.）
2. 调用 /openapi/stockconcern/v2/add API
3. 验证返回 errcode=0

## 删除自选股

1. 调用 /openapi/stockconcern/v2/remove API
2. 验证返回 errcode=0
```

## 7. Skill vs MCP：Token 经济学

agent-browser 同时提供 CLI（Skill 方式）和 MCP Server 两种集成路径。为什么我选 Skill？

### 空闲成本对比

| 维度 | Skill (CLI) | MCP Server |
|------|-------------|------------|
| 空闲 Token 消耗 | ~150 Token（仅描述信息） | **12,000-24,000 Token**（40+ 工具 Schema，始终加载） |
| 每次操作成本 | ~50-200 Token（bash 单行命令） | ~500-2,000 Token（结构化工具调用） |
| 命令链接 | `&&` 链接减少往返 | 每个操作 = 独立工具调用 |
| 预授权 | SKILL.md frontmatter 内置 | 手动配置，已知 Bug |
| 成熟度 | 官方维护（Vercel） | 社区包装（单人作者） |
| Vercel 推荐 | **是**（官方发布 SKILL.md） | 否（关闭了多个 MCP PR） |

### 算一笔账

假设一个典型会话：30 分钟写代码，2 分钟浏览器操作（5 次动作），再写 30 分钟代码。

**MCP 方式：**
- 空闲成本：18,000 Token（Schema 始终占据上下文窗口）
- 操作成本：5 x 1,000 = 5,000 Token
- 合计：**23,000 Token**

**Skill 方式：**
- 空闲成本：150 Token（仅描述）
- Skill 加载：~800 Token（调用时加载，结束后卸载）
- 操作成本：5 x 150 = 750 Token
- 合计：**1,700 Token**

**差距 13.5 倍**。而且你花在非浏览器工作上的时间越多，差距越大。

## 8. 完整架构：8 个 Skill 的交易闭环

我的完整交易系统由 8 个 Claude Code Skill 组成：

```
┌─────────────────────────────────────────────────────┐
│                    每日工作流                          │
├───────────────┬──────────────┬───────────────────────┤
│    盘前        │    盘中       │    盘后               │
├───────────────┼──────────────┼───────────────────────┤
│/stock-pre-market│/stock-trade │/stock-post-market     │
│  隔夜数据分析   │  记录交易    │  持仓快照             │
│  生成操作计划   │  更新告警    │  盈亏分析             │
│               │  同步自选股   │  次日计划             │
├───────────────┴──────────────┴───────────────────────┤
│                    支撑 Skill                         │
├──────────┬───────────┬────────────┬──────────────────┤
│/stock-   │/stock-    │/stock-     │/stock-           │
│analyze   │position   │alerts      │watchlist         │
│ 个股分析  │ 仓位计算   │ 价格告警   │ 自选股同步        │
│          │           │ (launchd)  │ (agent-browser)  │
└──────────┴───────────┴────────────┴──────────────────┘
                                         ↓
                                    Chrome CDP
                                         ↓
                                    同花顺 API
```

关键链路：

```
/stock-trade（记录交易到 PostgreSQL）
  → /stock-watchlist（同步到券商自选股）
    → agent-browser（调用鉴权 API）
      → Chrome CDP → 同花顺 API（带用户 Cookie）
```

整个链路——数据库写入、API 调用、结果验证——在 5 秒内完成，总共约 500 Token。

## 9. 避坑指南

### CORS：子域名陷阱

这个坑花了我两个小时。同花顺的 API 在 `t.10jqka.com.cn`，主站是 `www.10jqka.com.cn`。我先导航到主站，再去调 API 子域名：

```bash
# 错误 — CORS 失败
agent-browser open "https://www.10jqka.com.cn"
agent-browser eval 'fetch("https://t.10jqka.com.cn/api/data")'
# TypeError: Failed to fetch

# 正确 — 同源
agent-browser open "https://t.10jqka.com.cn/"
agent-browser eval 'fetch("https://t.10jqka.com.cn/api/data")'
# 成功！
```

**原则**：调 API 前，必须先导航到 API **所在的子域名**。`www.` 和 `t.` 是不同源。

### Heredoc 单引号

JavaScript 里充斥着 shell 爱解释的字符：`$`、反引号、`!`、`{}`。用单引号 heredoc 防止 shell 展开：

```bash
# 错误 — shell 会展开 $ 和反引号
agent-browser eval "fetch(`https://api.com/${path}`)"

# 正确 — heredoc 绕过 shell
agent-browser eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

`<<'EOF'`（EOF 外面有引号）是关键。没有引号的话，shell 仍然会展开 heredoc 内的变量。

### 限流保护

连续调用 API 时加延迟：

```bash
agent-browser eval '...'  # 调用 1
agent-browser wait 2000    # 等 2 秒
agent-browser eval '...'  # 调用 2
```

不加延迟，快速连发的请求容易触发目标 API 的限流（429 错误或临时封禁）。

### 安全注意

CDP 给予的是你浏览器的**完全访问权限**——所有标签页、所有 Cookie、所有 localStorage。使用建议：

- 仅在个人电脑上启用
- 注意 CDP 开启时你登录了哪些网站
- 不要把 CDP 端口暴露到网络
- Chrome 重启后 CDP 自动关闭（这是安全特性，不是 Bug）

## 10. 总结

这不是"agent-browser vs MCP"的二选一。每个工具有自己的适用场景：

- **Computer Use / MCP 浏览器工具**：适合视觉探索未知网站、截图验证、需要"看到"页面的任务
- **agent-browser + CDP**：适合结构化数据提取、鉴权 API 调用、已知页面的重复性工作流

核心洞见很简单：**大多数浏览器自动化任务不需要视觉**。它们需要数据。把"渲染页面 → 截图 → Base64 → 视觉模型 → 提取文本"替换成"fetch() → JSON"，Token 降低 95%，速度提升 4 倍。

而且，这是一个**通用方案**。agent-browser 是 CLI 工具——任何能执行 shell 命令的 AI Agent 都可以用它。我用的是 Claude Code，但 Codex、Cursor、Windsurf、Copilot 也完全适用，配置方法一模一样。不绑定厂商，不需要 SDK。只要你的 Agent 能跑 `bash`，它就能操控你的浏览器。

如果你的 AI Agent 经常与网页交互——特别是需要鉴权的 API——试试 CDP 方案。安装只要 5 分钟，Token 的节省会随着每次操作不断累积。

---

*完整指南（含 Benchmark、代码示例、Claude Code Skill 模板）：[github.com/chenyanchen/agent-browser-guide](https://github.com/chenyanchen/agent-browser-guide)*
