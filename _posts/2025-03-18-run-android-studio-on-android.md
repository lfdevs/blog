---
title: 安卓手机/平板运行 Android Studio 并打包应用、真机调试的笔记
author: lfdevs
date: 2025-03-18 10:38:00 +0800
categories: [Android]
tags: [termux, proot, chroot, android, development]
---
> 本文于 2025 年 3 月 18 日在 Bilibili 发布：[https://www.bilibili.com/opus/1045510823576862721](https://www.bilibili.com/opus/1045510823576862721)
{: .prompt-tip }

{% include embed/bilibili.html id='BV1wLXTYhEkD' %}

一般来说，通过 Android Termux 部署 Linux 容器运行原生的 arm64 应用可以做到即装即用，问题很少。但 Android Studio 基于 IDEA 社区版，居然只有 Linux x86 版本，而没有 Linux arm64 版本。不过它的绝大部分组件都是基于 Java 的，我们可以通过修改一些文件和配置使它可以在 Linux arm64 的容器中运行。除了 XML 布局预览器、模拟器外，其他功能应该都可以正常使用。

> 本文的 Linux 容器环境基于 UP 前一篇笔记（[高通平台下 Termux Chroot/Proot 容器实现全局硬件加速的笔记](https://blog.lfdevs.com/posts/hardware-acceleration-in-termux-chroot-proot-with-turnip-zink/)​）的配置，但理论上同时适用于 Proot 和 Chroot 容器，而且 GPU 加速开启与否都行。
{: .prompt-tip }

## 一、安装步骤

首先安装一下必要的工具。

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget ark -y
```

下载 Android Studio、SDK、NDK、模拟器补丁。

```bash
cd ~/下载
wget https://redirector.gvt1.com/edgedl/android/studio/ide-zips/2024.2.2.15/android-studio-2024.2.2.15-linux.tar.gz https://github.com/lzhiyong/android-sdk-tools/releases/download/35.0.2/android-sdk-tools-static-aarch64.zip https://github.com/lzhiyong/android-sdk-tools/releases/download/34.0.3/android-sdk-tools-static-aarch64.zip https://github.com/lzhiyong/termux-ndk/releases/download/android-ndk/android-ndk-r27b-aarch64.zip https://cache-redirector.jetbrains.com/intellij-jbr/jbr_jcef-21.0.5-linux-aarch64-b750.29.tar.gz
```

由于在 Termux 的 Linux 容器里很难运行真正的 Android 模拟器，故使用 [Victor Laureano 大佬](https://stackoverflow.com/users/5453590/victor-laureano) 提供的补丁文件使 Android Studio 误认为已安装模拟器，避免报错。由于补丁文件的链接在[原回答](https://stackoverflow.com/a/77020732/31562594)中已失效，故在这里直接提供：[emulator-ultracompact.zip](/assets/attachment/2025-03/emulator-ultracompact.zip)

![](/assets/img/posts/2025-03/0011.jpg)

解压 Android Studio 安装包。

```bash
mkdir ~/Android
tar -zxvf ./android-studio-2024.2.2.15-linux.tar.gz -C ~/Android/
```

使用 arm64 架构的 JBR（Android Studio 内置的 OpenJDK 21 运行环境） 的覆盖原本的 x86 架构的 JBR。

```bash
tar -zxvf ./jbr_jcef-21.0.5-linux-aarch64-b750.29.tar.gz
cp -f ./jbr_jcef-21.0.5-linux-aarch64-b750.29 ~/Android/android-studio/jbr/
```

添加相关的环境变量。编辑 `~/.profile` 文件，在文件末尾新增一行，添加如下的配置。

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export ANDROID_USER_HOME=$HOME/.android
export ANDROID_EMULATOR_HOME=$ANDROID_USER_HOME
export ANDROID_AVD_HOME=$ANDROID_EMULATOR_HOME/avd/
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/tools/bin:$ANDROID_HOME/platform-tools
```

使配置生效。

```bash
source ~/.profile
```

执行以下命令启动 Android Studio。可以在桌面添加启动文件，方便以后打开。

```bash
~/Android/android-studio/bin/studio.sh
```

![](/assets/img/posts/2025-03/0012.jpg)

## 二、SDK、NDK配置

第一次打开 Android Studio 会自动弹出安装向导窗口，选择 *“自定义”* 的选项，取消勾选 Android Virtual Device，我们后面会通过补丁文件解决模拟器报错的问题。

![](/assets/img/posts/2025-03/0013.jpg)

![](/assets/img/posts/2025-03/0014.jpg)

![](/assets/img/posts/2025-03/0015.jpg)

安装向导完成后，用 Ark 打开模拟器补丁文件 `emulator-ultracompact.zip`，将其中的 `emulator` 目录解压到 `~/Android/Sdk` 目录中。

![](/assets/img/posts/2025-03/0016.jpg)

用 Ark 打开 **v34.0.3** 版本的 `android-sdk-tools-static-aarch64.zip`，将其中的 `platform-tools` 目录解压并覆盖到 `~/Android/Sdk` 目录中。v35.0.2 版本的 `adb` 有点问题，所以不推荐使用该版本的 `platform-tools`。

![](/assets/img/posts/2025-03/0017.jpg)

![](/assets/img/posts/2025-03/0018.jpg)

在 Android Studio 的欢迎窗口打开 SDK Manager，勾选自己需要的 Build-Tools 和 NDK，注意版本对应，因为后面要替换相应的文件。例如 `27.1.12297006` 版本对应的 NDK 为 `r27b`，lzhiyong 大佬编译的 arm64 原生 NDK 恰好有这个版本。

![](/assets/img/posts/2025-03/0019.jpg)

![](/assets/img/posts/2025-03/0020.jpg)

![](/assets/img/posts/2025-03/0021.jpg)

SDK Manager 保存设置并下载文件完毕后，效仿前面替换 `platform-tools` 的操作，分别替换对应版本的 `build-tools` 和 `ndk` 文件。两者也均在 `~/Android/Sdk` 目录中。

![](/assets/img/posts/2025-03/0022.jpg)

替换完成后可删除`~/下载`目录内前面下载的安装包等多余的文件，以节省存储空间。

新建一个项目进入 Android Studio 的主界面。此时若出现软件窗口的标题栏不显示的问题，可以按 **Ctrl + Alt + S** 快捷键打开设置，关闭 *Appearance & Behavior -> Appearance -> UI Options -> Merge main menu with window title* 选项，然后重启软件。若仍未解决，可删除当前版本的 Android Studio 和 `~/.config/Google` 目录下对应版本的配置目录，安装 [`2023.3.1.20` 版本](https://redirector.gvt1.com/edgedl/android/studio/ide-zips/2023.3.1.20/android-studio-2023.3.1.20-linux.tar.gz)。安装完成后注意重新覆盖 JBR。

![](/assets/img/posts/2025-03/0023.jpg)

![](/assets/img/posts/2025-03/0024.jpg)

在项目根目录的 `gradle.properties` 文件添加如下的配置，指定 SDK 和 NDK 的路径。注意将 `<你的用户名>` 和 `<覆盖过文件的版本>` 替换为自己的，`<覆盖过文件的版本>` 应与项目 `app/build.gradle` 的 `compileSdk` 相对应。

```ini
sdk.dir=/home/<你的用户名>/Android/Sdk
ndk.dir=/home/<你的用户名>/Android/Sdk/ndk/27.1.12297006
android.aapt2FromMavenOverride=/home/<你的用户名>/Android/Sdk/build-tools/<覆盖过文件的版本>/aapt2
```

添加配置后重新执行同步操作，若下载速度慢，可换国内镜像源，参考教程：[Android Studio 2024版本新建项目换源教程](https://blog.csdn.net/github_74110837/article/details/143092695)

## 三、运行、调试应用

在安卓设置的 *“开发者选项 -> 调试 -> 无线调试”* （需要安卓版本为 11 或以上）中打开无线调试的开关，点击 *“使用配对码配对设备”* 。让弹出来的配对对话框**保持开启**，通过小窗或分屏打开 Termux 或 Termux:X11 ，在 Linux 容器内使用 `adb pair 127.0.0.1:<配对的端口号>` 命令进行配对。输入配对码成功后，可使用 `adb devices` 命令查看是否连接成功。若没有列出自己的设备，可关闭配对对话框，使用 `adb connect 127.0.0.1:<无线调试的端口号>` 命令进行连接。注意两个端口号的区别，配对成功后只需无线调试的端口号进行连接，无需再次配对。

若使用 `adb pair 127.0.0.1:<配对的端口号>` 命令时报错 `adb: unknown command pair`，而且使用 `adb --version` 命令时有输出类似于以下的信息，则你很有可能安装了 Debian 软件源里较低版本的 `adb` （`adb pair` 命令需要 `adb` 版本 30 及以上），可以使用命令 `sudo apt purge adb -y` 卸载该版本的 `adb` 即可正常使用 `adb pair` 命令。

```
$ adb --version
Android Debug Bridge version 1.0.41
Version 29.0.6-debian
Installed as /usr/lib/android-sdk/platform-tools/adb
```

关于安卓项目的运行、调试、打包成 APK 文件等方法，直接参考 PC 版本的教程即可，本文不再赘述。

## 相关资源

- android-sdk-tools (lzhiyong)：[https://github.com/lzhiyong/android-sdk-tools](https://github.com/lzhiyong/android-sdk-tools)
- termux-ndk (lzhiyong)：[https://github.com/lzhiyong/termux-ndk](https://github.com/lzhiyong/termux-ndk)
- Android Studio 历史版本存档（网站语言切换为英文可正常显示）：[https://developer.android.google.cn/studio/archive](https://developer.android.google.cn/studio/archive)
- JetBrainsRuntime：[https://github.com/JetBrains/JetBrainsRuntime](https://github.com/JetBrains/JetBrainsRuntime)

## 参考资料

- [使用Termux安装xfce4桌面, Android Studio, Code Server(VSCode) - Alain's Blog](https://www.alainlam.cn/?p=859)
- [Does an Android Studio Linux arm64 version exist? - Stack Overflow](https://stackoverflow.com/questions/71067886/does-an-android-studio-linux-arm64-version-exist/77020732#77020732)
