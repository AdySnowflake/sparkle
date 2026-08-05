# 上游发布版本合并与 CI 构建指南

本文档用于把上游仓库的某个**已发布版本**合并到本仓库的 `dev` 分支，并通过本仓库的 GitHub Actions 构建。

## 核心原则

- 只合并上游发布版本对应的固定数字 tag，例如 `1.26.7`。
- 不直接合并 `upstream/master`，否则可能带入尚未发布的提交。
- 不使用 `rolling`，因为它可能随上游滚动更新。
- `git fetch` 只下载对象，不会修改 `dev`，也不会触发 CI。
- 普通 `git merge` 在没有冲突时会自动提交。使用 `--no-commit --no-ff` 可以先检查结果，再决定是否提交。
- CI 只会看到已经推送到 GitHub 的提交；本地 fetch 或未提交的 merge 不会触发 CI。

## 当前远程配置

本仓库预期配置如下：

```text
origin    git@github.com:AdySnowflake/sparkle.git (fetch)
origin    git@github.com:AdySnowflake/sparkle.git (push)
upstream  git@github.com:xishang0128/sparkle.git  (fetch)
upstream  DISABLED                                (push)
```

检查配置：

```bash
git remote -v
git config --get remote.pushDefault
```

`remote.pushDefault` 应输出 `origin`。`upstream` 的 push 地址设置为 `DISABLED`，用于防止误推送。

如果在新的工作副本中需要重新配置：

```bash
git remote add upstream git@github.com:xishang0128/sparkle.git
git remote set-url --push upstream DISABLED
git config remote.pushDefault origin
```

## 每次发布后的标准操作

下面以 `1.26.8` 为例。操作时将它替换为上游实际发布的版本号。

### 1. 确认上游发布 tag

列出上游 tag：

```bash
git ls-remote --tags --refs upstream
```

在上游发布页面确认要合并的固定数字 tag。不要选择 `rolling`，也不要仅凭 `master` 当前的位置判断发布版本。

确认指定 tag 存在：

```bash
git ls-remote --exit-code --tags upstream refs/tags/1.26.8
```

### 2. 确保本地 `dev` 可安全操作

```bash
git switch dev
git status
```

开始前应确保：

- 没有未提交的文件修改。
- 没有尚未完成的 merge 或 rebase。
- 本地 `dev` 已包含自己希望保留的修改。

如需同步自己远程仓库中的 `dev`，使用只允许快进的方式：

```bash
git pull --ff-only origin dev
```

### 3. 只获取目标发布 tag

```bash
git fetch --no-tags upstream tag 1.26.8
git show --no-patch --decorate 1.26.8
```

这不会获取或合并 `upstream/master`，也不会修改当前工作区。

### 4. 合并，但停在 commit 前

```bash
git merge --no-commit --no-ff 1.26.8
```

如果没有冲突，Git 会显示类似：

```text
Automatic merge went well; stopped before committing as requested
```

此时 merge 尚未提交，可以先检查：

```bash
git status
git diff --cached --stat
git diff --cached
```

## 冲突处理

### 暂时不处理冲突

如果发生冲突，而当前不准备处理：

```bash
git merge --abort
```

这会回到合并前的 `dev`。以后可以重新执行获取 tag 和 merge 的步骤。

### 立即解决冲突

查看冲突文件：

```bash
git diff --name-only --diff-filter=U
```

逐个修改冲突文件后，将已解决文件加入暂存区：

```bash
git add <已解决的文件>
```

确认已经没有未解决文件：

```bash
git diff --name-only --diff-filter=U
git diff --check
```

第一条命令应无输出，第二条命令不应报告冲突标记或空白错误。

## 合并时检查 CI 配置

上游可能修改或新增 `.github/workflows` 下的文件。提交前必须检查：

```bash
git diff --cached -- .github/workflows
```

有两种处理方式：

1. 保留上游 workflow，推送后在 GitHub Actions 页面手动禁用不需要的 workflow。
2. 完全保留合并前的本地 workflow，不接受本次上游 workflow 变化：

```bash
git restore --source=HEAD --staged --worktree .github/workflows
```

第二条命令会丢弃本次 merge 带来的所有 workflow 修改和新增文件，执行前应先检查 diff，确认这是预期行为。

## 完成 merge commit

检查无误后提交：

```bash
git commit -m "merge: upstream release 1.26.8"
```

确认生成的是 merge commit，并查看它的两个父提交：

```bash
git show --no-patch --pretty=raw HEAD
```

然后推送 `dev`：

```bash
git push origin dev
```

推送分支不会自动推送刚刚获取的上游 tag。不要使用 `git push --tags`，避免把上游 tag 批量推到自己的仓库。

## 触发云端构建

当前自定义 workflow 文件为 `.github/workflows/build-sparkle.yml`，支持以下方式：

- 在 GitHub Actions 页面手动运行 `Build Sparkle`，并选择 `dev`。
- 推送名称以 `v` 开头、且指向本次 `dev` merge commit 的构建 tag。

如果使用构建 tag，先确认当前 `HEAD` 就是刚刚推送的 `dev`：

```bash
git status
git log -1 --oneline --decorate
```

然后创建并推送自己的构建 tag，例如：

```bash
git tag v1.26.8-build.1
git push origin v1.26.8-build.1
```

上游的 `1.26.8` 和自己的 `v1.26.8-build.1` 是两个不同的 tag：前者标记上游发布点，后者标记包含本地修改的实际构建点。

## 失败后的恢复

merge 尚未提交时撤销：

```bash
git merge --abort
```

merge 已提交但尚未推送时，优先使用可审计的 revert，而不要改写已经共享的历史：

```bash
git revert -m 1 <merge-commit-sha>
```

如果 merge commit 已经推送，也使用 `git revert -m 1`，然后正常推送新的 revert commit。不要对共享的 `dev` 执行强制推送。

## 每次操作的简要清单

```text
[ ] 在上游发布页面确认固定数字 tag
[ ] 确认处于 dev 且工作区干净
[ ] git fetch --no-tags upstream tag <版本>
[ ] git merge --no-commit --no-ff <版本>
[ ] 解决冲突，或 git merge --abort
[ ] 检查源码和 .github/workflows 的 staged diff
[ ] 创建 merge commit
[ ] git push origin dev
[ ] 手动运行 Build Sparkle，或推送自己的 v* 构建 tag
[ ] 检查 GitHub Actions 构建结果
```
