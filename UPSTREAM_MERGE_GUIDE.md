# 上游发布版本合并指南

本文档用于把上游仓库的某个**已发布版本**合并到本仓库的 `dev` 分支。

## 核心原则

- 只合并上游发布版本对应的固定数字 tag。
- 不直接合并 `upstream/master`，否则可能带入尚未发布的提交。
- 不使用 `rolling`，因为它可能随上游滚动更新。
- `git fetch` 只下载对象，不会修改 `dev`。
- 普通 `git merge` 在没有冲突时会自动提交。使用 `--no-commit --no-ff` 可以先检查结果，再决定是否提交。

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

先设置本次要合并的上游版本号，使用固定数字 tag，不要使用 `rolling`：

```bash
VERSION="<上游版本号>"
```

### 1. 确认上游发布 tag

列出上游 tag：

```bash
git ls-remote --tags --refs upstream
```

在上游发布页面确认要合并的固定数字 tag。不要选择 `rolling`，也不要仅凭 `master` 当前的位置判断发布版本。

确认指定 tag 存在：

```bash
git ls-remote --exit-code --tags upstream "refs/tags/$VERSION"
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
git fetch --no-tags upstream tag "$VERSION"
git show --no-patch --decorate "$VERSION"
```

这不会获取或合并 `upstream/master`，也不会修改当前工作区。

### 4. 合并，但停在 commit 前

```bash
git merge --no-commit --no-ff "$VERSION"
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
git diff --cached --check
```

第一条命令应无输出，后两条命令不应报告冲突标记或空白错误。

## 合并时检查 CI 配置

上游可能修改或新增 `.github/workflows` 下的文件。提交前必须检查：

```bash
git diff --cached -- .github/workflows
```

发现上游修改或新增 workflow 文件时，应执行以下流程：

1. 保留并暂存上游对 `.github/workflows` 的修改和新增文件。
2. 明确通知用户本次合并改变了哪些 workflow 文件，以及这些变化可能带来的影响。
3. 检查本仓库自有的 `.github/workflows/build-sparkle.yml`，结合上游 workflow 的变化，协助用户更新本地构建 workflow，使其继续符合本仓库的构建需求。
4. 将确认后的 `.github/workflows/build-sparkle.yml` 修改加入暂存区：

```bash
git add .github/workflows/build-sparkle.yml
```

5. 在用户确认 workflow 调整完成前，不创建 merge commit。

不要使用 `git restore --source=HEAD --staged --worktree .github/workflows` 覆盖上游 workflow 变化。

## 完成 merge commit

只有在用户确认上游 workflow 变化及 `.github/workflows/build-sparkle.yml` 调整完成后，才可以继续提交。

检查无误后提交：

```bash
git commit -m "merge: upstream release $VERSION"
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
[ ] 如 workflow 有变化，通知用户并说明影响
[ ] 检查并协助修改 .github/workflows/build-sparkle.yml
[ ] 等待用户确认 workflow 调整完成
[ ] 创建 merge commit
[ ] git push origin dev
```
