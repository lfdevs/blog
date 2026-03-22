---
title: 高通骁龙平台下 Termux Chroot/PRoot 容器实现全局硬件加速的笔记
author: lfdevs
date: 2025-03-09 22:28:00 +0800
categories: [Android]
tags: [termux, linux, proot, chroot, turnip, zink]
---
> 本文于 2025 年 3 月 9 日在 Bilibili 发布：[https://www.bilibili.com/opus/1042342069764358179](https://www.bilibili.com/opus/1042342069764358179)
{: .prompt-tip }

{% include embed/bilibili.html id='BV1qVRsYkEDs' %}

笔者接触 [Termux](https://termux.dev/cn/index.html) 这款强大的 Android 虚拟终端大概已有两年半，虽相关的学习多浮于表面，但仍摸索了一些玩法，利用它运行 Linux 容器便是其中之一。由于相关的知识技巧比较零碎，故总结了这篇笔记，记录一下关键步骤和常见问题的解决办法，后面如果解决了什么新的问题也会更新在这里。

> 本文仅涉及使用 Termux 运行**原生 arm64 架构的 Linux 容器**，不涉及跨架构模拟、图形 API 转译等内容（Windows 游戏转译可以选择 [Winlator](https://winlator.org/)、[Mobox](https://github.com/olegos2/mobox) 等）。本文使用的驱动为 **Turnip + Zink**。
{: .prompt-tip }

## 一、Termux 配置

Termux 利用 [TMOE](https://gitee.com/mo2/linux) 部署 Linux 容器的教程很多，这里就不在赘述。笔者部署时选的是 Debian 13 的 Chroot 容器，安装了 KDE Plasma 桌面。没 Root 的话也可以选择 PRoot 容器，也可以正常使用（笔者在骁龙 888 、骁龙 8+ Gen 1、骁龙 8 Elite 和 骁龙 8 Elite Gen 5 的 PRoot 容器上测试成功）。

如果是 Android 12 及以上版本的系统，需要解除单个应用的多线程限制，在 TMOE 里可以快捷设置。

![](/assets/img/posts/2025-03/0001.jpg)

安装 Linux 容器过程中可以不安装 vnc 服务（性能不太行），因为本文用的是 [Termux:X11](https://github.com/termux/termux-x11) 连接容器内的桌面环境。KDE Plasma 桌面环境在**容器内**使用 TMOE 安装即可。安装完成后需要挂载 tmp，关闭容器后在**容器外**的 TMOE 进行操作。

![](/assets/img/posts/2025-03/0002.jpg)

![](/assets/img/posts/2025-03/0003.jpg)

## 二、Termux:X11 配置

安装 Termux:X11 后，建议先设置 Preferences。若连接键鼠使用，建议修改图中橙色方框中的选项为图中所示配置；若需要在 Linux 容器中正常使用 Alt + Tab 等快捷键，需要在安卓设置中打开 Termux:X11 的无障碍服务，如图中绿色方框中所示。

![](/assets/img/posts/2025-03/0004.jpg)

![](/assets/img/posts/2025-03/0005.jpg)

![](/assets/img/posts/2025-03/0006.jpg)

设置好后在 Termux 中安装 X11 源。

```bash
pkg install x11-repo -y
```

安装好后可以通过 `termux-change-repo` 命令更换国内的软件源以加快下载速度，然后安装连接 Termux:X11 和硬件加速相关的软件包。

```bash
pkg update && pkg upgrade -y
```

## 三、Linux 容器配置
在容器外，先启动 `termux-x11` 服务，再启动容器，可以参考以下脚本：

```bash
#!/bin/bash
kill -9 $(pgrep -f "termux.x11") 2>/dev/null
termux-x11 :2 -dpi 96 &
tmoe ls
```

进入容器后，不要启动 KDE 桌面会话，先使用 `vim` 等工具编辑 `~/.config/kwinrc` 文件（若用户目录下 `.config` 文件夹不存在需要先新建），禁用 KDE 的桌面特效，在 `[Compositing]` 部分添加或修改以下内容：

```ini
[Compositing]
Enabled=false
```

然后安装相关驱动及工具（假设安装的是 Debian 13）：

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget mesa-utils vulkan-tools -y
wget https://github.com/lfdevs/mesa-for-android-container/releases/download/debian%2F25.0.7-2-turnip/mesa-vulkan-drivers_25.0.7-2+deb13u1_arm64_unpatched.deb
sudo apt install --reinstall ./mesa-vulkan-drivers_25.0.7-2+deb13u1_arm64_unpatched.deb -y
sudo apt-mark hold mesa-vulkan-drivers:arm64
rm ./mesa-vulkan-drivers_25.0.7-2+deb13u1_arm64_unpatched.deb
```

> 若手机的 SoC 为骁龙 8 Elite 等更新的处理器，则上面的驱动无法正常使用，可以前往笔者的这个仓库下载最新的驱动（安装使用方法请查看仓库的 README）：[lfdevs/mesa-for-android-container: A Mesa build for containers on Android (Proot, Chroot, LXC, etc.), to support hardware acceleration with Adreno GPU.](https://github.com/lfdevs/mesa-for-android-container)
{: .prompt-info }

使用 root 权限修改 `/etc/profile` 文件，在文件末尾添加以下的两行内容，设置 `XDG_RUNTIME_DIR` （该环境变量在正常的 Linux 系统上由 `systemd` 自动设置，但当宿主系统为 Android 时，Linux 容器（Chroot/PRoot）一般无法使用 systemd，故需手动设置）和 `DISPLAY` （与上面启动 `termux-x11` 服务命令的`:2`相对应）全局环境变量：

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DISPLAY=':2'
```

使环境变量生效：

```bash
source /etc/profile
```

创建目录：

```bash
sudo mkdir -p /run/user/$(id -u)
sudo chown $USER:$USER /run/user/$(id -u)
chmod 700 /run/user/$(id -u)
```

修改 /tmp 目录的权限，避免出现因权限问题导致的桌面会话启动失败的问题：

```bash
sudo chmod -R 777 /tmp
```

使用以下命令启动 KDE Plasma 的 X11 会话：

```bash
sudo service dbus start
MESA_LOADER_DRIVER_OVERRIDE=zink TU_DEBUG=noconform /etc/X11/xinit/Xsession
```

若不希望进入桌面会话时自动启动终端，可以编辑 `/etc/X11/xinit/Xsession` 文件，将 `open_terminal` 这行注释掉。

切换到 Termux:X11 应用（注意防止 Termux 掉后台）：

![](/assets/img/posts/2025-03/0007.jpg)

![](/assets/img/posts/2025-03/0008.jpg)

![](/assets/img/posts/2025-03/0009.jpg)

![](/assets/img/posts/2025-03/0010.jpg)

若图标、字体等元素太小，可以在显示设置中调整缩放倍数。调整后点击“应用”不会立即生效，需要注销后切回到 Termux 按 Enter 或 Ctrl + C 键结束 KDE 会话，然后使用前面的命令再次启动 KDE Plasma 的 X11 会话。

## 四、常见问题

### 1\. 安装 fcitx 5 和拼音输入法

```bash
sudo apt install fcitx5 fcitx5-pinyin fcitx5-module-cloudpinyin fcitx5-frontend-all fcitx5-chinese-addons -y
```

### 2\. 在 Chromium、VS Code 等“浏览器套壳”应用无法激活 fcitx 5 输入法

按照本文的启动命令启动 KDE Plasma 一般不会出现此问题，若仍出现此问题，可尝试使用 root 权限修改 `/etc/environment` 文件，添加以下的四行内容：

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
```

注销用户，切回到 Termux 按 Enter 或 Ctrl + C 键结束 KDE 会话，然后使用前面的命令再次启动 KDE Plasma 的 X11 会话。

### 3\. 一些应用程序调用 KDE 桌面的接口打开文件夹时，只能打开文件

这个疑似是 arm64 架构独有的 KDE 桌面问题，可以先将要选择的文件夹改为其他名称，然后再新建一个**与文件夹原名称同名的空文件**，选择完成后再删除空文件，将文件夹的名称改回来。

> 请**谨慎修改系统文件夹**（如 `/usr/bin` 等）的名称，以免出现不可预知的问题。
{: .prompt-danger }

### 4\. 使用较新版本的 Termux:X11 时，默认使用 Android 输入法，而不是容器内的输入法

开启 Termux:X11 Preferences 的 **Keyboard -> Enforce char based input** 选项。

### 5\. 启动 KDE Plasma 6 桌面时，屏幕左上角会出现一个白色矩形

这是进程 `xwaylandvideobridge` 导致的，使用以下命令结束它即可（Termux 容器中它没用）。

```bash
pkill -f xwaylandvideobridge
```

如果需要在启动桌面的时候自动结束这个进程，则可以新建一个 Shell 脚本，填入以下内容，并**赋予它执行权限**。然后在“系统设置 -> 系统 -> 自动启动”中添加它为登录脚本，添加后点击该项的“查看属性”图标，选择属性窗口的“权限”选项卡，勾选“**允许将文件作为程序执行**”。可以根据桌面启动的实际情况调整延时执行该命令的秒数（即 `sleep 5` 一行）。

```bash
#!/bin/bash
sleep 5
pkill -f xwaylandvideobridge
```

## 相关资源

- [Termux](https://github.com/termux/termux-app)
- [Termux:X11](https://github.com/termux/termux-x11)
- [TMOE 文档](https://doc.tmoe.me/zh/)
- [Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/)

## 参考资料

- [WIP: freedreno/drm: Add KGSL backend for freedreno (!21570) · Merge requests · Mesa / mesa · GitLab](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21570)
- [experimental dri3 support for turnip kgsl on Termux:X11 by xMeM · Pull Request #17561 · termux/termux-packages](https://github.com/termux/termux-packages/pull/17561)
- [[not for merging] enhance(main/mesa): Lucas Fryzek and xMeM's freedreno kgsl for `mesa` 25.1.1 by robertkirkman · Pull Request #24840 · termux/termux-packages](https://github.com/termux/termux-packages/pull/24840)
- [使用Termux-11出错 · Issue #IAZWYG · Moe/TMOE - Gitee.com](https://gitee.com/mo2/linux/issues/IAZWYG)
- [Termux-Desktops/Documentation/HardwareAcceleration.md at main · LinuxDroidMaster/Termux-Desktops](https://github.com/LinuxDroidMaster/Termux-Desktops/blob/main/Documentation/HardwareAcceleration.md)
- [termux-desktop/docs/hw-acceleration.md at main · sabamdarif/termux-desktop](https://github.com/sabamdarif/termux-desktop/blob/main/docs/hw-acceleration.md)
- [I can't control Minecraft in Termux-x11 with mouse and keyboard (Bluetooth mouse and keyboard). : r/termux](https://www.reddit.com/r/termux/comments/1g799ey/i_cant_control_minecraft_in_termuxx11_with_mouse/)
- [PRoot Linux only: DRI3 patch mesa Turnip driver new build "Adreno 750" surport : r/termux](https://www.reddit.com/r/termux/comments/19dpqas/proot_linux_only_dri3_patch_mesa_turnip_driver/)
- [解决搜狗拼音输入法在vscode中输入不了汉字的问题_vscode中使用中文输入法失效-CSDN博客](https://blog.csdn.net/wyw1749750673/article/details/144372483)
- [绫袅LingNc_的评论​](https://www.bilibili.com/video/BV1wjtizxE58?comment_on=1&comment_root_id=271040609345&comment_secondary_id=275602531729#reply275602531729)
