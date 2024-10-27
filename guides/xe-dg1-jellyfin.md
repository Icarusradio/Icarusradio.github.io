---
layout: default
---
# 使用内核 xe 驱动使 Intel DG1 可以在 Jellyfin 下解码
## 背景
Linux 内核在 6.8 版本引入了 `xe` 内核模块（[参考](https://www.phoronix.com/news/Linux-6.8-Released)），来支持 Intel 较新的显卡，DG1 也包括在内。而在 2024 年 10 月 26 日，Jellyfin 官方更新到 [10.10](https://jellyfin.org/posts/jellyfin-release-10.10.0) 版本，正式支持内核 `xe` 驱动。所以目前使用内核版本大于等于 6.8 的 Linux 发行版（例如 [Arch Linux](https://archlinux.org/) 和 [Ubuntu](https://ubuntu.com/)）可以直接使用 DG1 在 Jellyfin 下转码，而不需要像之前一样安装 backport i915 的内核和修改驱动（参见：[Ubuntu 22.04 下修改驱动使 Intel DG1 可以在 Jellyfin 下解码](./ubuntu-dg1-jellyfin)）。本文以 Ubuntu Server 22.04 LTS 为例，介绍如何安装 6.8 内核并开启 Jellyfin 硬件转码。

## 安装 6.8 内核
如果 Ubuntu 在安装时选择的是 HWE 内核，那么有可能通过 apt 更新就可以安装 6.8 内核。首先命令行执行 `uname -r` 查看内核版本号，如果返回结果类似 `6.8.0-47-generic`，证明已经安装 6.8 内核了，可以跳过这一步。如果返回版本不是，请按照下述步骤安装 6.8 内核。

最快捷的办法就是安装 Ubuntu 的 HWE 内核：

```bash
sudo apt update # 更新仓库
sudo apt install --install-recommends linux-generic-hwe-22.04
```

安装完成后重启，重启后命令行执行 `uname -r`，如果显示 6.8 版本说明安装成功。

## 添加内核启动项，使用 xe 内核驱动
因为目前 `xe` 内核模块还在实验阶段，Linux 内核默认不会加载，需要添加内核启动项。首先使用 `lspci -nn` 查看 DG1 的 PCI ID，这里以华硕圣旗全高 80EU 版本为例，返回值如下：

```
01:00.0 VGA compatible controller [0300]: Intel Corporation DG1 [Iris Xe Graphics] [8086:4908] (rev 01)
```

这里 `[8086:4908]` 前四位是制造商 ID，后四位就是我们需要的 PCI ID。然后修改 `/etc/default/grub`，找到 `GRUB_CMDLINE_LINUX_DEFAULT` 这一行，在已有参数后面添加 `i915.force_probe=!4908 xe.force_probe=4908`，记得新增参数要与原有参数以空格隔开。如果之前内核添加过其他 `i915` 相关参数（比如 `i915.enable_guc=3`），请删除，因为不会再使用 `i915` 内核模块驱动 DG1。

修改完成后保存，执行 `sudo update-grub` 更新 GRUB 启动文件，然后重启。重启完成后执行 `sudo lspci -v` 查看 DG1 对应信息：

```
01:00.0 VGA compatible controller: Intel Corporation DG1 [Iris Xe Graphics] (rev 01) (prog-if 00 [VGA controller])
        Subsystem: ASUSTeK Computer Inc. DG1 [Iris Xe Graphics]
        Physical Slot: 0
        Flags: bus master, fast devsel, latency 0, IRQ 45
        Memory at 80000000 (64-bit, non-prefetchable) [size=16M]
        Memory at 7000000000 (64-bit, prefetchable) [size=4G]
        Expansion ROM at <ignored> [disabled]
        Capabilities: [40] Vendor Specific Information: Len=0c <?>
        Capabilities: [70] Express Endpoint, MSI 00
        Capabilities: [ac] MSI: Enable+ Count=1/1 Maskable+ 64bit+
        Capabilities: [d0] Power Management version 3
        Capabilities: [100] Latency Tolerance Reporting
        Kernel driver in use: xe
        Kernel modules: i915, xe
```

如果出现 `Kernel driver in use: xe` 说明内核已经在使用 `xe` 内核模块驱动 DG1。驱动完成后就可以使用 Jellyfin 调用 DG1 硬件解码了。

## 结语
使用最新 Jellyfin 10.10，内核版本 `6.8.0-47-generic` 测试解码能力，与皮蛋熊大佬的测试（[数据](https://blog.kkk.rs/archives/32)）对比，可以看出基本区别不大，说明成功解码了。

|文件|格式|帧率|帧率（by 皮蛋熊）|
|:-:|:-:|:-:|:-:|
|Taylor Swift|4K H264|310|290|
|三星HDR|4K HEVC|315|293|
|蔡依林|8K HEVC|138|134|
|地球上|8K HEVC|122|122|
|Meridian|8K AV1|93|93|
