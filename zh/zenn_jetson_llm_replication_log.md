# Zenn Jetson LLM 复现日志

## 日期: 2025年05月09日

### 博客步骤/当前目标:

- 分析 Zenn 博客文章《Jetson Orin Nano で日本語LLMを動かしてみた》([https://zenn.dev/headwaters/articles/jetson-orin-nano-llm](https://zenn.dev/headwaters/articles/jetson-orin-nano-llm)) 的 "1. SSD に Jetson OS と Jetson SDK をインストール" 步骤，并决定是否需要执行。

### 执行操作:

- 对比博客描述与当前 Jetson Orin Nano 的环境信息 (`jetson_environment_info.md`)。

### 观察记录/遇到的问题:

- **博客中步骤 "1. SSD に Jetson OS と Jetson SDK をインストール" 的核心内容包括：**
    1.  物理准备：安装 NVMe SSD，（可选）使用 `nvme-cli` 初始化 SSD，设置 Jetson 进入强制恢复模式。
    2.  宿主PC准备：需要 Ubuntu 20.04 PC（因为文章目标是安装 JetPack 5.1.3）。
    3.  使用 NVIDIA SDK Manager：在宿主 PC 上通过 SDK Manager 选择 JetPack 5.1.3 并将其刷写到 Jetson Orin Nano 的 NVMe SSD。
    4.  Jetson 初始设置：完成 OS 的首次启动配置。
- **当前 Jetson Orin Nano 环境信息摘要：**
    - 操作系统已安装在 NVMe SSD (`/dev/nvme0n1p1` 挂载为 `/`)。
    - 当前运行的是 JetPack 6.0 系列 (R36, Rev 2.0) 和 Ubuntu 22.04.5 LTS。
    - 无需进行 `nvme format` 底层格式化，因为系统已稳定运行。

### 解决方案/临时对策 (决定跳过步骤1的原因):

- **主要原因：** 当前系统已经满足了 Zenn 文章第一步操作的最终目的——即在 NVMe SSD 上拥有一个可运行的 JetPack 系统。
- **具体分析：**
    1.  **OS 已在 NVMe SSD 上：** 您的系统已经从 NVMe SSD 启动并运行，无需重新进行物理安装或使用 SDK Manager 从头刷写基础系统到 SSD 的操作。
    2.  **JetPack 版本差异：** Zenn 文章的目标是安装 JetPack 5.1.3 (需要 Ubuntu 20.04 宿主机)。您当前运行的是更新的 JetPack 6.0 系列。除非有特殊需求要降级到 JetPack 5.1.3 (这将是一个完整的、涉及宿主PC和恢复模式的刷机过程)，否则没有必要执行文章中的刷机步骤。
    3.  **初始化 SSD 非必需：** 文章中提到的 `nvme-cli format` 是针对裸盘或有问题磁盘在安装全新操作系统前的操作。既然您的系统已在 NVMe SSD 上稳定运行，此操作不仅非必需，而且会擦除现有系统。

### 心得/学到的知识:

- 确认了当前 Jetson Orin Nano 的系统状态已满足 Zenn 文章第一部分的核心要求。
- 理解了 Zenn 文章第一步主要是一个完整的系统刷写流程，适用于首次配置或特定版本安装。
- 决定跳过 Zenn 文章的第一步，直接从第二步 "2. Jetson Orin Nano の RAM の最適化" 开始进行环境优化和后续操作。

---

## 日期: 2025年05月09日

### 博客步骤/当前目标:

- 根据当前 Jetson Orin Nano (JetPack 6.0, Ubuntu 22.04, NVMe SSD 为根目录) 环境，制定并记录 Zenn 文章第二步 "2. Jetson Orin Nano の RAM の最適化" 的具体执行计划和注意事项。

### 针对当前环境的RAM优化执行计划:

**前提:** 目标是为 LLM 运行释放最大可用 RAM，并将当前 3.7Gi 的 `zram` 替换为约 16GB 基于 NVMe SSD 的 swap 文件。

#### 步骤 2.1: 禁用图形用户界面 (GUI)

- **操作规划:** 执行以下命令将系统默认启动目标更改为多用户命令行界面。
  ```bash
  sudo systemctl set-default multi-user.target
  ```
- **预期影响:** 重启后系统将进入命令行模式，不再加载桌面环境，预计释放约 800MB RAM。
- **如何恢复:** 如果之后想恢复桌面环境，可以使用 `sudo systemctl set-default graphical.target` 并重启。
- **记录 (2025年05月09日):**
    - **已执行命令:**
      ```bash
      dtc@ubuntu:~/wrokspace/LLM$ sudo systemctl set-default multi-user.target
      Removed /etc/systemd/system/default.target.
      Created symlink /etc/systemd/system/default.target → /lib/systemd/system/multi-user.target.
      ```
    - 此更改将在下次重启后生效。

#### 步骤 2.2: 禁用 nvargus-daemon 服务

- **操作规划:** 如果不使用摄像头功能，执行以下命令禁用相机服务。
  ```bash
  sudo systemctl disable nvargus-daemon.service
  ```
- **预期影响:** 释放少量系统资源。
- **如何恢复:** 如果之后需要使用相机服务，可以使用 `sudo systemctl enable nvargus-daemon.service` 并重启 (或 `sudo systemctl start nvargus-daemon.service` 立即启动)。
- **记录 (2025年05月09日):**
    - **已执行命令:**
      ```bash
      dtc@ubuntu:~/wrokspace/LLM$ sudo systemctl disable nvargus-daemon.service
      Removed /etc/systemd/system/multi-user.target.wants/nvargus-daemon.service.
      ```
    - 此更改将在下次重启后生效。

#### 步骤 2.3: Swap 空间配置 (核心调整部分)

**a. 调查并尝试禁用当前 `zram` 配置:**

- **操作规划1 (调查 `nvzramconfig.service`):**
  ```bash
  systemctl status nvzramconfig.service
  ```
- **操作规划2 (调查活动 zram 设备):**
  ```bash
  swapon -s
  lsblk | grep zram
  ```
- **记录 (2025年05月09日 - zRAM 调查结果):**
    - **执行 `systemctl status nvzramconfig.service` 输出:**
      ```
      ○ nvzramconfig.service - ZRAM configuration
           Loaded: loaded (/etc/systemd/system/nvzramconfig.service; enabled; vendor preset: enabled)
           Active: inactive (dead) since Thu 1970-01-01 09:00:59 JST; 55 years 4 months ago
         Main PID: 951 (code=exited, status=0/SUCCESS)
              CPU: 97ms

       1月 01 09:00:58 ubuntu nvzramconfig.sh[1122]: ラベルはありません, UUID=48746f39-f11b-4e67-a258-8dffff68c45f
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1131]: スワップ空間バージョン 1 を設定します。サイズ = 635 MiB (665870336 バイト)
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1131]: ラベルはありません, UUID=62ddb39b-3daa-49ec-8fa3-77745b4bab6c
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1143]: スワップ空間バージョン 1 を設定します。サイズ = 635 MiB (665870336 バイト)
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1143]: ラベルはありません, UUID=29f6745f-c1cc-4e19-a5b1-1dbc9aed4eed
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1160]: スワップ空間バージョン 1 を設定します。サイズ = 635 MiB (665870336 バイト)
       1月 01 09:00:58 ubuntu nvzramconfig.sh[1160]: ラベルはありません, UUID=8039c050-414c-4579-9efa-32316dcf8330
      ```
    - **执行 `swapon -s` 及 `lsblk | grep zram` (综合输出观察):**
      ```
      Filename                                Type            Size            Used            Priority
      /dev/zram0                              partition       650264          0               5
      /dev/zram1                              partition       650264          0               5
      /dev/zram2                              partition       650264          0               5
      /dev/zram3                              partition       650264          0               5
      /dev/zram4                              partition       650264          0               5
      /dev/zram5                              partition       650264          0               5

      zram0        252:0    0   635M  0 disk [SWAP]
      zram1        252:1    0   635M  0 disk [SWAP]
      zram2        252:2    0   635M  0 disk [SWAP]
      zram3        252:3    0   635M  0 disk [SWAP]
      zram4        252:4    0   635M  0 disk [SWAP]
      zram5        252:5    0   635M  0 disk [SWAP]
      ```
    - **分析:**
        - `nvzramconfig.service` 服务是 `enabled` 状态，并在系统启动早期配置了6个约635MB的zram设备作为swap。
        - **结论：** `nvzramconfig.service` 是当前系统上负责创建和配置这些 `zram` Swap的服务。

- **操作规划3 (禁用 `nvzramconfig` 并临时移除 zRAM):**
  ```bash
  sudo systemctl stop nvzramconfig.service
  sudo systemctl disable nvzramconfig.service
  # 临时移除zram设备进行验证 (用户选择选项A)
  sudo swapoff /dev/zram0
  sudo swapoff /dev/zram1
  sudo swapoff /dev/zram2
  sudo swapoff /dev/zram3
  sudo swapoff /dev/zram4
  sudo swapoff /dev/zram5
  # 验证命令
  swapon -s
  lsblk | grep zram
  free -h
  ```
- **预期影响:** 停止并禁止现有 zram swap 的自动加载，并临时移除当前活动的zram swap。
- **记录 (2025年05月09日 - 执行禁用与验证zRAM):**
    - **执行 `sudo systemctl stop nvzramconfig.service` 和 `sudo systemctl disable nvzramconfig.service`:** (用户报告无错误输出)
    - **执行 `sudo swapoff /dev/zramX` (X从0到5):** (用户报告无错误)
    - **验证命令输出:**
      ```bash
      # swapon -s (从free -h推断为空)

      # lsblk | grep zram
      zram0        252:0    0   635M  0 disk 
      zram1        252:1    0   635M  0 disk 
      zram2        252:2    0   635M  0 disk 
      zram3        252:3    0   635M  0 disk 
      zram4        252:4    0   635M  0 disk 
      zram5        252:5    0   635M  0 disk 

      # free -h
                         total        used        free      shared  buff/cache   available
      Mem:           7.4Gi       2.3Gi       2.7Gi        38Mi       2.5Gi       4.9Gi
      Swap:             0B          0B          0B
      ```
    - **执行结果分析:**
        - `nvzramconfig.service` 已成功停止和禁用。
        - 所有 `zram` 设备已成功通过 `swapoff` 从活动 Swap 中移除。
        - `lsblk` 输出显示 `zram` 块设备节点依然存在，但不再作为 `[SWAP]` 使用。
        - `free -h` 输出明确显示当前系统 Swap 为 0B。
        - **结论: 成功清除了现有的 zram Swap，可以继续创建新的基于 SSD 的 Swap 文件。**

**b. 创建新的 Swap 文件 (在 NVMe SSD 上):**

- **决策:** 确定 Swap 文件路径。鉴于根目录 `/` 已在 NVMe SSD，建议路径为 `/16GB_swapfile`。
- **操作规划1 (创建文件):**
  ```bash
  sudo fallocate -l 16G /16GB_swapfile
  ```
- **操作规划2 (设置权限，可选但推荐):**
  ```bash
  sudo chmod 600 /16GB_swapfile
  ```
- **预期影响:** 在 NVMe SSD 上预分配一个 16GB 的文件作为 swap 空间。
- **记录:** (将在实际操作后记录执行的命令和选择的路径)

**c. 格式化并激活 Swap 文件:**

- **操作规划1 (格式化):**
  ```bash
  sudo mkswap /16GB_swapfile
  ```
- **操作规划2 (激活):**
  ```bash
  sudo swapon /16GB_swapfile
  ```
- **预期影响:** 新的 16GB swap 空间被系统识别并立即启用。
- **记录:** (将在实际操作后记录执行的命令、命令输出)

**d. 使 Swap 文件永久生效 (修改 `/etc/fstab`):**

- **操作规划 (推荐的安全方法):**
  ```bash
  echo '/16GB_swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```
- **验证规划 (可选):** `cat /etc/fstab` 查看是否已添加。
- **预期影响:** 系统重启后会自动挂载并使用 `/16GB_swapfile` 作为 swap。
- **记录:** (将在实际操作后记录执行的命令，以及 `/etc/fstab` 添加后的内容)

#### 步骤 2.4: 重启系统使初步优化生效

- **操作规划:**
  ```bash
  sudo reboot
  ```
- **预期影响:** 应用所有之前的更改 (GUI禁用, nvargus禁用, 新Swap配置)。
- **记录:** (将在实际操作后记录重启后，通过 `free -h` 和 `swapon -s` 验证内存和新 swap 状态)

#### 步骤 2.5: [可选] 进一步禁用不必要的服务

**a. 列出当前启用的服务:**

- **操作规划:**
  ```bash
  systemctl list-unit-files --type=service --state=enabled
  ```
- **预期影响:** 输出当前系统所有开机自启的服务列表。
- **记录:** (将在实际操作后记录命令输出的服务列表，用于后续决策)

**b. 根据需求选择性禁用服务:**

- **参考 Zenn 文章列表，并结合自身需求判断。** 特别注意 `wpa_supplicant.service` (若使用Wi-Fi则保留) 和 `snapd.service` (若使用Snap包则保留或谨慎处理)。
- **示例 (仅当确认不需要时才执行):**
  ```bash
  # sudo systemctl disable bluetooth.service
  # sudo systemctl disable snap.cups.cupsd.service
  # ... (其他来自Zenn文章列表中的服务)
  ```
- **预期影响:** 进一步释放少量内存和系统资源。
- **记录:** (将在实际操作后记录最终决定禁用哪些服务，执行的命令)

**c. [可选] 再次重启系统:**

- **操作规划 (如果执行了步骤2.5.b):**
  ```bash
  sudo reboot
  ```
- **预期影响:** 使可选服务禁用生效。
- **记录:** (将在实际操作后记录重启后，再次通过 `free -h` 验证内存使用情况)

### 接下来关注的重点:

- 记录每一步操作的详细输出和遇到的任何问题。

---

## 术语解释

### Jetson Orin Nano

- **是什么：** NVIDIA 公司推出的一款小型、低功耗、高性能的边缘计算设备（或称为模块）。它属于 Jetson Orin 系列的一员，专为在设备端直接运行人工智能 (AI) 和机器学习 (ML) 应用而设计。通常，您会购买一个包含 Jetson Orin Nano 模块和参考载板的开发者套件 (Developer Kit) 来进行开发。
- **用来干什么的：** 用于构建和部署 AI 驱动的嵌入式系统，例如智能机器人、自主无人机、智能摄像头、便携式医疗设备等。它可以在本地处理复杂的 AI 推理任务，而无需完全依赖云端计算。

### JetPack 6.0

- **是什么：** NVIDIA 为其 Jetson 系列边缘 AI 平台提供的综合性软件开发套件 (SDK)。JetPack 6.0 是指该 SDK 的一个特定版本。
- **用来干什么的：** 它包含了运行 Jetson 设备并开发 AI 应用所需的一切软件组件，主要包括：
    - **Jetson Linux (L4T - Linux for Tegra)：** 专门为 Jetson 优化的 Linux 操作系统。
    - **CUDA Toolkit：** NVIDIA 的并行计算平台和编程模型，允许开发者利用 GPU 进行通用计算。
    - **cuDNN：** NVIDIA CUDA 深度神经网络库，为深度学习框架提供 GPU 加速。
    - **TensorRT：** NVIDIA 的高性能深度学习推理优化器和运行时库，可以加速深度学习模型在 NVIDIA GPU 上的推理性能。
    - **OpenCV, VisionWorks 等视觉和图像处理库。**
    - **驱动程序和其他开发工具。**

    JetPack 简化了 Jetson 平台的软件安装和管理过程。

### SSD (Solid State Drive - 固态硬盘)

- **是什么：** 一种使用固态电子存储芯片阵列制成的硬盘。与传统的机械硬盘 (HDD) 不同，它没有移动部件。
- **用来干什么的：** 用作计算机的存储设备，用于安装操作系统、应用程序和存储用户数据。相比机械硬盘，SSD 具有更快的读写速度、更低的延迟、更好的抗震性、更低的噪音和功耗。在 Jetson 设备上使用 SSD (尤其是 NVMe SSD) 可以显著提升系统响应速度和数据加载速度。

### SDK (Software Development Kit - 软件开发工具包)

- **是什么：** 通常是一组软件开发工具的集合，允许开发者为特定的软件包、软件框架、硬件平台或操作系统创建应用程序。
- **用来干什么的：** SDK 提供了必要的库、API (应用程序编程接口)、代码示例、文档、调试工具等，帮助开发者更高效地构建、测试和部署软件。例如，JetPack 就是 Jetson 平台的 SDK。

### NVMe SSD (Non-Volatile Memory Express SSD - 非易失性内存主机控制器接口规范固态硬盘)

- **是什么：** 一种使用 NVMe 接口标准的高性能固态硬盘。NVMe 是一种专为 SSD 设计的通信接口和驱动程序，旨在充分利用 PCIe (Peripheral Component Interconnect Express) 总线的高带宽。
- **用来干什么的：** 与传统的 SATA 接口 SSD 相比，NVMe SSD 提供更高的吞吐量和更低的延迟，从而带来更快的启动速度、应用程序加载时间和文件传输速度。在 Jetson Orin Nano 这样的设备上使用 NVMe SSD，可以最大程度地发挥其处理性能，特别是在需要快速读写大量数据的 AI 应用中。

--- 