---
title: GitHub Desktop
description: 
date: 2025-12-22
categories:
    - 
    - 
---





GitHub Desktop 是 GitHub 官方推出的图形化界面（GUI）工具，它极大地简化了 Git 的操作流程，非常适合不想记忆繁琐 Git 命令行（如 `git add`, `git commit`, `git push`）的用户，或者需要直观查看代码变更差异（Diff）的开发者。

以下是 GitHub Desktop 的详细使用指南，涵盖从安装到日常协作的全流程：

------

### 1. 下载与安装 (Installation)

- **下载：** 访问 [GitHub Desktop 官网](https://desktop.github.com/) 下载适用于 Windows 或 macOS 的安装包。
- **登录：**
  1. 安装完成后打开软件。
  2. 点击 **"Sign in to GitHub.com"**。
  3. 浏览器会弹出授权页面，点击确认授权，软件会自动登录并同步你的账户信息。

------

### 2. 仓库管理 (Repositories)

在使用之前，你需要先通过以下三种方式之一引入项目：

#### A. 克隆现有仓库 (Clone)

如果你已经在 GitHub 上有一个项目，或者想下载别人的开源项目：

1. 点击左上角的 **"File"** -> **"Clone repository..."**。
2. 在弹出的列表中选择你账号下的仓库，或者切换到 **"URL"** 选项卡输入别人的仓库地址。
3. **Local path：** 选择代码保存到本地电脑的路径。
4. 点击 **Clone**。

#### B. 创建新仓库 (Create)

1. 点击 **"File"** -> **"New repository..."**。
2. 填写 Name（项目名）、Description（描述）、Local path（本地路径）。
3. **注意：** 建议勾选 "Initialize this repository with a README" 和根据项目类型选择 **Git Ignore**（例如 Node, Python 等，这会自动忽略不需要上传的垃圾文件）。

#### C. 添加本地文件夹 (Add Local)

如果你电脑上已经有一个代码文件夹：

1. 点击 **"File"** -> **"Add local repository..."**。
2. 选择该文件夹路径。
3. 如果该文件夹还不是 Git 仓库，软件会提示你点击 "create a repository" 进行初始化。

------

### 3. 核心工作流：修改、提交与推送 (The Workflow)

#### 第一步：修改代码

在你的编辑器（如 VS Code）中修改文件并保存。

#### 第二步：查看变更 (Changes)

回到 GitHub Desktop，你会看到左侧栏列出了所有有变动的文件：

- **绿色**代表新增行，**红色**代表删除行。
- 你可以勾选或取消勾选某些文件，决定哪些文件包含在本次提交中。

#### 第三步：提交 (Commit)

在左下角的输入框中：

1. **Summary (必填)：** 简要描述你做了什么（例如：“修复登录页面 Bug”）。
2. **Description (选填)：** 详细描述细节。
3. 点击蓝色的 **Commit to [branch-name]** 按钮。
   - *注意：此时修改只是保存到了“本地仓库”，还没有上传到 GitHub。*

#### 第四步：推送 (Push)

点击顶部工具栏右侧的 **"Push origin"** 按钮。

- 这一步会将你的本地 Commit 上传到 GitHub 服务器，此时别人才能看到你的代码。

------

### 4. 分支管理与协作 (Branching & Collaboration)

#### 切换/新建分支

不要总是在 `main` 或 `master` 主分支上直接修改代码，通常需要新建分支开发功能。

- 点击顶部中间的 **"Current Branch"** 下拉菜单。
- **新建：** 点击 "New Branch"，输入名字（如 `feature/login`），点击 Create。
- **切换：** 在列表中点击已有的分支名，软件会自动切换你本地文件夹里的文件。

#### 拉取最新代码 (Fetch & Pull)

在开始工作前，养成点击顶部 **"Fetch origin"** 的习惯。

- **Fetch：** 只是检查服务器有没有更新。
- **Pull：** 如果按钮变成了 "Pull origin"，说明服务器有更新，点击它将别人的代码下载并合并到你本地。

#### 发起合并请求 (Pull Request)

当你完成了某个分支的开发并 Push 后：

1. GitHub Desktop 会显示 **"Create Pull Request"** 按钮。
2. 点击后会自动打开浏览器进入 GitHub 页面。
3. 你在网页上创建 PR，请求将你的分支合并进主分支。

------

### 5. 实用技巧 (Pro Tips)

- **查看历史 (History)：** 点击左侧栏的 "History" 标签，可以查看每一次提交的记录，以及每次提交具体改了哪些代码。
- **撤销更改 (Discard)：** 如果你写乱了代码想重来，在 "Changes" 列表中右键点击文件，选择 **"Discard changes"**，文件会恢复到上次提交的状态。
- **解决冲突 (Conflicts)：** 如果 Pull 代码时提示冲突，GitHub Desktop 会提示你选择使用“你的版本”还是“服务器的版本”，或者让你打开编辑器手动合并。
- **与编辑器集成：** 点击菜单栏 **"Repository"** -> **"Open in Visual Studio Code"** (或你安装的其他编辑器)，可以快速打开当前项目。

### 总结流程

1. **Fetch origin** (开工前先拉取)
2. **New Branch** (新建分支开发)
3. 写代码 -> 回到软件看 Diff
4. 写 Summary -> **Commit**
5. **Push origin**
6. **Create Pull Request**



GitHub Desktop 除了基础的提交和推送，还隐藏了许多**提高效率**和**挽救失误**的高级功能。

以下是进阶使用技巧：

------

### 1. 精细化提交 (Partial Commits)

场景： 你在一个文件里同时修改了“数据库配置”和“Nginx配置”，但你希望分开两次提交，保持记录清晰。

操作技巧：

- 不要直接勾选整个文件。
- 在右侧的代码变更预览区域，你可以直接点击**每一行代码左侧的行号（Gutter）**。
- **蓝色**表示选中，**白色**表示未选中。
- **效果：** 你可以只提交文件中的第 10-15 行，而把第 20-30 行留给下一次提交。

### 2. "后悔药"功能 (Fixing Mistakes)

Git 命令行撤销操作很繁琐，但 GitHub Desktop 把这些做成了极其简单的按钮：

#### A. 撤销刚才的 Commit (Undo)

场景： 刚点了 Commit 按钮，突然发现忘了把一个文件加进去，或者提交信息写错了。

操作： 提交后几秒内，界面下方会弹出一个通知框，点击蓝色的 "Undo" 按钮。

- **效果：** 刚才的提交会消失，所有修改重新变回“未提交”的状态，你可以补上文件或修改信息后再次提交。

#### B. 修正上一次提交 (Amend)

场景： 已经提交了一会儿了，但发现上一个提交漏了东西。

操作：

1. 在 "Changes" 面板勾选你漏掉的文件。
2. 勾选提交框下方的 **"Amend commit"** 选项。
3. **效果：** 新的修改会直接合并到**上一次**的提交记录中，而不是产生一个新的 Commit 记录（保持历史干净）。
   - *注意：如果你的代码已经 Push 到服务器了，尽量不要用这个功能，否则会产生版本冲突。*

#### C. 回滚历史版本 (Revert)

场景： 昨天提交的代码导致生产环境报错，需要紧急回退。

操作：

1. 点击左侧 **"History"** 标签。
2. 找到那个有问题的 Commit。
3. 右键点击，选择 **"Revert changes in commit"**。
4. **效果：** 软件会自动生成一个新的 Commit，内容是把刚才的修改“反向”做一遍（删除新增的，恢复删除的）。这是最安全的撤销方式。

------

### 3. 像“摘樱桃”一样搬运代码 (Cherry-pick)

场景： 你在 dev 分支修复了一个 Bug，现在需要把这个修复同步到 prod (生产) 分支，但你不想合并整个 dev 分支的代码。

操作技巧（这是 GitHub Desktop 最酷的功能）：

1. 切换到目标分支（例如 `prod`）。
2. 点击 **"History"**。
3. 在顶部的分支对比框中，选择包含修复代码的分支（例如 `dev`）。
4. 找到那个修复 Bug 的 Commit，**直接用鼠标把它拖拽 (Drag & Drop)** 到顶部的当前分支栏上。
5. **效果：** 这个特定的 Commit 就会被复制到当前分支。

------

### 4. 快速忽略文件 (.gitignore)

场景： 你的项目里总是出现 .DS_Store，或者 Docker 生成的日志文件 app.log，你不想每次都手动取消勾选。

操作：

1. 在 "Changes" 列表中，右键点击那个垃圾文件。
2. 选择 **"Ignore file"**（忽略该文件）或者 **"Ignore all .log files"**（忽略所有后缀为 .log 的文件）。
3. **效果：** GitHub Desktop 会自动帮你修改 `.gitignore` 文件，以后这些垃圾文件再也不会出现在变动列表里了。

------

### 5. 自动暂存 (Auto Stashing)

场景： 你正在 feature-A 分支写代码写到一半，突然老板让你去 main 分支修一个紧急 Bug。以前你需要手动 git stash，否则无法切换分支。

操作：

- 直接在 GitHub Desktop 顶部切换分支。
- **效果：** 软件会自动检测到你还有未提交的修改，并询问你：
  - *Leave my changes on [current-branch]:* 把修改留在当前分支（相当于自动 Stash）。
  - *Bring my changes to [new-branch]:* 把修改带到新分支去。
- 当你修完 Bug 切回来时，点击 **"Restore"**，刚才写了一半的代码又回回来了。

------

### 6. 命令行与其他工具集成

虽然用 GUI，但有时候必须用命令行（比如执行 `chmod +x` 或 `docker build`）。

- **快捷键：** `Ctrl + ~` (Windows) 或 `Ctrl + Alt + T` (默认设置不同) 可以直接在当前仓库路径下打开终端（PowerShell, Git Bash, 或 Terminal）。
- **设置：** 在 **File** -> **Options** -> **Integrations** 中，你可以设置默认打开的编辑器（如 VS Code）和默认的终端工具（建议设置为 Git Bash 或 Windows Terminal）。

### 总结

- **精细控制：** 点击行号部分提交。
- **修正错误：** 善用 Amend 和 Revert。
- **跨分支神器：** 鼠标拖拽 Cherry-pick。
- **保持整洁：** 右键加入 Ignore。

