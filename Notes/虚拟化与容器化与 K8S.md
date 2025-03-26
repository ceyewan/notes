## 1 虚拟化

KVM（Kernel-based Virtual Machine，基于内核的虚拟机）是 Linux 内核中的一个虚拟化技术。它允许一台物理服务器运行多个虚拟机（VM），每个虚拟机就像一台独立的电脑，拥有自己的操作系统（如 Windows、Ubuntu 等）。KVM 是开源的，广泛用于云计算和服务器虚拟化。依赖**硬件辅助虚拟化**、**Hypervisor 架构**和 **QEMU 组件**运行。

### 1.1 硬件辅助虚拟化

- 现代 CPU 内置了虚拟化支持（VT-x、AMD-V），允许 CPU 能直接识别和运行虚拟机的指令；
- CPU 提供两种运行模式，**Root 模式**和 **Non-Root 模式**，当虚拟机启动时，CPU 切换到 Non-Root 模式，虚拟机的代码直接在硬件上运行，就像跑在真实的电脑上。而访问硬件设备这种需要宿主机帮助的操作，CPU 会退出到 Root 模式，让 KVM 接管。

### 1.2 Hypervisor 架构

- Hypervisor（虚拟机监视器）是虚拟化的核心，它在物理硬件和虚拟机之间架起一座桥梁，负责分配和管理资源。
- KVM 是一个 **Type-1 Hypervisor**（裸金属型），直接运行在硬件上，而不是依赖另一个操作系统。
- KVM 其实是一个 Linux 内核模块（.ko 文件）。加载它后，Linux 内核就变成了一个 Hypervisor。

### 1.3 QEMU 组件

- QEMU 是 KVM 的用户态模拟器，负责模拟虚拟机的硬件（如磁盘、网络、显卡）。
- KVM 负责 CPU 和内存虚拟化。QEMU 提供设备模型（如 virtio 驱动）和 I/O 处理。
- QEMU 默认模拟真实硬件效率不高，通过 **Virtio** 半虚拟化驱动，可以绕过模拟，直接和宿主机通信。

### 1.4 虚拟化与容器化

- **虚拟化**：通过虚拟化技术（如 KVM、VMware、Hyper-V），在一台物理服务器上运行多个完整的虚拟机（VM）。每个虚拟机拥有独立的操作系统、内核和虚拟硬件。
- **容器化**：基于操作系统级虚拟化（如 Docker、LXC），在同一主机内核上运行多个隔离的用户空间实例（容器）。容器共享主机操作系统内核，无需独立的操作系统。
- **虚拟化**：依赖 Hypervisor（如 KVM）在硬件和虚拟机之间建立抽象层。Hypervisor 可以是 Type-1（直接运行在硬件上，如 KVM + QEMU）或 Type-2（运行在主机操作系统上，如 VirtualBox）。
- **容器化**：无 Hypervisor，直接运行在主机操作系统之上，通过内核的命名空间（Namespaces）和控制组（cGroups）实现隔离和资源限制。

| **特性** | **虚拟化 (KVM)** | **容器化 (Docker)** |
| --- | --- | --- |
|**架构**|Hypervisor + 独立 OS|共享主机内核|
|**资源消耗**|高（GB 级镜像，固定分配）|低（MB 级镜像，按需分配）|
|**启动速度**|慢（10-60 秒）|快（毫秒至 2 秒）|
|**I/O 性能**|中等（有 Hypervisor 开销）|高（接近原生）|
|**隔离性**|强（内核级隔离）|中等（用户空间隔离）|
|**适用场景**|多 OS、多租户、高安全性|微服务、开发测试、快速部署|

## 2 Linux 容器基础

### 2.1 Container

容器是一种轻量级的虚拟化技术，它允许你在操作系统上隔离运行的应用程序及其依赖环境。与传统的虚拟机（VM）不同，容器不包含完整的操作系统，而是共享宿主操作系统的内核。容器通过打包代码、运行时、系统工具和库，使得应用程序可以在不同的环境中一致运行。它的核心优势是高效、快速启动和资源占用少。

### 2.2 Linux Namespace

Namespace 是 Linux 内核提供的一种资源隔离机制，用于将系统的全局资源分隔成独立的单元。每个命名空间内的进程只能看到属于自己命名空间的资源，而与其他命名空间隔离。常见的命名空间类型包括：

- **PID Namespace**：隔离进程 ID，容器内的进程有独立的 PID 编号。
- **Network Namespace**：隔离网络资源，如网络接口、IP 地址和端口。
- **Mount Namespace**：隔离文件系统挂载点，容器有自己的文件系统视图。
- **UTS Namespace**：隔离主机名和域名，容器可以有自己的 hostname。
- **User Namespace**：隔离用户和权限，容器内的 root 可以映射为宿主机的普通用户。
- **IPC Namespace**：隔离进程间通信资源，如消息队列。

通过 Namespace，容器中的进程仿佛运行在一个独立的 " 操作系统 " 中，但实际上它们共享同一个内核。

创建一个 Linux 命名空间非常简单，我们使用一个名为 `unshare` 的包来创建一个独立的进程：

```go
sudo unshare --fork --pid --mount-proc bash
```

这条命令使用了 Linux 的 unshare 工具，它的作用是创建一个新的命名空间，并将后续的进程放入这个独立的命名空间中。让我们逐个分析选项：

- `unshare` 是一个 Linux 命令行工具，用于在当前进程中 " 脱离 " 某些全局命名空间，或者为新创建的进程分配独立的命名空间；
- `--fork` 这个选项告诉 unshare 在创建新的命名空间后，立即通过 fork() 系统调用创建一个子进程。新命名空间的隔离效果会应用到这个子进程，而不是当前的 shell。这样可以避免当前终端被隔离到一个新命名空间中；
- `--pid` 这个选项指定创建一个新的 **PID Namespace**（进程 ID 命名空间）。在新的 PID Namespace 中，子进程会拥有独立的进程 ID 空间，进程编号从 1 开始，且无法看到宿主系统或其他命名空间中的进程；
- `--mount-proc` 这个选项会在新的命名空间中重新挂载 /proc 文件系统。/proc 是一个虚拟文件系统，反映了当前进程的运行状态。如果不加这个选项，新 PID Namespace 中的 /proc /可能仍然显示宿主系统的进程列表；加上它后，/proc 只反映新命名空间内的进程。
- bash 这是 unshare 执行的命令，表示在新命名空间中启动一个新的 bash shell。所有后续命令都将在这个独立的 bash 中运行。

这条命令会创建一个独立的进程，并为其分配一个 `bash` shell。在该进程中运行 `ps` 命令时，你会发现只有两个进程：`bash` 和 `ps`。而在宿主系统中，你可以看到 `unshare` 进程，因为宿主系统能够查看所有全局进程，而新创建的命名空间是隔离的。

> - 这条命令没有使用 `--net` 选项，因此新进程仍然共享宿主系统的网络命名空间。如果加上（例如 `sudo unshare --fork --pid --net --mount-proc bash`），则会创建一个独立的 **Network Namespace**。
> - 在新的 Network Namespace 中，网络接口、IP 地址和路由表都与宿主隔离。你可以用 `ip link` 检查，发现默认只有一个 lo（回环接口），无法访问宿主的网络资源。这与 Docker 的网络隔离类似，Docker 会为每个容器分配独立的网络栈。

### 2.3 Cgroups

Cgroup（Control Groups）是 Linux 内核的另一项功能，用于限制和管理进程组的资源使用。它可以控制 CPU、内存、磁盘 I/O、网络带宽等资源的分配和优先级。Cgroup 将进程组织成层次化的组，并为每个组设置资源限制或配额。例如：

- 限制一个容器只能使用 1GB 内存。
- 限制容器占用 CPU 的比例不超过 50%。

Cgroup 的核心作用是确保容器之间不会相互干扰，同时优化系统资源的分配。

Cgroups 将确定进程可以使用的 CPU 和内存限制。可以使用 cgcreate 和 cgexec 等工具手动创建并使用 Cgroup。

首先，需要安装 `cgroup-tools` 包，因为它提供了操作 Cgroup 的命令行工具。在基于 Debian 的系统（如 Ubuntu）上，可以运行 `sudo apt-get install cgroup-tools`。

使用 `cgcreate` 命令可以创建一个新的 Cgroup。例如：

```go
sudo cgcreate -g memory:my-process
```

- `cgcreate`：这是创建 Cgroup 的工具，负责在系统中定义一个新的控制组。
- `-g`：指定要控制的资源子系统（subsystem）和 Cgroup 的名称。这里 memory 表示我们要限制内存资源。
- `memory:my-process`：定义了资源类型和 Cgroup 的名称，memory 是子系统，my-process 是自定义的 Cgroup 名称。

执行这条命令后，系统会在 `/sys/fs/cgroup/memory` 目录下创建一个名为 `my-process` 的子目录。这个目录包含多个文件，用于设置和管理该 Cgroup 的资源限制。例如：

- `memory.limit_in_bytes`：设置该 Cgroup 中进程可使用的最大内存量，以字节为单位。
- `memory.kmem.limit_in_bytes`：限制内核内存使用。

要为 `my-process` Cgroup 设置一个 50MB 的内存限制，可以运行：

```go
sudo echo 50000000 >  /sys/fs/cgroup/memory/my-process/memory.limit_in_bytes
```

要启动一个受限于 my-process Cgroup 的进程，可以使用 cgexec 命令：

```go
sudo cgexec -g memory:my-process bash
```

- `cgexec`：这是一个工具，用于在指定 Cgroup 中执行命令或进程。
- `-g memory:my-process`：将进程分配到 memory:my-process 这个 Cgroup，其中 memory 是资源子系统，my-process 是 Cgroup 名称。
- `bash`：在新 Cgroup 中启动一个 bash shell。

### 2.4 Cgroups with Namespace

Namespace 和 Cgroup 是实现容器的两大支柱，它们相辅相成：

1. **隔离（Namespace）**：Namespace 负责将容器的运行环境与宿主系统和其他容器隔离开来。例如，通过 PID Namespace，容器内的进程看不到宿主系统的其他进程；通过 Network Namespace，容器拥有独立的网络栈。
2. **资源控制（Cgroup）**：Cgroup 负责限制容器使用的资源，避免某个容器耗尽系统资源。例如，一个容器可能被限制在 2 个 CPU 核心和 512MB 内存内运行。
3. **协同作用**：Namespace 提供 " 边界 "，让容器内的进程感觉自己独占系统；Cgroup 提供 " 天花板 "，限制这个边界内的资源使用。二者结合，实现了既独立又可控的运行环境。

可以使用 cgroups 与 namespaces 来创建一个独立的进程，并限制其可使用的资源。

```go
sudo cgexec -g cpu,memory:my-process unshare -uinpUrf --mount-proc sh -c "/bin/hostname my-process && chroot mktemp -d /bin/sh"
```

1. **`cgexec -g cpu,memory:my-process`**
    - 使用 `cgexec` 在指定的 Cgroup 中运行后续命令。
    - **`-g cpu,memory:my-process`**：将进程放入名为 `my-process` 的 Cgroup，同时限制其 CPU 和内存资源。前提是你已通过 `cgcreate -g cpu,memory:my-process` 创建了这个 Cgroup，并设置了限制（如 `memory.limit_in_bytes`）。
2. **`unshare`**
    - 创建新的命名空间，实现进程环境的隔离。
    - **`-u`**（UTS Namespace）：隔离主机名和域名。
    - **`-i`**（IPC Namespace）：隔离进程间通信资源。
    - **`-n`**（Network Namespace）：隔离网络资源，如网络接口和 IP。
    - **`-p`**（PID Namespace）：隔离进程 ID，进程在新命名空间中从 PID 1 开始。
    - **`-U`**（User Namespace）：隔离用户和组 ID，支持权限隔离。
    - **`-r`**：将当前用户映射为新命名空间中的 root 用户（需配合 `-U`）。
    - **`-f`**（fork）：在新命名空间中 fork 一个子进程，避免影响当前 shell。
    - **`--mount-proc`**：在新 PID Namespace 中重新挂载 `/proc`，确保进程视图隔离。
3. **`sh -c "/bin/hostname my-process && chroot mktemp -d /bin/sh"`**
    - 在新环境中运行一个 shell，并执行指定的命令。
    - **`/bin/hostname my-process`**：设置新命名空间的主机名为 `my-process`（依赖 UTS Namespace）。
    - **`chroot mktemp -d /bin/sh`**：
        - **`mktemp -d`**：创建一个临时目录。
        - **`chroot`**：将进程的根文件系统切换到这个临时目录。
        - **`/bin/sh`**：在新根目录下启动一个 shell。

这条命令手动实现了一个简易容器：Namespace 提供隔离，Cgroup 提供资源控制，`chroot` 提供文件系统隔离。这正是 Docker 等容器技术的基础原理，只不过 Docker 封装得更高级（镜像管理、网络配置等）。

### 2.5 OverlayFS

OverlayFS 是 Linux 内核支持的一种联合文件系统，广泛用于 Docker。它有三个关键目录：

1. **LowerDir**：只读层，通常是容器镜像的基础文件（如 Ubuntu 的根文件系统）。
2. **UpperDir**：可写层，记录容器运行时的修改（如新建文件、修改配置）。
3. **MergedDir**：合并层，用户看到的最终文件系统，是 LowerDir 和 UpperDir 的叠加结果。

其中 LowerDir 是一个只读访问目录，采用写时复制（CoW），将修改写入 UpperDir，不影响 LowerDir。要删除时，在 UpperDir 创建 "**白出文件**"（whiteout），**标记**文件被删除，MergedDir 看不到它。

- **镜像分层**：Docker 镜像由多层组成（如 FROM ubuntu + RUN apt install nginx），每层是一个 LowerDir。
- **容器运行**：启动容器时，OverlayFS 将镜像层（LowerDir）与容器层（UpperDir）合并。容器停止后，UpperDir 可丢弃，镜像保持不变。

### 2.6 Docker

Docker 是一个开源的容器化平台，基于 Namespace 和 Cgroup 等 Linux 内核技术，简化了容器的创建、管理和部署。Docker 在这些底层技术之上，提供了用户友好的工具和标准化的工作流程：

- **Docker Engine**：核心组件，负责创建和管理容器。
- **Docker Image**：容器的只读模板，包含应用程序及其依赖。
- **Docker Container**：从镜像启动的运行实例。
- **Dockerfile**：定义如何构建镜像的脚本。

Docker 的优势在于：

- **易用性**：通过简单的命令（如 docker run）即可启动容器。
- **标准化**：镜像可以在任何支持 Docker 的环境中运行。
- **生态系统**：Docker Hub 提供了丰富的预构建镜像。

简单来说，Docker 是对 Namespace 和 Cgroup 的高级封装，降低了容器技术的使用门槛，让开发者能更专注于应用开发而非底层实现。

## 3 容器运行时

容器的创建旨在提供一个独立的环境，让程序运行时不受同一主机上其他程序的干扰。然而，仅依赖 Linux 命名空间和 Cgroup 来运行容器会面临一些挑战。

第一个挑战是**操作复杂**。要创建一个容器，需要执行多条命令：创建命名空间、设置 Cgroup、配置资源限制；删除容器时，还需手动清理命名空间和 Cgroup。

第二个挑战是**管理困难**。当运行数十个容器时，如何追踪每个容器的状态、用途及关联进程？仅靠手动命令难以高效管理。

第三个挑战是**重复劳动**。许多容器已包含所需内容并存储在 Container Registry 上，但我们无法直接下载运行，只能从头构建，费时费力。

为何不开发一个工具，简化这些工作？只需一条命令即可创建或删除容器，同时管理多个运行中的容器，清晰显示每个容器的用途和进程。此外，该工具还能从网上下载现成容器。这就是容器运行时（Container Runtime）的由来。

Container Runtime 是管理容器生命周期的工具，涵盖创建、删除、打包和共享容器：

- **低级容器运行时**：专注于创建和删除容器。
- **高级容器运行时**：负责管理容器，下载镜像，解包后交给低级运行时以创建和运行容器。

一些高级容器运行时甚至包括将容器打包成 Container Image 并将其传输到 Container Registry 的能力。

> - **Container Image** 是一个只读的、轻量级的文件包，包含了运行容器所需的所有内容：应用程序代码、运行时环境、系统库、配置文件等。它就像一个快照，捕捉了容器运行时的完整状态。在 Docker 中，你可以用 Dockerfile 定义镜像内容，然后通过 docker build 打包成镜像。
> - **Container Registry** 是一个存储和管理容器镜像的仓库，通常运行在云端或本地服务器上。它类似于代码的 GitHub，但专为容器镜像设计。

### 3.1 低级容器运行时

低级容器运行时是容器技术栈中的底层工具，主要负责创建和删除容器。它直接与 Linux 内核的特性（如 Cgroup 和 Namespace）交互，执行容器的核心操作，而不涉及高级管理功能（如镜像下载或容器编排）。它的任务是 " 启动一个隔离的、资源受限的进程 "，然后在任务完成后清理相关资源。低级容器运行时将执行以下操作：

- 使用 Cgroup（控制组）为容器分配资源限制，例如 CPU、内存等；
- 将命令行接口（CLI）或指定的程序放入 Cgroup，确保其运行受资源限制约束；
- 调用 unshare 或类似机制，创建多个命名空间（如 PID、Network、UTS 等），隔离容器的进程环境，使其与宿主和其他容器独立；
- 通过 chroot 或其他方式，为容器指定一个独立的根文件系统，通常从镜像中提取，确保容器有自己的文件环境。
- 在容器运行结束后，释放 Cgroup 和命名空间占用的资源，完成清理工作。

 `runc` 是最常见的低级容器运行时，由 Open Container Initiative（OCI）标准化，广泛应用于 Docker 等高级运行时底层。它的作用是根据配置文件（通常是 config.json）创建并运行容器。下面启动了一个名为 `runc-container` 的容器。

```go
runc run runc-container
```

### 3.2 高级容器运行时

高级容器运行时将专注于管理多个容器、传输和管理容器镜像，以及将容器镜像加载和解包到低级容器运行时。

 `containerd` 是一个常见的高级容器运行时，提供以下功能：

- 从 Container Registry 下载容器镜像。
- 管理镜像的存储与版本。
- 从镜像创建并运行容器。
- 管理容器的生命周期。

例如，我们将运行以下命令来创建一个 Redis 容器，该容器在容器注册表上有一个可用的 Redis 容器镜像。

- **下载镜像**：`sudo ctr images pull docker.io/library/redis:latest`
- **创建容器**：`sudo ctr container create docker.io/library/redis:latest redis`
- **删除容器**：`sudo ctr container delete redis`
- **列出内容**：`sudo ctr images list` 或 `sudo ctr container list`

containerd 的核心组件包括：
- **CRI**（Container Runtime Interface）是 Kubernetes 定义的标准接口，用于与容器运行时通信。
- Shim 是一个轻量进程，每个容器对应一个 containerd-shim 实例。containerd 主进程不直接管理容器，而是通过 Shim 间接控制。
- **runc**：用于直接创建和运行容器。

协作流程：
1. containerd 准备容器环境（拉镜像、挂载文件系统）。
2. 生成 OCI 规范的配置文件，交给 Shim。
3. Shim 调用 runc create 启动容器，runc 退出后 Shim 接管。

### 3.3 OCI 标准规范

1. 镜像格式规范
    - 文件系统层：基于 tar 的分层结构（如 OverlayFS 的 LowerDir）。
    - 配置文件：manifest.json（镜像元数据）和 config.json（容器运行参数）。

2. 运行时规范
    - 定义容器创建和运行的标准，如 config.json 的格式。
    - 包括 Namespace、cGroups、挂载点等配置。

### 3.4 安全容器 gVisor Vs Kata Containers

普通容器（如 runc）共享主机内核，安全性较低。安全容器通过增强隔离解决这一问题，代表是 gVisor 和 Kata Containers。

1. gVisor 用户态内核
    - gVisor 在容器和主机内核之间插入一个用户态内核（称为 **Sentry**）。
    - Sentry 拦截容器的系统调用（syscalls），用 Go 语言实现部分内核功能。
    - 用户态拦截开销较高，容器无法直接访问主机内核，安全性提高。

2. Kata Containers 轻量级 VM
    - 每个容器运行在一个轻量级虚拟机（VM）中，使用独立内核。
    - 基于 KVM 或 QEMU，结合 virtio 优化性能。
    - 内核级隔离，安全性接近传统 VM，远超普通容器。启动慢但性能高。

## 4 Docker 深度解析

Docker 是最早全面支持容器交互的工具之一，提供以下功能：

- 构建镜像（`Dockerfile` / `docker build`）。
- 管理容器镜像（`docker images`）。
- 创建、删除和管理容器（`docker run`、`docker rm`、`docker ps`）。
- 共享容器镜像（`docker push`）。
- 提供用户界面（UI），替代纯命令行操作。

Docker 的架构是一个分层设计，主要由四层组成：**Client、Daemon、Containerd 和 Runc**。它们协同工作，完成从用户命令到容器运行的整个流程。

### 4.1 四层模型

1. **Docker Client**
    - 用户接口，接收命令（如 docker run）并发送给 Daemon。
2. **Docker Daemon**（dockerd）
    - 核心服务进程，管理镜像、容器、网络和存储。
3. **Containerd**
    - 高级容器运行时，负责容器生命周期管理（如创建、启动、停止）。
4. **Runc**
    - 遵循 OCI 运行时规范，根据 config.json 直接创建和运行容器。

### 4.2 镜像构建

Docker 镜像构建是 Docker 的核心功能，Dockerfile 是构建的蓝图。

- **选择合适基础镜像**：使用轻量镜像（如 alpine 而非 ubuntu），减少体积。
- **减少层数**：每条指令生成一层，合并 RUN 命令减少层数。
- **清除缓存**：删除临时文件，避免增加镜像体积。
- **多阶段构建**：在单一 Dockerfile 中定义多个阶段，只保留最终产物。例如第一阶段需要 Go 编译器，第二阶段只需要执行可执行文件，不需要 Go 环境。
- 
