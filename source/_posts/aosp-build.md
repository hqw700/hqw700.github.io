---
title: Pixel 3上Android 11源码下载，编译与烧录
date: 2021/1/1 20:46:25
tag: [工具使用]
excerpt: 本文主要介绍在Pixel 3机器上的AOSP 11源码的下载、编译与烧录。
---

研究Android系统最好有一套AOSP源码和一台可以编译运行的机器，下面是我Google Pixel 3上编译官方源码的过程记录。
## 一，源码下载
### 1.1 AOSP源码下载
从清华镜像站下载速度很快  
加 `--depth 1` 控制git深度可以极大节省硬盘空间。最终.repo目录21G左右  
``` bash
repo init --depth 1 -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-11.0.0_r1 
repo sync -j20
```

### 1.2 pixel 3专有硬件库下载
Android 11上不加硬件库会导致刷机后无限重启，但之前pixel 1时没有硬件库也能启动，这里踩了一个坑。
记得匹配好对应的BUILD_ID和代码, 查看build/core/build_id.mk文件，我的BUILD_ID=RP1A.200720.009，pixel 3的代号是blueline，所以对应的下载地址是：
https://developers.google.cn/android/drivers?hl=zh-cn#bluelinerp1a.200720.009
将两个文件下载到源码根目录，然后执行，会自动生成vendor文件
``` bash
~/code/android11$ tar zxvf google_devices-blueline-rp1a.200720.009-6cd41940.tgz
~/code/android11$ tar zxvf qcom-blueline-rp1a.200720.009-f772c38c.tgz
~/code/android11$ ./extract-google_devices-blueline.sh
~/code/android11$ ./extract-qcom-blueline.sh
```

## 二，编译
### 2.1 编译环境
编译环境搭建参考官方文档：https://source.android.com/setup/build/initializing
我用的win10应用市场下载的wsl2 ubuntu-18.04的虚拟机，挺好用的，不过比较耗内存，建议将内存加到32G

### 2.2 编译
i5-9600k的机器耗时3个小时，有点久。
``` bash
~/code/android11$ source build/envsetup.sh
~/code/android11$ lunch aosp_blueline-userdebug
~/code/android11$ make -j16
...
[ 99% 90776/91631] //frameworks/base/packages/SystemUI:SystemUI r8 [common]
Invalid descriptor (deserialized from Kotlin @Metadata): (LLandroid/animation/Animator;;)L;
Invalid descriptor (deserialized from Kotlin @Metadata): (LLandroid/animation/Animator;;)L;
[ 99% 91145/91631] //art/build/apex:art-check-debug-apex-gen generate art-check-debug-apex-gen.dummy
--bitness=auto, trying to autodetect. This may be incorrect!
  Detected multilib
[100% 91631/91631] build out/target/product/blueline/obj/PACKAGING/check-all-partition-sizes_intermediates/check_all_partition_sizes_log

#### build completed successfully (02:55:07 (hh:mm:ss)) ####

```

## 三，烧录
### 3.1 解锁OEM
在开发者选项里打开`OEM unlocking`选项，如果置灰连上外网看看。如果是有锁的机器尝试找第三方解锁。
### 3.2 bootloader解锁
``` bash
~/code/android11$ adb reboot bootloader
~/code/android11$ fashboot devices
xxxxxx
~/code/android11$ fastboot flashing unlock
~/code/android11$ fastboot reboot
```

### 3.3 烧录
``` bash
~/code/android11$ export ANDROID_PRODUCT_OUT=./out/target/product/blueline
~/code/android11$ fastboot flashall -w
...
File system type raw not supported.
Erasing 'metadata'                                 OKAY [  0.007s]
Erase successful, but not automatically formatting.
File system type raw not supported.
Rebooting                                          OKAY [  0.000s]
Finished. Total time: 101.511s

```
![pixel 3](/img/blog/pixel3_aosp.png) 
