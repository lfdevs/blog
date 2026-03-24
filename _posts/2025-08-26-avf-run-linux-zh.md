---
title: 利用 Android 虚拟化框架（AVF）运行自定义 Linux 虚拟机
author: lfdevs
date: 2025-08-26 16:24:42 +0800
categories: [Android]
tags: [avf, linux]
---
> 本文于 2025 年 8 月 26 日在 GitHub 发布：[https://github.com/lfdevs/run-linux-on-android-guide/blob/main/docs/zh-HanS/avf-linux.md](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/docs/zh-HanS/avf-linux.md)
{: .prompt-tip }

{% include embed/bilibili.html id='BV1PdaJzbEa4' %}

[Android 虚拟化框架（AVF）](https://source.android.com/docs/core/virtualization)提供安全且私密的执行环境来执行代码。Android 提供了实现 AVF 所需的所有组件的参考实现。尽管 AVF 主要用来提供比 Android 应用沙盒安全系数更高、甚至经过正式验证的隔离保证，但我们也可以利用它来运行自定义 Linux 虚拟机。Android 15 QPR2 及以上版本的“Linux 开发环境”也验证了这一点。

## 系统需求

  * **操作系统**：Android 13 及以上
  * **SoC**：Google Tensor、联发科天玑 9400、三星猎户座 2500 及更新的 SoC

> 由于高通的限制，骁龙 8 Elite 目前仅支持受保护的虚拟机 ，如 Microdroid，不能运行未经 Google 或 OEM 签名的自定义 Linux 虚拟机。但如果有 `root` 权限，您也可以参考 [polygraphene 的指南](https://github.com/polygraphene/gunyah-on-sd-guide)。您可以使用 `adb shell` 运行命令 `/apex/com.android.virt/bin/vm run-microdroid` 来确定您的 Android 设备是否支持运行自定义 Linux 虚拟机，运行 Microdroid 时可能需要通过 `adb -s localhost:8000 shell` 连接虚拟机的 Shell。本文所有操作步骤适用于大部分的 Linux 发行版，其他操作系统使用相应的命令即可。
{: .prompt-info }

## 无头虚拟机

由于从头开始自行制作虚拟机镜像的步骤较为繁琐，所以本节仅讨论使用 Google 官方专门为 AVF 制作的 Debian 镜像的情况。

### 1\. 下载镜像

1. 下载 Google 官方最新的 Debian 镜像并解压。

   ```bash
   wget https://dl.google.com/android/ferrochrome/latest/aarch64/images.tar.gz
   tar -zxvf images.tar.gz
   ```

2. 将解压得到的 `root_part` 4K 对齐。向上调整 `root_part` 的文件大小，字节数需要为 4096 的整倍数。可参考以下命令。

   ```bash
   truncate -s 6442450944 root_part # 调整为 6 GiB
   ```

### 2\. （可选）修改镜像

可以利用 `chroot` 对 `root_part` 做一些简单的修改，如更改语言、安装升级软件包等。

1. 挂载镜像和系统特殊目录。

   ```bash
   sudo mkdir -p /mnt/debian_vm
   sudo mount -o loop root_part /mnt/debian_vm
   sudo mount --bind /dev /mnt/debian_vm/dev/
   sudo mount --bind /proc /mnt/debian_vm/proc/
   sudo mount --bind /sys /mnt/debian_vm/sys/
   ```

2. 如果主系统不是 `aarch64` 架构，需要使用 `qemu-aarch64-static` 来进行二进制翻译。若主系统是 Debian 系发行版，可参考以下命令。

   ```bash
   sudo apt update
   sudo apt install -y qemu-user-static binfmt-support
   sudo cp /usr/bin/qemu-aarch64-static /mnt/debian_vm/usr/bin/
   ```

3. 进入 `chroot` 环境并进行修改。

   ```bash
   # 进入 chroot 环境
   sudo chroot /mnt/debian_vm
   # 配置 DNS
   rm /etc/resolv.conf
   echo "nameserver 8.8.8.8" | tee /etc/resolv.conf
   # 升级软件包
   apt update && apt upgrade -y
   # 根据您的需要安装新的软件包
   apt install -y vim nano wget curl git sysbench locales
   # 重新配置语言环境
   dpkg-reconfigure locales
   ```

4. 退出 `chroot` 环境并卸载相关挂载点。

   ```bash
   exit
   sudo umount /mnt/debian_vm/sys
   sudo umount /mnt/debian_vm/proc
   sudo umount /mnt/debian_vm/dev
   sudo umount /mnt/debian_vm
   sudo rmdir /mnt/debian_vm
   ```

### 3\. 运行虚拟机

1. 可以修改 `vm_config.json` 文件来配置虚拟机。比如修改分配给虚拟机的运行内存大小：

   ```json
   "memory_mib" : 8192
   ```

2. 将 `images.tar.gz` 解压后的文件发送到您的 Android 设备。

   ```bash
   adb push vm_config.json /data/local/tmp/
   adb push vmlinuz /data/local/tmp/
   adb push initrd.img /data/local/tmp/
   adb push kernel_extras_part /data/local/tmp/
   adb push root_part /data/local/tmp/
   ```

3. 在 `adb shell` 内执行以下命令来运行虚拟机。

   ```bash
   /apex/com.android.virt/bin/vm run /data/local/tmp/vm_config.json
   ```

4. 在虚拟机的 Shell 内执行以下命令来关闭虚拟机。

   ```bash
   shutdown 0
   ```

## 使用 u-boot 运行虚拟机

### 1\. 下载 u-boot.bin

1. 使用浏览器前往 [aosp_u-boot-mainline Branch Grid](https://ci.android.com/builds/branches/aosp_u-boot-mainline/grid) 网站，在 `boot_crosvm_aarch64` 列选择最新的项目。

   ![](/assets/img/posts/2025-08/0001.png)

2. 在打开的新标签页中选择 `u-boot.bin` 文件，浏览器会自动下载该文件。

### 2\. 下载 Linux 发行版的原始磁盘映像

使用的原始磁盘映像需要包含 Linux 系统完整的分区。本文选择的是 Debian 13 的 `nocloud` 云镜像，您也可以前往 [Debian Official Cloud Images](https://cloud.debian.org/images/cloud/) 自行下载。

```bash
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-nocloud-arm64.tar.xz
tar -xvf debian-13-nocloud-arm64.tar.xz
mv disk.raw debian-13-nocloud-arm64.raw
```

### 3\. （可选）修改镜像

1. 扩容镜像中的分区。若主系统是 Debian 系发行版，可参考以下命令。

   ```bash
   sudo apt update
   sudo apt install -y qemu-utils
   qemu-img resize -f raw debian-13-nocloud-arm64.raw 8G
   sudo losetup -fP debian-13-nocloud-arm64.raw
   # 使用 KDE 分区管理器或 GParted 等应用调整 /dev/loop0 的分区（请根据实际情况确定循环设备编号）
   sudo losetup -d /dev/loop0
   ```

2. 如果主系统不是 `aarch64` 架构，可使用 `qemu-system-aarch64` 来启动虚拟机。若主系统是 Debian 系发行版，可参考以下命令。

   ```bash
   sudo apt update
   sudo apt install -y qemu-system
   qemu-system-aarch64 \
     -machine virt \
     -cpu cortex-a72 \
     -smp 8 \
     -m 8192 \
     -drive if=none,file=debian-13-nocloud-arm64.raw,format=raw,id=hd0 \
     -device virtio-blk-pci,drive=hd0 \
     -netdev user,id=net0,hostfwd=tcp::2222-:22 \
     -device virtio-net-pci,netdev=net0 \
     -device virtio-gpu-pci \
     -device qemu-xhci \
     -device usb-kbd \
     -device usb-tablet \
     -display gtk,gl=on \
     -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd
   ```

3. 虚拟机启动后，可在 QEMU 的 `serial0` 视图登录 root 账户并进行修改。

   ```bash
   # 升级软件包
   apt update && apt upgrade -y
   # 根据您的需要安装新的软件包
   apt install -y vim nano wget curl git sysbench locales
   # 重新配置语言环境
   dpkg-reconfigure locales
   ```

### 4\. 运行虚拟机

1. 编写 `u_boot_vm_config.json` 文件来启动虚拟机。可以使用或参照 [u_boot_vm_config.json](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/configurations/avf/u-boot/u_boot_vm_config.json)。

2. 将相关文件发送到您的 Android 设备。

   ```bash
   adb push debian-13-nocloud-arm64.raw /data/local/tmp/
   adb push u-boot.bin /data/local/tmp/
   adb push u_boot_vm_config.json /data/local/tmp/
   ```

3. 在 `adb shell` 内执行以下命令来运行虚拟机。

   ```bash
   /apex/com.android.virt/bin/vm run /data/local/tmp/u_boot_vm_config.json
   ```

4. 在虚拟机的 Shell 内执行以下命令来关闭虚拟机。

   ```bash
   shutdown 0
   ```

## FAQ

### 虚拟机怎么连接网络？

由于 Android 的限制，创建虚拟网卡需要 root 权限。如果您的 Android 设备能够使用 root 权限，则可以使用 [setup_network.sh](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/scripts/avf/setup_network.sh) 来为虚拟机创建一个虚拟网卡 `crosvm_tap`。

1. 在您的 Android 设备上执行 `setup_network.sh`。

   ```bash
   wget https://github.com/lfdevs/run-linux-on-android-guide/raw/refs/heads/main/scripts/avf/setup_network.sh
   adb push setup_network.sh /data/local/tmp/
   adb root
   adb shell chmod +x /data/local/tmp/setup_network.sh
   adb shell /data/local/tmp/setup_network.sh
   adb unroot
   ```

2. 在虚拟机内配置静态 IP 地址。如果您使用的是 Google 官方的 Debian 镜像或 Debian 官方的 `nocloud` 云镜像，则可以新建一个配置文件 `/etc/systemd/network/20-virtio-net.network`，并写入以下内容。

   ```ini
   [Match]
   Name=eth0

   [Network]
   Address=192.168.1.2/24
   Gateway=192.168.1.1
   DNS=8.8.8.8
   DNS=1.1.1.1
   ```

3. 重新加载 `systemd-networkd` 服务以应用新配置。

   ```bash
   systemctl restart systemd-networkd
   ```

> 该方法参考了[这篇 AVF 文档](https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/tags/android-15.0.0_r5/docs/custom_vm.md)，如果您遇到了任何问题或有其他方案，欢迎在 [Issues](https://github.com/lfdevs/run-linux-on-android-guide/issues) 中一起讨论。
{: .prompt-info }

### 虚拟机怎么使用桌面环境？

由于 Android 的限制，在 Android 和虚拟机之间进行端口转发或 vsock 通信均需要 root 权限。但我没有符合条件的设备进行测试，如果您有任何想法或方案，欢迎在 [Issues](https://github.com/lfdevs/run-linux-on-android-guide/issues) 中一起讨论。

## 参考资料

* [Android Virtualization Framework (AVF) overview](https://source.android.com/docs/core/virtualization)
* [Virtualization - Android Code Search](https://cs.android.com/android/platform/superproject/main/+/main:packages/modules/Virtualization/)
* [Custom VM](https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/tags/android-15.0.0_r5/docs/custom_vm.md)
