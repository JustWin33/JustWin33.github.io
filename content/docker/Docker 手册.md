---
title: docker
description: 命令用法
date: 2025-11-07
categories:
    - 
    - 
---



# Docker 手册

## 1. 核心概念速览

### 1.1 镜像与容器

在 Docker 的生态系统中，**镜像（Image）** 和 **容器（Container）** 是两个最核心且基础的概念，它们之间的关系类似于面向对象编程中的类（Class）与实例（Instance）。**镜像是静态的、只读的模板**，它包含了运行一个应用程序所需的所有内容，包括代码、运行时环境、系统工具、系统库以及设置。可以将镜像理解为一个轻量级的、可执行的软件包，它定义了应用程序及其所有依赖项。镜像是通过**分层（Layered）** 文件系统构建的，每一层都代表了镜像构建过程中的一次修改，这种分层结构使得镜像的存储和分发非常高效。当对镜像进行修改时，例如更新一个软件包，Docker 会创建一个新的层，而不是替换整个镜像，这大大减少了存储空间的占用和网络传输的数据量。镜像本身是不可变的，一旦创建，其内容就不会改变，这保证了应用在任何环境中的一致性。

**容器则是镜像的动态运行实例**。当使用 `docker run` 命令启动一个镜像时，Docker 引擎会在镜像的只读层之上添加一个**可写的容器层**，从而创建一个独立的、隔离的运行环境。这个可写的容器层允许应用程序在运行时进行写操作，例如创建文件、修改配置等，但这些修改仅存在于该容器层中，并不会影响到底层的只读镜像。因此，一个镜像可以被用来创建多个相互隔离的容器，每个容器都有自己的文件系统、进程空间和网络接口。容器的生命周期是短暂的，当容器被删除时，其可写层也会被一并删除，除非使用了数据卷（Volume）或绑定挂载（Bind Mount）等持久化机制。这种设计使得容器非常适合用于微服务架构，每个服务都可以被打包成一个独立的镜像，并以容器的形式运行，实现了应用间的解耦和快速部署。理解镜像与容器的区别与联系，是掌握 Docker 技术的第一步，也是后续学习容器编排、服务发现和持续集成等高级主题的基础。

### 1.2 数据卷与持久化

Docker 容器的设计理念之一是**无状态（Stateless）** ，即容器的文件系统是临时的，当容器被删除时，其内部的所有数据都会丢失。然而，在实际应用中，许多服务（如数据库、文件存储服务等）都需要持久化存储数据。为了解决这个问题，Docker 提供了多种数据持久化机制，其中最核心的是**数据卷（Volume）** 和**绑定挂载（Bind Mount）** 。数据卷是由 Docker 引擎管理的、独立于容器生命周期的存储区域。当创建一个数据卷时，Docker 会将其存储在宿主机的一个特定目录下（通常是 `/var/lib/docker/volumes/`），并且可以通过卷名来引用它。数据卷可以被一个或多个容器同时挂载和共享，实现了容器间的数据共享和重用。由于数据卷的生命周期独立于容器，即使所有引用它的容器都被删除，数据卷中的数据依然会保留下来，直到用户显式地删除该数据卷。这使得数据卷非常适合用于生产环境，特别是需要持久化存储数据库数据、应用日志或用户上传文件的场景。

与数据卷不同，**绑定挂载（Bind Mount）** 是将宿主机文件系统上的一个特定目录或文件直接映射到容器内部。这种方式提供了更高的灵活性，因为开发者可以直接在宿主机上访问和修改容器内的文件，非常适合用于开发环境。例如，开发者可以将本地的代码目录绑定挂载到运行应用的容器中，这样每次修改代码后，无需重新构建镜像，容器内的应用就能立即生效，极大地提高了开发效率。然而，绑定挂载的灵活性也带来了一些潜在的安全风险，因为它打破了容器与宿主机之间的文件系统隔离。此外，Docker 还提供了一种名为 **`tmpfs`** 的挂载方式，它将数据存储在宿主机的内存中，而不是磁盘上。这种方式适用于存储临时数据，如缓存文件或会话信息，因为它能提供极高的读写性能，并且在容器停止后数据会自动清除，不会占用磁盘空间。选择哪种持久化方式取决于具体的应用场景：数据卷适用于生产环境的数据持久化和共享，绑定挂载适用于开发环境的代码实时同步，而 `tmpfs` 则适用于对性能要求极高的临时数据存储。

### 1.3 网络与通信

Docker 的网络功能为容器化应用提供了强大的通信能力，使得容器之间以及容器与外部世界之间可以进行灵活、高效的数据交换。默认情况下，Docker 会创建一个名为 **`bridge`** 的虚拟网桥，所有新创建的容器都会自动连接到这个网络。在同一个 `bridge` 网络中，容器可以通过各自的 IP 地址进行通信。然而，由于容器的 IP 地址在每次重启时可能会发生变化，这种方式在实际应用中并不方便。为了解决这个问题，Docker 支持自定义网络，用户可以通过 `docker network create` 命令创建一个自定义的 `bridge` 网络。当容器连接到同一个自定义网络时，它们不仅可以通过 IP 地址通信，更重要的是可以**通过容器名称进行通信**，因为 Docker 内置的 DNS 服务会自动将容器名称解析为其在自定义网络中的 IP 地址。这种基于容器名称的通信方式极大地简化了服务间的调用，特别是在使用 Docker Compose 等编排工具时，服务之间可以直接通过服务名进行访问，而无需关心其具体的 IP 地址。

除了默认的 `bridge` 模式，Docker 还支持多种网络模式以满足不同的需求。**`host` 模式**允许容器共享宿主机的网络栈，这意味着容器将使用宿主机的 IP 地址和端口，从而避免了网络地址转换（NAT）带来的性能损耗，适用于对网络性能要求极高的场景。然而，`host` 模式也牺牲了容器的网络隔离性，可能会带来安全风险。**`none` 模式**则为容器提供了一个完全隔离的网络环境，容器内部没有任何网络设备，需要用户手动配置网络，适用于一些特殊的、需要完全控制网络栈的场景。对于跨主机部署的容器集群，Docker 提供了 **`overlay` 网络模式**，它允许不同宿主机上的容器像在同一个局域网中一样进行通信，是实现 Docker Swarm 集群服务发现和负载均衡的基础。通过灵活配置这些网络模式和管理命令，开发者可以构建出复杂、可靠且高性能的容器化应用网络架构。

## 2. 镜像管理命令

### 2.1 获取与查看镜像

#### 2.1.1 拉取镜像：`docker pull`

`docker pull` 命令是获取 Docker 镜像最基本也是最常用的方式。它从配置的镜像仓库（默认为 Docker Hub）中下载指定的镜像到本地。命令的基本语法是 `docker pull [OPTIONS] NAME[:TAG|@DIGEST]`。其中，`NAME` 是镜像的名称，例如 `nginx` 或 `ubuntu`。`TAG` 是镜像的标签，用于标识镜像的不同版本，例如 `1.21.5` 或 `latest`。如果不指定 `TAG`，Docker 会默认拉取 `latest` 标签的镜像。`@DIGEST` 则是通过镜像的摘要（一个唯一的哈希值）来精确指定要拉取的镜像版本，这可以确保每次拉取的都是完全相同的镜像，避免了因标签更新而带来的不确定性。例如，要拉取特定版本的 Nginx 镜像，可以执行 `docker pull nginx:1.21.5`。这个命令会从 Docker Hub 下载 `nginx` 镜像的 `1.21.5` 版本。在拉取过程中，Docker 会显示下载进度，包括每一层的下载状态。由于镜像是分层的，如果本地已经存在某些层，Docker 会跳过这些层的下载，从而加快拉取速度。

除了从默认的 Docker Hub 拉取镜像，`docker pull` 也支持从其他镜像仓库拉取。用户可以在镜像名称前加上仓库的地址，例如 `docker pull registry.cn-hangzhou.aliyuncs.com/acs/nginx:latest`，这表示从阿里云的容器镜像服务（ACR）拉取 `nginx` 镜像。为了加速镜像的下载，特别是在国内网络环境下，通常会配置镜像加速器。通过在 Docker 的配置文件 `/etc/docker/daemon.json` 中添加 `registry-mirrors` 字段，并填入加速器地址，可以显著提高 `docker pull` 的速度。例如，配置 `"registry-mirrors": ["https://docker.nju.edu.cn"]` 后，Docker 在拉取镜像时会优先从南京大学的镜像源获取。`docker pull` 命令是构建和部署容器化应用的第一步，熟练掌握其用法和配置，是高效使用 Docker 的基础。

#### 2.1.2 查看本地镜像：`docker images`

`docker images` 命令用于列出本地主机上存储的所有 Docker 镜像。执行该命令后，会返回一个格式化的表格，其中包含了镜像的关键信息，方便用户快速了解本地镜像的概况。表格的列通常包括 `REPOSITORY`（仓库名/镜像名）、`TAG`（标签）、`IMAGE ID`（镜像的唯一标识符）、`CREATED`（镜像的创建时间）和 `SIZE`（镜像的大小）。例如，执行 `docker images` 可能会看到如下输出：

```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        1.21.5    605c77e624dd   2 weeks ago    141MB
ubuntu       20.04     ba6acccedd29   2 months ago   72.8MB
centos       7         eeb6ee3f44bd   3 months ago   204MB
```

这个输出清晰地展示了本地有三个镜像：一个 Nginx 镜像、一个 Ubuntu 镜像和一个 CentOS 镜像，以及它们各自的版本标签、ID、创建时间和大小。`IMAGE ID` 是镜像的唯一标识，通常是一个 12 位的十六进制字符串，在需要对镜像进行精确操作时（如删除、打标签等）非常有用。`TAG` 则用于区分同一个镜像的不同版本，例如 `nginx:1.21.5` 和 `nginx:latest` 虽然名称相同，但代表了不同的版本。

`docker images` 命令还支持一些选项来过滤和格式化输出。例如，使用 `-q` 或 `--quiet` 选项可以只显示镜像的 ID，这在需要批量处理镜像时非常有用，例如 `docker rmi $(docker images -q)` 可以删除所有本地镜像。使用 `--filter` 选项可以根据特定条件过滤镜像，例如 `docker images --filter "dangling=true"` 可以列出所有未被任何容器引用的“悬空”镜像。此外，还可以通过 `--format` 选项自定义输出格式，例如 `docker images --format "table {{.Repository}}\\t{{.Tag}}\\t{{.Size}}"` 可以只显示仓库名、标签和大小三列。这些高级选项使得 `docker images` 不仅是一个简单的列表工具，更是一个强大的镜像管理辅助命令。

#### 2.1.3 查看镜像详情：`docker inspect`

`docker inspect` 命令用于获取 Docker 对象（包括镜像、容器、网络、数据卷等）的详细配置信息。当应用于镜像时，它会返回一个包含大量信息的 JSON 格式输出，这些信息涵盖了镜像的方方面面，是深入了解镜像内部结构和配置的利器。命令的基本语法是 `docker inspect [OPTIONS] IMAGE [IMAGE...]`，其中 `IMAGE` 可以是镜像的名称（如 `nginx:1.21.5`）或镜像 ID（如 `605c77e624dd`）。返回的 JSON 数据包含了镜像的架构（Architecture）、操作系统（Os）、创建时间（Created）、作者（Author）、配置（Config）、根文件系统（RootFS）、分层信息（Layers）等。

在 `Config` 字段中，可以找到镜像的默认环境变量（Env）、工作目录（WorkingDir）、默认命令（Cmd）、入口点（Entrypoint）、暴露的端口（ExposedPorts）等关键配置信息。这些信息对于理解容器启动时的行为至关重要。例如，通过查看 `Cmd` 和 `Entrypoint`，可以知道容器启动时会执行哪个命令。`RootFS` 字段则详细列出了构成镜像的所有层（Layers），每一层都有一个唯一的 `diff_id`，这与 `docker history` 命令的输出相对应，展示了镜像的构建历史。`Architecture` 和 `Os` 字段则表明了该镜像是为哪种硬件架构和操作系统构建的，这对于在多架构环境中部署应用非常重要。

由于 `docker inspect` 的输出是 JSON 格式，通常会结合 `jq` 等命令行工具来提取特定的信息。例如，`docker inspect nginx:1.21.5 | jq '.[0].Config.Env'` 可以只查看该镜像设置的环境变量。`docker inspect` 是进行故障排查、镜像分析和自动化脚本编写时不可或缺的工具。通过它，用户可以获取到比 `docker images` 和 `docker history` 更为详尽和底层的镜像信息，从而实现对 Docker 环境的精细化管理和控制。

#### 2.1.4 查看镜像历史：`docker history`

`docker history` 命令用于展示一个 Docker 镜像的构建历史，即构成该镜像的每一层（Layer）是如何被创建的。这对于理解镜像的构成、分析镜像大小以及优化 Dockerfile 都非常有帮助。命令的基本语法是 `docker history [OPTIONS] IMAGE`，其中 `IMAGE` 可以是镜像的名称或 ID。执行该命令后，会返回一个表格，列出了镜像的每一层，从上到下依次显示了从基础镜像到最终镜像的构建过程。表格的列通常包括 `IMAGE`（层的 ID）、`CREATED`（层的创建时间）、`CREATED BY`（创建该层的指令）、`SIZE`（该层的大小）和 `COMMENT`（注释）。

例如，执行 `docker history nginx:1.21.5` 可能会看到如下输出：

```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
<missing>      2 weeks ago    /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  EXPOSE 80                    0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  ENTRYPOINT ["/docker-entr…   0B        
<missing>      2 weeks ago    /bin/sh -c #(nop) COPY file:09a214a3e51c9a5…   4.61kB    
<missing>      2 weeks ago    /bin/sh -c #(nop) COPY file:0fd5fca330dcd6a7…   1.04kB    
<missing>      2 weeks ago    /bin/sh -c #(nop) COPY file:0b866ff3fc1ef5b0…   1.95kB    
<missing>      2 weeks ago    /bin/sh -c #(nop) COPY file:65504f71f5855ca…   1.2kB     
<missing>      2 weeks ago    /bin/sh -c set -x     && addgroup --system -…   61.1MB    
<missing>      2 weeks ago    /bin/sh -c #(nop)  ENV PKG_RELEASE=1~bullseye   0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  ENV NJS_VERSION=0.7.1        0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  ENV NGINX_VERSION=1.21.5     0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B        
<missing>      2 weeks ago    /bin/sh -c #(nop)  CMD ["bash"]                 0B        
<missing>      2 months ago   /bin/sh -c #(nop) ADD file:09675d11695f65c55…   80.4MB    
```

从这个输出可以清晰地看到，这个 Nginx 镜像是基于一个 Debian 基础镜像（最后一行，大小为 80.4MB），然后通过一系列的 `RUN`、`COPY`、`ENV` 等指令逐层构建而成。`CREATED BY` 列显示了创建每一层的具体 Dockerfile 指令。`SIZE` 列则显示了每一层对镜像总大小的贡献。通过分析这些信息，可以找出镜像中体积较大的层，并针对性地优化 Dockerfile，例如合并多个 `RUN` 指令、清理不必要的缓存文件等，从而减小最终镜像的体积。`docker history` 是镜像优化和故障排查过程中的一个重要工具。

### 2.2 管理镜像

#### 2.2.1 删除镜像：`docker rmi`

`docker rmi` 命令用于删除本地主机上的一个或多个 Docker 镜像。其基本语法是 `docker rmi [OPTIONS] IMAGE [IMAGE...]`，其中 `IMAGE` 可以是镜像的名称（如 `nginx:latest`）或镜像 ID（如 `605c77e624dd`）。这个命令是清理本地磁盘空间、移除不再需要的镜像的常用手段。在执行删除操作时，Docker 会检查是否有正在运行的容器引用了该镜像。如果存在正在运行的容器，Docker 会拒绝删除该镜像，并提示错误信息。这是为了防止意外删除正在使用的镜像，导致容器无法正常运行。

为了强制删除镜像，即使存在引用它的容器，可以使用 `-f` 或 `--force` 选项。例如，`docker rmi -f nginx:latest` 会强制删除 `nginx:latest` 镜像，无论是否有容器正在使用它。需要注意的是，强制删除正在运行的容器所依赖的镜像可能会导致容器出现问题，因此应谨慎使用。一个更安全的做法是，先停止并删除所有引用该镜像的容器，然后再执行 `docker rmi` 命令。例如，可以先执行 `docker ps -a` 找到所有基于该镜像的容器，然后使用 `docker rm` 删除它们，最后再执行 `docker rmi`。

`docker rmi` 命令也支持批量删除镜像。例如，`docker rmi $(docker images -q)` 会删除本地所有的镜像。这个命令通常与 `docker images -q`（只输出镜像 ID）结合使用，以实现自动化清理。此外，还可以结合 `--filter` 选项来删除特定条件的镜像，例如 `docker rmi $(docker images -q --filter "dangling=true")` 可以删除所有未被任何容器引用的悬空镜像。熟练掌握 `docker rmi` 及其选项，对于维护一个干净、高效的本地 Docker 环境至关重要。

#### 2.2.2 重命名/打标签：`docker tag`

`docker tag` 命令用于为本地的一个镜像创建一个新的标签（Tag），或者将一个镜像标记为另一个名称。这个命令并不会创建一个新的镜像，也不会复制镜像的数据，它只是为同一个镜像 ID 创建了一个新的引用。其基本语法是 `docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`。其中，`SOURCE_IMAGE` 是源镜像的名称或 ID，`TARGET_IMAGE` 是目标镜像的名称和标签。例如，`docker tag nginx:1.21.5 myregistry.com/mynginx:v1.0` 命令会将本地的 `nginx:1.21.5` 镜像标记为 `myregistry.com/mynginx:v1.0`。

这个命令在多个场景下都非常有用。首先，在将镜像推送到私有仓库之前，通常需要使用 `docker tag` 为镜像打上符合私有仓库命名规范的标签。例如，私有仓库的地址通常是 `registry.example.com`，那么就需要将镜像标记为 `registry.example.com/namespace/image:tag` 的形式。其次，`docker tag` 也常用于为镜像创建有意义的版本标签。例如，在开发过程中，可以为每次构建的镜像打上不同的版本号标签，如 `v1.0`, `v1.1`, `v2.0` 等，以便于版本管理和回滚。一个镜像可以有多个标签，它们都指向同一个镜像 ID。例如，`nginx:latest` 和 `nginx:1.21.5` 可能指向同一个镜像。

由于 `docker tag` 只是创建了一个引用，因此它几乎是瞬时完成的，不会占用额外的磁盘空间。要删除一个标签，可以使用 `docker rmi` 命令，但需要注意的是，只有当镜像的所有标签都被删除后，镜像本身的数据才会被真正移除。`docker tag` 是镜像管理流程中的一个重要环节，它使得镜像的命名、版本控制和分发变得更加灵活和高效。

#### 2.2.3 导出镜像：`docker save`

`docker save` 命令用于将一个或多个 Docker 镜像打包成一个 tar 归档文件。这个命令在需要将镜像离线传输、备份或在没有网络连接的环境中部署应用时非常有用。其基本语法是 `docker save [OPTIONS] IMAGE [IMAGE...]`。例如，`docker save -o nginx.tar nginx:1.21.5` 命令会将 `nginx:1.21.5` 镜像保存为当前目录下的 `nginx.tar` 文件。其中，`-o` 或 `--output` 选项用于指定输出文件的路径和名称。

`docker save` 命令会将镜像的所有层（Layers）以及相关的元数据（如标签、创建时间等）都打包到 tar 文件中。这意味着，当使用 `docker load` 命令导入这个 tar 文件时，可以完整地恢复出原始的镜像。如果需要一次性导出多个镜像，可以在命令中指定多个镜像名称或 ID，例如 `docker save -o images.tar nginx:1.21.5 ubuntu:20.04`。这会将 `nginx:1.21.5` 和 `ubuntu:20.04` 两个镜像都打包到 `images.tar` 文件中。

导出的 tar 文件可以通过各种方式（如 U 盘、scp、rsync 等）传输到目标机器。在目标机器上，只需使用 `docker load` 命令即可将镜像导入到本地的 Docker 环境中。`docker save` 和 `docker load` 的组合，为 Docker 镜像的离线分发提供了一种简单而可靠的解决方案，特别适用于企业内部网络、安全要求高的环境或网络条件不佳的场景。需要注意的是，导出的 tar 文件可能会比较大，因为它包含了镜像的所有层。

#### 2.2.4 导入镜像：`docker load`

`docker load` 命令是 `docker save` 的逆操作，它用于从一个 tar 归档文件中导入 Docker 镜像。这个命令通常与 `docker save` 配合使用，用于在离线环境中恢复或部署镜像。其基本语法是 `docker load [OPTIONS]`。用户可以通过 `-i` 或 `--input` 选项指定要导入的 tar 文件，或者通过管道（pipe）从标准输入（stdin）读取数据。例如，`docker load -i nginx.tar` 命令会从 `nginx.tar` 文件中导入镜像。如果 tar 文件是通过管道传输的，可以使用 `cat nginx.tar | docker load`。

当执行 `docker load` 命令时，Docker 会读取 tar 文件中的内容，并将其中的镜像层和元数据恢复到本地的镜像仓库中。导入完成后，可以使用 `docker images` 命令来验证镜像是否已经成功导入。导入的镜像会保留其原始的名称和标签。如果本地已经存在同名的镜像，`docker load` 会覆盖它。

`docker load` 命令在以下场景中特别有用：
1.  **离线部署**：在无法访问公共镜像仓库（如 Docker Hub）的环境中，可以先将镜像在联网环境中用 `docker save` 导出，然后将 tar 文件拷贝到离线环境中，再用 `docker load` 导入。
2.  **镜像备份与恢复**：可以将重要的镜像用 `docker save` 导出并备份，当需要恢复时，再用 `docker load` 导入。
3.  **镜像分发**：在团队内部或企业内部，可以将构建好的应用镜像导出为 tar 文件，然后分发给其他成员或部署到生产服务器上。

`docker load` 和 `docker save` 共同构成了 Docker 镜像的离线管理方案，为镜像的传输、备份和分发提供了便利，是 Docker 工具链中不可或缺的一部分。

## 3. 容器管理命令

### 3.1 创建与运行容器：`docker run`

`docker run` 是 Docker 中最核心、最复杂的命令之一，它用于基于指定的镜像创建一个新的容器，并启动该容器。这个命令集成了大量的选项，允许用户对容器的运行方式进行精细的控制，包括网络设置、数据卷挂载、环境变量配置、资源限制等。其基本语法是 `docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`。其中，`IMAGE` 是指定要运行的镜像，`COMMAND` 和 `ARG` 是可选的，用于覆盖镜像中定义的默认启动命令和参数。

当执行 `docker run` 命令时，Docker 引擎会执行以下步骤：
1.  **检查本地镜像**：首先，Docker 会在本地查找指定的镜像。如果本地不存在，它会尝试从配置的镜像仓库（如 Docker Hub）中拉取该镜像。
2.  **创建容器**：Docker 会基于镜像创建一个可写的容器层，并设置好容器的文件系统。
3.  **配置网络**：根据命令行选项，Docker 会为容器分配一个网络接口，并将其连接到指定的网络（默认为 `bridge` 网络）。
4.  **挂载数据卷**：如果指定了 `-v` 或 `--mount` 选项，Docker 会将数据卷或宿主机目录挂载到容器内部。
5.  **设置环境变量**：通过 `-e` 选项设置的环境变量会被添加到容器的运行环境中。
6.  **启动容器**：最后，Docker 会启动容器，并执行镜像中定义的默认命令（或通过 `COMMAND` 参数指定的命令）。

`docker run` 命令的选项非常丰富，掌握这些选项是高效使用 Docker 的关键。例如，`-d` 选项可以让容器在后台以守护进程模式运行；`-p` 选项用于端口映射，将容器的端口暴露到宿主机上；`-v` 选项用于挂载数据卷，实现数据持久化；`--name` 选项可以为容器指定一个易于识别的名称。通过组合使用这些选项，可以灵活地创建出满足各种需求的容器化应用。

#### 3.1.1 核心选项详解

`docker run` 命令提供了大量的选项来控制容器的行为，以下是一些最常用和最重要的核心选项的详细解释：

*   **`-d`, `--detach`**: 在后台运行容器并打印出容器 ID。这是运行长期服务（如 Web 服务器、数据库）时最常用的选项。容器启动后，会立即返回命令行提示符，而容器则在后台继续运行。例如：`docker run -d nginx`。
*   **`-i`, `--interactive`**: 保持容器的标准输入（STDIN）打开。通常与 `-t` 选项一起使用，用于创建一个交互式的会话。即使容器没有连接终端，`-i` 也会保持输入流打开，使得可以向容器发送输入。
*   **`-t`, `--tty`**: 为容器分配一个伪终端（pseudo-TTY）。这使得容器内的命令行界面看起来更像一个真实的终端，支持颜色输出、命令行编辑等功能。通常与 `-i` 结合使用，即 `-it`，以获得一个完整的交互式 shell 体验。例如：`docker run -it ubuntu /bin/bash`。
*   **`-p`, `--publish`**: 将容器的端口映射到宿主机。格式为 `宿主机端口:容器端口`。这是让外部网络访问容器内服务的关键。例如，`-p 8080:80` 表示将容器的 80 端口映射到宿主机的 8080 端口。也可以指定协议，如 `-p 8080:80/tcp`。如果省略宿主机端口，Docker 会自动分配一个随机端口。
*   **`-v`, `--volume`**: 挂载数据卷或宿主机目录到容器。格式为 `宿主机路径:容器路径`（绑定挂载）或 `卷名:容器路径`（数据卷挂载）。这是实现数据持久化的主要方式。例如，`-v /data:/var/lib/mysql` 将宿主机的 `/data` 目录挂载到容器的 `/var/lib/mysql` 目录。还可以指定读写权限，如 `-v /data:/var/lib/mysql:ro` 表示只读挂载。
*   **`--name`**: 为容器指定一个名称。如果不指定，Docker 会自动生成一个随机的名称。指定一个有意义的名称可以方便地引用和管理容器，例如 `docker run --name my-web-server nginx`。
*   **`--network`**: 指定容器连接的网络。默认为 `bridge`。可以连接到自定义的 `bridge` 网络，或者使用 `host`、`none` 等特殊网络模式。例如，`--network my-custom-net`。
*   **`-e`, `--env`**: 设置环境变量。格式为 `KEY=VALUE`。可以多次使用该选项来设置多个环境变量。例如，`-e MYSQL_ROOT_PASSWORD=my-secret-pw`。
*   **`--rm`**: 在容器退出后自动删除容器。这在运行一次性任务或测试时非常有用，可以避免手动清理容器的麻烦。例如，`docker run --rm alpine ls -l`。
*   **`--restart`**: 配置容器的重启策略。例如，`--restart=always` 表示无论容器因何种原因退出，Docker 都会自动重启它。这对于需要保持高可用的服务非常重要。

熟练掌握这些核心选项，并根据实际需求进行组合，是创建和管理功能强大、配置灵活的 Docker 容器的基础。

#### 3.1.2 交互式运行示例

交互式运行容器是 Docker 的一个强大功能，它允许用户进入一个容器的 shell 环境，就像登录到一台虚拟机一样，可以执行命令、查看文件、调试程序等。这主要通过组合使用 `-i`（保持标准输入打开）和 `-t`（分配伪终端）两个选项来实现。最常见的交互式运行命令是 `docker run -it IMAGE /bin/bash`，其中 `IMAGE` 是要运行的镜像，`/bin/bash` 是容器启动后要执行的命令，即启动一个 bash shell。

例如，要在一个 Ubuntu 镜像中启动一个交互式会话，可以执行以下命令：
```bash
docker run -it ubuntu:20.04 /bin/bash
```
执行后，命令行提示符会变为 `root@container_id:/#`，表示你已经进入了容器的 root 用户 shell。在这里，你可以像在普通的 Ubuntu 系统中一样执行各种命令，比如 `ls -l` 查看目录内容，`apt-get update` 更新软件包列表，或者 `cat /etc/os-release` 查看系统信息。所有的操作都只在当前容器内部生效，不会影响到宿主机或其他容器。

要退出交互式会话，可以输入 `exit` 命令或按下 `Ctrl+D`。此时，容器会停止运行，因为容器的主进程（`/bin/bash`）已经结束。如果希望在退出后容器仍然在后台运行，可以使用 `docker run -itd IMAGE /bin/bash` 命令，即在 `-it` 的基础上再添加 `-d`（后台运行）选项。这样，即使退出了 shell，容器也会作为一个守护进程继续运行。之后，可以使用 `docker exec -it CONTAINER_ID /bin/bash` 再次进入这个正在运行的容器。

交互式运行容器在开发和调试过程中非常有用。例如，当应用容器无法正常启动时，可以以交互式方式运行同一个镜像，手动启动应用并查看错误日志，从而快速定位问题。它也是学习 Docker 和 Linux 系统的一个绝佳方式，可以安全地进行各种实验，而不用担心破坏宿主机环境。

#### 3.1.3 后台运行示例

在生产环境中，大多数服务（如 Web 服务器、数据库、消息队列等）都需要作为长期运行的守护进程在后台运行。Docker 提供了 `-d` 或 `--detach` 选项来实现这一功能。当使用 `docker run -d` 命令时，Docker 会在启动容器后立即返回容器 ID，并将容器置于后台运行，而不会占用当前终端。这是部署服务时最常用的方式。

例如，要在后台运行一个 Nginx Web 服务器，可以执行以下命令：
```bash
docker run -d --name my-nginx -p 80:80 nginx
```
在这个命令中：
*   `-d` 表示在后台运行容器。
*   `--name my-nginx` 为容器指定了一个易于识别的名称 `my-nginx`。
*   `-p 80:80` 将容器的 80 端口映射到宿主机的 80 端口，使得外部用户可以通过访问宿主机的 IP 地址来访问 Nginx 服务。
*   `nginx` 是要运行的镜像名称。

执行命令后，终端会立即返回一个容器 ID，例如 `a1b2c3d4e5f6`。此时，Nginx 服务已经在后台容器中启动并运行。你可以通过 `docker ps` 命令来查看正在运行的容器，确认 `my-nginx` 容器的状态。现在，你就可以在浏览器中访问宿主机的 IP 地址，看到 Nginx 的欢迎页面了。

对于需要在后台运行的应用，其 Dockerfile 中通常会定义一个能够持续运行的命令作为 `CMD` 或 `ENTRYPOINT`。例如，Nginx 镜像的默认命令是 `nginx -g "daemon off;"`，这个命令会让 Nginx 在前台运行，从而保持容器的活跃状态。如果容器的主进程退出，容器也会随之停止。因此，在设计需要后台运行的应用时，必须确保其主进程是长期运行的。`docker run -d` 是容器化部署服务的基础，它使得服务的启动和管理变得非常简单和高效。

#### 3.1.4 端口映射与数据卷挂载示例

端口映射和数据卷挂载是 `docker run` 命令中两个至关重要的功能，它们分别解决了容器与外部世界的通信问题和容器的数据持久化问题。

**端口映射** 通过 `-p` 选项实现，它将容器内部的网络端口映射到宿主机的端口上，从而使得外部网络可以访问容器内运行的服务。其基本格式为 `-p 宿主机端口:容器端口`。例如，运行一个 Tomcat 应用服务器，其默认端口是 8080，如果希望外部通过宿主机的 8080 端口来访问它，可以这样运行容器：
```bash
docker run -d -p 8080:8080 --name my-tomcat tomcat:9.0
```
在这个例子中，任何发送到宿主机 `IP:8080` 的 HTTP 请求都会被 Docker 引擎转发到 `my-tomcat` 容器的 8080 端口。端口映射还支持指定协议（如 `tcp` 或 `udp`），以及绑定到特定的宿主机 IP 地址，例如 `-p 127.0.0.1:8080:8080` 表示只允许从本地回环地址访问。

**数据卷挂载** 通过 `-v` 选项实现，它将宿主机上的目录或 Docker 管理的数据卷挂载到容器内部，从而实现数据的持久化和共享。这对于需要保存状态的服务（如数据库）或需要在宿主机和容器之间共享文件（如开发环境中的代码）至关重要。例如，运行一个 MySQL 数据库，并希望将其数据文件持久化存储在宿主机的 `/data/mysql` 目录下，可以这样运行容器：
```bash
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -v /data/mysql:/var/lib/mysql \
  mysql:8.0
```
在这个例子中，`-v /data/mysql:/var/lib/mysql` 将宿主机的 `/data/mysql` 目录挂载到了容器的 `/var/lib/mysql` 目录（MySQL 默认的数据存储路径）。这样，即使 `my-mysql` 容器被删除，数据库的数据文件依然会保存在宿主机的 `/data/mysql` 目录中，当再次启动一个新的 MySQL 容器并挂载同一个目录时，数据将得以恢复。

端口映射和数据卷挂载的结合使用，使得 Docker 容器能够灵活地对外提供服务，并安全地管理其数据，是构建健壮、可维护的容器化应用的核心技术。

### 3.2 容器生命周期管理

#### 3.2.1 查看容器：`docker ps`

`docker ps` 命令是用于查看和管理 Docker 容器状态的核心工具。默认情况下，它只列出当前正在运行的容器。执行该命令后，会返回一个格式化的表格，包含了容器的核心信息，如 `CONTAINER ID`（容器 ID）、`IMAGE`（使用的镜像）、`COMMAND`（启动命令）、`CREATED`（创建时间）、`STATUS`（状态）、`PORTS`（端口映射）和 `NAMES`（容器名称）。例如：

```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                  NAMES
a1b2c3d4e5f6   nginx:1.21.5   "/docker-entrypoint.…"   10 minutes ago   Up 10 minutes   0.0.0.0:80->80/tcp     my-nginx
f7g8h9i0j1k2   mysql:8.0      "docker-entrypoint.s…"   2 hours ago      Up 2 hours      3306/tcp, 33060/tcp    my-mysql
```

这个输出清晰地展示了当前有两个正在运行的容器：`my-nginx` 和 `my-mysql`，以及它们的详细信息。

为了查看包括已停止在内的所有容器，可以使用 `-a` 或 `--all` 选项，即 `docker ps -a`。这对于查找之前运行过但现在已经退出的容器非常有用，可以帮助用户了解容器的完整历史状态。

`docker ps` 还提供了其他一些有用的选项：
*   **`-q`, `--quiet`**: 只显示容器的 ID。这在需要批量操作容器时非常方便，例如 `docker stop $(docker ps -q)` 可以停止所有正在运行的容器。
*   **`--filter`, `-f`**: 根据条件过滤容器。例如，`docker ps -f "status=exited"` 可以列出所有已退出的容器，`docker ps -f "name=my-nginx"` 可以查找名称包含 `my-nginx` 的容器。
*   **`--format`**: 自定义输出格式。例如，`docker ps --format "table {{.Names}}\\t{{.Status}}"` 可以只显示容器名称和状态。

`docker ps` 是日常 Docker 操作中最高频使用的命令之一，熟练掌握其各种选项，可以极大地提高容器管理的效率。

#### 3.2.2 启动/停止容器：`docker start/stop`

`docker start` 和 `docker stop` 是用于管理容器生命周期的两个基本命令。`docker start` 用于启动一个或多个已经创建但当前处于停止状态的容器，而 `docker stop` 则用于优雅地停止一个或多个正在运行的容器。

`docker start` 命令的基本语法是 `docker start [OPTIONS] CONTAINER [CONTAINER...]`，其中 `CONTAINER` 可以是容器的名称或 ID。例如，`docker start my-nginx` 会启动名为 `my-nginx` 的容器。当一个容器被停止后，其文件系统和配置仍然保留在 Docker 中，只是其内部的进程被终止。使用 `docker start` 可以重新启动这个容器，恢复其运行状态。这对于临时停止服务进行维护或调试非常有用。

`docker stop` 命令的基本语法是 `docker stop [OPTIONS] CONTAINER [CONTAINER...]`。默认情况下，`docker stop` 会向容器的主进程发送一个 `SIGTERM` 信号，给容器一个机会来优雅地关闭，例如保存数据、关闭连接等。如果在一定时间（默认为 10 秒）后容器仍未停止，Docker 会再发送一个 `SIGKILL` 信号强制终止容器。可以通过 `-t` 或 `--time` 选项来指定等待的时间。与 `docker kill` 命令不同，`docker stop` 提供了一种更温和的停止方式，有助于保证数据的完整性。这两个命令是容器生命周期管理中最基本的操作，用于控制服务的启停。

#### 3.2.3 删除容器：`docker rm`

`docker rm` 命令用于删除一个或多个已经停止的容器。其基本语法是 `docker rm [OPTIONS] CONTAINER [CONTAINER...]`。当一个容器被删除后，其占用的磁盘空间会被释放，容器内的所有数据（除非通过数据卷持久化）都会丢失。因此，在执行删除操作前需要谨慎确认。默认情况下，`docker rm` 只能删除已经停止的容器。如果尝试删除一个正在运行的容器，命令会失败并提示错误。

如果需要强制删除一个正在运行的容器，可以使用 `-f` 或 `--force` 选项。这个选项会先向容器发送 `SIGKILL` 信号强制停止容器，然后再将其删除。例如，`docker rm -f my_container` 会强制删除名为 `my_container` 的容器。此外，`docker rm` 还支持 `-v` 或 `--volumes` 选项，在删除容器的同时，也删除与之关联的匿名数据卷。这对于清理不再需要的临时数据非常有用。为了方便批量删除已停止的容器，可以结合使用 `docker ps` 命令，例如 `docker rm $(docker ps -a -q)` 会删除所有容器（包括正在运行的，因为这里没有加 `-f` 选项，所以会失败），而 `docker container prune` 命令则可以一键删除所有已停止的容器，是清理环境的快捷方式。

### 3.3 容器内部操作

#### 3.3.1 进入容器：`docker exec`

`docker exec` 命令用于在一个正在运行的容器中执行新的命令。这是进入容器内部进行调试、管理或执行一次性任务的最常用方式。其基本语法是 `docker exec [OPTIONS] CONTAINER COMMAND [ARG...]`。其中，`CONTAINER` 是目标容器的 ID 或名称，`COMMAND` 是要在容器内执行的命令。

最常用的场景是进入容器的交互式 shell。这通常通过组合使用 `-i`（保持标准输入打开）和 `-t`（分配伪终端）选项来实现。例如，要进入一个名为 `my_nginx` 的 Nginx 容器的 bash shell，可以执行：

```bash
docker exec -it my_nginx /bin/bash
```

执行该命令后，用户就会获得一个容器内部的 shell 会话，可以像操作一个普通的 Linux 系统一样执行命令。当需要退出时，输入 `exit` 即可，这只会退出 shell 会话，而不会停止容器本身。除了进入 shell，`docker exec` 还可以用于执行任何其他命令，例如 `docker exec my_nginx nginx -t` 可以在容器内测试 Nginx 配置文件的语法是否正确，而无需进入交互式 shell。这种非交互式的用法在自动化脚本中非常有用。

#### 3.3.2 文件复制：`docker cp`

`docker cp` 命令用于在宿主机和容器之间复制文件或目录。这是一个非常实用的功能，用于在容器和外部世界之间交换数据。其基本语法是 `docker cp [OPTIONS] SRC_PATH DEST_PATH`。其中，`SRC_PATH` 和 `DEST_PATH` 可以是宿主机路径或容器路径，容器路径的格式为 `CONTAINER:PATH`。

例如，要将宿主机上的 `index.html` 文件复制到名为 `my_nginx` 的容器的 `/usr/share/nginx/html/` 目录下，可以执行：

```bash
docker cp index.html my_nginx:/usr/share/nginx/html/
```

反之，要将容器内的文件复制到宿主机，只需将源路径和目标路径调换即可。例如，要将 `my_nginx` 容器内的 Nginx 配置文件复制到宿主机的当前目录，可以执行：

```bash
docker cp my_nginx:/etc/nginx/nginx.conf .
```

`docker cp` 命令在容器调试和数据备份等场景中非常有用。例如，当容器内的应用出现问题时，可以将其日志文件复制到宿主机进行详细分析。或者，当需要备份容器内生成的数据时，也可以使用该命令将数据目录复制出来。需要注意的是，`docker cp` 只能用于正在运行的容器，或者已经停止但文件系统仍然存在的容器。

#### 3.3.3 查看容器进程：`docker top`

`docker top` 命令用于显示一个正在运行的容器内部的所有进程。这对于监控容器内应用的运行状态、排查性能问题非常有帮助。其基本语法是 `docker top CONTAINER [ps OPTIONS]`，其中 `CONTAINER` 是目标容器的 ID 或名称。该命令实际上是在容器内部执行 `ps` 命令，并将结果输出到宿主机终端。

执行 `docker top` 后，会显示一个进程列表，其中包含了进程 ID（PID）、父进程 ID（PPID）、启动命令（CMD）等信息。通过查看这个列表，可以了解容器内正在运行哪些进程，以及它们之间的关系。例如，可以判断一个服务是否成功启动，或者是否存在僵尸进程。`docker top` 命令还支持传递 `ps` 命令的选项，例如 `docker top my_container aux` 可以显示更详细的进程信息，包括 CPU 和内存使用率。结合 `docker stats` 命令，可以更全面地监控容器的资源消耗和进程状态，是进行容器性能调优和故障排查的重要工具。

#### 3.3.4 查看容器资源使用：`docker stats`

`docker stats` 命令用于实时显示一个或多个容器的资源使用情况，包括 CPU、内存、网络和磁盘 I/O。这是一个非常强大的监控工具，可以帮助用户了解容器的性能表现，及时发现资源瓶颈。其基本语法是 `docker stats [OPTIONS] [CONTAINER...]`。如果不指定容器，该命令会显示所有正在运行的容器的统计信息。

执行 `docker stats` 后，会显示一个动态更新的表格，其中包含了容器的 ID、名称、CPU 使用率、内存使用量/限制、内存使用率、网络接收/发送的数据量以及块设备 I/O 等信息。这些数据每秒更新一次，可以实时反映容器的负载情况。例如，通过观察 CPU 使用率，可以判断容器内的应用是否繁忙；通过观察内存使用量，可以判断是否存在内存泄漏的风险。`docker stats` 命令还支持 `--no-stream` 选项，它只会显示一次当前的统计信息，然后退出，这在脚本中非常有用。通过持续监控 `docker stats` 的输出，可以为容器的资源限制（如 `-m` 和 `--cpus`）提供数据依据，从而实现更精细化的资源管理。

## 4. 数据持久化

### 4.1 三种方式对比

Docker 提供了三种主要的数据持久化机制：Volume、Bind Mount 和 tmpfs。它们各有特点，适用于不同的场景。

| 方式           | 特点                                                         | 适用场景                                         | 核心操作                                                     |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------- | :----------------------------------------------------------- |
| **Volume**     | 由 Docker 引擎管理，存储在宿主机特定区域（`/var/lib/docker/volumes/`），与容器生命周期解耦，支持多容器共享，易于备份和迁移。 | 生产环境、数据库数据、需要长期保存和共享的数据。 | 创建：`docker volume create 卷名`；挂载：`-v 卷名:容器路径`  |
| **Bind Mount** | 将宿主机上的任意目录或文件直接挂载到容器中，灵活性高，性能与宿主机文件系统相同。 | 开发环境、需要实时同步代码、配置文件的场景。     | 挂载：`-v 宿主机绝对路径:容器路径`（例：`-v /data/html:/usr/share/nginx/html`） |
| **tmpfs**      | 数据存储在宿主机的内存中，读写速度极快，但数据是临时的，容器停止后数据即丢失。 | 存放临时文件、敏感信息（如密钥）、高并发缓存。   | 挂载：`--tmpfs 容器路径`                                     |

**Volume** 是 Docker 官方推荐的生产环境数据持久化方式。它的生命周期独立于容器，由 Docker 引擎统一管理，这使得数据的管理、备份和迁移更加方便和安全。用户无需关心数据在宿主机上的具体存储位置，只需通过卷名进行引用即可。

**Bind Mount** 提供了最大的灵活性，因为它允许用户将宿主机上的任何路径挂载到容器中。这在开发过程中非常有用，例如，可以将本地的源代码目录直接挂载到容器中，这样任何代码的修改都可以立即在运行的应用中生效，无需重新构建镜像。然而，由于它依赖于宿主机的文件系统结构，因此在不同环境之间移植性较差，并且可能存在权限问题。

**tmpfs** 是一种特殊的挂载类型，它将数据存储在内存中。这使得数据的读写速度非常快，非常适合用作临时文件存储或缓存。但是，由于数据只存在于内存中，一旦容器停止或宿主机重启，所有数据都会丢失。此外，它也不能在容器之间共享。

### 4.2 常用操作与示例

#### 4.2.1 创建与管理数据卷

数据卷（Volume）的管理主要通过 `docker volume` 子命令来完成。

-   **创建数据卷**:
    ```bash
    docker volume create my-vol
    ```
    这会创建一个名为 `my-vol` 的数据卷。创建后，Docker 会在其默认的存储区域（通常是 `/var/lib/docker/volumes/my-vol/_data`）为该卷分配一个目录。

-   **查看数据卷**:
    ```bash
    docker volume ls
    ```
    该命令会列出所有已创建的数据卷。

-   **查看数据卷详情**:
    ```bash
    docker volume inspect my-vol
    ```
    这会以 JSON 格式显示数据卷的详细信息，包括其名称、驱动、挂载点（Mountpoint）等。通过 `Mountpoint` 可以找到数据在宿主机上的实际存储路径。

-   **删除数据卷**:
    ```bash
    docker volume rm my-vol
    ```
    这会删除指定的数据卷。注意，只有当没有容器使用该卷时，才能成功删除。

-   **清理未使用的数据卷**:
    ```bash
    docker volume prune
    ```
    这是一个危险操作，它会删除所有未被任何容器使用的数据卷。执行前系统会要求确认。

#### 4.2.2 Nginx 数据持久化示例

以 Nginx 为例，演示如何使用 Volume 和 Bind Mount 实现数据持久化。

**使用 Volume 持久化 Nginx 的静态文件和配置**:
```bash
# 1. 创建数据卷
docker volume create nginx-html
docker volume create nginx-conf

# 2. 运行容器并挂载数据卷
docker run -d --name my-nginx \
  -p 80:80 \
  -v nginx-html:/usr/share/nginx/html \
  -v nginx-conf:/etc/nginx \
  nginx
```
在这个例子中，我们创建了两个数据卷 `nginx-html` 和 `nginx-conf`，分别用于存储 Nginx 的静态网页文件和配置文件。然后，在运行容器时，使用 `-v` 选项将这两个卷挂载到容器内的相应路径。这样，无论是修改网页内容还是更新 Nginx 配置，所有变更都会被持久化到数据卷中，即使容器被删除，数据也不会丢失。

**使用 Bind Mount 进行开发调试**:
```bash
# 假设本地有一个项目目录 ./my-website，里面包含 index.html 和 css/ 文件夹
docker run -d --name my-nginx-dev \
  -p 8080:80 \
  -v $(pwd)/my-website:/usr/share/nginx/html:ro \
  nginx
```
在这个开发场景中，我们将本地项目目录 `./my-website` 直接挂载到容器的 `/usr/share/nginx/html` 目录。`:ro` 表示以只读模式挂载，防止容器内的进程意外修改宿主机上的源文件。这样，开发者在本地对 `index.html` 或 CSS 文件的任何修改，都会立即反映在浏览器中访问 `http://localhost:8080` 的结果上，极大地提高了开发效率。

## 5. 网络配置

### 5.1 网络模式

Docker 提供了多种网络模式，以满足不同场景下的容器通信需求。

-   **bridge 模式**: 这是 Docker 的默认网络模式。当 Docker 服务启动时，它会在宿主机上创建一个名为 `docker0` 的虚拟网桥。所有使用 bridge 模式的容器都会连接到这个网桥上，并被分配一个独立的 IP 地址。容器之间可以通过 IP 地址进行通信，但要通过容器名通信，则需要将它们连接到用户自定义的 bridge 网络。容器与宿主机外部网络的通信需要通过 NAT（网络地址转换）实现。

-   **host 模式**: 使用 `--net=host` 选项指定。在这种模式下，容器将共享宿主机的网络栈，不再拥有独立的网络命名空间。容器的网络配置（如 IP 地址、端口）与宿主机完全相同。这种模式的优点是网络性能高，没有 NAT 带来的开销，但缺点是牺牲了网络隔离性，容器与宿主机之间的端口可能会发生冲突。

-   **none 模式**: 使用 `--net=none` 选项指定。在这种模式下，容器拥有自己的网络命名空间，但没有任何网络配置。容器内部除了 `lo`（回环）接口外，没有其他网络接口。这种模式适用于对安全性要求极高、需要完全隔离网络环境的场景，用户可以根据需要手动为容器配置网络。

-   **overlay 模式**: 这是一种跨主机网络模式，主要用于 Docker Swarm 集群。它允许不同宿主机上的容器通过虚拟网络进行通信，就像它们在同一个局域网中一样。Overlay 网络利用 VXLAN 等技术在底层物理网络之上创建一个虚拟的、可跨越主机的二层网络，是实现分布式应用服务发现和负载均衡的基础。

-   **macvlan 模式**: 这种模式可以为容器分配一个宿主机网络上的独立 MAC 地址，使其在物理网络上表现为一个独立的设备。容器将直接暴露在物理网络上，可以直接获取物理网络中的 IP 地址。这种模式适用于需要与物理网络直接集成、对网络性能和隔离性都有较高要求的场景。

### 5.2 网络管理命令

Docker 提供了一系列 `docker network` 子命令来管理网络。

#### 5.2.1 查看网络：`docker network ls`

`docker network ls` 命令用于列出 Docker 主机上所有的网络。执行该命令会返回一个表格，包含网络的 `NETWORK ID`、`NAME`、`DRIVER` 和 `SCOPE`。默认情况下，系统会包含 `bridge`、`host` 和 `none` 这三个预定义的网络。

#### 5.2.2 创建网络：`docker network create`

`docker network create` 命令用于创建一个新的网络。默认情况下，它会创建一个 `bridge` 类型的网络。例如，要创建一个名为 `my-net` 的自定义桥接网络，可以执行：

```bash
docker network create my-net
```
创建后，可以使用 `docker network ls` 查看。在创建网络时，可以指定多种选项，如 `--driver` 指定网络驱动（如 `bridge`, `overlay`），`--subnet` 和 `--gateway` 指定子网和网关等。

#### 5.2.3 连接网络：`docker network connect`

`docker network connect` 命令用于将一个正在运行的容器连接到一个网络。这使得容器可以获得该网络中的一个 IP 地址，并能够与同一网络中的其他容器通信。例如，要将名为 `my-container` 的容器连接到 `my-net` 网络，可以执行：

```bash
docker network connect my-net my-container
```
连接后，容器就可以通过容器名直接访问 `my-net` 网络中的其他容器了。一个容器可以同时连接到多个网络，实现复杂的网络拓扑。

## 6. 资源限制与系统配置

### 6.1 资源限制

为了防止单个容器耗尽宿主机的所有资源（CPU、内存等），从而影响其他容器或宿主机的稳定性，Docker 提供了资源限制功能。这些功能主要依赖于 Linux 内核的 cgroups（控制组）技术。

#### 6.1.1 限制CPU

Docker 提供了多种方式来限制容器的 CPU 使用：

-   **`--cpus`**: 这是最直观的方式，用于限制容器可以使用的 CPU 核心数。例如，`--cpus=1.5` 表示容器最多可以使用 1.5 个 CPU 核心的计算能力。这个限制是硬性的，即使宿主机 CPU 空闲，容器也无法超过这个限制。
-   **`--cpu-shares`**: 这个选项用于设置 CPU 的相对权重。它是一个相对值，用于在多个容器竞争 CPU 资源时，决定它们获得 CPU 时间的比例。默认值是 1024。例如，如果一个容器的 `--cpu-shares` 设置为 2048，另一个设置为 1024，那么在 CPU 资源紧张时，前者将获得两倍于后者的 CPU 时间。这个限制是软性的，只有在资源竞争时才起作用。
-   **`--cpuset-cpus`**: 这个选项用于将容器绑定到特定的 CPU 核心上。例如，`--cpuset-cpus="0,1"` 表示容器只能在宿主机的第 0 和第 1 个 CPU 核心上运行。这可以用于实现 CPU 的硬隔离，避免容器之间的 CPU 资源争抢。

#### 6.1.2 限制内存

限制内存使用主要通过 `-m` 或 `--memory` 选项实现。例如，`-m 512m` 表示限制容器的内存使用上限为 512MB。这是一个硬性限制，如果容器尝试使用超过这个限制的内存，可能会触发 OOM（Out of Memory） killer，导致容器被强制终止。

此外，还可以使用 `--memory-swap` 选项来设置内存和交换空间（swap）的总和限制。例如，`--memory=512m --memory-swap=1g` 表示容器可以使用 512MB 的物理内存和 512MB 的交换空间，总计 1GB。如果 `--memory-swap` 设置为与 `--memory` 相同的值，则表示禁用交换空间。

### 6.2 容器与宿主机时间同步

由于 Docker 容器默认使用 UTC 时间，而宿主机通常使用本地时区（如中国的 CST，即 UTC+8），这会导致容器内的时间与宿主机不一致，可能引发日志记录、定时任务等问题。解决这个问题主要有以下几种方法：

1.  **挂载宿主机的时间文件**: 这是最常用和推荐的方法。在运行容器时，将宿主机的 `/etc/localtime` 文件挂载到容器的相同位置。
    ```bash
    docker run -d -v /etc/localtime:/etc/localtime:ro 镜像名
    ```
    `:ro` 表示以只读模式挂载。这样，容器就会使用与宿主机相同的时区设置。

2.  **设置环境变量**: 许多 Linux 发行版和应用程序支持通过 `TZ` 环境变量来设置时区。
    ```bash
    docker run -d -e TZ=Asia/Shanghai 镜像名
    ```
    这种方法简单，但依赖于容器内的基础镜像和应用程序是否支持该环境变量。

3.  **在 Dockerfile 中指定**: 在构建镜像时，可以在 Dockerfile 中通过 `RUN` 指令来设置时区。
    ```dockerfile
    RUN ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone
    ```

### 6.3 镜像加速配置

在中国大陆，由于网络原因，从 Docker Hub 拉取镜像可能会非常慢。为了解决这个问题，可以配置镜像加速器。镜像加速器是 Docker Hub 的镜像缓存服务器，可以显著提高镜像的下载速度。

配置方法如下：

1.  **编辑 Docker 配置文件**：
    ```bash
    sudo mkdir -p /etc/docker
    sudo vim /etc/docker/daemon.json
    ```

2.  **添加镜像加速器地址**：
    在文件中添加 `registry-mirrors` 字段，并填入一个或多个加速器地址。
    ```json
    {
      "registry-mirrors": [
        "https://docker.nju.edu.cn",
        "https://registry.cn-hangzhou.aliyuncs.com",
        "http://hub-mirror.c.163.com"
      ]
    }
    ```

3.  **重启 Docker 服务**：
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```
    重启后，Docker 在拉取镜像时会优先从配置的镜像加速器获取，从而大大加快下载速度。

## 7. Dockerfile 与镜像构建

Dockerfile 是一个用于自动化构建 Docker 镜像的文本文件，它包含了一系列指令，每一条指令都会在镜像上创建一个新的层，从而定义了镜像的内容和启动方式。通过使用 Dockerfile，开发者可以确保应用环境的一致性，实现快速、可重复的部署。掌握 Dockerfile 的编写是高效使用 Docker 的核心技能之一。本章节将详细解析 Dockerfile 的核心指令，并介绍如何使用 `docker build` 命令来构建自定义镜像。

### 7.1 Dockerfile 核心指令

Dockerfile 中的指令是构建镜像的基石，它们定义了从基础镜像选择、文件复制、环境配置到最终启动命令的完整流程。理解并熟练运用这些指令，是编写高效、安全、可维护的 Dockerfile 的前提。以下将详细介绍最常用和最重要的核心指令。

#### 7.1.1 `FROM`：指定基础镜像

`FROM` 指令是 Dockerfile 中**必须且是第一条**非注释指令，它为后续的构建过程指定了一个基础镜像 。所有的构建操作都是在这个基础镜像之上进行的。选择合适的基础镜像至关重要，它直接影响到最终镜像的大小、安全性和维护成本。例如，对于需要轻量级环境的应用，可以选择 `alpine` 或 `debian:slim` 等精简版 Linux 发行版作为基础镜像 。`FROM` 指令的语法非常灵活，支持指定镜像的标签（tag）或摘要（digest）以确保版本的一致性。

**语法：**
```dockerfile
FROM <image>[:<tag>] [AS <name>]
```
- `<image>`: 指定基础镜像的名称，例如 `ubuntu`, `python`, `nginx`。
- `<tag>`: 指定基础镜像的版本标签，例如 `22.04`, `3.9-slim`, `1.21.5`。如果省略，默认为 `latest` 。
- `AS <name>`: 为当前的构建阶段命名，这在多阶段构建（Multi-stage builds）中非常有用，可以在后续的构建阶段中引用之前阶段的产物，从而有效减小最终镜像的体积 。

**示例：**
```dockerfile
# 使用官方的 Ubuntu 22.04 作为基础镜像
FROM ubuntu:22.04

# 使用 Python 3.9 的 slim 版本，并命名此阶段为 "builder"
FROM python:3.9-slim AS builder
```
在多阶段构建的示例中，我们可以在后续阶段使用 `COPY --from=builder` 来从 `builder` 阶段复制编译好的文件，而无需将编译工具链包含在最终的生产镜像中，这是一种优化镜像大小的常用技巧 。

#### 7.1.2 `RUN`：执行命令

`RUN` 指令用于在镜像的构建过程中执行命令，是安装软件包、配置环境和执行其他构建任务的核心指令 。每一条 `RUN` 指令都会在当前镜像层之上创建一个新的层，并提交执行结果。为了优化镜像大小和构建速度，最佳实践是将多个相关的命令通过 `&&` 连接，并在同一行中执行，这样可以减少镜像的层数。此外，在执行如 `apt-get` 或 `yum` 等包管理器命令后，通常会清理缓存以进一步减小镜像体积 。

`RUN` 指令支持两种格式：
1.  **Shell 格式**：`RUN <command>`。在 Linux 上，默认使用 `/bin/sh -c` 来执行命令；在 Windows 上，则使用 `cmd /S /C` 。
2.  **Exec 格式**：`RUN ["executable", "param1", "param2"]`。这种格式不调用 shell，而是直接执行指定的可执行文件，因此不会进行 shell 处理，如变量替换等。

**示例：**
```dockerfile
# Shell 格式：更新包列表并安装 nginx，然后清理缓存
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*

# Exec 格式：直接执行 echo 命令
RUN ["/bin/bash", "-c", "echo 'Hello, Docker!'"]
```
在编写 `RUN` 指令时，应始终考虑其对镜像层数和大小的影响。将多个命令合并成一个 `RUN` 指令，并在命令末尾清理不必要的文件和缓存，是构建高效镜像的关键步骤 。

#### 7.1.3 `COPY` 与 `ADD`：复制文件

`COPY` 和 `ADD` 指令都用于将文件或目录从构建上下文（即 `docker build` 命令指定的路径，通常是 Dockerfile 所在的目录）复制到镜像的文件系统中 。尽管两者功能相似，但 `COPY` 指令的行为更明确、可预测，因此在大多数情况下是首选。`ADD` 指令则提供了一些额外的功能，如自动解压压缩文件和从远程 URL 下载文件，但这些功能有时会导致不可预测的行为，因此应谨慎使用 。

**`COPY` 指令：**
- **功能**：纯粹地复制本地文件或目录到镜像中。
- **语法**：`COPY [--chown=<user>:<group>] <src>... <dest>`
- **最佳实践**：推荐优先使用 `COPY`，因为它简单、透明，只执行最基本的复制操作，保证了构建过程的可预测性 。

**`ADD` 指令：**
- **功能**：除了 `COPY` 的功能外，还支持：
    - **自动解压**：如果源文件是 `tar`, `gzip`, `bzip2` 等格式的压缩包，`ADD` 会自动将其解压到目标路径。
    - **远程 URL 支持**：如果源文件是一个 URL，`ADD` 会尝试下载该文件到镜像中。
- **语法**：`ADD [--chown=<user>:<group>] <src>... <dest>`
- **注意事项**：由于 `ADD` 的额外功能，其行为可能不如 `COPY` 直观。例如，如果远程 URL 的文件权限或内容发生变化，可能会导致构建结果不一致。因此，除非确实需要自动解压或下载功能，否则应优先使用 `COPY` 。

**示例：**
```dockerfile
# 使用 COPY 复制本地文件
COPY requirements.txt /app/

# 使用 ADD 自动解压一个 tar 包
ADD app.tar.gz /opt/

# 使用 ADD 从 URL 下载文件
ADD https://example.com/data.zip /tmp/
```
在选择 `COPY` 和 `ADD` 时，应遵循“最小惊讶原则”，即选择行为最简单、最可预测的指令。对于大多数场景，`COPY` 是更好的选择。

#### 7.1.4 `ENV`：设置环境变量

`ENV` 指令用于在镜像中设置环境变量，这些变量在镜像构建过程和容器运行时都有效 。环境变量是配置容器化应用的一种灵活方式，可以用来传递配置信息，如数据库连接字符串、API 密钥、应用端口等，而无需硬编码到应用代码中。通过 `ENV` 设置的变量，可以在 Dockerfile 的后续指令中通过 `$VARIABLE_NAME` 的形式引用，也可以在运行的容器中通过 `env` 命令或应用程序代码访问。

**语法：**
```dockerfile
ENV <key>=<value> ...
```
可以一次设置多个环境变量，用空格分隔。

**示例：**
```dockerfile
# 设置单个环境变量
ENV NODE_ENV=production

# 设置多个环境变量
ENV APP_PORT=8080 APP_VERSION=1.0.0

# 在后续指令中引用环境变量
WORKDIR $APP_HOME
```
与 `ARG` 指令不同，`ENV` 设置的环境变量会持久化到最终的镜像中，并在容器运行时依然存在。这使得 `ENV` 非常适合定义那些需要在容器运行时保持不变的配置项。而 `ARG` 定义的变量仅在构建过程中有效，容器运行后则不存在，适用于传递构建时的参数，如版本号等 。

#### 7.1.5 `CMD` 与 `ENTRYPOINT`：容器启动命令

`CMD` 和 `ENTRYPOINT` 指令都用于定义容器启动时要执行的命令，但它们在行为上有关键区别，理解这两者的差异对于正确配置容器的启动行为至关重要 。

**`CMD` 指令：**
- **作用**：为容器提供一个**默认的**启动命令。当 `docker run` 命令没有指定要运行的命令时，`CMD` 的内容会被执行。
- **可被覆盖**：如果在 `docker run` 命令的末尾指定了其他命令，`CMD` 的内容将被完全覆盖。
- **语法**：
    - `CMD ["executable", "param1", "param2"]` (Exec 格式，推荐)
    - `CMD command param1 param2` (Shell 格式)
    - `CMD ["param1", "param2"]` (作为 ENTRYPOINT 的默认参数)

**`ENTRYPOINT` 指令：**
- **作用**：配置容器使其像一个可执行文件一样运行。`ENTRYPOINT` 定义的命令在容器启动时**一定会被执行**，并且 `docker run` 命令行中指定的所有参数都会被当作参数传递给 `ENTRYPOINT`。
- **不可被覆盖**：`ENTRYPOINT` 本身不会被 `docker run` 的命令行参数覆盖（除非使用 `--entrypoint` 标志）。
- **语法**：
    - `ENTRYPOINT ["executable", "param1", "param2"]` (Exec 格式，推荐)
    - `ENTRYPOINT command param1 param2` (Shell 格式)

**最佳实践：组合使用 `ENTRYPOINT` 和 `CMD`**
一种常见的模式是使用 `ENTRYPOINT` 定义容器的主进程或可执行文件，然后使用 `CMD` 为该进程提供默认的参数。这样，容器既可以像一个标准的可执行文件一样运行，又允许用户在运行时通过 `docker run` 轻松覆盖默认参数 。

**示例：**
```dockerfile
# 使用 ENTRYPOINT 定义主命令
ENTRYPOINT ["java", "-jar", "app.jar"]

# 使用 CMD 提供默认参数
CMD ["--server.port=8080"]
```
**运行效果分析：**
- `docker run myimage`: 容器将执行 `java -jar app.jar --server.port=8080`。
- `docker run myimage --server.port=9090`: `CMD` 的默认参数被覆盖，容器将执行 `java -jar app.jar --server.port=9090`。

这种组合方式提供了极大的灵活性，使得镜像既易于使用（有合理的默认值），又易于扩展（可以方便地修改参数）。

### 7.2 构建镜像：`docker build`

编写完 Dockerfile 后，需要使用 `docker build` 命令来根据 Dockerfile 中的指令构建一个新的 Docker 镜像。这个命令会读取 Dockerfile，并将构建上下文（通常是当前目录及其子目录）发送给 Docker 守护进程，然后由守护进程逐条执行指令，最终生成镜像 。

**核心命令：**
```bash
docker build [OPTIONS] PATH | URL | -
```
- **`-t, --tag`**: 为构建成功的镜像指定一个名称和可选的标签（tag）。格式为 `name:tag`。如果不指定 `tag`，默认为 `latest`。可以指定多个 `-t` 参数来为镜像添加多个标签 。
- **`PATH`**: 指定构建上下文的路径。`.` 表示当前目录，这是最常用的方式。Docker 会将该路径下的所有内容（除了 `.dockerignore` 中排除的文件）发送给 Docker 守护进程 。
- **`-f, --file`**: 如果 Dockerfile 的文件名不是默认的 `Dockerfile`，或者不在构建上下文的根目录，可以使用此选项指定 Dockerfile 的路径。

**示例：**
```bash
# 在当前目录构建镜像，并命名为 myapp:v1
docker build -t myapp:v1 .

# 使用指定路径下的 Dockerfile 构建镜像
docker build -t myapp:v2 -f /path/to/Dockerfile .

# 构建并添加多个标签
docker build -t myapp:latest -t myapp:v1.0.0 .
```
在构建过程中，Docker 会缓存每一条指令的执行结果。如果某条指令及其之前的所有指令都没有发生变化，Docker 会直接使用缓存的层，从而大大加快构建速度。当某条指令发生变化时，该指令及其之后的所有层都将被重新构建。理解并利用这一缓存机制，是优化 Dockerfile 构建效率的关键 。

## 8. 部署与环境配置

### 8.1 CentOS 安装步骤

在 CentOS 系统上安装 Docker 的步骤如下：

1.  **卸载旧版本**（如果存在）：
    ```bash
    sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
    ```

2.  **安装依赖包**：
    ```bash
    sudo yum install -y yum-utils
    ```

3.  **设置 Docker 仓库**（推荐使用国内源，如阿里云）：
    ```bash
    sudo yum-config-manager \
        --add-repo \
        http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    ```

4.  **安装 Docker Engine**：
    ```bash
    sudo yum install -y docker-ce docker-ce-cli containerd.io
    ```

5.  **启动 Docker 并设置开机自启**：
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

6.  **验证安装**：
    ```bash
    sudo docker run hello-world
    ```
    如果看到欢迎信息，说明 Docker 已成功安装并运行。

### 8.2 配置镜像加速器

在中国大陆，为了加快从 Docker Hub 拉取镜像的速度，强烈建议配置镜像加速器。以下是手动配置的方法：

1.  **创建或编辑配置文件**：
    ```bash
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-'EOF'
    {
      "registry-mirrors": [
        "https://docker.nju.edu.cn",
        "https://registry.cn-hangzhou.aliyuncs.com",
        "http://hub-mirror.c.163.com"
      ]
    }
    EOF
    ```

2.  **重启 Docker 服务**：
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

配置完成后，再次使用 `docker pull` 命令时，速度会有显著提升。

## 9. 快速清理命令

### 9.1 清理已退出容器

随着时间的推移，系统中可能会积累大量已退出（Exited）的容器，占用磁盘空间。可以使用以下命令一键清理所有已退出的容器：

```bash
docker rm $(docker ps -a -q -f "status=exited")
```
或者使用更简洁的 `docker container prune` 命令，它会删除所有已停止的容器，并在执行前要求确认。

### 9.2 强制删除所有容器

在开发或测试环境中，有时需要快速清理所有容器（无论其运行状态）。可以使用以下命令强制删除所有容器：

```bash
docker rm -f $(docker ps -aq)
```
**警告**：这是一个非常危险的操作，会无条件地停止并删除系统中的所有容器，请谨慎使用。

### 9.3 清理未使用数据卷

未使用的数据卷（即没有任何容器引用的卷）也会占用磁盘空间。可以使用以下命令清理它们：

```bash
docker volume prune
```
执行此命令后，系统会列出所有将被删除的卷，并要求确认。确认后，这些未使用的数据卷将被永久删除。