---
title: Git&Github
description: 
date: 2025-11-10
categories:
    - 
    - 
---


# Git & GitHub 入门操作详解

## Git的 "四个空间" 模型

### 工作区 (Working Directory)

它就是你在电脑上能直接看到的项目文件夹，包含着所有项目文件。你用你的代码编辑器（比如 VS Code）打开项目，进行添加、删除、修改代码等所有操作，都是在工作区里完成的。它是最直观、最原始的空间。

**补充说明：** 工作区本质上就是普通文件系统中的一个目录，你可以像操作任何文件夹一样操作它。Git 在这里的唯一作用是"观察"——它会实时监控文件内容的变化，但**不会自动保存**这些变化。你可以随时用 `git status` 命令查看哪些文件被修改了。工作区是开发者与代码直接交互的地方，所有创意和编码都发生在这里。

### 暂存区 (Staging Area)

工作区的文件修改后，你不能直接把它"存档"。你需要一个中间步骤，让你选择哪些修改要被打包存档。暂存区就是这个地方。

为什么需要它？ 假设你同时修改了3个文件，但只有2个是关于"修复登录bug"的，另1个是改着玩的。你就可以只把那2个文件"加入购物车"（git add），然后"结账"（git commit）。暂存区给了你一个精挑细选和反悔的机会，确保每一次提交都是一个逻辑完整的单元。

**详细解释：** 暂存区是一个隐藏的索引文件（位于 `.git/index`），它记录了即将被提交的文件快照。当你执行 `git add` 时，Git 实际上做了三件事：
1. 计算文件的 SHA-1 哈希值（内容指纹）
2. 将文件内容压缩并存储到 `.git/objects` 目录
3. 在暂存区中更新该文件的索引信息

这个设计让你可以**分批次、精细化地组织提交内容**。比如你可以先用 `git add file1.c` 添加一个功能文件，调试后再 `git add file2.c`，最后一次性提交。如果想移除某个已暂存的文件，使用 `git reset HEAD <file>` 可以将其移回工作区。

### 本地仓库 (Local Repository)

当你执行"结账"命令（git commit）后，暂存区（购物车）里所有的内容会被打包成一个版本（我们称之为一次"提交"），然后被永久地保存在你本地电脑的仓库里。

这个本地仓库位于你项目的 `.git` 隐藏文件夹中。它就像一个精密的数据库，记录了你项目从创建之初到现在的每一次提交，包括谁、在什么时间、修改了什么内容。这是你个人的、完整的项目历史备份。

**深入理解：** `.git` 文件夹是 Git 的核心，其内部结构包括：
- `HEAD`：指向当前所在的分支
- `objects/`：存储所有数据对象（提交、树、文件内容）
- `refs/`：存储分支和标签的指针
- `config`：仓库的配置信息

每次 `git commit` 会创建一个**提交对象**（包含提交信息、作者、时间戳和指向父提交的指针），一个**树对象**（记录目录结构），以及若干个**文件对象**（存储文件内容）。这些对象通过哈希值相互关联，形成一条不可篡改的历史链。这意味着**你的完整历史永远保存在本地**，即使断网也能查看和回溯。

### 远程仓库 (Remote Repository)

个人开发时，有前三个空间就足够了。但要团队协作，就需要一个所有成员都能访问的中央服务器来同步代码。这个服务器就是远程仓库。

它通常托管在像 GitHub、GitLab 或 Gitee 这样的代码托管平台上。团队成员可以：
- 从远程仓库克隆 (clone) 一份完整的项目到自己的本地仓库。
- 将自己本地仓库中新的提交推送 (push) 到远程仓库，分享给他人。
- 从远程仓库拉取 (pull) 别人的提交，更新自己的本地代码。

**工作流程：** 在工作区写代码 → add 到暂存区 → commit 到本地仓库 → push 到远程仓库。

**补充说明：** 远程仓库本质上是你本地仓库的**网络副本**，但它还提供了额外的协作功能：
- **权限管理**：控制谁可以读写代码
- **持续集成**：自动运行测试和部署
- **代码审查**：通过 Pull Request 机制进行代码评审
- **问题追踪**：关联 Issue 和提交

需要注意的是，`push` 和 `pull` 操作都是**网络密集型**的，需要认证（SSH 密钥或 Personal Access Token）。

---

## 准备 Git 环境

### 安装 Git

Git 是一个命令行工具，首先需要将它安装到你的电脑上。

**安装细节：**
- **Windows**：下载的安装包包含了完整的 Unix 工具集（通过 Git Bash），让你能在 Windows 上使用 Linux 命令。选择 `Override the default branch name` 并设为 `main` 是为了与 GitHub 的新规范保持一致，避免后续分支命名混乱。
- **macOS**：系统自带的 Git 版本可能较旧，如需最新版可用 Homebrew 安装：`brew install git`
- **Linux**：包管理器安装后，建议验证版本 `git --version`，必要时添加 Git 官方 PPA 获取最新版本

安装完成后，打开你的终端（Windows 用户请使用 Git Bash），输入以下命令来验证是否安装成功：

```bash
git --version
```

**执行效果：** 终端会打印类似 `git version 2.30.0` 的版本号。如果出现"command not found"，说明 Git 未正确安装或未加入系统 PATH。

### 首次配置：为你的代码署名

安装好 Git 后，还有一件至关重要的事情要做：告诉 Git 你是谁。

**配置原理：** 每次提交时，Git 会将这些信息写入提交对象中，永久记录在版本历史里。这不仅是协作礼仪，更是法律层面确认代码著作权的重要依据。

请在终端中运行以下两条命令，将引号中的内容替换成`你自己的用户名和邮箱`。

```bash
# 设置你的全局用户名（提交时会显示这个名字）
git config --global user.name "Your Name"

# 设置你的全局邮箱（GitHub 会用它来关联你的提交和账号）
git config --global user.email "your.email@example.com"
```

**参数解析：**
- `--global`：将配置写入 `~/.gitconfig` 文件，这台电脑上**所有仓库**都会使用这个配置。如果只想对当前仓库生效，可去掉该参数，配置将保存在 `.git/config` 中
- `user.name`：可以是任意字符串，但建议使用 GitHub 用户名
- `user.email**：**必须**与 GitHub 账号的邮箱一致，否则提交无法关联到你的 GitHub 账户

**验证配置：**
```bash
git config --list  # 查看所有配置
git config user.name  # 单独查看用户名
```

**配置级别优先级：** 项目级配置（.git/config）> 全局配置（~/.gitconfig）> 系统级配置（/etc/gitconfig）

### 注册 GitHub

- Git 是工具：一个在你电脑上运行的命令行软件
- GitHub 是平台：一个网站，提供了远程仓库的托管服务，并围绕代码构建了一个庞大的开发社区

**重要提示：** 注册 GitHub 后，务必完成以下设置：
1. **验证邮箱**：否则无法创建仓库
2. **设置 SSH 密钥**：避免每次推送都输入密码
   ```bash
   ssh-keygen -t ed25519 -C "your.email@example.com"
   ```
   然后将 `~/.ssh/id_ed25519.pub` 的内容添加到 GitHub → Settings → SSH and GPG keys
3. **启用双因素认证**：保护账户安全

---

## 本地管理你的代码

Git 最核心的三个命令：
- `git init`：初始化一个新仓库
- `git add`：将文件变更添加到暂存区
- `git commit`：将暂存区的变更永久保存到版本库

### 创建你的项目

创建一个新文件夹，这个可以放在你喜欢放的位置。

```bash
# 创建项目目录
mkdir my-first-git-project

# 进入这个文件夹
cd my-first-git-project
```

**命令解释：**
- `mkdir`（make directory）：创建新目录
- `cd`（change directory）：切换当前工作目录
- **最佳实践**：项目名称使用小写字母、连字符（kebab-case），避免空格和特殊字符

现在，这个文件夹就是我们的工作区 (Working Directory)。它是我们编写代码、创建文件、进行所有实际工作的地方。但目前，它还只是一个普通的文件夹，Git 还不知道它的存在。

### 初始化仓库 (git init)：开启版本控制

为了让 Git 开始管理这个文件夹，我们需要在这个文件夹内部运行一个命令，来"激活"它。

```bash
git init
```

**命令详解：**
- `git init` 是 "initialize"（初始化）的缩写
- 它会在当前目录创建 `.git` 子目录，这是 Git 的"大脑"
- 执行后，当前目录及其所有子目录都被纳入版本控制范围
- **输出含义**：`Initialized empty Git repository in ...` 表示成功创建了一个空的 Git 仓库

**`.git` 目录结构预览：**
```
.git/
├── HEAD         # 当前分支指针
├── config       # 仓库配置文件
├── description  # 仓库描述
├── hooks/       # 钩子脚本
├── info/        # 全局排除文件
├── objects/     # 数据对象存储
└── refs/        # 分支和标签指针
```

**警告：** 除非你非常清楚自己在做什么，否则永远不要手动修改 `.git` 文件夹里的任何内容！把它当成 Git 的专属地盘。

从现在起，my-first-git-project 文件夹就不再普通了，它已经是一个 Git 仓库了。

### 创建文件并检查状态 (`git status`)

让我们在工作区里创建第一个文件。一个项目通常会有一个 README.md 文件，用来描述项目信息。

```bash
# 创建并写入一句话到 README.md 文件中
echo "Hello, Git! This is my first project." > README.md
```

**命令解析：**
- `echo`：输出文本到终端
- `>`：重定向符号，将输出写入文件（覆盖模式）
- `>>`：如果要追加内容，可使用双大于号

现在，我们的工作区有新内容了。Git 是否察觉到了呢？让我们来问问 Git 当前的状态。`git status` 是你在使用 Git 过程中最常用、也最有用的命令，它会告诉你仓库当前的状态。

```bash
git status
```

**命令深度解析：**
- `git status` 比较工作区、暂存区和 HEAD（最后一次提交）之间的差异
- 它输出三部分信息：当前分支、暂存区状态、工作区状态
- **高频使用**：建议每次 `add` 和 `commit` 前都运行此命令确认状态

终端会返回一些信息，其中最关键的是：

```bash
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.md

nothing added to commit but untracked files present (use "git add" to track)
```

解读一下：
- **Untracked files (未跟踪的文件)**：Git 发现了一个新面孔 README.md，但它并不在版本管理的范围内。对于 Git 来说，这个文件就像不存在一样，任何修改都不会被记录。
- **use "git add ..."**：Git 很贴心地提示我们，可以使用 `git add` 命令来开始跟踪这个文件。

### 添加暂存区 (`git add`)：准备提交

现在，我们要把 README.md 文件从工作区的"待定"状态，转移到暂存区 (Staging Area)，表示"我确定要把这个文件的当前版本加入到下一次存档中"。

```bash
git add README.md
```

**命令详解：**
- 语法格式：`git add <文件路径>` 或 `git add .`（添加所有变更）
- **底层操作**：Git 计算文件的 SHA-1 哈希值，将文件内容压缩存入 `.git/objects`，并在暂存区更新索引
- **常见用法**：
  - `git add .`：添加工作区所有修改（不包括删除）
  - `git add -A`：添加所有修改、删除和新增
  - `git add -p`：交互式添加，可以选择文件的某些部分（patch）

这个命令没有任何输出，是正常的。想知道发生了什么？再次使用我们的好朋友 `git status`：

```bash
git status
```

输出变了：
```bash
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   README.md
```

解读一下：
- **Changes to be committed (待提交的变更)**：README.md 已经进入了暂存区，整装待发，准备被正式存档。
- **暂存区的价值**：就像一个购物车，你可以不断地往里添加（`git add`）或移除商品（`git rm --cached`），直到你确认所有要买的东西都在里面了，才去结账（`git commit`）。这种机制让你可以**精确控制每次提交的内容**，保持提交的原子性。

**撤销暂存操作：** 如果误将文件加入暂存区，可用：
```bash
git rm --cached README.md  # 将文件移回工作区，不改变文件内容
```

### 提交到仓库 (`git commit`)：完成存档

`git commit` 命令会将暂存区里所有的内容打包，创建一个新的版本记录（一个快照），并永久地保存在你的本地仓库里。

**提交机制深度解析：**
- Git 会创建一个**提交对象**（commit object），包含：
  - 作者信息（name, email）和提交者信息
  - 提交时间戳
  - 父提交的指针（形成历史链）
  - 指向树对象（目录结构）的指针
  - 提交信息（commit message）
- 每个提交都有唯一的 SHA-1 哈希值作为 ID
- 提交后，HEAD 指针会自动指向新的提交

每一次提交，都必须附带一条提交信息 (commit message)，用 `-m` 参数来指定。这条信息是对本次修改的简短描述，例如 "修复了xx bug" 或 "增加了用户登录功能"。良好的提交信息是团队协作的基石！

```bash
git commit -m "Initial commit: Add README file"
```

**参数说明：**
- `-m`：message 的缩写，后面紧跟提交信息
- **提交信息规范**：
  - 使用现在时（"Add" 而不是 "Added"）
  - 首字母大写，不超过 50 个字符
  - 空一行后添加详细说明（如果必要）

终端会显示类似如下的输出，告诉你这次提交的概要：

```bash
[master (root-commit) a1b2c3d] Initial commit: Add README file
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

**输出解析：**
- `[master (root-commit) a1b2c3d]`：master 分支，首次提交（没有父节点），提交 ID 前 7 位
- `1 file changed, 1 insertion(+)`：统计信息
- `create mode 100644`：文件权限（普通文件）

恭喜你！你已经成功完成了第一次提交！

现在，如果我们再运行 `git status`，会得到：

```bash
On branch master
nothing to commit, working tree clean
```

这表示：暂存区已清空，工作区所有文件都与最新提交保持一致。

### 总结

完成了一个完整的本地 Git 操作循环：
1. 在工作区修改代码（我们创建了 README.md）
2. 使用 `git add` 将修改添加到暂存区（挑选要存档的内容）
3. 使用 `git commit` 将暂存区内容提交到本地仓库（正式存档）

这个 "修改 -> add -> commit" 的循环，是你未来 90% 的本地 Git 操作，请务必牢记。

**练习建议：** 尝试再修改 README.md，添加一行文字，然后重复 `add -> commit` 流程，观察 `git log` 的变化。

---

## 版本历史的查看与回溯

### 1. 查看提交历史 (`git log`)

你的 `.git` 仓库就像一本详细的日记，记录了每一次 commit 的所有细节。要阅读这本日记，我们使用 `git log` 命令。

```bash
git log
```

**命令深度解析：**
- `git log` 读取 `.git/refs/heads/` 目录下的分支指针，然后遍历提交链
- 默认从 HEAD 开始，按时间倒序显示
- 每条记录显示完整的 SHA-1 哈希、作者、日期和提交信息

运行后，你会看到类似这样的输出，从最新到最旧排列：

```bash
commit a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 (HEAD -> master)
Author: Your Name <your.email@example.com>
Date:   Mon Oct 26 20:21:42 2023 +0800

    Initial commit: Add README file
```

每一条记录都包含了：
- **Commit ID (哈希值)**：那一长串唯一的 `a1b2c3d...` 字符串，是每个版本的身份证号。这是 SHA-1 哈希值，由提交内容计算得出，任何微小改动都会改变这个值。
- **Author (作者)**：你之前配置的用户名和邮箱
- **Date (日期)**：提交发生的时间
- **Commit Message (提交信息)**：你在 `-m` 后面写的那段描述

**实用技巧：**
- `git log --oneline`：单行显示，只显示哈希前 7 位和提交信息，非常清晰
- `git log --graph --oneline --all --branches`：图形化显示分支合并历史
- `git log -p`：显示每次提交的具体差异
- `git log --author="Your Name"`：筛选特定作者的提交

### 2. 版本回溯：回到过去

想象一下，你刚写了一堆代码，结果发现把事情搞砸了，想回到某个正常工作的版本。Git 的时光机能帮你实现！

#### 场景：我只想恢复某一个文件

这是最安全、最常用的操作。假设你不小心把 README.md 文件改乱了，想把它恢复到上一个提交时的样子。

首先，用 `git log --oneline` 找到你想回到的那个版本的 Commit ID（比如 a1b2c3d）。

然后运行：

```bash
# 格式: git checkout <commit_id> – <file_path>
git checkout a1b2c3d -- README.md
```

**安全恢复机制详解：**
- 这个命令**只影响指定文件**，不改变仓库的当前状态（HEAD 位置不变）
- **工作原理**：Git 从 `.git/objects` 中提取指定版本的文件内容，覆盖工作区的文件
- **使用场景**：
  - 误删或改乱某个文件
  - 需要查看历史版本但不切换分支
  - 恢复被错误修改的配置文件

**恢复后的操作：**
- 恢复的文件会自动添加到暂存区
- 如果想撤销恢复，可用 `git reset HEAD README.md` 移回工作区
- 确认无误后，可用 `git commit` 提交这次恢复操作

瞬间，你的 README.md 文件就从"灾难现场"恢复到了 a1b2c3d 这个版本时的状态。这种方式只会影响指定的文件，项目中的其他文件安然无恙。

#### 场景：我要让整个项目回到过去（危险操作！）

如果你想让整个项目（所有文件）都回滚到某个旧版本，可以使用 `git reset`。

 **⚠️ 警告：这是一个有破坏性的操作！**  它会**永久丢弃**你在那个旧版本之后的所有修改和提交。请三思而后行。

```bash
# 格式: git reset --hard <commit_id>
git reset --hard a1b2c3d
```

**三种 reset 模式详解：**

Git 提供了三种不同"硬度"的回滚方式：

1. **`--soft`**（软重置，安全）：
   ```bash
   git reset --soft a1b2c3d
   ```
   - 只移动 HEAD 指针，不改变暂存区和工作区
   - 效果：a1b2c3d 之后的提交被"撤销"，但修改保留在暂存区
   - 用途：合并多个提交为一个（重新整理提交历史）

2. **`--mixed`**（混合重置，默认，较安全）：
   ```bash
   git reset --mixed a1b2c3d
   ```
   - 移动 HEAD 指针，重置暂存区，但**保留工作区**
   - 效果：a1b2c3d 之后的提交被撤销，修改保留在工作区（未暂存）
   - 用途：撤销提交但需要保留修改内容

3. **`--hard`**（硬重置，危险）：
   ```bash
   git reset --hard a1b2c3d
   ```
   - 移动 HEAD 指针，重置暂存区和工作区
   - 效果：**彻底删除** a1b2c3d 之后的所有提交和修改，无法恢复
   - 用途：本地实验性开发彻底失败，需要完全放弃

执行后，你的整个工作区都会被重置到 a1b2c3d 这个版本的状态。之后的所有更改都仿佛从未发生过。

 **⚠️ 重要安全提示：**  如果已经 `push` 到远程仓库，使用 `reset` 后再次 `push` 会**被拒绝**（因为改变了公共历史）。此时应使用 `git revert` 创建"反向提交"来撤销更改，而不是删除历史。

---

## "多人模式"：与 GitHub 协作

到目前为止，我们所有的操作都在本地电脑上。代码的备份和分享都无法实现。现在，是时候把我们的本地仓库连接到 GitHub 这个"云端档案馆"，开启真正的协作之旅了。

与远程仓库交互，通常有两种起点：
1. 从无到有：你先在本地创建了一个项目，然后想把它推送到 GitHub 上分享和备份。
2. 从有到有：项目已经存在于 GitHub 上（例如公司的项目或开源项目），你需要把它复制到本地开始工作。

我们分别来看这两种情况。

### 场景一：推送本地新项目到 GitHub

这正是我们当前的状况。我们已经在本地创建了 my-first-git-project 并有了一次提交。现在把它推上去。

#### 1. 在 GitHub 上创建空的远程仓库

首先，你需要为本地项目在 GitHub 上准备一个"家"。

- 登录你的 GitHub 账号
- 点击右上角的 + 号，选择 New repository
- Repository name (仓库名)：建议和你的本地文件夹名保持一致，例如 my-first-git-project
- Description (描述): 简单描述一下你的项目
- 保持 Public (公开) 状态，任何人都可以看到你的代码
- **重要：不要勾选 "Add a README file", "Add .gitignore", "Choose a license"**。因为我们本地已经有项目了，需要创建一个完全空的仓库来接收我们的代码
- 点击 Create repository

**创建策略说明：**
- **空仓库**：适合已有本地项目，避免历史冲突
- **带 README 的仓库**：适合从零开始，GitHub 会帮你初始化第一个提交
- 如果误选了初始化选项，Git 会提示"拒绝合并不相关历史"，需要用 `git pull --allow-unrelated-histories` 强制合并

#### 2. 连接本地与远程 (`git remote add`)

创建成功后，GitHub 会给你一个仓库地址，通常是 `https://github.com/YourUsername/my-first-git-project.git`。

回到你的终端，在本地项目文件夹里，运行以下命令，告诉本地 Git 这个远程仓库的存在：

```bash
# 格式: git remote add <远程仓库别名> <远程仓库URL>
git remote add origin https://github.com/YourUsername/my-first-git-project.git
```

**命令详解：**
- `remote` 子命令管理远程仓库列表
- `add`：添加新的远程仓库
- **origin**：是我们给这个远程仓库起的一个**别名**，这是 Git 的一个通用惯例，代表"主要的、默认的远程仓库"
- **URL 类型**：
  - HTTPS：`https://github.com/...`（需要每次输入密码或 Token）
  - SSH：`git@github.com:...`（推荐，配置 SSH 密钥后免密）

**验证连接：**
```bash
git remote -v  # 查看所有远程仓库及其 URL
git remote show origin  # 查看远程仓库详细信息
```

#### 3. 推送你的代码 (`git push`)

连接已经建立，现在我们可以把本地的代码历史"推送"到 origin（也就是 GitHub）上了。

```bash
git push -u origin main
```

**参数深度解析：**
- `push`："推送"动作，将本地提交上传到远程
- `origin`：指定要推送到哪个远程仓库（就是我们刚才设置的别名）
- `main`：指定要推送本地的哪个分支（GitHub 现在默认分支名为 main）
-  **`-u`**  ：`--set-upstream` 的缩写，**极其重要**。它会：
  1. 在远程创建 main 分支（如果不存在）
  2. 将本地的 main 分支与远程的 main 分支**关联**起来
  3. 设置默认值，这样未来只需输入 `git push` 即可

第一次推送时，系统可能会提示你输入 GitHub 的用户名和密码（或者 Personal Access Token）来完成认证。**推荐使用 Personal Access Token**，因为 GitHub 已停止支持密码认证。

推送成功后，刷新你的 GitHub 仓库页面，你会惊喜地发现，你本地的 README.md 文件和提交历史已经完完整整地出现在了网页上！

**后续推送简化：**
关联后，以后只需：
```bash
git push  # 自动推送到已关联的远程分支
git push origin main  # 完整写法
```

### 场景二：克隆一个已存在的项目 (`git clone`)

这是更常见的协作方式。比如你要加入一个新团队，第一件事通常就是从 GitHub 把项目代码克隆到你的电脑上。

`git clone` 命令帮你一步到位。

- 打开你想克隆的项目的 GitHub 页面
- 点击绿色的 "Code" 按钮，复制 HTTPS 或 SSH 地址
- 打开你的终端，`cd` 到你想存放这个项目的目录下（比如 Desktop 或 Documents）
- 运行 git clone 命令：

```bash
# 格式: git clone <远程仓库URL>
git clone https://github.com/some-user/some-awesome-project.git
```

**完整克隆过程详解：**
这个命令会做几件美妙的事情：
1. **创建目录**：在当前目录下创建一个名为 `some-awesome-project` 的新文件夹
2. **下载所有数据**：将远程仓库的所有文件、完整版本历史、所有分支和标签都下载到 `.git` 文件夹
3. **检出工作区**：自动将最新的代码文件解压到工作区，你可以立即开始编辑
4. **自动配置远程**：帮你设置好远程仓库的连接，并命名为 `origin`，你无需再手动 `git remote add`
5. **默认分支**：自动切换到远程仓库的默认分支（通常是 main 或 master）

**克隆选项：**
```bash
git clone --depth 1 <URL>      # 浅克隆，只下载最近一个提交（节省空间）
git clone -b develop <URL>     # 克隆后自动切换到 develop 分支
git clone --single-branch <URL> # 只克隆特定分支
```

克隆完成后，`cd some-awesome-project` 进入文件夹，你就可以开始工作了。

**克隆后的状态：**
```bash
git remote -v          # 会看到 origin 已配置好
git branch -a          # 查看所有本地和远程分支
git log --oneline      # 查看完整历史
```

### 后续操作：保持同步 (`git pull`)

无论你是通过 push 还是 clone 开始的远程协作，当你的同事（或者未来的你在另一台电脑上）更新了 GitHub 上的代码后，你就需要将这些最新的变更同步到你的本地仓库。这时使用 `git pull`：

```bash
git pull origin main
```

**完整同步机制详解：**
`git pull` 实际上是**两个命令的组合**：`git fetch` + `git merge`

1.  **`git fetch`**  ：下载阶段
   - 从远程仓库（origin）的 main 分支下载所有新的提交和引用
   - **不改变**你本地的工作区和暂存区
   - 将远程分支更新为 `origin/main`（远程追踪分支）
   - 此时你的本地 main 分支还未更新

2.  **`git merge`**  ：合并阶段
   - 将 `origin/main` 的更改合并到你的本地 main 分支
   - 如果本地没有新的提交，会执行"快进合并"（Fast-forward）
   - 如果本地也有新提交，会创建一个新的"合并提交"（Merge commit）

**图形化理解：**
```
本地状态：      A---B---C (你的提交)
                |
远程状态：      A---B---D---E (同事的提交)

git pull 后：   A---B---C---M (合并提交)
                     \     /
                      D---E
```

**冲突处理：**
如果两个人修改了同一文件的同一部分，`git pull` 会提示冲突。此时需要：
1. 手动编辑冲突文件，解决冲突
2. `git add <冲突文件>` 标记为已解决
3. `git commit` 完成合并

**最佳实践：**
- **推送前先拉取**：`git pull` → 解决冲突 → `git push`
- **开始工作前拉取**：每天开始工作时先 `git pull`，避免积累过多冲突
- **使用 rebase 保持线性历史**：`git pull --rebase` 可以将你的提交"嫁接"到远程最新提交之后，历史更整洁

---

## Git 命令常规操作详解

| 命令         | 核心功能                                              | 详细说明与使用场景                                           |
| ------------ | ----------------------------------------------------- | ------------------------------------------------------------ |
| **add**      | 添加文件内容至暂存区                                  | `git add .` 添加所有修改；`git add -p` 交互式选择部分修改；`git add -u` 只添加已跟踪文件的修改 |
| **bisect**   | 通过二分查找定位引入 bug 的变更                       | `git bisect start` 开始查找；`git bisect good <commit>` 标记正常版本；`git bisect bad <commit>` 标记问题版本；Git 自动跳转到中间提交点供你测试，极大提高调试效率 |
| **branch**   | 列出、创建或删除分支                                  | `git branch` 查看本地分支；`git branch -a` 查看所有分支；`git branch feature` 创建分支；`git branch -d feature` 删除分支（已合并）；`git branch -D feature` 强制删除（未合并） |
| **checkout** | 检出一个分支或路径到工作区                            | `git checkout main` 切换分支；`git checkout -b feature` 创建并切换分支；`git checkout -- <file>` 撤销工作区修改（危险，不可逆） |
| **clone**    | 克隆一个版本库到新目录                                | `git clone <URL>` 默认克隆；`git clone --depth 1` 浅克隆（节省空间）；`git clone --bare` 克隆裸仓库（用于服务器部署） |
| **commit**   | 记录变更到版本库（本地）                              | `git commit -m "message"` 快速提交；`git commit -a` 自动暂存已跟踪文件并提交；`git commit --amend` 修改最后一次提交（可用于补充遗漏文件或修正信息） |
| **diff**     | 显示提交之间、提交和工作区之间等的差异                | `git diff` 查看工作区 vs 暂存区；`git diff --cached` 查看暂存区 vs 最后一次提交；`git diff HEAD` 查看工作区 vs 最后一次提交；`git diff master..feature` 比较两个分支 |
| **fetch**    | 从另外一个版本库下载对象和引用                        | `git fetch origin` 下载所有更新但不合并；`git fetch origin main` 只下载指定分支；与 `pull` 的区别在于 `fetch` 不会改变本地工作区 |
| **grep**     | 输出和模式匹配的行                                    | `git grep "function name"` 在所有提交中搜索字符串；`git grep -n "TODO"` 显示行号；比 `grep -r` 更快，因为它搜索的是 Git 的对象数据库 |
| **init**     | 创建一个空的 Git 版本库或重新初始化一个已存在的版本库 | `git init` 在当前目录创建；`git init <directory>` 在指定目录创建；`git init --bare` 创建裸仓库（服务器用） |
| **log**      | 显示提交日志                                          | `git log --oneline` 单行显示；`git log --graph` 图形化显示分支；`git log --stat` 显示文件统计；`git log -S "keyword"` 搜索代码增减历史 |
| **merge**    | 合并两个或更多开发历史                                | `git merge feature` 将 feature 分支合并到当前分支；`git merge --no-ff feature` 禁用快进合并（保留分支历史）；遇到冲突时需手动解决 |
| **mv**       | 移动或重命名一个文件、目录或符号链接                  | `git mv old.txt new.txt` 等价于 `mv old.txt new.txt` + `git add new.txt` + `git rm old.txt`；Git 会自动跟踪重命名操作 |
| **pull**     | 获取并合并另外的版本库或一个本地分支                  | `git pull origin main` 拉取并合并；`git pull --rebase` 使用变基代替合并（保持线性历史）；`git pull --ff-only` 只接受快进合并（安全模式） |
| **push**     | 更新远程引用和相关的对象                              | `git push origin main` 推送分支；`git push -u origin main` 设置上游分支；`git push --force` 强制推送（危险，会覆盖远程历史）；`git push --tags` 推送所有标签 |
| **rebase**   | 本地提交转移至更新后的上游分支中                      | `git rebase main` 将当前分支变基到 main 分支上；用于整理提交历史，使其线性化；**不要在公共分支上使用 rebase**，因为它会改变提交历史 |
| **reset**    | 重置当前 HEAD 到指定状态                              | `git reset --soft HEAD~1` 撤销最后一次提交，保留修改在暂存区；`git reset --mixed HEAD~1` 撤销提交，修改留在工作区；`git reset --hard HEAD~1` 彻底删除最后一次提交和修改（危险） |
| **rm**       | 从工作区和索引中删除文件                              | `git rm file.txt` 删除文件并暂存；`git rm --cached file.txt` 只从暂存区删除，保留工作区文件（停止跟踪）；`git rm -r directory` 递归删除目录 |
| **show**     | 显示各种类型的对象                                    | `git show HEAD` 显示最后一次提交详情；`git show a1b2c3d` 显示特定提交；`git show HEAD:README.md` 显示提交时的文件内容 |
| **status**   | 显示工作区状态                                        | `git status` 显示详细状态；`git status -s` 或 `git status --short` 简短格式；文件状态包括：未跟踪（Untracked）、已修改（Modified）、已暂存（Staged） |
| **tag**      | 创建、列出、删除或校验一个 GPG 签名的 tag 对象        | `git tag v1.0` 创建轻量标签；`git tag -a v1.0 -m "Release 1.0"` 创建附注标签（推荐，包含完整信息）；`git push --tags` 推送标签到远程 |