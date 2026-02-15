# 本地后端奇偶校验矩阵（桌面 sidecar）

该矩阵通过将 `src/services/*.ts` 消费者映射到 `api/*.js` 处理程序并将每个功能分类为：

- **完全本地**：无需用户凭据即可在桌面 sidecar 上工作。
- **需要用户提供的 API 密钥**：本地端点存在，但功能取决于配置的机密。
- **需要云回退**：sidecar 存在，但操作行为取决于云中继路径。

## 优先关闭顺序

1. **优先级1（核心面板+地图）：** LiveNewsPanel、MonitorPanel、StrategicRiskPanel、关键地图图层。
2. **优先级 2（情报连续性）：** 摘要和市场小组。
3. **优先级 3（增强）：** 丰富和依赖于中继的跟踪附加功能。

## 特征奇偶校验矩阵

|优先|功能/面板 |服务来源 (`src/services/*.ts`) | API 路线 | API 处理程序 (`api/*.js`) |分类|关闭状态 |
|---|---|---|---|---|---|---|
| P1 |实时新闻面板 | `src/services/live-news.ts` | `/api/youtube/live` | `api/youtube/live.js` |完全本地化 | ✅ 可用本地端点；频道级视频回退已经实现。 |
| P1 |监控面板| _无（面板本地关键字匹配）_ | _无_ | _无_ |完全本地化 | ✅ 仅客户端（无后端依赖）。 |
| P1 | StrategyRiskPanel 缓存覆盖 | `src/services/cached-risk-scores.ts` | `/api/risk-scores` | `api/risk-scores.js` |需要用户提供的 API 密钥 | ✅ 显式回退：当缓存源不可用时，面板继续进行本地聚合评分。 |
| P1 |地图图层（冲突、中断、AIS、军事航班）| `src/services/conflicts.ts`、`src/services/outages.ts`、`src/services/ais.ts`、`src/services/military-flights.ts` | `/api/acled-conflict`、`/api/cloudflare-outages`、`/api/ais-snapshot`、`/api/opensky` | `api/acled-conflict.js`、`api/cloudflare-outages.js`、`api/ais-snapshot.js`、`api/opensky.js` |需要用户提供的 API 密钥 | ✅ 显式回退：禁用不可用的提要，同时本地/静态图层的地图渲染保持活动状态。 |
| P2 |总结 | `src/services/summarization.ts` | `/api/groq-summarize`、`/api/openrouter-summarize` | `api/groq-summarize.js`、`api/openrouter-summarize.js` |需要用户提供的 API 密钥 | ✅ 显式后备链：Groq → OpenRouter → 浏览器模型。 |
| P2 |市场面板| `src/services/markets.ts`、`src/services/polymarket.ts` | `/api/coingecko`、`/api/polymarket`、`/api/finnhub`、`/api/yahoo-finance` | `api/coingecko.js`、`api/polymarket.js`、`api/finnhub.js`、`api/yahoo-finance.js` |完全本地化 | ✅ 在 sidecar 模式下维护多提供商和缓存感知的获取行为。 |
| P3 |飞行丰富 | `src/services/wingbits.ts` | `/api/wingbits` | `api/wingbits/[[...path]].js` |需要用户提供的 API 密钥 | ✅ 显式后备：仅启发式分类模式。 |
| P3 | OpenSky 中继后备路径 | `src/services/military-flights.ts` | `/api/opensky` | `api/opensky.js` |需要云后备 | ✅ 记录中继回退；当继电器不可用时，不会出现硬故障。 |

## 非奇偶校验关闭动作完成

- 在 `ServiceStatusPanel` 中添加了**桌面就绪状态 + 非奇偶校验回退可见性**，以便操作员可以在桌面运行时查看接受状态和每个功能的回退行为。
- 将本地 sidecar 策略保留为默认路径：桌面 sidecar 在本地执行 `api/*.js` 处理程序，并且仅在处理程序执行或中继路径失败时使用云回退。

## 桌面就绪验收标准

当以下所有检查均呈绿色时，桌面构建被视为**准备就绪**：

1. **启动：** 应用程序启动并启用本地 sidecar 运行状况报告。
2. **地图渲染：** 即使可选提要不可用，地图也会加载本地/静态图层。
3. **核心情报面板：** LiveNewsPanel、MonitorPanel、StrategicRiskPanel 渲染无致命错误。
4. **摘要：** 至少有一个摘要路径有效（提供商支持或浏览器后备）。
5. **市场面板：**面板呈现并返回来自至少一个市场提供商的数据。
6. **实时跟踪：**至少有一种实时模式（AIS 或 OpenSky）可用。

这些检查现在在服务状态 UI 中显示为“桌面就绪情况”。
