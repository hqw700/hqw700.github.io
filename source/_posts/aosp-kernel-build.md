---
title: Pixel 3上Linux内核源码的下载，编译与烧录
date: 2021-01-02 16:05:16
tag: [Android系统, 教程, Linux]
category: [工具使用]
excerpt: 本文主要介绍在Pixel 3 Linux内核的编译和烧录, 由于内核版本库没有对应上的原因，最终以失败告终。最后介绍了Pixel 1上编译内核的方法和流程。
---

> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 3 Android 11  
> 源码版本：kernel android-msm-crosshatch-4.9-android11

官方教程地址https://source.android.com/setup/build/building-kernels#id-version, 已经写的很详细了, pixel 3建议下载的是android-msm-crosshatch-4.9-android11分支（此处是个坑，后面会提到）
## 一，源码下载
### 1.1 AOSP源码下载
依旧是将Google的地址`android.googlesource.com`替换清华的地址`aosp.tuna.tsinghua.edu.cn` 
加 `--depth 1` 控制git深度可以极大节省硬盘空间。最终.repo目录21G左右  
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
``` bash
adb reboot bootloader           ## 进入bootloader模式
fastboot boot boot.img          ## 从新内核启动，但不烧录，重启会失效
fastboot flash boot boot.img    ## 烧录新内核
```

## 四，错误
经过上面的步骤，烧录的内核无法正常启动，无限重启，短暂的logcat中显示内核版本不对。
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
### 4.2 切换对应的内核版本
https://android.googlesource.com/kernel/msm/+/5bded8e40b62

根据版本号里的git id提示，找到该版本对应的分支[https://android.googlesource.com/kernel/msm/+/refs/heads/android-msm-crosshatch-4.9-r-beta-3], 切换过去编译，烧录之后系统可以起来了，但是有很多外设报错：蓝牙，音频等，界面黑屏。点开应用之后系统就重启了。后面也尝试了其它的分支版本，依旧是不能正常运行。  

## 五，Android 7.1上编译kernel
在pixel 3 Android 11 折腾无果后，想起之前有在pixel 1 Android7.1上有编译过kernel, 电脑里也有环境，简单再尝试一下。
### 5.1 编译和下载
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
### 5.2 烧录
``` bash
adb reboot bootloader 
fastboot flash boot boot.img 
```
### 5.3 重启后查看内核信息
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
## 六，问题总结

pixel 1上采用的是旧的编译内核方式，直接下载对应的git节点编译即可。pixel 3没有了kernel defconfig文件，无法采用这个方式，但同样是切换到目前内核所对应的git节点，依旧也没有成功，估计是在新编译方式操作上，漏了一些。   
目前我Android 11版本是比较早期的android-11.0.0_r1，而kernel只有个android 11分支，应该是对应的是最新的AOSP分支。所以我要正常运行应该要将我的AOSP分支切到最新，由于切换和重新编译比较耗时，就先不折腾了，后续研究要涉及最新kernel后再弄吧。其实要研究些基本原理，研究旧版的系统会减少一些阻碍，越新的系统机制越复杂。