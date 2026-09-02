---
name: 代码提交
description: 智能 Git 提交技能：先检查本地代码是否最新，若落后则自动同步远程最新代码，若遇到合并冲突则自动打开 VSCode 解决冲突，最后完成提交。使用场景：用户说"提交代码"、"commit"、"提交改动"、"推送代码"或运行 /代码提交 时触发。
---

# /代码提交 — 智能同步并提交代码

## 总体流程

按顺序执行以下步骤，每步完成后才进入下一步。

## Step 1 — 检查 git 仓库状态

首先确认当前目录是 git 仓库，并查看工作区状态：

```bash
git status
git diff --stat
```

如果不是 git 仓库，告知用户并停止。

如果工作区没有任何变更（clean），告知用户"没有需要提交的改动"并停止。

## Step 2 — 检测 Fork 并同步上游仓库

查看所有已配置的远程：

```bash
git remote -v
```

### 情况 A：存在 `upstream` 远程

直接获取上游最新代码并检查是否落后：

```bash
git fetch upstream
UPSTREAM_BRANCH=$(git remote show upstream | grep 'HEAD branch' | awk '{print $NF}')
git log HEAD..upstream/${UPSTREAM_BRANCH} --oneline 2>/dev/null
```

- 如果**无落后**，跳到 Step 3。
- 如果**落后于上游**，同步上游代码（优先 rebase 保持线性历史）：

```bash
git stash
git rebase upstream/${UPSTREAM_BRANCH}
```

**rebase 结果有两种情况：**

#### 情况 A：rebase 成功，无冲突

继续恢复暂存的本地改动：

```bash
git stash pop
```

- 如果 stash pop **无冲突**：清理 stash 并继续。
- 如果 stash pop **有冲突**：进入 **Step 2b — stash pop 冲突处理**。

#### 情况 B：rebase 有冲突

进入 **Step 4b — 冲突处理**（此时冲突来自 rebase，需执行 `git rebase --continue`）。

---

**上游同步成功后**，由于 rebase 重写了本地历史，与 origin 产生分叉，必须使用 `--force-with-lease` 推送（比 `--force` 更安全，会检查 origin 是否被他人更新）：

```bash
git push --force-with-lease origin $(git rev-parse --abbrev-ref HEAD)
```

继续进入 Step 3。

## Step 2b — stash pop 冲突处理

stash pop 冲突与 rebase 冲突不同，**无需** `git rebase --continue`。

### 自动解决所有冲突

#### 2b-1 读取并分析所有冲突文件

```bash
git diff --name-only --diff-filter=U
```

用 Read 工具读取每个冲突文件，逐块分析，**统一策略：以 Ours（upstream 最新代码）为基础，将 Theirs（本地修改）重新应用上去**：

1. 分析 Ours 和 Theirs 相对于公共祖先各自做了什么改动
2. 以 Ours 为基础版本（保证同步到最新代码）
3. 将 Theirs 的本地修改（增删改的意图）叠加到 Ours 上
4. 若两侧修改的区域不重叠，直接合并两侧内容
5. 若同一区域有差异，理解 Theirs 的修改意图并将其应用到 Ours 对应位置
6. 用 Edit 工具写回合并结果，去除所有冲突标记

#### 2b-2 生成 HTML 冲突报告（保留，不删除）

解决完成后，使用 Write 工具将报告保存到 **skill 目录** `/usr2/feiwa/.claude/skills/代码提交/conflict_review.html`（仓库外，不会被 `git add` 误提交），记录：
- 页面标题：`Conflict Review — <仓库名> <当前时间>`
- 每个冲突文件独立一个 `<section>`
- 每个冲突块展示：
  - **Ours**（`<<<<<<< Updated upstream` 侧）：红色背景 `#ffe0e0`
  - **Theirs**（`>>>>>>> Stashed changes` 侧）：绿色背景 `#e0ffe0`
  - **Resolution**：蓝色背景 `#e0eeff`，显示合并结果（以 Ours 为基础，本地修改已叠加）+ 一句说明做了什么合并
- 代码使用 `<pre>` 保留缩进，HTML 特殊字符转义

生成后用浏览器打开（不阻塞）：

```bash
xdg-open /usr2/feiwa/.claude/skills/代码提交/conflict_review.html 2>/dev/null || open /usr2/feiwa/.claude/skills/代码提交/conflict_review.html 2>/dev/null || true
```

**HTML 文件保存在 skill 目录下，不主动删除，供用户事后查阅。**

#### 2b-3 标记已解决

```bash
git add <已解决的文件>
git stash drop
```

告知用户：冲突已自动解决，解决方案已保存至 `conflict_review.html`，继续流程。

### 情况 B：不存在 `upstream`，尝试自动检测是否为 Fork

使用 `gh` CLI 检查当前仓库是否为 fork（如已安装）：

```bash
gh repo view --json isFork,parent 2>/dev/null
```

- 如果 `isFork` 为 `true`：提取 `parent.url`，**直接自动添加 upstream**（无需询问）：
  ```bash
  git remote add upstream <parent 仓库 clone URL>
  ```
  添加成功后重新执行 Step 2。

- 如果 `gh` 未安装，或 `isFork` 为 `false`，或命令失败：跳过本步骤，继续 Step 3。

## Step 3 — 获取 origin 最新信息

```bash
git fetch origin
git status -sb
```

根据输出判断状态：

- **`[ahead N]`（仅领先）**：无需同步，跳到 Step 5。
- **`[behind N]`（仅落后）**：进入 Step 4 同步 origin。
- **`[ahead N, behind M]`（分叉）**：通常发生在 rebase upstream 之后，**直接自动执行 force-with-lease 推送**（告知用户正在执行）：
  ```bash
  git push --force-with-lease origin $(git rev-parse --abbrev-ref HEAD)
  ```
  推送成功后跳到 Step 5。
- **无标记（同步）**：跳到 Step 5。

## Step 4 — 同步 origin 最新代码

优先使用 rebase 方式同步，保持线性历史：

```bash
git stash
git pull --rebase origin $(git rev-parse --abbrev-ref HEAD)
git stash pop
```

或直接（如果工作区已暂存）：

```bash
git pull --rebase origin $(git rev-parse --abbrev-ref HEAD)
```

同步结果有四种情况：

### 情况 A：同步成功，无冲突
继续进入 Step 5。

### 情况 B：出现 rebase/merge 冲突（来自 `git pull --rebase`）
进入 **Step 4b — 冲突处理**。

### 情况 C：同步失败（网络、权限等）
告知用户具体错误，**自动跳过同步，直接继续提交本地改动**，进入 Step 5。

### 情况 D：stash pop 有冲突（`git pull --rebase` 成功但 `git stash pop` 冲突）
进入 **Step 2b — stash pop 冲突处理**（先尝试自动解决，再开 VSCode）。

## Step 4b — 冲突处理：自动解决并生成 HTML 报告

检测到冲突文件后，执行：

```bash
git diff --name-only --diff-filter=U
```

#### 4b-1 自动解决所有冲突

用 Read 工具读取每个冲突文件，**统一策略：以 Ours（upstream 最新代码）为基础，将 Theirs（本地修改）重新应用上去**：

1. 分析 Ours 和 Theirs 相对于公共祖先各自做了什么改动
2. 以 Ours 为基础版本（保证同步到最新代码）
3. 将 Theirs 的本地修改意图叠加到 Ours 上
4. 若两侧修改区域不重叠，直接合并两侧内容
5. 若同一区域有差异，理解 Theirs 的修改意图并将其应用到 Ours 对应位置
6. 用 Edit 工具写回合并结果，去除所有冲突标记

#### 4b-2 生成 HTML 冲突报告（保留，不删除）

使用 Write 工具将报告保存到 **skill 目录** `/usr2/feiwa/.claude/skills/代码提交/conflict_review.html`（仓库外，不会被 `git add` 误提交），记录：
- 页面标题：`Conflict Review — <仓库名> <当前时间>`
- 每个冲突块展示：
  - **Ours**：红色背景 `#ffe0e0`
  - **Theirs**：绿色背景 `#e0ffe0`
  - **Resolution**：蓝色背景 `#e0eeff`，显示最终保留的内容 + 决策理由
- 代码使用 `<pre>`，HTML 特殊字符转义

生成后打开浏览器（不阻塞）：

```bash
xdg-open /usr2/feiwa/.claude/skills/代码提交/conflict_review.html 2>/dev/null || open /usr2/feiwa/.claude/skills/代码提交/conflict_review.html 2>/dev/null || true
```

**HTML 文件保存在 skill 目录下，不主动删除，供用户事后查阅。**

#### 4b-3 标记已解决，继续操作

```bash
git add <已解决的文件>
```

如果是 rebase 冲突：

```bash
git rebase --continue
```

如果是 merge 冲突：

```bash
git merge --continue
```

告知用户：冲突已自动解决，解决方案已保存至 `conflict_review.html`，继续进入 Step 5。

## Step 5 — 准备提交信息

查看将要提交的变更内容：

```bash
git diff --cached --stat
git diff --stat
git log --oneline -5
```

首先判断是否为 **amend 模式**：若 `$ARGUMENTS` 包含 `追加` 或 `amend`（不区分大小写），则进入 amend 模式。

### 普通模式

根据 diff 内容，结合用户在 `$ARGUMENTS` 中提供的描述（如有），起草一条清晰的提交信息：
- 如果 `$ARGUMENTS` 非空（且不是 `追加`/`amend` 关键词本身），将其作为提交信息主体
- 否则根据变更内容自动生成简洁的**英文**提交信息
- 格式：`<type>: <short description>` （如 `feat: add user login`、`fix: resolve null pointer exception`）

告知用户拟定的提交信息，**直接使用该信息进入 Step 6**，无需用户确认。

### Amend 模式

获取上一笔提交信息作为默认值：

```bash
git log -1 --pretty=%B
```

- 若 `$ARGUMENTS` 中除关键词外还有其他内容，使用该内容作为新的提交信息
- 否则**沿用上一笔提交信息不变**

告知用户将 amend 到上一笔提交，提交信息为 `<复用或更新的信息>`，**直接进入 Step 6**。

## Step 6 — 执行提交

将所有变更加入暂存区：

```bash
git add -A
```

**普通模式**：

```bash
git commit -s -m "<提交信息>"
```

**Amend 模式**：

```bash
git commit --amend -s -m "<提交信息>"
```

## Step 7 — 推送

提交完成后，**自动推送到远程仓库**（告知用户正在推送）。

**普通模式**：

```bash
git push origin $(git rev-parse --abbrev-ref HEAD)
```

**Amend 模式**：amend 重写了历史，需使用 `--force-with-lease`：

```bash
git push --force-with-lease origin $(git rev-parse --abbrev-ref HEAD)
```

推送完成后，告知用户远程 URL 和分支名。

## 注意事项

- 不要跳过 pre-commit hook（不使用 `--no-verify`）
- 不要强制推送（不使用 `--force`），除非用户明确要求
- 如果分支是 `main` 或 `master` 且用户要求 force push，需额外警告并确认
- 所有冲突必须完全解决后才能继续提交
