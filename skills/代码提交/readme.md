# 代码提交 Skill — 使用说明

## 一、功能概述

"代码提交"是一个智能 Git 提交技能。它不仅仅是执行一次 `git commit`，而是完整地处理从"检查同步状态"到"推送远程"的全流程，并能自动解决合并冲突。

**触发方式：**
- 对话中说 `提交代码`、`commit`、`提交改动`、`推送代码`
- 或在 Claude Code 中运行 `/代码提交`

---

## 二、完整执行流程

Skill 会按以下步骤自动依次执行，每步完成后才进入下一步：

| 步骤 | 说明 |
|------|------|
| **Step 1** 检查仓库状态 | 确认当前目录是 git 仓库，检查是否有变更。若无变更则停止并告知用户。 |
| **Step 2** 同步上游（upstream） | 若配置了 `upstream` 远程，自动检查是否落后并 rebase 到最新。若未配置，通过 `gh` CLI 自动检测是否为 Fork，若是则自动添加 upstream。 |
| **Step 3** 检查 origin 状态 | 检查本地分支与 origin 的关系（领先/落后/分叉），决定下一步操作。 |
| **Step 4** 同步 origin | 若落后于 origin，执行 `git pull --rebase` 同步最新提交，保持线性历史。 |
| **Step 5** 准备提交信息 | 分析 diff 内容，自动生成英文提交信息（`type: description` 格式）。若用户附带了描述则优先使用。 |
| **Step 6** 执行提交 | `git add -A` 暂存所有变更，`git commit -s` 完成提交。 |
| **Step 7** 推送远程 | 自动推送到当前分支对应的 `origin`，并告知用户远程 URL 和分支名。 |

---

## 三、冲突自动解决

同步过程中若遇到合并冲突，Skill 会**全自动**尝试解决，无需手动操作：

1. 逐一读取所有冲突文件
2. 以**上游最新代码（Ours）**为基础，将本地修改（Theirs）叠加上去
3. 用 Edit 工具写回合并结果，清除所有 `<<<<<<<` / `=======` / `>>>>>>>` 标记
4. 生成 HTML 冲突报告，保存至：
   ```
   /usr2/feiwa/.claude/skills/代码提交/conflict_review.html
   ```
   并自动用浏览器打开，供用户审阅
5. 标记冲突已解决，继续提交流程

### HTML 报告颜色含义

| 颜色 | 含义 |
|------|------|
| 红色背景 `#ffe0e0` | **Ours** — 上游最新代码 |
| 绿色背景 `#e0ffe0` | **Theirs** — 本地修改 |
| 蓝色背景 `#e0eeff` | **Resolution** — 最终合并结果 + 决策说明 |

> HTML 文件保存在 skill 目录下，不会被 `git add` 误提交到仓库，可事后随时查阅。

---

## 四、使用示例

```bash
# 普通提交（自动生成提交信息）
/代码提交

# 附带提交信息
/代码提交 fix: resolve crash on startup

# Amend 上一笔提交（追加修改，不新建 commit）
/代码提交 追加
/代码提交 amend fix: resolve crash on startup
```

---

## 五、注意事项

- 默认使用 **rebase** 方式同步，保持线性 git 历史，避免多余的 merge commit
- 不跳过 pre-commit hook，不使用 `--no-verify`
- 普通推送不使用 `--force`；rebase 后必要时使用更安全的 `--force-with-lease`
- 推送 `main` / `master` 分支时若需 force push，会额外警告并要求用户确认
- 若网络或权限导致同步失败，Skill 自动跳过同步，直接提交本地改动

---

## 六、文件位置

| 文件 | 路径 |
|------|------|
| Skill 定义 | `/local/mnt/workspace/feiwa/codes/AI_code/skills/代码提交/SKILL.md` |
| 冲突报告 | `/usr2/feiwa/.claude/skills/代码提交/conflict_review.html` |
