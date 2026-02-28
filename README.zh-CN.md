# 通过 CDP 将 AI Agent 连接到你的真实 Chrome

> 用 [agent-browser](https://github.com/vercel-labs/agent-browser) + Chrome DevTools Protocol，让 AI Agent 使用你浏览器的 Cookie、登录态和扩展——Token 消耗降低 10-15 倍。

[English Version](README.md)

---

## 目录

| 我想…… | 跳转到 |
|--------|-------|
| **立即上手** | [快速开始](#快速开始) |
| **先了解问题** | [问题](#问题) → [性能对比](#性能对比) |
| **集成到工作流** | [集成模式](#集成模式) → [避坑指南](#避坑指南) |
| **对比 Skill 和 MCP** | [Skill vs MCP](#skill-vs-mcp为什么-skill-更优) |
| **看真实案例** | [案例](#真实案例) |
| **查术语** | [术语表](#术语表) |

---

## 快速开始

### 做什么

将 agent-browser 连接到你正在运行的 Chrome，让 AI Agent 继承你浏览器的 Cookie、登录态和扩展。

### 为什么

默认情况下，agent-browser 会启动一个**全新的无头浏览器**——没有 Cookie、没有登录态。访问公开页面没问题。

但如果你的 Agent 需要：
- **调用带鉴权的 API**（Cookie、SSO、内部工具）
- **访问拦截自动化浏览器的网站**（微信公众号、Reddit、wise.com）
- **操作公司内网系统**

……你需要连接到你**真实的 Chrome**。下面的配置就是做这件事。

> **安全提示：** CDP 授予浏览器的完全访问权限——所有标签页、Cookie 和 localStorage。仅在个人电脑上启用。[详情 →](#安全)

### 怎么做

#### 1. 启用 Chrome 远程调试

在 Chrome 地址栏输入：

```
chrome://inspect/#remote-debugging
```

勾选 **"Allow remote debugging for this browser instance"**。不需要重启 Chrome。（需要 Chrome 145+）

#### 2. 安装 agent-browser

```bash
# macOS
brew install agent-browser

# 或用 npm
npm install -g agent-browser && agent-browser install
```

#### 3. 配置自动连接

```bash
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{
  "autoConnect": true
}
EOF
```

配置后 agent-browser 会连接你正在运行的 Chrome，而不是启动无头浏览器。

#### 4. 验证

```bash
agent-browser get url
# 应该输出你当前 Chrome 标签页的 URL
```

搞定。你的 AI Agent 现在可以控制你的 Chrome 了。

## 问题

Claude in Chrome 连接的是你的真实浏览器——有 Cookie 和登录态。它优先通过 DOM 和无障碍树操作页面，无法直接操作时才回退到截图。但视觉模型的介入仍然带来额外开销：

| 问题 | 影响 |
|------|------|
| **视觉模型开销** | 当 Claude in Chrome 回退到截图时，每次往返涉及渲染、Base64 编码和视觉模型推理——增加显著的 Token 消耗和延迟 |
| **访问受限** | 部分网站在 Claude in Chrome 的自动化模式下仍然被拦截或行为异常：wise.com、reddit.com、mp.weixin.qq.com 等 |

出了问题怎么办？Agent 多截几张图来排查，增加 Token 消耗和耗时。

## 解决方案

```bash
agent-browser eval 'document.title'
```

一条命令通过 CDP 连接到你正在运行的 Chrome——继承所有 Cookie 和登录态。

```
AI Agent（Claude Code / Cursor / Codex / Windsurf / ...）
  └── bash: agent-browser <command>
        └── Chrome DevTools Protocol (CDP)
              └── 你的 Chrome（包含所有 Cookie、扩展、登录态）
```

**这是一个通用方案。** agent-browser 是一个 CLI 工具——任何能执行 shell 命令的 AI Agent 都可以使用它。Claude Code、Codex、Cursor、Windsurf、Copilot——配置完全相同。不绑定任何厂商，不需要 SDK 集成。只要你的 Agent 能跑 `bash`，它就能操控你的浏览器。

## 性能对比

> **环境：** macOS Sequoia 15.3、Chrome 145+、中国大陆住宅宽带
> **日期：** 2026-02-28 | **样本：** 股票交易系统中 4 个任务，每个单次运行
> **agent-browser Token：** 实测数据（CLI 输出字符数 ÷ 4）
> **Claude in Chrome Token：** 基于使用经验估算，尚未独立 benchmark。[协助测量 →](https://github.com/chenyanchen/agent-browser-guide/issues)
> **注意：** 结果受页面内容、网络条件和 Chrome 版本影响。[完整方法论 →](benchmarks/results.md)

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
| 网站访问（本次测试范围） | wise.com、reddit、微信被拦截 | 所有测试站点均可访问 |
| Cookie / 登录态 | ✅ 有（你的真实 Chrome） | ✅ 有（通过 CDP 连接你的 Chrome） |
| 工作机制 | DOM + 无障碍树，截图回退 | JavaScript eval / 无障碍树 |
| 空闲 Token 开销 | MCP：12-24K Token 常驻 | Skill：~150 Token |

## 集成模式

### 模式 A：获取 JSON API（eval + fetch）

利用浏览器的 Cookie 直接调用带鉴权的 API：

```bash
# 先导航到 API 的域名（确保同源 Cookie）
agent-browser open "https://api.example.com/"
agent-browser wait 2000

# 调用 API — Cookie 自动带上
agent-browser eval --stdin <<'EOF'
fetch("https://api.example.com/user/watchlist")
  .then(r => r.json())
  .then(d => JSON.stringify(d))
EOF
```

**输出：** 纯 JSON。~100-300 Token。没有 HTML，没有 Markdown，没有噪音。

### 模式 B：导航 + 交互（snapshot + refs）

需要点击、填表或读取页面内容时：

```bash
agent-browser open "https://example.com/dashboard"
agent-browser wait --load networkidle
agent-browser snapshot -i
# 输出：
# - button "Sign Out" [ref=e1]
# - link "Settings" [ref=e2]
# - textbox "Search" [ref=e3]

agent-browser fill @e3 "query"
agent-browser press Enter
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
    → agent-browser（调用鉴权 API）
      → Chrome CDP → 券商 API（带用户 Cookie）
```

完整架构见 [examples/04-skill-chaining.md](examples/04-skill-chaining.md)。

## 避坑指南

### CORS：必须导航到正确的子域名

API 只能接收**同源**的 Cookie。如果 API 在 `t.example.com`，你必须先导航到那里：

```bash
# 错误 — CORS 失败
agent-browser open "https://www.example.com"
agent-browser eval 'fetch("https://t.example.com/api/data")'
# TypeError: Failed to fetch

# 正确 — 同源
agent-browser open "https://t.example.com/"
agent-browser eval 'fetch("https://t.example.com/api/data")'
# 成功！
```

### Chrome 重启

重启 Chrome 后，需要重新在 `chrome://inspect/#remote-debugging` 启用远程调试。这是设计如此——Chrome 出于安全考虑不会持久化 CDP 访问权限。

### Shell 转义

用 `<<'EOF'`（单引号 heredoc）避免 shell 解释 JavaScript 中的特殊字符：

```bash
# 错误 — shell 会展开 $、!、反引号
agent-browser eval "fetch(`https://api.com/${path}`)"

# 正确 — heredoc 绕过 shell
agent-browser eval --stdin <<'EOF'
fetch(`https://api.com/${path}`)
EOF
```

### 限流

连续 API 调用之间加延迟：

```bash
agent-browser eval '...'  # 调用 1
agent-browser wait 2000    # 等 2 秒
agent-browser eval '...'  # 调用 2
```

### Tab 争用（Agent vs 人类）

你和 Agent 共用同一个 Chrome——会互相干扰吗？实测下来不会。agent-browser 连接到某个 tab 后会锁定它，你可以自由使用其他 tab。

规则：不要关闭 Agent 的 tab，不要手动操作它，避免在工作流中途执行 `agent-browser close`。

重度使用可以考虑[独立 Chrome Profile](docs/gotchas.md#7-tab-contention-agent-vs-human)。

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

**我们的实际体验：** MCP 即使不用浏览器也会常驻消耗 12-24K Token。Skill 方式空闲时成本近乎为零。对于间歇性浏览器使用（大多数开发场景），Skill 是更节省 Token 的选择。

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

使用 agent-browser 后（截至 2026-02-28）：**217 Token、3.5 秒，20+ 次日常运行表现一致**。

完整的 Skill 链路：

```
/stock-trade（记录交易到 PostgreSQL）
  → /stock-watchlist（同步到同花顺自选股）
    → agent-browser（调用鉴权 API）
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

## 术语表

| 术语 | 定义 |
|------|------|
| **CDP** | Chrome DevTools Protocol——Chrome 开发者工具（F12）使用的底层协议。允许外部程序控制正在运行的 Chrome 实例。 |
| **MCP** | Model Context Protocol——Anthropic 的 AI 工具连接协议。MCP Server 注册的工具 Schema 会加载到每次 API 调用中。 |
| **Skill** | Claude Code 的功能：按需加载的 Markdown 文件（SKILL.md），声明指令和工具权限。空闲时零 Token 开销。 |
| **WebFetch** | Claude Code 内置的网页抓取工具。将 HTML 转换为 Markdown 并通过快速模型摘要。 |
| **snapshot** | agent-browser 命令，返回页面的紧凑无障碍树（结构化文本，而非截图）。 |
| **eval** | agent-browser 命令，在浏览器页面上下文中执行 JavaScript，等同于在 DevTools 控制台运行代码。 |
| **无头浏览器** | 没有可见窗口的浏览器。自动化工具（Puppeteer、Playwright）常用。除非特别配置，否则无法访问用户的 Cookie 和登录态。 |

## 平台支持

| 平台 | 状态 | 说明 |
|------|------|------|
| macOS (Homebrew) | ✅ 已验证 | `brew install agent-browser` |
| macOS (npm) | ✅ 已验证 | `npm install -g agent-browser && agent-browser install` |
| Linux (npm) | 应可运行 | 作者尚未测试 |
| Windows (npm) | 应可运行 | 作者尚未测试 |
| Chrome 145+ | ✅ 推荐 | 通过 `chrome://inspect` 免重启启用 CDP |
| Chrome < 145 | 支持 | 需要 `--remote-debugging-port` 启动参数。[配置方法 →](docs/setup.md#older-chrome--145-or-custom-port) |
| Chromium 系（Edge、Brave） | 应可运行 | 通过 CDP，未测试 |

## 资源

- [agent-browser](https://github.com/vercel-labs/agent-browser) — CLI 工具
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) — 底层协议
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — 构建可复用 Skill

## 许可证

[MIT](LICENSE)
