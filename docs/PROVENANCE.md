# 基线记录与重新基线

本仓库是 `microsoft/vscode` 的**快照派生**：内容取自某个发布标签，但不保留上游 git 历史，也不与上游合并。本文记录基线信息与将来升级基线的手工流程。

## 基线

| 项 | 值 |
| --- | --- |
| 上游仓库 | `https://github.com/microsoft/vscode` |
| 标签 | `1.147.0` |
| 提交 | `8a14f0cc1b08d531eb83255ea3764ae22aa932b5` |
| 快照日期 | 2026-11-24 |
| 获取方式 | `git clone --depth 1 --branch 1.147.0 --single-branch`，随后重建 `.git` |
| 上游许可 | MIT，版权归 Microsoft Corporation |

初始化方式（已完成，留作记录）：

```bat
git clone --depth 1 --branch 1.147.0 --single-branch https://github.com/microsoft/vscode.git .
rmdir /s /q .git
git init
:: 放入本项目自有文件后提交
```

## 这个选择的后果

**收益**：仓库完全属于本项目，提交历史只包含 Hestia 的工作，不受上游噪音影响。

**代价**（必须清楚，不是待办）：

1. 无法 `merge` 或 `rebase` 上游。新增的上游安全修复与缺陷修复**不会**自动进入本仓库。
2. 无法使用 `git blame` 追溯上游代码的原始意图。
3. 升级到新标签只能手工重放差异，见下节。

## 重新基线流程

以下流程**尚未在本项目执行过**，首次执行时应把它当作实验并记录实际耗时与问题。

### 1. 取得新快照

```bat
:: 新目录，不要覆盖当前工作区
git clone --depth 1 --branch <新标签> --single-branch https://github.com/microsoft/vscode.git C:\temp\vscode-<新标签>
```

### 2. 生成差异清单

`PATCHES.md` 列出了本项目全部有意改动。用 git 直接比较两棵树，验证清单是否完整：

```bat
git --no-index diff --stat C:\temp\vscode-<旧标签> C:\temp\vscode-<新标签>
```

再以本项目工作区为基准，确认每个已登记改动在新标签中是否仍然成立：

```bat
git --no-index diff C:\temp\vscode-<新标签>\<路径> <本项目>\<路径>
```

### 3. 在新分支上重放

- 从新快照建立分支，逐条应用 `PATCHES.md` 中的改动。
- 每应用一条就编译一次，不要攒到最后。
- 上游可能已变更相关实现（例如 `defaultChatAgent` 的使用点），必须重新确认降级逻辑。

### 4. 验证

按 `docs/BUILD.md` 第 8 节执行完整验证链，尤其是打包与静默安装。

### 5. 更新记录

- 更新本文的基线表格（标签、提交、快照日期）。
- 更新 `PATCHES.md`，删除已失效条目，新增本次发现的分歧。

## 风险

- 手工重放容易遗漏。以 `git --no-index diff` 的输出为准，不要只依赖清单。
- 上游若重构了被改动文件的周边结构，改动可能"编译通过但行为错误"。必须执行真机启动验证。
