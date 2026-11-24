# Hestia

<p align="center">
  <img alt="Hestia" src="resources/win32/code_150x150.png" width="150">
</p>

Hestia 是一款独立的桌面 AI 编辑器，基于 [Visual Studio Code 的 Code OSS 源码](https://github.com/microsoft/vscode)为构建。

## 项目状态

当前不提供：扩展市场、自动更新、代码签名、账号体系。

## 基线

| 项 | 值 |
| --- | --- |
| 上游仓库 | `github.com/microsoft/vscode` |
| 标签 | `1.147.0` |
| 提交 | `8a14f0cc1b08d531eb83255ea3764ae22aa932b5` |

本仓库不包含上游 git 历史，也不与上游合并。如需升级基线，流程见 [docs/PROVENANCE.md](docs/PROVENANCE.md)。

本项目相对基线的全部改动记录在 [PATCHES.md](PATCHES.md)。

## 开发

规则入口：[AGENTS.md](AGENTS.md)。构建与排错：[docs/BUILD.md](docs/BUILD.md)。

```bat
nvm use 24.22.1
pnpm install
pnpm run compile-client
.\scripts\code.bat
```

环境要求（Node 必须精确为 `24.22.1`，只能用 pnpm）与打包顺序见 `docs/BUILD.md`。

## 已知限制

- 安装包未做代码签名，安装时会出现 SmartScreen 提示。
- 无自动更新，升级需手动安装新版本。
- 无扩展市场，仅内置固定扩展集。
- 上游安全与缺陷修复不会自动合入，需按 `docs/PROVENANCE.md` 手工重新基线。

## 许可与致谢

本项目基于 [microsoft/vscode](https://github.com/microsoft/vscode) 的 `1.147.0` 源码派生，遵循 [MIT 许可](LICENSE.txt)。

上游代码版权归 Microsoft Corporation 所有：

```txt
Copyright (c) 2015 - present Microsoft Corporation
```
