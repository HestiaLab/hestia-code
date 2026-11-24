# 构建与打包

本文只记录在本项目上**实际验证过**的步骤。上游 wiki 的步骤已过期（它仍建议 `npm`，而本仓库要求 `pnpm`）。

## 1. 环境准备

| 依赖 | 版本 | 获取方式 | 备注 |
| --- | --- | --- | --- |
| Node.js | **24.22.1** | `nvm install 24.22.1` | 必须与 `.nvmrc` 完全一致 |
| pnpm | **12.16.0** | `npm i -g pnpm@12.16.0` | 若 npm 提示忽略安装脚本，加 `--allow-scripts=pnpm` |
| Python | 3.8+ | 系统安装 | node-gyp 需要 |
| Visual C++ 生成工具 | VS 2019+ | Visual Studio Installer | 勾选"使用 C++ 的桌面开发"；VS 2026 已验证可用 |
| Inno Setup | 6.x | `winget install JRSoftware.InnoSetup` | 仅打包安装包需要 |
| Windows SDK | 10.x | 随 VS 安装 | 提供 `signtool.exe` |

切换版本后确认：

```bat
node -v
pnpm -v
```

## 2. 安装依赖

```bat
pnpm install
```

这一步会编译原生模块（`@parcel/watcher`、`@vscode/sqlite3`、`@vscode/windows-ca-certs` 等），首次约 7–8 分钟，占用约 3 GB。

> **国内网络注意**：`foundry-local-sdk` 的安装脚本会从 `api.nuget.org` / `packagefeedproxy.microsoft.io` 下载约 100 MB 的 ONNX 运行时，该下载在国内网络下常长时间无进展。本产品不使用本地模型推理，可跳过：
>
> ```bat
> set FOUNDRY_LOCAL_SKIP_INSTALL=1
> pnpm install
> ```
>
> 跳过只影响 agent-host 的本地模型能力，不影响编译、启动与打包。

**不要用 `npm install`**：`preinstall` 会校验 Node 版本并依赖 pnpm 传入的 `npm_config_node_gyp`，用 npm 会得到 `Run this script with pnpm run preinstall` 之类的失败。

## 3. 编译

```bat
pnpm run compile-client
```

本仓库不包含 Copilot 扩展（见 `PATCHES.md` 第 4 节），因此使用 `compile-client` 而不是 `compile`。

## 4. 开发运行

```bat
.\scripts\code.bat
```

首次运行会下载 Electron 到 `.build/electron`。若网络受限导致长时间无进展：

```bat
set ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/
.\scripts\code.bat
```

启动成功的标志：窗口标题出现 `Hestia Dev`。

## 5. 打包应用

```bat
set PATH=C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64;%PATH%
pnpm run gulp vscode-win32-x64
```

- 产物目录：`..\VSCode-win32-x64`（与仓库同级）
- 主程序：`Hestia.exe`

`signtool.exe` 必须可达，因为打包会用 `rcedit` 改写原生依赖的版本信息，改写前需先剥离已有签名。

## 6. 打包安装包

顺序固定：

```bat
pnpm run gulp vscode-win32-x64-inno-updater
pnpm run gulp vscode-win32-x64-user-setup
```

- 第一条把 `inno_updater.exe`、`vcruntime140.dll` 复制到 `VSCode-win32-x64\tools`
- 第二条调用 Inno Setup 编译 `build/win32/code.iss`

产物：`.build\win32-x64\user-setup\VSCodeSetup.exe`

> 安装包文件名沿用 `VSCodeSetup.exe`，来自 `gulpfile.vscode.win32.ts` 的固定命名。若要改名，需同时修改该文件与发布流程。

静默安装（用于验证）：

```bat
start /wait "" ".build\win32-x64\user-setup\VSCodeSetup.exe" /VERYSILENT /SUPPRESSMSGBOXES /NORESTART /MERGETASKS=!runcode /DIR="%LOCALAPPDATA%\Programs\HestiaTest"
```

## 7. 常见失败与处理
| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `preinstall: ... Node.js version ... required` | Node 版本不符 | `nvm use 24.22.1` |
| `Run this script with pnpm run preinstall` | 用了 npm | 改用 pnpm |
| Electron 下载卡在 0 字节 | 网络 | 设置 `ELECTRON_MIRROR` |
| `spawn signtool.exe ENOENT` | SDK 不在 PATH | 把 `...\Windows Kits\10\bin\<ver>\x64` 加到 PATH |
| `No files found matching "...\VSCode-win32-x64\tools\*"` | 未先跑 `inno-updater` | 按第 6 节顺序执行 |
| 打包卡在 `prepareBuiltInCopilotRipgrepShim` | 误恢复了 Copilot 无条件打包 | 检查 `gulpfile.vscode.ts` 的条件判断 |
| `git` 报 `shallow` 相关错误 | 仓库是浅克隆（本项目为快照） | 属预期，不要执行 `--unshallow` |
| `pnpm install` 长时间停在 `foundry-local-sdk install` | ONNX 运行时下载被墙 | `set FOUNDRY_LOCAL_SKIP_INSTALL=1` 后重试 |
| 启动时报 `builtInExtensionsEnabledWithAutoUpdates is not iterable` | 误删了 `product.json` 必填字段 | 恢复为 `[]`，见 `PATCHES.md` 第 1 节 |

## 8. 验证清单

改动涉及构建、`product.json`、品牌资源或核心源码时，至少完成：

```bat
pnpm run compile-client
.\scripts\code.bat           :: 窗口能打开，标题为 Hestia Dev
pnpm run gulp vscode-win32-x64
pnpm run gulp vscode-win32-x64-inno-updater
pnpm run gulp vscode-win32-x64-user-setup
```

必要时追加：静默安装 → 启动 → 卸载，确认产品名、图标与卸载记录正确。

针对性单元测试：

```bat
.\scripts\test.bat --reporter dot --run <测试文件路径>
```
