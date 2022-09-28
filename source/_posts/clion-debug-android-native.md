---
title: 【工具使用】使用clion调试Android Native程序
date: 2022-09-28 18:00
tag: [Android系统,工具使用,教程,调试]
category: [工具使用]
excerpt: 使用IDE调试surfaceflinger
banner_img: /img/bg/clion_banner.png
---

> 运行环境: windows + WSL2 
> 调试机器：pixel 3 + aosp10
> 使用工具: clion, adb

CLion是IntelliJ开发的C/C++跨平台IDE，和Android Studio有相同的UI和交互，弥补了Android Studio无法用来调试和索引native代码的不足。Android源码下有将源码生成为CLion project的文档(`$AOSP/build/soong/docs/clion.md`), 但使用clion调试不需要用到该部分功能，所以不做介绍。  
我的环境是用WSL2下载和编译android源码，windows下可以在文件管理器中通过`\\wsl$`路径直接查看代码(类似于samba)，adb和CLion都运行在windows上。 
下面以surfaceflinger程序为例的详细步骤：

## 1，使用Clion打开源码  
进入源码目录，右键Open Folder as CLion Project打开, 如`$AOSP/frameworks/native/services/surfaceflinger`

打开后点击 Run --> Edit configurations, 然后选择Remote Debug, 配置设置为:
```
Debugger: Bundled GDB
'target remote' arg: :12345 (如果是远程环境需要加上ip, 如ip:12345)
Sysroot: /home/huangqw/code/out/target/product/blueline/symbols (符号表的路径，路径为linux下的绝对路径，也可以是windows下路径)
```
![](/img/blog/clion_remote_debug_config.png)
![](/img/blog/clion_remote_debug_config2.png)

## 2, 创建.gdbinit文件
在windows的`C:\Users\%USERNAME%`目录下创建.gdbinit文件，设置以下内容。其中关键是 `set dir 源码目录`
```
set dir /home/huangqw/code
set solib-absolute-prefix /home/huangqw/code/out/target/product/blueline/symbols
set solib-search-path /home/huangqw/code/out/target/product/blueline/symbols/system/lib64:/home/huangqw/code/out/target/product/blueline/symbols/system/lib64/hw:/home/huangqw/code/out/target/product/blueline/symbols/system/lib64/ssl/engines:/home/huangqw/code/out/target/product/blueline/symbols/system/lib64/drm:/home/huangqw/code/out/target/product/blueline/symbols/system/lib64/egl:/home/huangqw/code/out/target/product/blueline/symbols/system/lib64/soundfx:/home/huangqw/code/out/target/product/blueline/symbols/vendor/lib64:/home/huangqw/code/out/target/product/blueline/symbols/vendor/lib64/hw:/home/huangqw/code/out/target/product/blueline/symbols/vendor/lib64/egl
```

## 3，Windows终端下操作
```
使用adb连接手机：
adb root
adb forward tcp:12345 tcp:12345
adb shell
blueline:/ # gdbserver64 :12345 --attach `pidof surfaceflinger`
    Attached; pid = 654
    gdbserver: Unable to determine the number of hardware watchpoints available.
    gdbserver: Unable to determine the number of hardware breakpoints available.
    Listening on port 12345
```

## 4，调试
回到CLion中，点击Run --> Debug "Unnamed" 开始调试
在截屏代码下打个断点，如SurfaceFlinger::renderScreenImplLocked
![](/img/blog/clion_debug_ok.png)

然后手动截图，成功断点如上图，接下来就可以单步运行，正常的话可以打开项目外的系统库文件。如图所示堆栈里的Looper函数。如显示的是汇编代码则不正常, 请检查.gdbinit文件，或者在gdb终端里set dir "xxx" 设置源码目录。
![](/img/blog/clion_debug_ok2.png)

## 5，变量显示optimized out问题
调试时变量显示optimized out，无法正常查看。这是由于C++编译优化的原因，在Android.bp的cflags里加个"-O0"，重新编译surfaceflinger， push生成即可。
![](/img/blog/clion_debug_ok3.png)