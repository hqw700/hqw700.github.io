---
title: adb connect USB devices in WSL2
date: 2024-03-21 00:11:12
tag: [WSL2]
category: [工具使用]
excerpt: 在WSL2下使用usb adb连接真机
---
# 在WSL2下使用usb adb连接真机
> 注意：新版本中命令已改变
https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb#attach-a-usb-device

## 一，Windows下安装
### usbipd-win应用安装
- 进入 https://github.com/dorssel/usbipd-win/releases
- 下载最新的.msi文件
- 运行下载的 usbipd-win_x.msi 文件

## 二，WSL2下安装
以wsl2安装的ubuntu为例，在ubuntu下运行：
``` bash
wsl2:~$ sudo apt install linux-tools-generic hwdata
wsl2:~$ sudo update-alternatives --install /usr/local/bin/usbip usbip /usr/lib/linux-tools/*-generic/usbip 20
```

## 三，Windows下运行
以管理员打开powershell
### 3.1 列出所有usb
``` bash
PS C:\WINDOWS\system32> usbipd list
Connected:
BUSID  VID:PID    DEVICE                                        STATE
1-5    1de1:f105  Display capture-UVC05                         Not shared
1-9    045e:02d1  Xbox One 控制器                                Not shared
1-10   18d1:4ee7  Android Composite ADB Interface               Not shared
4-4    1a81:2039  USB 输入设备                                   Not shared
```

### 3.2 将usb绑定到指定的wsl2中
~~如果存在多个wsl2虚机，需要使用-d指定~~
新版本中不需要指定虚机，将虚机保持开启即可。
``` shell
PS C:\WINDOWS\system32> usbipd bind --busid 2-4 --force
PS C:\WINDOWS\system32> sbipd attach --wsl --busid 2-4
```

## 四，WSL2下运行
然后就可以在wsl2下看到该设备
``` shell
wsl2:~$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 002: ID 18d1:4ee7 Google Inc. Nexus/Pixel Device (charging + debug)
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
### 4.1 adb测试
``` shell
wsl2:~$ adb devices
List of devices attached
96KAXXXXX      device

wsl2:~$ adb shell
bonito:/ $
```

## 五，问题：
### 需要sudo启动adb start-server
该问题是由于无udev服务导致，即使写rules文件也无效。
``` shell
wsl2:~$ lsusb
Bus 001 Device 003: ID 18d1:4ee7 Google Inc.

## 其中18d1为vendorid, 4ee7为productid
wsl2:~$ sudo vim /etc/udev/rules.d/70-android.rules
SUBSYSTEM=="usb", ATTRS{idVendor}=="18d1", ATTRS{idProduct}=="4ee7",MODE="0666"

wsl2:~$ sudo service udev reload
udev does not support containers, not started
```
解决方案是wsl2打开systemd的支持 --> [使用 systemd 通过 WSL 管理 Linux 服务](https://learn.microsoft.com/zh-cn/windows/wsl/systemd)


``` shell
wsl2:~$ sudo vim /etc/wsl.conf
[boot]
systemd=true
```
然后在powershell重启wsl
``` shell
PS C:\WINDOWS\system32> wsl --shutdown 
```
