---
title: Android Emulator串口使用和问题处理
date: 2025/9/5 00:16
tag: [虚拟化, bugfix]
category: [虚拟化]
excerpt: emulator串口的基本使用，以及修复串口无法使用问题
---

## 一，基本使用

在模拟器启动时，可以通过参数 --shell-serial 指定串口功能，有以下几种参数

| 启动命令 | 连接方式 |
| --- | --- |
| \-shell-serial stdio | 当前终端 |
| \-shell-serial tcp::4444,server,nowait | socat -,raw,echo=0,escape=0x04 TCP:localhost:4444 （ctrl + d exit） |
| \-shell-serial unix:/tmp/emu-console,server,nowait | socat -,raw,echo=0,escape=0x04 unix:/tmp/emu-console （ctrl + d exit） |

## 二，串口无法使用问题处理
### 调试
- 手动启动串口，adb root shell 中：
``` shell
# setprop ctl.start console
```

- dmesg log
``` log
init: Control message: Could not ctl.start for 'console' from pid: 5156 (setprop ctl.start console): Couldn't open console '/dev/ttyAMA0': Permission denied
[  408.263647] type=1400 audit(1756997294.564:1863): avc: denied { read write } for comm="init" name="ttyAMA0" dev="tmpfs" ino=331 scontext=u:r:init:s0 tcontext=u:object_r:device:s0 tclass=chr_file permissive=0
```

- `chmod 777 /dev/ttyAMA0`问题一样
- `setenforce 0`串口正常
- 查看selinux权限
``` shell
# ls -liZ /dev/ttyAMA0                                                                                                                                   
331 crw-rw-rw- 1 root root u:object_r:device:s0  204,  64 2025-09-04 22:41 /dev/ttyAMA0
```
### Fixed
- 修改aosp源码
``` diff
device/generic/goldfish$ git diff 
diff --git a/sepolicy/common/file_contexts b/sepolicy/common/file_contexts
index d58270b..f6a03d4 100644
--- a/sepolicy/common/file_contexts
+++ b/sepolicy/common/file_contexts
@@ -20,6 +20,7 @@
 /dev/dri/renderD128          u:object_r:gpu_device:s0
 /dev/ttyGF[0-9]*             u:object_r:serial_device:s0
 /dev/ttyS2                   u:object_r:console_device:s0
+/dev/ttyAMA[0-9]*            u:object_r:serial_device:s0
 
 # kernel console
 /dev/hvc0                    u:object_r:serial_device:s0
```

- 编译
```
make selinux_policy -j8
adb sync
adb reboot
```
- 重启后串口正常，查看selinux权限
``` shell
# ls /dev/ttyAMA0 -liZ
393 crw------- 1 root root u:object_r:serial_device:s0  204,  64 2025-09-04 23:56 /dev/ttyAMA0
```
## 三，在user版本中使用串口
正在在user版本中是无法使用串口的，可以修改init.rc， 在user版本中启动串口并支持root
``` diff
system/core$ git diff .
diff --git a/rootdir/init.rc b/rootdir/init.rc
index 2b53d88..5f3c0b4 100644
--- a/rootdir/init.rc
+++ b/rootdir/init.rc
@@ -475,6 +475,7 @@ on init
     start servicemanager
     start hwservicemanager
     start vndservicemanager
+    start console
 
 # Run boringssl self test for each ABI.  Any failures trigger reboot to firmware.
 on init && property:ro.product.cpu.abilist32=*
@@ -1264,9 +1265,9 @@ service console /system/bin/sh
     class core
     console
     disabled
-    user shell
-    group shell log readproc
-    seclabel u:r:shell:s0
+    user root
+    group root log readproc
+    seclabel u:r:su:s0
     setenv HOSTNAME console
 
 on property:ro.debuggable=1
``` 

> 操作系统：Mac mini M4
> 编译机：Ubuntu-22.04 AMD 5800X
> 手机：emulator 36.1 ARM64 on Mac
> 源码版本：AOSP android-13.0.0_r74