---
name: git-sync
description: Git 初始化、暂存、提交、首次配置 GitHub 远程与推送、以及日常推送的中文分步说明。 涵盖 ~/Documents 下的项目与 Claude Code 全局 Skill 目录 ~/.claude/skills。 适用于询问推送到 GitHub、git_sync、远程 origin、commit 与 push 的区别、未跟踪文件、 main 分支、HTTPS 令牌，或将本地文件夹同步到 GitHub 等场景。 正文与 workflow-text-format/SKILL.md 双向锁定；增改须符合该 Skill「层级标识」等规则。
tools: ["Read", "Bash"]
model: sonnet
effort: low
---
# 0. 生成规范

- **双向锁定**：本文件 `/Users/jiachenyang/.claude/skills/git-sync/SKILL.md` 与 `/Users/jiachenyang/.claude/skills/workflow-text-format/SKILL.md` 建立 **双向锁定**：对本文件正文的增删改须符合后者 **`# 2`**～**`# 4`**；在本文件中引入后者尚未记载的文本格式时，须 **回写** `workflow-text-format/SKILL.md`；与存量 **冲突** 时 **停手** 并向用户说明，请用户选定方案后再改。

- **格式依据**：工作流正文与树形层级的 **整体逻辑** 以 **workflow-text-format** 为准。

- **树形次级**：新增、调整或加深嵌套 `├─`、`└─`、`│` 时，须先查阅 **workflow-text-format** **`# 1`** 下 **树形次级增改** 与 **`# 2` · 树形次级**，定稿前完成该 Skill 所列自检。

# 1. 使用说明

- **概述**：首次将本地推到 GitHub，可按 **`# 2`**～`**# 6`** 顺序操作；多机同步见 **`# 7`**。「项目根目录」指你希望纳入版本控制的那一层目录：常见为 **`~/Documents` 下的某个项目**；若用 Git 管理 **Claude Code 全局 Skill**，则多为 **`~/.claude/skills`**（勿随意改路径，以免 Claude Code 读不到）。按你的场景选用对应 `cd`。

# 2. 在 GitHub 上新建仓库

- **操作**：
    ├─ 登录 [GitHub](https://github.com)，右上角 **+** → **New repository**。
    ├─ 填写 **Repository name**，可选 **Description**。
    ├─ 建议勾选 **Public** 或 **Private** 按需选择。
    ├─ **不要**勾选「Add a README / .gitignore / license」（本地已有代码时，避免首次推送产生无关冲突；若仓库完全是空的也可以后续再补）。
    └─ 点击 **Create repository**，页面会显示仓库地址，记下 **HTTPS** 或 **SSH** 地址，例如：
        ├─ HTTPS：`https://github.com/你的用户名/仓库名.git`
        └─ SSH：`git@github.com:你的用户名/仓库名.git`

# 3. 本地初始化 Git（若尚未初始化）

- **git init 与目录**：

`**git init**`：在当前目录创建 `.git`，该目录成为 Git 仓库，之后才能跟踪变更、`commit`、配置远程并 `push`。若已有 `.git`（例如克隆的项目），已是仓库，**勿重复**执行。

先 `cd` 到项目根目录，再 `git init`。下两行 `cd` 对应两种场景，**只执行与你相符的那一行**（勿两条连续执行，否则只会留在最后进入的目录）。

```bash
# 常规项目（改文件夹名；~/Documents 即「文稿」）
cd ~/Documents/你的项目文件夹

# 或：Claude Code 全局 Skill（勿改路径）
cd ~/.claude/skills

# 初始化
git init
```

- **提交者信息**：

若本机从未配置过提交者信息（仅需做一次）：

`**user.name**` / `user.email`：写入每次 `commit` 的作者信息；**不是**登录 GitHub，也**不替代** `push` 所需的 HTTPS 令牌或 SSH。

**邮箱与 GitHub**：与账号内**已验证邮箱**或 **noreply** 一致时，提交在网页上易归到你的账号；不一致时仍可推送，但可能显示为未关联用户。

```bash
# 全局：写入 commit 的作者显示名
git config --global user.name "yjc"

# 全局：写入 commit 的作者邮箱（建议与 GitHub 已验证邮箱或 noreply 一致）
git config --global user.email "yangchengchengbadi@gmail.com"
```

# 4. 本地更新暂存区

- **范围**：本节在**本机**完成两件事：**更新暂存区**（`git add`），再 **`commit` 成历史**；不含推送到 GitHub 的 **`push`**。上传远程见 **`# 5`** 或 **`# 6`**。

```bash
# 暂存：当前目录及子目录中待纳入版本控制的改动（含未跟踪文件）
git add .

# 核对：暂存区与工作区是否与你预期一致
git status

# 提交：将暂存区快照记为一次历史节点，-m 为本次说明
git commit -m "initial commit"
```

- **commit 与 GitHub**：`commit` 只在**本机**把暂存区内容写入 `.git` 的历史（生成一次「快照」），**不会**自动同步到 GitHub。网页上要看到代码，还须完成 **`# 5`** 里的 **`git push`**（并确保已 `git remote add origin …`）。

- **分支改名为 main**：若默认分支名是 `master` 而你想用 `main`（与 GitHub 默认一致），可先查看历史再改名：

```bash
# 查看本地提交历史（当前分支）；想简短可改用：git log --oneline
git log

# 将当前分支重命名为 main，与 GitHub 默认分支名一致（默认已是 main 时可略过）
git branch -M main

```

# 5. 指定仓库并执行首次推送

- **远程与首次 push**：把下面远程地址换成你在 **`# 2`** 里复制的仓库 URL（**每个本地仓库通常只做一次** `remote add`）。日常再次修改后的推送见 **`# 6`**。

```bash
# 首次：添加名为 origin 的远程，指向你的 GitHub 仓库  (e.g. https://github.com/yangchengchengbadi-pixel/test.git)
git remote add origin ?

# 首次推送：将本地 main 推到远程，-u 建立跟踪，之后同分支多数可直接 git push
git push -u origin main

# 要以本地为准覆盖远程 main 时用（会改写远程历史；勿与首次推送混用）。--force-with-lease：若远程在你上次 fetch 之后又有新提交则拒绝推送，比 --force 不易误覆盖他人
# git push --force-with-lease -u origin main
```

若远程仓库默认分支仍是 `master`，则把上面命令里的 `main` 改成 `master`，或与 GitHub 网页上显示的分支名保持一致。

# 6. 日常修改后再推送

- **流程**：`origin` 已添加、且做过一次带 `-u` 的 `push` 之后，**以后每次改完代码**，按顺序做三件事即可：**暂存 → 提交 → 推送**。

```bash
# 常规项目（改文件夹名；~/Documents 即「文稿」）
cd ~/Documents/?

# 或：Claude Code 全局 Skill（勿改路径）
cd ~/.claude/skills

# 暂存：把本次要纳入提交的改动加入暂存区（也可写成具体文件名）
git add .

# 核对（可选）
git status

# 提交：在本地生成一次新历史，说明信息改成与本次改动相符的文案。
git commit -m ?

# 推送：若已用 -u 建立过跟踪，多数情况下可直接执行 git push
git push
```

# 7. 双机协作：机器 B 如何 pull（假定云端已有内容）

- **总则**：任一台机器在**改代码或 `push` 之前**，若远程上已有别人新推的提交，都需先与云端对齐（下面以机器 B 场景写得最多；**机器 A 在 B 推送之后，同样用本段「同一套历史」款的 `pull` 即可**）。**以 `#` 开头的是说明**；**不以 `#` 开头的行才是要执行的命令**。默认分支若不是 `main`，把各块中的 `main` 改成远程实际分支名。

- **尚无仓库**：**B 上还没有这份仓库时**，地址用 **`# 2`** 里的 HTTPS 或 SSH。**克隆前必须先 `cd` 到正确位置**：两种放法对「当前目录」的要求不同，二选一执行对应块；以 `#` 开头的是说明。

```bash
# 方式一：在「父目录」下克隆 → 会新建子目录（名与 GitHub 仓库名一致，多为「仓库名」），再进入即仓库根
cd ~/Documents
git clone https://github.com/你的用户名/仓库名.git
cd 仓库名

# 方式二：先进入「将用作仓库根的空目录」→ 再克隆到当前目录（不新建子目录；该目录须为空，否则不能 clone 到点号）
cd ~/Documents/你的项目文件夹
git clone https://github.com/你的用户名/仓库名.git .
```

- **同一套历史**：**本机目录已与该 GitHub 仓库是「正常」同一套历史**（常见：从云端 `git clone` 过；或机器 A 当初 `init` / 首次 `push` 建起远程后一直在本机开发）：别人（如机器 B）已向远程推送新提交时，进入仓库根拉取即可。不限于机器 B；**机器 A 在 B `push` 之后也同样可以 `pull`**。

```bash
# 进入仓库根（两条只选一条）
cd ~/Documents/你的项目文件夹
# cd ~/.claude/skills

# 拉取远程 main，与机器 A 推送对齐
git pull origin main --no-rebase
```

- **非 clone 绑定**：**B 上 `origin` 已指向该仓库，但当初并不是从云端 `clone` 出来的**（例如本机 `git init` 后再绑的远程）：**不要**用上面直接 `pull`。**先备份**你还要保留的文件到目录外，再在**父目录**删掉整个项目文件夹，然后重新 `clone`（文件夹名按实际改）：

```bash
# 进入放项目的父目录（二选一；全局 Skill 时父目录一般为 ~/.claude）
cd ~/Documents
# cd ~/.claude

# 备份完成后删除原文件夹（名称须与下面 clone 的最后一项一致）
rm -rf 你的项目文件夹

# 从云端重新克隆（URL 见文档 # 2；最后一项为本地文件夹名）
git clone https://github.com/你的用户名/仓库名.git 你的项目文件夹

# 进入新克隆的仓库根
cd 你的项目文件夹
```

- **拉取后**：拉取或对齐成功后，在 B 上按 **`# 6`** 改代码、`git add`、`git commit`、`git push`。若 `git push` 提示未设置上游：

```bash
# 首次在本机为该分支建立跟踪（分支名按实际）
git push -u origin main
```

# 8. 常见问题

- **条目**：
    ├─ **Authentication failed**：HTTPS 需使用 [Personal Access Token](https://github.com/settings/tokens) 代替密码；或改用 SSH，并配置本机 SSH 公钥到 GitHub。
    ├─ **remote origin already exists**：改用 `git remote set-url origin 仓库地址` 修改地址，或先 `git remote remove origin` 再重新 `add`。
    └─ **拒绝推送因非快进**：说明远程已有提交，需先拉取再推；合并方式见 **`# 7`** 中 `git pull origin main --no-rebase`，或按团队规范使用 `git pull origin main --rebase`，或确认是否推错了仓库。

完成 `git push` 后，在 GitHub 仓库页面刷新即可看到代码。

# 9. Instructions（供 Agent）

- **条目**：
    ├─ `git commit -m` 的说明文字**必须加引号**（含空格或多词时），否则会出现 `pathspec 'commit' did not match` 类错误。
    ├─ **`# 3`、`# 6`，以及 `# 7` 里与场景互斥的 `cd`**：同一节里**两条备选路径的 `cd`（**如** `~/Documents/…` 与 `~/.claude/skills`）**只能选一条**；**`# 7`「尚无仓库」**中**方式一**与**方式二**也二选一。若选方式一，「`cd` 父目录 → `clone` → `cd` 仓库名」是**同一路线**的连续步骤，不与此条矛盾。
    ├─ 根据用户场景提示替换占位符：`???`、仓库 URL、`你的项目文件夹`；`user.name` / `user.email` 示例请提醒用户改为自己的信息，勿照抄他人邮箱。
    └─ **`# 7`** 中「非 clone 绑定」款含删除原文件夹与 `rm -rf`：须提醒用户**先备份**；命令中的路径、文件夹名须替换为实际值，避免删错目录。
