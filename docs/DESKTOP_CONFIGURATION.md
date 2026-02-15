# 桌面运行时配置架构

World Monitor 桌面现在使用运行时配置模式，其中包含每个功能的切换和秘密支持的凭据。

## 密钥

桌面保管库架构支持服务和中继使用的以下必需密钥：

- `GROQ_API_KEY`
- `OPENROUTER_API_KEY`
- `FRED_API_KEY`
- `EIA_API_KEY`
- `CLOUDFLARE_API_TOKEN`
- `ACLED_ACCESS_TOKEN`
- `WINGBITS_API_KEY`
- `WS_RELAY_URL`
- `VITE_OPENSKY_RELAY_URL`
- `OPENSKY_CLIENT_ID`
- `OPENSKY_CLIENT_SECRET`
- `AISSTREAM_API_KEY`
- `VITE_WS_RELAY_URL`

## 特征架构

每个功能包括：

- `id`：稳定特征标识符。
- `requiredSecrets`：必须存在且有效的密钥列表。
- `enabled`：运行时设置面板中的用户切换状态。
- `available`：计算的 (`enabled && requiredSecrets valid`)。
- `fallback`：面向用户的降级行为描述。

## 桌面秘密存储

桌面构建通过 Rust `keyring` 条目（`world-monitor` 服务命名空间）支持的 Tauri 命令绑定将机密保留在操作系统凭证存储中。

前端**不将秘密存储在明文文件中**。

## 降级行为

如果所需的机密丢失/禁用：

- 总结：Groq/OpenRouter 已禁用，浏览器模型回退。
- FRED / EIA：经济和石油分析返回空状态。
- Cloudflare / ACLED：中断/冲突返回空状态。
- Wingbits：飞行丰富功能已禁用，启发式飞行分类仍然存在。
- AIS / OpenSky 中继：实时跟踪功能被彻底禁用。
