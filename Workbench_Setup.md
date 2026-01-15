
# Workstation Setup Guide

本仓库用于记录 **新工作站（Ubuntu）从裸系统到可用于科研与开发的完整配置流程**，重点覆盖：

* NVIDIA 独立显卡驱动（官方 `.run` 安装方式）
* 网络代理
* 常用软件与基础工具
* Git / SSH 使用与安全注意事项
* Python / 深度学习基础环境（Anaconda + PyTorch）

当前文档以 **Ubuntu 24.04 LTS + NVIDIA GPU** 为目标环境，后续可根据需要扩展至其他发行版或硬件平台。

---

## 1. 系统与硬件环境

* **操作系统**：Ubuntu 24.04 LTS
* **显卡**：NVIDIA RTX 5090 独立显卡
* **NVIDIA 驱动版本**：580.119.02
* **推荐内核**：Linux Generic Kernel
* **不推荐**：HWE / OEM 内核（可能导致 DKMS 编译失败）

---

## 2. NVIDIA 独立显卡驱动安装

> ⚠️ 本节为**高风险配置**。
> 若操作不当，可能导致图形界面无法启动。
> **请严格按照步骤顺序执行，不要跳步。**

---

### 2.1 准备系统

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential libssl-dev
```

---

### 2.2 切换到通用内核（关键）

```bash
sudo apt install --install-recommends -y linux-generic linux-headers-generic
dpkg -l | grep linux-image
sudo reboot
```

> 确认重启后使用的是 `generic` 内核。

---

### 2.3 完全清理旧 NVIDIA / CUDA 驱动

```bash
# 停止显示管理器（按桌面环境选择）
sudo systemctl stop gdm3       # GNOME
# sudo systemctl stop lightdm # LightDM
# sudo systemctl stop sddm    # KDE

# 卸载驱动
sudo apt purge -y *nvidia* *cuda*
sudo apt autoremove --purge -y

# 清理残留文件
sudo rm -rf /usr/src/nvidia*
sudo rm -rf /var/lib/dkms/nvidia*
sudo rm -rf /etc/modprobe.d/nvidia*
```

---

### 2.4 下载官方 NVIDIA 驱动

```bash
mkdir -p ~/nvidia-driver
cd ~/nvidia-driver

wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.119.02/NVIDIA-Linux-x86_64-580.119.02.run
chmod +x NVIDIA-Linux-x86_64-580.119.02.run
```

---

### 2.5 运行安装程序

```bash
# 基本安装
sudo ./NVIDIA-Linux-x86_64-580.119.02.run
```

或使用推荐的非交互方式：

```bash
sudo ./NVIDIA-Linux-x86_64-580.119.02.run \
  --dkms \
  --no-questions \
  --accept-license \
  --no-opengl-files \
  --disable-nouveau
```

---

### 2.6 恢复图形界面并重启

```bash
sudo systemctl set-default graphical.target
sudo reboot
```

---

### 2.7 安装后验证

```bash
nvidia-smi
lsmod | grep nvidia
nvidia-settings
nvcc --version    # 若已安装 CUDA
```

---

## 3. 常见 NVIDIA 驱动错误与解决方案

### 3.1 `Failed to initialize NVML: Driver/library version mismatch`

**原因**：用户态驱动与内核模块不匹配。

```bash
sudo apt purge -y *nvidia*
sudo apt autoremove --purge -y
sudo reboot
# 按上述流程重新安装
```

**若上述方法不起作用**：可能是内核头文件不匹配，例如有 linux-headers-6.8.0-90-generic，但运行的是 6.14.0-1019-oem 内核；或API不匹配：NVIDIA驱动无法与当前内核通信。建议进一步排查工作日志，若内核不匹配，可尝试如下方案修复。

**方案1：安装匹配的OEM内核头文件（首选）**
```bash
# 安装与当前内核匹配的头文件
sudo apt install linux-headers-6.14.0-1019-oem linux-oem-24.04c

# 重新构建NVIDIA模块
sudo dkms remove nvidia/580.95.05 --all
sudo dkms add -m nvidia -v 580.95.05
sudo dkms build -m nvidia -v 580.95.05
sudo dkms install -m nvidia -v 580.95.05
```

**方案2：切换到匹配的内核版本**
```bash
# 查看可用的6.8.0内核
apt list linux-image-6.8.0* linux-headers-6.8.0*

# 安装6.8.0内核
sudo apt install linux-image-6.8.0-90-generic linux-headers-6.8.0-90-generic

# 更新GRUB并重启
sudo update-grub
sudo reboot

# 重启后选择6.8.0内核
```

**方案3：完全切换到通用内核系列**
```bash
# 安装完整的通用内核包
sudo apt install --install-recommends linux-generic linux-headers-generic

# 移除OEM内核（可选）
sudo apt purge linux-image-6.14.0-1019-oem linux-headers-6.14.0-1019-oem

# 更新GRUB并重启
sudo update-grub
sudo reboot
```


**推荐的操作顺序：**
1. **首先尝试方案1**（安装匹配的OEM头文件）
2. **如果方案1失败，使用方案3**（切换到通用内核）
3. **重启后验证**：
   ```bash
   nvidia-smi
   lsmod | grep nvidia
   ```


---

### 3.2 `Unable to load the kernel module nvidia.ko`

**原因**：内核头文件缺失或 DKMS 编译失败。

```bash
# 解决方案
sudo apt install -y linux-headers-$(uname -r)
sudo dkms remove nvidia/580.119.02 --all
sudo dkms install nvidia/580.119.02
sudo update-initramfs -u
```

---

### 3.3 ERROR: The Nouveau kernel driver is currently in use

**原因**：Nouveau 驱动未禁用

```bash
# 解决方案
sudo tee /etc/modprobe.d/blacklist-nouveau.conf << EOF
blacklist nouveau
options nouveau modeset=0
EOF

sudo update-initramfs -u
sudo reboot
```

---

### 3.4 DKMS build failed
**原因**：DKMS构建错误
```bash
sudo cat /var/lib/dkms/nvidia/580.119.02/build/make.log
sudo dkms remove nvidia/580.119.02 --all
sudo dkms build nvidia/580.119.02 --no-sign
```

---

## 4. 网络代理配置

* 使用的代理工具是Clash。配置Clash for linux。（通常根据机场不同，配置也有所区别，具体可参见自己的机场的教程。）
* 图形界面使用 **Clash Verge（Linux GUI）**，与 Windows 下 Clash 图形界面在布局与配置路径上略有不同。详情参见Clash Verge官方仓库，https://github.com/clash-verge-rev/clash-verge-rev。
* 一般建议启用 **系统代理**
* 配置系统代理（通常根据机场不同，配置也有所区别，具体可参见自己的机场的教程。）

* 是否启用 TUN 模式视网络环境而定


---

## 5. 常用软件下载与 Linux 安装方式说明

本节说明 **Linux（以 Ubuntu / Debian 系为例）下常见的软件安装方式**，以及在手动安装第三方软件时需要注意的事项。

---

### 5.1 不同 Linux 发行版的软件包格式

Linux 不同发行版使用不同的软件包管理体系：

* **Debian系（包括Debian, Ubuntu等）**

  * 常用安装包格式：`.deb`
  * 包管理工具：`apt`

* **Red Hat / CentOS / Rocky Linux 系**

  * 常用安装包格式：`.rpm`
  * 包管理工具：`dnf` / `yum`

> 本文档默认以 **Debian 系** 为目标环境。

---

### 5.2 `.deb` 安装包（推荐方式）

对于 Ubuntu 及其他 Debian 系发行版，**优先下载 `.deb` 后缀的软件包**，常用软件通常都推出了支持Debian系的安装包，浏览器搜索进入其官网下载即可。

#### 安装方式

* 双击 `.deb` 文件，Ubuntu 会自动调用系统安装器完成安装
* 或在终端中执行：

```bash
sudo apt install ./package-name.deb
```

#### 安装位置说明

* 通过 `.deb` 安装的软件，其程序文件由系统统一管理
* 用户侧配置文件通常位于用户主目录下
* 若软件以 Snap 形式提供，其相关数据会位于：

```text
~/snap/
```

---

### 5.3 压缩包形式的软件（`.tar`, `.tar.gz` 等）

部分软件仅提供 **压缩包形式**，常见后缀包括：

* `.tar`
* `.tar.gz`
* `.tgz`

此类软件 **不会被系统自动管理**，通常需要：

1. 手动解压
2. 通过命令行运行或安装

#### 常见解压命令

```bash
gunzip file.gz
tar -xvf file.tar
tar -zxvf file.tar.gz
```

---

### 5.4 安装目录的重要说明（非常关键）

对于通过 **解压方式手动安装的软件**，需注意以下原则：

* **严禁** 将软件安装目录放入 `~/snap/` 目录
* `snap/` 目录为 Snap 应用专用目录
* 用户对该目录的写入通常需要 `sudo` 权限
* Ubuntu 系统也 **不允许普通用户直接向 snap 目录写入文件**

#### 推荐做法

* 将解压后的软件放置在：

  * 用户主目录下的独立文件夹（如 `~/apps/`）
  * 或 `/opt/`（需管理员权限）

---

### 5.5 小结

* Ubuntu / Debian 系优先使用 `.deb` 安装包
* 压缩包形式的软件需手动解压与管理
* 不要将手动安装的软件放入 `snap/` 目录
* 能被系统包管理器管理的软件，优先交由系统管理


### 5.6 常见软件目录

* WPS Office
* Google Chrome
* Wechat
* Vscode

---

## 6. 常用工具配置

### 6.1 SSH Server

```bash
sudo apt install -y openssh-server
sudo systemctl start ssh
sudo systemctl enable ssh
```

检查运行状态：

```bash
ps aux | grep sshd
```


---

## 6. 常用工具配置：SSH 与 Git

本节介绍 **SSH Server 的安装与基本使用**，以及 **Git 的安装、配置与安全使用规范**。
该部分内容在后续 **远程登录、虚拟机访问、代码协作** 等场景中均为基础配置。

---

### 6.1 SSH 配置

为便于后续可能的虚拟机配置，以及通过 SSH 方式访问本机或虚拟机，需要安装并启用 SSH Server。

---

#### 6.1.1 安装 SSH Server

```bash
sudo apt install -y openssh-server
```

---

#### 6.1.2 启动并设置开机自启

```bash
sudo systemctl start ssh
sudo systemctl enable ssh
```

---

#### 6.1.3 检查 SSH Server 是否正在运行

一种简单的检查方式是查看是否存在正在运行的 `sshd` 进程：

```bash
ps aux | grep sshd
```

若有输出（且不只是 grep 本身），说明 SSH Server 正在运行。

---

### 6.2 Git 使用与配置

Git 是常用的版本控制工具，支持：

* 本地代码的版本管理
* 多人协作与代码合并
* 代码历史追踪与回退

清华有自己的 Git 平台（GitLab Tsinghua），此外 **GitHub** 是最常用的公共代码托管平台。

---

#### 6.2.1 安全提示（非常重要）

对于 **实验室公用电脑或共享工作站**，Git 使用需特别注意安全风险：

* Git 配置可能泄露个人身份信息
* SSH 私钥一旦泄露，风险较高

**推荐原则：**

* 尽量避免在公用机器上操作个人 GitHub 仓库
* 优先使用 **HTTPS + Token** 方式访问仓库
* 避免长期存放个人 SSH 私钥

若必须使用 SSH：

* 生成 **带密码的 SSH Key**
* 设置严格的文件权限
* 离开时务必锁屏 / 注销，并清空 SSH Agent
* 定期检查 GitHub / GitLab 账户中的 SSH Key 列表，及时撤销不用的密钥

---

#### 6.2.2 Git 安装（Debian / Ubuntu 系）

```bash
sudo apt update
sudo apt install -y git
```

验证安装是否成功：

```bash
git --version
```

示例输出：

```text
git version 2.x.x
```

---

#### 6.2.3 Git 基本身份配置

安装完成后，需要配置 Git 的用户身份信息：

```bash
git config --global user.name "Your Name"
git config --global user.email "name@example.com"
```

> 建议使用与 GitHub / GitLab 注册时一致的邮箱地址，便于平台识别提交者身份。

---

### 6.3 SSH Key 生成与管理

#### 6.3.1 安装 SSH 客户端

```bash
sudo apt install -y openssh-client
```

---

#### 6.3.2 生成 SSH Key

```bash
ssh-keygen
```

过程中会提示：

```text
Enter passphrase (empty for no passphrase):
```

* 若不想设置访问密码，可直接回车
* 设置密码可提高安全性（推荐）

生成的文件：

* `~/.ssh/id_rsa`：私钥（必须保密）
* `~/.ssh/id_rsa.pub`：公钥（可公开）

---

#### 6.3.3 重新生成密钥的说明

若曾设置过密码但遗忘，可删除旧密钥并重新生成：

```bash
rm ~/.ssh/id_rsa
rm ~/.ssh/id_rsa.pub
```

随后重新执行 `ssh-keygen`。

> 注意：重新生成密钥后，需要在 Git 平台上重新配置 SSH Key。

---

### 6.4 Git.Tsinghua 配置

#### 6.4.1 添加 SSH Key

1. 登录 [https://git.tsinghua.edu.cn/](https://git.tsinghua.edu.cn/)
2. 点击右上角头像 → **Preferences**
3. 左侧选择 **SSH Keys**
4. 在终端中执行：

```bash
cat ~/.ssh/id_rsa.pub
```

5. 复制以 `ssh-rsa`（或 `ssh-ed25519`）开头的整行内容
6. 粘贴到网页中的 Key 输入框
7. Title 可随意填写
8. 点击 **Add Key**

---

#### 6.4.2 测试连接

```bash
ssh -T git@git.tsinghua.edu.cn
```

首次连接若提示 `yes/no`，输入 `yes` 并回车。
成功后会看到类似提示：

```text
Welcome to GitLab, @username!
```

---

#### 6.4.3 克隆仓库

1. 在 Git.Tsinghua 页面中打开目标仓库
2. 点击 **Clone**
3. 选择 **Clone with SSH**
4. 复制 `git@git.tsinghua.edu.cn:` 开头的地址
5. 在终端中执行：

```bash
git clone <repository-url>
```

若提示目录已存在，说明仓库已克隆，可直接进入目录：

```bash
cd repository-name
```

---

### 6.5 GitHub 配置

#### 6.5.1 添加 SSH Key

1. 登录 GitHub
2. 点击右上角头像 → **Settings**
3. 进入 **SSH and GPG keys**
4. 点击 **New SSH Key**
5. 在终端中执行：

```bash
cat ~/.ssh/id_rsa.pub
```

6. 复制公钥内容并粘贴
7. 点击 **Add SSH Key**

---

#### 6.5.2 测试连接

```bash
ssh -T git@github.com
```

成功后会显示：

```text
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

---

#### 6.5.3 克隆 GitHub 仓库

1. 打开 GitHub 仓库页面
2. 点击 **Code**
3. 选择 **Use SSH**
4. 复制 `git@github.com:` 开头的地址
5. 在终端中执行：

```bash
git clone <repository-url>
```

---


---

## 6.6 实验室公用机器下的 Git / SSH 使用建议

在实验室公用服务器或工作站上使用 Git 与 SSH，应优先考虑账号与密钥安全，避免个人凭据的长期暴露。

### 6.6.1 基本原则

* 非必要不配置 SSH Key
* 不复用个人设备上的密钥
* 所有认证方式应可随时撤销
* 不同用户使用各自的系统账号

---

### 6.6.2 Git 使用建议

**优先使用 HTTPS + Token** 方式访问 Git 仓库，避免在公用机器上长期保存 SSH 私钥。

* Token 可在 GitHub / GitLab 网页端随时吊销
* 权限与有效期可控，安全性更高

不建议在公用机器上进行全局身份配置：

```bash
git config --global user.name
git config --global user.email
```

如有需要，建议使用仓库级配置，或在使用结束后及时清除。

---

### 6.6.3 SSH 使用建议

仅在确有需要时（如长期服务器、自动化任务）才配置 SSH Key。

如必须使用 SSH Key，应遵循：

* 为该机器单独生成密钥
* 建议设置密钥密码（passphrase）
* 在 Git 平台中清晰标注用途
* 使用完成后及时删除本地密钥，并在网页端撤销

---

### 6.6.4 使用结束后的检查

* 清理本地 SSH Key 与 Agent
* 注销或锁屏当前用户会话
* 不在脚本或配置文件中明文保存 Token 或密钥

---

### 6.6.5 小结

在实验室公用环境中：

* **HTTPS + Token 优先**
* **SSH Key 谨慎使用**
* **不长期保留个人凭据**

遵循以上原则，可有效降低账号与密钥泄露风险。

### 6.7 本节小结

* SSH 是远程访问与自动化的重要基础工具
* Git 是科研与开发中不可或缺的版本控制系统
* 在公用或实验室机器上，应优先考虑安全性
* SSH Key 需妥善管理，避免长期暴露风险

---

## 7. VS Code 扩展配置

Visual Studio Code（VS Code）是 Linux 环境下常用的轻量级开发编辑器。通过合理配置扩展，可以显著提升代码编写、调试与文档编辑效率。

### 7.1 推荐安装的扩展

以下扩展适用于大多数学习与科研场景，建议统一安装。

#### （1）中文语言支持

* **Chinese (Simplified) Language Pack for Visual Studio Code**
  提供 VS Code 界面的简体中文支持，便于初学者使用。

安装后如未自动生效，可在命令面板中切换显示语言。

---

#### （2）Python

* **Python（Microsoft）**

功能包括：

* Python 语法高亮与代码补全
* 调试支持
* 虚拟环境与解释器管理

该扩展是后续使用 Jupyter、调试脚本与运行科研代码的基础。

---

#### （3）Jupyter

* **Jupyter（Microsoft）**

支持：

* 在 VS Code 中直接打开和运行 `.ipynb` 文件
* 交互式执行代码单元
* 与 Python 扩展深度集成

适用于数据分析、实验记录与算法原型验证等场景。

---

### 7.2 可选扩展（按需安装）

根据具体使用需求，可选择性安装以下扩展。

#### （1）LaTeX 支持

* **LaTeX Workshop**

适用于论文与技术文档写作，主要功能包括：

* LaTeX 语法高亮
* 一键编译与 PDF 预览
* 正向 / 反向搜索

使用前需确保系统中已安装 LaTeX 发行版（如 TeX Live）。

---

#### （2）Markdown 支持

* **Markdown All in One**
* **Markdown Preview Enhanced（可选）**

适用于 README、实验记录与说明文档编写，提供：

* Markdown 语法增强
* 目录自动生成
* 实时预览与导出功能

---


合理配置 VS Code 扩展，可以在 Linux 环境下形成高效、统一的开发与写作工作流。

---

## 8. Python / 深度学习环境配置

本节主要介绍 Linux 环境下常见的 Python 与深度学习开发环境配置方案，适用于科研与工程开发场景。

### 8.1 Anaconda

Anaconda 是常用的 Python 虚拟环境与包管理工具，**强烈推荐用于科研环境**。

主要特点：

* 支持多 Python 版本与多虚拟环境并存（这是很有必要的，例如，当前有的老旧库只支持python3.7，而很多新库的python3.12支持更好，那么就需要分别建立python3.7和python3.12的虚拟环境。）
* 依赖管理相对稳定，适合复杂科研项目

参考配置教程：
[https://blog.csdn.net/2320_76211938/article/details/155226164](https://blog.csdn.net/2302_76211938/article/details/155226164)

常用命令示例：

```bash
conda create -n myenv python=3.10
conda activate myenv
conda install <libname>
conda uninstall <libname>
conda list
```

建议：

* 对于依赖库明显有差别的项目，建立多个虚拟环境进行管理。
* 避免在 base 环境中直接开发

---

### 8.2 PyTorch（GPU 版本）

在具备 NVIDIA GPU 的情况下，建议安装 **GPU 版本 PyTorch** 以加速模型训练。

注意事项：

* PyTorch 所需 CUDA 版本需与系统 CUDA 驱动匹配
* 本地开发场景下 **无需使用 Docker 或 NVIDIA Container Toolkit**

若系统已安装 CUDA，可在 `~/.bashrc` 中配置环境变量（示例）：

```bash
nano ~/.bashrc

export PATH=/usr/local/cuda-12.5/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.5/lib64:$LD_LIBRARY_PATH
```

配置完成后需执行：

```bash
source ~/.bashrc
```

参考教程：
[https://blog.csdn.net/little_carter/article/details/135934842](https://blog.csdn.net/little_carter/article/details/135934842)
注意：
1. 不要安装该教程提到的CuDNN（加速器）。
2. 其介绍的是windows下的图形界面版本，与linux安装细节略有出入，请注意这一点。
3. 注意教程安装的是cuda12.x，应该按自己的cuda版本修改环境配置：

      ```bash
      nano ~/.bashrc
      # NVIDIA CUDA Toolkit Environment Variables
      export PATH=/usr/local/cuda-12.5/bin${PATH:+:${PATH}} # 改成自己的cuda版本
      export LD_LIBRARY_PATH=/usr/local/cuda-12.5/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}} #改成自己的cuda版本 
      ```

---

### 8.3 常用 Python 库与安装方式说明（重要）

在 Linux 系统中，**同一软件可能同时存在多种安装方式**。对于 Python 科研环境，理解 `conda`、`pip` 与 `apt` 的区别非常重要，否则容易导致环境混乱、库版本冲突，甚至系统 Python 损坏。

---

#### 8.3.1 三种常见安装方式的区别

##### （1）`conda install`（**首选方式**）

`conda` 是 **环境级包管理器**，同时管理：

* Python 解释器版本
* Python 库
* 非 Python 依赖（如 `libstdc++`、`cudatoolkit`）

特点：

* 依赖解析较完整，适合科研环境
* 安装的库 **仅作用于当前 Conda 虚拟环境**
* 不影响系统自带 Python

适用场景：

* 数值计算、科学计算、深度学习相关库
* 需要稳定、可复现环境的实验代码

示例：

```bash
conda install numpy scipy matplotlib pandas
```

---

##### （2）`pip install`（**补充方式**）

`pip` 是 **Python 生态内部的包管理工具**，只管理 Python 库本身。

特点：

* 包数量最多，更新速度快
* 不管理系统级依赖
* 容易与已有库产生版本冲突（尤其在复杂环境中）

适用场景：

* Conda 仓库中不存在的库
* 某些更新较快的研究型代码库

注意事项：

* **必须在已激活的 Conda 环境中使用 pip**
* 不要在系统 Python 下随意使用 `pip install`

示例：

```bash
pip install <package_name>
```

---

##### （3）`apt install`（**不推荐用于科研 Python 环境**）

`apt` 是 **系统级包管理器**，用于管理操作系统软件。

特点：

* 安装位置为系统目录（如 `/usr/lib`）
* 通常需要 `sudo` 权限
* 库版本较旧，更新节奏慢

问题：

* 容易与 Conda / Pip 安装的库混用
* 可能破坏系统 Python 环境
* 不利于实验复现与环境迁移

适用场景（有限）：

* 安装系统工具（如 `git`、`cmake`、`wget`）
* **不建议** 用于安装 Python 科研库

示例（不推荐）：

```bash
sudo apt install python3-numpy
```

---

#### 8.3.2 推荐的安装优先级（重要）

在科研与实验环境中，**推荐始终遵循以下顺序**：

1. **优先使用 `conda install`**
2. Conda 无对应包时，再使用 `pip install`
3. **避免使用 `apt install` 安装 Python 库**

该顺序有助于：

* 降低依赖冲突风险
* 保持环境结构清晰
* 提高实验可复现性

---

#### 8.3.3 常用基础 Python 库

以下是科研与工程开发中常用的基础库：

* `numpy`：数值计算基础库
* `scipy`：科学计算与数值算法
* `matplotlib`：绘图与可视化
* `pandas`：数据处理与分析

建议统一在 **已激活的 Conda 虚拟环境** 中安装与使用：

```bash
conda activate myenv
conda install numpy scipy matplotlib pandas
```

---

#### 8.3.4 建议

* 一个项目 ≈ 一个 Conda 环境
* 不清楚用什么方式安装时，**优先选 conda**
* 不要混合使用 `sudo pip install`
* 不要随意操作系统 Python

---



## 至此，用于深度强化学习的工作站软件与开发环境已完成基础配置。需要注意的是，该环境并非一劳永逸，后续仍需根据具体研究任务、软件版本更新及硬件变化进行持续维护与调整。

