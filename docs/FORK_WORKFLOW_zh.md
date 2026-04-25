# Fork 仓库维护与提交指南

> 本文档面向 **个人 fork 仓库** 的日常使用：commit、push、与上游同步、提 PR。
>
> - 你的 fork：`https://github.com/JustinLamster/seekdb_explore.git`
> - 上游仓库：`https://github.com/oceanbase/seekdb.git`
> - 主力开发分支：`develop`
>
> 命令在 Windows（Git Bash / PowerShell）和 Linux / WSL 下通用。Windows 用户额外注意"行尾"那一节。

---

## 目录

- [Part 1 — 一次性配置](#part-1--一次性配置)
- [Part 2 — 日常工作流](#part-2--日常工作流)
- [Part 3 — 同步 fork 与上游](#part-3--同步-fork-与上游)
- [Part 4 — 提 PR 给上游](#part-4--提-pr-给上游)
- [Part 5 — 常见坑与避雷](#part-5--常见坑与避雷)
- [Part 6 — 命令速查表](#part-6--命令速查表)
- [Part 7 — 应急操作](#part-7--应急操作)

---

## Part 1 — 一次性配置

### 1.1 配置身份信息

```bash
git config --global user.name  "JustinLamster"
git config --global user.email "你的邮箱@example.com"
```

注意：这里的 email 必须是你 GitHub 账号绑定的邮箱（或 GitHub 提供的 `noreply` 邮箱），否则 GitHub 不会把 commit 关联到你的头像。

> 想找你的 noreply 邮箱：GitHub → Settings → Emails → "Keep my email addresses private"，下方会显示 `xxxxx+JustinLamster@users.noreply.github.com`。

### 1.2 重新组织 remote（关键）

clone 下来时 `origin` 默认指向 **上游 OceanBase**（你没有写权限）。日常操作中你会大量 `git push`、`git pull`，所以应该让 `origin` 指向你能 push 的 fork。

```bash
# 查看当前
git remote -v
# origin  https://github.com/oceanbase/seekdb.git (fetch)
# origin  https://github.com/oceanbase/seekdb.git (push)

# 把上游改名 origin → upstream（只读用）
git remote rename origin upstream

# 把你的 fork 加成新的 origin（默认 push 目标）
git remote add origin https://github.com/JustinLamster/seekdb_explore.git

# 验证
git remote -v
# origin    https://github.com/JustinLamster/seekdb_explore.git (fetch)
# origin    https://github.com/JustinLamster/seekdb_explore.git (push)
# upstream  https://github.com/oceanbase/seekdb.git              (fetch)
# upstream  https://github.com/oceanbase/seekdb.git              (push)

# 防止手滑 push 到上游：把 upstream 的 push URL 设成无效地址
git remote set-url --push upstream DISABLED
```

### 1.3 GitHub 认证

**HTTPS + Personal Access Token（推荐入门）：**

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token
2. 勾选 `repo` 权限，生成后复制（只显示一次）
3. 第一次 `git push` 时，用户名填 GitHub 用户名，密码**粘贴 token**（不是 GitHub 密码）
4. Windows 会用 Credential Manager 自动记住

**SSH（推荐长期使用）：**

```bash
# 生成密钥
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
# 一路回车，密钥默认存在 ~/.ssh/id_ed25519

# 显示公钥，复制
cat ~/.ssh/id_ed25519.pub

# 粘贴到 GitHub → Settings → SSH and GPG keys → New SSH key

# 把 origin 切到 SSH 协议
git remote set-url origin git@github.com:JustinLamster/seekdb_explore.git

# 测试
ssh -T git@github.com
```

### 1.4 Windows 行尾配置（重要）

seekdb 是 Linux 项目，所有源文件用 LF。Windows 上 git 默认会在 checkout 时把 LF 转 CRLF，commit 时再转回 LF——这通常没问题，但有时候会"污染"二进制文件或脚本。

```bash
# Windows 推荐设置：checkout 转 CRLF，commit 转 LF
git config --global core.autocrlf true

# 如果你只在 WSL/Linux 里改 seekdb，建议关掉自动转换
git config --global core.autocrlf input
```

如果发现某个文件 `git diff` 显示整个文件都改了，但你其实没动，多半是行尾问题。检查方法：

```bash
git diff --stat                       # 看哪些文件改了
git diff -- <file> | head             # 如果全是 ^M 字符，就是行尾
```

### 1.5 检查 .gitignore 是否合理

clone 下来的仓库已经有完善的 `.gitignore`，常规使用不需要动。但如果你新增了构建产物路径、IDE 配置目录，最好补一下，避免误提交。**永远不要**把以下东西 commit 上去：

- 构建产物：`build_debug/`、`build_release/`
- 第三方依赖：`deps/3rd/usr/`
- IDE 个人配置：`.vscode/settings.json`（除非项目级）、`.idea/`
- secrets：`.env`、`*.key`、`*.pem`、含密码的 yaml
- 大文件：模型权重、数据集、vector 索引文件（>100MB GitHub 会拒）
- OS 垃圾：`.DS_Store`、`Thumbs.db`

---

## Part 2 — 日常工作流

### 2.1 心智模型

```
upstream/develop  ─────●─────●─────●─────●  (上游持续更新)
                              │
                              ▼ git fetch upstream
本地 develop      ─────●─────●  (跟着 upstream，不直接改)
                              │
                              ▼ git checkout -b feat/xxx
本地 feat/xxx     ─────●─────●─────●  (你的工作分支)
                                    │
                                    ▼ git push origin feat/xxx
fork  feat/xxx    ─────────────────●  (备份 + 提 PR 用)
```

**两条铁律**：
1. **不要在 `develop` 上直接 commit 自己的实验性改动**——这条分支专门用来跟上游同步。
2. **不要 `git push upstream`**——你没有写权限，且这是错误目标。

### 2.2 一次完整流程示例

假设你想加一份"混合检索调研笔记"：

```bash
# Step 1: 同步上游 develop 到本地
git checkout develop
git fetch upstream
git merge --ff-only upstream/develop      # 只允许 fast-forward，避免意外合并

# Step 2: 从最新 develop 切一个工作分支
git checkout -b docs/hybrid-search-notes

# Step 3: 写代码 / 改文件 ...
# (省略具体编辑)

# Step 4: 看自己改了啥
git status                                # 文件级别
git diff                                  # 已修改但未 stage 的内容
git diff --staged                         # 已 stage 的内容

# Step 5: 选择性 stage（不要用 git add -A）
git add docs/hybrid_search_notes.md
git add src/some_file.cpp
# 想交互式选择某个文件的部分 hunk：
git add -p src/some_file.cpp

# Step 6: 提交
git commit -m "docs: add hybrid search investigation notes"

# Step 7: 推到 fork
git push -u origin docs/hybrid-search-notes
# -u 第一次推送时设置上游跟踪，之后直接 git push / git pull 即可
```

### 2.3 Commit 信息规范

参考现有项目风格（看 `git log --oneline`），seekdb 用的是简短描述 + 可选的标签前缀：

```
[CP] advance scan bug fix
fix(android): avoid including libaio.h on Android platform
[vector index] fix debug_sync position lead to mysqltest case hung
```

你自己的 fork 没硬性要求，但建议：

- **第一行**：50 字符以内的概括
- **空一行**
- **正文**：解释 *为什么* 这么改（**不是** *改了什么*——diff 已经说明），列出影响、副作用、参考的 issue
- **结尾**：可选 `Co-Authored-By:` 等 trailer

例：

```
docs: add hybrid search investigation notes

调研 seekdb 在 vector + FTS 混合检索路径下的实现。
笔记主要梳理了：
- 查询计划走哪个 operator
- 距离度量与全文打分如何融合
- 已知性能瓶颈

为后续给 retrieval 模块提优化做准备。
```

提交多行 message：

```bash
git commit                                # 不带 -m 会打开编辑器
# 或者用 HEREDOC（bash）
git commit -m "$(cat <<'EOF'
docs: add hybrid search investigation notes

...正文...
EOF
)"
```

### 2.4 改了又想撤销 / 重做

```bash
# 撤销工作区改动（未 add）
git restore <file>                        # 把单个文件恢复到 HEAD
git restore .                             # 全部恢复（小心！）

# 撤销已 add（回到工作区，仍保留改动）
git restore --staged <file>

# 改最后一次 commit message（还没 push）
git commit --amend                        # 打开编辑器
git commit --amend -m "新 message"

# 把刚才漏 add 的文件加进上一次 commit（还没 push）
git add <file>
git commit --amend --no-edit

# 已经 push 出去后，绝对不要再 amend——会造成历史分叉
# 应该追加一个新 commit：
git commit -m "fix: address typo from previous commit"
```

---

## Part 3 — 同步 fork 与上游

上游 `develop` 不断更新，你的 fork 不会自动跟着变。需要定期同步。

### 3.1 命令行同步（推荐）

```bash
git checkout develop
git fetch upstream
git merge --ff-only upstream/develop      # 把本地 develop fast-forward 到上游
git push origin develop                   # 把同步结果推到 fork
```

如果 `--ff-only` 失败（说明你在本地 develop 上误 commit 了东西），先把那些 commit 摘到工作分支：

```bash
git checkout develop
git log upstream/develop..HEAD            # 看你在 develop 上多了什么
git checkout -b rescue/local-develop      # 把这些 commit 救到新分支
git checkout develop
git reset --hard upstream/develop         # 强制对齐上游（你的 commit 已经在 rescue 分支里）
git push --force-with-lease origin develop
```

### 3.2 工作分支同步上游最新

如果你的 `feat/xxx` 分支已经做了一阵，上游又有新东西需要合进来：

```bash
git checkout feat/xxx
git fetch upstream

# 方式 A：merge（保留分支历史，更安全）
git merge upstream/develop

# 方式 B：rebase（线性历史更干净，但会改写 commit）
git rebase upstream/develop
```

**经验法则**：分支只有自己用 → rebase；分支已经被别人 fork 或 review → merge。

### 3.3 GitHub 网页同步

GitHub 上你的 fork 页面会显示 "This branch is N commits behind oceanbase:develop"，点 **Sync fork** → **Update branch** 也能同步。但只对 `develop` 这种和上游同名的分支有效，且只能 fast-forward。命令行更灵活。

---

## Part 4 — 提 PR 给上游

如果你的改动想贡献给 OceanBase：

```bash
# 1. 确保工作分支基于最新 upstream/develop
git checkout feat/xxx
git fetch upstream
git rebase upstream/develop               # 线性历史，方便 review

# 2. 推到 fork
git push -u origin feat/xxx
# 如果你 rebase 过、之前 push 过：
git push --force-with-lease origin feat/xxx

# 3. 在 GitHub 上开 PR
# 浏览器打开 https://github.com/JustinLamster/seekdb_explore
# GitHub 会提示 "Compare & pull request"
# 目标仓库选 oceanbase/seekdb，目标分支选 develop
```

**PR 描述要写清**：动机（为什么做）、方案（怎么做的）、测试（验证过什么）、风险（可能影响哪些地方）。参考已合并 PR 的格式。

---

## Part 5 — 常见坑与避雷

### 5.1 误 push 到 upstream

如果你按 [Part 1.2](#12-重新组织-remote关键) 把 `upstream` 的 push URL 设成 `DISABLED`，就推不出去。否则 GitHub 会拒（你没权限），但报错信息里可能会让你误以为是网络问题。

### 5.2 在 develop 上直接 commit

最常见的新手坑。如果发现自己已经在 develop 上做了几个 commit：

```bash
# 把这些 commit 转移到新分支
git checkout -b feat/rescue
# develop 回到上游状态
git checkout develop
git reset --hard upstream/develop
```

### 5.3 提交了大文件

GitHub 单文件硬上限 100MB（>50MB 会警告）。一旦 commit 了大文件，即使后续删除，git 历史里还在，仓库会臃肿。

```bash
# push 前自查
git diff --stat HEAD~1                    # 看上次 commit 改了什么
find . -size +50M -not -path './.git/*' -not -path './build_*'

# 万一已经 commit 但还没 push
git reset --soft HEAD~1                   # 撤销 commit，保留改动
# 把大文件加进 .gitignore，重新 commit

# 已经 push 出去——需要 git filter-repo 重写历史，麻烦
# 大文件应该用 Git LFS 或者放在仓库外
```

### 5.4 提交了 secrets / 私有信息

**一旦 push 了 token / 密码，立刻去对应平台旋转密钥，不要试图靠 force push 删除——你不知道有谁已经 fetch 了。**

清理本地历史：

```bash
# 用 git filter-repo（推荐）
pip install git-filter-repo
git filter-repo --path secret_file.yaml --invert-paths
```

### 5.5 行尾混乱导致 diff 异常

参考 [Part 1.4](#14-windows-行尾配置重要)。如果某个文件 `git diff` 显示整个文件都改了：

```bash
git checkout -- <file>                    # 重置该文件
# 检查 .gitattributes 是否有 text=auto / eol=lf 设置
```

### 5.6 push 时被拒（non-fast-forward）

```
! [rejected]        feat/xxx -> feat/xxx (non-fast-forward)
error: failed to push some refs
```

意味着远端有你本地没有的 commit。**不要无脑 `git push -f`**——可能覆盖别人的工作。

```bash
# 先看远端有什么
git fetch origin
git log HEAD..origin/feat/xxx             # 远端比本地多什么

# 如果远端只有你自己的 commit（自己另一台机器/网页直接改的）
git pull --rebase origin feat/xxx
git push origin feat/xxx

# 如果你本地 rebase 过，确认要覆盖远端
git push --force-with-lease origin feat/xxx
# --force-with-lease 比 --force 安全：远端如果有未知更新会拒，避免覆盖别人的 commit
```

---

## Part 6 — 命令速查表

| 场景 | 命令 |
|------|------|
| 看远程仓库 | `git remote -v` |
| 看当前分支 | `git branch --show-current` |
| 看所有分支 | `git branch -a` |
| 看状态 | `git status` |
| 看未 stage 改动 | `git diff` |
| 看已 stage 改动 | `git diff --staged` |
| 看历史（简洁） | `git log --oneline -20` |
| 看历史（图形） | `git log --oneline --graph --all -30` |
| 切分支 | `git checkout <branch>` 或 `git switch <branch>` |
| 新建分支 | `git checkout -b <branch>` 或 `git switch -c <branch>` |
| stage 文件 | `git add <file>` |
| stage 部分 hunk | `git add -p <file>` |
| 取消 stage | `git restore --staged <file>` |
| 撤销工作区改动 | `git restore <file>` |
| 提交 | `git commit -m "msg"` |
| 修改最后 commit | `git commit --amend` |
| 同步上游到本地 develop | `git fetch upstream && git merge --ff-only upstream/develop` |
| 推到 fork | `git push origin <branch>` |
| 第一次推（设置 tracking） | `git push -u origin <branch>` |
| 安全的强推 | `git push --force-with-lease origin <branch>` |
| 拉远端最新（rebase 模式） | `git pull --rebase origin <branch>` |
| 临时存改动 | `git stash` |
| 恢复 stash | `git stash pop` |
| 看 stash 列表 | `git stash list` |
| 删本地分支 | `git branch -d <branch>`（merge 过的） / `-D`（强删） |
| 删远端分支 | `git push origin --delete <branch>` |

---

## Part 7 — 应急操作

### 7.1 我刚才 commit 了不该 commit 的东西，还没 push

```bash
git reset --soft HEAD~1                   # 撤销 commit，保留所有改动在 stage
git reset HEAD~1                          # 撤销 commit，保留改动但取消 stage
git reset --hard HEAD~1                   # 撤销 commit + 丢弃所有改动（危险！）
```

### 7.2 我刚 push 了，想撤回

如果分支只有你自己用：

```bash
git reset --hard HEAD~1
git push --force-with-lease origin <branch>
```

如果别人也用这个分支（比如已经发了 PR，有人 review 了），**不要 force push**。改用 revert：

```bash
git revert HEAD                           # 生成一个反向 commit
git push origin <branch>
```

### 7.3 我把分支搞得一团糟，想从远端重新开始

```bash
git fetch origin
git checkout <branch>
git reset --hard origin/<branch>          # 本地强制对齐远端，丢弃所有本地改动
```

### 7.4 我误删了一个分支

```bash
git reflog                                # 看所有 HEAD 移动历史
# 找到删之前的 commit hash，例如 abc1234
git checkout -b <recovered-branch> abc1234
```

只要那个 commit 还在 reflog 里（默认保留 90 天），就能找回来。

### 7.5 我想看某个文件历史上某个版本

```bash
git log -- <file>                         # 看这个文件的 commit 历史
git show <commit>:<file>                  # 看某个 commit 时该文件的内容
git checkout <commit> -- <file>           # 把工作区的该文件回退到某个 commit 的版本
```

### 7.6 我 merge / rebase 出现冲突

```bash
# 查看冲突文件
git status                                # 标记 "both modified" 的就是冲突文件

# 编辑冲突文件，解决 <<<<<<< / ======= / >>>>>>> 标记
# 解决后：
git add <file>
git commit                                # merge 的情况（会自动生成 merge commit message）
# 或
git rebase --continue                     # rebase 的情况

# 想放弃这次 merge / rebase：
git merge --abort
git rebase --abort
```

---

## 附录 — 推荐每日工作开头的"热身"流程

养成习惯，每次开始干活前跑一遍：

```bash
# 1. 看现在站在哪
git status
git branch --show-current

# 2. 同步上游
git fetch upstream
git fetch origin

# 3. 看上游有没有新东西
git log HEAD..upstream/develop --oneline | head -10

# 4. 如果当前在 develop，fast-forward
# 如果在工作分支，决定要不要 rebase / merge upstream
```

5 秒钟就能知道仓库当前的"势能差"，避免基于过期代码工作。
