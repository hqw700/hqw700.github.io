---
title: Pixel 3上Linux内核源码的下载，编译与烧录
date: 2021-01-02 16:05:16
tag: [Android系统, 教程, Linux]
category: [工具使用]
excerpt: (2021-11-10 重新编辑) 本文主要介绍在Pixel 3 Linux内核源码下载，编译和烧录。
---

官方教程地址https://source.android.com/setup/build/building-kernels, 已经写的很详细了, 由于翻译不及时，不同语言页面下显示的分支不一致，都可以使用。
## 一，源码下载
### 1.1 AOSP源码下载
依旧是将Google的地址`android.googlesource.com`替换清华的地址`aosp.tuna.tsinghua.edu.cn` 
加 `--depth 1` 控制git深度可以极大节省硬盘空间。最终.repo目录21G左右  
注：kernel分支不需要和aosp分支对应。
``` bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
repo init --depth 1 -u https://aosp.tuna.tsinghua.edu.cn/kernel/manifest -b android-msm-crosshatch-4.9-android11
repo sync -j20
```

## 二，编译
### 2.1 编译kernel
执行build/build.sh即可
``` bash
~/code/kernel$ ./build/build.sh
========================================================
 Copying modules files
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/extra/wlan.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/drivers/input/touchscreen/stm/ftm5.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/drivers/input/touchscreen/sec_ts/sec_touch.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/drivers/input/touchscreen/heatmap.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/drivers/media/v4l2-core/videobuf2-vmalloc.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/drivers/media/v4l2-core/videobuf2-memops.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/soc/pinctrl-wcd.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/ipc/wcd-dsp-glink.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/snd-soc-sdm845-max98927.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/codecs/snd-soc-wcd9xxx.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/codecs/snd-soc-cs35l36.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/codecs/wcd-core.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/codecs/snd-soc-wcd-spi.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/codecs/wcd934x/snd-soc-wcd934x.ko
  lib/modules/4.9.210-gdee0d123_audio-gee051ba/kernel/techpack/audio/asoc/snd-soc-sdm845.ko
========================================================
 Files copied to /home/huangqw/code/kernel/out/android-msm-pixel-4.9/dist
 ```
### 2.2 编译kernel报错
``` bash
/home/huangqw/code/kernel/private/msm-google/scripts/extract-cert.c:21:10: fatal error: 'openssl/bio.h' file not found
#include <openssl/bio.h>
         ^~~~~~~~~~~~~~~
1 error generated.
scripts/Makefile.host:101: recipe for target 'scripts/extract-cert' failed
make[3]: *** [scripts/extract-cert] Error 1
make[3]: *** Waiting for unfinished jobs....
```
解决：sudo apt-get install libssl-dev


### 2.3 打包boot.img
切换到android11源码目录，将TARGET_PREBUILT_KERNEL设置为kernel生成的Image.lz4文件的路径，官方文档提到的是Image.lz4-dtb，不用理会，android11生成的是Image.lz4。而且不单单是设置这个文件，整个out文件夹都会关联过去。  
可以用`get_build_var BOARD_VENDOR_KERNEL_MODULES`命令查看对应的编译环境变量
``` log
~/code/android11$ source build/envsetup.sh
~/code/android11$ lunch aosp_blueline-userdebug
~/code/android11$ export TARGET_PREBUILT_KERNEL=/home/huangqw/code/kernel/out/android-msm-pixel-4.9/dist/Image.lz4
~/code/android11$ make -j16 bootimage
  PRODUCT_SOONG_NAMESPACES=device/google/crosshatch hardware/google/av hardware/google/camera hardware/google/interfaces hardware/google/pixel hardware/qcom/sdm845 vendor/google/camera vendor/qcom/sdm845 vendor/google/interfaces vendor/qcom/blueline/proprietary
  ============================================
  Environment variable TARGET_PREBUILT_KERNEL was modified ( => /home/huangqw/code/kernel/out/android-msm-pixel-4.9/dist/Image.lz4), regenerating...
  Environment variable TARGET_PREBUILT_KERNEL was modified ( => /home/huangqw/code/kernel/out/android-msm-pixel-4.9/dist/Image.lz4), regenerating...
  [100% 222/222] Target boot image from recovery: out/target/product/blueline/boot.img

#### build completed successfully (01:02 (mm:ss)) ####
``` 
生成out/target/product/blueline/boot.img文件，如要重新编译原来的内核，将`export TARGET_PREBUILT_KERNEL=`置空重新编译就行


## 三，烧录
### 3.1 **【重要】** 先push生成的ko文件到/vendor/lib/modules下
```
adb root
adb remount -R
adb push ../kernel/out/android-msm-pixel-4.9/dist/*.ko /vendor/lib/modules
```

### 3.2 fastboot 更新bootimg
``` bash
adb reboot bootloader           ## 进入bootloader模式
fastboot boot boot.img          ## 从新内核启动，但不烧录，重启会失效
fastboot flash boot boot.img    ## 烧录新内核
```

### 3.3 烧录dtbo.img
如果修改了设备树，还需要烧录dtbo.img，生成在kernel\out\android-msm-pixel-4.9\dist\dtbo.img
``` bash
fastboot flash dtbo dtbo.img
```

## 四，验证
### 4.1 查看内核版本
手机内：
```
adb shell cat /proc/version
Linux version 4.9.223-g5bded8e40b62-ab6647920 (android-build@abfarm-us-west1-c-0040) (Android (6443078 based on r383902) clang version 11.0.1 (https://android.googlesource.com/toolchain/llvm-project b397f81060ce6d701042b782172ed13bee898b79)) #0 SMP PREEMPT Thu Jul 2 03:22:48 UTC 2020
android-msm-crosshatch-4.9-r-beta-3
```
编译生成的：
```
 grep -a 'Linux version' Image.lz4
 Linux version 4.9.232 ...
```

## 五，总结
本文成文于21年初，当时并没有研究linux内核的计划，花了一个下午时间简单尝试下，失败了就没继续，没想到本文对一些朋友造成困扰，Kyr1os同学还特地发邮件来提供正确的方法。现在重新操作了一遍，确认了方法可行。kernel分支直接使用官方页面上最新的就行，我重新编译使用的是android-msm-crosshatch-4.9-android12分支，可以正常运行。   
由于ko文件是打包进vendor.img的，每个分支官方都会提供。尝试了下，自己编译的vendor.img无法开机，解包后再打包也未成功，就不再折腾。如果需要提前预置好ko文件，可以打包进system.img或者在内核里将这些ko模块预置。


## 附: Android 7.1上编译kernel
之前在pixel 3 Android 11 折腾无果后，想起之前有在pixel 1 Android7.1上有编译过kernel, 电脑里也有环境，简单再尝试一下。
### 编译和下载
``` bash
mkdir kernel
git clone --depth 1 https://aosp.tuna.tsinghua.edu.cn/kernel/msm.git -b android-msm-marlin-3.18-nougat-mr2
cd msm
export PATH=$PATH:/home/huangqw/code/aosp/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-
export SUBARCH=arm64
make marlin_defconfig
make -j16
croot
source build/envsetup.sh
lunch aosp_sailfish-userdebug
export TARGET_PREBUILT_KERNEL=/home/huangqw/code/aosp/kernel/msm/arch/arm64/boot/Image.gz-dtb
make bootimage
```
### 烧录
``` bash
adb reboot bootloader 
fastboot flash boot boot.img 
```
### 重启后查看内核信息
烧录前  
```
$ adb shell cat /proc/version
Linux version 3.18.31-g086ab5b (android-build@vpbs30.mtv.corp.google.com) (gcc version 4.9.x 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Thu Feb 16 00:50:36 UTC 2017  
```
烧录后  
```
$ adb shell cat /proc/version
Linux version 3.18.31-g6309b4bd (huangqw@DESKTOP-14RLEFF) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Fri Jan 1 17:20:52 CST 2021
```


> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 3 Android 11  
> 源码版本：kernel android-msm-crosshatch-4.9-android11

> 参考1: [构建内核  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/setup/build/building-kernels)  
> 参考2: [Touchscreen not working after building and flashing clean msm kernel for the Google Pixel blueline](https://groups.google.com/g/android-building/c/ou630PviyDc)