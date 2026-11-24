# 与基线快照的差异

本仓库基于 `microsoft/vscode` 标签 `1.147.0` 的快照（见 `docs/PROVENANCE.md`），不保留上游历史，也不与上游合并。本文件记录**本项目相对该快照的全部有意差异**，用途有三：

- 让人清楚哪些行为是 Hestia 自己的选择，而不是上游缺陷。
- 重新基线时作为重放清单。
- 避免后续改动误以为"这里本来就该这样"。

维护规则：

- 每个逻辑改动一条记录，写明涉及文件与原因。
- 变更行为时在同一个提交里更新本文件。
- 改动被废弃时删除该条，而不是留下过期说明。

## 1. 产品标识（`product.json`）

- 产品改名为 `Hestia`（`nameShort`/`nameLong`），`applicationName` 为 `hestia`，数据目录为 `.hestia` / `.hestia-shared`，`win32DirName`/`win32NameVersion` 为 `Hestia`，`win32RegValueName` 为 `Hestia`，`win32AppUserModelId` 为 `HestiaLab.Hestia`，`win32ShellNameShort` 为 `H&estia`，tunnel/server 应用名同步改名，`urlProtocol` 为 `hestia`。
- 为 Hestia 分配独立的 Inno Setup 应用 ID（`win32x64AppId`、`win32x64UserAppId`），使其可与 VS Code、Code OSS 并存安装。
- `licenseUrl`、`serverLicenseUrl`、`reportIssueUrl` 指向本项目。
- 将 `builtInExtensionsEnabledWithAutoUpdates` 设为空数组，而不是删除。该字段在 `IProductConfiguration` 中是**必填**的 `readonly string[]`，删除后 `extensionsScannerService` 等处在遍历时抛 `is not iterable`。
- 删除 `defaultChatAgent`、`trustedExtensionAuthAccess`、`voiceWsUrl`、`webviewContentExternalBaseUrlTemplate`。这些字段引用了微软服务、GitHub Copilot 扩展或微软 CDN，且在类型上是可选或在使用处已做空值判断。

原因：重新命名的产品不应继承其他厂商的身份、遥测端点与 CDN。刻意不提供 `webviewContentExternalBaseUrlTemplate`，让 workbench 回退到本地 `vscode-webview://` 端点，而不是 `*.vscode-cdn.net`。

## 2. Windows 品牌资源（`resources/win32/**`）

- `code.ico`、`default.ico`、`sessions.ico`、`code_150x150.png`、`code_70x70.png`。
- Inno Setup 向导位图（`inno-big-*.bmp`、`inno-small-*.bmp`）。
- `VisualElementsManifest.xml`：`ShortDisplayName` 改为 `Hestia`。
- `appx/AppxManifest.xml`：`PublisherDisplayName` 改为 `HestiaLab`。

沿用上游文件名，打包脚本与 `code.iss` 无需改动。替换内容时不要改名。

## 3. 安装器文案与厂商元数据

- `build/win32/code.iss`：`AppPublisher` 改为 `HestiaLab`，发布/支持/更新 URL 指向本项目，"请改用 System 安装包"的提示不再链接到 `code.visualstudio.com`。
- `build/gulpfile.vscode.ts`、`build/gulpfile.reh.ts`：原生依赖的 Windows 二进制元数据中 `CompanyName` 改为 `HestiaLab`，`LegalCopyright` 改为 `Copyright (C) 2026 HestiaLab`。
- `build/lib/electron.ts`：Electron 打包配置的 `companyName` 改为 `HestiaLab`、`copyright` 改为 `Copyright (C) 2026 HestiaLab`。这一项决定 `Hestia.exe` 自身的文件属性，漏改会导致 exe 属性里仍显示 Microsoft Corporation。

## 4. 构建不打包 Copilot

- `build/gulpfile.vscode.ts`：仅当 `product.defaultChatAgent` 存在时才执行 `prepareCopilotRipgrepShimTask` 与 `compileCopilotExtensionBuildTask`。

原因：本产品不配置默认 Chat Agent，Copilot VSIX 及其原生运行时 shim 没有消费者。无条件构建会让打包依赖下载 `@github/copilot-*` 压缩包，随产品分发则等于转发了本项目不使用的专有扩展。

## 5. workbench 在缺少默认 Chat Agent 时不崩溃

以下位置原本都假定 `product.defaultChatAgent` 一定存在；删除该字段后逐一改为降级处理。

- `src/vs/workbench/contrib/welcomeOnboarding/browser/onboardingVariationA.ts`：删除模块级 `assertDefined`（它把产品配置差异变成加载 workbench 包时的未捕获异常）；`show()` 在未配置 Agent 时直接返回。
- `src/vs/workbench/contrib/welcomeOnboarding/test/browser/onboardingVariationA.test.ts`：跳过该套件。该套件验证 AI 登录向导，缺少默认 Chat Agent 时界面无法渲染。
- `src/vs/workbench/contrib/welcomeGettingStarted/browser/startupPage.ts`：`tryShowOnboarding` 在未配置默认 Chat Agent 时提前返回。
- `src/vs/workbench/services/accounts/browser/defaultAccount.ts`：`toDefaultAccountConfig` 接受 `undefined` 并返回休眠配置，而不是在服务构造期解引用空值。

**教训**：删除 `product.json` 的字段必须全仓搜索引用，并逐一确认降级行为。三处崩溃与一处 `is not iterable` 都只有在真机启动时才暴露。判定标准：先看 `src/vs/base/common/product.ts` 里该字段是否必填——必填字段只能用空值（`[]` / `{}`）替代，不能删除。

## 6. 文档

- `README.md`：替换上游 README，说明 Hestia 定位、验证状态、构建步骤、已知限制与许可致谢。保留 Microsoft 的 MIT 版权声明，并声明无隶属关系。
- `AGENTS.md`：本项目常驻规则（项目身份、环境与命令、语言约定、改动边界、验证纪律）。
- `docs/BUILD.md`、`docs/PROVENANCE.md`：构建与基线记录。
- `.github/copilot-instructions.md`：保留上游内容（对本快照仍然准确），追加一节指向 `AGENTS.md`。

## 已知缺口

- 打包任务仍会把 CLI 的 `copilot-shim` 复制到 `bin/`。无 Chat Agent 时它是惰性的，但应从打包任务中移除。
- 词典与扩展推荐仍使用上游默认 URL，公开发布前需要复核。
- 安装包文件名仍为 `VSCodeSetup.exe`（`gulpfile.vscode.win32.ts` 中固定），未改为 Hestia 命名。
- 安装包未做代码签名，安装时会出现 SmartScreen 提示。
- 不提供自动更新与扩展市场。
