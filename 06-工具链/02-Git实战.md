---
tags:
  - Git
  - 工具
  - 版本控制
created: 2026-08-17
---

# Git 实战

> Git 是**版本控制工具**，核心职责：记录代码历史、团队协作、随时回退。和你 GitHub 仓库配合使用。

---

## 1️⃣ 核心概念

```
三种状态：
  modified（已修改）  →  working directory（工作区）
  staged（已暂存）    →  staging area（暂存区）
  committed（已提交）  →  local repository（本地仓库）

文件流转：
  工作区 → git add → 暂存区 → git commit → 本地仓库
  ←────── git checkout / git restore ←───────

四个区域：
  Working Directory（工作区）：你正在编辑的文件夹
  Staging Area（暂存区）：git add 后的文件快照
  Local Repository（本地仓库）：.git 目录，历史记录
  Remote Repository（远程仓库）：GitHub / Gitee / GitLab

HEAD：指向当前分支最新提交的指针（书签）
  HEAD~1 = 当前提交的父提交
  HEAD~2 = 祖父提交
```

---

## 2️⃣ 日常操作（你项目最常用的）

```powershell
# ===== 查看 =====
git status                  # 查看哪些文件变了
git status -s              # 简洁模式
git log --oneline          # 提交历史（精简）
git log --oneline -10      # 最近10条

# ===== 暂存 =====
git add filename.txt       # 暂存单个文件
git add src/               # 暂存整个目录
git add .                  # 暂存所有改动（慎用！）

# ===== 提交 =====
git commit -m "feat: 新增用户登录功能"    # 提交
git commit -am "fix: 修复空指针"           # 暂存所有已跟踪文件并提交（新文件不包含）
git commit --amend         # 修改上一次 commit 信息（还没 push 时用）

# ===== 撤销 =====
git checkout -- filename.txt    # 丢弃工作区改动（危险！）
git restore filename.txt       # 同上，Git 2.23+ 推荐
git reset HEAD filename.txt    # 取消暂存（移出暂存区）

# ===== 回退版本 =====
git reset --soft HEAD~1        # 回退1个 commit，保留改动在暂存区
git reset --mixed HEAD~1       # 回退1个 commit，保留改动在工作区（默认）
git reset --hard HEAD~1        # 回退1个 commit，丢弃所有改动（危险！）
git reset --hard abc1234       # 回退到某个 commit ID
```

---

## 3️⃣ 分支操作

```powershell
# ===== 查看 =====
git branch                   # 本地分支（* 是当前分支）
git branch -a                # 所有分支（包括远程）

# ===== 创建/切换 =====
git branch new-feature       # 创建分支
git checkout new-feature     # 切换到分支
git switch new-feature       # 同上，Git 2.23+ 推荐
git checkout -b new-feature  # 创建 + 切换一步到位
git switch -c new-feature    # 同上

# ===== 删除 =====
git branch -d new-feature    # 删除分支（已合并才允许）
git branch -D new-feature    # 强制删除

# ===== 合并 =====
git checkout main            # 切回主分支
git merge new-feature        # 合并新分支进来

# 合并冲突时：
# 1. 打开冲突文件
# 2. 手动解决（删掉 <<< === >>> 标记）
# 3. git add filename.txt
# 4. git commit -m "merge: 解决冲突"
```

---

## 4️⃣ 远程操作

```powershell
# ===== 拉取/推送 =====
git fetch origin             # 拉取远程最新状态（不合并）
git pull origin main        # 拉取 + 合并
git push origin main         # 推送到远程
git push -u origin main     # 首次推送，设置上游分支
git push --force            # 强制推送（⚠️ 危险！只在自己分支用）

# ===== 远程分支 =====
git checkout -b local-name origin/remote-name   # 拉取远程分支并切换
git push origin --delete old-branch  # 删除远程分支
```

---

## 5️⃣ 其他常用操作

```powershell
# ===== Rebase（整理提交历史）=====
git rebase main
# 把当前分支的提交"复制"到 main 之上，历史变成直线
# ⚠️ 不要 rebase 已经 push 到远程的提交！

# ===== Stash（临时保存工作区）=====
git stash                    # 暂存当前工作区（不提交）
git stash list               # 查看暂存列表
git stash pop                # 恢复暂存 + 删除暂存
git stash apply              # 恢复暂存（保留记录）

# ===== 标签 =====
git tag v1.0.0               # 创建轻量标签
git tag -a v1.0.0 -m "第一个正式版本"  # 创建附注标签
git push origin v1.0.0      # 推送标签到远程

# ===== 查看具体改动 =====
git diff                     # 工作区 vs 暂存区
git diff --staged           # 暂存区 vs 最新 commit
git diff HEAD               # 工作区 vs 最新 commit
git show <commit-id>        # 某次提交的具体内容
git blame filename.txt       # 逐行看是谁改的
```

---

## 6️⃣ GitHub SSH Key 配置

```powershell
# 1. 生成 SSH Key
ssh-keygen -t ed25519 -C "your_email@example.com"

# 2. 查看公钥
cat ~/.ssh/id_ed25519.pub

# 3. 复制公钥，粘贴到 GitHub → Settings → SSH Keys

# 4. 测试连接
ssh -T git@github.com
```

---

## 7️⃣ .gitignore（你项目已有的）

```gitignore
# 编译产物
target/
*.class
*.jar
*.war

# IDE
.idea/
*.iml
.vscode/

# 本地配置（包含 API Key！安全重点）
application-local.yml
*.local.yml
.env

# 日志和系统文件
*.log
.DS_Store

# 运行时生成的文件
data/
*.txt
```

```gitignore
# 高级技巧
# 忽略但保留目录 → 创建 .gitkeep 文件
# 不忽略某文件
!error.log
# 忽略已跟踪文件的改动（不提交但保留本地）
git update-index --assume-unchanged filename.txt
```

---

## 8️⃣ 提交信息规范（Conventional Commits）

```
格式：<type>(<scope>): <description>

type：
  feat     新功能
  fix      修复 bug
  docs     文档改动
  style    格式（不影响代码）
  refactor 重构
  perf     性能优化
  test     测试相关
  chore    构建/工具/依赖

示例：
  feat(love): 新增恋爱专家对话接口
  fix(terminal): 修复命令白名单校验逻辑
  docs: 更新 README 使用说明
  refactor(agent): 重构 ReAct Agent 思考循环
  chore(deps): 升级 Spring AI 到 1.0.0-M6

⚠️ 不要：
  ❌ "update" / "fix bug" / "xxx"
  ✅ "feat: 新增登录功能" / "fix: 修复空指针异常"
```

---

## 9️⃣ Git Flow 工作流

```
main（生产） ───────────────────────────────────────────▶
                ↑                           ↑
                │ merge                     │ merge
                ↓                           ↓
            develop ←───────────────────────
               ↑
     ↑_________|___________↑_______________
     ↑         ↑           ↑              ↑
  feature/  refactor/    bugfix/       hotfix/
  login     optimize    sidebar       urgent-fix
  从 develop 创建，完成后合并回 develop
```

---

## 🔟 速查清单

```powershell
# 你项目的日常流程
git status                               # 看改动
git add . && git commit -m "feat: xxx"  # 暂存+提交
git push origin main                     # 推送

# 撤销
git restore filename.txt               # 丢弃工作区改动
git reset HEAD filename.txt            # 取消暂存

# 回退（危险！）
git reset --hard HEAD~1                # 回退1个 commit

# 解决冲突
git fetch origin
git merge origin/main
# 手动解决 → git add → git commit → git push

# 查看
git remote -v                          # 远程地址
git log --oneline -10                  # 最近10条提交
```

---

## ❓ 面试题

**Q: Git 的 merge 和 rebase 区别？**
> `git merge` 保留真实分叉历史，适合协作分支。`git rebase` 把当前分支的提交"复制"到目标分支之后，历史变成一条直线，更干净，但会改变提交历史。**规则：已经 push 到远程的分支不要 rebase。**

**Q: Git 怎么撤销已经 push 的提交？**
> `git revert HEAD`（安全撤销，创建新提交反向之前的改动，不改历史）。或者 `git reset --hard HEAD~n` + `git push --force`（危险，改历史）。

**Q: Git 如何解决合并冲突？**
> 1. `git pull` 报 CONFLICT；2. 打开冲突文件，会看到下面这种冲突标记；3. 手动保留想要的代码，删掉三行标记；4. `git add` 暂存；5. `git commit` 完成合并。

```text
<<<<<<< HEAD
你当前分支的代码
=======
要合并进来的代码
>>>>>>> branch-name
```

**Q: .gitignore 不生效怎么办？**
> 文件已经被 git track 后再写入 .gitignore 无效。需 `git rm --cached filename` 清除跟踪，然后重新 add。


---

## 🔗 相关笔记

- [[01-Maven实战|Maven 实战]]
- [[03-IDEA实战|IDEA 实战]]
