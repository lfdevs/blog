---
title: Running Custom Linux VMs with the Android Virtualization Framework (AVF)
author: lfdevs
date: 2025-08-26 16:24:43 +0800
categories: [Android]
tags: [avf, linux]
---
> This note was published on GitHub on August 26, 2025: [https://github.com/lfdevs/run-linux-on-android-guide/blob/main/docs/en-US/avf-linux.md](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/docs/en-US/avf-linux.md)
{: .prompt-tip }

{% include embed/bilibili.html id='BV1PdaJzbEa4' %}

The [Android Virtualization Framework (AVF)](https://source.android.com/docs/core/virtualization) provides secure and private execution environments for executing code. Android provides a reference implementation of all the components needed to implement AVF. Although AVF is primarily used to provide stronger, even formally verified, isolation assurances over those offered by Android's app sandbox, it can also be used to run custom Linux virtual machines. This is also demonstrated by the "Linux Development Environment" in Android 15 QPR2 and later.

## System Requirements

  * **Operating System**: Android 13 and later
  * **SoC**: Google Tensor, MediaTek Dimensity 9400, Samsung Exynos 2500, and newer SoCs

> Due to Qualcomm's restrictions, the Snapdragon 8 Elite currently only supports protected VMs like Microdroid and cannot run custom Linux VMs that are not signed by Google or the OEM. However, with `root` access, you can refer to [polygraphene's guide](https://github.com/polygraphene/gunyah-on-sd-guide). You can use `adb shell` to run the command `/apex/com.android.virt/bin/vm run-microdroid` to determine if your Android device supports running custom Linux VMs; you may need to connect to the VM's shell via `adb -s localhost:8000 shell` when running Microdroid. All steps in this article apply to most Linux distributions; use the corresponding commands for other operating systems.
{: .prompt-info }

## Headless VMs

Since creating a VM image from scratch is a rather cumbersome process, this section only discusses using the official Debian image made by Google specifically for AVF.

### 1\. Download the Image

1. Download the latest official Debian image from Google and extract it.

   ```bash
   wget https://dl.google.com/android/ferrochrome/latest/aarch64/images.tar.gz
   tar -zxvf images.tar.gz
   ```

2. 4K-align the extracted `root_part`. Round up the file size of `root_part` to the nearest multiple of 4096 bytes. You can use the following command.

   ```bash
   truncate -s 6442450944 root_part # Resize to 6 GiB
   ```

### 2\. (Optional) Modify the Image

You can use `chroot` to make simple modifications to `root_part`, such as changing the language or installing/upgrading packages.

1. Mount the image and special system directories.

   ```bash
   sudo mkdir -p /mnt/debian_vm
   sudo mount -o loop root_part /mnt/debian_vm
   sudo mount --bind /dev /mnt/debian_vm/dev/
   sudo mount --bind /proc /mnt/debian_vm/proc/
   sudo mount --bind /sys /mnt/debian_vm/sys/
   ```

2. If the host system is not an `aarch64` architecture, you need to use `qemu-aarch64-static` for binary translation. If the host system is a Debian-based distribution, you can use the following commands.

   ```bash
   sudo apt update
   sudo apt install -y qemu-user-static binfmt-support
   sudo cp /usr/bin/qemu-aarch64-static /mnt/debian_vm/usr/bin/
   ```

3. Enter the `chroot` environment and make your changes.

   ```bash
   # Enter the chroot environment
   sudo chroot /mnt/debian_vm
   # Configure DNS
   rm /etc/resolv.conf
   echo "nameserver 8.8.8.8" | tee /etc/resolv.conf
   # Upgrade packages
   apt update && apt upgrade -y
   # Install new packages as needed
   apt install -y vim nano wget curl git sysbench locales
   # Reconfigure locales
   dpkg-reconfigure locales
   ```

4. Exit the `chroot` environment and unmount the relevant mount points.

   ```bash
   exit
   sudo umount /mnt/debian_vm/sys
   sudo umount /mnt/debian_vm/proc
   sudo umount /mnt/debian_vm/dev
   sudo umount /mnt/debian_vm
   sudo rmdir /mnt/debian_vm
   ```

### 3\. Run the VM

1. You can modify the `vm_config.json` file to configure the VM. For example, to change the amount of RAM allocated to the VM:

   ```json
   "memory_mib" : 8192
   ```

2. Send the extracted files from `images.tar.gz` to your Android device.

   ```bash
   adb push vm_config.json /data/local/tmp/
   adb push vmlinuz /data/local/tmp/
   adb push initrd.img /data/local/tmp/
   adb push kernel_extras_part /data/local/tmp/
   adb push root_part /data/local/tmp/
   ```

3. Execute the following command inside `adb shell` to run the VM.

   ```bash
   /apex/com.android.virt/bin/vm run /data/local/tmp/vm_config.json
   ```

4. Execute the following command inside the VM's shell to shut it down.

   ```bash
   shutdown 0
   ```

## Running a VM with u-boot

### 1\. Download u-boot.bin

1. Using a web browser, go to the [aosp_u-boot-mainline Branch Grid](https://ci.android.com/builds/branches/aosp_u-boot-mainline/grid) website and select the latest build in the `u-boot_crosvm_aarch64` column.

   ![](/assets/img/posts/2025-08/0001.png)

2. In the new tab that opens, select the `u-boot.bin` file, and the browser will automatically download it.

### 2\. Download a Raw Disk Image for a Linux Distribution

The raw disk image used must contain the complete partitions for the Linux system. This article uses the Debian 13 `nocloud` cloud image, which you can also download yourself from [Debian Official Cloud Images](https://cloud.debian.org/images/cloud/).

```bash
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-nocloud-arm64.tar.xz
tar -xvf debian-13-nocloud-arm64.tar.xz
mv disk.raw debian-13-nocloud-arm64.raw
```

### 3\. (Optional) Modify the Image

1. Resize the partition in the image. If the host system is a Debian-based distribution, you can use the following commands.

   ```bash
   sudo apt update
   sudo apt install -y qemu-utils
   qemu-img resize -f raw debian-13-nocloud-arm64.raw 8G
   sudo losetup -fP debian-13-nocloud-arm64.raw
   # Use an application like KDE Partition Manager or GParted to resize the partition on /dev/loop0 (please determine the loop device number based on your actual setup)
   sudo losetup -d /dev/loop0
   ```

2. If the host system is not an `aarch64` architecture, you can use `qemu-system-aarch64` to boot the VM. If the host system is a Debian-based distribution, you can use the following commands.

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

3. After the VM boots, you can log in to the root account in QEMU's `serial0` view and make changes.

   ```bash
   # Upgrade packages
   apt update && apt upgrade -y
   # Install new packages as needed
   apt install -y vim nano wget curl git sysbench locales
   # Reconfigure locales
   dpkg-reconfigure locales
   ```

### 4\. Run the VM

1. Write a `u_boot_vm_config.json` file to launch the VM. You can use or refer to [u_boot_vm_config.json](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/configurations/avf/u-boot/u_boot_vm_config.json).

2. Send the relevant files to your Android device.

   ```bash
   adb push debian-13-nocloud-arm64.raw /data/local/tmp/
   adb push u-boot.bin /data/local/tmp/
   adb push u_boot_vm_config.json /data/local/tmp/
   ```

3. Execute the following command inside `adb shell` to run the VM.

   ```bash
   /apex/com.android.virt/bin/vm run /data/local/tmp/u_boot_vm_config.json
   ```

4. Execute the following command inside the VM's shell to shut it down.

   ```bash
   shutdown 0
   ```

## FAQ

### How does the VM connect to the network?

Due to Android's restrictions, creating a virtual network interface requires root access. If your Android device has root access, you can use [setup_network.sh](https://github.com/lfdevs/run-linux-on-android-guide/blob/main/scripts/avf/setup_network.sh) to create a virtual network interface named `crosvm_tap` for the VM.

1. Execute `setup_network.sh` on your Android device.

   ```bash
   wget https://github.com/lfdevs/run-linux-on-android-guide/raw/refs/heads/main/scripts/avf/setup_network.sh
   adb push setup_network.sh /data/local/tmp/
   adb root
   adb shell chmod +x /data/local/tmp/setup_network.sh
   adb shell /data/local/tmp/setup_network.sh
   adb unroot
   ```

2. Configure a static IP address inside the VM. If you are using Google's official Debian image or Debian's official `nocloud` cloud image, you can create a new configuration file `/etc/systemd/network/20-virtio-net.network` and add the following content.

   ```ini
   [Match]
   Name=eth0

   [Network]
   Address=192.168.1.2/24
   Gateway=192.168.1.1
   DNS=8.8.8.8
   DNS=1.1.1.1
   ```

3. Reload the `systemd-networkd` service to apply the new configuration.

   ```bash
   systemctl restart systemd-networkd
   ```

> This method is based on [this AVF document](https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/tags/android-15.0.0_r5/docs/custom_vm.md), and if you encounter any issues or have other solutions, feel free to discuss them in the [Issues](https://github.com/lfdevs/run-linux-on-android-guide/issues).
{: .prompt-info }

### How to use a desktop environment in the VM?

Due to Android's restrictions, both port forwarding and vsock communication between Android and the VM require root access. However, I do not have a suitable device for testing. If you have any ideas or solutions, feel free to discuss them in the [Issues](https://github.com/lfdevs/run-linux-on-android-guide/issues).

## References

  * [Android Virtualization Framework (AVF) overview](https://source.android.com/docs/core/virtualization)
  * [Virtualization - Android Code Search](https://cs.android.com/android/platform/superproject/main/+/main:packages/modules/Virtualization/)
  * [Custom VM](https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/tags/android-15.0.0_r5/docs/custom_vm.md)
