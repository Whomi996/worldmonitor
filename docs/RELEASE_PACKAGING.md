# 桌面发布打包指南（本地、可复制）

本指南为两种桌面变体提供了可重复的本地打包步骤：

- **完整** → `World Monitor`
- **技术** → `Tech Monitor`

变体身份由 Tauri 配置控制：

- 完整：`src-tauri/tauri.conf.json`
- 技术：`src-tauri/tauri.tech.conf.json`

## 先决条件

- Node.js + npm
- Rust 工具链
- 操作系统本机 Tauri 构建先决条件：
- macOS：Xcode 命令行工具
- Windows：Visual Studio 构建工具 + NSIS + WiX

安装依赖项（这还会安装桌面脚本使用的固定 Tauri CLI）：

```bash
npm ci
```

所有桌面脚本从 `node_modules/.bin` 调用本地 `tauri` 二进制文件； `npm ci` 之后不需要下载运行时 `npx` 包。
如果本地 CLI 丢失，`scripts/desktop-package.mjs` 现在会快速失败，并显示显式 `npm ci` 修复消息。

## 网络预检和修复

在 CI 或托管网络中运行桌面打包之前，请验证连接和代理配置：

```bash
npm ping
curl -I https://index.crates.io/
env | grep -E '^(HTTP_PROXY|HTTPS_PROXY|NO_PROXY)='
```

如果这些方法失败，请使用支持的补救措施之一：

- 内部 npm 镜像/代理。
- 内部货物稀疏索引/注册表镜像。
- 预售 Rust 板条箱 (`src-tauri/vendor/`) + 货物离线模式。
- CI 工件/缓存策略，在构建之前恢复所需的包输入。

请参阅 `docs/TAURI_VALIDATION_REPORT.md` 了解故障分类标签和故障排除流程。

## 打包命令

查看脚本用法/帮助：

```bash
npm run desktop:package -- --help
```

### macOS (`.app` + `.dmg`)

```bash
npm run desktop:package:macos:full
npm run desktop:package:macos:tech
# or generic runner
npm run desktop:package -- --os macos --variant full
```

### Windows (`.exe` + `.msi`)

```bash
npm run desktop:package:windows:full
npm run desktop:package:windows:tech
# or generic runner
npm run desktop:package -- --os windows --variant tech
```

Bundler 目标固定在 Tauri 配置中，并通过打包脚本强制执行：

- macOS：`app,dmg`
- 窗户：`nsis,msi`


## Rust 依赖模式（在线 vs 受限网络）

从`src-tauri/`开始，该项目支持两种打包路径：

### 1) 标准在线构建（默认）

使用正常的 Cargo 行为 (crates.io)：

```bash
cd src-tauri
cargo generate-lockfile
cargo tauri build --config tauri.conf.json
```

### 2) 受限网络构建（预供应商或内部镜像）

可选的供应源在 `src-tauri/.cargo/config.toml` 中定义。要使用它，首先在具有注册表访问权限的计算机上准备供应商的 crate：

```bash
# from repository root
cargo vendor --manifest-path src-tauri/Cargo.toml src-tauri/vendor
```

然后使用任一方法启用离线模式：

- 一次性 CLI 覆盖（无文件更改）：

```bash
cd src-tauri
cargo generate-lockfile --offline --config 'source.crates-io.replace-with="vendored-sources"'
cargo tauri build --offline --config 'source.crates-io.replace-with="vendored-sources"' --config tauri.conf.json
```

- 本地覆盖文件（推荐用于 CI/可重复离线作业）：

```bash
cp src-tauri/.cargo/config.local.toml.example src-tauri/.cargo/config.local.toml
cd src-tauri
cargo generate-lockfile --offline
cargo tauri build --offline --config tauri.conf.json
```

对于 CI 或内部镜像，将 `src-tauri/vendor/` 作为工件发布并在受限网络构建之前恢复它。如果您的组织使用内部 crates 镜像而不是供应商，请将 `source.crates-io.replace-with` 指向 CI 特定 Cargo 配置中的该镜像并运行相同的构建命令。

## 可选的签名/公证挂钩

默认情况下，未签名的包装有效。

如果环境变量中存在签名凭据，Tauri 将在相同的打包命令期间自动签名/公证。

### macOS Apple 开发者签名+公证

打包前设置（开发者ID签名）：

```bash
export TAURI_BUNDLE_MACOS_SIGNING_IDENTITY="Developer ID Application: Your Company (TEAMID)"
export TAURI_BUNDLE_MACOS_PROVIDER_SHORT_NAME="TEAMID"
# optional alternate key accepted by Tauri tooling:
export APPLE_SIGNING_IDENTITY="Developer ID Application: Your Company (TEAMID)"
```

对于公证，选择一种身份验证方法：

```bash
# Apple ID + app-specific password
export APPLE_ID="you@example.com"
export APPLE_PASSWORD="app-specific-password"
export APPLE_TEAM_ID="TEAMID"

# OR App Store Connect API key
export APPLE_API_KEY="ABC123DEFG"
export APPLE_API_ISSUER="00000000-0000-0000-0000-000000000000"
export APPLE_API_KEY_PATH="$HOME/.keys/AuthKey_ABC123DEFG.p8"
```

然后运行标准或显式符号脚本别名：

```bash
npm run desktop:package:macos:full
# or
npm run desktop:package:macos:full:sign
```

### Windows Authenticode 签名

打包前设置（PowerShell）：

```powershell
$env:TAURI_BUNDLE_WINDOWS_CERTIFICATE_THUMBPRINT="<CERT_THUMBPRINT>"
$env:TAURI_BUNDLE_WINDOWS_TIMESTAMP_URL="https://timestamp.digicert.com"
# optional: if using cert file + password instead of cert store
$env:TAURI_BUNDLE_WINDOWS_CERTIFICATE="C:\path\to\codesign.pfx"
$env:TAURI_BUNDLE_WINDOWS_CERTIFICATE_PASSWORD="<PFX_PASSWORD>"
```

然后运行标准或显式符号脚本别名：

```powershell
npm run desktop:package:windows:full
# or
npm run desktop:package:windows:full:sign
```

## 变体感知输出（名称/图标）

- 完整版本：`World Monitor` / `world-monitor`
- 技术变体：`Tech Monitor` / `tech-monitor`

Tauri 中配置了不同的名称：

- `src-tauri/tauri.conf.json` → `World Monitor` / `world-monitor`
- `src-tauri/tauri.tech.conf.json` → `Tech Monitor` / `tech-monitor`

如果您想要特定于变体的图标，请在每个配置中单独设置 `bundle.icon` 并将每个变体指向专用图标资源。

## 输出位置

工件是在以下条件下生产的：

```text
src-tauri/target/release/bundle/
```

常用子文件夹：

- `app/` → macOS `.app`
- `dmg/` → macOS `.dmg`
- `nsis/` → Windows `.exe` 安装程序
- `msi/` → Windows `.msi` 安装程序

## 发布清单（清洁机器）

1. 构建所需的操作系统 + 变体包。
2. 将工件移动到干净的机器（或新的虚拟机）。
3. 安装/启动：
- macOS：安装 `.dmg`，将应用程序拖至应用程序，启动。
- Windows：运行 `.exe` 或 `.msi`，从“开始”菜单启动。
4. 验证启动：
- 应用程序窗口打开时不会崩溃。
- 地图视图渲染。
- 初始数据加载路径没有致命错误。
5. 验证变体身份：
- 窗口标题和产品名称与预期变体匹配。
6. 如果启用签名：
- 验证操作系统对话框/属性中的代码签名元数据。
- 验证 macOS 上的公证/Gatekeeper 接受情况。
