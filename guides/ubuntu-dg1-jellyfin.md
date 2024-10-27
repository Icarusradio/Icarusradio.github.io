---
layout: default
---
# Ubuntu 22.04 下修改驱动使 Intel DG1 可以在 Jellyfin 下解码
## 10.27 更新
Jellyfin 目前最新版本 10.10 已经支持使用 `xe` 内核驱动，具体请参考[使用内核 xe 驱动使 Intel DG1 可以在 Jellyfin 下解码](../xe-dg1-jellyfin)。如果之前按照这个教程配置过 Ubuntu，请先删除 `intel-i915-dkms` 和 `intel-fw-gpu` 两个包，将内核退回发行版提供的版本。如果希望节省空间，也可以将额外安装的 6.5 内核一并删除。

## 9.19 更新
目前 Linux 内核从 6.8 版本开始，提供了新的 `xe` 内核驱动。这个驱动可以支持 DG1 一些解码功能，并且不需要编译 iHD 驱动和安装特定内核。但是目前 Jellyfin 不支持使用 `xe` 内核模块进行解码（[参考](https://github.com/jellyfin/jellyfin/issues/11485)），所以想使用 Jellyfin 进行解码仍需要下述步骤。

## 起因
最近看了皮蛋熊大佬的直通 DG1 教程（[NAS独显转码新王晋级， Intel DG1驱动第二阶段，PVE内核+群晖SA6400驱动分享！](https://www.bilibili.com/video/BV1wb4y1G7kF)）自己也入手了一块 DG1。

但是大佬提供的教程里面只有群晖 SA6400，而我本人不喜欢群晖的系统，希望能够在原生的 Linux 上运行 Jellyfin。于是便有了本篇教程。

## 背景知识
查阅 [Jellyfin 官网教程](https://jellyfin.org/docs/general/administration/hardware-acceleration/intel#known-issues-and-limitations-on-linux)可以得知：

> 6. The kernel support for Intel Gen 12 DG1 is incomplete in upstream. Intel DKMS and custom iHD driver are required.

即 Linux 内核对 DG1 的支持是不完整的。如果需要 Jellyfin 成功解码需要安装 DKMS 和定制的 iHD 驱动，换句话说内核与驱动都需要修改。

Intel 官方开源了暂时没有推送至主线内核的更新：[intel-gpu/intel-gpu-i915-backports](https://github.com/intel-gpu/intel-gpu-i915-backports)。这个可以不用编译，因为 Intel 官方维护了一个 Ubuntu 仓库提供编译好的软件包。不过推荐安装之前先查阅一下这个 GitHub 仓库，原因后面会提及。

查阅 Intel 官网可以得知驱动分三个版本：Rolling, Production 和 Long Term Support (LTS)。如果追求新的特性选择 Rolling 版本，追求稳定则选择 Production 或者 LTS 版本。LTS 相比 Production 可以提供长达三年的支持。

每个版本具体发行信息可以在 [Intel 的发行信息](https://dgpu-docs.intel.com/releases/index.html)里查看。点开每个版本发行信息，可以得知在 Ubuntu 22.04 下，DG1 在三个版本均测试通过。

## 准备工作
### 安装前注意事项
如果是物理机安装，首先请参阅皮蛋熊大佬的教程修改 BIOS 来支持点亮 DG1。如果无法点亮 DG1 请先解决这个问题，本篇教程不涉及解决此问题。确认可以点亮后，请拔下 DG1，使用 CPU 自带的核显或者其他亮机卡完成安装。

如果是虚拟机安装，首先请参阅皮蛋熊大佬的教程修改 PVE 内核来支持直通 DG1。设置完成后创建虚拟机，先不要直通 DG1，等设置完成后再直通。

### 安装 Ubuntu 22.04 LTS
上述设置完成后，物理机和虚拟机系统安装方式类似，本教程以虚拟机安装为例。推荐安装 server 版本，本教程以 server 版为例，desktop 版本遇到的额外问题后面会单独描述。

在 PVE 中创建虚拟机，打开高级选项。OS 选择 Ubuntu server 安装镜像，System 里机器选择 q35，BIOS 选择 UEFI 并且取消勾选 `Pre-Enroll keys`，这样安全启动默认是关闭的。最后勾选 `Qemu Agent`，开启后管理虚拟机会更方便。

![pve-1](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/2c549e24-411e-49ed-91d0-dabd49ea91e2)

硬盘设置合适的大小即可，推荐设置 50-100G。因为 Jellyfin 会在本地缓存转码文件，占用空间较大，所以建议硬盘空间设置大一些。CPU 类型拉至最下面选择 `host`，内存设置合适大小，推荐 8-16G。后面都默认即可，设置完成打开虚拟机，准备安装 Ubuntu server。

![pve-2](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/58ef5624-ff7d-40b1-bd3c-ae65e4c21af8)

开启虚拟机后，进入安装界面，选择第一项 "Try or Install Ubuntu Server"，等待进入安装界面。这个界面下键盘上下左右移动光标位置，回车是确认，空格是勾选和取消勾选。

语言选择 English，键盘选择 English (US)，即都是默认。安装类型选择 `Ubuntu server (minimized)`，因为只是用来运行 Jellyfin，所以可以选择精简系统，如果有其它需求则选择默认的 `Ubuntu server`。

![ubuntu-1](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/590c6f4b-738c-4ccd-8a63-b4aa2186c6df)

设置好网络连接后，建议更换国内的镜像，这里我选择的是国内高校联合搭建的镜像站。也可以选择其他镜像站（比如阿里云，腾讯云等）。

![ubuntu-2](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/51ff670e-6791-4cbf-986d-0d9ee5a426c9)

划分分区时选择 `Use an entire disk` 并取消勾选 LVM，然后点击 Done 确认。弹出对话框选择 Continue 继续。

![ubuntu-3](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/6c25ee57-e6a5-4b3a-8ae3-c9b8f3e4a236)

在这里设置计算机名称，用户名和密码。本教程里将会使用这个创建用户运行 docker 版 Jellyfin。

![ubuntu-4](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/b98a3380-d822-4aaa-97a2-10258f2ee43c)

询问 Ubuntu Pro 时选择不需要，即 "Skip for now"。然后选择 Install OpenSSH server，这样安装完就可以通过 SSH 连接了。机器安装完系统后全程使用 SSH 连接，这步很重要。下一步推荐的软件全都不要选，跳过进入到安装。安装完成后根据提示重启进入系统。

机器开机后，根据 IP 和刚刚创建的用户名以及密码，通过 SSH 登录。进入系统后更新软件，然后再安装一些软件：

```bash
sudo apt update
sudo apt upgrade
sudo apt install qemu-guest-agent dialog bash-completion nano
```

这里安装的软件的作用：
1. `qemu-guest-agent` 是对应之前勾选 `Qemu Agent`，如果是物理机则不需要安装。
2. `dialog` 是安装软件包需要的一个软件，不装不影响系统使用，但是每次 apt 都会提示你装这个。
3. `bash-completion` 可以增强命令行 Tab 键补全功能，推荐安装。
4. `nano` 是命令行文本编辑器，也可以选择 `vim` 等其他编辑器，看个人喜好。

如果有其它需求可以再安装别的包，比如教程里使用 docker 运行 Jellyfin 则需要安装 docker。这里参考高校联合镜像站的教程（[Docker CE 软件仓库镜像使用帮助](https://help.mirrors.cernet.edu.cn/docker-ce/)）：

```bash
export DOWNLOAD_URL="https://mirrors.cernet.edu.cn/docker-ce"
curl -fsSL https://get.docker.com/ | sudo -E sh
```

安装完成后关闭机器。物理机拔出安装 U 盘，同时也可以拔出键盘鼠标显示器连接线。虚拟机在硬件设置里删除安装盘。操作完成后再打开机器。

![pve-3](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/5d065ee7-51b0-413f-b116-5ad321162805)

### 添加 Intel 官方 Ubuntu 仓库
首先先添加 Intel 的 GPG 密钥：
```bash
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | sudo gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg
```

添加 LTS 仓库（与下面 Rolling 仓库二选一）：

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy/lts/2350 unified" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list
```

添加 Rolling 仓库（与上面 LTS 仓库二选一）：

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy client" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list
```

更新仓库：
```bash
sudo apt update
```

## 安装 DKMS 和定制的 iHD 驱动
### 安装 DKMS
**防杠声明**：理论上 DKMS 并不需要安装指定内核，每当有新内核安装时会自动编译模块。但是本教程为了保证成功，安装了 Intel 测试成功的内核版本。本教程无法保证在不安装对应内核版本下也能正常工作。后续更新 DKMS 也推荐将内核一同更新。

在安装之前，先用 `apt` 指令查看仓库内版本

```bash
apt-cache madison intel-i915-dkms
```

这里以 Rolling 仓库为例，目前（2024 年 5 月 24 日）可查询到的版本为：

```
intel-i915-dkms | 1.24.1.11.240117.14+i16-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.9.11.231003.15+i19-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.8.20.230810.22+i33-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.7.17.230608.24+i37-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.6.24.230425.29+i44-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.5.19.230406.21.5.17.0.1034+i38-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
```

即最新版本为 `1.24.1.11.240117.14+i16-1`，记住这个版本。打开 [intel-gpu/intel-gpu-i915-backports](https://github.com/intel-gpu/intel-gpu-i915-backports) 这个仓库。在上方标签里搜索 `24.1.11` 找到对应标签并点击。

![github-1](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/389e205f-c59b-483e-ae6b-d9b07c135a66)

然后点击 `versions` 文件，查看测试对应的内核版本，比如这个标签的文件内容如下：

```
BACKPORTS_RELEASE_TAG="I915_24.1.11_PSB_240117.14"
DII_KERNEL_TAG="I915-24.1.11"
DII_KERNEL_HEAD="b204f78969214"
UBUNTU_22.04_SERVER_KERNEL_VERSION="5.15.0-100"
UBUNTU_22.04_DESKTOP_KERNEL_VERSION="6.5.0-25"
VANILLA_5.15LTS_KERNEL_VERSION="5.15.150"
VANILLA_6.1LTS_KERNEL_VERSION="6.1.80"
SLES15_SP4_KERNEL_VERSION="5.14.21-150400.24.108"
SLES15_SP5_KERNEL_VERSION="5.14.21-150500.55.49"
RHEL_9.0_KERNEL_VERSION="5.14.0-70.30.1"
RHEL_9.2_KERNEL_VERSION="5.14.0-284.30.1"
RHEL_9.3_KERNEL_VERSION="5.14.0-362.13.1"
RHEL_8.9_KERNEL_VERSION="4.18.0-513.5.1"
RHEL_8.8_KERNEL_VERSION="4.18.0-477.13.1"
RHEL_8.6_KERNEL_VERSION="4.18.0-372.32.1"
VANILLA_5.10LTS_KERNEL_VERSION="5.10.211"
VANILLA_5.4LTS_KERNEL_VERSION="5.4.270"
```

这里我们关注 `UBUNTU_22.04_DESKTOP_KERNEL_VERSION`，即测试验证的内核版本为 `6.5.0-25`。安装对应的内核版本：

```bash
sudo apt install linux-image-6.5.0-25-generic linux-headers-6.5.0-25-generic linux-modules-6.5.0-25-generic linux-modules-extra-6.5.0-25-generic
```

如果安装的是 server 版本，安装完成后重启即可，因为 server 版默认安装 5.15 内核，启动时会自动选择 6.5 内核进行启动。Desktop 版本默认安装 6.5 内核，可能版本号会比这个大，需要修改 GRUB 来指定启动使用的内核。（*待施工：添加修改教程*）

重启完成后输入 `uname -a` 来确认启动内核是否为安装版本。如果出现类似输出，说明启动内核已经更换成功。

```
Linux jellyfin 6.5.0-25-generic #25~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Tue Feb 20 16:09:15 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```

确认成功后，安装 DKMS 和 GPU 固件。

```bash
sudo apt install intel-i915-dkms intel-fw-gpu
```

安装完成后关闭机器。物理机装上 DG1，如果使用核显安装系统，请 BIOS 内选择仅独立显卡工作来关闭核显；如果使用亮机卡安装系统，请拆下亮机卡。请确保不要把 DG1 连上显示器，因为之前某个版本删除了图形输出能力，只支持硬件解码，所以连上也没有用。虚拟机上直通 DG1，并将显示关闭，理由同物理机，PVE 自带的控制台也是看不了显示输出的。

![pve-4](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/189459cd-cbed-419a-8d4f-c1e7bfd26464)

操作完成后打开机器，输入 `sudo dmesg | grep -i backport` 来确认内核是否成功安装。如果出现类似如下结果说明安装成功。

```
[    1.271469] BACKPORTED INTEL VSEC PCI PROBE
[    1.348464] COMPAT BACKPORTED INIT
[    1.348931] Loading modules backported from I915-24.1.11
[    1.349358] Backport generated by backports.git I915_24.1.11_PSB_240117.14
[    1.645684] [drm] I915 BACKPORTED INIT
[    4.554580] VSEC CLASS BACKPORTED INIT
[    4.555701] BACKPORTED VSEC TELEMETRY INIT
[    4.568976] [drm] I915 SPI BACKPORTED INIT
```

### 安装 Jellyfin
Ubuntu 下安装 Jellyfin 有两种方式，一种是通过 apt 安装官方编译的 deb 包，另一种是使用 docker。这里选择 docker 安装，镜像选择 nyanmisaka 大佬制作的 [Jellyfin 中国特供版](https://hub.docker.com/r/nyanmisaka/jellyfin)。

首先查看自己用户和用户组的 ID，命令行输入 `id` 指令，查看显示的 `uid` 和 `gid` 的对应值，这里以两个值均是 1000 为例，请修改为自己机器上的数值。

再查看显卡对应用户组，输入 `ls -l /dev/dri/` 指令，查看 `renderD128` 对应的用户组。这里显示为 `render`。

```
total 0
drwxr-xr-x  2 root root         80  5月 24 00:23 by-path
crw-rw----+ 1 root video  226,   0  5月 24 00:23 card0
crw-rw----+ 1 root render 226, 128  5月 24 00:23 renderD128
```

然后输入指令 `getent group render | cut -d: -f3` 查看 `render` 组对应的 ID。这里以 ID 是 110 为例，请修改为自己机器上的数值。

管理 Jellyfin 的配置文件有两种方式：
1. 创建本地文件夹，这样可以更方便的查看日志文件
2. 使用 docker 卷管理，这样可以交给 docker 管理

#### 创建本地文件夹
输入指令 `mkdir -p ~/Jellyfin/{config,cache}` 在自己的用户目录创建配置和缓存文件夹

然后创建 docker 容器（指令需要注意和修改的地方参阅下方注意事项）：

```bash
sudo docker run -d \
 --name=jellyfin \
 --mount type=bind,source=/home/USERNAME/Jellyfin/config,target=/config \
 --mount type=bind,source=/home/USERNAME/Jellyfin/cache,target=/cache \
 --mount type=bind,source=/path/to/media,target=/media \
 --user 1000:1000 \
 --group-add="110" \
 --net=host \
 --restart=unless-stopped \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 nyanmisaka/jellyfin
```

#### 使用 docker 卷管理
先创建两个 docker 卷分别存储 Jellyfin 的配置文件与缓存：

```bash
sudo docker volume create jellyfin-config
sudo docker volume create jellyfin-cache
```

然后创建 docker 容器（指令需要注意和修改的地方参阅下方注意事项）：

```bash
sudo docker run -d \
 --name=jellyfin \
 --volume jellyfin-config:/config \
 --volume jellyfin-cache:/cache \
 --mount type=bind,source=/path/to/media,target=/media \
 --user 1000:1000 \
 --group-add="110" \
 --net=host \
 --restart=unless-stopped \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 nyanmisaka/jellyfin
```

#### 注意事项
1. 请将 `--user 1000:1000` 里数值修改为自己用户对应的 ID。
2. 请将 `--group-add="110"` 里数值修改为查询到的 `render` 用户组 ID。
3. 在创建本地文件夹中，请将前两行 `--mount` 参数里的 `USERNAME` 替换成你自己的用户名，因为这里需要填写绝对路径。
4. 请将 `--mount type=bind,source=/path/to/media,target=/media` 中 `/path/to/media` 修改为存储媒体的绝对路径，并确保 1 中设置的用户对这个文件夹有读写权限。

### 安装定制 iHD 驱动
安装完成 Jellyfin 后，输入指令进入 docker 内命令行：

``` bash
sudo docker exec -it jellyfin /bin/bash
```

在命令行内输入指令测试 ffmpeg：

```bash
/usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```

此时应该会报错，报错内容类似如下：

```
Trying display: drm
libva info: VA-API version 1.21.0
libva info: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_21
double free or corruption (!prev)
Aborted (core dumped)
```

因为我们还没有替换驱动文件，docker 内自带的文件不支持 DG1。这里我编译好了驱动文件，请去[这个仓库](https://github.com/Icarusradio/intel-media-driver-dg1/releases)下载。这个仓库我会尽量保持与 Intel 官方最新版本一致。如果想要自行编译，请参阅文章附录的编译教程。

用以下指令将下载的 `iHD_drv_video.so` 替换 docker 内的：

```bash
sudo docker cp iHD_drv_video.so jellyfin:/usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
```

此时再输入上述指令进入 docker 内命令行并执行测试 ffmpeg 指令，如果出现类似输出证明替换成功：

```
Trying display: drm
libva info: VA-API version 1.21.0
libva info: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_20
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.21 (libva 2.21.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 24.1.5 ()
vainfo: Supported profile and entrypoints
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileNone                   : VAEntrypointStats
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Simple            : VAEntrypointEncSlice
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointFEI
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointFEI
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline: VAEntrypointFEI
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointFEI
      VAProfileHEVCMain               : VAEntrypointEncSliceLP
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointEncSliceLP
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointEncSliceLP
      VAProfileVP9Profile1            : VAEntrypointVLD
      VAProfileVP9Profile1            : VAEntrypointEncSliceLP
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointEncSliceLP
      VAProfileVP9Profile3            : VAEntrypointVLD
      VAProfileVP9Profile3            : VAEntrypointEncSliceLP
      VAProfileHEVCMain12             : VAEntrypointVLD
      VAProfileHEVCMain12             : VAEntrypointEncSlice
      VAProfileHEVCMain422_10         : VAEntrypointVLD
      VAProfileHEVCMain422_10         : VAEntrypointEncSlice
      VAProfileHEVCMain422_12         : VAEntrypointVLD
      VAProfileHEVCMain422_12         : VAEntrypointEncSlice
      VAProfileHEVCMain444            : VAEntrypointVLD
      VAProfileHEVCMain444            : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_10         : VAEntrypointVLD
      VAProfileHEVCMain444_10         : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_12         : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain10          : VAEntrypointVLD
      VAProfileHEVCSccMain10          : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain444         : VAEntrypointVLD
      VAProfileHEVCSccMain444         : VAEntrypointEncSliceLP
      VAProfileAV1Profile0            : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointEncSliceLP
```

再输入以下指令检查 OpenCL。

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -v verbose -init_hw_device vaapi=va:/dev/dri/renderD128 -init_hw_device opencl@va
```

如果出现类似输出证明成功：

```
ffmpeg version 6.0.1-Jellyfin Copyright (c) 2000-2023 the FFmpeg developers
  built with gcc 12 (Debian 12.2.0-14)
  configuration: --prefix=/usr/lib/jellyfin-ffmpeg --target-os=linux --extra-version=Jellyfin --disable-doc --disable-ffplay --disable-ptx-compression --disable-static --disable-libxcb --disable-sdl2 --disable-xlib --enable-lto --enable-gpl --enable-version3 --enable-shared --enable-gmp --enable-gnutls --enable-chromaprint --enable-opencl --enable-libdrm --enable-libass --enable-libfreetype --enable-libfribidi --enable-libfontconfig --enable-libbluray --enable-libmp3lame --enable-libopus --enable-libtheora --enable-libvorbis --enable-libopenmpt --enable-libdav1d --enable-libsvtav1 --enable-libwebp --enable-libvpx --enable-libx264 --enable-libx265 --enable-libzvbi --enable-libzimg --enable-libfdk-aac --arch=amd64 --enable-libshaderc --enable-libplacebo --enable-vulkan --enable-vaapi --enable-amf --enable-libvpl --enable-ffnvcodec --enable-cuda --enable-cuda-llvm --enable-cuvid --enable-nvdec --enable-nvenc
  libavutil      58.  2.100 / 58.  2.100
  libavcodec     60.  3.100 / 60.  3.100
  libavformat    60.  3.100 / 60.  3.100
  libavdevice    60.  1.100 / 60.  1.100
  libavfilter     9.  3.100 /  9.  3.100
  libswscale      7.  1.100 /  7.  1.100
  libswresample   4. 10.100 /  4. 10.100
  libpostproc    57.  1.100 / 57.  1.100
[AVHWDeviceContext @ 0x62e0982862c0] libva: VA-API version 1.21.0
[AVHWDeviceContext @ 0x62e0982862c0] libva: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
[AVHWDeviceContext @ 0x62e0982862c0] libva: Found init function __vaDriverInit_1_20
[AVHWDeviceContext @ 0x62e0982862c0] libva: va_openDriver() returns 0
[AVHWDeviceContext @ 0x62e0982862c0] Initialised VAAPI connection: version 1.21
[AVHWDeviceContext @ 0x62e0982862c0] VAAPI driver: Intel iHD driver for Intel(R) Gen Graphics - 24.1.5 ().
[AVHWDeviceContext @ 0x62e0982862c0] Driver not found in known nonstandard list, using standard behaviour.
[AVHWDeviceContext @ 0x62e0982b7200] 0.0: Intel(R) OpenCL Graphics / Intel(R) Iris(R) Xe Graphics
[AVHWDeviceContext @ 0x62e0982b7200] Intel QSV to OpenCL mapping function found (clCreateFromVA_APIMediaSurfaceINTEL).
[AVHWDeviceContext @ 0x62e0982b7200] Intel QSV in OpenCL acquire function found (clEnqueueAcquireVA_APIMediaSurfacesINTEL).
[AVHWDeviceContext @ 0x62e0982b7200] Intel QSV in OpenCL release function found (clEnqueueReleaseVA_APIMediaSurfacesINTEL).
Hyper fast Audio and Video encoder
usage: ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...

Use -h to get full help or, even better, run 'man ffmpeg'
```

安装成功后，重启不会失效。由于之前 docker 运行参数配置了 `--restart=unless-stopped`，所以只要不手动关闭 Jellyfin，开机会自动启动。按照 nyanmisaka 大佬说法目前暂时没有整合驱动的想法，所以每次更新 docker 镜像需要重新替换驱动文件。

> 这个ENABLE_PRODUCTION_KMD之前是和主线内核不兼容的，所以我们暂时没开。

运行测试视频解码，全部统一 1080P 10Mbps 码率。与皮蛋熊大佬的测试（[数据](https://blog.kkk.rs/archives/32)）对比，可以看出基本区别不大，说明成功解码了。

|文件|格式|帧率|帧率（by 皮蛋熊）|
|:-:|:-:|:-:|:-:|
|Taylor Swift|4K H264|308|290|
|三星HDR|4K HEVC|210|293|
|蔡依林|8K HEVC|120|134|
|地球上|8K HEVC|119|122|
|Meridian|8K AV1|92|93|

## 结语
这篇教程其实很早就打算写了，但是自己拖延症导致拖了很久，还是向各位抱歉。

教程有什么问题，欢迎去我的 GitHub 仓库 ([Icarusradio/intel-media-driver-dg1](https://github.com/Icarusradio/intel-media-driver-dg1/discussions)) 的 Discussions 中提问，我会尽量抽空回复的。也欢迎各位有能力的大佬帮助我回答他人的问题。

最后是对 nyanmisaka 和皮蛋熊两位大佬的感谢。

首先是感谢 nyanmisaka 大佬。他不仅制作了特供版 Jellyfin 造福广大国内用户，他本人也是 Jellyfin 开发人员之一，Jellyfin 官方文档很多地方都是他写的。没有他制作的详细文档，我也没有办法一步步摸索出驱动 DG1 的解决方案。

其次是感谢皮蛋熊大佬。他成功让 DG1 能在 PVE 下直通拓展了这张显卡更多的可玩性，相比他的工作，我做的这些只能说是锦上添花。

欢迎各位点击两位大佬的主页去支持他们：
- nyanmisaka 的 [GitHub 主页](https://github.com/nyanmisaka)
- [皮蛋熊的博客](https://blog.kkk.rs/)

## 附录
### 编译 iHD 驱动
开源驱动位于 [intel/media-driver](https://github.com/intel/media-driver)，这里说明了如何编译支持 DG1 的驱动：

> Media-driver requires special i915 kernel mode driver (KMD) version to support the following platforms since upstream version of i915 KMD does not fully support them(pending patches upstream). To enable these platforms, it requires to specify ENABLE_PRODUCTION_KMD=ON (default: OFF) build configuration option.  
>   - DG1/SG1
>   - ATSM

编译驱动其实不限制 Linux 发行版，这里为了方便仍使用 Ubuntu server 22.04 LTS 进行编译。

先点开上述 GitHub 仓库 Releases 下载 Intel 发布的源代码。这里选择最新的 "Intel Media Driver 2024Q1 Release - 24.1.5" 版本，拖至最下方点击 "Source code (tar.gz)" 下载源代码。在发行说明里 Intel 标注了 Gmmlib 和 Libva 需求的版本，因为 Ubuntu 官方仓库软件版本过老，直接安装可能会编译无法通过，有两种解决办法：
1. 安装 Intel 官方维护仓库版本（推荐）
2. 手动编译安装 Gmmlib 和 Libva

#### 安装 Intel 官方维护仓库版本
这个方法相比手动编译安装，可以用包管理器管理安装的软件包。因为我们只需要编译出驱动文件替换 docker 内的即可，系统内其实不需要这些软件包，可以在编译完成后删除节省空间。

按照前面的步骤添加 Intel 官方仓库，然后安装编译所需的软件包：

```bash
sudo apt install libva-dev libigdgmm-dev autoconf libtool libdrm-dev xorg xorg-dev openbox libx11-dev libgl1-mesa-glx cmake g++
```

等待安装完后，将之前下载的驱动源代码上传至 Ubuntu 并解压。在同一目录下创建两个文件夹：`build_media` 和 `install_media`：

```bash
tar xf media-driver-intel-media-24.1.5.tar.gz
mkdir -p build_media install_media
```

当前文件夹下应该有三个文件夹：`media-driver-intel-media-24.1.5`, `build_media` 和 `install_media`。然后开始编译：

```bash
cd build_media
cmake -DENABLE_PRODUCTION_KMD=ON -DBUILD_CMRTLIB=OFF ../media-driver-intel-media-24.1.5 # 设置编译选项
make -j"$(nproc)"                                                                       # 利用所有核心编译
DESTDIR=../install_media make install                                                   # 安装到 install_media 文件夹
strip ../install_media/usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so                    # 去除多余的 symbol，缩减体积
```

编译好的驱动文件就在 `install_media/usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so`，按照上面步骤替换 docker 内文件即可。

#### 手动编译安装 Gmmlib 和 Libva
相比上个方法，这个方法并不推荐，因为手动安装的不方便用包管理器进行管理，只推荐在上个方法失败时使用。由于手动安装会影响包管理器，建议使用虚拟机编译，并在编译之前创建一个快照备份。编译完成后可以通过快照恢复至编译前状态。

以目前（2024 年 9 月 19 日）最新版本 [24.2.5](https://github.com/intel/media-driver/releases/tag/intel-media-24.2.5) 为例，发行日志写明依赖项如下：

> Dependencies
>   - GmmLib: [intel-gmmlib-22.3.20](https://github.com/intel/gmmlib/releases/tag/intel-gmmlib-22.3.20)
>   - Libva: [2.22.0](https://github.com/intel/libva/releases/tag/2.22.0)

点开链接后均选择 `Source code (tar.gz)` 下载源代码。先安装编译 `libva` 所需要的软件包：

```bash
sudo apt install cmake pkg-config meson libdrm-dev automake libtool
```

然后解压源代码包，并开始编译：

```bash
tar xf libva-2.22.0.tar.gz
cd libva-2.22.0
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make
sudo make install
```

之后解压并编译 `gmmlib` ：

```bash
tar xf gmmlib-intel-gmmlib-22.3.20.tar.gz
cd gmmlib-intel-gmmlib-22.3.20
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j"$(nproc)"
sudo make install
```

之后就和上述一样编译 `media-driver` 并提取 `iHD_drv_video.so` 即可。

### 构建 DKMS 安装包
之前提过 Intel 官方有三个版本：Rolling, Production 和 Long Term Support (LTS)。但是软件仓库却只有 Rolling 和 LTS，如何安装 Production 版呢？或者 GitHub 上 Rolling 版已更新，但是仓库里并没有。这些问题都可以通过自行打包 DKMS 安装包来实现。

这里用词使用 “构建” 而不是 “编译”，因为这里只是将 Intel 公布的源代码塞入 deb 安装包，并没有编译的过程。

以目前（2024 年 9 月 19 日）最新 Rolling 版本 [I915_24WW33.3_950.13_24.4.12_240603.18](https://github.com/intel-gpu/intel-gpu-i915-backports/releases/tag/I915_24WW33.3_950.13_24.4.12_240603.18) 为例。首先下载这个 tag 对应的代码，点击 `Source code (tar.gz)` 下载源代码。

然后安装打包所需的依赖：

```bash
sudo apt install dkms make debhelper devscripts build-essential flex bison mawk
```

然后解压并打包 DKMS 安装包：

```bash
tar xf intel-gpu-i915-backports-I915_24WW33.3_950.13_24.4.12_240603.18.tar.gz
cd intel-gpu-i915-backports-I915_24WW33.3_950.13_24.4.12_240603.18
make i915dkmsdeb-pkg OS_DISTRIBUTION=UBUNTU_22.04_DESKTOP
```

这里我们选择 `UBUNTU_22.04_DESKTOP`，这个对应的内核版本是 6.5，如果想要打包 6.8 版本对应的 DKMS 就选择 `UBUNTU_24.04_DESKTOP` 或者 `UBUNTU_24.04_SERVER`，`UBUNTU_22.04_SERVER` 对应的内核版本是 5.15，不建议选择。等待命令执行结束，生成的 deb 安装包就在这个文件夹的上一级目录。
