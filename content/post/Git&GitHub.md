---
title: Git & GitHub 
description: 入门操作教程
categories:
    - Test
    - Git
    - GitHub
---

## Git的 "四个空间"模型
![](https://telegraph-image-5tj.pages.dev/file/AgACAgUAAyEGAAS6dLpwAAMJaPRHS_HqpI_X4jvit3oZ3IVUxncAAkoMaxspIaFXCCGXH9obv0QBAAMCAAN5AAM2BA.png)

### 工作区 (Working Directory)

它就是你在电脑上能直接看到的项目文件夹，包含着所有项目文件。你用你的代码编辑器（比如 VS Code）打开项目，进行添加、删除、修改代码等所有操作，都是在工作区里完成的。它是最直观、最原始的空间。

### 暂存区 (Staging Area)

工作区的文件修改后，你不能直接把它“存档”。你需要一个中间步骤，让你选择哪些修改要被打包存档。暂存区就是这个地方。

为什么需要它？ 假设你同时修改了3个文件，但只有2个是关于“修复登录bug”的，另1个是改着玩的。你就可以只把那2个文件“加入购物车”（git add），然后“结账”（git commit）。暂存区给了你一个精挑细选和反悔的机会，确保每一次提交都是一个逻辑完整的单元。

### 本地仓库(Local Repository)

当你执行“结账”命令（git commit）后，暂存区（购物车）里所有的内容会被打包成一个版本（我们称之为一次“提交”），然后被永久地保存在你本地电脑的仓库里。

这个本地仓库位于你项目的 .git 隐藏文件夹中。它就像一个精密的数据库，记录了你项目从创建之初到现在的每一次提交，包括谁、在什么时间、修改了什么内容。这是你个人的、完整的项目历史备份。

### 远程仓库(Remote Repository)

个人开发时，有前三个空间就足够了。但要团队协作，就需要一个所有成员都能访问的中央服务器来同步代码。这个服务器就是远程仓库。

它通常托管在像 GitHub、GitLab 或 Gitee 这样的代码托管平台上。团队成员可以：

 ·从远程仓库克隆 (clone) 一份完整的项目到自己的本地仓库。

 ·将自己本地仓库中新的提交推送 (push) 到远程仓库，分享给他人。
 ·从远程仓库拉取 (pull) 别人的提交，更新自己的本地代码。

 在工作区写代码 -> add到暂存区 -> commit到本地仓库 -> push到远程仓库。

## 准备Git环境

安装 Git、配置 Git、注册 GitHub 账号

### 安装Git

Git是一个命令行工具，首先需要将它安装到你的电脑上。

 · Windows:前往官方官网下载安装包，选择好你要存放的地方，注意：选择 `“Override...”` 并将分支名设为 `main`，别的一路点击`"Next"`即可。在开始菜单里找到`"Git Bash"`并打开
 
 · masOS:打开“终端 (Terminal)” 应用，输入 `git --version`。如果系统提示你安装命令行开发者工具，请同意安装，macOS 会自动为你装好 Git。如果你的系统已经安装了 Xcode，那么 Git 也已经自带了。

 · Linux: 打开终端，根据你的发行版，运行相应的命令：

 `Debian/Ubuntu: sudo apt-get install git`

 `Fedora/CentOS: sudo yum install git`


安装完成后，打开你的终端（Windows 用户请使用 Git Bash），输入以下命令来验证是否安装成功：

`git --version`

1、如果终端打印出类似 git version 2.30.0 这样的版本号信息，恭喜你，Git 已经成功安装！

2、首次配置：为你的代码署名
安装好 Git 后，还有一件至关重要的事情要做：告诉 Git 你是谁。

请在终端中运行以下两条命令，将引号中的内容替换成`你自己的用户名和邮箱`。

设置你的全局用户名（提交时会显示这个名字）

`git config --global user.name "Your Name"`

设置你的全局邮箱(GitHub会用它来关联你的提交和账号)
{{< highlight html >}}
git config --global user.email "your.email@example.com"
{{< /highlight >}}
 提示：`--global` 参数表示这台电脑上所有的Git仓库都会使用这个配置。你只需要设置一次就够了。

3、注册GitHub
 · Git是工具：一个在你电脑上运行的命令行软件
 · GitHub是平台： 一个网站，提供了远程仓库的托管服务，并围绕代码构建了一个庞大的开发社区


## 本地管理你的代码

Git最核心的三个命令：

 ·`git init`:初始化一个新仓库。
 
 ·`git add`:将文件变更添加到暂存区。

 ·`git commit`:将暂存区的变更永久保存到版本库。

### 创建你的项目

创建一个新文件夹，这个可以放在你喜欢放的位置。

 `mkdir my-first-git-project`

进入这个文件夹

 `cd my-first-git-project`

 现在，这个文件夹就是我们的工作区 (Working Directory)。它是我们编写代码、创建文件、进行所有实际工作的地方。但目前，它还只是一个普通的文件夹，Git 还不知道它的存在。

初始化仓库 (git init)：开启版本控制

 为了让 Git 开始管理这个文件夹，我们需要在这个文件夹内部运行一个命令，来“激活”它。

 `git init`

 当你运行这个命令后，你会看到类似这样的输出： `Initialized empty Git repository in /path/to/your/my-first-git-project/.git/`

 这个命令做了什么？它在你的项目文件夹里创建了一个名为 .git 的隐藏子文件夹。这个 .git 文件夹就是你的本地仓库 (Local Repository)。它像一个神秘的保险箱，存放着你项目的所有版本历史、配置信息等

 警告：除非你非常清楚自己在做什么，否则永远不要手动修改 .git 文件夹里的任何内容！把它当成 Git 的专属地盘。

从现在起，my-first-git-project 文件夹就不再普通了，它已经是一个 Git 仓库了。

### 创建文件并检查状态(`git status`)

让我们在工作区里创建第一个文件。一个项目通常会有一个 README.md 文件，用来描述项目信息。

创建并写入一句话到 README.md 文件中

`echo "Hello, Git! This is my first project." > README.md`

现在，我们的工作区有新内容了。Git 是否察觉到了呢？让我们来问问 Git 当前的状态。git status 是你在使用 Git 过程中最常用、也最有用的命令，它会告诉你仓库当前的状态。

`git status`

终端会返回一些信息，其中最关键的是：

On branch master

No commits yet

Untracked files:

(use "git add <file>..." to include in what will be committed) README.md

nothing added to commit but untracked files present (use "git add" to track)

解读一下：

 ·Untracked files (未跟踪的文件)：Git 发现了一个新面孔 README.md，但它并不在版本管理的范围内。
 
 ·use “git add …”: Git 很贴心地提示我们，可以使用 git add 命令来开始跟踪这个文件。

###添加暂存区(`git add`):准备提交

现在，我们要把 README.md 文件从工作区的“待定”状态，转移到暂存区 (Staging Area)，表示“我确定要把这个文件的当前版本加入到下一次存档中”。

`git add README.md`

这个命令没有任何输出，是正常的。想知道发生了什么？再次使用我们的好朋友 git status：

`git status`

输出变了：
 On branch master

 No commits yet

 Changes to be committed:

 (use "git rm --cached <file>..." to unstage) new file: README.md

解读一下：

 ·Changes to be committed (待提交的变更)：README.md 已经进入了暂存区，整装待发，准备被正式存档。
 
 ·暂存区就像一个购物车，你可以不断地往里添加（git add）或移除商品，直到你确认所有要买的东西都在里面了，才去结账（git commit）。

### 提交到仓库(git commit):完成存档

git commit 命令会将暂存区里所有的内容打包，创建一个新的版本记录（一个快照），并永久地保存在你的本地仓库里。

每一次提交，都必须附带一条提交信息 (commit message)，用 -m 参数来指定。这条信息是对本次修改的简短描述，例如 “修复了xx bug” 或 “增加了用户登录功能”。良好的提交信息是团队协作的基石！

`git commit -m "Initial commit: Add README file"`

终端会显示类似如下的输出，告诉你这次提交的概要：

[master (root-commit) a1b2c3d] Initial commit: Add README file 1 file changed, 1 insertion(+) create mode 100644 README.md

恭喜你！你已经成功完成了第一次提交！

现在，如果我们再运行 git status，会得到：

 On branch master

 nothing to commit, working tree clean

### 总结完成了一个完整的本地 Git 操作循环：

 在工作区修改代码（我们创建了 README.md）。

 使用 git add 将修改添加到暂存区（挑选要存档的内容）。

 使用 git commit 将暂存区内容提交到本地仓库（正式存档）。

这个 “修改 -> add -> commit” 的循环，是你未来90%的本地 Git 操作，请务必牢记。

## 版本历史的查看与回溯

1. 查看提交历史(`git log`)

 你的 .git 仓库就像一本详细的日记，记录了每一次 commit 的所有细节。要阅读这本日记，我们使用 git log 命令。

 `git log`

 运行后，你会看到类似这样的输出，从最新到最旧排列：

 commit a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 (HEAD -> master)

 Author: Your Name <your.email@example.com>

 Date: Mon Oct 26 20:21:42 2023 +0800 Initial commit: Add README file

每一条记录都包含了：

 ·Commit ID (哈希值)：那一长串唯一的 a1b2c3d... 字符串，是每个版本的身份证号。

 ·Author (作者)：你之前配置的用户名和邮箱。
 
 ·Date (日期)：提交发生的时间。
 
 ·Commit Message (提交信息)：你在 -m 后面写的那段描述。

 小技巧：如果觉得信息太多，可以试试 `git log --oneline`，它会用更简洁的单行格式显示历史，非常清晰！

2. 版本回溯：回到过去

想象一下，你刚写了一堆代码，结果发现把事情搞砸了，想回到某个正常工作的版本。Git 的时光机能帮你实现！

场景：我只想恢复某一个文件

这是最安全、最常用的操作。假设你不小心把 README.md 文件改乱了，想把它恢复到上一个提交时的样子。

首先，用 `git log --oneline` 找到你想回到的那个版本的 Commit ID（比如 a1b2c3d）。

然后运行：

格式: `git checkout <commit_id> – <file_path>`

`git checkout a1b2c3d -- README.md`

瞬间，你的 README.md 文件就从“灾难现场”恢复到了 a1b2c3d 这个版本时的状态。这种方式只会影响指定的文件，项目中的其他文件安然无恙。

场景：我要让整个项目回到过去（危险操作！）

如果你想让整个项目（所有文件）都回滚到某个旧版本，可以使用 `git reset`。
 ⚠️ 警告：这是一个有破坏性的操作！ 它会 永久丢弃 你在那个旧版本之后的所有修改和提交。请三思而后行。

 格式: git reset --hard <commit_id>

 `git reset --hard a1b2c3d`

 执行后，你的整个工作区都会被重置到 a1b2c3d 这个版本的状态。之后的所有更改都仿佛从未发生过。

## 多人模式”：与 GitHub 协作

到目前为止，我们所有的操作都在本地电脑上。代码的备份和分享都无法实现。现在，是时候把我们的本地仓库连接到 GitHub 这个“云端档案馆”，开启真正的协作之旅了。

与远程仓库交互，通常有两种起点：

 从无到有：你先在本地创建了一个项目，然后想把它推送到 GitHub 上分享和备份。

 从有到有：项目已经存在于 GitHub 上（例如公司的项目或开源项目），你需要把它复制到本地开始工作。

我们分别来看这两种情况。

### 推送本地新项目到 GitHub

这正是我们当前的状况。我们已经在本地创建了 my-first-git-project 并有了一次提交。现在把它推上去。

1. 在GitHub 上创建空的远程仓库

首先，你需要为本地项目在 GitHub 上准备一个“家”。

 登录你的 GitHub 账号。

 点击右上角的 + 号，选择 New repository。

 Repository name (仓库名)：建议和你的本地文件夹名保持一致，例如 my-first-git-project。

 Description (描述): 简单描述一下你的项目。

 保持 Public (公开) 状态，任何人都可以看到你的代码。

 重要：不要 勾选 “Add a README file”, “Add .gitignore”, “Choose a license”。因为我们本地已经有项目了，需要创建一个完全空的仓库来接收我们的代码。

 点击 Create repository。

2. 连接本地与远程 (`git remote add`)

创建成功后，GitHub 会给你一个仓库地址，通常是 https://github.com/YourUsername/my-first-git-project.git。

回到你的终端，在本地项目文件夹里，运行以下命令，告诉本地 Git 这个远程仓库的存在：

格式: git remote add <远程仓库别名> <远程仓库URL>

 `git remote add origin https://github.com/YourUsername/my-first-git-project.git`

 · origin 是我们给这个远程仓库起的一个别名，这是 Git 的一个通用惯例，代表“主要的、默认的远程仓库”。

3. 推送你的代码(`git push`)

连接已经建立，现在我们可以把本地的代码历史“推送”到 origin（也就是 GitHub）上了。
 
 `git push -u origin main`

 · `push`：是“推送”的动作。
 
 · `origin`: 指定要推送到哪个远程仓库。

 · `main`:指定要推送本地的哪个分支。（现在你可以就理解为我们要推送的就是 main 这条主线上的所有代码）。

 · `-u`：这个参数很重要，它会把本地的 main 分支和远程的 main 分支关联起来。这样，未来你再推送时，只需简单地输入 `git push` 即可。

 第一次推送时，系统可能会提示你输入 GitHub 的用户名和密码（或者 Personal Access Token）来完成认证。

推送成功后，刷新你的 GitHub 仓库页面，你会惊喜地发现，你本地的 README.md 文件和提交历史已经完完整整地出现在了网页上！

### 场景二：克隆一个已存在的项目 (`git clone`)

这是更常见的协作方式。比如你要加入一个新团队，第一件事通常就是从 GitHub 把项目代码克隆到你的电脑上。

`git clone` 命令帮你一步到位。

 打开你想克隆的项目的 GitHub 页面。

 点击绿色的 <> Code 按钮，复制 HTTPS 地址。
 
 打开你的终端，cd 到你想存放这个项目的目录下（比如 Desktop 或 Documents）。
 
 运行 git clone 命令：

格式: git clone <远程仓库URL>

`git clone https://github.com/some-user/some-awesome-project.git`

这个命令会做几件美妙的事情：

 在当前目录下创建一个名为 `some-awesome-project` 的新文件夹。

 将远程仓库的所有文件和完整的版本历史都下载到这个文件夹里。

 自动帮你设置好远程仓库的连接，并命名为 `origin`。你无需再手动 `git remote add`。

克隆完成后，`cd some-awesome-project` 进入文件夹，你就可以开始工作了。

后续操作：保持同步 (`git pull`)

无论你是通过 push 还是 clone 开始的远程协作，当你的同事（或者未来的你在另一台电脑上）更新了 GitHub 上的代码后，你就需要将这些最新的变更同步到你的本地仓库。这时使用`git pull`：

`git pull origin main`

这个命令会从远程仓库 `origin` 的 `main` 分支上，拉取最新的代码变更，并自动与你本地的代码合并。养成在开始一天的工作前先 `git pull` 的好习惯，可以有效避免很多冲突。


## Git命令常规操作
{{< highlight html >}}
add :添加文件内容至暂存区（索引）
bisect:通过二分查找定位引入bug的变更
branch:列出、创建或删除分支
checkout:检出一个分支或路径到工作区
clone：克隆一个版本库到一个新目录
commit:记录变更到版本库（本地）
diff:显示提交之间、提交和工作区之间等的差异
fetch:从另外一个版本库下载对象和引用
grep:输出和模式匹配的行
init:创建一个空的
Git:版本库或重新初始化一个已存在的版本库
log:显示提交日志
merge：合并两个或更多开发历史
mv:移动或重命名一个文件、目录或符号链接
pull:获取并合并另外的版本库或一个本地分支
push：更新远程引用和相关的对象
rebase：本地提交转移至更新后的上游分支中
reset:重置当前HEAD到指定状态
rm：从工作区和索引中删除文件
show：显示各种类型的对象
status：显示工作区状态
tag:创建、列出、删除或校验一个GPG签名的tag对象
{{< /highlight >}}



















