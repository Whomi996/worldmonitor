# Tauri 验证报告

## Scope
通过检查前端编译、TypeScript 完整性和 Tauri/Rust 构建执行，验证 World Monitor Tauri 应用程序的桌面构建准备情况。

## 桌面验证之前的预检检查

首先运行这些检查，以便快速对故障进行分类：

1. npm 注册表可达性
- `npm ping`
2. crates.io 稀疏索引可达性
- `curl -I https://index.crates.io/`
3. 网络需要时提供代理配置
- `env | grep -E '^(HTTP_PROXY|HTTPS_PROXY|NO_PROXY)='`

如果其中任何检查失败，请将下游桌面构建失败视为环境级别，直到修复网络路径。

## 命令运行

1. `npm ci` — 失败，因为环境阻止从 npm (`403 Forbidden`) 下载固定的 `@tauri-apps/cli` 包。
2. `npm run typecheck` — 成功。
3. `npm run build:full` — 成功（仅警告）。
4. `npm run desktop:build:full` — 在此环境中无法运行，因为 `npm ci` 失败，因此本地 `tauri` 二进制文件不可用（当发生这种情况时，桌面脚本现在会快速失败，并显示清晰的 `npm ci` 修复消息）。
5. `cargo check`（来自 `src-tauri/`）— 失败，因为环境阻止从 `https://index.crates.io` (`403 CONNECT tunnel failed`) 下载包。

## Assessment
- Web 应用程序部分编译成功。
- 本次运行中的完整 Tauri 桌面验证被 **外部环境中断/限制** 阻止（注册表访问被 HTTP 403 拒绝）。
- 在此验证过程中，在项目源中没有观察到代码/运行时缺陷。

## 未来 QA 的失败分类

在未来的报告中使用这些标签，以便结果具有可操作性：

1. **外部环境中断**
- 症状：npm/crates 注册表请求因传输/身份验证/网络错误（403/5xx/超时/DNS/代理）而失败，与存储库状态无关。
- 操作：在健康的网络中重试或修复凭据/代理/镜像可用性。

2. **预期失败：未配置离线模式**
- 症状：有意在没有互联网的情况下运行构建，但缺少所需的离线输入（对于 Rust：没有 `vendor/` 工件、没有内部镜像映射或未启用离线覆盖；对于 JS：没有准备好的包缓存）。
- 操作：首先配置脱机工件/镜像配置，启用脱机覆盖（`config.local.toml` 或 CLI `--config`），然后重新运行。

## 下一步是验证桌面端对端
选择一个支持的路径：

- 在线路径：
- `npm ci`
- `npm run desktop:build:full`

- 受限网络路径：
- 恢复预构建的离线工件（包括 `src-tauri/vendor/` 或内部镜像映射）。
- 使用映射到供应商/内部源的 `source.crates-io.replace-with` 和 `--offline` （如果适用）运行 Cargo。

在 `npm ci` 之后，桌面构建使用本地 `tauri` 二进制文件，并且不依赖于运行时 `npx` 包检索。

## 受限环境的修复选项

如果预检失败，请使用以下经批准的补救措施之一：

- 为 Node 包配置内部 npm 镜像/代理。
- 为 Rust 箱子配置内部 Cargo 注册表/稀疏索引镜像。
- 预供应商 Rust 箱子 (`src-tauri/vendor/`) 并在离线模式下运行 Cargo。
- 使用 CI 运行程序在构建之前从受信任的内部存储恢复包/缓存工件。

有关发布打包的详细信息，请参阅 `docs/RELEASE_PACKAGING.md`（部分：**网络预检和修复**）。
