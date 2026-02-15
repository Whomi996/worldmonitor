# WorldMonitor 开发笔记

## 🤖 模型偏好（2026 年 1 月 30 日）

**对于 WorldMonitor 中的所有编码任务，始终使用：**

|任务|型号|别名 |
|------|-------|-------|
| **编码** | `openrouter/anthropic/claude-sonnet-4-5` | `sonnet` |
| **编码** | `openai/gpt-5-2` | `codex` |

**永远不要默认使用 MiniMax 来执行编码任务。**

**如何使用首选模型运行：**
```bash
# Sonnet for coding
clawdbot --model openrouter/anthropic/claude-sonnet-4-5 "build me..."

# Codex for coding  
clawdbot --model openai/gpt-5-2 "build me..."
```

**设置为默认值：**
```bash
export CLAUDE_MODEL=openrouter/anthropic/claude-sonnet-4-5
```

## 关键：Git 分支规则

**未经明确的用户许可，切勿合并或推送到不同的分支。**

- 如果在 `beta` 上，则仅推送到 `beta` - 切勿在没有询问的情况下合并到 `main`
- 如果在 `main` 上，请留在 `main` 上 - 切勿在没有询问的情况下切换分支和推送
- 切勿在没有明确请求的情况下合并分支
- 继续工作时，提交后推送到当前分支是可以的

## 关键：RSS 代理白名单

在 `src/config/feeds.ts` 中添加新的 RSS 源时，您**必须**也将源域添加到 `api/rss-proxy.js` 中的允许列表中。

### Why
RSS 代理具有安全允许列表 (`ALLOWED_DOMAINS`)，可阻止对未明确列出的域的请求。来自未列出域的源将返回 HTTP 403“域不允许”错误。

### 如何添加新提要

1. 将 feed 添加到 `src/config/feeds.ts`
2. 从源 URL 中提取域（例如，`https://www.ycombinator.com/blog/rss/` → `www.ycombinator.com`）
3. 将域添加到 `api/rss-proxy.js` 中的 `ALLOWED_DOMAINS` 数组
4. 将更改部署到 Vercel

### Example
```javascript
// In api/rss-proxy.js
const ALLOWED_DOMAINS = [
  // ... existing domains
  'www.ycombinator.com',  // Add new domain here
];
```

### 调试提要问题
如果面板显示“没有可用新闻”：
1.打开浏览器DevTools→Console
2. 查找 `HTTP 403` 或“域不允许”错误
3. 检查域名是否在 `api/rss-proxy.js` 白名单中

## 网站变体

由 `VITE_VARIANT` 环境变量控制的两个变体：

- `full`（默认）：地缘政治焦点 - worldmonitor.app
- `tech`：技术/初创公司焦点 -startups.worldmonitor.app

### 本地运行
```bash
npm run dev        # Full variant
npm run dev:tech   # Tech variant
```

### Building
```bash
npm run build:full  # Production build for worldmonitor.app
npm run build:tech  # Production build for startups.worldmonitor.app
```

## 定制饲料刮刀

有些来源不提供 RSS 源。自定义刮刀位于 `/api/` 中：

|端点|来源 |笔记|
|----------|--------|-------|
| `/api/fwdstart` | FwdStart 时事通讯 (Beehiiv) |抓取存档页面，30 分钟缓存 |

### 添加新的抓取工具
1.创建`/api/source-name.js`边缘函数
2.抓取源码，返回RSS XML格式
3. 添加到 feeds.ts：`{ name: 'Source', url: '/api/source-name' }`
4.无需添加到rss-proxy白名单（直接API，不代理）

## AI 总结和缓存

AI Insights 面板使用服务器端 Redis 缓存来消除用户之间的重复 API 调用。

### 所需的环境变量

```bash
# Groq API (primary summarization)
GROQ_API_KEY=gsk_xxx

# OpenRouter API (fallback)
OPENROUTER_API_KEY=sk-or-xxx

# Upstash Redis (cross-user caching)
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxx
```

### 它是如何运作的

1. 用户访问 → `/api/groq-summarize` 获得头条新闻
2. 服务器哈希头条 → 检查 Redis 缓存
3. **缓存命中** → 立即返回（无API调用）
4. **缓存未命中** → 调用 Groq API → 存储在 Redis 中（24h TTL） → 返回

### 型号选择

- **llama-3.1-8b-instant**：14,400 请求/天（用于摘要）
- **llama-3.3-70b-versatile**：1,000 个请求/天（质量但有限）

### 后备链

1. Groq（快速，14.4K/天）→ Redis 缓存
2.OpenRouter（50/天）→Redis缓存
3.浏览器T5（无限制，较慢，无缓存）

### 设置 Upstash

1. 在 [upstash.com](https://upstash.com) 创建免费帐户
2.创建新的Redis数据库
3. 将 REST URL 和令牌复制到 Vercel 环境变量

## 服务状态面板

`api/service-status.js` 中的状态页面 URL 必须与实际状态页面端点匹配。常见格式：
- Statuspage.io：`https://status.example.com/api/v2/status.json`
- Atlassian：`https://example.status.atlassian.com/api/v2/status.json`
- event.io：相同端点但返回 HTML，由 `incidentio` 解析器处理

目前已知的网址：
- 人类：`https://status.claude.com/api/v2/status.json`
- 缩放：`https://www.zoomstatus.com/api/v2/status.json`
- 概念：`https://www.notion-status.com/api/v2/status.json`

## 允许的 Bash 命令

无需用户批准即可使用以下附加 bash 命令：
- `Bash(ps aux:*)` - 列出正在运行的进程
- `Bash(grep:*)` - 搜索文本模式
- `Bash(ls:*)` - 列出目录内容

## Bash 指南

### 重要提示：避免导致输出缓冲问题的命令
- 监视或检查命令输出时，请勿通过 `head`、`tail`、`less` 或 `more` 管道输出
- 不要使用 `| head -n X` 或 `| tail -n X` 截断输出 - 这些会导致缓冲问题
- 相反，让命令完全完成，或者使用 `--max-lines` 标志（如果命令支持）
- 对于日志监控，更喜欢直接读取文件而不是通过过滤器进行管道传输

### 检查命令输出时：
- 尽可能不使用管道直接运行命令
- 如果需要限制输出，请使用特定于命令的标志（例如，`git log -n 10` 而不是 `git log | head -10`）
- 避免可能导致输出无限期缓冲的链式管道
