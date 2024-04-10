---
title: Custom Android Emulator kernel
date: 2024/4/8 22:53
tag: [虚拟化,Android系统,教程]
category: [虚拟化]
excerpt: 下载源码并编译Android emulator的内核，以x86 Android 9的4.4内核为例。
---

Take Android9 x86-64 kernel compilation as an example

## Download Source code
```
$ repo init --depth 1 -u https://mirrors.bfsu.edu.cn/git/AOSP/kernel/manifest \
  -b p-goldfish-android-goldfish-4.4-dev
$ repo sync -c --no-tags
```

## Custom config
```
$ BUILD_CONFIG=goldfish/build.config.goldfish.x86_64 LTO=thin build/config.sh
```
After executing config.sh, the **goldfish/arch/x86/configs/x86_64_ranchu_defconfig** file will be modified.

## Build
```
$ BUILD_CONFIG=goldfish/build.config.goldfish.x86_64 LTO=thin build/build.sh -j20
```
gen the kernel file: **./out/x86_64/dist/bzImage**

## Run
Run as previous post: [Install Android SDK and run emulator in linux](https://hqw700.github.io/2024/03/26/2024-03-26-linux-install-android-emulator/)  

```
$ emulator ... -qemu -kernel ./out/x86_64/dist/bzImage ...
```