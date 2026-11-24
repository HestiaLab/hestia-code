# AGENTS.md

本文件是本仓库的常驻规则。任何人与 AI 助手在修改本仓库前都应先读完本文件；与 `.github/copilot-instructions.md` 冲突时，**以本文件为准**。

## 1. 项目身份

- **Hestia**（仓库 `hestia-code`）是一款独立的桌面 AI 编辑器，以 Visual Studio Code 的 Code OSS 源码为起点重新构建。
- 基线：`microsoft/vscode` 标签 **`1.147.0`**（commit `8a14f0cc1b08d531eb83255ea3764ae22aa932b5`）。
- 本仓库**不包含上游 git 历史**，是独立的提交起点。
- 上游代码以 MIT 许可使用，版权归 Microsoft Corporation。分发时必须保留 `LICENSE.txt` 及版权声明。
- Hestia 与 Microsoft 无隶属或背书关系，对外表述不得暗示这一点。

## 2. 上游策略（先读这一节）

本仓库是**快照派生**，与 `microsoft/vscode` 没有共同祖先：

- **不存在「同步上游」操作**。不要添加 `upstream` 远端，不要尝试 `merge` 或 `rebase` 上游分支。
- VS Code 的安全修复、缺陷修复**不会**自动进入本仓库。这是本项目的既定取舍，不是待办事项。
- 如确需升级到更新的上游标签，按 `docs/PROVENANCE.md` 的手工重新基线流程执行，并在独立分支上验证后才合并。

任何"顺手把上游某文件更新一下"的行为都属于越界改动。

## 3. 环境与命令（Windows x64，均已实测通过）

| 项 | 要求 | 不满足的后果 |
| --- | --- | --- |
| Node.js | **24.22.1**，与 `.nvmrc` 完全一致 | `preinstall` 直接失败 |
| pnpm | **12.16.0**，由 `package.json` 的 `packageManager` 固定 | 无 |
| Python | 3.8+ | node-gyp 无法编译原生模块 |
| Visual C++ 生成工具 | VS 2019 或更高（含"使用 C++ 的桌面开发"） | 原生模块编译失败 |
| Inno Setup | 6.x，仅打包安装包需要 | 无法产出安装包 |

```bat
nvm use 24.22.1
pnpm install
pnpm run compile-client
.\scripts\code.bat
```

打包顺序**不可颠倒**：

```bat
set PATH=C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64;%PATH%
pnpm run gulp vscode-win32-x64
pnpm run gulp vscode-win32-x64-inno-updater
pnpm run gulp vscode-win32-x64-user-setup
```

三条硬性约束（踩过坑，不要试错重来）：

1. **只能用 pnpm**。`npm install` 会破坏 `preinstall` 的类型检查并可能绕过 `packageManager` 固定版本。
2. **打包需要 `signtool.exe` 在 `PATH`**（Windows SDK）。缺失时报 `spawn signtool.exe ENOENT`。
3. **`inno-updater` 必须先于 `user-setup`**。否则 Inno Setup 因找不到 `VSCode-win32-x64\tools\*` 而中止。

首次下载 Electron 若卡住，设置镜像：

```bat
set ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/
```

详细说明与排错见 `docs/BUILD.md`。

## 4. 语言约定

团队以中文为主。

**用中文**

- 新增文档（本文件、`docs/**`、`PATCHES.md`、扩展 README）。
- 新增代码注释。前缀固定为 `// Hestia：`，便于检索与区分上游注释。
- 提交信息正文。主题行保持英文，便于与工具链和上游术语对齐。
- 中文段落不做固定宽度硬换行，一个自然段占一行。

**必须英文（硬约束）**

- 所有经 `nls.localize()` / `localize2()` 外部化的用户可见字符串。翻译流水线以英文为源文本，写成中文会让 zh-cn 变成源语言，其余语言全部失去源文本。中文界面应通过语言包 + 默认 locale 实现。
- 标识符：setting id、command id、配置键、API 名、context key。它们被外部引用，翻译等于破坏性变更。
- 被测试断言、构建脚本消费或 ESLint 规则匹配的字符串。
- 不翻译上游已有的注释，只在本项目新增或改写的行使用中文。

编码：提交进仓库的内容为 UTF-8 无 BOM、LF 换行。Windows 工作区出现 CRLF 属 `text=auto` 与 `core.autocrlf` 的正常行为；必须避免的是 CRLF 或 BOM 进入索引，用 `git ls-files --eol` 核对。

## 5. 改动边界

- **默认写扩展，不改核心**。能在 `extensions/` 或独立扩展里实现的能力，不要动 `src/vs/**`。
- 必须改核心时：只改必要的最小范围，并在 `PATCHES.md` 登记改动文件与原因。
- 不为了"顺手整理"修改与任务无关的代码；不做大规模重命名或格式化。
- 不引入新依赖，除非现有依赖无法满足且已评估长期成本。

## 6. 已知脆弱点

- `product.json` 已删除 `defaultChatAgent`、`trustedExtensionAuthAccess`、`voiceWsUrl`、`webviewContentExternalBaseUrlTemplate`，并将 `builtInExtensionsEnabledWithAutoUpdates` 设为空数组。
  **修改 `product.json` 字段前，先看 `src/vs/base/common/product.ts` 中该字段是否必填**：必填字段只能用空值替代，不能删除，否则运行时会抛错（例如删除 `builtInExtensionsEnabledWithAutoUpdates` 会导致 `is not iterable`）。删除后必须全仓搜索其全部引用并逐一确认降级行为——此前删除 `defaultChatAgent` 导致三处启动崩溃（见 `PATCHES.md` 第 5 节）。
- Copilot 扩展及其 ripgrep shim 的打包已按 `product.defaultChatAgent` 条件化。无该配置时不要恢复无条件打包，否则打包会依赖下载 `@github/copilot-*` 并转发本产品不使用的专有扩展。
- 品牌资源沿用上游文件名（`code.ico`、`inno-big-*.bmp` 等）。替换内容时保持文件名不变，否则打包脚本与 `code.iss` 需要同步改动。

## 7. 验证纪律

报告状态时严格区分以下五种，不允许混用：

- **已通过**：实际执行且结果满足预期。
- **已失败**：实际执行但结果不符合预期。
- **未执行**：没有运行。
- **无法执行**：因环境、权限或依赖限制无法运行。
- **结果不确定**：运行了但证据不足以判断。

代码阅读不等于测试通过；类型检查不等于运行正确；单元测试不等于真机可用。修改构建脚本、`product.json`、打包相关文件后，至少执行一次真实打包与启动。

无法执行完整验证时，说明已做的针对性检查与残余风险。

## 8. 不确定时

- 不确定某个 API 或配置项的行为：读源码、跑命令或查官方文档验证，不要凭记忆断言。
- 不确定改动范围：先读 `PATCHES.md` 与相关目录的既有实现，再动手。
- 不确定需求歧义是否影响实现：影响正确性或兼容性时先问，不影响时选可逆的默认方案并说明假设。

## 9. 目录地图

| 路径 | 归属 | 说明 |
| --- | --- | --- |
| `src/vs/` | 上游为主 | 核心源码。改动需登记 |
| `extensions/` | 上游为主 + 自有扩展 | 内置扩展；自有 AI 能力优先放这里 |
| `build/` | 上游 + 少量改动 | 构建脚本，改动需登记 |
| `resources/win32/` | 已替换 | Hestia 品牌资源 |
| `product.json` | 已修改 | 产品标识与能力开关 |
| `PATCHES.md` | 本项目 | 与基线的全部差异 |
| `docs/` | 本项目 | 构建、来源、决策记录 |

## 10. 文档索引

- `docs/BUILD.md`：环境准备、构建、打包、排错。
- `docs/PROVENANCE.md`：基线记录与重新基线流程。
- `PATCHES.md`：与上游基线的全部有意差异。
