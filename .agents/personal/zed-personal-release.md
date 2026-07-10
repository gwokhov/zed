# 个人 Zed 分支开发与发布流程

这份文档只用于本地个人 fork 维护。处理分支、提交、打包、release 前先阅读它；普通代码修改不需要强制套用这里的发布流程。

## 开发分支流程

常规目标是让 `personal/current` 保持为个人可用分支：以 `upstream/main` 为基础，叠加个人功能提交和选定的外部 PR，最后打包发布。

如果需要剔除某个已合入的旧 PR，优先从最新 `upstream/main` 重建 `personal/current`，再按顺序合并需要保留的个人功能分支和新的外部 PR。不要把旧远端 `personal/current` 直接 merge 回来，否则可能把要剔除的历史重新带回。

建议流程：

1. `git fetch upstream origin`
2. 在动 `personal/current` 前创建备份分支，例如 `backup/personal-current-before-<reason>-YYYYMMDD`
3. 从 `upstream/main` 重建或 checkout `personal/current`
4. 合并要保留的个人功能提交或功能分支
5. 合并选定的外部 PR 或 squash commit
6. 从 `personal/current` 切功能分支，例如 `feat/git-panel-staging-ui-fixes`
7. 在功能分支上小步修改，通过开发模式预览
8. 确认后提交功能分支，再快进合并回 `personal/current`

开发模式预览常用：

```sh
cargo run -- .
```

如果需要停止开发模式，优先向对应进程发送 `Ctrl-C`，确认没有残留 `cargo run` 或 `target/debug/zed` 进程。

推送 `personal/current` 时，如果远端还是重建前历史，普通 push 会非快进失败。确认差异后使用：

```sh
git push --force-with-lease origin personal/current
```

使用前必须确认：

- 本地 `personal/current` 是期望发布的完整状态
- 远端差异来自旧个人分支历史，而不是别人刚推的新工作
- 不需要把远端旧历史 merge 回来

## 提交规范

不要主动提交；只有在明确要求提交时才 commit。

提交格式：

```text
<type>(<scope>): <subject>
```

`type` 从以下值中选择：

```text
feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
```

规则：

- `scope` 使用最相关的模块名、目录名或功能名
- `subject` 使用简体中文
- `subject` 使用祈使句
- `subject` 不超过 72 个字符
- `subject` 不以句号结尾
- 复杂变更可以添加简短 body
- 破坏性变更使用 `BREAKING CHANGE:`

示例：

```text
fix(git_ui): 优化暂存分组操作交互
style(ui): 调整图标尺寸
feat(editor): 对齐 diff 行号变更标记
```

提交前检查：

```sh
git status --short --branch
git diff --check
```

如果只改了 Rust 文件，至少确认格式：

```sh
rustfmt --edition 2024 --check <file>
```

如果要新增或修改测试，先询问 G。

## 打包流程与坑点

打包前确认：

```sh
git status --short --branch
cat crates/zed/RELEASE_CHANNEL
rustc --version --verbose
git rev-parse --short=10 HEAD
```

个人 macOS release 包常用命令：

```sh
TERM=xterm-256color script/bundle-mac
```

产物路径：

```text
target/aarch64-apple-darwin/release/Zed-aarch64.dmg
```

本地 release 包可复制成带功能名和 commit 的文件名：

```text
Zed-Personal-<Feature>-<short-sha>-aarch64.dmg
```

例如：

```text
Zed-Personal-Staging-UI-f6cd7615-aarch64.dmg
```

### `cargo bundle` 颜色错误

在当前环境中，`script/bundle-mac` 可能在创建 app bundle 时失败：

```text
Error(Term(ColorOutOfRange), State { ... })
```

这不是 Zed 代码编译失败，而是 `cargo-bundle` 的终端颜色输出和当前 `$TERM` 环境不兼容。解决方式是明确设置：

```sh
TERM=xterm-256color script/bundle-mac
```

也可以单独验证 bundle 步骤：

```sh
cd crates/zed
cp Cargo.toml Cargo.toml.backup
sed -i.backup "s/package.metadata.bundle-dev/package.metadata.bundle/" Cargo.toml
TERM=xterm-256color CARGO_BUNDLE_SKIP_BUILD=true cargo bundle --release --target aarch64-apple-darwin --select-workspace-root
mv Cargo.toml.backup Cargo.toml
```

### 脚本失败后的恢复

`script/bundle-mac` 会临时把 `crates/zed/Cargo.toml` 中的 channel bundle metadata 改名。如果脚本中途失败，可能留下：

```text
crates/zed/Cargo.toml.backup
```

恢复方式：

```sh
mv crates/zed/Cargo.toml.backup crates/zed/Cargo.toml
git status --short --branch
```

确认工作树干净后再重试。

### 签名说明

如果没有以下 Apple 签名和 notarization 环境变量，脚本会使用本机 adhoc 签名：

```text
MACOS_CERTIFICATE
MACOS_CERTIFICATE_PASSWORD
APPLE_NOTARIZATION_KEY
APPLE_NOTARIZATION_KEY_ID
APPLE_NOTARIZATION_ISSUER_ID
```

个人包可以接受 adhoc 签名，但它不等同官方签名包。脚本会提示 entitlements 不完整，部分能力例如 universal links 可能不可用。

## Release 说明规范

Release tag 建议：

```text
personal-<feature>-YYYYMMDD
```

示例：

```text
personal-staging-ui-20260710
```

Release title 建议：

```text
Zed Personal <Feature> YYYY-MM-DD
```

示例：

```text
Zed Personal Staging UI 2026-07-10
```

Asset 命名建议：

```text
Zed-Personal-<Feature>-<short-sha>-aarch64.dmg
```

Release body 建议包含：

- 基于哪个分支或 commit
- 包含哪些个人功能
- 构建目标
- asset 文件名
- 最后一节固定为 `Release Notes:`

模板：

```md
Personal Zed build based on upstream main with local UI changes.

Included changes:
- ...

Build:
- Commit: <full-sha>
- Target: aarch64-apple-darwin
- Package: <asset-name>

Release Notes:

- Improved ...
```

创建 release 示例：

```sh
gh release create <tag> \
  target/aarch64-apple-darwin/release/<asset-name> \
  --repo gwokhov/zed \
  --target <full-sha> \
  --title "<title>" \
  --notes-file <notes-file> \
  --latest
```

删除旧 release 时，先确认新 release 和 asset 上传成功，再删除旧 release 和对应远端 tag：

```sh
gh release delete <old-tag> --repo gwokhov/zed --cleanup-tag --yes
git tag -d <old-tag>
```

最终检查：

```sh
gh release list --repo gwokhov/zed --limit 10
gh release view <tag> --repo gwokhov/zed --json tagName,name,url,targetCommitish,assets,isDraft,isPrerelease
git status --short --branch
git log --oneline --decorate --max-count=5
```
