---
title: Android 终端 App 笔记
author: lfdevs
date: 2026-03-03 12:00:00 +0800
categories: [Android]
tags: [avf, linux, terminal, gpu, virgl]
---
## 兼容性
### 桌面环境

> 本节使用 Pixel 6 Pro（SoC 为 Tensor G1，GPU 为 Mali-G78）测试，系统版本为`BP4A.251205.006`，终端 App 启用 GPU 硬件加速。不同设备的情况可能会有差异。
{: .prompt-info }

|            | Wayland | X11 |
| :--------: | :-----: | :-: |
|   Weston   |   ✔支持   | 不适用 |
|   Xfce 4   |   不适用   | ✔支持 |
|   GNOME    |  ❌不支持   | ✔支持 |
| KDE Plasma |  ❌不支持   | ✔支持 |

## 使用技巧
### 调整虚拟机的运行内存

为`droid`用户添加`sudo`权限后，或者使用`root`账户，在**虚拟机内**编辑`/mnt/internal/linux/vm_config.json`文件，修改`memory_mib`字段的值，默认为`4096`（即 4 GiB）。可以视宿主机（即 Android 设备）物理运行内存的大小适度调整。

```json
    "memory_mib": 4096
```

### 启用 GPU 硬件加速

目前， Android 的终端 App 支持使用`virglrenderer`后端进行硬件加速。但由于宿主机 ANGLE 的限制，支持的图形 API 版本通常较低。以 Pixel 6 Pro 为例，启用 GPU 硬件加速后，最高仅支持到 OpenGL 2.1 和 OpenGL ES 3.1。可以通过指定环境变量`MESA_GL_VERSION_OVERRIDE=4.6`来强制启动一些要求高版本 OpenGL 的应用（如 Blender 3.x、Minecraft Java Edition），但稳定性可能较差。

为`droid`用户添加`sudo`权限后，或者使用`root`账户，在**虚拟机内**编辑`/mnt/internal/linux/vm_config.json`文件，针对`gpu`字段做如下的修改。GPU 硬件加速将在重启虚拟机后生效。

```diff
     "gpu": {
-        "backend": "2d"
+        "backend": "virglrenderer",
+        "context_types": ["virgl2"]
     }
```

启用 GPU 硬件加速后，Pixel 6 Pro `glmark2`与`glmark2-es2`基准测试的分数在 500 分左右。

<details  markdown="1">
  <summary>基准测试详细结果</summary>

**glmark2:**

```
droid@debian:~$ glmark2
=======================================================
    glmark2 2023.01
=======================================================
    OpenGL Information
    GL_VENDOR:      Mesa
    GL_RENDERER:    virgl (Mali-G78)
    GL_VERSION:     2.1 Mesa 25.0.7-2~bpo12+1
    Surface Config: buf=32 r=8 g=8 b=8 a=8 depth=24 stencil=0 samples=0
    Surface Size:   800x600 windowed
=======================================================
[build] use-vbo=false: FPS: 600 FrameTime: 1.667 ms
[build] use-vbo=true: FPS: 888 FrameTime: 1.127 ms
[texture] texture-filter=nearest: FPS: 781 FrameTime: 1.281 ms
[texture] texture-filter=linear: FPS: 388 FrameTime: 2.582 ms
[texture] texture-filter=mipmap: FPS: 833 FrameTime: 1.202 ms
[shading] shading=gouraud: FPS: 699 FrameTime: 1.432 ms
[shading] shading=blinn-phong-inf: FPS: 671 FrameTime: 1.491 ms
[shading] shading=phong: FPS: 662 FrameTime: 1.512 ms
[shading] shading=cel: FPS: 240 FrameTime: 4.174 ms
[bump] bump-render=high-poly: FPS: 458 FrameTime: 2.184 ms
[bump] bump-render=normals: FPS: 793 FrameTime: 1.262 ms
[bump] bump-render=height: FPS: 820 FrameTime: 1.221 ms
[effect2d] kernel=0,1,0;1,-4,1;0,1,0;: FPS: 689 FrameTime: 1.451 ms
[effect2d] kernel=1,1,1,1,1;1,1,1,1,1;1,1,1,1,1;: FPS: 338 FrameTime: 2.962 ms
[pulsar] light=false:quads=5:texture=false: FPS: 828 FrameTime: 1.208 ms
[desktop] blur-radius=5:effect=blur:passes=1:separable=true:windows=4: FPS: 216 FrameTime: 4.638 ms
[desktop] effect=shadow:windows=4: FPS: 503 FrameTime: 1.990 ms
[buffer] columns=200:interleave=false:update-dispersion=0.9:update-fraction=0.5:update-method=map: FPS: 225 FrameTime: 4.463 ms
[buffer] columns=200:interleave=false:update-dispersion=0.9:update-fraction=0.5:update-method=subdata: FPS: 195 FrameTime: 5.139 ms
[buffer] columns=200:interleave=true:update-dispersion=0.9:update-fraction=0.5:update-method=map: FPS: 250 FrameTime: 4.014 ms
[ideas] speed=duration: FPS: 397 FrameTime: 2.519 ms
[jellyfish] <default>: FPS: 628 FrameTime: 1.593 ms
[terrain] <default>: FPS: 94 FrameTime: 10.693 ms
[shadow] <default>: FPS: 298 FrameTime: 3.366 ms
[refract] <default>: FPS: 250 FrameTime: 4.016 ms
[conditionals] fragment-steps=0:vertex-steps=0: FPS: 735 FrameTime: 1.362 ms
[conditionals] fragment-steps=5:vertex-steps=0: FPS: 754 FrameTime: 1.327 ms
[conditionals] fragment-steps=0:vertex-steps=5: FPS: 783 FrameTime: 1.278 ms
[function] fragment-complexity=low:fragment-steps=5: FPS: 742 FrameTime: 1.348 ms
[function] fragment-complexity=medium:fragment-steps=5: FPS: 724 FrameTime: 1.382 ms
[loop] fragment-loop=false:fragment-steps=5:vertex-steps=5: FPS: 783 FrameTime: 1.278 ms
[loop] fragment-steps=5:fragment-uniform=false:vertex-steps=5: FPS: 749 FrameTime: 1.337 ms
[loop] fragment-steps=5:fragment-uniform=true:vertex-steps=5: FPS: 714 FrameTime: 1.402 ms
=======================================================
                                  glmark2 Score: 566 
=======================================================
```

**glmark2-es2:**

```
droid@debian:~$ glmark2-es2
=======================================================
    glmark2 2023.01
=======================================================
    OpenGL Information
    GL_VENDOR:      Mesa
    GL_RENDERER:    virgl (Mali-G78)
    GL_VERSION:     OpenGL ES 3.1 Mesa 25.0.7-2~bpo12+1
    Surface Config: buf=32 r=8 g=8 b=8 a=8 depth=32 stencil=0 samples=0
    Surface Size:   800x600 windowed
=======================================================
[build] use-vbo=false: FPS: 570 FrameTime: 1.757 ms
[build] use-vbo=true: FPS: 786 FrameTime: 1.272 ms
[texture] texture-filter=nearest: FPS: 767 FrameTime: 1.304 ms
[texture] texture-filter=linear: FPS: 783 FrameTime: 1.278 ms
[texture] texture-filter=mipmap: FPS: 793 FrameTime: 1.261 ms
[shading] shading=gouraud: FPS: 240 FrameTime: 4.176 ms
[shading] shading=blinn-phong-inf: FPS: 685 FrameTime: 1.462 ms
[shading] shading=phong: FPS: 313 FrameTime: 3.200 ms
[shading] shading=cel: FPS: 644 FrameTime: 1.555 ms
[bump] bump-render=high-poly: FPS: 461 FrameTime: 2.170 ms
[bump] bump-render=normals: FPS: 816 FrameTime: 1.226 ms
[bump] bump-render=height: FPS: 799 FrameTime: 1.252 ms
[effect2d] kernel=0,1,0;1,-4,1;0,1,0;: FPS: 735 FrameTime: 1.362 ms
[effect2d] kernel=1,1,1,1,1;1,1,1,1,1;1,1,1,1,1;: FPS: 457 FrameTime: 2.188 ms
[pulsar] light=false:quads=5:texture=false: FPS: 765 FrameTime: 1.308 ms
[desktop] blur-radius=5:effect=blur:passes=1:separable=true:windows=4: FPS: 215 FrameTime: 4.655 ms
[desktop] effect=shadow:windows=4: FPS: 488 FrameTime: 2.050 ms
[buffer] columns=200:interleave=false:update-dispersion=0.9:update-fraction=0.5:update-method=map: FPS: 226 FrameTime: 4.433 ms
[buffer] columns=200:interleave=false:update-dispersion=0.9:update-fraction=0.5:update-method=subdata: FPS: 186 FrameTime: 5.404 ms
[buffer] columns=200:interleave=true:update-dispersion=0.9:update-fraction=0.5:update-method=map: FPS: 244 FrameTime: 4.115 ms
[ideas] speed=duration: FPS: 398 FrameTime: 2.519 ms
[jellyfish] <default>: FPS: 659 FrameTime: 1.518 ms
[terrain] <default>: FPS: 93 FrameTime: 10.827 ms
[shadow] <default>: FPS: 473 FrameTime: 2.117 ms
[refract] <default>: FPS: 251 FrameTime: 3.987 ms
[conditionals] fragment-steps=0:vertex-steps=0: FPS: 746 FrameTime: 1.342 ms
[conditionals] fragment-steps=5:vertex-steps=0: FPS: 681 FrameTime: 1.470 ms
[conditionals] fragment-steps=0:vertex-steps=5: FPS: 778 FrameTime: 1.286 ms
[function] fragment-complexity=low:fragment-steps=5: FPS: 772 FrameTime: 1.296 ms
[function] fragment-complexity=medium:fragment-steps=5: FPS: 662 FrameTime: 1.511 ms
[loop] fragment-loop=false:fragment-steps=5:vertex-steps=5: FPS: 58 FrameTime: 17.287 ms
[loop] fragment-steps=5:fragment-uniform=false:vertex-steps=5: FPS: 1 FrameTime: 24287.644 ms
[loop] fragment-steps=5:fragment-uniform=true:vertex-steps=5: FPS: 1 FrameTime: 3886.317 ms
=======================================================
                                  glmark2 Score: 500 
=======================================================
```

</details>

## 常见问题
### 分辨率

终端 App 的虚拟显示器目前可能会固定为某个特定的分辨率，无法修改，如`1280x720`。根据 AOSP 的源码来看，分辨率的值目前是写死在源代码里的，无法通过常规办法修改。
### 操作系统

终端 App 的虚拟机目前使用的操作系统为 Debian 12，不支持更换。根据 AOSP 的源码来看，该系统经过了一定程度的定制，修改了内核源码，增加了一些与 AVF 有关的服务等。

升级 Debian 的版本（如升级到 Debian 13）也会导致终端 App 无法启动，可能是因为在全量升级的过程中更新了内核。
### 设置时区

终端 App 的默认时区为`UTC`，可以参考以下命令修改时区。

```bash
sudo timedatectl set-timezone Asia/Shanghai
```

## 参考资料

- [Android 虚拟化框架 (AVF) 概览 - Android Open Source Project](https://source.android.com/docs/core/virtualization)
- [Virtualization - Android Code Search](https://cs.android.com/android/platform/superproject/+/android-latest-release:packages/modules/Virtualization/)
