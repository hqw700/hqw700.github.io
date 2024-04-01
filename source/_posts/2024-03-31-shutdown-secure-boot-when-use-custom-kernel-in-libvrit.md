---
title: Shutdown secure boot when use custom kernel in libvrit
date: 2024/3/31 23:36
tag: [虚拟化,教程]
category: [虚拟化]
excerpt: 使用virt-install安装虚拟机并关闭secure boot, 以支持自定义kernel。
---


## 一，安装命令
以安装ubuntu20.04为例
```
$ qemu-img create -f qcow2 ubuntu.qcow2 64G
$ virt-install \
--boot uefi \
--name ubuntu \
--vcpus 4 \
--cpu host-passthrough \
--ram 4096 \
--memballoon none \
--clock offset='localtime' \
--network network=default \
--graphics vnc,listen=0.0.0.0,port=5901 \
--video=qxl \
--disk ./ubuntu.qcow2 \
--cdrom=./ubuntu20.04.iso \
--boot cdrom,hd \
--input tablet
```


## 二，安装自定义内核
出现以下错误
Loading Linux 5.10.54 ...
error: bad shim signature
![](/img/blog/atsm-boot-error.png)

### 2.1 该问题是由于此时secureboot状态为enabled
```
$ mokutil --sb-state
SecureBoot enabled
```

## 三，关闭secure boot
```
$ virsh edit ubuntu
<os firmware='efi'>
  <loader secure='yes'/>
  <firmware>
    <feature enabled='yes' name='secure-boot'/>
    <feature enabled='no' name='enrolled-keys'/>
  </firmware>
</os>
```

### 3.1 重新创建并启动虚机
如果libvirt版本高于8.1.0, 直接使用`virsh start $vm --reset-nvram`即可。  
8.1.0版本以下采用以下命令
```
$ virsh dumpxml ubuntu > ubuntu-with-nosecure.xml
$ virsh undefine --nvram ubuntu
$ virsh define ubuntu-with-nosecure.xml
$ virsh start ubuntu
```

虚机启动后查看此时SecureBoot状态，为disabled即可进行自定义内核安装
```
$ mokutil --sb-state
SecureBoot disabled
Platform is in Setup Mode
```
