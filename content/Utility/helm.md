---
title: helm
description: 
date: 2025-12-14
categories:
    - 
    - 
---
在 DevOps 和云基础设施的背景下，**Helm** 是 **Kubernetes 的标准包管理器**。

你可以把它想象成 Ubuntu 上的 `apt`、CentOS 上的 `yum` 或 macOS 上的 `Homebrew`——但它是专门用于将应用程序部署到 Kubernetes 集群上的。

以下是关于 Helm 是什么、如何工作以及为何它如此重要的详细中文解析：

------

### 1. 核心概念 (Core Concepts)

Helm 依赖三个主要概念来管理 Kubernetes 应用程序：

- **Chart（图表/包）：** Helm 的包被称为 *Chart*。它包含了在 Kubernetes 集群中运行应用程序、工具或服务所需的所有资源定义（YAML 文件）。
- **Config（配置/值）：** 配置信息与包本身是分离开的。通常使用一个文件（一般是 `values.yaml`）将特定的设置（如镜像版本、服务端口或持久卷大小）注入到 Chart 模板中。
- **Release（发布）：** 当你安装一个 Chart 时，它会创建一个 *Release*。这是该 Chart 在你集群中运行的一个特定实例。你可以多次安装同一个 Chart 来创建不同的 Release（例如 `mysql-dev` 和 `mysql-prod`）。

### 2. 为什么使用 Helm？

手动管理原始的 Kubernetes YAML 清单（如 Deployments, Services, Ingresses, ConfigMaps）会变得非常复杂且容易出错。Helm 通过以下方式解决了这个问题：

- **模板化 (Templating)：** 它允许你在 YAML 文件中使用变量。你无需硬编码副本数量，而是写成 `replicas: {{ .Values.replicaCount }}`，这使得你的清单可以在不同的环境（开发、测试、生产）中重复使用。
- **版本控制 (Version Control)：** Helm 会追踪 Release 的版本历史。如果新的部署失败了，你可以运行 `helm rollback` 命令立即回滚到上一个正常工作的版本。
- **依赖管理 (Dependency Management)：** 如果你的应用程序依赖于数据库，你可以在 Chart 中定义这个依赖关系，Helm 会确保数据库与你的应用一起被安装。

### 3. Chart 的文件结构

如果你查看一个 Helm Chart 的目录内部，通常会看到这样的结构：

| **文件/目录** | **描述**                                                     |
| ------------- | ------------------------------------------------------------ |
| `Chart.yaml`  | 关于 Chart 的元数据（名称、版本、描述）。                    |
| `values.yaml` | Chart 的默认配置值。                                         |
| `templates/`  | 包含实际 K8s 清单模板的目录，这些模板会与配置值结合生成最终文件。 |
| `charts/`     | 包含任何 Chart 依赖项的目录。                                |

### 4. 基本命令速查表

以下是 Helm 工作流中最常用的命令：

**仓库 (Repositories)**

- `helm repo add [名称] [url]` – 添加一个 chart 仓库。
- `helm repo update` – 更新本地可用 chart 的缓存。

**管理 (Management)**

- `helm install [发布名称] [chart]` – 将 chart 安装到集群中。
- `helm upgrade [发布名称] [chart]` – 升级现有的 release。
- `helm rollback [发布名称] [版本号]` – 将 release 回滚到之前的版本。
- `helm uninstall [发布名称]` – 从集群中移除 release。

**检查 (Inspection)**

- `helm list` – 列出当前命名空间下的所有 release。
- `helm show values [chart]` – 查看 chart 的默认配置值。





安装 Helm 通常非常简单，最推荐的方式是使用**包管理器**（Package Manager），这样以后升级也更方便。

以下是针对不同操作系统的安装方法：

### 1\. Windows 用户 (推荐)

由于您正在学习 PowerShell，最便捷的方法是使用 **Winget** (Windows 自带) 或 **Chocolatey**。

打开您的 PowerShell（管理员模式），输入以下任一命令：

  * **使用 Winget (系统自带):**
    ```powershell
    winget install Helm.Helm
    ```
  * **使用 Chocolatey (如果您已安装):**
    ```powershell
    choco install kubernetes-helm
    ```
  * **使用 Scoop:**
    ```powershell
    scoop install helm
    ```

### 2\. macOS 用户

如果您使用的是 Mac，推荐使用 **Homebrew**。

打开终端 (Terminal) 并运行：

```bash
brew install helm
```

### 3\. Linux 用户

Linux 有多种安装方式，最通用的方式是使用官方的安装脚本：

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

如果您使用的是 **Ubuntu** 并且喜欢用 Snap：

```bash
sudo snap install helm --classic
```

-----

### 4\. 验证安装

安装完成后，请关闭并重新打开您的终端/PowerShell，然后输入以下命令来确认 Helm 是否就绪：

```bash
helm version
```

如果看到类似 `version.BuildInfo{Version:"v3.x.x"...}` 的输出，说明安装成功了！

### 重要提示：你需要 kubectl

Helm 是用来指挥 Kubernetes 的，所以 Helm 必须能连接到一个集群。这意味着您本地通常也需要安装 `kubectl` 并配置好集群连接（kubeconfig）。





用 **Redis**（一个非常流行的数据库）来练手。这是最经典的 Helm 入门操作。

在开始之前，请确保您的 **Docker Desktop** 已经启动，并且在设置中启用了 Kubernetes（或者您有其他的 Kubernetes 集群正在运行）。

请在 PowerShell 中依次执行以下步骤：

### 第一步：添加“应用商店”（添加仓库）

Helm 安装好后，默认是空的。我们需要先添加一个仓库（Repository），就像在手机上添加应用商店源一样。我们将使用 **Bitnami** 仓库，因为他们的 Chart 质量非常高。

复制并运行：

```powershell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 第二步：更新缓存

添加仓库后，需要告诉 Helm 获取最新的软件列表（类似刷新网页）。

运行：

```powershell
helm repo update
```

*您应该会看到类似 "Update Complete" 的提示。*

### 第三步：搜索软件（可选）

确认一下 Redis 是否在列表中：

运行：

```powershell
helm search repo redis
```

*您会看到很多结果，找到 `bitnami/redis` 这一行就是我们要用的。*

### 第四步：正式安装！

现在我们来安装 Redis。我们将这个“实例”命名为 `my-redis`。

运行：

```powershell
helm install my-redis bitnami/redis
```

**发生了什么？**

1.  您会看到屏幕上输出**一大段文字**。不要慌，这是正常的！
2.  这段文字包含了如何连接这个数据库的说明（密码在哪里、端口是多少）。
3.  Helm 已经告诉 Kubernetes：“嘿，帮我拉取 Redis 镜像，配置好网络，并把它们跑起来。”

### 第五步：验证安装

让我们看看它是不是真的在运行。

**方法 A：用 Helm 查看**

```powershell
helm list
```

*您应该能看到 `my-redis` 的状态是 `deployed`。*

**方法 B：用 kubectl 查看（看底层的 Pod）**

```powershell
kubectl get pods
```

*您应该会看到类似 `my-redis-master-0` 和 `my-redis-replicas-0` 的东西。状态可能是 `ContainerCreating`（正在下载）或者 `Running`（已运行）。*

### 第六步：清理（卸载）

练手结束后，如果您不想留着它占用电脑资源，可以把它一键卸载。

运行：

```powershell
helm uninstall my-redis
```

*它会提示 `release "my-redis" uninstalled`，这表示所有相关的 Pod、服务和配置都已被清理干净。*





在 Helm 中，修改配置的核心在于理解 **Values（值）** 的概念。

你可以把 Helm Chart 想象成一份**“填空题”**试卷（Template），而配置就是你的**“答案卡”**（Values）。Helm 的工作就是把答案填进试卷里，生成最终给 Kubernetes 的指令。

我们有两种主要方式来修改配置：

### 方法一：快速修改（使用 `--set`）

如果你只想改一两个简单的设置（比如改个密码，或者改个端口），可以直接在命令行里用 `--set` 参数。

我们继续以 Redis 为例。默认情况下，Bitnami 的 Redis 会安装“主从复制”模式（会启动 3 个 Pod，比较占电脑内存）。我们把它改成“单机”模式（standalone），并且设置一个自定义密码。

请运行：

PowerShell

```
helm install my-redis-simple bitnami/redis --set architecture=standalone --set auth.password=MySecretPass123
```

**命令解析：**

- `--set architecture=standalone`: 告诉 Chart 我只要一个节点，不要集群。
- `--set auth.password=...`: 覆盖默认的随机密码。

你可以运行 `kubectl get pods`，你会发现这次只启动了 **1 个** Pod，而不是默认的 3 个。

------

### 方法二：专业修改（使用 YAML 文件）

这是生产环境中最常用的方法。当你要修改几十个参数时，写在命令行里太乱了。我们会把配置写在一个文件里，通常叫 `values.yaml`。

步骤 1：查看所有可配置的选项

首先，我们需要知道这个软件到底有哪些“开关”可以拨动。我们可以把默认配置导出到一个文件里看看。

PowerShell

```
helm show values bitnami/redis > values.yaml
```

步骤 2：编辑这个文件

现在，用记事本或 VS Code 打开当前目录下生成的 values.yaml 文件。

你会看到几百行配置！别担心，你只需要修改你关心的部分，其他的保持默认即可。

比如，找到 `architecture` 那一行，把它从 `replication` 改为 `standalone`。或者找到 `master.persistence.size` 把磁盘大小从 `8Gi` 改为 `1Gi`（节省空间）。

*为了演示方便，您可以创建一个名为 `my-settings.yaml` 的新文件，只写我们要改的内容：*

YAML

```
# my-settings.yaml
architecture: standalone
auth:
  password: "MySuperStrongPassword"
master:
  persistence:
    size: 500Mi
```

步骤 3：应用配置文件

现在告诉 Helm：“安装 Redis，但要使用我文件里的设置。”

PowerShell

```
helm install my-custom-redis bitnami/redis -f my-settings.yaml
```

*(注意：`-f` 是 `--values` 的缩写)*

------

### 进阶：如果我已经安装了，怎么修改？

如果你已经安装了 `my-redis`，突然想修改它的密码或内存限制，不需要卸载重装！

Helm 有一个强大的 **Upgrade** 功能。

假设你修改了 `my-settings.yaml`，想把变更应用到正在运行的 `my-custom-redis` 上：

PowerShell

```
helm upgrade my-custom-redis bitnami/redis -f my-settings.yaml
```

Helm 会自动计算差异，只更新需要变动的部分（比如重启 Pod 以应用新密码）。

### 总结

| **场景**             | **方法**    | **命令示例**                         |
| -------------------- | ----------- | ------------------------------------ |
| **临时/简单修改**    | `--set`     | `helm install ... --set key=value`   |
| **复杂/生产配置**    | `-f 文件名` | `helm install ... -f values.yaml`    |
| **修改已运行的服务** | `upgrade`   | `helm upgrade [名字] [chart] -f ...` |

------

### 既然您已经掌握了安装和配置...

您想尝试自己 **编写一个最简单的 Helm Chart** 吗？我们可以写一个简单的网页服务器，体验一下如何自己制作那个“填空题”模板。



太好了！动手“造轮子”是学习最快的方法。

我们将创建一个名为 `mychart` 的 Chart。默认情况下，Helm 会帮我们生成一个 **Nginx**（网页服务器）的模板。

请在 PowerShell 中跟着我一步步操作：

### 第一步：自动生成骨架

Helm 提供了一个命令，可以一键生成标准的目录结构，不需要手动新建文件夹。

运行：

```powershell
helm create mychart
```

现在，查看一下生成了什么（进入目录）：

```powershell
cd mychart
ls
```

*你会看到 `Chart.yaml`、`values.yaml` 和 `templates` 文件夹。这就是一个标准的 Helm 包结构。*

-----

### 第二步：理解“填空题”原理

我们需要看看 Helm 是如何把“配置”填进“模板”里的。

**1. 看看答案卡 (`values.yaml`)**
打开 `values.yaml` (用记事本或 `code .` 如果你有 VS Code)。
找到第一行：

```yaml
replicaCount: 1
```

*这表示默认只启动 1 个副本。*

**2. 看看试卷 (`templates/deployment.yaml`)**
打开 `templates` 文件夹里的 `deployment.yaml`。
找到 `replicas:` 这一行，你会看到类似这样的代码：

```yaml
replicas: {{ .Values.replicaCount }}
```

**这就是魔法所在！** 也就是告诉 Helm：“去 `values.yaml` 文件里找 `replicaCount` 的值，填在这里。”

-----

### 第三步：修改配置

既然是我们可以自己控制的 Chart，我们来改点东西。

请打开 `mychart/values.yaml` 文件：

1.  把 `replicaCount: 1` 改为 `replicaCount: 2` (我们要启动两个实例)。
2.  (可选) 找到 `service:` 部分，把 `type: ClusterIP` 改为 `type: NodePort` (这样更容易从外部访问，但这步不做也没关系)。
3.  **保存文件**。

-----

### 第四步：安装你的本地 Chart

以前我们安装是用 `bitnami/redis`（从云端下载），现在我们要安装刚才创建在**本地硬盘**上的文件夹。

请确保你退回到了 `mychart` 的**上一级目录**（或者在命令里指定路径）：

```powershell
cd ..
helm install my-website ./mychart
```

*注意：这里用的是 `./mychart`，表示“安装当前目录下的 mychart 文件夹”。*

### 第五步：见证奇迹

让我们看看它是否跑起来了，而且是不是真的是 2 个副本。

运行：

```powershell
kubectl get pods
```

你应该会看到类似这样的输出（有两行）：

```text
NAME                          READY   STATUS    RESTARTS   AGE
my-website-nginx-xxxx-yyyy    1/1     Running   0          30s
my-website-nginx-xxxx-zzzz    1/1     Running   0          30s
```

### 第六步：访问你的网页

既然它是一个网页服务器，我们得亲眼看到那个网页。我们可以使用 `port-forward` 把它的端口映射到你电脑上。

运行：

```powershell
kubectl port-forward svc/my-website-mychart 8080:80
```

*(注意：如果这步报错，可能是服务名称不同。你可以先用 `kubectl get svc` 查一下确切的服务名)*

现在，打开你的浏览器（Chrome/Edge），访问：
**http://localhost:8080**

你应该会看到 **"Welcome to nginx\!"** 的页面。这就是你刚刚用 Helm 部署的应用！

-----

### 恭喜！

你已经完成了一个 Helm 工程师 80% 的日常工作：

1.  `helm create` (创建)
2.  修改 `values.yaml` (配置)
3.  `helm install` (部署)

**现在的你可能有一个疑问：**
目前的模板里都是简单的“填空”。如果我想根据情况做判断呢？
比如：**“如果是生产环境(Prod)，就开启高可用；如果是测试环境(Dev)，就关闭。”**

这需要用到 Helm 的**逻辑控制（If/Else）**。想学这一招进阶技巧吗？



这正是 Helm 模板语言（基于 Go 模板）最强大的地方。通过简单的 `if/else` 逻辑，你可以让同一个 Chart 在开发环境和生产环境中表现出完全不同的行为。

我们来做一个有趣的实验：**“消失的配置文件”**。
我们将创建一个 ConfigMap（配置文件），但设置一个开关：**只有当开关打开时，这个文件才会被创建。**

请继续在您的 `mychart` 目录中操作。

### 第一步：设置开关（修改 values.yaml）

首先，我们需要在“答案卡”里加一个选项。

打开 `mychart/values.yaml`，在文件的**最后一行**添加以下内容：

```yaml
enableSpecialFeature: true
```

*这表示我们默认开启这个“特殊功能”。*

-----

### 第二步：编写带逻辑的模板

我们不修改现有的文件，而是新建一个文件来专门测试逻辑，这样更清晰。

在 `mychart/templates/` 目录下，新建一个文件，命名为 `my-config.yaml`。
将以下代码复制进去：

```yaml
{{- if .Values.enableSpecialFeature }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config-map
data:
  message: "Congratulations! You found the secret feature."
{{- end }}
```

**逻辑解析：**

  * `{{- if .Values.enableSpecialFeature }}`：这是判断语句的开始。意思是：“如果 `values.yaml` 里的 `enableSpecialFeature` 是 `true`，那么...”
  * **中间的 YAML 内容**：这是正常要生成的资源内容。
  * `{{- end }}`：这是判断语句的结束。

**注意那个减号 `-`**：`{{-` 表示“删除这个标签产生的空行/空格”。在 YAML 中空格非常敏感，加上这个是个好习惯，可以让生成的代码更整洁。

-----

### 第三步：应用更改

现在我们来更新我们之前安装的 `my-website`。

运行：

```powershell
helm upgrade my-website ./mychart
```

*提示：一定要在 `mychart` 文件夹的**上一级**运行这个命令。*

### 第四步：验证是否存在

看看这个 ConfigMap 是否被成功创建了。

运行：

```powershell
kubectl get configmap
```

你应该能看到一个名为 `special-config-map` 的项目。这说明 `if` 判断通过了！

-----

### 第五步：见证“消失术” (Else/False)

现在，假设我们要部署到另一个环境，不需要这个功能。我们不需要改代码，只需要改配置。

我们使用 `--set` 临时覆盖配置，把开关关掉：

```powershell
helm upgrade my-website ./mychart --set enableSpecialFeature=false
```

现在再次检查：

```powershell
kubectl get configmap
```

你会发现 `special-config-map` **不见了**！
Helm 检测到条件为 `false`，所以直接跳过了那段 YAML，Kubernetes 也就把这个不再需要的资源删除了。

-----

### 进阶：If / Else 组合拳

除了决定“存不存在”，你还可以用来决定“具体的值”。

比如，你想实现：

  * 如果是生产环境 (`production`)，副本数设为 5。
  * 否则（测试环境），副本数设为 1。

你可以这样写（**仅供阅读理解，无需操作**）：

```yaml
replicas: {{ if eq .Values.env "production" }} 5 {{ else }} 1 {{ end }}
```

  * `eq` 是 "equal"（等于）的缩写。
  * 这就实现了根据环境自动调整参数。

### 总结

您现在已经掌握了 Helm 的三大法宝：

1.  **模板 (Templates)**：把变量挖空。
2.  **值 (Values)**：填入具体的配置。
3.  **逻辑 (Logic)**：用 `if` 控制资源的创建和内容的生成。

这基本上就是编写 Helm Chart 的核心了！



**打包分享**、**实战救灾（回滚）以及原理解析**。

请继续在 PowerShell/终端中操作。

-----

### 第一部分：如何打包 (Packaging)

您做好了 `mychart`，现在想把它像一个压缩包一样发给同事，或者上传到私有仓库。

**1. 确定版本号**
打开 `mychart/Chart.yaml`。
确保 `version: 0.1.0` 这一行是您想要的版本。

**2. 执行打包命令**
在 `mychart` 文件夹的**上一级目录**运行：

```powershell
helm package ./mychart
```

**3. 结果**
您会发现当前目录下多了一个文件：`mychart-0.1.0.tgz`。
这就是您的“成品”！

  * 您可以把它发给同事。
  * 同事拿到后，不需要解压，直接运行 `helm install my-app ./mychart-0.1.0.tgz` 即可安装。

-----

### 第二部分：实战回滚 (Rollback) —— 您的“后悔药”

为了演示回滚，我们需要先人为制造一次“由于手抖导致的错误更新”。

**1. 制造“事故现场” (创建一个新版本)**
假设我们脑子一热，把副本数改成了 10 个，这太浪费资源了。
运行：

```powershell
helm upgrade my-website ./mychart --set replicaCount=10
```

此时，Helm 会更新应用，您会看到 Pod 数量开始疯狂增加。

**2. 查看历史记录 (关键步骤)**
在回滚之前，您必须知道“回滚到哪里去”。
运行：

```powershell
helm history my-website
```

您会看到类似这样的表格：

```text
REVISION    UPDATED      STATUS      CHART           DESCRIPTION
1           ...          SUPERSEDED  mychart-0.1.0   Install complete
2           ...          SUPERSEDED  mychart-0.1.0   Upgrade complete (之前做的开关测试)
3           ...          DEPLOYED    mychart-0.1.0   Upgrade complete (刚才手抖搞的10副本)
```

*注意：`REVISION` (版本号) 是递增的。当前我们在版本 3。*

**3. 一键回滚 (救命稻草)**
我们要回到版本 2（也就是副本数正常、或者开关测试那个状态）。
运行：

```powershell
helm rollback my-website 2
```

**4. 验证结果**
Helm 会提示 `Rollback was a success`。
再次查看历史：

```powershell
helm history my-website
```

您会发现多了一个 **版本 4**，描述是 `Rollback to 2`。
*注意：Helm 的回滚不是“删除版本3”，而是“根据版本2的配置，生成一个新的版本4”。历史总是向前的。*

检查 Pod 数量，应该又变回正常了。

-----

### 第三部分：Helm 回滚机制是如何工作的？ (Under the Hood)

您可能会好奇：*Helm 怎么知道版本 2 长什么样？它把旧文件存在哪里了？*

**1. 秘密就在 Kubernetes 的 Secrets 里**
Helm 并没有在您的本地电脑存数据库，而是把每一次发布的**所有信息**（包括当时的 Chart 模板、Config 值）都压缩加密后，存到了 Kubernetes 集群的 **Secrets**（保密字典）里。

**2. 亲眼看看这些存档**
您可以运行以下命令，看看 Helm 在集群里藏了什么：

```powershell
kubectl get secrets
```

您会看到类似这样的列表：

```text
sh.helm.release.v1.my-website.v1
sh.helm.release.v1.my-website.v2
sh.helm.release.v1.my-website.v3
sh.helm.release.v1.my-website.v4
```

**每一个版本对应一个 Secret 文件！**

**3. 它是如何还原的？（三方比对）**
当您执行 `helm rollback my-website 2` 时，Helm 做了以下事情：

1.  **读取存档**：从集群下载 `sh.helm.release.v1.my-website.v2` 这个 Secret。
2.  **解压状态**：获取当时所有的 YAML 定义。
3.  **计算差异 (Diff)**：将“版本 2”的配置与“当前集群正在运行”的状态进行比对。
4.  **应用补丁**：计算出需要增加、删除或修改哪些资源，才能让集群变回版本 2 的样子，然后执行 `kubectl apply`。

-----

### 总结

  * **打包 (`package`)**：生成 `.tgz` 文件，方便分发。
  * **历史 (`history`)**：查看发布的时光轴。
  * **回滚 (`rollback`)**：基于历史版本号，生成新的修正版本。
  * **原理**：Helm 把每次发布的快照都存在了 K8s 的 Secret 里，像游戏的“存档点”一样，随时读取。

至此，您已经掌握了从**安装、配置、开发、逻辑编写**到**运维回滚**的 Helm 完整核心链路！您可以自豪地说您会用 Helm 了！