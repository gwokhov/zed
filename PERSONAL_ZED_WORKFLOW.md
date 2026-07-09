# 个人 Zed Fork 维护规则

本文档记录这个 fork 的长期维护方式：保留官方 Zed 更新，同时长期携带 `#59025`，并允许后续叠加个人功能。

## 远端

- `origin` 指向个人 fork：`git@github.com:gwokhov/zed.git`。
- `upstream` 指向官方仓库：`https://github.com/zed-industries/zed.git`。
- `upstream` 只用于 fetch，不向官方仓库 push。

检查：

```sh
git remote -v
```

## 分支职责

- `main`
  - 只跟踪 `upstream/main`。
  - 不在这里写个人功能，不直接提交自定义代码。
  - 用来接收官方 Zed 新版本功能。

- `codex/pr-59025-original`
  - 保存 `#59025` 的原始 PR head。
  - 只作为备份和对照，不在这里开发。

- `feature/git-panel-staged-unstaged-sections`
  - 维护 `#59025` 功能 patch。
  - 当官方 `main` 更新后，优先把这个分支 rebase 到最新 `main`。
  - 只解决 `#59025` 与官方代码的冲突，不混入其他个人功能。

- `personal/current`
  - 长期自用集成分支。
  - 打包、日常使用、叠加个人功能都基于这个分支。
  - 可以合入 `feature/git-panel-staged-unstaged-sections` 和其他个人功能分支。

- `feature/my-xxx`
  - 后续每个个人功能单独开分支。
  - 做完后再合入 `personal/current`。

## 更新官方 Zed

先更新官方基线：

```sh
git fetch upstream
git switch main
git merge --ff-only upstream/main
```

再更新 `#59025` patch 分支：

```sh
git switch feature/git-panel-staged-unstaged-sections
git rebase main
```

如果有冲突，只解决 `#59025` 与当前官方代码的冲突。解决后至少运行：

```sh
cargo check -p git_ui -p project -p collab -p proto
```

最后更新自用集成分支：

```sh
git switch personal/current
git rebase main
git merge --no-ff feature/git-panel-staged-unstaged-sections
```

如果 `personal/current` 上还有其他个人功能分支，也逐个合入。

## 开发个人功能

从 `personal/current` 切分支：

```sh
git switch personal/current
git switch -c feature/my-feature
```

完成后合回：

```sh
git switch personal/current
git merge --no-ff feature/my-feature
```

## 打包规则

- 从 `personal/current` 打包。
- 优先使用 Zed 官方打包脚本。
- 自用包可以接受 ad-hoc 签名。
- 如果要公开分发，需要 Developer ID 签名和 notarization。

Apple Silicon 自用 DMG 的入口仍是：

```sh
./script/bundle-mac aarch64-apple-darwin
```

## 提交规则

- 不主动在 `main` 提交。
- 不主动向 `upstream` push。
- 提交信息使用仓库约定格式，例如：

```text
fix(git_ui): 适配 staged unstaged 分区变基
```

## 当前基线

- `#59025` 功能分支：`feature/git-panel-staged-unstaged-sections`
- 长期自用分支：`personal/current`
- 当前 `#59025` rebase 修复提交：`1f455e839e`
- 已验证：

```sh
cargo check -p git_ui -p project -p collab -p proto
```
